# Module 5 — `const`, `constexpr`, `consteval`, const-correctness

Two separate ideas share a keyword prefix and confuse everyone at first. `const` is about
**promises** — "I won't modify this" — a contract checked by the compiler at zero runtime cost.
`constexpr`/`consteval`/`constinit` are about **when computation happens** — compile time vs runtime.
HFT cares intensely about both: promises unlock optimizations and make objects safe to share across
threads for reads (Module 16), and moving work to compile time means it costs *literally nothing* when
the program runs. Interviewers use the `const int* p` vs `int* const p` puzzle and "what's the
difference between `const` and `constexpr`?" as quick filters — you should be able to answer both
without hesitating.

---

## 1. The mental model: promises vs timing

Keep these two axes separate in your head from the start:

```
        WHAT it controls              WHEN it acts
const      "cannot be modified"       enforced at COMPILE time, costs nothing at RUNTIME
constexpr  "MAY run at compile time"  runs at compile time IF inputs are constant, else runtime
consteval  "MUST run at compile time" always compile time; runtime call = compile error
constinit  "initialized at compile    guarantees no runtime static-init; value may still change
            time"                      later (it is not const)
```

`const` answers *"can this change?"* `constexpr` answers *"can the compiler compute this before the
program even starts?"* They're independent — you can have `const` runtime values and non-`const`
compile-time-initialized values.

---

## 2. `const` — a promise not to modify

At its simplest, `const` on a variable prevents reassignment:

```cpp
const int max = 100;   // can't reassign
max = 200;             // ERROR
```

But the real power of `const` is in **interfaces** — it documents and enforces what a function or
method *won't* touch:

```cpp
class Order {
    int qty_;
public:
    int qty() const { return qty_; }  // const member fn: promises not to modify *this
    void setQty(int q) { qty_ = q; }  // non-const: may modify
};

void audit(const Order& o) {
    o.qty();       // OK — const method callable on a const ref
    // o.setQty(5); // ERROR — can't call non-const method through const ref
}
```

**const-correctness** is the discipline of marking everything `const` that doesn't need to mutate.
The payoffs:

- The compiler **catches accidental writes** at compile time — a whole class of bugs becomes
  impossible.
- Callers can read a signature and know exactly what a function *won't* modify. `void f(const T&)`
  says "I only read your object."
- A genuinely `const` object is **safe to share across threads for concurrent reads without a lock**
  (Module 16) — this is a big deal in low-latency systems.
- It's a **free** optimization hint: the compiler can keep values in registers, knowing they won't
  change underneath it.

Rule of thumb: **make it `const` unless it needs to change** — parameters, methods, locals.

### How `const` looks in memory (it usually doesn't)

`const` on a local or parameter is a *compile-time contract*, not a runtime tag. There is no extra
byte, no flag stored with the object:

```
int x        →  [ 4 bytes on the stack ]   ← compiler lets you write to it
const int x  →  [ 4 bytes on the stack ]   ← IDENTICAL bytes; compiler REFUSES writes at compile time
```

The exception: a **namespace-scope `const`/`constexpr` with a compile-time value** may be placed in a
read-only data segment (`.rodata`) and shared — or eliminated entirely if the compiler can inline the
value everywhere it's used. Writing through a cast to such an object is UB precisely because the
memory may be genuinely read-only at the hardware page level.

### `const` member functions and `mutable`

A `const` method can't modify members — *except* those marked `mutable`, the escape hatch for
bookkeeping that doesn't affect the object's logical state (caches, hit counters, lazy-computed
values, a mutex):

```cpp
class Cache {
    mutable int hits_ = 0;   // physical state, not logical state
public:
    int get() const { ++hits_; return 42; }  // allowed to write hits_ despite const
};
```

