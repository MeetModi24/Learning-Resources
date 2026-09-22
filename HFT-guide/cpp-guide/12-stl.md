# Module 12 — The STL: containers, iterators, algorithms, complexity & cache

The Standard Template Library gives you battle-tested containers and algorithms so you rarely
hand-roll a data structure. For HFT the key skill is *not* using them — it's knowing each one's
**complexity AND its cache behavior**, because Big-O counts operations while the hardware charges you
for **memory access patterns**. One cache miss (~100 ns / ~200–300 cycles) costs as much as a hundred
arithmetic operations, so an "O(n)" contiguous scan routinely beats an "O(1)" pointer-chasing insert.
This module explains *why*, with the memory layouts that make it obvious.

---

## 1. The two families: contiguous vs. node-based

Every container decision starts with one question: **is the data in one contiguous block, or
scattered across separately-allocated nodes?**

```
std::vector<int> v = {10,20,30,40}      (ONE heap block, contiguous)
   heap: ┌────┬────┬────┬────┐
         │ 10 │ 20 │ 30 │ 40 │   <- CPU prefetcher streams this; ~1 cache miss for the whole run
         └────┴────┴────┴────┘

std::list<int> l = {10,20,30,40}        (FOUR separate node allocations, scattered)
   node@0x100 ┌────┬─prev─┬─next─┐   next ──▶ node@0x9F0 ┌────┬────┬────┐ ──▶ 0x2C0 ──▶ 0x740
              │ 10 │ null │0x9F0 │                        │ 20 │... │... │
              └────┴──────┴──────┘
   every ── hop is a pointer chase to an unpredictable address -> a likely cache miss EACH TIME
```

Contiguous containers (`vector`, `array`, and mostly `deque`) let the hardware **prefetcher** stream
the next cache line while you process the current one — traversal is nearly free. Node-based
containers (`list`, `map`, `set`) store each element in its own heap allocation with pointer links;
every traversal step is a jump to an unpredictable address → a likely cache miss. Each node also
carries **overhead** (two link pointers for a list; two child pointers + parent + color bit for a
red-black tree node), so they use far more memory per element too.

This single distinction drives almost every HFT container verdict below.

---

## 2. Containers and their tradeoffs

| Container | Layout | Lookup | Insert | Cache behavior | HFT verdict |
|-----------|--------|--------|--------|----------------|-------------|
| `std::vector` | contiguous | O(n) scan / O(1) index | O(1) amortized at end | **excellent** | default choice |
| `std::array<T,N>` | contiguous, fixed, inline | O(1) index | — (fixed size) | excellent | when size known at compile time |
| `std::deque` | chunked (blocks) | O(1) index | O(1) both ends | good | FIFO queues |
| `std::list` | linked nodes | O(n) | O(1) *at a known position* | **terrible** (pointer chasing) | rarely — see §3 |
| `std::map` / `std::set` | red-black tree (nodes) | O(log n) | O(log n) | **poor** (node-per-element, scattered) | avoid on hot path |
| `std::unordered_map` | hash + bucket lists | O(1) avg, O(n) worst | O(1) avg | mediocre (bucket + node indirection) | ID lookups, carefully |

A few things the table understates:

- **`std::array` lives inline** — no heap at all; it's a fixed-size block *inside* the enclosing
  object/stack frame. `std::vector` always heaps its buffer (the vector object itself is just
  {pointer, size, capacity}).
- **`std::deque`** is a sequence of fixed-size chunks with an index of chunk pointers — O(1) push at
  both ends and stable element addresses, but iteration crosses chunk boundaries so it's a bit less
  cache-friendly than `vector`.
- **`unordered_map`** in the standard is a chained hash table: bucket array → per-node linked list.
  Every lookup does a hash, a bucket index, then chases node pointers — two indirections. Custom
  **open-addressing** flat maps (all entries in one contiguous array, probing on collision) are far
  more cache-friendly and are what HFT shops usually build.

---

## 3. The vector-beats-list truth (the famous result)

`std::list` advertises **O(1)** insertion in the middle; `std::vector` is **O(n)** (it must shift
elements). Yet **`vector` usually wins even for middle inserts**, up to surprisingly large `n`,
because:

- Vector elements are contiguous → the shift is a `memmove` the prefetcher streams; near-zero cache
  misses, and it's a tight, vectorizable loop.
- To insert into a `list` you first have to *find* the position — an **O(n) traversal that is all
  pointer chasing**, one cache miss per hop. The "O(1) insert" ignores the O(n)-cache-miss walk to
  reach the spot.

