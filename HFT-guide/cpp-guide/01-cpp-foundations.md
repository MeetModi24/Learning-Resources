# Module 1 — C++ foundations: build model, types, initialization & casts

C++ is unusual because the compiler needs to understand both the **meaning of your program** and the
**physical representation of its data**. Before reasoning about pointers, lifetime, templates, or
cache lines, you need a small set of foundations: how source files become a binary, what the basic
types promise, how initialization differs from assignment, and which conversions are safe.

This is not a syntax catalogue. It is the minimum language model that later modules assume, with the
parts that repeatedly appear in systems and HFT interviews.

---

## 1. From source file to executable

A C++ build is best understood as a pipeline:

```text
source (.cpp)
    │  preprocessor: expands #include and macros
    ▼
translation unit
    │  compiler: parses, type-checks, optimizes, emits machine code
    ▼
object file (.o)
    │  linker: resolves names and combines object files/libraries
    ▼
executable
```

Useful commands make each stage visible:

```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic -E main.cpp -o main.i  # preprocess only
g++ -std=c++20 -Wall -Wextra -Wpedantic -S main.cpp -o main.s  # emit assembly
g++ -std=c++20 -Wall -Wextra -Wpedantic -c main.cpp -o main.o  # compile only
g++ main.o book.o -o matching_engine                       # link
```

Use `-g` for debug information and sanitizers while testing; use an optimization level such as
`-O2` for realistic performance measurements. Never infer production latency from an unoptimized
debug build.

### Translation units, declarations, and definitions

After preprocessing, one `.cpp` file plus all text included into it is a **translation unit**. Each
translation unit is compiled independently. This explains many otherwise mysterious rules:

```cpp
// price.hpp
#pragma once

struct Price { long ticks; };       // complete class definition: normally in the header
long midpoint(Price, Price);         // declaration: tells callers the signature

// price.cpp
#include "price.hpp"
long midpoint(Price a, Price b) {    // definition: supplies the function body
    return a.ticks + (b.ticks - a.ticks) / 2;
}
```

- A **declaration** introduces a name and type.
- A **definition** creates the entity or supplies its implementation.
- The **One Definition Rule (ODR)** broadly requires one program-wide definition for ordinary
  non-inline functions and variables, while class and template definitions may appear identically
  in many translation units.
- Include guards or `#pragma once` prevent a header from being expanded twice in one translation
  unit. They do not, by themselves, solve every program-wide ODR violation.
- Template definitions normally live in headers because the compiler must see the full recipe at
  the point of instantiation (Module 12).

`inline` primarily permits the same identical function or variable definition in multiple
translation units; it does **not** force machine-code inlining. The optimizer makes that decision.

### Prefer language features to macros

The preprocessor performs token substitution before C++ type checking:

```cpp
#define SQUARE(x) x * x
int n = SQUARE(2 + 3);          // becomes 2 + 3 * 2 + 3 == 11, not 25

constexpr int square(int x) noexcept { return x * x; }  // typed, scoped, debuggable
```

Use macros mainly for conditional compilation and include guards. Prefer `constexpr`, functions,
templates, and typed constants for program logic.

---

## 2. Types: representation is part of the contract

C++'s built-in integer types have minimum ranges, not universally fixed sizes. Check the target
platform when layout matters:

```cpp
#include <cstdint>
#include <limits>

int local_count = 0;                // natural type for ordinary local arithmetic
std::uint32_t quantity = 0;         // exact width when this type exists
std::int64_t price_ticks = 0;       // explicit representation for stored/wire data

static_assert(sizeof(std::uint32_t) == 4);
static_assert(std::numeric_limits<std::int64_t>::digits == 63);
```

Practical rule:

- Use `int` for ordinary local arithmetic when its range is enough.
- Use fixed-width types for protocols, files, shared-memory layouts, hashes, and compact hot structs.
- Use `std::size_t` for object sizes and container indices, but remember it is unsigned.
- Represent prices as integer ticks, not binary floating point (Module 10).

### Signed/unsigned and overflow

