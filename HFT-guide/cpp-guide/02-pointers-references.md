# Module 2 — Pointers & references

Two ways to refer to an object *indirectly*. Everything built later — data structures, function
parameters, the entire order-book graph of price levels and linked orders — is built on these. A
pointer is "a variable that holds an address"; a reference is "another name for an existing object."
The distinction sounds trivial and is the source of an enormous fraction of both C++ bugs and C++
interview questions, so this module builds both from the byte level up.

---

## 1. Every object has an address

Recall from Module 1 that memory is a huge byte array and every byte has a numeric address. The
address-of operator `&` gives you the address where an object lives:

```cpp
int x = 5;
std::cout << &x;  // e.g. 0x7ffee3b8 — the address where x's bytes sit on the stack
```

That address is itself just a number (a 64-bit integer on a 64-bit machine). A pointer is nothing
more magical than a variable that *stores such a number* and remembers what type lives there.

```
Stack
┌───────────────┐
│ x: 5          │  ← lives at address 0x7ffee3b8
└───────────────┘
       ▲
   &x == 0x7ffee3b8   (this value is what a pointer to x would hold)
```

---

## 2. Pointers: a variable holding an address

```cpp
int x = 5;
int* p = &x;   // "p is a pointer to int"; p holds x's address
```

`p` is its own object with its own address, and its *value* is the address of `x`:

```
Stack
┌───────────────┐
│ x:  5         │  0x7ffee3b8
├───────────────┤
│ p:  0x7ffee3b8│  0x7ffee3c0   ──┐
└───────────────┘                 │
        └─────────────────────────┘  p "points at" x
```

**Dereference** with `*` to reach the value the pointer refers to:

```cpp
std::cout << *p;   // 5   — "the value AT that address"
*p = 10;           // writes THROUGH the pointer: x is now 10
```

> The `*` symbol is two different things. In a **declaration** (`int* p`) it's part of the type
> ("pointer to int"). In an **expression** (`*p`) it's the dereference operator ("go to the address
> and give me what's there"). Same glyph, opposite directions — beginners conflate them constantly.

A pointer is always the same size regardless of what it points to — 8 bytes on a 64-bit platform —
because it just holds an address. `sizeof(int*) == sizeof(double*) == sizeof(HugeStruct*) == 8`.

### Null pointers

```cpp
int* p = nullptr;      // points to nothing (holds address 0)
if (p) { /* ... */ }   // pointers convert to bool: non-null == true
std::cout << *p;       // dereferencing null -> UB / crash (SIGSEGV)
```

A pointer that points to nothing holds the sentinel value `nullptr` (numerically 0). Always ask
"can this be null?" *before* dereferencing. Use `nullptr`, never the legacy `NULL` or bare `0` —
`nullptr` has a distinct type (`std::nullptr_t`) that participates correctly in overload resolution
and template deduction, whereas `NULL`/`0` are integers in disguise and cause subtle bugs.

### Pointers reassign and do arithmetic

Unlike references (below), a pointer can be **re-seated** to point somewhere else, and it supports
**arithmetic** for walking contiguous memory:

```cpp
int a = 1, b = 2;
int* p = &a;   // -> a
p = &b;        // re-seated: -> b now

int arr[3] = {10, 20, 30};
int* q = arr;          // arr decays to &arr[0]
std::cout << *(q + 1); // 20 — q+1 advances by sizeof(int) (4 bytes), NOT 1 byte
```

The critical subtlety: pointer arithmetic is in **units of the pointed-to type**, not bytes. `q + 1`
adds `sizeof(int)` bytes to the address. This is exactly how array indexing works under the hood —
`arr[i]` is *defined* as `*(arr + i)`.

```
int arr[3]:   ┌────┬────┬────┐
              │ 10 │ 20 │ 30 │
              └────┴────┴────┘
address:      1000 1004 1008     (each int is 4 bytes)
              ▲    ▲
             q    q+1  (advances 4 bytes, not 1)
```

Walking contiguous memory this way is the basis of cache-friendly array traversal (Module 15) — the
prefetcher loves the predictable, stride-`sizeof(T)` access pattern.

### `const` and pointers (read the declaration right-to-left)

There are two independent things you can freeze: the *pointee* (what it points at) and the *pointer*
(where it points). Read right-to-left past the `*`:

```cpp
const int* p;            // pointer to const int   — can't change *p, CAN re-seat p
int* const p = &x;       // const pointer to int   — CAN change *p, can't re-seat
const int* const p = &x; // const pointer to const int — both frozen
```

Trick: whatever is to the *left* of the `*` is what's const about the pointee; a `const` to the
*right* of the `*` freezes the pointer itself. `const int*` and `int const*` are identical; the
second reads more consistently ("pointer to int-const").

---

## 3. References: an alias for an existing object

```cpp
int x = 5;
int& ref = x;   // ref IS x — another name for the same object
ref = 10;       // no deref needed; changes x
```

A reference is not a separate object holding an address (conceptually). It's an **alias** — a second
name bound to the *same* memory as `x`. There is no separate "reference variable" to inspect;
`&ref == &x`, and `ref` and `x` are indistinguishable at the language level.

```
int x = 5;  int& ref = x;

┌───────────────┐
│ x / ref: 5    │   one object, two names — writing through either touches the same bytes
└───────────────┘
```

> Implementation note: under the hood the compiler usually implements a reference *as* a pointer
> (especially reference parameters), but the *language semantics* are "alias, cannot be null, cannot
> be re-seated." Answer interview questions with the semantics, and mention the implementation only
> if asked "how is it implemented?"

Two rules make references safer than pointers:

1. **Must be initialized at declaration** — there is no null reference and no "empty" reference.
   `int& r;` doesn't compile.
2. **Can never be re-seated** — `ref` aliases `x` for its entire life. Assigning to `ref` changes
   `x`'s *value*; it does not repoint `ref`.

```cpp
int a = 1, b = 2;
int& r = a;
r = b;   // this means a = b (a becomes 2). r STILL aliases a, NOT b.
```

That single behavior — assignment writes through instead of rebinding — is the crux of the
pointer-vs-reference difference. Pointers can be null and re-seated; references cannot be either.

---

## 4. Why references exist: passing without copying

By default C++ passes function arguments **by value** — the callee gets a *copy*:

```cpp
void f(int n) { n = 99; }   // n is a copy; the caller's variable is unchanged
```

Fine for an `int`. Ruinous for a large object — copying a `std::vector<Order>` with a million
elements copies a million elements on every call. **Pass by reference** avoids the copy and (unless
`const`) lets you modify the original:

```cpp
void f(int& n) { n = 99; }  // operates on the caller's variable directly — no copy
```

### `const&` — the workhorse parameter

When you want to pass a big object cheaply *and* promise not to modify it, take it by
`const T&`:

```cpp
void print(const std::string& s) { std::cout << s; }  // no copy, read-only
```

`const T&` is the default way to pass anything bigger than a couple of machine words. It binds to
both lvalues and temporaries (rvalues), costs one pointer under the hood, and documents "I won't
change this." You will write it constantly. (For *small* trivially-copyable types like `int`,
`double`, or a pointer, pass by value — a copy is cheaper than the indirection.)

```
Pass by value:                 Pass by const&:
  caller's obj  ─copy─►  n       caller's obj  ◄──alias── s   (no copy; s reads the original)
  (expensive for big T)          (one pointer under the hood)
```

---

## 5. Pointer vs reference — which do I reach for?

- **Reference** — the thing definitely exists and won't change identity. Safer default; use it for
  most function parameters and for "I'm just aliasing this object." Can't be null, can't dangle *as
  easily* (though a reference to a destroyed object still dangles).
- **Pointer** — you need one of the three things a reference can't do:
  1. **Nullability** — "this might be absent" (an optional relationship, a not-found result).
  2. **Re-seating** — "this points at different objects over its lifetime" (an iterator-like cursor,
     a `current` node).
  3. **Arithmetic** — arrays, buffers, walking raw memory.

Linked lists, trees, and graphs need pointers because their links are *optional* (a leaf's child is
null) and *reassignable* (splicing a node changes links). You cannot build them with references. For
ownership, though, prefer smart pointers (Module 13) over raw pointers — raw pointers here mean
"non-owning observer."

---

## Common pitfalls & UB

- **Dereferencing null or a dangling pointer** — UB; often a crash, sometimes silent corruption.
- **Confusing `int* p, q;`** — this declares `p` as `int*` and `q` as plain `int`. The `*` binds to
  the declarator, not the type. Declare one pointer per line to avoid it.
