# Module 13 — Smart pointers & ownership

A **raw** `new` hands you a pointer and a debt: somewhere, someday, you must `delete` it exactly once
— not zero times (leak), not twice (double-free), and not after someone else already did (use-after-free).
Smart pointers pay that debt for you by encoding *ownership* — "who is responsible for freeing this
object" — directly in the type system, so RAII (Module 7) runs the `delete` automatically at the right
moment. This module is where the abstract lifetime rules of Module 1 become a concrete, everyday design
tool. Ownership clarity is one of the things HFT interviewers probe most directly, because a system that
is fuzzy about who owns what is a system that leaks, races, or crashes under load.

---

## 1. The problem smart pointers solve

Start from the pain. Here is manual heap management:

```cpp
Order* o = new Order(id, px, qty);   // I now owe a delete
if (rejected) return;                // BUG: leaked — returned without delete
process(o);
delete o;                            // finally freed... unless process() threw first
```

Three separate ways to get this wrong, and all of them are silent at compile time:

- **Leak** — a path returns/throws before `delete`. Memory is never reclaimed; the process bloats until
  it's killed. In a long-running trading process this is fatal.
- **Double-free** — two code paths both `delete` the same block. Corrupts the allocator's internal
  free-list; the *next* unrelated allocation crashes, so the stack trace points nowhere near the bug.
- **Use-after-free (dangling)** — you `delete` then dereference. Undefined behavior: sometimes "works",
  sometimes reads garbage, sometimes an attacker's payload.

The fix is not "be more careful." The fix is to make the compiler track ownership. A smart pointer is a
tiny class that *owns* a heap object and, via its destructor (RAII), frees it automatically when the
owner goes out of scope. You stop writing `delete` at all.

The modern C++ ownership vocabulary is exactly three types:

| Type          | Ownership model            | Size            | Runtime cost of copy         |
|---------------|----------------------------|-----------------|------------------------------|
| `unique_ptr`  | exactly **one** owner      | 1 pointer       | not copyable (move only)     |
| `shared_ptr`  | **shared** among N owners  | 2 pointers      | atomic refcount inc/dec      |
| `weak_ptr`    | **observes**, owns nothing  | 2 pointers      | atomic weak-count inc/dec    |

The whole art is picking the weakest one that expresses your actual intent.

---

## 2. `std::unique_ptr` — exclusive ownership (the default)

Exactly one `unique_ptr` owns the object at a time. When it dies, the object dies. It is **zero-overhead**:
the same size as a raw pointer and, with optimizations on, compiles to the same machine code — the
destructor call is inlined into a plain `delete`.

```cpp
#include <memory>
auto p = std::make_unique<Order>(id, price, qty);  // heap Order, owned by p
p->reduce(10);                                       // -> and * work like a raw pointer
// p goes out of scope -> ~unique_ptr runs -> delete on the Order. No leak, no manual delete.

std::unique_ptr<Order> q = std::move(p);   // ownership TRANSFERRED; p is now nullptr
// std::unique_ptr<Order> r = q;           // COMPILE ERROR: copy would mean two owners
```

`unique_ptr` is **move-only** by design. Copying it would create two owners, and two owners means two
`delete`s means a double-free — so the language deletes the copy constructor. Moving is fine: it hands
the pointer over and nulls the source, preserving "exactly one owner" at every instant.

### Memory layout — why it's free

```
unique_ptr<Order> p:        heap
┌──────────────┐        ┌─────────────────┐
│ ptr ─────────┼───────▶│  Order object   │
└──────────────┘        └─────────────────┘
   (one machine word, on the stack)

sizeof(unique_ptr<Order>) == sizeof(Order*) == 8   // on a 64-bit target
```

There is no control block, no counter, nothing beside the pointer. The "smartness" lives entirely in the
type's *destructor*, which runs at compile-time-known points — there is nothing extra at runtime.
`std::move(p)` after the diagram simply copies the 8-byte address into `q` and writes `nullptr` into `p`.

**Always prefer `std::make_unique<T>(args...)` over `unique_ptr<T>(new T(args...))`.** `make_unique` is
exception-safe (Module 14 explains the subtle ordering hazard of a bare `new` inside a function-argument
list), never leaves a raw `new` visible, and reads better.

### Custom deleters — RAII for C resources

