# Module 9 — Move semantics & rvalue references (deep)

Module 4 gave the intuition (steal from temporaries instead of copying them). This module makes you
*fluent* — the subtleties that separate "heard of moves" from "understands moves," which is exactly
what HFT interviews probe. If you can explain reference collapsing, why `return std::move(x)` is
wrong, and the `noexcept`-move / vector-realloc link without hesitating, you're in the top tier of
candidates on this topic.

---

## 1. Recap: what a move actually is, in memory

An rvalue reference `T&&` binds to temporaries; a move constructor/assignment transfers a resource's
ownership (a **pointer swap**) instead of duplicating it (a **deep copy**). "Transfer ownership" is
concrete: you copy the *handle* (the pointer/size), then null the source's handle so its destructor
frees nothing.

```
before move:  a.buf ─▶ [heap 1000B]      b.buf = ? (uninitialized)
              a.len = 1000

  Buffer b = std::move(a);   // move ctor: b takes a's pointer, then a is emptied

after move:   a.buf = nullptr            b.buf ─▶ [heap 1000B]   (SAME block — no copy, no alloc)
              a.len = 0                   b.len = 1000
              (a's dtor later: delete nullptr → harmless)
```

A copy would instead `malloc` a second 1000-byte block and `memcpy` into it. For a `std::string`,
`std::vector`, or `std::unique_ptr`, that's the whole performance story: **move = 2-3 pointer writes;
copy = allocate + copy N bytes.** For a plain `int`/`double`, there's nothing to steal, so "move" is
just a copy (and that's fine — no harm).

---

## 2. Value categories, one level deeper

Module 4 introduced lvalues (have a name/address, persist) and rvalues (temporaries, about to die).
The standard actually splits values into finer categories, and interviewers occasionally push here:

- **lvalue** — has identity, can't be (implicitly) moved from: a named variable `x`, `*p`, `arr[i]`.
- **prvalue** ("pure rvalue") — a temporary with no name: `42`, `x + y`, `makeBuffer()` (the returned
  temporary), `std::string("hi")`.
- **xvalue** ("eXpiring value") — has identity **but** is movable-from: the result of `std::move(x)`
  or a function returning `T&&`.

The practical grouping: **glvalue** = lvalue ∪ xvalue (has identity); **rvalue** = prvalue ∪ xvalue
(movable-from). All you usually need: *rvalues bind to `T&&` and can be moved from; lvalues bind to
`T&` (or `const T&`) and can't be moved unless you `std::move` them (turning them into an xvalue).*

---

## 3. Reference collapsing & forwarding references

`T&&` means different things depending on context — this is the single most misunderstood corner:

```cpp
void f(Widget&& w);          // rvalue reference — binds ONLY to rvalues

template <typename T>
void g(T&& x);               // FORWARDING (a.k.a. universal) reference — binds to ANYTHING
```

The difference: `Widget&&` is a concrete rvalue reference. But `T&&` where `T` is a **template
parameter being deduced** is a *forwarding reference* — it binds to lvalues and rvalues alike,
*preserving* their value category. This works via **reference collapsing**, the rule for what happens
when references stack up during deduction:

```
  T& &   →  T&       (lvalue ref wins)
  T& &&  →  T&
  T&& &  →  T&
  T&& && →  T&&       (only rvalue-ref-to-rvalue-ref stays rvalue)
```

When you pass an **lvalue** to `g(T&& x)`, `T` deduces to `Widget&`, so the parameter type is
`Widget& &&` → collapses to `Widget&`. Pass an **rvalue** and `T` deduces to `Widget`, parameter type
`Widget&&`. So the *same* `T&&` syntax adapts to whatever came in — that's the mechanism behind
perfect forwarding.

> **The trap:** `T&&` is a forwarding reference **only** when `T` is deduced *for that very call*.
> `template<class T> void h(std::vector<T>&& v)` is a plain rvalue reference (`T` is deduced, but the
> parameter isn't the bare `T&&`). And `auto&& r = expr;` *is* a forwarding reference (`auto` deduces
> like `T`).

---

## 4. `std::move` and `std::forward` — casts, not actions

Neither of these *moves* or *forwards* anything at runtime. They are compile-time **casts** that
change the value category so overload resolution picks the move/forward path.

- **`std::move(x)`** — *unconditionally* casts to an rvalue (`static_cast<T&&>`). It means "I'm done
  with `x`, feel free to gut it." It does **not** move by itself — it enables the next function
  (a move ctor/assign) to be chosen.
- **`std::forward<T>(x)`** — *conditionally* casts, preserving the caller's original category. Used
  inside a forwarding-reference function to pass an argument onward *as whatever it was*.

Inside `g(T&& x)`, the *named* parameter `x` is an **lvalue** (it has a name!), even if it was bound to
an rvalue. So a bare `consume(x)` would always copy. To preserve the original category, forward it:

```cpp
template <typename T>
void relay(T&& x) {
    consume(std::forward<T>(x));  // lvalue in → passed as lvalue; rvalue in → passed as rvalue
}
```

**Rule:** use `std::move` for rvalue references (`T&&` where `T` is fixed), `std::forward<T>` for
forwarding references (deduced `T&&`). Mixing them up is a classic bug: `std::move` on a forwarding
reference will happily gut an lvalue the caller still intended to use.

---

## 5. Why moves must be `noexcept`

`std::vector` reallocation is the killer example:

```cpp
std::vector<Buffer> v;
v.push_back(...);   // when capacity is exceeded, vector allocates a bigger array
                    // and must transfer the existing elements into it.
```

Vector promises **strong exception safety**: if transferring element *k* throws, the original vector
must be left untouched (commit-or-rollback). It can only guarantee that if the transfer *can't throw* —
because if a move threw halfway, the already-moved-from source elements are gutted and there's no way
back. So:

- Move ctor is `noexcept` → vector **moves** elements (fast, O(1) each).
- Move ctor might throw → vector **copies** elements (slow, allocates each) to preserve rollback.

```
push_back triggers growth:

   noexcept move:    [old array] ══move══▶ [new bigger array]   (steal pointers, cheap)
   throwing move:    [old array] ──copy──▶ [new bigger array]   (deep copy each, kept until success)
```

A missing `noexcept` on your move constructor can silently turn *every* vector growth into a full
deep copy — a huge, invisible performance regression. **Always mark correct moves `noexcept`.**

```cpp
Buffer(Buffer&& o) noexcept { /* steal */ }   // the noexcept is not optional in practice
```

The container checks this via `std::move_if_noexcept` / the `is_nothrow_move_constructible` trait — a
good follow-up answer if an interviewer asks "how does the vector *know*?"

---

## 6. The moved-from state

After `auto b = std::move(a)`, `a` is **valid but unspecified**:

- **Valid:** its destructor will run correctly; you may assign a new value to it; you may call any
  operation with no precondition on its value (`clear()`, `size()`, `push_back()` on a container).
- **Unspecified:** don't rely on *what* value it holds. `a.front()` after moving a vector is UB
  (empty precondition), even though the vector is "valid."

Design your own move to leave the source in a **safe empty state** (nulled pointers, zero size) so its
destructor is a no-op and any later assignment is clean. This is not optional — a move that leaves a
dangling pointer in the source causes a double-free when both objects are destroyed.

---

## 7. Copy elision / RVO / NRVO — the compiler often skips the move entirely

```cpp
Buffer make() {
    Buffer b(1000);
    return b;         // NRVO: b constructed directly in the caller's storage — no move at all
}
Buffer x = make();    // zero copies, zero moves
```

- **RVO** (Return Value Optimization — returning an *unnamed* temporary, `return Buffer(1000);`) is
  **mandatory** in C++17: the object is constructed directly in the caller's slot, guaranteed.
- **NRVO** (Named RVO — returning a *named* local, `return b;`) is **optional** but done by every
  serious compiler at `-O1`+.
- **Don't write `return std::move(b);`.** It looks like an optimization but it *prevents* NRVO: it
  turns the return expression into an xvalue, which no longer matches the "returning a local object"
  form the compiler elides. You force an actual move where you'd have gotten *nothing*. Just
  `return b;`. (Compilers even warn: `-Wpessimizing-move`.)

Mental model: the copy/move you were worried about often doesn't happen at all — the value is built in
place. Reach for `std::move` only when returning a *member* or a *by-value parameter* (where NRVO
can't apply), not a plain local.

---

## 8. When does a move actually help?

Only for types that **own a resource** (heap buffer, file, socket, another movable object). Moving a
type that's all trivial value members is just a copy — nothing to steal:

- `std::string`, `std::vector`, `std::unique_ptr` → move is a big win (pointer steal vs. allocate +
  copy).
- A POD `struct Order { uint64_t id; int64_t price; uint32_t qty; }` → move == copy (all trivial
  members, register/`memcpy` sized). That's fine; no benefit, no harm — don't contort code to "move"
  a POD.

So don't sprinkle `std::move` on `int`s and small PODs expecting magic; it does nothing there.

---

## 9. Pass-by-value + move ("sink" parameters)

When a function *stores* (sinks) its argument, take it **by value** and `std::move` it into place.
This single overload optimally serves both lvalue-copy and rvalue-move callers:

```cpp
class Book {
    std::vector<Order> log_;
public:
    void record(Order o) { log_.push_back(std::move(o)); }  // caller can move OR copy in
};
// record(std::move(existing));  // rvalue → o move-constructed → moved into vector (2 cheap moves)
// record(makeOrder());          // rvalue temporary → same, moves through
// record(existing);             // lvalue → o copy-constructed once → then moved into vector
```

The alternative — writing two overloads `record(const Order&)` and `record(Order&&)` — is more code
and doesn't scale (N sink parameters → 2^N overloads). The by-value-and-move idiom is the standard
answer, with the caveat that for a *cheap-to-move* type it's near-optimal, and only a hand-tuned
`const&`/`&&` pair edges it out on the very hottest paths.

---

## 10. Common pitfalls & UB

- **`std::move` on a forwarding reference** (`T&&` deduced) — should be `std::forward<T>`; gutting an
  lvalue the caller still needs.
- **`return std::move(local)`** — disables NRVO, pessimizes (§7).
- **Missing `noexcept` on a correct move** — containers copy instead of move (§5).
- **Using a moved-from object's value** (not just destroying/reassigning it) — UB if it has a
  precondition (§6).
- **A move that leaves the source owning the resource too** (forgot to null the source) → double free.
- **Assuming `std::move` does work** — it's a cast; if no move ctor exists (or it's deleted), you
  silently get a *copy*.
- **`const` rvalue** — `std::move` on a `const T` yields `const T&&`, which binds to the *copy*
  constructor (`const T&`), not the move ctor. Moving a `const` object copies it.

---

## 11. Quiz — tough problems

**Q1.** What does this print?

```cpp
struct S {
    S() { std::cout << "ctor "; }
    S(const S&) { std::cout << "copy "; }
    S(S&&) noexcept { std::cout << "move "; }
};
S make() { S s; return s; }
int main() { S x = make(); }
```

**Answer:** `ctor ` only (in C++17). NRVO constructs `s` directly in `x`'s storage — no copy, no move.
If NRVO were disabled you'd see `ctor move`. You would **never** see `copy` (the move ctor exists and
`s` is an expiring local).

