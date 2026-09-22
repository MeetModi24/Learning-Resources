# Module 4 — lvalues, rvalues & an intro to moving

The distinction beginners find hardest — and it's really a *performance* feature in disguise. Value
categories exist so the compiler can tell, at each expression, whether an object is something that
**lives on** (must be preserved) or something that is **about to die** (safe to cannibalize). Get
this right and modern C++ returns megabyte objects by value for free; get it wrong and you ship code
that silently deep-copies on every hot-path call. HFT interviews probe this relentlessly because
"why is this 10× slower than it should be" almost always traces back to a copy that should have been
a move. Module 9 goes deep on the mechanics; this module builds the intuition and the mental model
of *where things live in memory*.

---

## 1. What "value category" even means

Beginners think of an expression's *type* (`int`, `std::string`). C++ also gives every expression a
second, orthogonal property: its **value category**. Type answers "what is it?"; value category
answers "**does it have a durable identity, or is it a disposable temporary?**"

The first-level split you must internalize:

- **lvalue** — has **identity**: a name, a stable address in memory, it persists beyond the current
  expression. You can take its address with `&`. It can appear on the **l**eft of `=`. Think:
  "a box that has a label on it and will still be there next line."
- **rvalue** — a **temporary**: the result of a computation, a literal, a function returning by
  value. It has no name you can refer to again and is about to be destroyed at the end of the full
  expression. You generally **cannot** take its address. Think: "a value in transit, with no box."

```cpp
int x = 5;
//  ^   ^-- rvalue: the literal 5 (a temporary, no address of its own)
//  lvalue: x (named, addressable, persists)

int y = x + 1;   // (x + 1) is an rvalue — a freshly manufactured temporary result
&x;              // OK — x has an address
// &(x + 1);     // ERROR — you can't take the address of a temporary
// &5;           // ERROR — same reason
```

A one-line test that gets you 90% of the way: **if you can legally write `&expr`, it's an lvalue.**

### The finer categories (know they exist)

C++11 refined this into a lattice. You need the top-level split first, but interviewers do ask for
the full picture:

```
                 expression
                /          \
           glvalue        rvalue
          /       \      /      \
      lvalue      xvalue       prvalue
   (has identity, (has identity, (pure temporary,
    not movable)   IS movable)    no identity)
```

- **prvalue** ("pure rvalue") — a literal or a computed result that initializes something: `5`,
  `x + 1`, `str.substr(2)`. No identity, produces a value.
- **xvalue** ("eXpiring value") — an object with identity that has been *marked as movable*, e.g.
  `std::move(x)` or a function returning `T&&`. It has an address but you've said "I'm done with it."
- **lvalue** — identity, not (implicitly) movable: `x`, `arr[3]`, `*ptr`, `obj.member`.
- **glvalue** = lvalue ∪ xvalue (has identity). **rvalue** = xvalue ∪ prvalue (movable).

For day-to-day reasoning the lvalue/rvalue split is enough; the xvalue slot is exactly what
`std::move` produces (Section 5).

---

## 2. Where do these things live in memory?

Value category is a *language* concept, but it maps onto a *physical* one. Here is the mental
picture for `Buffer d = make();` where `make()` returns by value:

```
STACK (frame of the caller)                 HEAP
┌───────────────────────────┐               ┌────────────────────────┐
│ d : Buffer                 │               │  [ a million ints ...] │
│   ├ data ──────────────────┼──────────────▶│  at 0xfff00            │
│   └ n    = 1000000         │               └────────────────────────┘
├───────────────────────────┤
│ (temporary returned by     │   ← a prvalue lives here *briefly*; in C++17 it is
│  make(), often elided      │     constructed directly into d's storage (RVO), so
│  straight into d)          │     often there is no separate temporary at all.
└───────────────────────────┘
```

The key insight: **the small handle (`data`, `n`) lives on the stack inside the object; the big
payload lives on the heap.** A temporary's *handle* is on the stack about to vanish, but the *heap
block it points to* is a separate allocation that does **not** have to be freed and re-created — it
can simply be re-owned. That gap between "handle" and "payload" is the entire reason moving exists.

---

## 3. The problem moving solves

Consider a class that owns a heap buffer:

```cpp
class Buffer {
    int* data;      // owns a heap array
    std::size_t n;
    // ctor: data = new int[n];   dtor: delete[] data;
};

Buffer make() {
    Buffer b;       // fills a million elements
    return b;       // returns a temporary (rvalue)
}
Buffer x = make();  // what happens?
```