A `unique_ptr` can free *anything*, not just heap memory, via a custom deleter. This turns any C-style
"open/close" resource into RAII:

```cpp
auto file = std::unique_ptr<std::FILE, decltype(&std::fclose)>(
    std::fopen("data", "r"), &std::fclose);   // fclose() called automatically on scope exit
```

Note the deleter is part of the type here, so this `unique_ptr` is two words (pointer + function
pointer). Use a *stateless* function-object deleter (an empty struct with `operator()`) if you want to
keep it one word — empty-base optimization removes the deleter's storage entirely.

---

## 3. `std::shared_ptr` — shared ownership (use deliberately)

Sometimes an object genuinely has no single owner: several independent parts of a system all need it
alive, and none of them knows which will be the last to finish with it. `shared_ptr` handles this with
**reference counting** — the object lives until the *last* `shared_ptr` pointing at it is destroyed.

```cpp
auto a = std::make_shared<Order>(...);   // strong refcount = 1
auto b = a;                               // COPY allowed -> strong refcount = 2
// a and b both point at the same Order
b.reset();                                // strong refcount = 1
// a goes out of scope -> strong refcount = 0 -> Order destroyed, memory freed
```

### Memory layout — the control block

This is the single most-asked `shared_ptr` interview point, so picture it exactly. A `shared_ptr` is
**two** pointers: one to the object, one to a separate heap-allocated **control block** that holds the
counts and the deleter.

```
a: shared_ptr<Order>          control block                managed Order
┌──────────────┐        ┌──────────────────────┐      ┌────────────────┐
│ obj  ────────┼───────▶│ strong = 2           │      │  Order fields  │
│ ctrl ────────┼──────┐ │ weak   = 0           │  ┌──▶│                │
└──────────────┘      └▶│ deleter / allocator  │  │   └────────────────┘
                        │ (object ptr)  ───────┼──┘
b: shared_ptr<Order>    └──────────────────────┘
┌──────────────┐              ▲
│ obj  ────────┼──────────────┘  (both b's pointers point at the same object + block)
│ ctrl ────────┼──────────────┘
└──────────────┘

sizeof(shared_ptr<Order>) == 2 * sizeof(void*) == 16
```

- **`strong` count**: how many `shared_ptr`s keep the object alive. Hits 0 → the *object* is destroyed.
- **`weak` count**: how many `weak_ptr`s (plus one held internally while strong > 0) observe it. The
  *control block itself* is freed only when **both** counts reach 0 — which is why a lingering `weak_ptr`
  keeps the (now-empty) control block around even after the object is gone.

`make_shared<Order>(...)` performs **one** allocation that fuses the control block and the object into a
single contiguous block (better cache locality, one `malloc` instead of two):

```
make_shared: one allocation
┌───────────────────────────────────┐
│ strong | weak | deleter │ Order …  │
└───────────────────────────────────┘
```

The trade-off: with `make_shared`, the object's storage is not freed until the *control block* is freed,
i.e. until the last `weak_ptr` also dies. If you have huge objects and long-lived `weak_ptr`s, a plain
`shared_ptr<T>(new T)` (two allocations) can free the object memory sooner. This is a niche point but a
sharp interviewer will ask it.

### Why HFT avoids `shared_ptr` on hot paths

1. **The refcount is atomic.** Every copy and destroy is an atomic increment/decrement (`lock xadd` on
   x86). Atomics are far more expensive than a plain integer op and, worse, when multiple cores touch the
   *same* control block they fight over its cache line (cache-line ping-pong / contention, Module 15).
   Passing `shared_ptr` by value into every function silently sprays atomic ops across your hot loop.
2. **Bigger than a raw pointer** (two words) — doubles the pointer traffic.
3. **Obscures ownership.** "Everyone owns it" usually means "nobody reasoned about its lifetime." That's
   a design smell, not just a performance one.

Rule: reach for `shared_ptr` only when ownership is *genuinely* shared **and** you cannot statically say
who outlives whom. Otherwise it's `unique_ptr` plus non-owning views.

> Even when you do use `shared_ptr`, pass it by `const shared_ptr&` (or pass a raw `T*`/reference to the
> pointee) when a callee only *observes* — that skips the atomic refcount bump. Copy the `shared_ptr`
> only when the callee actually needs to *extend* the object's lifetime.