---

**Q2.** Now the author "optimizes" `return s;` to `return std::move(s);`. What changes?

**Answer:** Prints `ctor move` — the `std::move` turns the return into an xvalue, **disabling NRVO**,
so you now pay for a move you'd otherwise have skipped entirely. This is a pessimization; compilers
warn `-Wpessimizing-move`. Lesson: never `std::move` a local you're returning.

---

**Q3.** Predict the output:

```cpp
void h(int&)        { std::cout << "lvalue "; }
void h(int&&)       { std::cout << "rvalue "; }
template<class T> void relay(T&& x) { h(std::forward<T>(x)); }
int main() {
    int a = 5;
    relay(a);
    relay(10);
    relay(std::move(a));
}
```

**Answer:** `lvalue rvalue rvalue `. `relay(a)`: `T=int&`, forwards as lvalue. `relay(10)`: `T=int`,
forwards as rvalue. `relay(std::move(a))`: `a` is cast to xvalue, `T=int`, forwards as rvalue. If
`relay` used `std::move(x)` instead of `std::forward<T>(x)`, all three would print `rvalue` — and
`relay(a)` would have gutted `a` behind the caller's back.

---

**Q4.** Why does this deep-copy despite the `std::move`?

```cpp
const std::string s = "hello";
std::string t = std::move(s);   // ?
```

