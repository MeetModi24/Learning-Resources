# Module 18 — C++20 features that matter (for HFT and this project)

Your resume says **C++20**. In an interview that is a promise you must be able to defend — "I set
the `-std=c++20` flag" won't survive the first follow-up. This module walks the C++20 additions that
genuinely matter for systems/HFT work, explains *why* each exists and what it replaces, and flags
the ones you'll actually use in the order book (★). By the end you should be able to name specific
features, say what problem each solves, and describe exactly where you used it.

---

## 1. Concepts — named, enforceable template constraints (★ used in project)

Before C++20, a template accepted *any* type and only failed deep inside instantiation, producing
the infamous 200-line error walls. **Concepts** let you state, up front, what a type must support —
turning those errors into a clear "this type doesn't satisfy `PriceLike`" at the call site. (Full
treatment in Module 10; here's the HFT-relevant summary.)

```cpp
#include <concepts>

template <std::integral T>          // constrains T to integer types
T next(T x) { return x + 1; }

template <typename T>
concept PriceLike = std::totally_ordered<T>            // can be <, <=, >, >=, ==
                 && std::is_trivially_copyable_v<T>;    // safe to memcpy, no hidden owner

template <PriceLike P>
void insert(P price);                // only accepts types that are ordered AND trivially copyable
```

**Why HFT cares.** Two reasons. First, readable errors — when a teammate passes the wrong type, they
get a one-line diagnostic, not a template avalanche. Second, and more importantly, concepts let you
**encode invariants the domain requires**: a `Price` type *must* be totally ordered (you sort the
book by it) and *should* be trivially copyable (you `memcpy` orders through queues and pools —
Module 15). Constraining on `is_trivially_copyable_v` catches, at compile time, someone accidentally
giving `Price` a `std::string` member that would silently break the pooling/queueing assumptions.

---

## 2. The `<bit>` header — portable, hardware-backed bit manipulation (★ used in project)

Bit tricks used to require compiler-specific intrinsics (`__builtin_ctz`) or error-prone hand-rolled
loops. C++20's `<bit>` standardizes them, and each maps to a single CPU instruction on modern
hardware:

```cpp
#include <bit>
std::countr_zero(x);    // count trailing zeros (ctz)  — index of the lowest set bit
std::countl_zero(x);    // count leading zeros  (clz)  — related to the highest set bit
std::popcount(x);       // number of set bits
std::bit_width(x);      // bits needed to represent x  (= 1 + floor(log2 x) for x>0)
std::has_single_bit(x); // is x a power of two?
std::bit_ceil(x);       // round x up to the next power of two
std::bit_floor(x);      // round x down to a power of two
```

### Why this is the heart of the fast order book

Represent occupied price levels as a **bitset**: bit *i* = "price level *i* has resting orders."
Then finding the best price is a single hardware instruction instead of a tree traversal.

```
occupied_ (a 64-bit word per 64 levels):

 level:  63                              ...                          2   1   0
        ┌───┬───┬───┬─── ... ───┬───┬───┬───┬───┬───┐
 bits:  │ 0 │ 0 │ 1 │           │ 0 │ 1 │ 0 │ 0 │ 1 │      bit set = level has orders
        └───┴───┴───┴─── ... ───┴───┴───┴───┴───┴───┘
                                          ▲           ▲
                        countr_zero → 0 ──┘           └── lowest set bit = best ask level
                        (one CPU instruction, no branch, no pointer chase)
```

```cpp
// best ask = lowest occupied price level:
std::uint64_t occupied = /* bitmask of levels with orders */;
int bestLevel = std::countr_zero(occupied);   // one instruction — no tree walk
```

This is the mechanism behind the "27M orders/sec" style designs referenced in the research doc: an
`std::map` finds the best price by walking to the tree's leftmost node (pointer chasing, cache
misses, ~log n); the bitset does it in one `countr_zero`. `has_single_bit` / `bit_ceil` are also how
you size ring buffers to a power of two so `& (N-1)` replaces `% N` (Module 16).

---

## 3. `std::span` — a non-owning view over contiguous memory (★ usable in project)