The distinction interviewers want: `const` protects **logical constness** (what the object *means* to
observers); `mutable` lets you change **physical state** that observers can't see. A `mutable
std::mutex` locked inside a `const` getter is the textbook legitimate use.

### East const vs West const

`const int` and `int const` are **identical**. "West const" (`const int`) reads more naturally in
English; "East const" (`int const`) reads consistently right-to-left, which is why it shines with
pointers (next section). Pick one style and be consistent.

---

## 3. `const` and pointers — read right-to-left

The single most-quizzed `const` puzzle. The trick: **read the declaration from the variable name
leftward (right-to-left).**

```cpp
int x = 5;
const int* p       = &x;   // "p is a pointer to (const int)"  — pointee is const
int* const q       = &x;   // "q is a (const pointer) to int"  — pointer is const
const int* const r = &x;   // both const
```

```
const int* p       →  can change WHERE p points;   CANNOT change *p (the int)
int* const q       →  CANNOT change where q points; can change *q (the int)
const int* const r →  can change neither
```

Memory picture — a `const int* p` (pointer-to-const):

```
STACK                              elsewhere
┌──────────────┐                   ┌──────────┐
│ p ───────────┼──────────────────▶│  int x   │  ← *p is read-only THROUGH p
└──────────────┘                   └──────────┘
   p itself is writable            (x may still be modified by another non-const path)
```

Note the subtlety: `const int* p` means *you can't modify the int **through p***, not that the int is
immutable — `x` itself can still be changed via `x = 7;`. This is "low-level const."

---

## 4. `constexpr` — compute at compile time when possible

```cpp
constexpr int square(int x) { return x * x; }

constexpr int a = square(5);  // computed at COMPILE time -> baked-in 25 in the binary
int n = readInput();
int b = square(n);            // n unknown at compile time -> runs at RUNTIME
```

`constexpr` means **"usable in a constant expression."** A `constexpr` *function* runs at compile
time *if its inputs are compile-time constants*, otherwise it silently runs at runtime — it's a
"best effort compile-time" marker. A `constexpr` *variable* is stronger: it **must** be initialized
with a compile-time constant, or the program won't compile.

Why it matters for HFT: lookup tables, array sizes, bit masks, and precomputed constants are all
computed by the compiler and **baked into the binary** — zero runtime cost, and the values sit in
`.rodata` ready to use:

```cpp
constexpr std::size_t PriceLevels = 1 << 16;  // 65536, computed at compile time
int book[PriceLevels];                          // fixed-size array, no runtime sizing

constexpr std::uint64_t mask = PriceLevels - 1; // 0xFFFF — masking replaces modulo (Module 18)
```

### What can be `constexpr`?

The rules have loosened every standard. In C++20 a `constexpr` function may contain loops, local
variables, `if`/`switch`, even most of the STL (`std::vector`, `std::string` are `constexpr` in
C++20). Restrictions that remain: no `static` locals with runtime init, no `goto`, no truly runtime
operations (I/O, `new`/`delete` that escapes the evaluation, `reinterpret_cast`). If you write a
`constexpr` function that *can't* be evaluated at compile time for the given inputs, it just runs at
runtime — no error, unless you assign its result to a `constexpr` variable.

### Timing picture

```
constexpr int t = square(5);   // COMPILE TIME:  binary literally contains 25
                                //   ┌ .rodata ┐
                                //   │ t = 25  │   ← no multiply ever executes at runtime
int b = square(readInput());   // RUNTIME:  a real imul instruction runs each call
```

---

## 5. `consteval` (C++20) — MUST run at compile time

```cpp
consteval int forceCompileTime(int x) { return x * x; }

constexpr int ok = forceCompileTime(5);  // fine — evaluated during compilation
int n = readInput();
// int bad = forceCompileTime(n);        // ERROR — n isn't a constant expression
```

`consteval` declares an **immediate function**: *every* call must be evaluated at compile time. If a
call can't be (its arguments aren't constant), it's a **compile error** — not a silent runtime
fallback like `constexpr`. Use it when a runtime evaluation would be a *bug*: building a
compile-time-only table, generating a hash of a string literal, enforcing that a config value is
known at build time.

The mental distinction interviewers want:

- `constexpr` = *"may"* run at compile time (falls back to runtime silently).
- `consteval` = *"must"* run at compile time (runtime call = error).

---

## 6. `constinit` (C++20) — guarantee static init at compile time

`constinit` attacks the **static initialization order fiasco**: when two global objects in different
translation units depend on each other, the order their runtime constructors run is unspecified, so
one may read the other before it's initialized. `constinit` forces the variable to be
*constant-initialized* — set up at compile time, before any runtime code runs — eliminating the race:

```cpp
constinit int g = square(10);  // guaranteed initialized at compile time, before main()
```

Crucially, `constinit` is **not** `const` — the value can still change at runtime. It only guarantees
*how it starts*. Use it for mutable globals that must be ready before any startup code touches them
(a global config, a lookup table, a logging sink).

```
const      → cannot change,  init timing unspecified for non-constexpr
constexpr  → cannot change,  guaranteed compile-time init
constinit  → CAN change,     guaranteed compile-time init (no static-init fiasco)
```

---

## 7. `const` vs `#define`