**Answer:** `s` is `const`, so `std::move(s)` produces a `const std::string&&`. The move constructor
takes `std::string&&` (non-const) and can't bind a `const` rvalue; overload resolution falls back to
the **copy** constructor (`const std::string&`). So you copy. Moral: don't declare things `const` if
you intend to move out of them.

---

**Q5.** Is `T&&` in `template<class T> void f(std::vector<T>&& v)` a forwarding reference?

**Answer:** **No.** A forwarding reference must be the *bare deduced parameter* `T&&`. Here the
parameter is `std::vector<T>&&` — a plain rvalue reference to a vector; `f` won't bind to an lvalue
vector. Only `template<class T> void f(T&& v)` (and `auto&&`) are forwarding references.

---

**Q6.** After the move, is calling `v.push_back(1)` and then `v.size()` well-defined?

```cpp
std::vector<int> v{1,2,3};
auto w = std::move(v);
v.push_back(1);
std::cout << v.size();
```

**Answer:** Well-defined. `v` is valid-but-unspecified after the move — its contents are unspecified,
but `push_back` and `size` have no value precondition, so both are legal. In practice a moved-from
`std::vector` is empty, so this very likely prints `1`, but you should only rely on "valid," not on the
specific emptiness.

---

**Q7.** You mark a move constructor `noexcept` but it can actually throw (it allocates). What happens?