A `std::span<T>` is nothing but a **pointer + a length** — a lightweight, non-owning *window* into a
contiguous sequence someone else owns. It replaces the old, error-prone C habit of passing a raw
pointer and a separate length argument.

```
std::span<const Order> s over a vector's storage:

   span object (16 bytes, on the stack)        the actual data (owned elsewhere)
   ┌──────────────┬──────────────┐             ┌────┬────┬────┬────┬────┐
   │ ptr ─────────┼──▶           │────────────▶│ O0 │ O1 │ O2 │ O3 │ O4 │
   │ size = 5     │              │             └────┴────┴────┴────┴────┘
   └──────────────┴──────────────┘             (span owns NOTHING — no copy, no free)
```

```cpp
#include <span>
void process(std::span<const Order> orders) {   // accepts vector, std::array, C array — no copy
    for (const auto& o : orders) { /* ... */ }
}

std::vector<Order> v = ...;
process(v);                    // whole vector
process({v.data() + 10, 32});  // a 32-element slice, still zero copy
```

**Why HFT cares.** It's the safe, modern way to hand around slices of a receive buffer during
zero-copy parsing — a feed-handler staple. You view directly into the network buffer without copying
bytes, and `span` carries the length so you can't walk off the end the way a bare pointer lets you.
Use `std::span<const T>` for read-only views. Caveat: a span is a *borrow* — never return one that
outlives the buffer it points into (dangling-view UB, the same trap as `string_view`, Module 4/12).

---

## 4. Ranges — composable, lazy algorithms

Ranges let you pipe algorithms together and drop the `begin()/end()` boilerplate. The key property
is **laziness**: a *view* computes elements on demand as you iterate, without building intermediate
containers.

```cpp
#include <ranges>
namespace rng = std::ranges;

rng::sort(v);                              // no v.begin(), v.end()

auto topBids = book
             | std::views::filter(isBid)   // lazy: nothing computed yet
             | std::views::take(10);        // still lazy — no allocation
for (auto& o : topBids) { /* filtering happens here, element by element */ }
```

