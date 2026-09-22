# Module 15 — Memory model, cache, alignment, false sharing

This is where "systems programming" stops being an abstraction and becomes literal physics: you
optimize for how the CPU *physically* moves bytes from DRAM into the registers where arithmetic
happens. Up to now the mental model has been "memory is a flat array of bytes you access in one
step" (Module 1). That model is a **lie the hardware tells you** — a convenient fiction. In reality
there is a steep hierarchy of caches between the core and RAM, and in HFT the difference between an
L1 hit (~1 ns) and a main-memory miss (~100 ns) is, quite literally, the whole game. A strategy that
computes the right answer 100 ns too late has computed the wrong answer.

This module has almost no new *syntax*. It's about a mental model of the machine that turns
"correct C++" into "fast C++." Everything here is invisible in the language spec and visible only in
a profiler.

---

## 1. Why memory is the bottleneck (the "memory wall")

For 40 years CPU clock speeds grew far faster than DRAM latency improved. The result: a modern core
can execute ~4 instructions per cycle at ~3 GHz, but a single access to main memory takes **200+
cycles**. In the time it takes to fetch one `int` from RAM, the core could have executed *hundreds*
of arithmetic instructions. This gap is called the **memory wall**, and it inverts the intuition
most beginners bring from a first CS course.

The beginner's instinct is to count operations: "this loop does N multiplies, that one does 2N, so
the first is faster." On modern hardware that instinct is usually **wrong**. What dominates is not
how many instructions you run but **how often you wait for memory**. An algorithm doing 2× the
arithmetic but touching memory in a cache-friendly, sequential pattern will crush a
"fewer-operations" algorithm that chases pointers all over the heap.

> **The reframe:** stop counting instructions, start counting *cache misses*. The CPU is fast; RAM
> is slow; the caches are the machinery that hides the gap — and your job is to keep the data they
> need close.

---

## 2. The memory hierarchy

The caches form a pyramid: small-and-fast at the top, huge-and-slow at the bottom. Each level is a
staging area for the level below it.

```
           ┌─────────────┐
 registers │  ~dozens    │  0 cyc      (data the core is using RIGHT NOW)
           ├─────────────┤
 L1  32-64K│  ~4 cycles  │  ~1 ns      per-core, split I-cache + D-cache
           ├─────────────┤
 L2 256K-1M│  ~12 cycles │  ~4 ns      per-core (usually)
           ├─────────────┤
 L3   ~MBs │  ~40 cycles │  ~15 ns     SHARED across all cores
           ├─────────────┤
 DRAM  GBs │ ~200+ cycles│  ~100 ns    main memory
           ├─────────────┤
 NVMe/SSD  │  ~10-100 µs │             (disk — off the map for hot paths)
           └─────────────┘
```

Ballpark numbers (Intel-class server, 2020s — memorize the *ratios*, not exact figures):

| Level             | Latency (cycles) | Latency (time) | Size            |
|-------------------|------------------|----------------|-----------------|
| Register          | 0                | —              | dozens          |
| L1 cache          | ~4               | ~1 ns          | 32–64 KB        |
| L2 cache          | ~12              | ~4 ns          | 256 KB–1 MB     |
| L3 cache          | ~40              | ~15 ns         | several–tens MB |
| Main memory (RAM) | ~200+            | ~100 ns        | GBs             |

A main-memory access costs **~100× an L1 hit** — the time of 100+ arithmetic instructions. When the
data you need is in L1, the core barely pauses. When it's in DRAM, the core *stalls* — it literally
sits idle for ~200 cycles waiting, unless it can find independent work to do (out-of-order
execution) to fill the gap. In a tight, dependent computation there is no independent work, so a
miss is a dead stop.

Every access first checks L1; miss → check L2; miss → L3; miss → DRAM. The data then gets *filled*
back up the hierarchy so the next access to it (or its neighbors) is fast. This is why locality —
reusing data and its neighbors soon — is everything.

---

## 3. Cache lines — the unit of transfer

Here is the single most important fact in this module: **the CPU never loads one byte. It loads a
whole cache line** — a fixed-size, aligned block, **64 bytes on x86-64** (and Apple/ARM often 64
or 128). When you read `arr[0]`, the hardware hauls in `arr[0..15]` (sixteen 4-byte ints) as one
unit, because they live in the same 64-byte line.

