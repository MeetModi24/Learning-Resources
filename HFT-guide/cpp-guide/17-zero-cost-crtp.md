# Module 17 — Zero-cost abstraction, CRTP, branch elimination

The C++ promise, in one sentence: *"you don't pay for what you don't use, and what you do use, you
couldn't hand-code better."* This module is the toolkit for **abstraction without runtime cost** —
the thing that lets HFT code be both clean *and* fast. Everywhere else in this guide you learned
what the language gives you; here you learn how to keep the compiler from adding overhead you never
asked for.

---

## 1. What "zero-cost abstraction" actually means

A *zero-cost abstraction* is a high-level construct that compiles down to the same machine code you
would have written by hand in low-level code. The convenience exists only at compile time; by the
time the CPU runs your program, the abstraction has evaporated.

Concrete examples you already use:

- A range-`for` loop (`for (auto& x : v)`) compiles to the same instructions as a hand-written
  index loop (`for (size_t i = 0; ...)`). The nicer syntax costs nothing.
- A lambda passed to `std::sort` gets **inlined** — the comparison becomes part of the sort's inner
  loop, exactly as if you'd hardcoded it. A C `qsort` with a function-pointer comparator *cannot*
  inline and is measurably slower.
- `std::unique_ptr<T>` is the same size (one pointer) and the same speed as a raw `T*`, but adds
  automatic cleanup. The safety is free.

Contrast that with a **non**-zero-cost abstraction: a virtual function call (Module 11) forces the
CPU to load a function address from a vtable at runtime and jump to it — an *indirect call* the
compiler usually can't inline or predict. `std::function` is worse: it may heap-allocate and it
always dispatches through a type-erased indirection.

The enemies of zero-cost are all forms of **runtime indirection**:

1. **Virtual calls** — the vtable jump (Module 11).
2. **`std::function`** — type erasure + possible allocation.
3. **Unpredictable branches** — the CPU guesses which way an `if` goes; a wrong guess flushes the
   pipeline (~15–20 cycles on a modern core).

This module removes all three. The recurring idea: **move the decision from runtime to compile
time.** If the compiler knows the answer while compiling, it bakes in the fast path and deletes the
machinery for the others.

---

## 2. Static vs dynamic dispatch — what it costs in memory and cycles

Before CRTP makes sense, you need to *see* the difference between the two kinds of dispatch.

**Dynamic dispatch (virtual):** the object carries a hidden pointer (the *vptr*) to a per-class
table of function addresses (the *vtable*). A virtual call reads the vptr, indexes into the vtable,
loads the target address, and does an indirect jump:

```
Object with virtual functions            The call `s->onTick()` at runtime:
┌──────────────┐                          1. load vptr        (from object)
│ vptr ────────┼──▶ ┌─────────────────┐   2. load slot        (vtable[k])
│ side         │    │ &Base::onTick    │   3. indirect jump    (can't inline,
│ price        │    │ &Base::onFill    │      branch predictor must guess target)
│ qty          │    │ ...              │
└──────────────┘    └─────────────────┘   +8 bytes per object, +1 indirection per call
```

**Static dispatch (CRTP / templates):** there is no vptr and no table. The exact function is chosen
by the compiler and its body is pasted directly into the call site:

```
Object with CRTP (no virtual)             The call `s.run()` at compile time:
┌──────────────┐                          run() body is inlined; onTick() body is
│ side         │   (no vptr — object is   inlined into it. Final machine code = the
│ price        │    smaller)              onTick() instructions, no jump at all.
│ qty          │
└──────────────┘                          0 bytes overhead, 0 indirections
```

So dynamic dispatch costs **8 bytes per object** (the vptr) plus **one indirect call per
invocation** that the compiler cannot inline and the branch predictor may mispredict. On a hot path
executed millions of times per second, those add up — and worse, they *block inlining*, which
prevents dozens of downstream optimizations (constant folding, dead-code elimination, keeping
values in registers). The real cost of a virtual call is rarely the jump itself; it's the
optimization wall it erects.

---

## 3. CRTP — the Curiously Recurring Template Pattern

CRTP gives you the *shape* of inheritance-based polymorphism — a base class defining a common
interface, derived classes providing behavior — but resolves everything at **compile time**, so
there's no vtable, no indirect call, and full inlining.

The trick is in the name: a class inherits from a template of the base, **passing itself** as the
template argument.