Because views don't allocate intermediate vectors, a `filter | transform | take` chain often
compiles to a single loop — expressive *and* efficient for analytics and reporting layers. **Caveat
for HFT:** on the absolute hottest loop, verify the compiler actually fuses and inlines the view
chain (support has matured but isn't uniform). Ranges shine in the cold analytics/reporting code;
be measured about the matching loop.

---

## 5. Three-way comparison `<=>` — the spaceship operator (★ used in project)

Before C++20, giving a type all six relational operators (`<`, `<=`, `>`, `>=`, `==`, `!=`) meant
writing them by hand — tedious and easy to get inconsistent. The **spaceship operator** generates
them all from one defaulted line (Module 8):

```cpp
struct Price {
    std::int64_t ticks;
    auto operator<=>(const Price&) const = default;   // generates <, <=, >, >=, and ==/!= too
};
```

`a <=> b` returns an ordering object (`std::strong_ordering` here) that's `<0`, `==0`, or `>0`. For
`Price`, `= default` does the obvious member-wise comparison. This is used directly to order prices
in the book, and it guarantees the six operators stay mutually consistent — no chance of `<` and
`>=` disagreeing because you edited one and forgot the other.

---

## 6. `constinit` and expanded `constexpr` (★ usable)

- **`constinit`** guarantees a static/global variable is initialized at *compile time*, defusing the
  "static initialization order fiasco" (two globals in different translation units racing to
  initialize each other). It asserts constant initialization without also making the variable
  `const`:
  ```cpp
  constinit int gPriceScale = 10000;   // guaranteed initialized before any dynamic init runs
  ```
- **Expanded `constexpr`** — C++20 lets far more run at compile time, including `constexpr` dynamic
  allocation and `std::vector`/`std::string` inside `constexpr` functions. Use it to precompute
  lookup tables, price-scaling constants, or masks entirely at compile time so they cost nothing at
  runtime (Module 5/17).

---

## 7. `[[likely]]` / `[[unlikely]]` — branch-probability hints (★ usable)

Covered in Module 17. C++20 standardizes the hint that tells the compiler which side of a branch is
the common case, so it lays out the hot path linearly (better i-cache behavior, better default
prediction):

```cpp
if (order.qty > 0) [[likely]]   { match(order); }
else               [[unlikely]] { reject(order); }
```

Purely a layout hint — never affects correctness. Use on the matching path where you know the
common case.

---

## 8. Designated initializers — named aggregate initialization

Borrowed from C, this lets you initialize aggregate members by name, which is self-documenting and
guards against silently swapping two same-typed arguments:

```cpp
Order o{ .id = 1, .price = 100, .qty = 5 };   // clearly which field is which
```

Rules to remember: members must be initialized in **declaration order**, and it works only for
aggregates (no user-declared constructors). Great for readable test fixtures and config structs.

---

## 9. `std::jthread` — a self-joining, cancellable thread (RAII for threads)

`std::thread` has a nasty gotcha: if you forget to `join()` it before it's destroyed, your program
calls `std::terminate`. `std::jthread` (C++20) fixes both problems:

```cpp
#include <thread>
{
    std::jthread worker([](std::stop_token st) {
        while (!st.stop_requested()) { /* do work */ }
    });
}   // destructor auto-requests stop AND joins — no manual cleanup, no terminate risk
```

It's **RAII for threads** (Module 7): the destructor requests cooperative cancellation via a
`std::stop_token` and then joins. For a feed-handler or logging thread that must shut down cleanly,
this removes an entire class of bugs.

---

## 10. Coroutines — know of them, likely not in this project

C++20 adds language-level `co_await` / `co_yield` / `co_return` for writing asynchronous code (and
generators) in a linear style. They're powerful for async network I/O and lazy sequence generation,
but they have allocation subtleties (the coroutine frame may heap-allocate unless elided) that make
them a poor fit for a *synchronous* matching engine. The honest interview answer: "I know coroutines
exist and where they'd fit — an async feed handler — but I kept the matching path synchronous and
allocation-free, so I didn't use them here."

---

## 11. Others worth naming

- **Modules** (`import`/`export`) — replace textual `#include` with a compiled interface for faster,
  cleaner builds; toolchain support is still maturing, so mention you know them but that build
  support varies.
- **`consteval`** (Module 5) — "immediate functions" that *must* run at compile time (stronger than
  `constexpr`, which merely *may*).
- **`<chrono>` calendar & time-zone support** — civil dates and time zones in the standard library,
  useful for session/timestamp handling.
- **Abbreviated function templates** — `auto` parameters (`void f(auto x)`) as shorthand for simple
  templates.

---

## 12. Common pitfalls & gotchas

- **Dangling `span`/`string_view`.** Both are non-owning borrows. Returning one that outlives the
  underlying buffer, or holding it after the owner is destroyed, is UB. Treat them like references.
- **Assuming ranges views are always faster.** They're lazy and often fuse, but on the hottest loop
  verify the compiler inlined the chain — an un-inlined view adapter can add indirection.
- **`constinit` ≠ `const`.** `constinit` guarantees *compile-time initialization*; the variable is
  still mutable afterward. Use `constexpr` if you also want immutability.
- **Designated initializers out of order.** Members must appear in declaration order; C allows
  reordering, C++ does not — it won't compile.
- **Claiming "C++20" generically in an interview.** Naming the flag invites "which features and
  why?" Have the specific story (Section 13) ready.
- **`<=> = default` on a type with a `std::string`/floating-point member.** Defaulted `<=>` on
  floats gives `partial_ordering` (NaN isn't ordered) — fine to know, but a reason to prefer integer
  ticks for `Price`.

---

## 13. Quiz — tough problems

**Q1.** `occupied = 0b0010'1000` (bits 3 and 5 set, LSB = bit 0). What do `std::countr_zero`,
`std::countl_zero` (assume 8-bit), `std::popcount`, and `std::has_single_bit` return?

**Answer:** `countr_zero` = **3** (three low zero bits before the first set bit → best/lowest level
is 3). `countl_zero` (8-bit width) = **2** (bits 7 and 6 are zero above the highest set bit at 5).
`popcount` = **2** (two set bits). `has_single_bit` = **false** (two bits set, not exactly one).

---

**Q2.** Why is a bitset + `countr_zero` asymptotically and practically faster than `std::map` for
finding the best price in an order book? Give both the complexity and the hardware reason.

**Answer:** Complexity: `std::map::begin()` is O(1) amortized but reaching the min still involves
tree structure maintained at O(log n) per insert/erase; the bitset find is O(words) ≈ O(1) for a
bounded price range. The practical reason is the hardware: `std::map` chases pointers across
scattered, cache-cold tree nodes (a cache miss per hop, ~100 ns each), whereas the bitset is a few
contiguous 64-bit words and `countr_zero` is a *single* branch-free CPU instruction. No pointer
chasing, no cache misses, no rebalancing.

---

**Q3.** What's wrong with this function, and how does `std::span` fix it?

```cpp
double sum(const double* data, size_t n);   // caller passes pointer + length separately
```

**Answer:** The caller can pass a wrong `n` (or forget to update it), reading out of bounds — a
classic buffer overrun with no compile-time or runtime guard. `std::span<const double>` bundles the
pointer and length into one object, so they can't get out of sync; it constructs safely from a
`vector`/`array`/C array and offers bounds-checked `.at()` / range-`for`. Signature becomes
`double sum(std::span<const double> data);`.

---

**Q4.** After `struct P { int a; double b; auto operator<=>(const P&) const = default; };`, what
ordering category does `P`'s comparison have, and why might that surprise you?

**Answer:** `std::partial_ordering`. Because `b` is a `double`, and floating-point has NaN, which is
*unordered* with everything — so the strongest guarantee the defaulted `<=>` can give is *partial*
ordering (a NaN member means `a <=> b` can be `unordered`). This is a concrete argument for
representing prices as integer **ticks** (`std::int64_t`), which yield `strong_ordering`.

---

**Q5.** Write a concept `Poolable` that a type must satisfy to live in the object pool from Module 3
(must be trivially copyable and default-constructible), and constrain a pool template on it.

**Answer:**

```cpp
template <typename T>
concept Poolable = std::is_trivially_copyable_v<T>
                && std::is_default_constructible_v<T>;

template <Poolable T>
class ObjectPool { /* ... */ };
```

Trivial copyability lets the pool `memcpy`/relocate slots and guarantees no hidden ownership;
default-constructibility lets it pre-create the storage array at startup.

---

**Q6.** You replace `#define N 1024; x % N` with something using `<bit>`. What do you write, what
must be true of `N`, and why is it faster?

**Answer:** If `N` is a power of two, `x % N` equals `x & (N - 1)`. You can assert the precondition
with `static_assert(std::has_single_bit(N));`, or size a buffer up with `std::bit_ceil(requested)`.
It's faster because a bitwise AND is one cycle, whereas integer division/modulo is many cycles (and
often not pipelined). This is exactly why the SPSC ring buffer (Module 16) uses power-of-two capacity.

---

**Q7.** Why is `std::jthread` preferable to `std::thread` for a shutdown-sensitive background thread,
and what happens if you forget to join a plain `std::thread`?

**Answer:** A `std::thread` destroyed while still joinable (not joined/detached) calls
`std::terminate` — an instant crash. `std::jthread`'s destructor automatically requests cooperative
stop (via `std::stop_token`) *and* joins, so it can't terminate and it shuts the worker down
cleanly. It's RAII applied to thread lifetime.

---

## 14. Indian HFT interview questions

**Q (Tower Research / Optiver): "Your resume says C++20 — which features did you actually use and
why?"** (This is *the* question this module exists for.) The defensible answer: "**Concepts** to
constrain the price/quantity value types to totally-ordered and trivially-copyable; **`<bit>`'s
`countr_zero`/`countl_zero`** on a price-level *bitset* to find best bid/ask in a single instruction
instead of a tree walk; **`operator<=>`** to order prices consistently; **`std::span`** for
zero-copy views into the input buffer; and **`[[likely]]`/`[[unlikely]]`** on the matching path."

**Q (IMC / Graviton): "How do you find the best bid/ask in O(1)?"**
Keep a bitset of occupied price levels (one bit per level). Best ask = lowest set bit =
`std::countr_zero(word)`; best bid = highest set bit, found via `countl_zero`. One branch-free CPU
instruction, no tree traversal, no rebalancing — versus `std::map`'s pointer-chasing walk to the
leftmost node.

**Q (Quadeye / AlphaGrep): "Why integer ticks for price instead of `double`?"**
Doubles have rounding error (0.1 isn't exact) and give only `partial_ordering` under `<=>` because
of NaN — both fatal for a system that compares and sums prices exactly. Integer ticks
(`std::int64_t` = price × tick-scale) are exact, give `strong_ordering`, are trivially copyable for
pooling, and compare in one instruction.

**Q (WorldQuant / Da Vinci): "What is `std::span` and where would you use it in a feed handler?"**
A non-owning pointer+length view over contiguous memory. In a feed handler I use it to hand slices
of the receive buffer to parsers without copying — zero-copy parsing — while carrying the length so
parsers can't overrun. It's the safe replacement for `(ptr, len)` argument pairs. Caveat: it must
not outlive the buffer.

**Q (Jump / HRT): "Are C++20 ranges appropriate on the hot path?"**
For cold analytics/reporting, yes — they're lazy, fuse into single loops, and are very readable. On
the *hottest* matching loop I verify the view chain actually inlined (check the assembly); if there's
residual indirection I fall back to a hand-written loop. Knowing *when not* to use a feature is part
of the answer.

**Q (Squarepoint / Millennium): "Explain concepts vs SFINAE."**
Both constrain templates, but concepts (C++20) are named, composable, and produce readable "type
doesn't satisfy X" errors at the call site, whereas SFINAE (`std::enable_if`) is an unnamed,
hard-to-read trick that fails deep in instantiation. Concepts also participate cleanly in overload
resolution ordering. I use concepts to require `PriceLike` (ordered + trivially copyable).

**Q (NK Securities / Mansard): "How do you guarantee a global config value is initialized before any
other global uses it?"**
Mark it `constinit` so it's guaranteed constant-initialized at compile time, sidestepping the
static-initialization-order fiasco. If it should also be immutable, use `constexpr` instead.

---

## 15. In the order book — your defensible C++20 story

Pull it together into one sentence you can say in an interview and then defend under follow-up:

> "I used C++20 **concepts** to constrain the price/quantity value types (ordered + trivially
> copyable), **`<bit>`'s `countr_zero`/`countl_zero`** on a price-level **bitset** to find best
> bid/ask in a single hardware instruction instead of a tree traversal, **`operator<=>`** to order
> prices, **`std::span`** for zero-copy views into the input buffer, `constexpr` for the
> compile-time level-count and masks, and **`[[likely]]`/`[[unlikely]]`** on the matching path.
> `std::jthread` runs the shutdown-clean I/O thread."

That turns the "C++20" on your resume from a flag into a claim with a mechanism behind every word.

---

## Key takeaways

- **Concepts** make template constraints named, composable, and readable — and let you encode domain
  invariants (`PriceLike` = totally ordered + trivially copyable) that catch misuse at compile time.
- **`<bit>`** turns bit tricks into portable single-instruction operations; the price-level
  **bitset + `countr_zero`** is the O(1) best-price mechanism at the core of the fast book.
- **`std::span`** is a non-owning pointer+length view — the safe replacement for `(ptr, len)` pairs,
  ideal for zero-copy buffer parsing (but never let it dangle).
- **`operator<=> = default`** generates all comparisons from one line; use integer ticks so prices
  get `strong_ordering`.
- **`constinit`** kills the static-init-order fiasco; expanded **`constexpr`** moves lookup tables to
  compile time; **`[[likely]]`** lays out hot branches; **`std::jthread`** is RAII for threads.
- The interview payoff isn't knowing the features exist — it's naming the *specific* ones you used,
  what each replaced, and where (including knowing when *not* to use ranges/coroutines on the hot
  path).

**Next:** Part B — [HFT infrastructure, Module 01: Market microstructure & the system pipeline](../hft-infrastructure-guide/01-market-microstructure-and-hft-systems.md) — where all this C++ gets used.
— the capstone where every module becomes a design decision.