Never use macros for constants. `const`/`constexpr` are **typed**, **scoped**, respect namespaces,
appear in the debugger, and obey normal lookup rules. `#define` is blind preprocessor text
substitution — no type, no scope, invisible to the debugger, and a source of subtle bugs:

```cpp
#define SQUARE(x) x*x
int r = SQUARE(2+3);   // expands to 2+3*2+3 = 11, not 25 — classic macro bug

constexpr int square(int x) { return x*x; }   // typed, scoped, correct
```

The one thing macros still do that `const` can't is textual tricks (stringizing, token pasting,
include guards) — but for *values*, always reach for `constexpr`.

---

## Common pitfalls & UB

- **Casting away `const` and then writing** (`const_cast` to modify) → UB if the object was
  *originally* `const` (it may live in read-only memory). Legal only if the underlying object was
  non-const to begin with.
- **`const int* p` ≠ immutable int.** You just can't modify *through p*; other paths can.
- **`constexpr` function silently runs at runtime** when inputs aren't constant — fine, but don't
  *assume* compile-time evaluation unless you assign to a `constexpr` variable or use `consteval`.
- **Members init in declaration order**, not init-list order — a `const` member depending on another
  can bite you (Module 6).
- **`mutable` on logical state** — abusing `mutable` to sidestep `const` breaks the contract callers
  rely on; reserve it for genuinely invisible bookkeeping.
- **Forgetting `constexpr` variables must be constant** — `constexpr int x = readInput();` won't
  compile.

---

## Quiz — tough problems

**Q1.** What's the difference between `const int* p`, `int* const p`, and `int const* p`?