**Answer:** If it *does* throw, `std::terminate` is called (a `noexcept` function that throws
terminates immediately — no unwinding). So `noexcept` is a **promise**, not a wish: only mark a move
`noexcept` if it genuinely can't throw (pure pointer-stealing moves can't; a "move" that allocates
can). Lying to get the vector fast-path will crash you on the throwing case.

---

**Q8.** How many allocations occur across these three `record` calls, given
`void record(Order o){ log_.push_back(std::move(o)); }` and `Order` is a POD?

```cpp
record(makeOrder());          // (a)
Order e = makeOrder();
record(e);                    // (b)
record(std::move(e));         // (c)
```

**Answer:** For a **POD** `Order`, "move" == copy (trivial members), so the by-value `o` is
constructed by copy/move-of-trivial and `push_back` copies it in. No per-call heap allocation from the
moves themselves; the only heap traffic is `log_` growing its backing array occasionally (amortized).
The by-value-sink idiom's *move* benefit only materializes for resource-owning element types
(`std::string`, nested vectors); for PODs it's neutral.

---

## 12. Indian HFT interview questions

**Q1. "Difference between `std::move` and `std::forward`? When each?"** (Tower, Optiver, Graviton,
Quadeye, HRT — extremely common.)
*Model answer:* Both are compile-time casts, not runtime operations. `std::move` unconditionally casts
to rvalue — use with a fixed-type `T&&` rvalue reference when you're done with the object.
`std::forward<T>` conditionally casts, preserving the caller's value category — use inside a
forwarding-reference (`template<class T> f(T&& )`) to pass the argument onward as whatever it was.
Mixing them guts lvalues the caller still needs.