Big-O counts *operations* and treats every operation as equal cost. Reality: one cache miss ≈ ~100
arithmetic ops. Bjarne Stroustrup's well-known benchmark shows `vector` beating `list` for insert-
and-erase-in-order workloads across essentially all realistic sizes. **The correct HFT instinct is
"prefer `std::vector` by default; justify anything else."** This is a top-5 interview talking point —
know it cold and know *why* (cache locality + prefetching, not operation count).

---

## 4. Iterators

An iterator is a **generalized pointer** — the glue between containers and algorithms:

```cpp
std::vector<int> v{3, 1, 2};
for (auto it = v.begin(); it != v.end(); ++it) std::cout << *it;   // explicit iterator loop
for (int x : v) std::cout << x;                                     // range-for (preferred)
```

**Iterator categories** determine which algorithms work and how fast:

- **Input/Output** — single-pass, read or write once (e.g. stream iterators).
- **Forward** — multi-pass, `++` only.
- **Bidirectional** — `++` and `--` (`list`, `map`, `set`).
- **Random-access** — `+ n`, `- n`, `it[i]`, iterator subtraction in O(1) (`vector`, `array`,
  `deque`). Only random-access iterators support `std::sort` (which needs to jump around) — that's
  why you **cannot** `std::sort` a `std::list` (it has its own `.sort()` member instead).

### Iterator/pointer invalidation (a top bug source)

Mutating a container can invalidate iterators, pointers, *and references* into it. The rules you must
know:

- **`vector::push_back`/`insert` that triggers reallocation invalidates ALL iterators, pointers, and
  references** — the whole buffer moved to a new address. (If capacity was sufficient, only iterators
  at/after the insertion point are invalidated.)
- **`vector::erase`** invalidates everything from the erased position onward.
- **`std::list`** keeps node pointers/iterators valid across inserts and other elements' erasures —
  only the erased node's own iterator dies. Same stability for `std::map`/`std::set` nodes.
