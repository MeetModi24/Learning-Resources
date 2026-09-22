# Module 6 — Classes: constructors, destructors, `this`, access control

A class bundles **data** (members) with **behavior** (methods) and controls **who can touch what**.
This module is the machinery; Module 7 (RAII) is the philosophy that gives it purpose. For HFT the
class is not just an abstraction tool — its *memory layout* (member order, padding, alignment,
whether it has a vtable) directly determines cache behavior and latency, so interviewers expect you
to reason about a class as bytes, not just as an API. We build the concept from zero and then look at
exactly how an object sits in memory.

---

## 1. Anatomy of a class

```cpp
class Order {
private:                 // hidden from outside — the default for `class`
    std::uint64_t id_;
    double        price_;
    std::uint32_t qty_;

public:                  // the interface
    Order(std::uint64_t id, double price, std::uint32_t qty)
        : id_{id}, price_{price}, qty_{qty}   // member initializer list
    {}

    double price() const { return price_; }          // accessor (const)
    void   reduce(std::uint32_t by) { qty_ -= by; }   // mutator
};
```

- `class` members default to **private**; `struct` members default to **public**. Otherwise the two
  are *identical* — same everything. Convention: `struct` for plain data aggregates, `class` when you
  enforce invariants.
- The trailing underscore (`id_`) is a common convention distinguishing members from locals/params.
- **Data + behavior + access control** is the whole idea: the private data can only be reached through
  the public methods, so the class controls its own invariants.

---

## 2. How an object looks in memory

This is the part beginners never see and interviewers always probe. An object is just its members
laid out **in declaration order**, with the compiler inserting **padding** so each member sits at an
address that's a multiple of its alignment. There is **no hidden overhead** unless the class has
virtual functions (then one vtable pointer is added — Section 8).

Take a deliberately badly-ordered struct:

```cpp
struct Bad {
    char   side;   // 1 byte,  align 1
    double price;  // 8 bytes, align 8
    int    qty;    // 4 bytes, align 4
};
```

```
offset:  0      1 .. 7            8 .. 15          16 .. 19   20 .. 23
         [side] [ 7B padding  ]   [   price    ]   [  qty  ]  [ 4B tail pad ]
         └ align: double needs offset%8==0, so 7 bytes wasted after side ┘
sizeof(Bad) == 24   (tail padding rounds size up to a multiple of the largest align, 8)
```

Now reorder largest-to-smallest:

```cpp
struct Good {
    double price;  // 8
    int    qty;    // 4
    char   side;   // 1
};
```

```
offset:  0 .. 7        8 .. 11   12    13 .. 15
         [   price ]   [ qty ]   [side][ 3B tail pad ]
sizeof(Good) == 16    (8 bytes saved — one third smaller, same fields)
```

**Rule interviewers love:** order members **largest alignment first** to minimize padding. On the hot
path a 16-byte order vs a 24-byte order means more orders per cache line (Module 15), so this is a
real latency lever, not pedantry.

- **`alignof(T)`** = the required address multiple (usually `sizeof` the largest scalar member).
- **`sizeof(T)`** is always a multiple of `alignof(T)` so arrays stay aligned.
- **`this`** is just the address of that byte block (Section 4).

You can force layout with `alignas` (e.g. `alignas(64)` to put a hot member on its own cache line —
Module 15/16) or ask the compiler to skip padding with `#pragma pack` (rarely, for wire formats — it
costs unaligned-access performance).

---

## 3. Constructors and the member initializer list

The `: id_{id}, price_{price}` part is the **member initializer list** — it *initializes* members
directly, before the constructor body runs. Prefer it over assigning in the body:

```cpp
Order(...) { price_ = price; }   // WORSE: price_ is default-constructed, THEN assigned (two steps)
Order(...) : price_{price} {}    // BETTER: directly initialized once
```

For scalars the difference is negligible, but for class-type members (a `std::string`, another object)
the body-assignment form default-constructs then copy-assigns — two operations where one suffices. For
**`const` members and references**, the init list is **mandatory** — they can't be assigned after
construction, only initialized.