```cpp
template <typename Derived>
class Strategy {
public:
    void run() {
        // Dispatch to the derived class WITHOUT a virtual call:
        static_cast<Derived*>(this)->onTick();
    }
};

class MomentumStrategy : public Strategy<MomentumStrategy> {  // <-- passes itself
public:
    void onTick() { /* ... */ }   // note: no 'virtual', no 'override'
};

MomentumStrategy s;
s.run();   // Strategy::run inlines the static_cast + onTick — fully resolved at compile time
```

**How it works, step by step.** When you write `Strategy<MomentumStrategy>`, the compiler knows the
concrete derived type *while compiling the base*. Inside `run()`, `this` has static type
`Strategy<MomentumStrategy>*`. The `static_cast<Derived*>(this)` — where `Derived` *is*
`MomentumStrategy` — is a compile-time cast to the real type, so the following `->onTick()` binds to
`MomentumStrategy::onTick` directly. No lookup, no table. Because the compiler can see the whole
chain, it inlines `onTick()` into `run()` and `run()` into the caller. The abstraction is gone by
the time the CPU sees it.

Compare the two approaches head-on:

| | Virtual functions | CRTP |
|-|-------------------|------|
| Dispatch | runtime (vtable) | compile time |
| Inlinable | no (indirection wall) | yes (fully) |
| Object overhead | +8 bytes (vptr) | none |
| Flexibility | new types can appear at **runtime** (plugins) | types fixed at **compile time** |
| Binary size | one function, shared | one instantiation **per type** (code bloat) |
| Error messages | clean | template-verbose |
| Use when | cold path, unknown/plugin types, heterogeneous containers | hot path, small fixed set of known types |

**Uses of CRTP:** static interfaces (as above), *mixins* (bolt reusable behavior onto many
unrelated classes), and *expression templates* (used by linear-algebra libraries like Eigen to fuse
`a + b + c` into a single loop with no temporaries). 

**Costs, stated honestly:** types must be known at compile time — you *cannot* load a new strategy
from a shared library at runtime the way virtual functions allow. You get one full copy of the
templated code per instantiating type (binary bloat, i-cache pressure if overused). And the syntax
is harder to read and produces worse compiler errors. Use CRTP where the type set is small, fixed,
and on the hot path — not as a blanket replacement for inheritance.

---

## 4. Branch elimination via templates

A branch that is checked millions of times but whose answer never changes within a run is pure
waste. If a decision is known at compile time, lift it there and the branch disappears entirely.

The order-book "which side?" check is the textbook example. A buy order crosses when its price is
≥ the resting ask; a sell crosses when its price is ≤ the resting bid. Naively:

```cpp
enum class Side { Buy, Sell };

// Runtime branch — evaluated on EVERY call, even though a given book side never changes:
bool crosses(Side s, std::int64_t restPrice, std::int64_t incoming) {
    return (s == Side::Buy) ? incoming <= restPrice : incoming >= restPrice;
}
```

Make `Side` a **template parameter** instead of a runtime argument, and use `if constexpr`
(Module 10) to discard the untaken branch at compile time:

```cpp
// Compile-time — the branch DISAPPEARS; the compiler generates two specialized functions:
template <Side S>
bool crosses(std::int64_t restPrice, std::int64_t incoming) {
    if constexpr (S == Side::Buy) return incoming <= restPrice;
    else                          return incoming >= restPrice;
}

crosses<Side::Buy>(rest, in);   // compiles to just `return in <= rest;` — no branch
```

`if constexpr` is not a runtime `if` — the compiler *deletes* the false branch's code, so
`crosses<Side::Buy>` contains only the buy comparison. You've traded one runtime branch (and a
possible misprediction) for two tiny specialized functions chosen at compile time. This is exactly
the "side-specialized templates (compile-time branch elimination)" optimization referenced in the
research doc — a real, citable technique, not folklore.

The cost, again, is code duplication: you now have two versions of the function. For a hot inner
loop that runs billions of times, that's an excellent trade. For a cold function, it's needless
bloat.

---

## 5. `[[likely]]` / `[[unlikely]]` — hinting the branches you can't remove

Sometimes a branch genuinely must stay at runtime (you can't know at compile time whether an order
will be rejected). C++20 lets you tell the compiler which way it *usually* goes, so it lays the hot
path out as a straight line — keeping the common case in contiguous, cache-warm instructions and
pushing the rare case out of the way:

```cpp
if (order.qty > 0) [[likely]] { match(order); }
else               [[unlikely]] { reject(order); }
```

Why this matters at the hardware level: the CPU fetches instructions ahead of time and predicts
branches. If the hot path is laid out linearly, the fetcher streams straight through it (fewer
i-cache misses) and the predictor's default guess is right. The rare `reject` path can live far
away in memory; taking it occasionally is fine because it's rare by construction. These attributes
change *code layout*, not correctness — measure to confirm they help, because modern predictors are
already very good.

---

## 6. `inline`, `always_inline`, and `constexpr`

Three related tools for shrinking or eliminating call overhead:

- **`inline`** today mostly means "this definition may appear in multiple translation units without
  violating the One Definition Rule." It is only a *hint* to actually inline the body — the
  optimizer makes the final call based on size and hotness.
- **Force inlining** (rarely, and only after measuring) with `[[gnu::always_inline]]` /
  `__attribute__((always_inline))` for tiny, hot functions the compiler is being too conservative
  about. Overusing this bloats the binary and can *hurt* by evicting other hot code from i-cache.
- **`constexpr`** (Module 5) is the ultimate zero-cost: it moves computation to *compile time*
  entirely. A `constexpr` lookup table or mask is computed once by the compiler and baked into the
  binary as data — the runtime cost is zero because the work already happened.

---

## 7. Tag dispatch and policy-based design

Instead of a runtime flag selecting behavior, encode the choice in a **type** and let overload
resolution or a template parameter pick the implementation at compile time.

*Policy-based design* is the STL's allocator model: `std::vector<T, MyAllocator>` swaps its
allocation strategy with zero runtime cost, because the allocator is a template parameter baked in
at compile time — no branch, no indirection. In HFT you might template a book on its allocation or
matching policy:

```cpp
template <typename MatchPolicy>
class Book { /* MatchPolicy::match(...) is chosen at compile time, inlined */ };
```

*Tag dispatch* uses empty "tag" types to select among overloads (the way `std::advance` picks a
different implementation for random-access vs forward iterators). Both techniques share the theme:
the decision is a type, so the compiler resolves it and there is nothing left to decide at runtime.

---

## 8. Common pitfalls & gotchas

- **Assuming CRTP always inlines.** It only inlines if the definitions are *visible* at the call
  site (typically in headers) and the optimizer chooses to. Split the derived method into a `.cpp`
  and you may lose the win.
- **Overusing CRTP/templates.** Every instantiation is a fresh copy of the code. Template a rarely
  called function on a dozen types and you bloat the binary and hammer the i-cache — the opposite of
  fast. Reserve these for genuinely hot, small-type-set paths.
- **Believing `inline` forces inlining.** It doesn't — it's an ODR/linkage keyword and a weak hint.
- **`if constexpr` with a runtime value.** The condition must be a compile-time constant. `if
  constexpr (someRuntimeBool)` won't compile as branch elimination.
- **Slapping `always_inline` everywhere.** Forcing inlining of a large function into many call sites
  can *increase* total instruction footprint and slow you down. Measure.
- **`[[likely]]` as a correctness tool.** It's purely a layout hint; a wrong hint just costs a
  little performance, never correctness — but a wrong hint on a truly 50/50 branch can hurt.
- **Forgetting virtual is sometimes right.** If you need runtime plugin loading or a heterogeneous
  container of unrelated types, virtual dispatch is the correct tool — CRTP can't do it. Zero-cost
  is a goal on the hot path, not a religion.

---

## 9. Quiz — tough problems

**Q1.** What is the size difference (`sizeof`) between an object of a class with one virtual
function and an otherwise identical CRTP-based class, on a 64-bit platform, and why?

**Answer:** The virtual class is **8 bytes larger** — it carries a hidden `vptr` (pointer to its
vtable). The CRTP class stores no per-object dispatch data; the type information lives entirely in
the template instantiation at compile time, so there's zero per-object overhead.

---

**Q2.** In `Strategy<Derived>::run()`, why is `static_cast<Derived*>(this)` safe here even though a
downcast is normally dangerous? When would it be undefined behavior?