**Q2. "What is a forwarding/universal reference and how does reference collapsing work?"**
*Model answer:* A `T&&` where `T` is deduced for that call binds to both lvalues and rvalues,
preserving category. Lvalue arg → `T = U&` → `U& &&` collapses to `U&`; rvalue arg → `T = U` → `U&&`.
The four collapse rules: only `&& &&` stays `&&`, everything with a `&` collapses to `&`. This powers
perfect forwarding. Caveat: `auto&&` is one too; `std::vector<T>&&` is not.

**Q3. "Why must a move constructor be `noexcept`? What breaks otherwise?"**
*Model answer:* `std::vector` (and friends) only *move* elements on reallocation if the move is
`noexcept`; otherwise they *copy* to preserve the strong exception guarantee. A non-`noexcept` move →
silent O(n) deep copies on every growth. It's the difference between amortized-cheap and quadratic-ish
container behavior. Checked via `move_if_noexcept`.

**Q4. "What's wrong with `return std::move(x);`?"**
*Model answer:* It disables NRVO — the compiler would otherwise construct the local directly in the
caller's storage (zero moves); `std::move` forces an actual move by making the return an xvalue.
Pessimization, warned by `-Wpessimizing-move`. Just `return x;`. (Exception: returning a *member* or
a by-value *parameter*, where NRVO can't apply anyway.)

**Q5. "Describe the state of a moved-from object. Is it safe to use?"**
*Model answer:* Valid but unspecified. Safe to destroy or assign a new value; safe to call operations
with no value precondition. Not safe to rely on its contents. For your own types, engineer the
moved-from state as a safe empty (null pointers, zero size) so the destructor is a no-op.

**Q6. "For a POD `Order` struct, does move buy you anything over copy?"**
*Model answer:* No — all members are trivial, so the move is a bitwise copy; there's nothing to steal.
Move semantics pay off only for resource-owning types. Don't contort POD code with `std::move`.

**Q7. "You take a parameter you'll store in a member — how do you write the signature?"**
*Model answer:* By value + `std::move` into the member (the "sink" idiom): one function serves
lvalue-copy and rvalue-move callers, moves through cheaply. For the very hottest path on a
cheap-to-move type, a `const&`/`&&` overload pair can edge it out, but by-value-and-move is the
default.

---

## 13. In the order book

Pooled `Order`s are trivially-movable PODs, so moves are copies (cheap, fixed-size) — the move
machinery doesn't help *there*. It matters in the *surrounding* infrastructure: input queues of
variable-size parsed messages, `std::vector<Trade>` result logs that grow (need `noexcept` moves to
grow cheaply rather than deep-copying each `Trade`), `std::string` symbol/venue fields, and sink-
parameter APIs (`record(Order o)`) for logging fills. The interview payoff: being able to say
precisely *where* moves help (resource-owning members, growing vectors) and where they're neutral
(POD `Order`s) — that distinction is what separates rote knowledge from understanding.

---

## Key takeaways

- A move **transfers ownership by stealing the handle and nulling the source** — 2-3 pointer writes vs.
  allocate-and-copy. Only resource-owning types benefit; PODs "move" == copy.
- `std::move` and `std::forward` are **compile-time casts**, not actions: `move` = unconditional
  rvalue cast (fixed `T&&`); `forward<T>` = conditional, category-preserving (deduced `T&&`).
- **Forwarding references** (`T&&` with deduced `T`, and `auto&&`) bind to anything via **reference
  collapsing** — the engine of perfect forwarding.
- **Mark correct moves `noexcept`** or containers deep-copy on reallocation. `noexcept` is a promise;
  a `noexcept` move that throws calls `std::terminate`.
- Moved-from = **valid but unspecified** (destroy/reassign OK, don't read the value). Don't write
  `return std::move(local)` — it kills NRVO. Use by-value + `std::move` for sink parameters.

**Next:** [10 — Templates → Concepts](10-templates-concepts.md)