> **Members are initialized in *declaration order*, not init-list order.** Listing them out of order
> compiles but is a warning-generating bug if one member's initializer reads another. Example:
> `int a_, b_;` with `: b_{5}, a_{b_}` initializes `a_` *first* (declaration order) from an
> uninitialized `b_` → garbage.

### Kinds of constructors

```cpp
class Widget {
public:
    Widget() = default;                 // default ctor (no args)
    Widget(int x);                      // parameterized
    Widget(const Widget&) = default;    // copy ctor (Module 7)
    Widget(Widget&&) = default;         // move ctor (Modules 4, 9)
    explicit Widget(double d);          // explicit: no implicit conversions
};
```

`explicit` prevents surprising implicit conversions:

```cpp
void take(Widget w);
take(3.0);   // if the double ctor is explicit → ERROR (good: no silent double→Widget)
```

**Mark single-argument constructors `explicit`** unless you *want* implicit conversion. Silent
conversions are a classic source of "why did my `Order` get constructed from an `int`?" bugs.

---

## 4. The `this` pointer

Inside a non-static method, `this` is a pointer to the object the method was called on — literally the
address of that byte block from Section 2:

```cpp
class Order {
    int qty_;
public:
    Order& setQty(int qty) {
        this->qty_ = qty;   // this-> optional here; needed only if a param shadows a member
        return *this;       // return the object itself → enables chaining
    }
};
// order.setQty(5).setQty(10);   // fluent chaining via return *this
```

Under the hood, a non-static method is a free function with a hidden first parameter: `order.reduce(5)`
compiles roughly to `Order::reduce(&order, 5)`, and inside, `this == &order`. A `const` method receives
`const Order*` as `this` — which is *why* it can't modify members. Static methods (Section 6) have
**no** `this`.

---

## 5. Destructors

`~ClassName()` is the cleanup half of an object's life (constructor acquires, destructor releases). No
return type, no parameters, exactly **one** per class. You almost never call it explicitly; the
compiler inserts the call. This is where RAII cleanup lives:

```cpp
class FileHandle {
    std::FILE* f_;
public:
    explicit FileHandle(const char* path) : f_{std::fopen(path, "r")} {}
    ~FileHandle() { if (f_) std::fclose(f_); }   // guaranteed cleanup
};
```

### When it runs (automatically, on every exit path)

1. **Stack object leaves scope** — at the closing `}`. This *is* the "automatic cleanup at scope exit"
   from Module 1; the destructor is what runs.
2. **Heap object is `delete`d** — `delete` runs the destructor *first*, then releases the raw memory.
3. **The enclosing object is destroyed** — each member's destructor runs too (order below).
4. **Exception unwinding** — as an exception propagates, destructors of all locals between `throw` and
   `catch` fire. This is what makes RAII exception-safe (Module 14).

The guarantee across all four: cleanup happens no matter *how* you leave — normal return, early
return, or exception. You write the `delete`/`fclose` once; it can't be skipped.

### Destruction order — reverse of construction

1. **Locals** are destroyed in the **reverse** order they were created (it's a stack — LIFO):
   `a; b; c;` → destroyed `c, b, a`.
2. **Members** are destroyed *after* the enclosing destructor's body runs, in **reverse declaration
   order**. You usually write nothing to clean them up — their own destructors run automatically. If
   every member is self-cleaning, your destructor body is often empty (or unneeded).
3. **Base class** destructor runs *after* the derived one (Module 11): derived cleanup first, then
   base.

### Virtual destructors (deleting through a base pointer)

If you `delete` a derived object through a **base pointer** and the base destructor isn't `virtual`,
only the base part is destroyed — the derived part leaks. UB.

```cpp
struct Base { virtual ~Base() = default; };  // virtual → correct polymorphic cleanup
struct Derived : Base { std::vector<int> buf; };

Base* p = new Derived;
delete p;   // virtual:  ~Derived (frees buf) then ~Base.  ✅
            // non-virtual: only ~Base runs → buf LEAKS.  ✗ UB
```

Rule: **any class meant to be inherited from and deleted polymorphically needs a `virtual`
destructor.** (Full treatment in Module 11.) Corollary: adding `virtual` also adds a vtable pointer to
every object (Section 8), so *don't* make destructors virtual on hot POD types you never delete
polymorphically.

### "If I don't write one, won't it free the memory anyway?"

Only if your members are **self-cleaning**. The compiler always generates a destructor when you don't
write one — but the generated destructor does exactly one thing: **destroy each member.** And
"destroy" means *call that member's destructor.* So the answer depends entirely on what the member is:

```cpp
class Leaks {
    int* data_;                    // RAW pointer
public:
    Leaks(std::size_t n) : data_{new int[n]} {}
    // no ~Leaks()
};   // generated dtor destroys data_ → but destroying an int* does NOTHING to
     // the heap array. delete[] is never called. LEAK.
```

A raw pointer is **trivial to destroy** — "destroying" it just discards the 8-byte pointer variable;
the heap block it points at is untouched. The compiler can't know `data_` *owns* that block (it might
be a borrowed/non-owning pointer, where freeing would be a bug), so it does nothing. **A class that
directly owns a raw resource MUST hand-write its destructor.**

