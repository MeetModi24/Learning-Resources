# Module 8 — Operator overloading & value semantics

Operator overloading lets your types behave like built-ins (`a + b`, `a < b`, `a == b`). Used well it
makes numeric/domain code read like math; abused, it hides cost behind innocent-looking symbols. HFT
interviewers watch for both **taste** (don't surprise the reader) and **cost-awareness** (an operator
that allocates or copies megabytes looks free at the call site). This module also covers *value
semantics* — the design principle that makes C++ types behave like numbers rather than shared
references — because operator overloading and value semantics are two sides of the same coin.

---

## 1. The basic idea

An operator is just a function with funny syntax. `a + b` is a call to `operator+(a, b)`; `a < b`
calls `operator<(a, b)`. You can define these for your own types.

```cpp
struct Price {
    std::int64_t ticks;   // integer ticks, NOT double (see §6)

    // member operator: the left operand is *this
    Price& operator+=(Price rhs) { ticks += rhs.ticks; return *this; }
};

// non-member (free) operator: symmetric, preferred for binary arithmetic
Price operator+(Price a, Price b) { return Price{a.ticks + b.ticks}; }
bool  operator==(Price a, Price b) { return a.ticks == b.ticks; }
bool  operator<(Price a, Price b)  { return a.ticks < b.ticks; }
```

The distinction between **member** and **free (non-member)** operators matters more than it looks:

- A **member** operator has `*this` as its implicit left operand. Because the left operand is already
  a `Price`, no conversion can be applied to it.
- A **free** operator takes both operands as parameters, so an implicit conversion can apply to
  *either* side. This is why `2 + price` can work (if `int→Price` is convertible) with a free
  `operator+` but not with a member one.

Guidelines that fall out of this:

- **Compound assignment (`+=`) as a member; binary arithmetic (`+`) as a free function** implemented
  in terms of `+=`. This gives symmetric conversion behavior and avoids duplicating logic.
- **Return by value for arithmetic** (`a + b` produces a new value); **return `*this` by reference for
  assignment** (`a += b` mutates and yields the same object, so chaining `(a += b) += c` works).
- Overload only when the meaning is *obvious*. `operator+` on two prices = clear. `operator+` that
  "adds an order to a book" = confusing; name it `book.add(order)` instead.

### Why `+=` first, then `+` in terms of it

```cpp
Price operator+(Price a, Price b) {   // a is a BY-VALUE copy
    a += b;                           // reuse the member logic
    return a;                         // return the modified copy
}
```

One source of truth for "how do two prices add," and the by-value first parameter is the copy you'd
have to make anyway.

---

## 2. The spaceship operator `<=>` (C++20)

Before C++20 you wrote all six comparisons (`< > <= >= == !=`) by hand — six functions of pure
boilerplate for every value type. Now one line generates them:

```cpp
#include <compare>
struct Price {
    std::int64_t ticks;
    auto operator<=>(const Price&) const = default;  // generates < <= > >= automatically
    bool operator==(const Price&) const = default;   // == generated separately (see note)
};
// Now Price supports all comparisons, and works in std::map / std::sort out of the box.
```

`operator<=>` (the "spaceship" / three-way comparison) returns an **ordering category**, not a bool:

- **`std::strong_ordering`** — a total order where equal values are truly interchangeable (integers,
  our tick prices). `a == b` implies `f(a) == f(b)` for any `f`.
- **`std::weak_ordering`** — a total order where "equivalent" values needn't be identical
  (case-insensitive strings: `"ABC"` and `"abc"` are equivalent but distinguishable).
- **`std::partial_ordering`** — some pairs are *unordered* (floating point: any comparison with `NaN`
  is unordered). This is why `double` members yield `partial_ordering` by default.

`= default` does **memberwise lexicographic** comparison in declaration order. Note that from C++20,
`a != b`, `a > b`, etc. are *rewritten* by the compiler in terms of `<=>` and `==`, so you only define
those two. `==` is defaulted separately because equality can often be computed faster than a full
ordering (e.g. compare sizes before contents), so the language keeps them independent.

Huge boilerplate saver for value types — and an order book compares prices constantly, so this is
directly relevant.

---

## 3. Value semantics vs reference semantics

A type has **value semantics** when copying it produces an independent, equal object — like `int`:

```cpp
int a = 5; int b = a; b = 99;   // a is still 5. b is its own thing.
```

Contrast with **reference semantics** (Java/Python objects, or C++ pointers) where `b = a` makes two
names for **one** object, so mutating through `b` is visible through `a`:

```cpp
// Java-style: List b = a;  b.add(x);  // a sees x too — aliasing
```

C++ **defaults to value semantics**. This is precisely why copy/move (Module 7) matter: copying a
value type must duplicate its state so the two objects are independent. In memory:

