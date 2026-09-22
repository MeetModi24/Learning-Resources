# Module 10 — Templates → Concepts

Templates are C++'s **compile-time generics**: you write the code once with a placeholder type, and
the compiler stamps out a fully specialized, concrete version for every type you actually use — with
**zero runtime overhead**. No boxing, no tagged unions, no virtual dispatch. This is the backbone of
"zero-cost abstraction" (Module 17) and of the entire STL, and it is *the* mechanism HFT code leans
on to get generic, reusable components that still compile down to the same machine code you'd have
written by hand.

This module builds the idea from the ground up — what "the compiler stamps out a version" actually
*means* in terms of the binary — then climbs to concepts, `if constexpr`, and the interview-grade
subtleties (two-phase lookup, code bloat, monomorphization vs. type erasure).

---

## 1. The core idea: monomorphization

Start with the mental model, because everything else follows from it.

In some languages (Java, C#) a "generic" is *one* piece of compiled code that works on many types by
boxing everything behind a common base (`Object`) and dispatching at runtime. That has a cost: an
indirection on every operation, and heap boxing for value types.

C++ templates do the opposite. A template is **not code** — it is a *recipe for generating code*.
When you write:

```cpp
template <typename T>
T max(T a, T b) { return (a < b) ? b : a; }

max(3, 7);        // T = int    — compiler generates a separate max<int>
max(2.5, 1.5);    // T = double — compiler generates a separate max<double>
```

the compiler performs **instantiation**: it substitutes `T = int`, produces a real function
`max<int>(int, int)`, and separately produces `max<double>(double, double)`. Each is exactly as fast
as if you'd hand-written that overload — the `<` is a plain integer/double compare, fully inlinable.
This process of "generate one concrete copy per type" is called **monomorphization**.

```
Source (one recipe)                 Binary (many concrete functions)
─────────────────────               ──────────────────────────────────
template<T> T max(T,T)     ──►       max<int>(int,int):    cmp / cmov ...
                           ──►       max<double>(dbl,dbl): comisd / ...
                           ──►       max<Order>(...):      generated on demand
```

The consequence you must be able to state in an interview: **the abstraction exists only in the
source; in the binary there is no generic `max`, only the specialized copies.** That is why there is
zero runtime cost — and also why templates cause *code bloat* (many copies) and slow compiles
(Section 8).

---

## 2. Function templates

```cpp
template <typename T>            // 'typename T' introduces a type parameter
T max(T a, T b) { return (a < b) ? b : a; }
```

- `template <typename T>` is the parameter list. `class` and `typename` are interchangeable here
  (`template <class T>` is identical); prefer `typename`.
- The compiler **deduces** `T` from the arguments at the call site — you rarely write `max<int>(...)`
  explicitly. `max(3, 7)` deduces `T = int`.

**Template argument deduction** is where beginners get surprised. A subtlety worth memorising:

```cpp
max(3, 7.0);   // ERROR: T deduced as int from 3 AND double from 7.0 — conflict, won't compile
```

Both parameters are the *same* `T`, so both arguments must deduce to the same type. Fixes: cast one
argument, or use two type parameters `template <typename A, typename B>`, or force it explicitly
`max<double>(3, 7.0)` (now `3` converts to `double`).

Deduction also **strips** top-level `const` and references unless you ask for them. `template<class T>
void f(T x)` called with a `const int&` deduces `T = int` (a copy). To preserve them you take
`const T&` or `T&&` (forwarding reference — Module 9).

---

## 3. Class templates and non-type template parameters

Templates aren't just functions. A **class template** parameterises a whole type:

```cpp
template <typename T, std::size_t N>   // N is a NON-TYPE template parameter (a value, not a type)
class RingBuffer {
    T data_[N];                        // fixed inline array — no heap!
    std::size_t head_ = 0, tail_ = 0;
public:
    bool push(const T& v) { /* ... */ return true; }
    bool pop(T& out)      { /* ... */ return true; }
};

RingBuffer<Order, 1024> queue;   // T = Order, N = 1024 — capacity baked into the type
```

Unlike function templates, class templates historically could **not** deduce their arguments — you
wrote `RingBuffer<Order, 1024>` explicitly. (C++17 added CTAD — class template argument deduction —
so `std::vector v{1,2,3}` deduces `std::vector<int>`, but for non-type params like `N` you still
specify them.)

**Non-type template parameters (NTTPs)** are a quiet superpower for HFT. Because `N` is a
compile-time constant:

- `data_[N]` is a fixed inline array — it lives *inside* the object (on the stack, or inline in a
  pooled slot), **no heap allocation, no runtime size field.**
- `N` can be used in `constexpr` arithmetic: masks (`& (N-1)` when `N` is a power of two — Module 18),
  compile-time bounds checks, loop unrolling hints.
- Two `RingBuffer<Order,1024>` and `RingBuffer<Order,2048>` are **different types** — you cannot
  accidentally mix them. The size is part of the type identity.

This is exactly how fixed-capacity lock-free queues (Module 16) are built: `SpscQueue<Message, 4096>`.

```
RingBuffer<Order,4> object in memory (no heap, all inline):
┌──────────────────────────────────────────────┐
│ data_[0] │ data_[1] │ data_[2] │ data_[3] │ head_ │ tail_ │
└──────────────────────────────────────────────┘
        one contiguous block, size fixed at compile time
```

---

## 4. Template specialization

Sometimes the generic recipe is wrong or suboptimal for one specific type. **Specialization** lets
you provide a custom implementation:

```cpp
template <typename T> struct Serializer {          // primary (generic) template
    static void write(const T& v) { /* generic byte copy */ }
};

template <> struct Serializer<Order> {             // FULL specialization for Order
    static void write(const Order& o) { /* hand-optimized packing */ }
};
```

`template <>` with the concrete type in `<...>` = **full (explicit) specialization**: "when `T` is
exactly `Order`, use this instead."

Class templates also allow **partial specialization** — specialize for a *family* of types:

```cpp
template <typename T> struct Serializer<T*> {      // partial: any pointer type
    static void write(T* p) { /* serialize what p points to */ }
};
```

This is the classic compile-time "select the best algorithm per type" tool — the STL uses it heavily
(e.g. `std::is_pointer`, iterator-category dispatch). Note: **function templates cannot be partially
specialized** — you use overloading (or `if constexpr`, Section 7) instead.

---

## 5. The pre-C++20 pain: duck typing and awful errors

Before concepts, templates were **duck-typed at compile time**: a template silently required whatever
operations its body used (`a < b` requires `operator<`), but that requirement was *implicit*. If you
instantiated `max<Widget>` and `Widget` had no `operator<`, the error appeared *deep inside* the
template body, often as a wall of 200 lines mentioning internal names you never wrote.

The old way to constrain templates was **SFINAE** ("Substitution Failure Is Not An Error") — abusing
`std::enable_if` so that ill-formed substitutions quietly removed an overload from consideration:

```cpp
// The old, ugly way to say "only for integral T":
template <typename T,
          typename = std::enable_if_t<std::is_integral_v<T>>>
T next(T x) { return x + 1; }
```

It worked, but it was unreadable, error messages were still bad, and the intent ("T must be
integral") was buried in boilerplate. Concepts replace all of this.

---

## 6. Concepts (C++20) — named, checkable constraints

A **concept** is a named, compile-time predicate on types — a Boolean you evaluate at compile time
that says "does type `T` support these operations?" It both *documents* and *enforces* what a template
requires:

```cpp
#include <concepts>

template <typename T>
concept Orderable = requires(T a, T b) {
    { a < b }  -> std::convertible_to<bool>;   // T must support <, result convertible to bool
    { a == b } -> std::convertible_to<bool>;   // and ==
};

template <Orderable T>          // constrain the template with the concept
T max(T a, T b) { return (a < b) ? b : a; }

max(3, 7);                 // OK: int satisfies Orderable
// max(Widget{}, Widget{}) // if Widget has no <, error: "Widget does not satisfy Orderable"
```

The `requires` block lists expressions that must be valid; `{ expr } -> Concept` also checks the
expression's *result type*. Three concrete wins:

- **Readable errors**: "`T` does not satisfy `Orderable`" instead of a nested instantiation dump.
- **Self-documenting signatures**: the constraint *is* the API contract, visible in the declaration.
- **Overload resolution by constraint**: when two constrained templates both match, the compiler
  picks the **more constrained** one — a clean replacement for SFINAE tag-dispatch.

Standard library concepts you'll actually use: `std::integral`, `std::floating_point`,
`std::convertible_to`, `std::same_as`, `std::totally_ordered`, `std::movable`, `std::invocable`.

```cpp
template <std::integral T>       // shorthand constraint: only integer types
T next(T x) { return x + 1; }

// Equivalent longhand forms (all identical):
template <typename T> requires std::integral<T>  T next2(T x);   // trailing requires
```

---

## 7. `if constexpr` — compile-time branching

Templates often need to do *different things* for different types. `if constexpr` chooses the branch
at **compile time**, and — critically — **the untaken branch is not even compiled** (it only has to
parse):

```cpp
template <typename T>
void serialize(const T& v) {
    if constexpr (std::integral<T>) {
        writeInt(v);       // only instantiated when T is integral
    } else if constexpr (std::floating_point<T>) {
        writeFloat(v);
    } else {
        writeGeneric(v);   // untaken branches are discarded — no cost, no requirement to be valid
    }
}
```

This matters because in a *plain* `if`, both branches must compile for every `T`. With `if
constexpr`, if `T` is `int`, the `writeFloat`/`writeGeneric` branches can even contain code that
would be *ill-formed* for `int` — they're discarded. It's the modern, readable replacement for
tag-dispatch and much SFINAE. Zero runtime cost: only one branch survives into the binary.

---

## 8. Variadic templates & the cost of templates

**Variadic templates** take any number of arguments of any types — how `std::make_unique`,
`emplace_back`, and type-safe logging are built:

```cpp
template <typename... Args>                       // Args is a "parameter pack"
void log(Args&&... args) {
    (std::cout << ... << args);                    // C++17 fold expression: expands the pack
}
log("order ", 42, " @ ", 100.5);                   // one call, mixed types, no allocation
```

The pack `Args...` is expanded at compile time; `emplace_back` uses this + perfect forwarding
(Module 9) to construct an element in place from arbitrary constructor arguments.

**The cost side — you must be able to name these tradeoffs:**

- **Code bloat**: each distinct instantiation is a separate function/class in the binary. Ten types
  through `max<T>` = ten `max` functions. This grows the binary and can pressure the **instruction
  cache** (Module 15) — relevant in HFT where i-cache misses cost latency.
- **Compile times**: monomorphization is expensive; heavily templated code (Boost, some STL headers)
  compiles slowly.
- **Two-phase lookup / errors**: historically brutal (fixed by concepts). Names in a template are
  looked up in two phases — non-dependent names at definition, dependent names at instantiation —
  which is why you sometimes need `typename T::iterator` and `this->` / `Base<T>::` to disambiguate.

The tradeoff is deliberate: you pay in **build time and binary size** to buy **zero runtime cost**.
For HFT that's exactly the right trade.

---

## Common pitfalls & UB

- **Definitions in headers**: templates must be visible where instantiated, so their definitions
  live in headers (or you use explicit instantiation). Putting a template *definition* in a `.cpp`
  file → linker error ("undefined reference") when used elsewhere.
- **Mismatched deduction**: `max(1, 2.0)` fails — one `T`, two deduced types. Not a bug, a design.
- **Forgetting `typename`/`template` disambiguators**: `T::value_type` in a dependent context needs
  `typename T::value_type`; a dependent member template needs `.template foo<...>()`.
- **`std::function` on the hot path**: it type-erases (heap-allocates for large callables, adds an
  indirect call) — that's the *opposite* of a template's zero-cost inlining. Prefer passing a lambda
  as a template parameter (`template<class F> void each(F f)`) so it inlines.
- **Assuming templates are runtime-polymorphic**: they are resolved entirely at compile time — you
  cannot pick a template instantiation based on a value known only at runtime.

---

## Quiz — tough problems

**Q1.** What does this print, and why?
```cpp
template <typename T> void f(T)        { std::cout << "T\n"; }
template <typename T> void f(T*)       { std::cout << "T*\n"; }
void f(int)                            { std::cout << "int\n"; }
int x = 0; f(&x); f(x);
```
**Answer:** `T*` then `int`. For `f(&x)` (an `int*`), the `f(T*)` template is *more specialized* than
`f(T)` and wins overload resolution. For `f(x)` (an `int`), a **non-template** function is preferred
over a template when both are equally good matches → `f(int)`.

**Q2.** Will this compile? `template<class T> T add(T a, T b){return a+b;} add(1, 2u);`
**Answer:** No. `1` deduces `T=int`, `2u` deduces `T=unsigned` — conflicting deductions for the same
`T`. Fix with two type params, an explicit `add<unsigned>(...)`, or a matching literal.

**Q3.** What's the difference in the *binary* between a `virtual` dispatch and a template call?
**Answer:** The virtual call leaves one function body plus an indirect call through the vtable
(resolved at runtime). The template produces N distinct monomorphized bodies, each a direct,
inlinable call (resolved at compile time). Template = more code, zero indirection; virtual = less
code, per-call indirection.

**Q4.** Why does `if constexpr` let you write a branch that would be ill-formed for some `T`, but a
plain `if` does not?
**Answer:** In `if constexpr`, the *discarded* branch in a template is not instantiated — it only has
to parse. A plain `if` requires *both* branches to be well-formed for the instantiated `T`, because
both are compiled.

**Q5.** True/false: `template <> void f<int>(int)` (a full specialization of a function template) and
`void f(int)` (an overload) are interchangeable.
**Answer:** False. They participate differently in overload resolution: overload resolution first
picks the best *primary* template/non-template, and *then* a matching explicit specialization is
selected. Specializing a function template can lead to surprising picks — the common advice is
"prefer overloading function templates, reserve specialization for class templates."

**Q6.** What is "code bloat" and why might it *hurt* performance despite templates being "zero-cost"?
**Answer:** Each instantiation is a separate copy in the binary. Many instantiations grow the binary
and can evict hot code from the instruction cache (L1i), adding latency — the runtime is zero-cost
*per call*, but the total code footprint isn't free.

**Q7.** Fill in a concept that requires `T` to have a `.value()` returning something convertible to
`double`.
**Answer:**
```cpp
template <typename T>
concept HasValue = requires(const T& t) { { t.value() } -> std::convertible_to<double>; };
```

**Q8.** Where must a template's *definition* live, and what error do you get if you put it in a
`.cpp`?
**Answer:** In a header (visible at every instantiation point), or you provide explicit
instantiations. Otherwise you get a **linker** error ("undefined reference to `foo<int>`") — it
compiles per TU but the definition isn't found when instantiated elsewhere.

**Q9.** `RingBuffer<Order,1024>` and `RingBuffer<Order,2048>` — same type or different, and can a
function take "any RingBuffer of Order"?
**Answer:** Different types (the NTTP `N` is part of the type). A function accepting "any" must itself
be a template: `template<std::size_t N> void drain(RingBuffer<Order,N>&)`.

---

## Indian HFT interview questions

*(Asked at Tower Research Gurgaon, Optiver, IMC, Graviton, Quadeye, AlphaGrep, WorldQuant, Da Vinci,
Squarepoint, Jump, HRT, NK Securities.)*

- **"How do templates achieve zero runtime overhead? Contrast with Java generics."**
  Monomorphization: the compiler generates a separate concrete function/class per type used, so each
  call is a direct, inlinable call — no boxing, no indirection. Java generics use type erasure + a
  single `Object`-based body with runtime casts/boxing, which costs an indirection and heap boxing
  for value types. C++ trades binary size and compile time for zero per-call cost.

- **"Template vs virtual function — when do you pay for polymorphism?"**
  Templates = compile-time polymorphism, resolved and inlined at build time, zero runtime cost, but
  the types must be known at compile time and you get code bloat. Virtual = runtime polymorphism, one
  body + an indirect call and no inlining per call, but flexible for runtime-decided types. HFT hot
  path prefers templates/CRTP; cold/plugin paths can use virtual.

- **"What is a non-type template parameter and why is it useful in low-latency code?"**
  A compile-time *value* (e.g. `std::size_t N`) baked into the type. Enables fixed inline arrays (no
  heap, no runtime size field), power-of-two mask arithmetic, loop unrolling, and compile-time bounds
  checks — exactly what fixed-capacity lock-free ring buffers need.

- **"What replaced SFINAE in C++20 and why is it better?"**
  Concepts (+ `requires`). Named, readable constraints that produce clean error messages, document
  the API contract in the signature, and drive overload resolution by "most constrained wins" — vs.
  `enable_if`/SFINAE which was unreadable boilerplate with poor diagnostics.

- **"Explain `if constexpr` and why the discarded branch rule matters."**
  Compile-time branch selection inside a template; the untaken branch is not instantiated (only
  parsed), so it can legally contain code invalid for the current `T`. A plain `if` requires both
  branches to compile. It replaces tag dispatch and most SFINAE with straight-line, readable code at
  zero runtime cost.

- **"Why do templates live in headers?"**
  Because instantiation needs the full definition at every point of use; the compiler can't
  instantiate `foo<T>` from just a declaration. Alternatives: explicit instantiation in one TU, or
  `extern template` to suppress redundant instantiations and cut build time/bloat.

- **"You have a `std::function` in a hot loop and profiling shows overhead. Fix it."**
  `std::function` type-erases: potential heap allocation for large callables plus an indirect call
  that defeats inlining. Replace it with a template parameter (`template<class F> void run(F&& f)`)
  so the concrete lambda inlines, or use a lightweight non-owning `function_ref`.

---

## In the order book

- **Side-specialized templates**: `template <Side S> class BookSide` lets the compiler eliminate the
  buy-vs-sell branch entirely — the "best bid vs best ask" comparison direction is resolved at
  compile time, not per-message. This is the "compile-time branch elimination" trick from the HFT
  research doc, and it's pure template monomorphization.
- **Fixed-capacity queues**: a `RingBuffer<Message, N>` / `SpscQueue<Message, N>` (NTTP `N`) feeds
  orders into the engine with no heap and power-of-two mask indexing.
- **Concepts as guardrails**: constrain `Price`/`Qty`-like template parameters to be
  `std::integral`/`std::totally_ordered`, catching misuse (e.g. passing a float price) at compile
  time with a readable error.
- **Lambdas over `std::function`**: matching-loop callbacks (e.g. "on fill") are passed as template
  parameters so they inline; `std::function` is reserved for the cold configuration layer where a
  stable ABI / type erasure is genuinely wanted.

---

## Key takeaways

- Templates are **recipes**, not code — the compiler **monomorphizes** one concrete copy per type
  used, giving **zero runtime cost** at the price of code bloat and compile time.
- **Non-type template parameters** bake compile-time *values* into the type — fixed inline arrays,
  mask arithmetic, distinct types per size — the foundation of HFT fixed-capacity structures.
- **Concepts** (C++20) are named, checkable constraints that replace SFINAE: readable errors,
  self-documenting APIs, and "most constrained overload wins" resolution.
- **`if constexpr`** selects a branch at compile time and *discards* the untaken one — enabling
  per-type code paths that a plain `if` couldn't express, at zero runtime cost.
- The template contract: pay in **build time + binary size**, buy **zero per-call overhead** — the
  correct trade for low-latency code.

**Next:** [11 — Inheritance, virtual functions, vtables](11-inheritance-virtual.md) — the *runtime*
polymorphism alternative, how the vtable actually lays out in memory, and why the hot path avoids it.