---

## 4. `std::weak_ptr` — non-owning observer that breaks cycles

`shared_ptr` has one catastrophic failure mode: **reference cycles**. If A holds a `shared_ptr` to B and
B holds a `shared_ptr` back to A, each keeps the other's strong count at ≥1 forever. Neither ever hits 0.
Both leak, silently, even though nothing outside can reach them.

```
   ┌─────────┐  shared (strong=1)  ┌─────────┐
   │    A    │────────────────────▶│    B    │
   │ strong=1│◀────────────────────│ strong=1│
   └─────────┘  shared (strong=1)  └─────────┘
   Outside pointers all gone, yet both counts stay at 1 → leaked forever.
```

`weak_ptr` observes an object *without* contributing to its strong count. Break exactly one direction of
the cycle with a `weak_ptr` (typically the "back" edge — child→parent, observer→subject):

```cpp
struct Node {
    std::shared_ptr<Node> child;   // owns downward
    std::weak_ptr<Node>   parent;  // observes upward — does NOT keep parent alive
};

if (auto p = node.parent.lock()) {   // atomically: if still alive, get a shared_ptr; else empty
    p->doThing();                    // safe: lock() gave us a strong ref for this scope
}                                    // p dies here, strong count drops again
```

You can never dereference a `weak_ptr` directly — because between "is it alive?" and "use it" another
thread could destroy it. `lock()` does both atomically: it returns a `shared_ptr` (bumping the strong
count so the object can't vanish while you hold it) or an empty one if the object is already gone. Use
`expired()` only for a hint; use `lock()` when you actually intend to touch the object.

---

## 5. The ownership decision tree

Walk this top-to-bottom and stop at the first match:

1. **Does it even need the heap?** If the object can live on the stack for a bounded scope, put it there
   (Module 1). No pointer, no ownership question, fastest of all.
2. **Single owner?** → `unique_ptr`. This is the answer ~90% of the time.
3. **Genuinely shared lifetime you can't statically order?** → `shared_ptr`. Justify it out loud.
4. **Observe without owning?** → a **raw pointer or reference** (if the owner provably outlives the
   observer), or a `weak_ptr` (if it might not, e.g. to break a cycle or watch a maybe-dead object).

The key modern idiom: **a raw `T*` or `T&` today means "I observe this, I do not own it."** Owning raw
pointers are the legacy thing to eliminate; non-owning raw pointers are perfectly modern and fast.

```cpp
void inspect(const Order* o);        // non-owning view: "I look, I never free" — correct & idiomatic
std::unique_ptr<Order> owner;        // owning: this is what frees it
```

---

## 6. Common pitfalls & UB

- **Two `shared_ptr`s from the same raw pointer** → two independent control blocks → double-free:
  ```cpp
  Order* raw = new Order(...);
  std::shared_ptr<Order> a(raw);
  std::shared_ptr<Order> b(raw);   // BUG: two control blocks, both will delete raw
  ```
  Fix: create one `shared_ptr` and copy *it*, or use `enable_shared_from_this` when a member needs a
  `shared_ptr` to itself.
- **Use-after-move** on a `unique_ptr`: after `std::move(p)`, `p` is `nullptr`; dereferencing it is a
  null-deref crash. Valid to reassign, invalid to use.
- **Reference cycles** with `shared_ptr` → leak (§4). Break with `weak_ptr`.
- **Returning a raw pointer/reference to a heap object owned by a local `unique_ptr`** → dangling once
  the local's scope ends and the object is freed.
- **Custom-deleter mismatch**: allocate with `malloc`, free with `delete` (or vice-versa) → UB. The
  deleter must match the allocation.
- **Throwing destructor** in an object managed by a smart pointer during stack unwinding → `terminate`
  (Module 14).

---

## 7. Quiz — tough problems

**Q1.** What does this print?
```cpp
auto a = std::make_shared<int>(7);
auto b = a;
{ auto c = a; std::cout << a.use_count() << ' '; }
std::cout << a.use_count();
```
**Answer:** `3 2`. Inside the block `a`, `b`, `c` all share → count 3. When `c` leaves scope the count
drops to 2. (`use_count()` reads the *strong* count.)

**Q2.** Does this compile? If not, why?
```cpp
std::unique_ptr<Order> f();
std::vector<std::unique_ptr<Order>> v;
v.push_back(f());
auto p = v[0];
```
**Answer:** The `push_back(f())` is fine — `f()` returns a prvalue (rvalue), which moves in. But
`auto p = v[0];` fails to compile: `v[0]` is an lvalue and `unique_ptr` is not copyable. You must write
`auto p = std::move(v[0]);` (which empties the vector slot) or take a reference `auto& p = v[0];`.

**Q3.** Spot the bug and its symptom:
```cpp
void handle(Order* raw) {
    std::shared_ptr<Order> guard(raw);   // "make it safe"
    process(*raw);
}
Order o; handle(&o);
```
**Answer:** `guard` is constructed from a pointer to a **stack** object. When `guard`'s count hits 0 it
calls `delete` on `&o` — but `o` was never `new`ed. Deleting a non-heap pointer is UB (heap corruption /
crash). Smart pointers must only ever own heap allocations.

**Q4.** How many times is the `Order` destroyed?
```cpp
Order* raw = new Order(...);
std::shared_ptr<Order> a(raw);
std::shared_ptr<Order> b(raw);
```
**Answer:** Twice → double-free → UB. Each `shared_ptr` built directly from `raw` gets its *own* control
block with its own count of 1, so each independently deletes `raw` at destruction.

**Q5.** Why does this leak, and what's the one-word fix?
```cpp
struct N { std::shared_ptr<N> next; };
auto a = std::make_shared<N>();
auto b = std::make_shared<N>();
a->next = b;  b->next = a;
```
**Answer:** A cycle: `a` and `b` keep each other's strong count at ≥1 forever, so neither reaches 0 when
the locals die. Fix: make one of the links a `weak_ptr`.

**Q6.** True or false: passing `shared_ptr<Order>` **by value** into a hot-loop function is free because
"it's just a pointer." **Answer:** False. By-value copy bumps the atomic strong count on entry and
decrements it on exit — two atomic RMW ops per call, plus cache-line contention if other cores touch the
same control block. Pass `const Order&` or a raw `const Order*` to observe.

**Q7.** After `make_shared`, when is the object's *memory* actually released?
```cpp
std::weak_ptr<Order> w;
{ auto s = std::make_shared<Order>(...); w = s; }   // s dies here
// ... w still in scope, expired
```
**Answer:** The `Order`'s **destructor** runs when `s` dies (strong count 0). But because `make_shared`
fused the object and control block into one allocation, the *memory* is not freed until the control block
is freed — i.e. until `w` (weak count) is also gone. The destructor and the deallocation happen at
different times here.

**Q8.** Is `sizeof(std::unique_ptr<T>)` always `sizeof(T*)`?
**Answer:** Only with the default (stateless) deleter. With a stateful deleter (e.g. a function pointer
like `&std::fclose`), the `unique_ptr` also stores the deleter, so it grows. Empty/stateless function-
object deleters are removed by empty-base optimization and keep it one word.

**Q9.** What's wrong here, and what would you use instead?
```cpp
class Session : /*...*/ {
    void arm() { registry.add(std::shared_ptr<Session>(this)); }  // hand out a shared_ptr to myself
};
```
**Answer:** Building a `shared_ptr` from `this` creates a *second* control block over an object that is
already (presumably) owned elsewhere → double-free. Correct approach: derive from
`std::enable_shared_from_this<Session>` and call `shared_from_this()`, which returns a `shared_ptr` tied
to the *existing* control block.

---

## 8. Indian HFT interview questions

Questions in this shape come up at Tower Research (Gurgaon), Optiver, IMC, Graviton, Quadeye, AlphaGrep,
WorldQuant, Da Vinci, Squarepoint, Jump, Hudson River, NK Securities, Mansard, and Qube desks.

**"unique_ptr vs shared_ptr — which do you default to and why?"**
Default to `unique_ptr`: exclusive ownership, zero overhead (same size and code as a raw pointer,
move-only so ownership is unambiguous). Reach for `shared_ptr` only when ownership is truly shared and
lifetime can't be statically ordered — it carries an atomic refcount and a control block, so it's slower
and it muddies who owns what. "Shared by default" is a red flag.

**"What exactly is inside a shared_ptr / what is the control block?"**
A `shared_ptr` is two pointers: one to the object, one to a heap control block holding the atomic
**strong** count, the atomic **weak** count, and the deleter/allocator. Object dies at strong 0; control
block dies at strong 0 **and** weak 0. Draw the diagram from §3.

**"make_shared vs shared_ptr<T>(new T) — difference?"**
`make_shared` does **one** allocation fusing object + control block: fewer `malloc`s, better locality,
and it's exception-safe. Downside: the object's memory isn't reclaimed until the last `weak_ptr` also
dies (fused block). `shared_ptr<T>(new T)` does two allocations but frees the object as soon as the strong
count hits 0. Prefer `make_shared` unless you have big objects with long-lived `weak_ptr`s.

**"Why is shared_ptr's refcount atomic, and why does that hurt in HFT?"**
Because copies/destroys may happen on different threads, the count must be race-free → atomic RMW. On the
hot path that means real cost per copy (`lock xadd`) plus cache-line contention when several cores touch
the same control block (Module 15 false sharing). So we avoid `shared_ptr` in the matching loop entirely.

**"How do you break a reference cycle?"**
Make the back-edge a `weak_ptr` so it observes without contributing to the strong count. Use `lock()` to
promote it to a temporary `shared_ptr` when you actually need to touch the object.

**"When is a raw pointer acceptable in modern C++?"**
When it's **non-owning** — a pure observer/view whose target is guaranteed (by design) to outlive it. A
raw `T*` today means "I look, I don't free." Owning raw pointers are what smart pointers replace.

**"You have millions of Orders on the hot path — do you use unique_ptr per order?"**
No. Even `unique_ptr`'s per-object heap allocation is too expensive and non-deterministic on the hot path.
Use an **object pool** (Module 3): pre-allocate a contiguous slab up front, hand out **indices or
non-owning raw pointers** into it, and never allocate in the matching loop. Smart pointers own the
*infrastructure* (the pool, config, sessions), not individual orders.

**"shared_ptr passed by value vs by const-ref into a function?"**
By value → atomic bump/drop per call (needed only if the callee extends lifetime). By `const&` (or a raw
pointer/reference to the pointee) → no refcount traffic, correct when the callee only observes. On a hot
path this is the difference between two atomics per call and zero.

---

## 9. In the order book

- Orders come from an **object pool** (Module 3), not individual `make_unique` — even `unique_ptr`'s
  per-object allocation and the non-determinism of the allocator are unacceptable on the matching path.
  The pool owns the contiguous storage for its whole lifetime; the book holds **non-owning raw pointers
  or 32-bit indices** into it (indices are half the size and survive pool reallocation — Module 15).
- `unique_ptr` / `shared_ptr` live in the *cold* infrastructure: the object owning the pool itself, the
  config loaded at startup, the network session, the logging sink. There, clarity beats nanoseconds, so
  smart pointers are exactly right.
- The HFT-specific insight interviewers want to hear: smart pointers are the correct default in general
  code, but the hot path deliberately drops one level lower to pooled storage + non-owning views. Knowing
  *when to drop below smart pointers* — and being able to justify it with the atomic-refcount and
  allocation-determinism arguments — is what separates a systems answer from a textbook answer.

---

## Key takeaways

- Smart pointers encode **ownership** in the type system so RAII frees objects automatically — you stop
  writing `delete` and stop leaking/double-freeing.
- `unique_ptr` = exclusive, move-only, **zero-overhead** (one pointer). The default for heap ownership;
  always create with `make_unique`.
- `shared_ptr` = shared, **atomic refcount + control block** (two pointers). Correct when ownership is
  genuinely shared and unorderable; costly and ownership-obscuring otherwise. Prefer `make_shared`.
- `weak_ptr` observes without owning and is the tool for **breaking `shared_ptr` cycles**; access via
  `lock()`.
- A **raw `T*`/`T&` is fine when non-owning** — that's the modern convention, not a code smell.
- HFT hot paths go *below* smart pointers to **object pools + indices/raw views**, because per-object
  allocation and atomic refcounts are too slow and non-deterministic; smart pointers stay in the cold
  infrastructure.

**Next:** [14 — Exceptions, `noexcept`, error handling without exceptions](14-exceptions-errors.md) — how
to signal failure, exception-safety guarantees, and why the hot path uses status codes instead.