```
value semantics (C++ default):        reference semantics (pointers/GC langs):

  a ─▶ [ state A ]                       a ─┐
  b ─▶ [ state A' ]  (independent copy)     ├─▶ [ shared state ]
                                         b ─┘   (aliased: mutate via b, a sees it)
```

Value types are:

- **Easy to reason about** — no spooky action at a distance, no aliasing surprises, no "who else has a
  reference to this?"
- **Safe to pass around and store** — you can hand out copies freely.
- **Potentially expensive to copy** — which is exactly why move semantics and `const&` exist: to keep
  the value-semantic model while avoiding needless duplication.

**Sizing rule of thumb:** small value types (`Price`, `Qty`, `OrderId` — one or two machine words)
should be cheap, `constexpr`-friendly, and passed **by value** (they fit in a register or two, so a
copy is a register move — often *cheaper* than the indirection of a reference). Big ones
(`std::vector`, `std::string`) pass by `const&`. Passing a two-word POD by `const&` is a common
pessimization: you force it onto the stack and add a pointer dereference to save nothing.

---

## 4. `operator[]`, `operator()`, and conversions

### Subscript with const/non-const overloads

```cpp
class Book {
    std::vector<Limit> levels_;
public:
    Limit&       operator[](std::size_t i)       { return levels_[i]; }  // mutable access
    const Limit& operator[](std::size_t i) const { return levels_[i]; }  // read-only access
};
```

Provide **both** a `const` and a non-`const` overload of `operator[]` so it works whether the object
is mutable or `const`. The compiler picks the right one based on the constness of the object you index
— a `const Book&` gets the `const` overload returning `const Limit&`, preventing accidental mutation
through a read-only handle. This is *const-correctness* (Module 5) applied to operators.

### `operator()` — functors (callable objects)

Overloading the call operator makes an object callable like a function. This is the workhorse for STL
customization and a **faster alternative to `std::function`**:

```cpp
struct ByPrice {
    bool operator()(const Order& a, const Order& b) const { return a.price < b.price; }
};
std::sort(v.begin(), v.end(), ByPrice{});  // functor: inlinable, zero indirection
```

Why a functor beats `std::function` here: the comparator's type is known at compile time, so the
compiler can **inline** the comparison straight into `std::sort`'s inner loop. A `std::function`
wrapper stores an erased callable behind a pointer — every comparison is an indirect call the
optimizer usually can't inline, and it may heap-allocate. In a hot sort of a million orders that's the
difference between "vectorized inner loop" and "a virtual call per compare." Lambdas are functors too:
each lambda has its own unique type, so `std::sort(v.begin(), v.end(), [](auto&a,auto&b){...})` is
just as inlinable.

### Conversion operators (use sparingly)

```cpp
struct Ticks { std::int64_t v; explicit operator std::int64_t() const { return v; } };
```

Mark conversion operators `explicit` unless you're certain an *implicit* conversion is what you want —
implicit conversions cause surprising overload resolution and hidden costs. `explicit` requires
`static_cast<std::int64_t>(t)`, which documents intent.

---

## 5. Strong typedefs — a quant-relevant pattern

Don't pass raw `int64_t` / `uint32_t` everywhere; wrap domain concepts in distinct types so the
compiler catches mix-ups that a bare integer would silently accept:

```cpp
struct OrderId { std::uint64_t v; auto operator<=>(const OrderId&) const = default; };
struct Qty     { std::uint32_t v; auto operator<=>(const Qty&)     const = default; };
struct Price   { std::int64_t  v; auto operator<=>(const Price&)   const = default; };

void cancel(OrderId id);
// cancel(qty);      // COMPILE ERROR — Qty is not OrderId, mix-up caught for free
// cancel(Qty{5});   // COMPILE ERROR
// cancel(OrderId{5});  // ok
```

With bare `uint64_t` for both, `cancel(quantity)` compiles and does the wrong thing at runtime. Strong
typedefs turn "passed quantity where an ID was expected" and "added a price to a quantity" into
**compile-time** errors, at zero runtime cost — the wrapper is a one-field struct that the compiler
lays out identically to the underlying integer. This is a beloved technique in trading systems where a
units mix-up can mean a wrong order to the exchange.

---

## 6. Never use `double` for money — the classic filter

This deserves its own section because it's a near-universal quant interview filter.

Floating point (`double`) cannot represent most decimal fractions exactly. `0.1` in binary is a
repeating fraction, stored as the nearest representable value:

```cpp
std::cout << std::setprecision(20) << 0.1;   // 0.10000000000000000555...
double s = 0.1 + 0.2;                          // 0.30000000000000004, NOT 0.3
if (0.1 + 0.2 == 0.3) { /* NEVER taken */ }   // false!
```