Contrast with self-cleaning members:

```cpp
class Fine {
    std::vector<int> data_;   // owns a heap array, but has its OWN destructor
public:
    Fine(std::size_t n) : data_(n) {}
    // no ~Fine() needed
};   // generated dtor destroys data_ → runs vector's destructor → delete[]. Freed. ✅
```

| Member | Compiler-generated dtor frees it? |
|---|---|
| Raw owning pointer (`int*` + `new`) | ❌ **No — leaks.** Write `delete[]` yourself. |
| `std::vector` / `std::string` / `unique_ptr` | ✅ Yes — their own destructors run |
| Scalar (`int`, `double`) | ✅ nothing to free anyway |

The rule underneath: **destroying a member = calling that member's destructor.** Raw pointers have no
meaningful one; RAII types do. This is the core reason to prefer `vector`/smart pointers over
`new`/`delete` — with them, "no destructor needed" is genuinely safe; with raw pointers, a forgotten
destructor is a silent leak. (And once a member manages the resource, you often need *no* destructor,
copy ctor, or move ctor at all — the "Rule of Zero," Module 7.)

---

## 6. Static members

Belong to the class, not any instance — one shared copy for the whole program, living in static
storage (not inside any object):

```cpp
class Order {
    static inline std::uint64_t nextId_ = 1;  // C++17 inline static: one shared counter
public:
    static std::uint64_t allocateId() { return nextId_++; }
};
```

Static methods have **no `this`** and can't touch non-static members (there's no object to touch).
`static inline` (C++17) lets you define and initialize the static member right in the class — before
C++17 you needed a separate out-of-class definition. In memory: `nextId_` is one fixed location; every
`Order` object shares it and none of them *contains* it.

---

## 7. Access control & `friend`

`private`/`protected`/`public` control access. `friend` grants a *specific* function or class access
to privates — use sparingly (it breaks encapsulation), most commonly for symmetric operators (Module
8) like `operator<<` or `operator==` that need to read both operands' internals. Note: access control
is **compile-time only** — it doesn't change layout or add runtime cost; a `private` member sits in
memory exactly where a `public` one would.

---

## 8. Layout with virtual functions (a preview of Module 11)

The moment a class has *any* virtual function, the compiler adds a hidden **vptr** (vtable pointer) —
usually as the first 8 bytes of the object — pointing to a per-class table of function addresses:

```cpp
struct Base { virtual void f(); int x; };   // now has a vptr
```

```
offset:  0 .. 7          8 .. 11    12 .. 15
         [   vptr    ]   [   x   ]   [ pad ]
                │
                ▼
          vtable (one per class, in .rodata): [ &Base::f ]
sizeof grew by 8 bytes; a virtual call = load vptr → index table → indirect call
```

This is why a POD `Order` should have **no** virtual functions on the hot path: the extra 8 bytes
shrink orders-per-cache-line, and every virtual call is an indirect branch the CPU may mispredict.
Interviewers ask "what does adding `virtual` cost?" — the answer is *8 bytes per object + an indirect
call + lost inlining*, covered fully in Module 11.

---

## Common pitfalls & UB