- **`unordered_map`**: insert may **rehash**, invalidating all *iterators* — but **references and
  pointers to elements stay valid** (nodes aren't moved, only the bucket array is rebuilt).

```
vector before push_back (capacity full):   buffer @ 0x400, it -> 0x408
                    push_back reallocates:  buffer @ 0x900  (copied+freed)
it still -> 0x408  ==  DANGLING  -> using it is UNDEFINED BEHAVIOR
```

Node stability across mutation is precisely *why* `list`/`map` still exist despite their cache
penalty — sometimes you need addresses that don't move.

---

## 5. Algorithms

Prefer standard algorithms over hand-written loops — clearer, correct, and often better optimized:

```cpp
#include <algorithm>
#include <numeric>
std::sort(v.begin(), v.end());                              // O(n log n), needs random-access iters
auto it = std::lower_bound(v.begin(), v.end(), key);        // O(log n) binary search on a SORTED range
int sum = std::accumulate(v.begin(), v.end(), 0);           // fold/reduce
auto n  = std::count_if(v.begin(), v.end(), [](int x){ return x > 0; });
v.erase(std::remove_if(v.begin(), v.end(), pred), v.end()); // erase-remove idiom (see below)
```

The **erase-remove idiom** deserves a note: `std::remove_if` doesn't shrink the container (an
algorithm can't — it only has iterators); it *shuffles* the kept elements to the front and returns an
iterator to the new logical end. `vector::erase` then physically drops the tail. Forgetting the
`erase` leaves garbage at the end — a classic bug.

C++20 **ranges** remove the `begin()/end()` noise and add lazy, composable **views** (Module 18):

```cpp
std::ranges::sort(v);                                              // no .begin()/.end()
auto evens = v | std::views::filter([](int x){ return x % 2 == 0; });  // lazy — no allocation
```

---

## 6. Lambdas — the essential companion to algorithms

```cpp
int threshold = 100;
auto expensive = [threshold](const Order& o) { return o.value() > threshold; };
//                ^capture list  ^params        ^body
std::count_if(orders.begin(), orders.end(), expensive);
```

- Capture: `[=]` by copy, `[&]` by reference, `[threshold]` names specifics. **Prefer explicit
  captures** — `[&]` on a lambda that outlives the enclosing scope is a dangling-reference bug.
- A lambda is a **compiler-generated functor** (an unnamed struct with an `operator()`). Passed as a
  *template* argument to an algorithm, its `operator()` **inlines** → zero overhead.
- Contrast `std::function`: it **type-erases** the callable, may heap-allocate for large captures,
  and adds an **indirect call** that defeats inlining. Use lambdas/functors on hot paths; reserve
  `std::function` for when you genuinely need runtime-swappable type erasure (cold paths).

---

## 7. `reserve` and small-buffer optimizations — the free wins

```cpp
std::vector<Trade> trades;
trades.reserve(10'000);   // pre-allocate capacity up front
```

If you know or can estimate the final size, `reserve` eliminates *all* the growth reallocations (each
of which copies/moves every existing element and invalidates iterators). Pure latency and
allocation-count win, no downside.

Related layout tricks the STL uses that you should recognise:

- **Small-String Optimization (SSO)**: `std::string` stores short strings (typically ≤15 chars)
  *inline* inside the string object — no heap allocation at all — and only heaps for longer strings.
  This is why passing/copying small strings is cheap and why `sizeof(std::string)` is ~32 bytes.
- **`vector` growth**: capacity grows geometrically (×1.5 or ×2), giving amortized O(1) `push_back` —
  but each growth is a full realloc+move, hence `reserve`.

---

## Common pitfalls & UB

- **Using an invalidated iterator** after `push_back`/`erase`/rehash → dangling, UB (Section 4).
- **`erase`ing in a loop wrong**: `for (auto it=v.begin(); it!=v.end(); ++it) if(pred(*it)) v.erase(it);`
  invalidates `it`. Use `it = v.erase(it);` (which returns the next valid iterator) or erase-remove.
- **`operator[]` on `std::map` inserts**: `m[key]` default-constructs an entry if `key` is missing —
  surprising in a read; use `.find`/`.contains` to just query.
- **`std::sort` on a `list`**: won't compile (bidirectional, not random-access) — use `list::sort`.
- **`[&]` capture outliving scope**: storing a lambda that captures locals by reference past their
  lifetime → dangling.
- **Assuming `unordered_map` is always O(1)**: adversarial or clustered keys → O(n) worst case
  (all in one bucket); and rehash invalidates iterators.

---

## Quiz — tough problems

**Q1.** `std::list` has O(1) insertion and `std::vector` O(n). Why does `vector` usually win for
"insert many elements in sorted order"?
**Answer:** Finding the insert position in a `list` is an O(n) *pointer-chasing* traversal (one cache
miss per hop); the `vector` shift is a contiguous `memmove` the prefetcher streams. Cache locality
beats operation count — one miss ≈ ~100 ops.

**Q2.** Is this UB?
```cpp
std::vector<int> v{1,2,3}; int* p = &v[0]; v.push_back(4); std::cout << *p;
```
**Answer:** Yes (in general). `push_back` may reallocate, moving the buffer; `p` then dangles. UB.
Only safe if `v.capacity() > v.size()` before the push (e.g. after a `reserve(4)`).

**Q3.** Why can't you `std::sort(myList.begin(), myList.end())`?
**Answer:** `std::sort` requires **random-access** iterators (it jumps by arbitrary offsets);
`std::list` provides only **bidirectional** iterators. `std::list` offers a member `.sort()` (a
merge sort that relinks nodes) instead.

**Q4.** After `unordered_map` rehashes on insert, which of {iterators, pointers, references} to
existing elements survive?
**Answer:** Pointers and references survive (the element *nodes* aren't moved); iterators are
invalidated (the bucket structure was rebuilt).

**Q5.** What's wrong with `for (auto it = v.begin(); it != v.end(); ++it) if (bad(*it)) v.erase(it);`
and give the correct forms.
**Answer:** `erase` invalidates `it` (and everything after in a vector), so `++it` is UB. Correct:
`for (auto it = v.begin(); it != v.end(); ) { if (bad(*it)) it = v.erase(it); else ++it; }` or the
erase-remove idiom `v.erase(std::remove_if(v.begin(), v.end(), bad), v.end());`.

**Q6.** Why is `std::map` a poor choice for a price→level lookup on the hot path, and what beats it?
**Answer:** It's a red-black tree — one heap node per price, scattered in memory, O(log n) with a
cache miss at each tree level. A **flat `std::vector`/array indexed by price ticks** (prices are a
bounded integer range) gives O(1) contiguous access — the cache-friendly structure the order book
uses at rung 3.

**Q7.** Two callbacks: one passed as a template parameter, one as `std::function`. Which is faster on
the hot path and why?
**Answer:** The template one. It inlines the concrete lambda's `operator()` → zero overhead.
`std::function` type-erases: possible heap allocation for the capture plus an indirect call that
prevents inlining.

**Q8.** `std::string s = "hi";` — does this allocate on the heap? What about a 100-char string?
**Answer:** No heap for `"hi"` — **Small-String Optimization** stores short strings inline in the
string object. A 100-char string exceeds the inline buffer, so it heap-allocates.

**Q9.** `std::deque` vs `std::vector` for a FIFO where you push at the back and pop at the front?
**Answer:** `deque` — O(1) at both ends and no shifting. A `vector` pop-front is O(n) (shift
everything). `deque` gives up a little cache locality (chunked) for cheap front operations.

---

## Indian HFT interview questions

*(Asked at Tower Research Gurgaon, Optiver, IMC, Graviton, Quadeye, AlphaGrep, WorldQuant, Da Vinci,
Squarepoint, Jump, HRT, NK Securities, Millennium.)*

- **"Why is `std::vector` faster than `std::list` despite O(n) inserts?"** *(the classic)*
  Cache locality + hardware prefetching. Vector is one contiguous block streamed by the prefetcher; a
  list scatters nodes, so every traversal hop is a likely cache miss (~100 ns), and even the "O(1)
  insert" needs an O(n) pointer-chasing walk to find the spot. Big-O ignores that a cache miss ≈ 100
  ops.

- **"`std::map` vs `std::unordered_map` — internals and when each?"**
  `map` = red-black tree: ordered iteration, O(log n), node-per-element (poor cache). `unordered_map`
  = chained hash: O(1) average, unordered, bucket+node indirection, O(n) worst case, rehash
  invalidates iterators. Use `map` when you need ordering, `unordered_map` for pure key lookup — and
  on the hot path prefer a flat/open-addressing map or a direct-indexed array.

- **"Design a faster hash map than `std::unordered_map` for order-ID lookup."**
  Open addressing (linear/robin-hood probing) in a single contiguous array — no per-node allocation,
  no pointer chasing, everything in cache. Power-of-two size for mask indexing, a good hash on the
  64-bit ID, and a load factor cap. That's the "flat_hash_map" pattern (Abseil / boost).

- **"Explain iterator invalidation for vector, list, and unordered_map."**
  vector: reallocation (push/insert past capacity) invalidates *all* iterators/pointers/refs; erase
  invalidates from the point on. list/map: only the erased element's iterator dies; others (and
  pointers/refs) stay valid. unordered_map: rehash invalidates iterators but keeps pointers/refs.

- **"What is the erase-remove idiom and why is it needed?"**
  Algorithms only have iterators, so `std::remove_if` can't shrink the container — it compacts kept
  elements to the front and returns the new end; `vector::erase(newEnd, end())` then drops the tail.
  Two steps because "remove" (algorithm) and "erase" (container) are separate.

- **"When would you *still* choose `std::list` in low-latency code?"**
  When you need **stable node addresses** across insert/erase, or O(1) splice/relink at a known
  position — e.g. an **intrusive** doubly-linked list of orders at a price level, where nodes are
  embedded in pooled objects (so no per-node allocation and the cache penalty disappears).

- **"Lambda vs `std::function` — performance?"**
  Lambda passed as a template arg inlines → zero cost. `std::function` type-erases → possible heap
  allocation + indirect call, no inlining. Use lambdas on the hot path; `std::function` only where
  runtime swappability is genuinely required.

---

## In the order book

- **Orders within a price level**: a `std::list`-like **intrusive** doubly-linked list — the prev/
  next links live *inside* the pooled `Order` object, not in separately-allocated nodes — giving O(1)
  cancel at a known position while pooling removes the usual list cache penalty (nodes are contiguous
  in the pool).
- **Order-ID → order**: `std::unordered_map` in the naive version, replaced by a custom
  **open-addressing** flat map for O(1) cancel/modify lookup without per-node allocation.
- **Price levels**: `std::map<Price, Level>` in the naive version (rung 1) for ordered iteration,
  replaced by a **flat `std::vector`/array indexed by price ticks** in the optimized version
  (rung 3) — the cache win that beats the tree, exactly because prices are a bounded integer range.
- **Result logs** (`std::vector<Trade>`): `reserve`d up front so growth never reallocates mid-match.

---

## Key takeaways

- The first question for any container is **contiguous vs. node-based** — it decides cache behavior,
  which usually dominates Big-O in practice.
- **Prefer `std::vector` by default; justify anything else.** It beats `std::list` even for
  middle-insert workloads because cache locality + prefetching outweigh operation counts.
- Know **complexity *and* cache profile** per container, and the **iterator-invalidation** rules
  (vector realloc kills everything; list/map keep nodes stable; unordered_map rehash kills iterators
  but keeps refs).
- Use **standard algorithms** (`sort`, `lower_bound`, erase-remove) and C++20 **ranges/views**; pass
  callables as **lambdas/template params** (inlinable), not `std::function`, on the hot path.
- `reserve` when the size is known, and remember **SSO** makes short strings allocation-free — small
  layout facts with real latency impact.

**Next:** [13 — Smart pointers & ownership](13-smart-pointers.md) — who owns the heap allocations, and
how RAII automates freeing them.
