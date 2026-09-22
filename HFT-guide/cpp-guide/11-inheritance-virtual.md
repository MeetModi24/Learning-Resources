# Module 11 — Inheritance, virtual functions, vtables — and when NOT to use them

Runtime (dynamic) polymorphism is a core OOP tool: one pointer to a base class, many possible derived
behaviors chosen *at runtime*. It's also, in an HFT hot path, frequently the **wrong** tool. This
module makes you fluent in *how* it works down to the byte level — the vtable, the vptr, the indirect
call — and *why* low-latency code reaches for templates, CRTP, or `variant` instead. Being able to
draw the object layout and justify the split between hot and cold paths is a strong interview signal.

---

## 1. Inheritance basics

```cpp
class Order {
public:
    virtual ~Order() = default;          // virtual destructor — REQUIRED for a polymorphic base
    virtual double value() const = 0;    // '= 0' -> pure virtual -> Order is ABSTRACT
};

class LimitOrder : public Order {
    double price_;
    std::uint32_t qty_;
public:
    LimitOrder(double p, std::uint32_t q) : price_(p), qty_(q) {}
    double value() const override { return price_ * qty_; }   // 'override' is checked by compiler
};
```

The vocabulary, precisely:

- `: public Order` establishes an **"is-a"** relationship — a `LimitOrder` *is an* `Order` and can be
  used wherever an `Order&`/`Order*` is expected (the **Liskov substitution** idea).
- `virtual` on a member function enables **dynamic dispatch**: a call through a base reference/pointer
  resolves to the *actual runtime type's* override.
- `override` (always write it) makes the compiler *verify* you are genuinely overriding a base
  virtual with a matching signature — it catches the classic bug where a subtle signature mismatch
  (a missing `const`, a different parameter type) silently creates a *new* function instead of
  overriding, so your "override" never gets called.
- `= 0` makes a function **pure virtual**; a class with any pure virtual is **abstract** and cannot
  be instantiated (`Order o;` is an error) — it exists only to be derived from.

---

## 2. Dynamic dispatch and the vtable — the memory picture

This is the section interviewers care about. When a type has any virtual function, the compiler adds
one hidden machinery layer:

1. Per **class**, it emits a static, read-only **vtable** — an array of function pointers, one slot
   per virtual function, pointing at that class's final overrides.
2. Per **object**, it inserts a hidden pointer, the **vptr**, usually as the *first* member (offset
   0), pointing at its class's vtable. This is why a polymorphic object is 8 bytes bigger than its
   data alone on a 64-bit machine.

```cpp
void process(const Order& o) {
    o.value();   // which value()? Decided at RUNTIME from o's real type.
}
process(LimitOrder{100.0, 5});   // calls LimitOrder::value()
```

Here is what a `LimitOrder` object and its vtable look like in memory:

```
 LimitOrder object                    vtable for LimitOrder (static, one per class)
 ┌───────────────────┐   points to   ┌────────────────────────────────┐
 │ vptr  ────────────┼──────────────▶│ [0] &LimitOrder::~LimitOrder    │
 ├───────────────────┤               │ [1] &LimitOrder::value          │
 │ price_  (double)  │               └────────────────────────────────┘
 ├───────────────────┤
 │ qty_    (uint32)  │               A MarketOrder object's vptr instead
 └───────────────────┘               points to MarketOrder's vtable, whose
   vptr at offset 0                   [1] slot holds &MarketOrder::value.
```

A virtual call `o.value()` compiles to roughly three steps:

```
1. load  vptr   = *(o + 0)          ; read the hidden vptr  (memory load)
2. load  target = *(vptr + slot*8)  ; read the function ptr from the vtable slot (memory load)
3. call  target                     ; INDIRECT call through the pointer
```

Two memory loads and one **indirect** call. Contrast a non-virtual call, which is a single **direct**
call to a known address (and usually gets inlined away entirely).

---

## 3. Why HFT hot paths avoid virtual functions

That machinery has real, measurable costs at nanosecond scale:

1. **Indirect call, no inlining.** The target address isn't known at compile time, so the compiler
   *cannot inline* the callee — and inlining is the single biggest enabler of downstream
   optimizations. A tiny `value()` that could have folded into the caller instead stays a full
   function call.
2. **Branch (target) misprediction.** The CPU predicts the indirect call's target; a miss flushes
   the pipeline — on the order of **~15–20 cycles** on modern x86. In a tight loop over mixed types,
   the predictor thrashes.
3. **Extra memory loads.** The vptr load + vtable load are two dependent loads on the critical path
   before the call can even start.
4. **Cache pressure two ways.** The vptr **bloats every object by 8 bytes**, so fewer objects fit per
   cache line (Module 15). And the vtables live *elsewhere* in memory, so an array of polymorphic
   objects scatters both data and dispatch targets.

For a function called millions of times per second on the critical path, this is exactly the
difference between hitting and missing a latency budget. None of it matters off the hot path.

---

## 4. The alternatives HFT reaches for

**1. Templates — compile-time polymorphism (Module 10).** Resolved at compile time, fully inlinable,
zero indirection. The first choice when the set of types is known at compile time.

**2. CRTP — static polymorphism (Module 17).** Inheritance-*shaped* code with compile-time dispatch —
the base is templated on the derived type, so the "virtual" call becomes a `static_cast` resolved at
compile time:

```cpp
template <typename Derived>
struct Strategy {
    void run() { static_cast<Derived*>(this)->runImpl(); }  // dispatch with NO vtable, inlinable
};
struct Momentum : Strategy<Momentum> { void runImpl(); };
```

**3. `std::variant` + `std::visit`** — a *closed* set of known types with type-safe dispatch, stored
inline (no heap, no vptr), often cache-friendlier and faster than virtual:

```cpp
using AnyOrder = std::variant<LimitOrder, MarketOrder, StopOrder>;
std::visit([](auto& o){ return o.value(); }, anyOrder);   // dispatch over a fixed type set
```

**4. Tagged union / enum + switch** — an `enum` type tag plus a `switch`. Old-school, branch-
predictable, cache-dense, and extremely common in real matching engines:

```cpp
enum class Type : std::uint8_t { Limit, Market, Stop };
switch (o.type) {
    case Type::Limit:  /* ... */ break;
    case Type::Market: /* ... */ break;
    case Type::Stop:   /* ... */ break;
}
```

The rule of thumb: **closed set of types, hot path → variant/enum/CRTP; open/extensible set, cold
path → virtual.**

---

## 5. When inheritance / virtual IS the right tool

Don't cargo-cult "virtual is slow." Use it freely where it belongs:

- **Cold paths**: startup, configuration, logging, admin/monitoring — called rarely; clarity and
  extensibility win, the indirection is invisible.
- **Plugin-style extensibility**: when new derived types are added *without recompiling* the code
  that calls through the base — the open/closed principle. A closed `variant` can't do this.
- **Small, stable interfaces** where dispatch cost is irrelevant to throughput.

The skill isn't avoiding virtual — it's *knowing which path you're on* and choosing accordingly.

---

## 6. The virtual destructor rule (and the UB if you break it)

```cpp
Order* o = new LimitOrder{100.0, 5};
delete o;   // Which destructor runs?
```

If `~Order()` is **virtual**, `delete o` dispatches through the vtable to `~LimitOrder()` first
(then `~Order()`) — correct. If `~Order()` is **not** virtual, `delete o` calls only `~Order()` — the
`LimitOrder` part is never destroyed. That is **undefined behavior**, and in practice leaks any
resources the derived part owned. Rule: **a base class you `delete` polymorphically must have a
virtual destructor.** (Corollary: if a class is *not* meant to be a polymorphic base, don't make it
one — a non-virtual destructor is a signal "don't delete me through a base pointer.")

---

## 7. Slicing — a classic trap

```cpp
void take(Order o);          // takes the base BY VALUE
take(LimitOrder{100.0, 5});  // SLICED: only the Order sub-object is copied; price_/qty_ chopped off
```