Signed integer overflow is **undefined behavior**. Unsigned arithmetic wraps modulo `2^N`, which is
well-defined but can still be a logic bug:

```cpp
std::size_t remaining = 0;
--remaining;                         // wraps to a very large value

int max = std::numeric_limits<int>::max();
// ++max;                            // signed overflow: UB
```

Mixed signed/unsigned comparisons are a frequent trap because the signed operand may be converted to
unsigned. Keep domains consistent and enable compiler warnings.

### Object type is not just a bag of bits

Two types of the same size do not become interchangeable merely because their bytes fit. Alignment,
lifetime, aliasing, and valid value representations matter. Modules 2, 7, and 16 build the full
model; for now, do not read an arbitrary byte buffer by casting it to an unrelated pointer type.

---

## 3. Initialization is not assignment

Initialization creates an object with its first value. Assignment changes an object that already
exists:

```cpp
std::string a{"BUY"};       // construction / initialization
std::string b;              // default construction
b = a;                      // copy assignment
```

Prefer brace initialization for most values:

```cpp
int count{};                // zero
double px{101.25};
std::vector<int> levels{1, 2, 3};

// int truncated{3.14};     // compile error: narrowing conversion
```

Important forms:

| Form | Example | Main property |
|---|---|---|
| default initialization | `int x;` | built-in local `x` is indeterminate |
| value initialization | `int x{};` | zero for scalar types |
| direct initialization | `Widget w(3);` | direct constructor selection |
| list initialization | `Widget w{3};` | rejects narrowing; prefers initializer-list overloads |
| copy initialization | `Widget w = 3;` | cannot use an `explicit` constructor |

The exception to “braces are always obvious” is a type with an
`std::initializer_list` constructor:

```cpp
std::vector<int> a(5, 1);   // five elements, all 1
std::vector<int> b{5, 1};   // two elements: 5 and 1
```

Read the API; punctuation can select a different constructor.

### The most vexing parse

```cpp
Widget w();                 // declares a function named w returning Widget
Widget w2{};                // constructs an object
```

When syntax can be parsed as a declaration, C++ traditionally chooses the declaration. Braces make
the intention unambiguous.

---

## 4. `auto`, `decltype`, and type aliases

`auto` asks the compiler to deduce a type from an initializer; it does not make C++ dynamically
typed:

```cpp
auto qty = 100u;            // unsigned int, fixed at compile time
const auto& order = book.bestOrder();
```

By-value `auto` drops top-level `const` and references. Write `auto&`, `const auto&`, or `auto&&`
when reference behavior matters. `decltype(expr)` asks for the type described by an expression and
obeys more exact rules:

```cpp
int x = 0;
decltype(x) a = 1;          // int
decltype((x)) b = x;        // int&: parenthesized x is an lvalue expression
```

This `decltype((x))` distinction is an interview favorite and becomes useful in generic code.

Prefer `using` for aliases:

```cpp
using OrderId = std::uint64_t;                    // alias, NOT a new type
using Callback = void (*)(OrderId);
template <class T> using Buffer = std::vector<T>; // alias template
```

An alias improves readability but does not prevent mixing values: `OrderId` above is still exactly
`std::uint64_t`. Use a wrapper/strong typedef pattern when the compiler must distinguish `OrderId`
from `ClientId` (Module 10).

---

## 5. Conversions and the four named casts

Implicit conversions are convenient but can lose information:

```cpp
double price = 101.75;
int ticks = price;                  // allowed, silently truncates
int checked{price};                 // rejected as narrowing
```

Named C++ casts document which kind of conversion you intend:

### `static_cast` — checked, related conversions

```cpp
auto ticks = static_cast<std::int64_t>(price / tick_size);
auto raw_side = static_cast<std::uint8_t>(Side::Buy);
```

Use it for numeric conversions, enum conversions, and compile-time-known up/down conversions where
the program already guarantees the dynamic type. A wrong unchecked downcast later used as the wrong
type is UB.

### `dynamic_cast` — checked polymorphic conversion