```
memory addresses:  ... 0x1000        0x1040        0x1080 ...
                       │             │             │
                    ┌──┴────────────┐┌─────────────┐┌──────────
 cache LINE (64B):  │ b0 b1 ... b63 ││ b0 ...   b63 ││ ...
                    └───────────────┘└─────────────┘
                    one coherency unit  — loaded/evicted/tracked as ONE thing
```

Three consequences flow from this, and they explain most of practical performance:

- **Spatial locality is free.** Data used together should *sit* together. Touching `arr[i]` pulls in
  `arr[i+1] … arr[i+15]` at no extra cost — the next iterations are already resident.
- **Sequential access is nearly free.** The hardware **prefetcher** detects a linear scan (you
  touched line N, then N+1, then N+2…) and speculatively loads lines *ahead* of you. Walk an array
  front-to-back and you approach zero misses; the data arrives before you ask.
- **Scattered access is murder.** Pointer-chasing (linked list, tree, hash-map-of-nodes) jumps to a
  random address each hop. Every hop is likely a fresh line, likely a miss, and the prefetcher can't
  predict where you'll go next. Each hop can cost ~100 ns.

This is *the* reason `std::vector` beats `std::list`, and a flat array beats `std::map`, far beyond
what their big-O suggests (Module 12). Big-O counts operations; it says nothing about whether each
operation is a 1 ns L1 hit or a 100 ns DRAM miss.

### Cache-line math (an interview staple)

If `struct Order` is 32 bytes, then `64 / 32 = 2` orders fit per line. Iterating a
`std::vector<Order>` touches a new line every 2 elements → ~1 miss per 2 orders on a cold array, but
the prefetcher hides even that on a sequential scan. If `Order` were 96 bytes, each spans *two*
lines, halving effective bandwidth and wasting the tail of each line if you only read one field.
**Shrinking a hot struct so more instances fit per line is one of the highest-leverage
optimizations there is.**

---

## 4. Alignment and padding — struct layout matters

Each scalar type must sit at an address that is a multiple of its size (a `double` at an 8-byte
boundary, an `int` at 4, etc.) — this is **alignment**, and the hardware requires it (misaligned
access is slow or, on some ISAs, a fault). To enforce it inside a struct, the compiler inserts
invisible **padding** bytes. The order you declare members in therefore changes `sizeof`.

```cpp
struct Bad {
    char   a;      // offset 0, 1 byte
                   // offset 1..7: 7 PADDING bytes so b lands on an 8-boundary
    double b;      // offset 8, 8 bytes
    char   c;      // offset 16, 1 byte
                   // offset 17..23: 7 PADDING bytes so sizeof is a multiple of 8
};  // sizeof(Bad) == 24
```

```
Bad in memory (24 bytes, 15 wasted):
 offset:  0    1                8                16   17            23
         [a][ pad pad pad pad pad pad pad ][    b (double)    ][c][ pad ... pad ]
          1B  └──── 7 wasted ─────────────┘   8B               1B └── 7 wasted ─┘
```

```cpp
struct Good {
    double b;      // offset 0, 8 bytes
    char   a;      // offset 8, 1 byte
    char   c;      // offset 9, 1 byte
                   // offset 10..15: 6 padding bytes to round sizeof to 16
};  // sizeof(Good) == 16
```

```
Good in memory (16 bytes, 6 wasted):
 offset:  0                8   9   10          15
         [    b (double)    ][a][c][ pad ... pad ]
          8B                 1B 1B └─ 6 wasted ──┘
```

**Rule: order members largest-alignment → smallest.** It minimizes padding, shrinks the object, and
packs more instances per cache line. `alignof(T)` tells you the required alignment; `sizeof(T)`
tells you the real footprint (padding included); `alignas(N)` lets you *force* a stricter alignment
(used below for false sharing). A common trick to see the waste: sum the member sizes and compare to
`sizeof` — the difference is pure padding.

`[[no_unique_address]]` (C++20) lets an empty member (e.g. a stateless comparator/allocator) occupy
zero bytes, another way to keep hot structs tight.