Without moving, `Buffer x = make()` would **copy** a million ints from the returned temporary into
`x` — allocate a *second* million-int block, memcpy across, then destroy the temporary's block.
Absurd: the source is dying at the semicolon anyway. We paid `O(n)` to duplicate data we're about to
throw away.

---

## 4. The insight: steal the pointer

The temporary's heap buffer is about to be freed. Instead of copying its contents, **take its
pointer** and null out the source so its destructor frees nothing. `O(1)` instead of `O(n)`:

```cpp
Buffer(Buffer&& other) noexcept {   // Buffer&& = rvalue reference: binds only to temporaries
    data = other.data;    // steal the handle
    n    = other.n;
    other.data = nullptr; // leave source empty so ITS dtor frees nothing
    other.n    = 0;
}
```

`T&&` (double ampersand) is an **rvalue reference** — a reference that binds *only to rvalues*. It is
the type system's way for a function to announce: "I know this argument is disposable, so I may gut
it." (`T&`, a plain lvalue reference, binds to lvalues; `const T&` binds to *both* — that's why it's
the safe read-only default from Module 2.)

### What actually happens byte-for-byte

`data = other.data;` is a **shallow copy of the pointer** — and that's the point. Say `other.data`
holds the address `0xfff00`:

```
before:  other.data ─▶ 0xfff00 ─▶ [ a million ints ]

data = other.data;     // copy the ADDRESS 0xfff00 (one machine word), not the ints
         this->data ─▶ 0xfff00 ─▶ [ the array ]   ← both now point at the SAME block
         other.data ─▶ 0xfff00 ─┘

other.data = nullptr;  // sever the source's claim
         this->data ─▶ 0xfff00 ─▶ [ the array ]   ← sole owner
         other.data ─▶ nullptr                    ← owns nothing
```

If we skipped the null-out, **both** destructors would `delete[] 0xfff00` → **double free** (Module
3). Nulling the source makes its later `delete[] nullptr` a safe no-op, so the block is freed
exactly once. A move is therefore: **shallow-copy the handle, then invalidate the old handle.**
Nothing in the heap relocates — the array stays at `0xfff00` the whole time; only *which pointer is
allowed to free it* changes. The **ownership** moves; the **data** stays put.

### This only works because the payload is on the heap

Moving beats copying *only when the object owns its payload indirectly* — it holds a small pointer
and the real data lives elsewhere (heap). If the data is stored **inline / by value** inside the
object, there is **no pointer to steal**, so move degrades to a full copy:

```cpp
struct Inline { int data[100]; };   // 400 bytes stored RIGHT HERE, in the object
// move ctor can only copy all 400 bytes element-by-element — nothing to reassign.
```

```
Buffer (heap-owning):            Inline (by-value):
  [ ptr ] ─▶ [ big buffer ]        [ 400 bytes, inline ]
   ^ steal the ptr, O(1)           ^ no handle to steal → copy all of it, O(n)
```

So for plain scalar / fixed-array / inline data (the stuff that lives directly on the stack or
directly inside a pooled object), **move == copy** — `std::move` on it is legal but buys nothing.
This is exactly why `OrderPool` hands out *pointers/indices* into `storage`: an `Order` is all inline
fields, so moving its bytes is no cheaper than copying — you get `O(1)` transfer only by moving a
*pointer to* the order, reintroducing the indirection yourself. **Rule: move helps only for types
that own a resource through a pointer (heap buffer, file handle, socket).**

The compiler then chooses copy vs move by the source's value category:

```cpp
Buffer a;
Buffer c = a;          // a is an lvalue (lives on) -> COPY (mustn't damage it)
Buffer d = make();     // rvalue temporary        -> MOVE (safe to gut)
```

Same syntax; copy-vs-move chosen automatically by whether the source is an lvalue or rvalue. *That's
the entire reason value categories exist.*

---

## 5. `std::move`: "treat this lvalue as disposable"

Sometimes you have an lvalue you're *done with* and want to force a move. `std::move` moves nothing
at runtime — it is purely a **compile-time cast** from lvalue to rvalue reference (`static_cast<T&&>`),
granting permission to steal. It produces an **xvalue**:

```cpp
Buffer a;
Buffer b = std::move(a);  // force MOVE: b steals a's guts
// a is now valid-but-empty. Don't rely on its value; you may reassign or destroy it.
```