Passing a derived object *by value* to a base-typed parameter copies only the base sub-object — the
derived data and even the vptr are "sliced" away, and you lose polymorphism. Dynamic dispatch
requires a **reference or pointer** (`const Order&`, `Order*`), never by value. This is an
ever-popular interview trap; the fix is always "pass by reference/pointer."

```cpp
void take(const Order& o);   // correct: no slicing, polymorphism preserved
```

---

## Common pitfalls & UB

- **Missing virtual destructor** on a polymorphic base you `delete` through → UB / leaks (Section 6).
- **Object slicing** by passing/storing derived objects by value in base-typed slots (Section 7).
- **Calling virtuals from a constructor/destructor**: during `Base`'s ctor the object *is* a `Base`
  (the vptr points at `Base`'s vtable), so a virtual call resolves to `Base`'s version, not the
  derived override — a subtle, well-known gotcha.
- **Forgetting `override`**: a signature mismatch silently creates a new function; your "override"
  is never dispatched. `override` turns this into a compile error.
- **`std::vector<Order>` of a polymorphic type**: stores sliced base objects. Store
  `std::vector<std::unique_ptr<Order>>` (or a `variant`) to keep polymorphism.
- **Storing polymorphic objects contiguously by value** to "help cache" — impossible; you must
  hold them via pointers, which re-introduces indirection. This is a core reason HFT uses `variant`.

---

## Quiz — tough problems

**Q1.** `sizeof` of a class with two `int` members and one virtual function on a 64-bit machine?
**Answer:** 16 bytes. The hidden vptr (8 bytes) + two `int`s (8 bytes) = 16, with the vptr typically
first and 8-byte aligned. Add a virtual and every object grows by a pointer.

**Q2.** What prints?
```cpp
struct B { B(){ f(); } virtual void f(){ std::cout << "B\n"; } };
struct D : B { void f() override { std::cout << "D\n"; } };
D d;
```
**Answer:** `B`. During `B`'s constructor the object's dynamic type is still `B` (its vptr points at
`B`'s vtable), so the virtual call dispatches to `B::f`, *not* `D::f`. Calling virtuals from ctors/
dtors is the trap.

**Q3.** Is this UB? `Base* p = new Derived; delete p;` where `~Base` is non-virtual.
**Answer:** Yes, undefined behavior. Without a virtual destructor only `~Base` runs; the `Derived`
sub-object is never destroyed. Leaks in practice, UB by the standard.

**Q4.** How many memory loads (minimum) does a single virtual call add versus a direct call, and why?
**Answer:** Two — the vptr load from the object, then the function-pointer load from the vtable slot —
before the indirect call. A direct call needs neither.

**Q5.** What's wrong here and how does the compiler catch it?
```cpp
struct Base { virtual void f(int) const; };
struct Der : Base { void f(int); };   // intended override
```
**Answer:** `Der::f(int)` is non-`const`, so it does *not* override `Base::f(int) const` — it's a new,
hidden function; the base version still dispatches. Writing `void f(int) override;` makes the compiler
reject it with "does not override," forcing you to fix the signature.

**Q6.** You have `std::vector<Shape>` where `Shape` is a polymorphic base. Why is this almost always
a bug, and what are two fixes?
**Answer:** The vector stores `Shape` *by value*, so any derived object pushed in is sliced to its
base part — polymorphism lost. Fixes: `std::vector<std::unique_ptr<Shape>>` (heap-allocated, virtual
dispatch preserved) or `std::vector<std::variant<Circle,Square,...>>` (closed set, stored inline, no
vptr).

**Q7.** Rank by dispatch cost on the hot path: (a) virtual call, (b) CRTP call, (c) `std::variant` +
`std::visit`, (d) direct non-virtual call. Explain the top and bottom.
**Answer:** Fastest → slowest roughly: direct ≈ CRTP (both compile-time, inlinable, no indirection) <
variant/visit (a small compile-time-bounded switch, no vptr but a tag test) < virtual (two loads +
indirect call + possible misprediction, no inlining). Direct/CRTP win because everything resolves at
compile time; virtual loses because nothing does.

**Q8.** Why can't you store polymorphic objects contiguously *by value* to get cache locality, and
how does `std::variant` sidestep this?
**Answer:** Polymorphism requires a base pointer/reference, so a by-value contiguous array would slice
everything to the base. `variant` stores the largest alternative inline with a small type tag — no
pointer, no vptr — so an array of `variant` *is* contiguous, keeping cache locality while still
supporting per-element type-safe dispatch.

---

## Indian HFT interview questions

*(Asked at Tower Research Gurgaon, Optiver, IMC, Graviton, Quadeye, AlphaGrep, WorldQuant, Da Vinci,
Squarepoint, Jump, HRT, NK Securities, Mansard.)*

- **"Draw the memory layout of a polymorphic object and explain a virtual call."**
  vptr at offset 0 pointing to the class's static vtable (an array of function pointers). A call =
  load vptr, load the slot's function pointer, indirect-call it. Every polymorphic object carries the
  8-byte vptr; each class has one shared vtable.

- **"What is the exact cost of a virtual call and why does HFT avoid it on the hot path?"**
  Two dependent memory loads (vptr, then vtable slot), an indirect call the compiler can't inline,
  and a possible target misprediction (~15–20 cycles). Plus per-object vptr bloat hurting cache
  density. All of it is on the critical path of a function called millions of times/sec.

- **"When is virtual acceptable?"**
  Cold paths (config, logging, admin), plugin extensibility where callers mustn't recompile, and
  interfaces where dispatch cost is negligible vs. the work done. The judgement is hot-vs-cold path.

- **"Why must a base class have a virtual destructor? What happens if it doesn't?"**
  So `delete base_ptr` dispatches to the derived destructor. Without it, only the base destructor
  runs, the derived part isn't destroyed → undefined behavior and resource leaks.

- **"What is object slicing? How do you avoid it?"**
  Copying a derived object into a base-typed *value* drops the derived data and vptr. Avoid by always
  passing/storing polymorphic types by reference or pointer (or `unique_ptr` / `variant`), never by
  base value.

- **"Give me three ways to do polymorphism without a vtable."**
  Templates (compile-time monomorphization), CRTP (`static_cast<Derived*>(this)` dispatch), and
  `std::variant` + `std::visit` (closed type set, inline storage). Plus enum-tag + switch for a fixed
  small set.

- **"Prefer composition over inheritance — why, in a latency context?"**
  Deep inheritance forces virtual dispatch and scattered layouts; composition + templates/variant
  keeps data contiguous and dispatch compile-time. You get reuse without paying the vtable and
  cache-locality tax.

---

## In the order book

The matching hot path uses **no virtual functions**. Order type is encoded as an `enum` type tag with
a `switch`, or the side is a template parameter (`BookSide<Side::Buy>`) so buy/sell dispatch is
resolved at compile time. Orders are stored **contiguously by value** in a pool (Module 15) — only
possible because they're not polymorphic — so the CPU prefetcher streams them with near-zero cache
misses. Any genuine polymorphism (e.g. pluggable strategies in a backtester, or a logging/reporting
sink) lives in the **cold configuration layer**, where a `virtual` interface or `std::function` costs
nothing measurable. Being able to draw that hot/cold split and defend the enum-vs-virtual choice is
exactly the kind of systems judgement these desks probe for.

---

## Key takeaways

- Runtime polymorphism works via a per-class **vtable** (array of function pointers) and a per-object
  **vptr** at offset 0; a virtual call = two loads + one indirect call, and it can't inline.
- The HFT cost is real: no inlining, possible target mispredict (~15–20 cycles), extra loads on the
  critical path, and 8 bytes of vptr bloat per object hurting cache density.
- Alternatives with **compile-time or vtable-free dispatch**: templates, CRTP, `std::variant`+visit,
  enum-tag + switch. Use them on the hot path.
- Virtual is *right* off the hot path: cold config, plugins, extensibility, small stable interfaces.
- Never forget the **virtual destructor** on a polymorphic base (UB otherwise), always write
  `override`, and never pass polymorphic types by value (slicing).

**Next:** [12 — The STL: containers, iterators, algorithms, complexity & cache](12-stl.md) — where
the same "Big-O lies, cache decides" lesson governs every container choice.