Accumulate thousands of these across a trading session and the rounding error is real money and
broken invariants (a book that doesn't balance, a P&L that drifts). The fix is **integer ticks or
cents**: represent a price as an integer number of the smallest tradable increment.

```cpp
struct Price { std::int64_t ticks; };   // e.g. price in 1/100th-cent ticks — EXACT
```

Integer prices are exact, compare with `==` reliably, and — bonus — can be used directly as **array
indices** into a price-level array (Module 15), which is how the fastest order books avoid a tree
lookup entirely. Saying "I'd never store price as a double; I'd use integer ticks" in the first two
minutes of a quant interview signals domain awareness instantly.

---

## 7. Common pitfalls & UB

- **Overloading an operator to do something surprising or hidden-expensive** — e.g. `operator+` that
  allocates and deep-copies. The reader sees a cheap `+`; the CPU does a `malloc`.
- **Returning a reference to a local from arithmetic** — `Price& operator+(...)` returning a
  stack temporary → dangling reference. Arithmetic returns **by value**.
- **Forgetting the `const` overload of `operator[]`** → can't index a `const` object, or accidental
  mutation slips through.
- **Defaulted `<=>` with a `double` member** silently yields `partial_ordering`; then the type won't
  work as a `std::map` key or with `std::sort` the way you expect (NaN is unordered). Use integer
  fields for ordered value types.
- **Implicit conversion operators** causing unexpected overload resolution — prefer `explicit`.
- **Asymmetric comparisons** pre-C++20 (defining `operator<` as a member so `2 < price` fails). C++20
  `<=>` + rewriting fixes this.
- **`std::function` in a hot comparator/callback** where a functor/lambda would inline — an invisible
  per-call indirection cost.

---

## 8. Quiz — tough problems

**Q1.** What does this print?

```cpp
struct P { int v; };
bool operator<(P a, P b) { return a.v < b.v; }
// no operator> defined
int main() {
    P x{3}, y{5};
    std::cout << (y > x);   // ?
}
```

**Answer:** In **C++17 and earlier: compile error** — `operator>` isn't defined and isn't derived from
`operator<`. In **C++20** it's *still* an error here, because rewriting only kicks in from
`operator<=>`/`operator==`, not from a hand-written `operator<`. Fix: define `auto operator<=>(const
P&) const = default;` and `>` (and `<`, `>=`, `<=`) are synthesized → prints `1`.

---

**Q2.** Why does `Price operator+(Price a, Price b)` take its parameters **by value** rather than
`const Price&`, while `void print(const std::string& s)` takes a reference?

**Answer:** `Price` is a one-word POD; by value it lives in a register and copying is a register move
— cheaper than the indirection of a reference. `std::string` is large and owns a heap buffer; copying
it allocates, so you pass `const&`. Rule: pass small trivially-copyable types by value, large/owning
types by `const&`. (Bonus: by-value `a` gives `operator+` a scratch copy to `+=` into and return.)

---

**Q3.** This comparator is passed to `std::sort` two ways. Which is faster and why?

```cpp
// A:
std::function<bool(const Order&, const Order&)> cmp = [](auto& a, auto& b){ return a.px < b.px; };
std::sort(v.begin(), v.end(), cmp);
// B:
std::sort(v.begin(), v.end(), [](auto& a, auto& b){ return a.px < b.px; });
```

**Answer:** **B.** In B the lambda has a unique compile-time type, so `std::sort`'s inner loop inlines
the comparison — no call overhead, vectorizable. In A the lambda is type-erased into `std::function`;
each of the O(n log n) comparisons is an indirect call the optimizer generally can't inline, plus
`std::function` may heap-allocate. Same logic, very different machine code.

---

**Q4.** Given `struct Money { double dollars; };`, why is `Money{0.1} + Money{0.2} == Money{0.3}`
dangerous, and what's the fix?

**Answer:** `0.1`, `0.2`, `0.3` aren't exactly representable in binary floating point; `0.1 + 0.2`
yields `0.30000000000000004`, so `==` is `false`. Any money-in-`double` design accumulates rounding
error. Fix: store integer cents/ticks (`std::int64_t`), which are exact and compare reliably.

---

**Q5.** You have `struct Qty { uint32_t v; };` and `struct Price { int64_t v; };` with defaulted
`<=>`. Does `qty < price` compile? Should it?

**Answer:** It does **not** compile (no heterogeneous `operator<` between `Qty` and `Price`), and
that's exactly the point of strong typedefs — comparing a quantity to a price is almost certainly a
bug, and the compiler catches it. With bare integers it would silently compile and compare garbage
semantics.

---

**Q6.** What's wrong here, and what will happen at runtime?

```cpp
struct Vec {
    int* d; std::size_t n;
    int& operator[](std::size_t i) { return d[i]; }
};
void sum(const Vec& v) {
    long s = 0;
    for (std::size_t i = 0; i < v.n; ++i) s += v[i];   // (!)
}
```

**Answer:** `v` is `const`, but the only `operator[]` is **non-const**, so `v[i]` fails to compile
(you can't call a non-const member on a const object). Fix: add `const int& operator[](std::size_t i)
const { return d[i]; }`. Real code needs both overloads for exactly this read-through-const case.

---

**Q7.** Why is `operator==` defaulted *separately* from `operator<=>` in C++20?

**Answer:** Equality is often cheaper to compute than a full ordering — e.g. two strings can be
declared unequal by comparing lengths first, without a lexicographic scan; a container can compare
sizes before elements. The language keeps `==` independent so you can (and the default does) optimize
it separately, and so that types with only equivalence (not ordering) can define `==` without a
spurious `<=>`.

---

## 9. Indian HFT interview questions

**Q1. "Why should you never use `double` for prices? What do you use instead?"** (Nearly universal —
Optiver, IMC, Graviton, AlphaGrep, Quadeye, Tower.)
*Model answer:* `double` can't represent decimal fractions exactly (`0.1 + 0.2 != 0.3`), so money
drifts and `==` is unreliable; errors accumulate over a session. Use integer ticks/cents
(`int64_t`) — exact, reliably comparable, and usable directly as array indices into a price-level
array. Signals domain awareness immediately.

**Q2. "Member vs non-member operator — when do you pick which?"**
*Model answer:* Compound assignment (`+=`, `-=`) and things that inherently mutate the left operand →
member. Symmetric binary operators (`+`, `==`, `<`) → free functions (so conversions apply to both
sides and `2 + price` works), implemented in terms of the member compound form. `<<` for streaming →
free (left operand is the stream, which you don't own).

**Q3. "What does `operator<=>` return, and why isn't it a `bool`?"**
*Model answer:* It returns an ordering category — `strong_ordering`/`weak_ordering`/
`partial_ordering` — encoding less/equal/greater (and "unordered" for partial). One three-way result
lets the compiler synthesize all four relational operators. `partial_ordering` exists because
floating point has `NaN`, which is unordered w.r.t. everything.

**Q4. "Functor vs `std::function` vs lambda as a comparator — performance?"**
*Model answer:* Functor and lambda have concrete compile-time types → the comparator inlines into the
algorithm's hot loop, zero indirection. `std::function` type-erases → indirect call per comparison,
possible heap allocation, no inlining. In HFT hot paths use templated callables / lambdas, not
`std::function`.

**Q5. "How would you prevent passing an order quantity where an order ID is expected — at compile
time?"**
*Model answer:* Strong typedefs — wrap each concept in a distinct one-field struct (`OrderId`, `Qty`,
`Price`), each with defaulted `<=>`/`==`. Mixing them becomes a type error at zero runtime cost, since
the wrapper lays out identically to the raw integer.

**Q6. "Give an example of operator overloading you'd consider bad taste."**
*Model answer:* Anything that surprises the reader or hides cost: `operator+` that mutates its
operands, `operator+` on a `Book` that inserts an order (should be a named `add`), an implicit
conversion operator that silently allocates, or a comparison that isn't a real ordering. The bar:
would a reader correctly predict the cost and effect from the symbol alone?

---

## 10. In the order book

`Price`, `Qty`, `OrderId` are strong value types with defaulted `<=>`/`==`, passed **by value** (each
is one machine word). Price comparison drives the AVL-tree / price-array ordering of levels, and
because prices are **integer ticks** they double as array indices into a dense price-level array
(Module 15) — no float error, and O(1) level lookup instead of a tree walk. A `ByPrice` functor (not
`std::function`) orders resting orders so the comparison inlines into the sort. `Book::operator[]` has
const/non-const overloads so read-only views can't mutate the book. Every one of these is a deliberate
value-semantics + cost-awareness choice, which is exactly what the interview is checking.

---

## Key takeaways

- Operators are functions with syntax: **compound-assign as members, symmetric binary ops as free
  functions** built on them; arithmetic returns by value, assignment returns `*this&`.
- **C++20 `<=>` + defaulted `==`** generate all comparisons from one/two lines; the ordering category
  (`strong`/`weak`/`partial`) tells you what kind of order you have — `double` members force `partial`.
- **Value semantics** (copies are independent) is the C++ default and the reason copy/move exist; pass
  small PODs by value, large/owning types by `const&`.
- **Never store money as `double`** — use integer ticks/cents; they're exact, comparable, and usable
  as array indices. This is a standard quant filter.
- Prefer **functors/lambdas over `std::function`** on hot paths (inlining), provide **const +
  non-const `operator[]`**, and use **strong typedefs** to catch unit mix-ups at compile time.

**Next:** [09 — Move semantics & rvalue references (deep)](09-move-semantics.md)