Read `std::move(a)` as: "I promise I'm finished with `a`'s contents — feel free to cannibalize." It
compiles to *zero instructions on its own*; all it does is change which overload (copy ctor vs move
ctor) the compiler selects. A common interview gotcha: `std::move` on a `const` object silently
falls back to a **copy**, because the move ctor takes `Buffer&&` (non-const) and can't bind to a
`const` xvalue, so overload resolution picks the copy ctor taking `const Buffer&`. `const` + `move`
= copy.

---

## 6. Deep dive: what the compiler does on `return x;`

Common confusion: "`x` is a named, addressable variable — an lvalue — so how does `return x;` move
from it?" Both facts are true at once, because *value category* and *how the return value is
constructed* are different questions.

- **What category is the expression `x`?** → **lvalue**. Always. `&x` is legal; it names an object.
  `return` does not reclassify it.
- **How is the returned object constructed?** → the return statement is *allowed to move from `x`*
  even though `x` is an lvalue, because `x` is a local about to be destroyed the instant the frame
  pops. Keeping it intact is pointless.

The exact steps the compiler takes for `return x;` where `x` is a local of the return type:

1. **Try NRVO (elision) first.** If it can, the compiler constructs `x` *directly in the caller's
   return slot* — `x` and the returned object are literally the same memory. No copy, no move, not
   even a move-constructor call. This is Named Return Value Optimization. Frequently `return x;`
   costs **nothing**.
2. **If elision doesn't apply, try to move.** Overload resolution for building the return value
   treats `x` **as if it were an rvalue** — so it looks for a **move constructor** (binds to `T&&`)
   first. Found → the returned object steals `x`'s guts (pointer swap), `x` is left valid-but-empty
   and then destroyed.
3. **Fall back to copy.** No usable move constructor (e.g. the type only has a copy ctor) → treat
   `x` as the lvalue it is and **copy**.

So the order is **elide → move → copy**, best to worst. `x` stays an lvalue by category the whole
time; step 2 is a special rule that *lets the return machinery move from a dying local*, often called
**implicit move on return**.

```cpp
std::string make() {
    std::string x = "hello";
    return x;   // step 1 (NRVO) usually; else step 2 picks string's MOVE ctor.
}               // NEVER a full deep copy in practice.
```

Corollary: **don't write `return std::move(x);`** for a plain local — it forces step 2 and *disables
step 1* (NRVO), because `std::move(x)` is an xvalue expression, not the named object the elision rule
requires. It's usually slower, and modern compilers will `-Wpessimizing-move` warn you.

---

## 7. Two different "lvalue → rvalue" things — don't conflate them

`x + 1` is an rvalue and `return x;` can move from `x`, but these ride on **different mechanisms**:

**lvalue-to-rvalue conversion** — *reading the value out of an object.* When you use a variable's
contents, the CPU reads the stored value into a register; that read yields a prvalue. `x` stays put
and unchanged.

```cpp
int y = x + 1;
// x undergoes lvalue→rvalue conversion: read x's value (say 10) into a register.
// operator+ then MANUFACTURES a fresh temporary 11.
// The expression (x + 1) is a prvalue because + produces a NEW value —
// not because x was "turned into" an rvalue. x is untouched.
```

**Implicit move on return** — *which constructor builds the result.* Not a value read; it's overload
resolution binding `x` to `T&&` so the move ctor wins, which **cannibalizes** `x`.

| | `x + 1` (lvalue→rvalue conversion) | `return x;` (implicit move) |
|---|---|---|
| About | **reading** the value stored in `x` | **which constructor** builds the result |
| Result | a fresh prvalue; `x` untouched | move ctor selected; `x` gutted (valid-but-empty) |
| Applies to | any read (mainly scalars) | class types with a move constructor |

Loosely both "involve an lvalue doing something rvalue-ish," but one is a **read that leaves the
source intact**, the other lets the source be **plundered because it's about to die**.

---

## 8. Binding rules — the reference cheat sheet

Which reference type can bind to which category is the single most quizzed table in this area:

| Reference | binds to lvalue? | binds to rvalue? | typical use |
|---|---|---|---|
| `T&`        | ✅ (non-const only) | ❌ | in/out parameter you'll modify |
| `const T&`  | ✅ | ✅ | **the read-only default** — binds to everything, extends temporary lifetime |
| `T&&`       | ❌ | ✅ | move ctor / "I may gut this" |

The one surprising row: **`const T&` binds to a temporary and extends its lifetime** to the lifetime
of the reference:

```cpp
const std::string& r = std::string("hi");  // temporary's life is extended to r's scope
// std::string& bad = std::string("hi");   // ERROR: non-const lvalue ref can't bind an rvalue
```