---

## 5. Data-oriented design: AoS vs SoA

Once you think in cache lines, you realize the *shape* of your data — not just its size — decides
how much of each loaded line you actually use.

- **Array of Structs (AoS):** `struct P { double x, y, z; }; P pts[N];`. All three fields of one
  point are adjacent. Great when you process points *whole* (use x, y, z together).
- **Struct of Arrays (SoA):** `double xs[N], ys[N], zs[N];`. All the x's are contiguous, separate
  from y's and z's. Great when you sweep *one field across all elements*.

```
AoS:  [x0 y0 z0][x1 y1 z1][x2 y2 z2]...   iterating just x → loads y,z too (wasted)
SoA:  [x0 x1 x2 ...][y0 y1 y2 ...]...      iterating just x → every loaded byte is an x
```

If you iterate a million orders touching only `price`, AoS drags each order's `qty`, `timestamp`,
etc. into cache alongside the price you wanted — most of every line is wasted. SoA loads *only*
prices: full bandwidth utilization, and it's SIMD-friendly (the CPU can vector-process 8 contiguous
prices at once). Choose based on the dominant access pattern. This is the heart of **data-oriented
design**: lay data out for the loop that's hot, not for how a human likes to think about an
"object."

### Hot/cold splitting

A related lever: keep frequently-touched ("hot") fields together and banish rarely-touched ("cold")
fields to a side array, so the hot data packs densely and no line is diluted by cold bytes.

```cpp
struct Order {          // HOT — touched on every match
    std::int64_t  price;
    std::uint32_t qty;
    std::uint32_t next;   // link index (Module 2/15): 32-bit index, not 64-bit pointer
};                        // 16 bytes → 4 per cache line

// COLD — parked in a parallel array, indexed by the same id:
struct OrderMeta { std::uint64_t timestamp; char clientTag[16]; };
```

Now a scan over `Order[]` during matching never loads a single timestamp byte.

---

## 6. False sharing — the multicore trap

This is the #1 multicore performance bug and a top-tier HFT interview question. To understand it you
need one fact about cache coherency (MESI, next section): **cores keep the caches consistent at
cache-line granularity.** When one core writes *any* byte of a line, every other core's cached copy
of that *entire* line is invalidated and must be re-fetched.

Now the trap: two threads write two *different* variables that happen to live on the *same* 64-byte
line. Logically there is no sharing — they touch different data. But physically the hardware sees one
line being written by two cores, so it **ping-pongs the line between the cores' caches**, each write
stealing the line back and invalidating the other core's copy. Throughput collapses; you can lose
10–100× and be baffled, because the code looks embarrassingly parallel.

```cpp
struct Counters {
    std::atomic<int> a;   // thread 1 hammers this
    std::atomic<int> b;   // thread 2 hammers this
};                        // a and b share ONE 64-byte line → FALSE SHARING
```

```
       core 0                         core 1
   writes Counters.a              writes Counters.b
        │                              │
        ▼                              ▼
   ┌──────────────── same 64B line ───────────────┐
   │  [a][b][.......................padding........]│
   └───────────────────────────────────────────────┘
        ▲                              ▲
        └────── line bounces back and forth ────────┘
     each write invalidates the other core's copy → coherency storm
```

The fix: pad/align so each variable owns its own line.

```cpp
struct Counters {
    alignas(64) std::atomic<int> a;   // own cache line
    alignas(64) std::atomic<int> b;   // own cache line
};
// Portable spelling of "64" (C++17):
//   std::hardware_destructive_interference_size  — min separation to AVOID false sharing
//   std::hardware_constructive_interference_size — max togetherness to SHARE a line on purpose
```

This is a real, recurring bug in the SPSC-queue projects this track builds toward: the producer's
`tail_` index and the consumer's `head_` index must sit on separate lines, or the two threads fight
over one line on every single push/pop (see Module 16).

---

## 7. Cache coherency (MESI) in one screen

You don't need to implement it, but you must be able to *explain* it, because false sharing,
acquire/release, and atomics all sit on top of it. Each cache line, in each core's cache, is in one
of four states (**MESI**):