**Answer:** It's safe *by construction*: CRTP guarantees the base `Strategy<MomentumStrategy>` is
only ever a base subobject of `MomentumStrategy`, so `this` really does point into a `Derived`. The
`static_cast` therefore always downcasts to the true dynamic type. It becomes UB if you violate the
CRTP contract — e.g. `class Bug : public Strategy<MomentumStrategy>` (passing the *wrong* type as
`Derived`), because then `this` isn't a `MomentumStrategy` and the cast lies.

---

**Q3.** Predict the machine-code difference between these two calls in a hot loop and explain which
optimizations the virtual version blocks:

```cpp
base->onTick();          // (A) virtual
crtpStrategy.run();       // (B) CRTP
```

**Answer:** (A) emits: load vptr → load vtable slot → indirect `call`. The indirect call is an
optimization wall — the compiler can't inline `onTick`, can't propagate constants into it, can't
eliminate dead code across the boundary, and the branch predictor must guess the target. (B) inlines
`onTick`'s body directly into the loop; the compiler then folds constants, hoists invariants, and
keeps values in registers across what used to be a call boundary. The inlining, not the saved jump,
is the big win.

---

**Q4.** Rewrite this runtime branch using compile-time dispatch so the branch is eliminated, and
state the cost:

```cpp
double fee(bool isMaker, double notional) {
    return isMaker ? notional * 0.0001 : notional * 0.0003;
}
```

**Answer:**

```cpp
template <bool IsMaker>
double fee(double notional) {
    if constexpr (IsMaker) return notional * 0.0001;
    else                   return notional * 0.0003;
}
```

Each instantiation contains only one multiply, no branch. **Cost:** two functions instead of one
(minor code duplication), and `IsMaker` must be known at compile time — you can't use this if the
maker/taker status is only known at runtime.

---

**Q5.** True or false: `std::function<void(Order&)>` is a zero-cost abstraction for a callback.
Justify, and give the zero-cost alternative.

**Answer:** **False.** `std::function` is type-erased: it dispatches through an indirection (like a
virtual call) and may heap-allocate if the callable doesn't fit its small-buffer optimization. The
zero-cost alternative is to make the callback a **template parameter** (`template <typename Fn> void
onTrade(Fn&& cb)`) or a lambda passed by value/forwarding reference, so the compiler knows the exact
callable and inlines it.

---

**Q6.** Your CRTP base has a method `run()` that calls `onTick()`, but a derived class *forgot* to
define `onTick()`. When and how do you find out? Compare with the virtual (pure-virtual) equivalent.

**Answer:** With CRTP you find out at **compile time**, but only when `run()` is instantiated for
that derived type — a "no member named `onTick`" error, sometimes with a confusing template
backtrace. With a `virtual void onTick() = 0;` pure virtual, forgetting to override it makes the
derived class abstract, so you get a clean, immediate "cannot instantiate abstract class" error at
the point of construction.

---

**Q7.** Why can overusing `[[gnu::always_inline]]` on a large, frequently-called function *reduce*
performance, even though inlining is supposed to be faster?

**Answer:** Inlining copies the function body into every call site. For a large function called from
many places, this multiplies its instruction footprint, blowing the L1 instruction cache. i-cache
misses stall the front-end of the CPU, and the aggregate slowdown can exceed the call overhead you
saved. Inlining is a size-for-speed trade; past a point the size cost dominates.

---

## 10. Indian HFT interview questions

**Q (Tower Research, Gurgaon): "Explain how you'd get polymorphism without a vtable, and when you'd
still choose virtual."**
Use CRTP: the base is a template parameterized on the derived type, and the base's methods
`static_cast<Derived*>(this)` to call into the derived class — resolved and inlined at compile time,
zero per-object overhead. I'd still use `virtual` when the concrete type is only known at runtime
(plugin strategies loaded from config, a heterogeneous container of order types), or on cold paths
where the indirect-call cost is irrelevant and clean errors/readability matter more.

**Q (Optiver): "What's the real cost of a virtual function call on the hot path — is it the jump?"**
The indirect jump itself is a few cycles, and often correctly predicted. The *real* cost is that it
blocks inlining: the compiler can't fold constants, hoist invariants, or eliminate dead code across
the call boundary, and it can't keep values in registers. On a matching loop that's the difference
between a tight register-resident inner loop and a series of memory round-trips. Plus the +8-byte
vptr hurts cache density if you have millions of objects.