This lifetime-extension trick is real but a classic trap when the reference is a *member* (it does
**not** extend across a constructor — see the quiz).

---

## Common pitfalls & UB

- **Using a moved-from object's value.** After `std::move(a)`, `a` is *valid but unspecified*. You
  may reassign or destroy it; reading its old value is a logic bug (not UB, but wrong).
- **`return std::move(local);`** — disables NRVO, usually pessimizes. Just `return local;`.
- **`std::move` on a `const` object** — silently copies (can't bind `const` to `T&&`).
- **Dangling reference to a temporary** — `const T& f() { return T{}; }` returns a reference to a
  destroyed temporary → UB. Lifetime extension does *not* apply through a function return.
- **Taking `&` of an rvalue** — ill-formed; the compiler stops you, but people try it in generic code.
- **Assuming move is always cheaper** — for inline/POD types it's identical to a copy.

---

## Quiz — tough problems

**Q1.** What is the value category of each: `42`, `x` (an `int` variable), `std::move(x)`, `x++`,
`++x`, `arr[i]`, `str.substr(1)`?

**Answer:** `42` prvalue; `x` lvalue; `std::move(x)` xvalue; `x++` prvalue (returns the old value by
value); `++x` lvalue (returns a reference to `x`); `arr[i]` lvalue; `str.substr(1)` prvalue.

**Q2.** Why does `const T&` bind to a temporary but `T&` does not?

**Answer:** Binding a non-const lvalue reference to a temporary would let you *mutate* an object
that's about to be destroyed and whose changes nobody can observe — almost always a bug, so the
language forbids it. `const T&` can't mutate, so it's safe; the standard additionally *extends the
temporary's lifetime* to the reference's scope.

**Q3.** Predict the output:
```cpp
struct S { S(){puts("ctor");} S(const S&){puts("copy");} S(S&&)noexcept{puts("move");} ~S(){puts("dtor");} };
S make(){ S s; return s; }
int main(){ S a = make(); }
```
**Answer:** With C++17 and NRVO: `ctor` then `dtor` — one construction, one destruction, **no copy
or move** (elided). If you compile with `-fno-elide-constructors` you'd instead see `ctor move dtor
dtor` (the return moves, then the temporary/`s` is destroyed). This is the canonical "does NRVO
fire?" question.

**Q4.** Is this UB?
```cpp
std::vector<int> v = {1,2,3};
auto w = std::move(v);
std::cout << v.size();
```
**Answer:** Not UB — it's well-defined but *unspecified*. `v` is a valid moved-from `vector`; calling
`size()` is legal. In practice libstdc++/libc++ leave it empty (`0`), but you must not *rely* on that.

**Q5.** Why can `return std::move(x)` be slower than `return x`?

**Answer:** `std::move(x)` is an xvalue, not the named local the NRVO rule keys on, so it **disables
copy elision**, forcing an actual move-constructor call where the compiler could have constructed the
result in place for free. For a type where move is expensive-ish (or where the elided path is free),
that's a regression.

**Q6.** Spot the bug:
```cpp
struct Wrapper { const std::string& s; Wrapper(const std::string& x): s(x) {} };
Wrapper w{std::string("temp")};
std::cout << w.s;
```
**Answer:** Dangling reference → UB. Lifetime extension of a temporary bound to a `const&` applies to
a *local reference variable*, **not** to a reference member initialized in a constructor. The
temporary dies at the end of the `Wrapper w{...}` full-expression; `w.s` then dangles.

**Q7.** After `std::move`, is the source object destroyed?

**Answer:** No. Move constructs a *new* object from the source and leaves the source alive in a
valid-but-unspecified state; the source's destructor still runs later, normally. Move ≠ destruction.

**Q8.** Given `void f(int&&); void f(const int&);`, which overload does `int i=5; f(i);` call, and
which does `f(5);` call?

**Answer:** `f(i)` calls `f(const int&)` (`i` is an lvalue, can't bind to `int&&`); `f(5)` calls
`f(int&&)` (rvalue prefers the rvalue-ref overload). This is exactly how `push_back(const T&)` vs
`push_back(T&&)` dispatch.

**Q9.** Why is `std::move` on a `const std::string` a silent performance bug?

**Answer:** The move ctor is `string(string&&)` — non-const. A `const` xvalue can't bind to it, so
overload resolution falls back to `string(const string&)`, a deep copy. The `std::move` compiled to
nothing and you copied anyway. Never mark a to-be-moved local `const`.

---

## Indian HFT interview questions

**Q (Graviton / Tower Research, Gurgaon): "Explain lvalue vs rvalue and why C++ needs the
distinction."**
Value category tells the compiler whether an object has durable identity (lvalue) or is a
disposable temporary (rvalue). The distinction lets overload resolution pick a **move** (steal the
resource, `O(1)`) for temporaries and a **copy** (duplicate, safe) for lvalues automatically — same
syntax, right cost. Without it, returning big objects by value or growing a `vector` would deep-copy.

**Q (Optiver): "Does `std::move` actually move anything?"**
No. It's a compile-time `static_cast` to an rvalue reference — zero runtime instructions. The move
*happens* later, when a move constructor/assignment is *selected* because the argument is now an
xvalue. `std::move` only grants permission.

**Q (Quadeye / AlphaGrep): "When is a move no faster than a copy?"**
When the type stores its payload inline (POD structs, fixed arrays, all-scalar types). There's no
pointer/handle to steal, so the "move" copies every byte. This is why the order book pools fixed-size
`Order` objects and passes indices/pointers rather than relying on element moves.

**Q (IMC / Jump): "What state is a moved-from object in? Can you use it?"**
Valid but unspecified. Its invariants hold, its destructor will run correctly, and you may assign it
a fresh value — but you must not read its prior value. For STL types it's typically empty, but that's
not guaranteed by the standard.

**Q (Da Vinci / Squarepoint): "Why shouldn't you write `return std::move(x)`?"**
It turns the returned named local into an xvalue, disabling NRVO/copy elision. The compiler could
have constructed the result directly in the caller's slot (zero cost); you forced a move instead.
Compilers warn with `-Wpessimizing-move`.

**Q (Hudson River / Millennium): "You profiled a hot function returning a `std::vector` and see a
memcpy of the whole buffer. What went wrong?"**
Likely a copy where a move was intended: the source was an lvalue you forgot to `std::move`, or the
type's move ctor isn't `noexcept` (so `vector` growth copies — Module 9), or the local was `const`.
Fix: make the move path reachable (`noexcept` moves, non-`const` locals, `return local;` for NRVO).

**Q (NK Securities / Mansard): "Given `f(int&&)` and `f(const int&)`, trace which is called for a
literal, a named int, and `std::move`d int."**
Literal `5` → `f(int&&)`; named `i` → `f(const int&)`; `std::move(i)` → `f(int&&)`. Demonstrates that
the *category of the argument expression*, not its declared type, drives dispatch.

---

## In the order book

When an order moves between data structures (e.g. handed from an input queue into the book), a move
is a pointer swap; a copy might duplicate a buffer. But with **pooled, fixed-size `Order` objects**,
`Order` is all inline fields — so a move of its bytes is identical to a copy. That's precisely why the
design passes **indices/pointers** into the pool rather than relying on element moves: the only way to
get true `O(1)` transfer for an inline-payload type is to move a *pointer to* it, reintroducing the
indirection yourself.

Where moves *do* pay off is the surrounding infrastructure: variable-size input messages, a
`std::vector<Trade>` fill log that grows (needs `noexcept` moves to grow cheaply — Module 9), and
"sink" APIs that take an argument by value and `std::move` it into storage. Understanding move-vs-copy
is what lets you reason about *where a hidden copy would otherwise cost you microseconds* on the
matching path — and articulate it when an interviewer points at a `memcpy` in your flamegraph.

---

## Key takeaways

- Every expression has a **value category** orthogonal to its type: lvalue (identity, persists) vs
  rvalue (temporary, disposable); the finer split is glvalue/xvalue/prvalue.
- Moving = **shallow-copy the handle, invalidate the source** — `O(1)`. It only helps types that own
  their payload through a pointer; for inline/POD data, **move == copy**.
- `std::move` is a **cast**, not an action; it produces an xvalue that grants "steal me" permission
  and compiles to nothing. `std::move` on `const` silently copies.
- `return x;` follows **elide → move → copy**; NRVO often makes it free, so never write
  `return std::move(x);`.
- `const T&` binds to both lvalues and rvalues (and extends a temporary's lifetime) — the safe
  read-only parameter default; `T&&` binds only to rvalues.
- Moved-from objects are **valid but unspecified** — reassign or destroy them, don't read them.

**Next:** [05 — `const`, `constexpr`, const-correctness](05-const-constexpr.md) — promises to the
compiler, and moving work from runtime to compile time.