- **M**odified — this core has the only copy and has changed it; RAM is stale. Must write back
  before anyone else reads.
- **E**xclusive — this core has the only copy and it matches RAM. Can write freely (→ M) without
  telling anyone.
- **S**hared — multiple cores hold read-only copies, all matching RAM.
- **I**nvalid — this copy is stale/absent; must re-fetch to use.

The rule that matters: **to write a line, a core must own it exclusively (E or M)**, which forces
every other core's copy to Invalid. That invalidation traffic — coherency messages on the
interconnect — is the physical cost behind false sharing and behind why atomic writes aren't free.
When two cores write the same line, they keep yanking it into M on their side and I on the other:
the "coherency storm."

---

## 8. Branch prediction & prefetching

Two more hardware realities that shape hot-path code:

- **Branch prediction.** The CPU pipeline is ~15–20 stages deep; it can't wait to *know* which way a
  branch goes, so it *guesses* and speculatively executes down the predicted path. A correct guess
  is free. A **misprediction** flushes the whole pipeline — ~15–20 wasted cycles. Predictable
  branches (loop conditions, "this error basically never happens" checks) cost nothing; the
  predictor learns them. *Data-dependent, 50/50 random* branches are the killers. `[[likely]]` /
  `[[unlikely]]` (C++20) hint the compiler which path is hot so it lays out code to favour it.
  Branch-*free* code (Module 17) removes the risk entirely on the hottest paths.
- **Prefetching.** The hardware prefetcher is excellent at *linear* patterns — lean on it by keeping
  data contiguous. `__builtin_prefetch(ptr)` lets you *manually* hint an upcoming access (e.g. next
  order in a linked structure) a few iterations early, but measure: a bad prefetch pollutes cache
  and hurts. Default to helping the hardware prefetcher (sequential layout) rather than fighting it.

---

## 9. Common pitfalls & UB

- **Optimizing instruction count instead of misses.** The classic beginner mistake; benchmark
  before believing any micro-optimization.
- **`std::list`/`std::map`/node-based containers on the hot path.** Each element is a separate heap
  allocation at a random address — a pointer-chase and likely a miss per element. Prefer contiguous
  containers (Module 12).
- **Fat hot structs.** Every cold byte you leave in a hot struct dilutes every cache line you load.
- **Assuming member order doesn't matter.** It changes `sizeof` and packing density.
- **False sharing you can't see.** Two adjacent per-thread counters, or a lock next to the data it
  guards, silently share a line. Not UB — just a stealth 50× slowdown.
- **Misaligned reinterpret_cast / packed structs.** Reading a `double` through a misaligned pointer
  is UB on some ISAs and slow on x86; `#pragma pack` to remove padding trades size for misaligned
  (slow/UB) accesses — rarely worth it.
- **Benchmarking a cold cache.** The first pass pays all the misses; warm the cache (or measure
  steady-state) or your numbers are meaningless.

---

## 10. Quiz — tough problems