**Q (IMC): "You have a `Side` that's fixed for the lifetime of a book. How do you remove the
buy/sell branch from the matching loop?"**
Template the book (or the comparison) on `Side` and use `if constexpr` to select the comparison.
`FastBook<Side::Buy>` compiles the loop with only `incoming <= restPrice`; the branch is gone
entirely. Cost is two instantiations of the matching code — worth it on the hottest loop in the
system.

**Q (Graviton / Quadeye): "What does 'zero-cost abstraction' mean, and name one abstraction in the
standard library that is NOT zero-cost."**
Zero-cost means the high-level construct compiles to the same code you'd write by hand — e.g.
`unique_ptr` is as fast as a raw pointer, range-`for` matches an index loop. `std::function` is
*not* zero-cost: it type-erases the callable (an indirection you can't inline) and may heap-allocate.
`std::shared_ptr`'s atomic refcount is also a non-trivial cost.

**Q (AlphaGrep): "How do you pass a callback into a hot function so it inlines?"**
Make it a template parameter or a forwarded lambda: `template <typename Fn> void
forEachFill(Fn&& cb)`. The compiler sees the exact callable and inlines it into the loop. Avoid
`std::function` on the hot path precisely because it defeats inlining.

**Q (WorldQuant / Da Vinci): "CRTP causes code bloat — explain, and when does that actually hurt?"**
Each distinct `Derived` produces a full copy of the templated base code (one per instantiation).
That grows the binary. It hurts when the code is large *and* many types instantiate it *and* it's
called from cache-sensitive paths — the copies compete for L1 instruction cache. For a small fixed
set of hot strategies it's a non-issue; for a wide, cold API it's the wrong tool.

**Q (Jump / HRT): "How would you profile whether your 'zero-cost' abstraction is actually zero
cost?"**
Compile with optimizations, then compare: (1) disassemble with `objdump`/Compiler Explorer to check
the abstraction inlined and no indirect call remains; (2) `perf stat` for branch-misprediction and
i-cache-miss counters; (3) a microbenchmark against the hand-written baseline. If the assembly and
the counters match the manual version, it's genuinely zero-cost — never assume, verify.

---

## 11. In the order book

- **`BookSide<Side::Buy>` / `BookSide<Side::Sell>`** — side is a template parameter, so the buy/sell
  price comparison in the matching loop is eliminated at compile time (Section 4). The hottest loop
  in the engine carries no side branch.
- **Callbacks as templates/lambdas, never `std::function`.** Trade-reporting callbacks are passed as
  template parameters so they inline into the matching loop (Section 6), keeping the fill path free
  of type-erased indirection.
- **CRTP for strategy interfaces** where the strategy set is fixed and hot — the `onTick`/`onFill`
  hooks resolve statically and inline, avoiding the vtable on every market update.
- **`[[likely]]`** on the common "order rests / partially fills" branch and **`[[unlikely]]`** on
  rejects, so the fill path is laid out linearly and stays i-cache-warm.
- These are small but genuine, defensible optimizations. In an interview, being able to say *"I
  templated the book on side to delete the comparison branch, and I kept `std::function` off the hot
  path so callbacks inline"* signals real cost-awareness — far stronger than "I used C++20."

---

## Key takeaways

- **Zero-cost abstraction** = the convenience exists only at compile time; the machine code equals
  the hand-written version. Its enemies are virtual calls, `std::function`, and unpredictable
  branches — all forms of runtime indirection.
- **Dynamic dispatch** costs +8 bytes per object (vptr) and, more importantly, an indirect call that
  **blocks inlining** and all the optimizations inlining unlocks.
- **CRTP** gives inheritance-shaped polymorphism resolved at compile time — no vtable, no
  indirection, full inlining — by having a class inherit from `Base<Itself>`. Cost: fixed types and
  code bloat.
- **Compile-time branch elimination** (template parameter + `if constexpr`) deletes branches whose
  answer is fixed per run, e.g. the order-book side comparison.
- **`[[likely]]`/`[[unlikely]]`** lay out the branches you can't remove; **`constexpr`** moves work
  to compile time entirely; **`inline`** is an ODR/linkage keyword and only a hint.
- Use these on hot, small-type-set paths — **not everywhere.** Premature templating bloats the
  binary, hurts i-cache, and wrecks readability. Measure before and after.

**Next:** [18 — C++20 features that matter](18-cpp20-features.md) — concepts, `<bit>`, `std::span`,
ranges, and the defensible "what I actually used" story.
