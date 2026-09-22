# Module 3 — Dynamic memory (`new`/`delete`) & why the heap is slow

Stack objects die when their scope ends (Module 1). But order-book orders must *outlive* the
function that created them — they rest in the book until filled or cancelled. That's what the
**heap** (free store) is for: memory whose lifetime **you** control. This module covers the raw
mechanism (`new`/`delete`), the three classic bugs it causes, *why* the heap is slow and — the part
HFT interviews really care about — *unpredictable*, and the object-pool trick that gets you heap
lifetime without heap latency.

---

## 1. Allocating on the heap

```cpp
int* p = new int;   // allocate storage for one int on the heap; p holds its address
*p = 5;
delete p;           // return the memory to the allocator
p = nullptr;        // hygiene: avoid a dangling pointer

int* arr = new int[100];  // 100 contiguous ints on the heap
delete[] arr;             // delete[] for arrays — NOT delete
```

The essential difference from a stack local: the heap object lives **until you `delete` it**, not
when the function returns. That's the entire point — it lets data survive past its creating scope.

The layout is "handle on the stack, payload on the heap." `p` (the handle, 8 bytes) lives on the
stack in the current frame; the `int` it points at lives on the heap and stays put when the frame is
popped:

```
Stack (this frame)              Heap (persists across calls)
┌──────────────────┐           ┌──────────────────┐
│ p: 0x55a2c0      │ ────────► │ 5   (int)        │  0x55a2c0
└──────────────────┘           └──────────────────┘
   dies at end of scope           lives until `delete p`
```

Match the forms **exactly**: `new` ↔ `delete`, `new[]` ↔ `delete[]`. Calling `delete` (scalar) on
an array allocated with `new[]` — or vice versa — is undefined behavior, because the allocator
tracks array size differently (it stashes the element count in a header before the block so
`delete[]` knows how many destructors to run).

---

## 2. The three deadly heap bugs

Manual memory management gives you full control and full responsibility. Three failure modes account
for most of C++'s dangerous reputation:

```cpp
// 1. LEAK — new without delete
void leak() { int* p = new int(5); }  // p (the handle) dies; the heap block is now unreachable → lost forever

// 2. USE-AFTER-FREE (dangling)
int* p = new int(5);
delete p;
std::cout << *p;   // UB — reading memory that's been returned to the allocator

// 3. DOUBLE FREE
int* q = new int(5);
delete q;
delete q;          // UB — corrupts the allocator's internal bookkeeping
```

- **Leak**: you lose the *only* pointer to a live block. The memory is still allocated but
  unreachable — it accumulates until the process dies. In a long-running trading process, a per-order
  leak is fatal over a session.
- **Use-after-free**: the block was returned to the allocator, which may have handed it to someone
  else. Reading gives stale/foreign data; writing corrupts another object. Like the dangling-stack
  pointer from Module 1, it often *appears* to work until the slot is reused.
- **Double free**: freeing an already-freed block corrupts the allocator's free-list metadata, which
  typically crashes *later*, somewhere unrelated — a nightmare to debug.

```
Use-after-free timeline:
  new  ──►  [block owned by you]
  delete ─► [block back on free-list]   *p here reads reclaimed memory
  (another new) ─► [same block now owned by someone else]   *p now corrupts THEIR object
```

These bugs are exactly why the modern language works so hard to eliminate raw `new`/`delete` from
everyday code.

---

## 3. Modern C++: you rarely write raw `new`/`delete`

You must *understand* `new`/`delete` (that's this module), but in real code you use tools that call
them for you and guarantee cleanup:

```cpp
#include <vector>
#include <memory>

std::vector<int> v(100);           // heap array; frees itself in its destructor

auto p = std::make_unique<int>(5); // heap int, owned by p
// ... use *p ...
// p goes out of scope -> automatically deletes. No leak, no manual delete, no double free.
```

This is **RAII** (Module 7): tie the heap object's lifetime to a *stack* object (the `vector`, the
`unique_ptr`), so the automatic stack cleanup from Module 1 handles the heap cleanup for you. You get
heap flexibility with stack safety — the single most important idiom in modern C++, and the reason a
well-written modern C++ program can be as memory-safe in practice as a garbage-collected one, without
the GC pauses (which matters enormously for HFT).

---

## 4. "If RAII ties it to a scope, why not just use the stack?"

A fair and common question — and mostly right. RAII ties lifetime to the **owner** object, not
blindly to the enclosing scope, and the owner can be *moved out* of the scope (Module 9). Prefer the
stack by default; reach for the heap only when the stack genuinely **can't** do the job. Four reasons
it can't:

1. **Too big.** The stack is a fixed 1–8 MB block (Module 1). `int big[10'000'000];` (40 MB)
   overflows it; `std::vector<int> big(10'000'000);` puts 40 MB on the heap with a tiny handle on
   the stack.