**Q1.** `struct Order { char side; std::int64_t price; std::int32_t qty; };` — what is
`sizeof(Order)` and how much is padding? Reorder to minimize it.
**Answer:** `side` at 0 (1B), then 7 pad bytes so `price` aligns to 8 → `price` at 8 (8B), `qty` at
16 (4B), then 4 tail pad bytes to make `sizeof` a multiple of 8 (the struct's alignment = 8) →
**`sizeof == 24`, 11 bytes padding.** Reordered `{ int64 price; int32 qty; char side; }`: price@0,
qty@8, side@12, 3 tail pad → **`sizeof == 16`**, only 3 wasted.

**Q2.** A `std::vector<Order>` where `sizeof(Order)==16`, on a 64-byte line: how many orders per
line, and roughly how many DRAM misses to scan 1,000,000 orders cold vs. warm?
**Answer:** 4 per line. Cold scan: ~1,000,000 / 4 = **250,000 lines**, but the hardware prefetcher
turns a sequential scan into near-zero *stalls* (it fetches ahead), so effective miss latency is
largely hidden. Warm (already resident): ~0 misses. The point: sequential layout lets the
prefetcher amortize the 250k line fills.

**Q3.** You have `alignas(64) std::atomic<int> a; alignas(64) std::atomic<int> b;` — thread 1
increments `a` in a tight loop, thread 2 increments `b`. Is there false sharing? Now remove the
`alignas`. What changes?
**Answer:** With `alignas(64)` each atomic is on its own line → **no false sharing**, both threads
scale. Without it, `a` and `b` share one line → **false sharing**, the line ping-pongs and
throughput can drop 10–100×, despite zero logical contention.

**Q4.** Struct `P{double x,y,z;}`, `N=10^6`. Loop A sums all `x`. Loop B computes `x*x+y*y+z*z` per
point. Which layout (AoS vs SoA) wins for each?
**Answer:** Loop A (one field): **SoA** — it streams only the `xs[]` array, every loaded byte used;
AoS drags y,z along, wasting 2/3 of bandwidth. Loop B (all fields together): **AoS** — one point's
x,y,z are on the same line, one load serves the whole computation; SoA would touch three separate
arrays (three streams).

**Q5.** Two threads, no atomics, no locks: thread 1 writes `g_x = 1`, thread 2 reads `g_x`. Is this
a data race? Does putting `g_x` in its own cache line fix it?
**Answer:** **Yes, a data race → UB** (Module 16). Cache-line separation fixes *false sharing*
(a performance problem), not *data races* (a correctness/UB problem). Different problems — you need
atomics or a lock for correctness regardless of layout.

**Q6.** Why can an algorithm doing 2× the arithmetic beat one doing half as much?
**Answer:** Because runtime is dominated by memory stalls, not instruction count. If the
"more arithmetic" version has a cache-friendly sequential access pattern (few misses) and the
"less arithmetic" version pointer-chases (many ~100 ns misses), the memory behaviour dwarfs the
arithmetic difference. Count misses, not ops.

**Q7.** You add one `std::string clientId` field to a hot 16-byte `Order` struct used in a tight
matching loop. Latency doubles even though you never read `clientId` in the loop. Why?
**Answer:** `std::string` is ~32 bytes (Module 12) → `Order` balloons to ~48+ bytes, so fewer orders
fit per line (1–2 instead of 4), and every loaded line now carries dead `clientId` bytes. The hot
loop's effective bandwidth and packing collapse. Fix: hot/cold split — move `clientId` to a side
array indexed by order id.

**Q8.** `#pragma pack(1)` removes all padding from `Order`, shrinking it. Free win?
**Answer:** No. It packs fields to unaligned offsets. Every access to a misaligned `int64` now costs
an extra load/merge on x86 (and is UB / faults on stricter ISAs). You trade a little size for
misaligned-access cost — almost never worth it on a hot struct; reorder fields instead.

**Q9.** Where would you place a `std::mutex` relative to the data it protects, cache-line-wise?
**Answer:** Ideally the lock and its data share a line if they're always accessed together by the
*same* core (constructive interference) — but if multiple cores contend the lock, the lock word
itself gets hammered; keep unrelated hot data off that line so lock traffic doesn't invalidate it.
The nuance being tested: locks generate the same coherency traffic as any other written line.

---

## 11. Indian HFT interview questions

These come up constantly at Tower Research (Gurgaon), Optiver, IMC, Graviton, Quadeye, AlphaGrep,
WorldQuant, Da Vinci, Squarepoint, Jump, HRT, NK Securities, Mansard, and the quant desks of
Millennium/Qube. Cache and layout are *core* C++ HFT territory — expect these in round 1.

- **"What is a cache line and why does its size matter?"** — 64 bytes on x86; it's the unit of
  transfer and coherency. It matters because it decides how many objects load per fetch (packing),
  makes sequential access fast (prefetch), and is the granularity at which false sharing happens.

- **"What is false sharing? How do you detect and fix it?"** — Two threads writing different
  variables on the same line, forcing coherency ping-pong. Detect: `perf c2c`, or a suspicious
  slowdown in "embarrassingly parallel" code that scales negatively with more threads. Fix:
  `alignas(std::hardware_destructive_interference_size)` (i.e. 64) so each hot variable owns a line;
  pad structures between per-thread data.

- **"Roughly how many cycles is an L1 hit vs a main-memory miss?"** — ~4 cycles vs ~200+; a ~100×
  gap. Know the ratio; they want to hear you *think* in it.

- **"AoS vs SoA — when would you pick each for market data?"** — SoA when you sweep one field across
  many instruments (e.g. compute mid-price over all symbols, or vectorize) — full bandwidth, SIMD.
  AoS when you process one instrument's whole record at once. Order books often go SoA / column
  layout for the price-ladder.

- **"How do you make a struct smaller without dropping fields?"** — Reorder largest→smallest to kill
  padding; shrink types (32-bit index instead of 64-bit pointer; `int32` price-in-ticks instead of
  `double`); hot/cold split; `[[no_unique_address]]` for empty members; bit-pack flags.

- **"Why is `std::list` slower than `std::vector` even when both are O(1) for your operation?"** —
  `list` nodes are separate heap allocations at scattered addresses → a cache miss (~100 ns) per
  hop, no prefetch, no packing. `vector` is contiguous → prefetcher-friendly, many elements per
  line. Big-O hides the 100× constant.

- **"What's the cost of a branch misprediction, and how do you avoid it on the hot path?"** — ~15–20
  cycles (pipeline flush). Avoid with predictable branches, `[[likely]]`/`[[unlikely]]`, or
  branch-free code (conditional moves, arithmetic-select) — Module 17.

- **"Explain cache coherency at a high level."** — MESI: each line, per core, is Modified/Exclusive/
  Shared/Invalid; writing requires exclusive ownership, invalidating others' copies. This is the
  machinery behind atomics' cost and false sharing.

- **"How would you lay out an order book for cache efficiency?"** — Contiguous array of price levels
  indexed by tick (no pointer chasing), compact hot `Order` struct, 32-bit link indices, hot/cold
  split, per-thread queue indices on separate lines. (See below.)

---

## 12. In the order book

Everything above converges on concrete design decisions:

- **`Order` is a compact, alignment-optimized struct.** Hot fields (price-in-ticks, qty, link
  index) are packed largest→smallest into ~16 bytes → 4 orders per cache line. Cold fields
  (timestamps, client tags) are split into a parallel side array indexed by order id, so the
  matching scan never loads a cold byte.
- **Price levels live in a contiguous array indexed by price tick**, not a `std::map`/AVL tree.
  Sequential, prefetcher-friendly, zero pointer-chasing. This is the single biggest win over a
  tree-based design — recall the finding that AVL rebalancing dominated latency: it wasn't the
  `O(log n)` rotations per se, it was the cache misses from chasing scattered tree nodes.
- **32-bit indices, not 64-bit pointers, for the intrusive linked list at each price level.** Two
  links now fit where one pointer did → denser lines, and the indices double as offsets into the
  contiguous order pool.
- **If the pipeline is multithreaded** (input thread → matching thread via an SPSC queue), the
  producer's `tail_` and consumer's `head_` get `alignas(64)` to kill false sharing — the exact fix
  from Section 6, and the bridge into Module 16.

## Key takeaways

- Runtime is dominated by **cache misses, not instruction count** — the L1↔DRAM gap is ~100×. Count
  misses.
- The CPU transfers **64-byte cache lines**, not bytes: spatial locality and sequential access are
  nearly free; pointer-chasing is ~100 ns per hop.
- **Struct layout matters:** order members largest→smallest to kill padding, shrink hot structs so
  more fit per line, and hot/cold-split cold fields into side arrays.
- **AoS vs SoA** is chosen by access pattern: SoA to sweep one field (bandwidth + SIMD), AoS to
  process whole records.
- **False sharing** — two threads writing different variables on one line — silently costs 10–100×;
  fix with `alignas(64)` / `hardware_destructive_interference_size`.
- **MESI coherency** (write needs exclusive ownership → invalidations) is the physics behind false
  sharing and atomic cost; **branch mispredictions** cost ~15–20 cycles.
- Contiguous, compact, cache-line-aware data structures are why the array-based order book beats the
  tree-based one by far more than big-O predicts.

**Next:** [16 — `std::atomic`, memory ordering, lock-free basics](16-atomics-lockfree.md) — building
predictable-latency concurrency on top of the coherency machinery you just learned.