- **Member init in the wrong order** — always declaration order; out-of-order init lists reading each
  other yield garbage.
- **Missing destructor on a raw-owning class** — silent leak; missing *virtual* destructor on a base →
  UB on polymorphic delete.
- **Non-`explicit` single-arg ctor** — enables surprising implicit conversions.
- **Throwing from a destructor** — during stack unwinding this calls `std::terminate` (Module 14).
- **Assuming access control affects layout** — it doesn't; `private`/`public` is compile-time only.
- **Padding surprises** — `sizeof` isn't the sum of member sizes; reorder to minimize it.
- **Using an object before its base/members are constructed** — calling a virtual from a base
  constructor dispatches to the *base* version (the derived part doesn't exist yet).

---

## Quiz — tough problems

**Q1.** What is `sizeof(S)`?
```cpp
struct S { char a; int b; char c; };
```
**Answer:** 12. Layout: `a`@0, 3 pad, `b`@4, `c`@8, 3 tail pad → 12 (multiple of `int`'s align 4).
Reordered `int b; char a; char c;` would be 8.

**Q2.** Predict the output:
```cpp
struct S { int a, b; S(): b(1), a(b) {} };   // note init-list order
S s; std::cout << s.a << " " << s.b;
```
**Answer:** `<garbage> 1`. Members init in **declaration order** (`a` then `b`), so `a(b)` reads `b`
*before* `b` is initialized → indeterminate. `b` then gets 1. Compilers warn with `-Wreorder`.

**Q3.** Does this leak? Why?
```cpp
class C { int* p_; public: C(): p_(new int[100]) {} };
{ C c; }
```
**Answer:** Yes — leaks 400 bytes. No user destructor, and the generated one destroys `p_` (a raw
pointer), which does nothing to the heap block. Add `~C(){ delete[] p_; }` or use `std::vector<int>`.

**Q4.** What does adding a single `virtual` function do to `sizeof` and to a call?

**Answer:** Adds an 8-byte vptr to every object (typically at offset 0), and turns calls to virtual
methods into an indirect call through the vtable (load vptr → index → call), losing inlining and
risking branch misprediction.

**Q5.** Is `this` ever `nullptr`? What happens if you call a method on a null pointer?

**Answer:** Calling `p->method()` with `p == nullptr` is **UB**. It may "work" if the method never
touches `this` (never reads a member / isn't virtual), but that's undefined and must not be relied on.

**Q6.** Spot the bug:
```cpp
struct Base { Base(){ init(); } virtual void init(){} };
struct Derived : Base { int* buf_; void init() override { buf_ = new int[10]; } };
Derived d;
```
**Answer:** During `Base`'s constructor the object is still a `Base` — the virtual call to `init()`
dispatches to `Base::init()`, **not** `Derived::init()`. `buf_` is never allocated. Never call virtuals
from constructors expecting derived behavior.

**Q7.** Why is `struct` vs `class` "only the default access"?

**Answer:** Because it *is* — the sole difference is `struct` defaults to `public` members and public
inheritance, `class` to `private`. Everything else (methods, ctors, inheritance, templates) is
identical. Interviewers ask this to check you're not carrying C baggage.

**Q8.** When is a hand-written destructor a mistake (Rule of Zero)?

**Answer:** When all members are self-cleaning RAII types (`vector`, `string`, `unique_ptr`). Writing
`~C() {}` there is at best redundant and at worst *suppresses* the implicitly-generated move
operations (Module 7), silently degrading moves to copies. Prefer no destructor.

**Q9.** Order these members to minimize `sizeof`: `char c; double d; int i; short s;`

**Answer:** `double d; int i; short s; char c;` → `d`@0(8), `i`@8(4), `s`@12(2), `c`@14(1), 1 tail pad
→ 16 bytes. The naive order would waste more on padding.

---

## Indian HFT interview questions

**Q (Tower Research / Graviton, Gurgaon): "How is a C++ object laid out in memory? What determines its
size?"**
Members are laid out in declaration order with padding so each is aligned to its `alignof`; `sizeof`
is rounded up to a multiple of the largest member's alignment. A vptr (8 bytes) is prepended only if
the class has virtual functions. Reordering members largest-alignment-first minimizes padding — which
matters because a smaller order means more orders per 64-byte cache line.

**Q (Optiver / IMC): "What's the cost of making a class polymorphic (adding `virtual`)?"**
8 bytes per object for the vptr, an indirect call through the vtable per virtual invocation (load
vptr → index → call), loss of inlining, and possible branch misprediction. On a hot POD type you
avoid it; use CRTP or non-virtual designs (Module 17) instead.

**Q (Quadeye / AlphaGrep): "In what order are constructors and destructors called for members and base
classes?"**
Construction: base class first, then members in *declaration order*, then the constructor body.
Destruction is the exact reverse: body, then members in reverse declaration order, then base. Locals
in a scope are destroyed LIFO.

**Q (Jump / Da Vinci): "Why must a base class destructor be `virtual`?"**
So that `delete base_ptr;` where `base_ptr` actually points to a derived object runs the *derived*
destructor (and its members' destructors) before the base's. Without it, only the base part is
destroyed → derived resources leak → UB. Only needed when you delete polymorphically.

**Q (Squarepoint / Millennium): "Member initializer list vs assignment in the constructor body —
which and why?"**
Initializer list: it *initializes* each member once, in declaration order, and is *mandatory* for
`const`/reference members and for members with no default constructor. Body assignment
default-constructs then assigns — two operations, wasteful for class-type members.

**Q (NK Securities / Mansard): "What's `explicit` for?"**
It stops a single-argument constructor from being used for implicit conversions, preventing surprising
silent conversions (e.g. an `int` turning into your class in a function call). Default to `explicit`
on single-arg constructors.

**Q (Hudson River / Qube): "When would you deliberately use a plain `struct` with public members and no
methods?"**
For a hot, data-only POD like `Order` where tight layout and trivial copyability matter more than
encapsulation. No invariants to protect, no vtable, predictable layout — you trade a little safety for
control and speed. Judgment, not dogma.

---

## In the order book

`Order`, `Limit` (a price level), and `Book` are classes. `Order` is nearly a **POD struct** (id,
price, qty, side, links) with members ordered to minimize padding and *no* virtual functions — so it
stays small (more orders per cache line) and trivially relocatable in the pool. `Book` enforces
invariants (price-time priority) behind a clean public interface (`addLimitOrder`, `cancel`,
`modify`). A `static inline` counter hands out monotonically increasing order IDs.

Destructor angle: `OrderPool`'s `storage` vector has **one** destructor call that frees the whole slab
at shutdown — and during trading **no per-order destructors fire** (you recycle slots via the
free-list, you don't destroy `Order`s). Fewer destructor calls on the hot path is part of why it's
fast. The same logic drives the move ctor's `other.data = nullptr` in Module 4: you empty the source
precisely so *its* destructor frees nothing (no double free). And because `Order` is all
self-cleaning/scalar members, it obeys the **Rule of Zero** (Module 7) — no hand-written destructor,
copy, or move needed.

---

## Key takeaways

- A class is **data + behavior + access control**; `class` vs `struct` differ *only* in default access
  (private vs public).
- An object is its members laid out in **declaration order** with **padding** for alignment; `sizeof`
  rounds up to the largest member's alignment. Order members largest-first to shrink objects — a real
  cache/latency lever.
- Prefer the **member initializer list**; members init in **declaration order** (not list order), and
  it's mandatory for `const`/reference members. Mark single-arg ctors `explicit`.
- Destructors run on **every** exit path (scope exit, `delete`, member/base destruction, exception
  unwinding), in **reverse** construction order. A raw-owning class **must** hand-write one; a base
  deleted polymorphically **must** have a `virtual` one.
- The compiler-generated destructor only *calls each member's destructor* — self-cleaning members (RAII
  types) make it safe (Rule of Zero); raw owning pointers make it a silent leak.
- Adding `virtual` costs **8 bytes/object (vptr) + an indirect call + lost inlining** — avoid it on hot
  POD types.

**Next:** [07 — RAII & the Rule of 0/3/5](07-raii-rule-of-five.md) — the philosophy that turns
constructors and destructors into leak-proof resource management.