Used with polymorphic class hierarchies (Module 8). A failed pointer cast returns `nullptr`; a failed
reference cast throws `std::bad_cast`. It uses runtime type information and does not belong in a hot
matching loop.

### `const_cast` — change cv-qualification

```cpp
void legacy(char*);
const char* text = "ABC";
// legacy(const_cast<char*>(text));  // only safe if legacy truly does not modify it
```

Removing `const` does not make an originally const object modifiable. Writing through that result is
UB. `const_cast` is mainly for carefully audited legacy interfaces.

### `reinterpret_cast` — low-level representation conversion

This expresses “treat these bits/address as another representation,” with almost no semantic safety.
It does not waive alignment, lifetime, or strict-aliasing rules. Prefer `std::bit_cast` for copying
equal-sized trivially copyable representations, and `std::memcpy`/explicit parsing for byte buffers.

Avoid C-style casts: one spelling may attempt several different conversions, hiding intent during
review.

---

## 6. Enumerations: names, scope, and representation

Prefer scoped enums for finite domain states:

```cpp
enum class Side : std::uint8_t { Buy, Sell };
enum class OrderType : std::uint8_t { Market, Limit, IOC };

Side side = Side::Buy;
// if (side == 0) {}         // rejected: no accidental integer comparison
```

Compared with an unscoped `enum`, `enum class`:

- keeps enumerators inside the enum's scope;
- prevents implicit integer conversions;
- prevents accidentally comparing unrelated enum types;
- can state an underlying type for compact layout or a protocol contract.

The underlying type controls storage, not semantic validity: receiving byte value `99` still needs
validation before treating it as a valid `Side`.

For flags, define explicit bitwise operators for the scoped enum rather than falling back to an
unscoped enum solely for implicit arithmetic.

---

## 7. Storage duration, scope, and linkage — the map

These terms answer different questions:

- **scope**: where can this name be used?
- **storage duration**: when does the object exist?
- **linkage**: can declarations in different scopes/translation units denote the same entity?

The four storage durations are:

1. **automatic** — ordinary locals, created/destroyed with the block;
2. **static** — globals, namespace variables, and static locals, lasting for the program;
3. **thread** — `thread_local`, one instance per thread;
4. **dynamic** — storage obtained explicitly at runtime, with separately managed lifetime.

Module 2 turns this map into the full memory/lifetime model. Two rules belong here:

```cpp
// header
inline constexpr std::size_t max_orders = 1u << 20; // safe header definition

// source file
namespace { int file_only_counter = 0; }            // internal linkage
```

- Prefer an unnamed namespace for source-file-local entities.
- Prefer function-local statics when initialization order between translation units would otherwise
  matter. Since C++11 their initialization is thread-safe, though the first call may pay a guard.

---

## 8. Attributes worth recognizing

Attributes communicate intent to the compiler without changing the core type system:

```cpp
[[nodiscard]] bool submit(const Order&);
[[maybe_unused]] constexpr int debug_tag = 7;

if (LIKELY_valid(order)) [[likely]] {
    // common path
}
```

- `[[nodiscard]]` warns when a result such as an error/status is ignored.
- `[[maybe_unused]]` suppresses a deliberate unused warning.
- `[[likely]]` / `[[unlikely]]` are layout hints, not guarantees; measure before using them
  (Module 19).
- `alignas` changes alignment and is covered with cache lines and false sharing in Module 16.

Never suppress a warning merely to make the build green. First decide whether it revealed a real
bug.

---

## Common pitfalls & UB

- Reading an uninitialized automatic scalar is UB; initialize deliberately.
- Signed overflow is UB; unsigned underflow wraps and may create huge indices.
- C-style casts hide whether qualification, hierarchy, or representation is being changed.
- `reinterpret_cast` does not make misaligned or type-punned access valid.
- `using OrderId = uint64_t` is an alias, not a strong type.
- `std::vector<int>(5, 1)` and `{5, 1}` call different constructors.
- Defining a non-inline function in a widely included header creates ODR/linker failures.
- Macros ignore types, scopes, evaluation count, and normal debugging rules.