- **Returning a pointer/reference to a local** (Module 1) — dangles.
- **Pointer arithmetic past the array bounds** — even *computing* `arr + n + 1` (one past "one past
  the end") is UB, let alone dereferencing it. `arr + n` (one-past-the-end) is legal to form but not
  to dereference.
- **Reference to a temporary through a function** — `int& r = getInt();` where `getInt` returns by
  value: dangling.
- **Using `NULL`/`0` instead of `nullptr`** — breaks overload resolution and templates.
- **Assuming re-assigning a reference rebinds it** — it writes through to the referent.

---

## Quiz — tough problems

**Q1.** What does this print?
```cpp
int a = 1, b = 2;
int& r = a;
r = b;
b = 5;
std::cout << a << " " << b;
```
**Answer:** `2 5`. `r = b` copies `b`'s value (2) into `a` (r aliases a). Then `b = 5` changes only
`b`. `r` never rebinds to `b`, so `a` stays 2.

**Q2.** What is `sizeof` here on a 64-bit machine?
```cpp
struct Node { int v; Node* next; };
sizeof(Node);   // ?
```
**Answer:** `16`, not 12. `int` is 4 bytes, `Node*` is 8 bytes, but the pointer must be 8-byte
aligned, so 4 bytes of padding are inserted after `v`. Layout: `[v:4][pad:4][next:8]`. (Alignment is
Module 15.)

**Q3.** Predict the output:
```cpp
int arr[5] = {10,20,30,40,50};
int* p = arr + 2;
std::cout << p[-1] << " " << *(arr + 4) - *p;
```
**Answer:** `20 20`. `p` points at `arr[2]` (30). `p[-1]` is `arr[1]` = 20. `*(arr+4)` is 50, `*p`
is 30, difference 20. (Negative indexing is legal as long as it stays in-bounds.)

**Q4.** Is this legal, and what does it mean?
```cpp
const int* const p = &x;
```
**Answer:** Legal. `p` is a const pointer to a const int: you cannot re-seat `p` and you cannot
modify `*p`. Both the pointer and the pointee are frozen.

**Q5.** Spot the bug:
```cpp
int* p, q;
p = new int(5);
q = new int(7);   // ?
```
**Answer:** `q` is an `int`, not an `int*` — the `*` in `int* p, q;` binds only to `p`. `q = new
int(7)` assigns a pointer value into an `int`, which won't compile. Declare pointers one per line.

**Q6.** What's wrong here, and does it always crash?
```cpp
int* make() { int x = 42; return &x; }
int main() { int* p = make(); std::cout << *p; }
```
**Answer:** Dangling pointer — `x` dies when `make` returns. UB. Often prints `42` in a debug build
(the slot isn't overwritten yet) and garbage/crash in optimized builds. Never rely on the value.

**Q7.** True or false: a reference occupies no memory.
**Answer:** Language-level, a reference has no observable storage of its own and `&ref == &referent`.
Implementation-level, a reference parameter or a reference member is typically stored as a pointer.
The correct interview answer is "semantically it's an alias with no separate identity; it's usually
implemented as a pointer."

**Q8.** Predict the output:
```cpp
void inc(int* p) { (*p)++; }
void inc(int& r) { r += 10; }
int main() { int x = 0; inc(&x); inc(x); std::cout << x; }
```
**Answer:** `11`. `inc(&x)` calls the pointer overload (`++` → 1); `inc(x)` calls the reference
overload (`+= 10` → 11). Passing an address selects the pointer version; passing the object selects
the reference version.

**Q9.** Why is `void f(char* p)` slower to reason about for the optimizer than `void f(char& c)`
when two params alias? (aliasing)
**Answer:** Two pointers may point at the same object (aliasing), forcing the compiler to reload
through them conservatively; it can't assume writes through one don't affect reads through the
other. `restrict`/`__restrict` (or distinct references known not to alias) let the optimizer keep
values in registers. Aliasing analysis is a real hot-path concern.

---

## Indian HFT interview questions

Asked on India desks — Tower Research (Gurgaon), Graviton (Gurgaon), Optiver, IMC, Quadeye,
AlphaGrep, WorldQuant, Da Vinci, NK Securities, Squarepoint, Jump.

**Q. "Difference between a pointer and a reference? Give three concrete ones."**
*Model answer:* (1) A pointer can be null; a reference must bind to a valid object. (2) A pointer can
be re-seated; a reference is bound for life. (3) A pointer supports arithmetic and has its own
storage/address; a reference is an alias (`&ref == &referent`). Use references for "definitely
present, won't change identity" (most params), pointers for optional/re-seatable/array cases.

**Q. "Why do we pass big objects by `const&` instead of by value?"**
*Model answer:* By value copies the whole object (potentially heap allocations and O(n) work);
`const&` passes a single pointer under the hood, no copy, and the `const` documents/enforces
read-only. For small trivially-copyable types (`int`, pointer), pass by value — the copy is cheaper
than the indirection and avoids an aliasing barrier.

**Q. "What's the size of a pointer, and does it depend on the type it points to?"**
*Model answer:* 8 bytes on a 64-bit platform, independent of pointee type — it just holds an
address. (Exception trivia: pointer-to-member and pointers to virtual/multiple-inheritance types can
be larger, but that's rarely what's being asked.)

**Q. "What is pointer arithmetic in units of — bytes?"**
*Model answer:* No — in units of `sizeof(pointee)`. `p + 1` on an `int*` advances 4 bytes. `arr[i]`
is defined as `*(arr + i)`. This is why cache-friendly traversal walks with stride `sizeof(T)`.

**Q. "How can two pointers hurt performance even when the code looks fine?"**
*Model answer:* Aliasing. If the compiler can't prove two pointers don't overlap, it must
conservatively reload memory instead of caching values in registers, and can't vectorize freely.
`__restrict` or using references/values known not to alias lets it optimize. Relevant when hand-
tuning hot loops.

**Q. "`nullptr` vs `NULL` vs `0` — why does the language bother with `nullptr`?"**
*Model answer:* `NULL`/`0` are integers, so they pick integer overloads and deduce as `int` in
templates, causing subtle bugs (e.g. calling `f(int)` instead of `f(char*)`). `nullptr` has its own
type `std::nullptr_t` that converts to any pointer type and never to an integer, fixing overload
resolution and template deduction.

---

## In the order book

Every reference-vs-pointer choice in the engine is deliberate:

- **`const Order&`** to pass an order into a matching or validation function — cheap (one pointer),
  read-only, and the order definitely exists. This is the workhorse parameter across the codebase.
- **Raw pointers (or 32-bit indices — Module 15)** to link resting orders in a doubly-linked list at
  each price level, and to hold `highestBuy` / `lowestSell` cursors. Links must be *nullable* (the
  head/tail of a level has no neighbor) and *re-seatable* (matching splices orders out), which is
  exactly what a reference cannot do. These raw pointers are **non-owning** — the object pool
  (Module 3) owns the storage; the list just observes it.
- **A hash map from order-ID → pointer** for O(1) cancel/modify lookup: given an incoming cancel for
  order 12345, jump straight to its node without scanning any level.

Using 32-bit indices instead of 8-byte pointers for the links is a classic HFT micro-optimization
(Module 15): it halves the link size, so more of the book fits in cache, and the "pointer" can't
outlive the pool. Every one of these is a conscious pointer-vs-reference-vs-index decision, and
being able to justify each is exactly what the interview is checking.

---

## Key takeaways

- A **pointer** is a variable holding an address (8 bytes on 64-bit); it can be null, re-seated, and
  do arithmetic. Dereference with `*`; take an address with `&`.
- A **reference** is an alias for an existing object — must be initialized, can never be re-seated,
  can't be null; assigning to it writes *through* to the referent.
- Pointer arithmetic is in units of `sizeof(pointee)`, not bytes — the foundation of array indexing
  and cache-friendly traversal.
- `const T&` is the default parameter for anything bigger than a couple of words: no copy, read-only;
  pass small trivial types by value.
- Choose a **reference** for "present, fixed identity" (most params); a **pointer/index** when you
  need nullability, re-seating, or arithmetic (linked structures, cursors).
- Use `nullptr` (typed) over `NULL`/`0` (integers) — it fixes overload resolution and template
  deduction.

**Next:** [Module 3 — Dynamic memory (`new`/`delete`) & why the heap is slow](03-dynamic-memory.md)
— giving objects a lifetime you control, and the object-pool trick that gives heap lifetime without
heap latency.