**Answer:** `const int* p` and `int const* p` are **identical** — pointer to const int (can't write
`*p`, can re-seat `p`). `int* const p` is a const pointer to int (can write `*p`, can't re-seat `p`).
Read right-to-left from the name.

**Q2.** Does `const` add any bytes to an object or cost anything at runtime?

**Answer:** No. `const` is a compile-time contract; the object's bytes are identical. It may *enable*
optimizations and may place namespace-scope constants in read-only memory, but there's no runtime tag
or check.

**Q3.** Will this compile? Why?
```cpp
int n = 5;
constexpr int x = n * 2;
```
**Answer:** No. `n` is a non-const runtime variable, so `n * 2` is not a constant expression, but a
`constexpr` *variable* requires one. Make `n` itself `const`/`constexpr` and it compiles.

**Q4.** What does this print, and is there UB?
```cpp
const int c = 10;
int* p = const_cast<int*>(&c);
*p = 20;
std::cout << c << " " << *p;
```
**Answer:** UB — modifying an object that was *declared* `const` is undefined. In practice you often
see `10 20` (the compiler substitutes the known constant `10` for `c` but the memory now holds `20`),
which is exactly the kind of "impossible" output that signals UB. Never `const_cast`-and-write a
truly const object.

**Q5.** `constexpr` vs `consteval` — when does each error out?

**Answer:** `constexpr` errors only if you *require* a constant (assign its result to a `constexpr`
variable / use in an array size) and the inputs aren't constant. `consteval` errors on *any* call
that can't be evaluated at compile time, always.

**Q6.** Is `mutable` a violation of `const`-correctness?

**Answer:** No — it's the sanctioned tool for **physical** state that doesn't affect **logical**
const-ness (caches, hit counters, a `std::mutex`). The object still *appears* unchanged to observers.
Abusing it to mutate logical state *is* a violation.

**Q7.** Predict:
```cpp
constexpr int fib(int n){ return n<2 ? n : fib(n-1)+fib(n-2); }
int arr[fib(10)];
```
How big is `arr`, and when is `fib(10)` computed?

**Answer:** `fib(10) == 55`, computed **at compile time** (an array bound is a constant-expression
context), so `arr` has 55 elements and no recursion runs at runtime. This is the classic
"compile-time computation" demonstration.

**Q8.** Why prefer `constinit` over a plain global for a lookup table used during startup?

**Answer:** A plain global with a runtime constructor is subject to the static-initialization-order
fiasco — another TU's global might read it before its constructor ran. `constinit` forces
compile-time (constant) initialization so the table is valid before any runtime code executes, while
still allowing later mutation (unlike `constexpr`).

---

## Indian HFT interview questions

**Q (Tower Research / Optiver): "Difference between `const` and `constexpr`?"**
`const` is a *promise not to modify* — a compile-time-checked contract with zero runtime footprint.
`constexpr` is about *when a value is computed* — it lets the compiler evaluate the expression at
compile time and bake the result into the binary. A `constexpr` variable is implicitly `const`; a
`const` variable is not necessarily `constexpr` (its value may be a runtime input).

**Q (Graviton / AlphaGrep): "Walk me through `const int* const p`."**
Right-to-left: `p` is a const pointer (can't re-seat) to a const int (can't modify through it). Both
the pointer and what it points at are read-only via `p`.

**Q (IMC / Jump): "Why does const-correctness matter in a multithreaded low-latency system?"**
A genuinely `const` object is safe for concurrent reads with no lock (no writer means no data race).
Marking read-only paths `const` documents that invariant, lets you share objects across threads
cheaply, and lets the compiler keep values in registers. It converts a class of concurrency and
aliasing bugs into compile errors.

**Q (Quadeye / NK Securities): "How would you build a lookup table that costs nothing at runtime?"**
`constexpr` (or `consteval`) function that fills a `std::array` at compile time, so the table lives in
`.rodata` and every lookup is a load with no computation. E.g. precomputed price→index maps, CRC
tables, or bit masks — `constexpr std::size_t mask = N - 1;` where `N` is a power of two replaces `%`
with `&`.

**Q (Da Vinci / Squarepoint): "When would you use `consteval` over `constexpr`?"**
When a runtime evaluation would be a bug and you want the compiler to *enforce* compile-time
evaluation — e.g. a `consteval` that validates and hashes a config string literal, so a
non-constant argument is a compile error rather than a silent runtime cost.

**Q (Millennium / Hudson River): "You mark a getter `const` but it caches a computed value. How?"**
Make the cache member `mutable` (and, if multithreaded, guard it with a `mutable std::mutex`). The
getter stays logically `const` — observers see no change — while physically updating hidden cache
state. This is the canonical legitimate use of `mutable`.

**Q (Mansard / Qube): "Is `#define PI 3.14` acceptable for constants?"**
No — use `constexpr double pi = 3.14;`. Macros are untyped, unscoped, invisible to the debugger, and
prone to substitution bugs (`SQUARE(a+b)`). `constexpr` gives you a typed, scoped, debuggable
compile-time constant.

---

## In the order book

- **Query methods** (`bestBid()`, `spread()`, `qtyAt(price)`) are `const` — they don't mutate the
  book. An interviewer immediately sees you understand read/write separation, and it makes the book
  safe to hand to a read-only snapshot/reporting thread.
- **`constexpr`** for the price-array size, tick masks, and bitset widths (Module 15): compile-time
  constants baked into the binary, and power-of-two sizes let `& (N-1)` replace `% N` (Module 18) —
  no runtime cost at all.
- **`const Order&`** parameters throughout for cheap, safe, read-only passing (Module 2).
- **`constinit`** for any global config/table that startup code touches before `main`'s body runs —
  it sidesteps the static-init-order fiasco while still allowing the table to be updated later.
- **`mutable`** on internal counters/caches (e.g. a cached best-bid, or a hit counter inside a `const`
  lookup) so query methods stay `const` without lying about it.

---

## Key takeaways

- `const` = a **promise not to modify**, checked at compile time, **zero** runtime cost; the discipline
  of applying it everywhere is *const-correctness* and it turns bugs into compile errors.
- Read pointer declarations **right-to-left**: `const int* p` (pointee const, re-seatable) vs
  `int* const p` (pointer const, dereference-writable).
- `constexpr` = *may* compute at compile time (silent runtime fallback); a `constexpr` **variable**
  *must* be a constant expression. `consteval` = *must* compute at compile time (runtime call =
  error).
- `constinit` guarantees compile-time initialization (kills the static-init-order fiasco) but the
  value can still change — it is **not** `const`.
- `mutable` is the sanctioned escape hatch for **physical** state (caches, counters, a mutex) inside
  a logically `const` method — not a license to mutate logical state.
- Prefer `constexpr` over `#define` for values: typed, scoped, debuggable, no substitution bugs.

**Next:** [06 — Classes: constructors, destructors, `this`, access control](06-classes.md) — the
machinery for bundling data with behavior and enforcing invariants.