---

## Quiz — tough problems

**Q1.** Why can two `.cpp` files compile successfully and still fail at the link step?
**Answer:** Each translation unit is compiled independently. Both can contain valid declarations,
yet the linker may find a referenced definition missing or find multiple forbidden definitions of
the same external symbol.

**Q2.** What are the values/sizes here?
```cpp
std::vector<int> a(4, 7);
std::vector<int> b{4, 7};
```
**Answer:** `a` has four elements, all `7`; `b` has two elements, `4` and `7`. Parentheses select the
count/value constructor; braces prefer the initializer-list constructor.

**Q3.** Why is `using Price = std::int64_t;` insufficient to stop a quantity being passed as a price?
**Answer:** An alias is the same type, not a new one. Use a wrapper such as `struct Price { int64_t
ticks; };` (with deliberate operators) for type-level separation.

**Q4.** Which cast should parse four bytes from a network buffer into an integer?
**Answer:** None by dereferencing a cast pointer. Decode bytes explicitly (including endianness), or
copy into a suitably typed object with a valid representation. `reinterpret_cast` does not solve
alignment, lifetime, aliasing, or byte-order problems.

**Q5.** What is the difference between `decltype(x)` and `decltype((x))` for `int x`?
**Answer:** `decltype(x)` is the declared type `int`; `decltype((x))` observes that `(x)` is an lvalue
expression and yields `int&`.

**Q6.** Why is `inline` not a performance promise?
**Answer:** In the language it mainly relaxes the ODR for identical definitions. The optimizer may
inline a function without the keyword or refuse to inline one that has it.

---

## HFT interview questions

- **“Walk me from `.cpp` to executable.”** Preprocess each source into a translation unit, compile
  each independently to an object file, then link definitions and libraries into the binary.
- **“Why integer ticks instead of `double` prices?”** Exact equality/order, deterministic arithmetic,
  stable hashing and serialization, and no binary floating-point rounding surprises.
- **“`static_cast` vs `dynamic_cast` vs `reinterpret_cast`?”** Related checked conversion at compile
  time; checked polymorphic conversion at runtime; low-level representation/address conversion with
  almost no safety.
- **“What is an ODR violation?”** A program has a forbidden number of definitions or non-identical
  definitions where the standard requires identity; symptoms range from linker errors to ill-formed
  programs with subtle behavior.
- **“Why use `enum class Side : uint8_t`?”** Domain type safety and scoped names, with an explicit
  compact representation useful in hot structs and wire formats.
- **“What flags do you enable?”** A modern language standard, strong warnings, debug info and
  sanitizers in testing; optimized builds plus profiling/assembly inspection for performance.

---

## In the order book

- `Price`, `Quantity`, and `OrderId` get explicit integer representations; the strongest design wraps
  semantically different values so they cannot be mixed accidentally.
- `Side` and `OrderType` are compact scoped enums rather than magic integers.
- Protocol bytes are decoded explicitly instead of type-punning a receive buffer.
- Status-returning hot-path methods are `[[nodiscard]]`, making ignored rejects visible at compile
  time.
- Public declarations live in headers, implementation in source files, and templates stay visible
  where instantiated. This keeps the build model predictable as the engine grows.

---

## Key takeaways

- C++ builds one translation unit at a time, then links; declarations, definitions, linkage, and the
  ODR explain the boundary.
- Choose types by domain and representation. Fixed-width integers matter at storage/protocol/layout
  boundaries; integer ticks are the correct price representation.
- Initialization and assignment are different operations; braces prevent narrowing but can prefer an
  initializer-list constructor.
- `auto` is static deduction, `decltype` preserves expression-category details, and aliases do not
  create new types.
- Use named casts to make intent reviewable, and remember that no cast cancels alignment, lifetime,
  or aliasing requirements.
- Prefer `enum class`, typed constants, functions, and templates over magic integers and macros.

**Next:** [Module 2 — Memory & object lifetime](02-memory-lifetime.md) — where objects live, when
they begin and end, and why lifetime errors become undefined behavior.