2. **Size known only at runtime.** Stack frame sizes are fixed at *compile* time. A true
   variable-length array `int arr[n];` with runtime `n` isn't standard C++; anything that *grows*
   (`vector::push_back`, a filling `map`) must live on the heap.
3. **Must outlive the creating scope.** You can't return a pointer/reference to a stack local (it
   dies at `}`). The heap lets an object be created here and handed back, with ownership transferred
   by **move**:

   ```cpp
   std::unique_ptr<Order> makeOrder() {
       auto o = std::make_unique<Order>(/*...*/);  // heap object
       return o;    // ownership MOVES to the caller; the Order does NOT die at }
   }
   // Order* bad() { Order o; return &o; }  // ❌ dangling — stack o dies at }
   ```

   The *handle* dies at `}`; the heap *payload* survives, now owned by the caller.
4. **Shared or polymorphic.** Several owners sharing one object until the *last* is done
   (`shared_ptr`), or "some subclass of `Shape`, exact type unknown until runtime" (varying size →
   can't fit a fixed stack slot). The stack expresses neither.

**Mental model: handle on the stack, payload on the heap.** Small handle → cheap to move, gets
automatic cleanup; big / variable-size / long-lived / shared payload → lives on the heap where those
are allowed. Use the stack directly when the object is small *and* compile-time-sized *and*
scope-local (most locals). Otherwise, heap via an RAII owner.

---

## 5. Why the heap is *slow* — and worse, *unpredictable* (the HFT core)

Stack allocation is ≈ one register op (move `rsp`). A heap `new` is dramatically more work, and the
work is *variable*:

1. **Search.** The allocator must find a free block big enough. General-purpose allocators
   (glibc `malloc`/ptmalloc, tcmalloc, jemalloc) maintain size-class bins and free-lists; finding and
   splitting a block is real work, and it maintains per-block metadata (a header before your data).
2. **Locking.** The heap is shared across threads, so `new`/`delete` may take a lock or use atomic
   operations on shared arenas — contention under multithreading causes stalls.
3. **Syscalls.** When the allocator runs out of pooled memory it asks the OS for more (`mmap` /
   `sbrk`) — a **system call**, orders of magnitude slower, and the first touch of a fresh page
   triggers a page fault.
4. **Cache scatter.** Two separate `new`s can land far apart in the address space. Walking objects
   that are scattered thrashes the CPU cache (a miss ≈ 100 ns vs an L1 hit ≈ 1 ns — Module 15).

Rough intuition: stack ≈ 1 ns; heap `new` ≈ tens to hundreds of ns, and *occasionally* microseconds
when it hits the OS. That last word — *occasionally* — is the real problem. HFT doesn't primarily
care about average latency; it cares about the **tail** (P99, P99.9). An allocator that's usually
fast but sometimes takes a syscall injects **jitter/variance** exactly where you can least afford it.
In a loop processing millions of orders per second, a `new` per order is fatal not because the mean
is high but because the worst case is unbounded and unpredictable.

```
Latency vs jitter — why the tail matters:
   new latency samples:  50ns 60ns 55ns 50ns ... 8000ns (syscall) ... 55ns
                                              ▲
                              this one spike blows your P99.9 and can miss a fill
```

---

## 6. The HFT fix: allocate once, reuse forever (object pool / slab)

The standard remedy is to do all heap allocation **once, at startup** (off the hot path), then
recycle fixed-size slots with O(1), allocation-free operations during trading:

```cpp
struct Order { /* id, price, qty, ... */ };

class OrderPool {
    std::vector<Order> storage;    // allocated ONCE at startup (off hot path)
    std::vector<Order*> freeList;  // pointers to available slots
public:
    explicit OrderPool(std::size_t n) : storage(n) {
        freeList.reserve(n);
        for (auto& o : storage) freeList.push_back(&o);
    }
    Order* acquire() {             // O(1), no heap call
        Order* o = freeList.back();
        freeList.pop_back();
        return o;
    }
    void release(Order* o) { freeList.push_back(o); }  // O(1)
};
```

During trading, "allocate an order" is just "pop a pointer off a vector" — no search, no lock, no
syscall, and no cache scatter, because every slot sits inside one contiguous `storage` block that
stays hot in cache. This is the "O(1), zero-allocation hot path" that HFT engines advertise.

```
storage: one contiguous heap block, allocated once
┌──────┬──────┬──────┬──────┬──────┐
│ Ord0 │ Ord1 │ Ord2 │ Ord3 │ Ord4 │   ← cache-friendly: neighbors are neighbors in memory
└──────┴──────┴──────┴──────┴──────┘
freeList (a stack of available slots): [&Ord4, &Ord3, ...]
   acquire() = pop the back;  release(p) = push p back.  Both O(1), no allocator touched.
```

Refinements you can mention: a *free-list threaded through the objects themselves* (union the
"next-free" pointer with the payload — no separate vector), and making the pool per-thread to avoid
any synchronization. But the core idea — pre-allocate, then recycle — is the one interviewers want.

---

## Common pitfalls & UB

- **Leak**: `new` with no matching `delete` (or losing the last pointer). Use RAII owners instead.
- **Use-after-free / dangling heap pointer**: reading/writing through a pointer after `delete`.
- **Double free**: `delete` on an already-deleted (or non-heap) pointer — corrupts the allocator.
- **Mismatched forms**: `delete` on `new[]`, or `delete[]` on `new` — UB.
- **`delete` on a stack pointer** (`int x; delete &x;`) — UB; you can only `delete` what `new`
  returned.
- **Deleting through a base pointer without a virtual destructor** (Module 11) — UB / partial
  destruction.
- **Assuming `new` is cheap on the hot path** — the jitter, not the mean, is what bites HFT.
- **`new`-ing in a tight loop** when a `reserve`d `vector` or a pool would do — needless allocator
  traffic.

---

## Quiz — tough problems

**Q1.** What's wrong, and what's the fix?
```cpp
int* p = new int[10];
delete p;
```
**Answer:** Mismatched form — `new[]` must pair with `delete[]`. `delete p` is UB (the allocator
looks for the array-size header in the wrong place and skips per-element destruction). Fix:
`delete[] p;`. Better: use `std::vector<int>` and never write either.

**Q2.** Does this leak?
```cpp
void f() {
    int* p = new int(5);
    if (compute()) throw std::runtime_error("x");
    delete p;
}
```
**Answer:** Yes, on the throw path. The exception unwinds past `delete p`, so the block leaks. This
is *precisely* why raw `new` is unsafe with exceptions and why RAII (`std::unique_ptr<int>`) is
mandatory — the smart pointer's destructor runs during unwinding and frees the block.

**Q3.** Predict the behavior:
```cpp
int* a = new int(1);
int* b = a;
delete a;
delete b;
```
**Answer:** Double free (UB). `a` and `b` point at the *same* block; deleting both frees it twice,
corrupting the allocator. Aliasing raw owning pointers is the setup for this bug; `shared_ptr`
exists to solve it (Module 13).

**Q4.** Why does `new int` cost ~50–200 ns while allocating a stack `int` costs ~1 ns? Give the
mechanism.
**Answer:** Stack alloc is a single `rsp` adjustment, no metadata, cache-hot. `new` must search a
free-list / size class, update allocator metadata, possibly lock a shared arena, and occasionally
syscall to the OS for more pages (with a page fault on first touch). It's more instructions, possible
contention, and — crucially — *variable* cost.

**Q5.** In an HFT context, why prefer a pre-sized object pool over `tcmalloc`/`jemalloc`, which are
already fast?
**Answer:** Even fast general allocators have a *tail*: occasional arena refills, lock contention,
and syscalls create latency spikes. A pool has a bounded, deterministic O(1) cost (pop/push a
pointer) with zero syscalls or locks on the hot path, so P99.9 is flat. HFT optimizes the worst case,
not the mean.

**Q6.** After `delete p; p = nullptr;`, is a later `delete p;` safe?
**Answer:** Yes — `delete nullptr;` is a well-defined no-op. That's exactly why nulling after delete
is good hygiene: it neutralizes an accidental double-free (though it doesn't help other aliases still
pointing at the freed block).

**Q7.** What does this print, and is it defined?
```cpp
std::vector<int> v;
v.reserve(4);
int* p = &v[0];
v.push_back(1); v.push_back(2); v.push_back(3); v.push_back(4);
std::cout << *p;
v.push_back(5);          // exceeds capacity
std::cout << " " << *p;
```
**Answer:** The first print is defined (`1` — within reserved capacity, no reallocation). The
`push_back(5)` exceeds capacity, so the vector **reallocates** to a new, larger heap block and frees
the old one — `p` now dangles. The second `*p` is UB. Iterator/pointer invalidation on reallocation
is a classic gotcha (Module 12).

**Q8.** Why store a free-list *inside* the pooled objects (a union with `next`) rather than in a
separate `vector<Order*>`?
**Answer:** It avoids a second allocation and a second cache line: the "next free" link lives in the
same slot as the payload (only one is active at a time), so the free-list touches memory you were
going to touch anyway. It's the intrusive free-list — smaller footprint, better locality, no
side vector to keep in sync.

---

## Indian HFT interview questions

Asked on India desks — Tower Research (Gurgaon), Graviton (Gurgaon), Optiver, IMC, Quadeye,
AlphaGrep, WorldQuant, Da Vinci, NK Securities, Squarepoint, Jump.

**Q. "Why do HFT systems avoid `new`/`delete` on the hot path?"**
*Model answer:* Not because the mean latency is high but because it's *unpredictable* — the
allocator may search, lock a shared arena, or syscall to the OS, producing latency spikes that blow
the P99.9 tail. We pre-allocate at startup and recycle fixed-size slots from a pool so the hot path
is O(1), lock-free, and syscall-free with deterministic latency.

**Q. "Design an allocator for fixed-size `Order` objects that never touches the OS during trading."**
*Model answer:* A slab/object pool: allocate one contiguous array of N `Order`s at startup, keep a
free-list of available slots (ideally intrusive — thread the free link through the unused slot).
`acquire` pops, `release` pushes, both O(1). Make it per-thread to avoid synchronization; size N to
the max expected live orders. Contiguity keeps the working set cache-hot.

**Q. "What are the classic memory bugs and how does modern C++ prevent them?"**
*Model answer:* Leak, use-after-free, double free, mismatched `new`/`delete[]`. RAII owners
(`unique_ptr`, `shared_ptr`, `vector`) tie the heap lifetime to a stack object so cleanup is
automatic and exception-safe, eliminating leaks and most dangling; `shared_ptr` handles shared
ownership so no one double-frees; `make_unique`/`make_shared` avoid raw `new` entirely.

**Q. "What happens to a `vector`'s elements on `push_back` past capacity, and why does it matter?"**
*Model answer:* It allocates a new larger block (typically 1.5–2× growth), moves/copies the elements
over, and frees the old block — invalidating all existing pointers, references, and iterators into
the vector. On the hot path this is a hidden allocation + O(n) copy + jitter, so we `reserve()`
up front or use a pool.

**Q. "Latency vs jitter — which does HFT optimize, and why does that change your allocator choice?"**
*Model answer:* Jitter (tail variance). A trade missed because of one 8 µs allocation spike costs
real money regardless of a good average. So we choose deterministic O(1) structures (pools,
ring buffers) over average-fast-but-spiky ones (general `malloc`).

**Q. "Stack vs heap — when do you *have* to use the heap?"**
*Model answer:* When the object is too big for the stack, its size is only known at runtime / it
grows, it must outlive the creating scope, or it's shared/polymorphic. Otherwise prefer the stack.
And even when the heap is required, prefer an RAII owner over raw `new`, and a pool over per-op
allocation on the hot path.

---

## In the order book

Orders come from an `OrderPool` sized at startup for the maximum expected number of live orders. The
book **never calls `new` while trading** — acquiring an order slot is a free-list pop, releasing one
(on cancel or full fill) is a free-list push, both O(1) and allocation-free. The pool's contiguous
`storage` block also gives spatial locality: orders that are allocated near each other in time sit
near each other in memory, so walking a busy price level stays cache-hot (Module 15).

The links between resting orders (the doubly-linked list at each price level) are non-owning raw
pointers or 32-bit indices *into the pool* — the pool owns the storage; nothing else ever `delete`s
an order. `std::vector<Trade>` result logs may grow, but they're `reserve()`d generously so
`push_back` doesn't reallocate mid-burst. The whole design is a direct application of this module:
heap lifetime is required (orders outlive their arrival function), but heap *latency* is engineered
out by allocating once and recycling. This single decision is a large fraction of "how it sustains
1M+ orders/sec with a flat tail."

---

## Key takeaways

- The **heap** gives objects a lifetime you control (`new`…`delete`), needed when data must outlive
  its creating scope — but you own correctness: leaks, use-after-free, double free, and mismatched
  `new`/`delete[]` are all UB.
- In modern C++ you almost never write raw `new`/`delete` — **RAII owners** (`vector`,
  `make_unique`, `make_shared`) tie heap lifetime to a stack object so cleanup is automatic and
  exception-safe.
- Prefer the stack; use the heap only when the object is too big, runtime-sized/growing, must
  outlive its scope, or is shared/polymorphic. Mental model: **handle on the stack, payload on the
  heap.**
- The heap is slow *and unpredictable*: search, locking, occasional syscalls, and cache scatter.
  HFT cares about the **tail/jitter**, which is exactly what `new` ruins.
- The HFT fix is an **object pool / slab**: allocate once at startup, then recycle fixed-size slots
  with O(1), lock-free, syscall-free `acquire`/`release`.
- `vector` reallocation on `push_back` past capacity invalidates all pointers/iterators — `reserve`
  or pool to avoid the hidden allocation and jitter.

**Next:** [Module 4 — lvalues, rvalues & an intro to moving](04-value-categories.md) — the value
categories that make it possible to *steal* resources from temporaries instead of copying them.
