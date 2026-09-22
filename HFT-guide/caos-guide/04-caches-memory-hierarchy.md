# Module 04 — Caches & the memory hierarchy

The single most important fact in low-latency programming: **the CPU is starving.** A modern core
can execute several instructions per nanosecond, but a read from main memory (DRAM) takes ~60–100 ns
— hundreds of wasted cycles. Caches exist to hide that gap, and almost every "why is my code slow?"
answer in HFT bottoms out in a cache miss. This module builds the memory hierarchy from zero: why it
exists, how a cache is physically organized, and how an address becomes a cache lookup.

---

## 1. The memory wall — why caches must exist

CPU speed and DRAM speed have diverged for decades. Cores got exponentially faster; DRAM latency
barely improved (it's limited by capacitor physics, not transistor count). The result is the
**memory wall**: the processor spends most of its time *waiting for memory*, not computing.

```
      Performance (log scale, historical)
  ^
  |                                   ___----  CPU (≈ +50%/yr for years)
  |                          ___-----
  |                  ___-----
  |          ___-----
  |   ___----                          ______  DRAM (≈ +7%/yr)  <-- the "wall"
  +-----------------------------------------------> time
```

Concretely, on a ~3 GHz core (1 cycle ≈ 0.3 ns):

- An add: ~1 cycle.
- A DRAM load: ~200+ cycles.

If every load hit DRAM, a 3 GHz core would effectively run like a 15 MHz one. Caches — small, fast
SRAM buffers holding recently/nearby-used data — are the fix. They work because real programs have
**locality**:

- **Temporal locality:** if you touched an address, you'll likely touch it again soon (loop
  counters, a hot `Order` object).
- **Spatial locality:** if you touched an address, you'll likely touch its neighbors soon (walking
  an array, reading struct fields).

Caches exploit both: they keep recently-used data (temporal) and fetch *whole lines* of neighbors at
once (spatial).

---

## 2. The hierarchy pyramid

Memory is arranged as a pyramid: small+fast at the top, large+slow at the bottom. Each level is a
cache of the one below it.

```
             ┌───────────────┐
             │   registers   │   ~0 cyc     handful of bytes ×  ~16-32 GPRs
           ┌─┴───────────────┴─┐
           │      L1 (I + D)   │  ~4 cyc    32 KB each, per-core
         ┌─┴───────────────────┴─┐
         │          L2           │ ~12 cyc   256 KB–1 MB, per-core
       ┌─┴───────────────────────┴─┐
       │           L3 (LLC)        │ ~40 cyc   8–36 MB, shared across cores
     ┌─┴───────────────────────────┴─┐
     │             DRAM              │ ~200 cyc  16–512 GB
   ┌─┴───────────────────────────────┴─┐
   │       SSD / NVMe / disk           │ ~10⁴–10⁶× worse
   └───────────────────────────────────┘
```

| Level | Typical size | Latency (cycles) | Latency (ns @3GHz) | Scope |
|-------|-------------|------------------|--------------------|-------|
| Register | ~256 B | 0 (in-pipeline) | — | per-core |
| L1d / L1i | 32 KB / 32 KB | ~4 | ~1.3 ns | per-core |
| L2 | 256 KB–1 MB | ~12 | ~4 ns | per-core |
| L3 (LLC) | 8–36 MB | ~40 | ~13 ns | shared |
| DRAM | 16–512 GB | ~200 | ~65–100 ns | shared |
| NVMe SSD | TB | ~10⁵ | ~50–100 µs | shared |

The famous "**latency numbers every programmer should know**" — memorize the *ratios*, not the exact
values. The interview-relevant one: **L1 is ~50× faster than DRAM.** Missing all the way to DRAM on
the hot path is a latency catastrophe.

---

## 3. SRAM vs DRAM — why fast memory is small and expensive

The two levels differ in the physical storage cell:

- **SRAM** (static RAM) — a **bit is 6 transistors** (a flip-flop). It holds its value as long as
  it's powered, needs no refresh, and can be read in ~1 cycle. But 6 transistors/bit is *huge* and
  power-hungry, so you can only afford kilobytes to megabytes of it. This is what caches are made of.
- **DRAM** (dynamic RAM) — a **bit is 1 transistor + 1 capacitor**. Tiny and dense (gigabytes are
  cheap), but the capacitor leaks, so it must be **refreshed** thousands of times per second, and
  reading it is slow (sense the tiny charge, then recharge). This is main memory.

```
SRAM cell (6T, fast, big):        DRAM cell (1T1C, slow, tiny):
   ___     ___                        wordline
  |   |>o<|   |   (cross-coupled          |
  |___|   |___|    inverters =         ───┴───  transistor
    |       |       stable latch)         │
  access   access                       ─┴─  capacitor (leaks → refresh)
```

That physical trade-off *is* the reason the hierarchy exists: you cannot have memory that is
simultaneously huge, fast, and cheap. So you stack a little fast memory on top of a lot of slow
memory and let locality do the rest.

---

## 4. Cache lines — the unit of transfer

A cache never moves a single byte. It moves a **cache line** — almost universally **64 bytes** on
x86-64 and AArch64. When you read one `int`, the hardware pulls in the entire aligned 64-byte block
containing it.

```
Address space, sliced into 64B lines:
 ... | line k-1 | line k                              | line k+1 | ...
                 └─ 64 bytes: [b0 b1 b2 ... b63] ─┘
                     reading b5 caches ALL of b0..b63
```

This is the hardware mechanism behind spatial locality, and it has huge practical consequences:

- **Sequential array traversal is fast**: one miss per 64 bytes, then 15 more `int`s (or 8 `int64`s)
  come free from the same line.
- **Pointer-chasing (linked lists, trees) is slow**: each node is a fresh, likely-uncached line — a
  miss per hop.
- A struct that straddles a line boundary costs *two* line fills to read. **Align hot structs to 64
  bytes** and keep them ≤ one line where possible.
- Two threads writing different variables that share a line ping-pong that line between cores —
  **false sharing** (covered in Module 05's coherency section).

Rule of thumb for interviews: *"How many `Order`s fit in a cache line?"* — if `sizeof(Order)==16`,
four per line; walking an array of them, you take one miss every four orders.

---

## 5. How a cache is organized — the address split

A cache must answer: "is address X here, and if so, where?" It does this by slicing the address into
three fields: **tag | index | offset**.

```
  63                      high bits              low bits            0
  ┌─────────────────────────┬───────────────┬──────────────────────┐
  │           TAG           │     INDEX     │    BLOCK OFFSET       │
  └─────────────────────────┴───────────────┴──────────────────────┘
        identifies which        picks the set     byte within the
        line (checked against   (row) to look in   64B line
        the stored tag)
```

- **Offset** = which byte inside the line. For a 64B line: `log2(64) = 6 bits`.
- **Index** = which *set* (row of the cache) to look in. `log2(number of sets)` bits.
- **Tag** = the remaining high bits, stored alongside the line and compared to confirm a hit.

Three organizations differ in how many lines share an index:

- **Direct-mapped (1-way):** every address maps to exactly one line. Simple and fast, but two hot
  addresses that map to the same set evict each other endlessly (**conflict misses**).
- **Fully-associative:** a line can go anywhere; you check all tags in parallel. No conflict misses,
  but comparing every tag is expensive — only used for tiny caches (like the TLB).
- **Set-associative (N-way):** the middle ground and what real caches use. The cache is divided into
  sets; each set holds **N** lines ("ways"). An address's index picks the set, then the tag is
  checked against all N ways in parallel. 8-way L1 is typical.

```
8-way set-associative lookup:
  address ─▶ [ tag | index | offset ]
                       │
                       ▼ index selects one set (row)
   set:  ┌── way0 ──┬── way1 ──┬ ... ┬── way7 ──┐
         │ tag|data │ tag|data │     │ tag|data │
         └────┬─────┴────┬─────┴─────┴────┬─────┘
              └──────────┴── compare all 8 tags to address.tag ──┘
                     hit if any matches → return data at offset
```

### Worked example (do this in the interview)

Given: 48-bit virtual addresses, 64 B lines, a **32 KB, 8-way** L1.

1. Offset bits: line = 64 B → `log2(64) = 6` bits.
2. Number of lines total: `32 KB / 64 B = 512` lines.
3. Lines per set = ways = 8, so **number of sets** = `512 / 8 = 64`.
4. Index bits: `log2(64) = 6` bits.
5. Tag bits: `48 − 6 (index) − 6 (offset) = 36` bits.

So the split is **[36-bit tag | 6-bit index | 6-bit offset]**. Being able to compute this on the
spot is a common screening question. (Note in Module 05: index+offset = 12 bits = exactly the
4 KB page offset — that's not a coincidence; it's what makes VIPT L1 caches work.)

---

## 6. Replacement policies

When a set is full and a new line arrives, one existing line must be evicted. The policy decides
which:

- **LRU (least-recently-used):** evict the line unused for the longest. Best hit rate but expensive
  to track exactly for 8+ ways (needs ordering bits per set).
- **Pseudo-LRU (tree-PLRU):** approximate LRU with a few bits per set — near-LRU quality, cheap
  hardware. What real caches actually use.
- **Random / round-robin:** even cheaper, slightly worse hit rate; sometimes used in lower levels.

The interview point: exact LRU is too costly in hardware for wide associativity, so caches
approximate it — and that approximation is invisible to correctness, only to performance.

---

## 7. Write policies

When the CPU *writes*, two orthogonal decisions matter:

- **Write-back vs write-through:**
  - *Write-through*: every write goes to the cache **and** the next level immediately. Simple,
    keeps memory current, but floods the bus with writes.
  - *Write-back* (what modern caches use): write only to the cache, mark the line **dirty**, and
    push it down only when it's evicted. Far less memory traffic; the "dirty bit" tracks which lines
    must be written back. This is why a line's state matters for coherency (Module 05).
- **Write-allocate vs no-write-allocate:** on a write *miss*, do you first fetch the line into cache
  (write-allocate, pairs with write-back) or write straight past it to memory (no-write-allocate,
  pairs with write-through)?

### Store buffers

Writes don't even wait for the cache. A core posts stores into a **store buffer** and moves on; the
store drains to L1 later. This hides write latency but means a core can see its *own* store before
other cores do — which is exactly the source of memory-reordering subtleties and why you need
`std::atomic` / fences (C++ guide Module 16). Interviewers love connecting "store buffer" to "why
does `x=1; y=1` become visible out of order to another thread."

---

## 8. L1 vs L2 vs L3 — how they actually differ

They're all SRAM caches of 64 B lines, but tuned for different jobs:

| Property | L1 | L2 | L3 (LLC) |
|----------|----|----|----------|
| Size | ~32 KB (×2: I and D) | 256 KB–1 MB | 8–36 MB |
| Latency | ~4 cyc | ~12 cyc | ~40 cyc |
| Associativity | 8-way | 8–16-way | 12–20-way |
| Scope | per-core, **split** into L1i (instructions) + L1d (data) | per-core, **unified** | **shared** across all cores |
| Design goal | *latency* — must be tiny to stay ~4 cyc | balance | *capacity* + inter-core sharing point |

Key interview points on "L1 vs L2":

- **L1 is split** (separate instruction and data caches) so the fetch unit and the load/store unit
  can access it simultaneously without contention. L2/L3 are unified.
- **L1 is small on purpose.** Latency scales with size (bigger array = longer wires + more decode);
  keeping L1 at 32 KB is what lets it answer in ~4 cycles. You can't just "make L1 bigger."
- **L3 is shared**, which makes it the coherency and communication point between cores, and the
  place where one noisy neighbor process can evict your hot data (the basis of cache-partitioning
  and isolation in HFT tuning).
- **Inclusive vs exclusive:** an *inclusive* L3 duplicates everything in L1/L2 (simplifies coherency
  — snoops check only L3), an *exclusive* hierarchy doesn't (more effective capacity). Intel
  historically inclusive-L3; modern designs vary.

---

## 9. The three C's — classifying misses

Every cache miss is one of three types (a standard interview taxonomy):

- **Compulsory (cold) miss:** the first-ever access to a line — it *had* to miss; nothing was
  cached yet. Fixed by prefetching, not by a bigger cache.
- **Capacity miss:** the working set is bigger than the cache, so lines you'll reuse get evicted
  before you return. Fixed by a bigger cache or a smaller working set (blocking/tiling).
- **Conflict miss:** the line was evicted not because the cache was full, but because too many hot
  addresses mapped to the *same set* (a limited-associativity artifact). Fixed by more associativity
  or by changing the access stride/layout. A fully-associative cache has *zero* conflict misses by
  definition.

Mnemonic: **compulsory** = first touch, **capacity** = too much data, **conflict** = bad mapping.

---

## Common pitfalls / misconceptions

- **"Bigger cache is always better."** No — bigger = slower. L1 is deliberately tiny to hit 4
  cycles. The hierarchy exists precisely because one big fast cache is impossible.
- **"A cache miss reads one byte."** It reads a whole 64 B line. Layout for spatial locality.
- **"Random access to a small array is fine."** If your stride jumps by the set-count × line-size,
  you can create pathological conflict misses even in a small array (see quiz).
- **Confusing associativity with size.** An 8-way 32 KB cache and a direct-mapped 32 KB cache hold
  the same data volume; associativity changes *conflict* behavior, not capacity.
- **Ignoring L1i.** Bloated code (huge inlined templates, fat exception tables) causes *instruction*
  cache misses that are just as deadly as data misses on the hot path.

---

## Quiz — tough problems

**Q1.** A 32 KB, 4-way set-associative L1 with 64 B lines, 64-bit addresses. Give the tag/index/
offset bit split.
**Answer:** Offset = log2(64) = **6**. Lines = 32 KB / 64 = 512; sets = 512 / 4 = 128; index =
log2(128) = **7**. Tag = 64 − 7 − 6 = **51**. Split: **[51 tag | 7 index | 6 offset]**.

**Q2.** `sizeof(Order) == 24`. You iterate a `std::vector<Order>` of 1,000,000 elements once,
sequentially. Roughly how many L1 misses (assume nothing pre-cached, 64 B lines, perfect prefetch
off)?
**Answer:** Total bytes = 24 MB. Lines = 24 MB / 64 B = **375,000 misses** (one compulsory miss per
line; the elements within a line come free). Note 24 B doesn't divide 64, so orders straddle lines —
but summed over the array it's still total_bytes/64. Padding to 32 B would make it worse (32 MB), to
16 B better (16 MB).

**Q3.** Two arrays `a` and `b`, each 4 KB and 4 KB-aligned, in a direct-mapped 32 KB cache with 64 B
lines. You loop `a[i] += b[i]`. Why might this thrash?
**Answer:** In a direct-mapped cache, `index = (addr / 64) mod 512`. If `a` and `b` are separated by
a multiple of the cache size (32 KB), corresponding elements `a[i]` and `b[i]` map to the *same set*
and evict each other every iteration → a conflict miss on every access despite tiny data. Fix:
offset one array, or use a set-associative cache. This is classic **cache-line aliasing / conflict
thrashing**.

**Q4.** Why is L1 not just made 1 MB to reduce misses?
**Answer:** Latency scales with size (longer bitlines/wordlines, bigger decoders, more tag
comparisons). A 1 MB L1 couldn't answer in ~4 cycles; it'd be ~12+ (that's literally what L2 *is*).
The hierarchy splits the difference: small fast L1, bigger slower L2/L3.

**Q5.** A struct is 40 bytes and 8-byte aligned. Reading the whole struct — best case and worst case
cache-line fills?
**Answer:** Best case (aligned so it fits in one 64 B line) = **1** line fill. Worst case (straddles
a line boundary) = **2** line fills. Aligning the struct to 64 B guarantees the best case for
anything ≤ 64 B.

**Q6.** You measure that reversing the loop order of a nested matrix loop (`[i][j]` vs `[j][i]`) is
10× faster. Which miss type did the fast version reduce, and why?
**Answer:** **Capacity/spatial-locality-driven misses.** Row-major arrays store `[i][j]` contiguous
in memory. Iterating `j` innermost walks a line sequentially (one miss per 64 B); iterating `i`
innermost jumps a full row per step, touching a new line every access → misses explode. Same data,
different access pattern.

**Q7.** True/false: increasing associativity from 4-way to 8-way can only reduce miss rate.
**Answer:** *Mostly* true for miss *rate* (fewer conflict misses), but **false** for latency/energy
— more ways = more parallel tag comparisons = higher access latency and power, and diminishing
returns past ~8-way. It's a trade-off, which is why L1 stops around 8-way.

**Q8.** What's the difference between a compulsory miss and a capacity miss, and which does
prefetching help?
**Answer:** Compulsory = first-ever access to a line (unavoidable by caching alone). Capacity =
reused data evicted because the working set exceeds the cache. **Prefetching** hides *compulsory*
(and some capacity) misses by fetching lines before they're demanded; it does nothing for a working
set that's fundamentally too big — that needs blocking/tiling or a smaller footprint.

**Q9.** A write-back cache holds a dirty line. The core is about to be powered off cleanly. What must
happen, and what bit tracks it?
**Answer:** The **dirty bit** marks lines modified in cache but not yet in memory. Before eviction
(or shutdown/`clflush`), dirty lines must be **written back** to the next level/memory or the write
is lost. Write-through caches never carry this hazard (memory is always current) at the cost of far
more write traffic.

**Q10.** In a 3 GHz core, a hot loop does 1 dependent DRAM load per iteration and nothing else. What
throughput can you expect, and what's the fix?
**Answer:** ~200 cycles/load ≈ 65 ns/iteration ≈ **~15 million iterations/sec** — abysmal, because
each load stalls the whole pipeline. Fix: restructure for locality (so loads hit L1/L2), prefetch
ahead, or overlap independent loads (memory-level parallelism) so the ~200 ns latencies pipeline
instead of serializing.

---

## Indian HFT interview questions

**Q: What's the difference between L1 and L2 cache?** (Extremely common — Optiver, Quadeye,
AlphaGrep.)
**A:** L1 is per-core, tiny (~32 KB), ~4-cycle latency, and *split* into separate instruction (L1i)
and data (L1d) caches so fetch and load/store don't contend. L2 is per-core, larger (256 KB–1 MB),
~12-cycle, and unified. The size difference is deliberate: L1 must stay small to hit 4 cycles;
latency grows with size. L3 below them is large (tens of MB) and *shared* across cores, acting as
the coherency/communication point.

**Q: How big is a cache line and why does it matter for performance?** (Tower Research, Graviton.)
**A:** 64 bytes on x86-64/ARM. It's the unit of transfer, so (1) sequential access amortizes one
miss over 64 bytes → arrays beat linked lists; (2) structs should be laid out and aligned so hot
fields share a line; (3) two threads writing different variables in the same line cause false
sharing — pad hot per-thread data to 64 B (`alignas(64)`).

**Q: Given a 32 KB 8-way L1 with 64 B lines and 48-bit addresses, compute the address split.**
(Screening question, Quadeye/HRT.)
**A:** Offset 6 bits (64 B); 512 lines / 8 ways = 64 sets → index 6 bits; tag = 48 − 6 − 6 = 36 bits.
And index+offset = 12 bits = the 4 KB page offset — which is what enables VIPT indexing.

**Q: Your code got 5× slower after adding a field to a struct. Why might that happen?** (Jump, IMC.)
**A:** The struct grew past a cache-line boundary (or its alignment changed), so what used to be one
line fill is now two, or the array's footprint no longer fits in L1/L2 → capacity misses. Also
possible: you introduced false sharing between threads. Fix by shrinking/reordering fields, splitting
hot/cold data, or `alignas`.

**Q: What are the three types of cache misses?** (Standard — Da Vinci, AlphaGrep.)
**A:** Compulsory (first touch — fix with prefetch), capacity (working set > cache — fix with
smaller footprint / blocking), conflict (too many addresses map to one set in a
limited-associativity cache — fix with more associativity or better layout).

**Q: Why can't we just build one huge fast memory and skip the hierarchy?** (Conceptual — Optiver.)
**A:** Physics and economics. SRAM is fast but 6 transistors/bit → huge and power-hungry, so only
KB–MB affordable; DRAM is 1T1C → cheap and dense but slow and needs refresh. Latency also grows with
size. So you stack a little fast SRAM over a lot of slow DRAM and let locality make the common case
fast.

**Q: Write-back vs write-through — which do modern caches use and why?** (Graviton, Millennium.)
**A:** Write-back. It writes only to cache, marks the line dirty, and defers the memory write until
eviction — drastically less memory traffic than write-through (which writes to memory on every
store). Write-through is simpler and keeps memory always current but doesn't scale for
write-heavy workloads; it survives mainly in specific coherency/IO contexts.

**Q: What is a store buffer and how does it relate to memory ordering?** (Advanced — HRT, Jump.)
**A:** A per-core queue where stores are posted so the core doesn't stall waiting for the cache
write. Stores drain to L1 asynchronously, so a core sees its own writes before other cores do, and
writes can appear reordered to other cores. That's why cross-thread communication needs `std::atomic`
with the right memory order (release/acquire) to force the necessary drains/fences.

---

## Key takeaways

- The **memory wall** — DRAM is ~50× slower than L1 — is why caches exist; real speed is decided by
  hit rates, not instruction counts.
- The hierarchy trades size for speed at each level because you *cannot* have big + fast + cheap
  memory (SRAM 6T vs DRAM 1T1C).
- Data moves in **64-byte lines**; exploit spatial + temporal **locality**, and layout/align hot
  data to lines.
- An address is **[tag | index | offset]**; set-associative caches use index → set, then compare N
  tags in parallel. You should be able to compute the split on demand.
- **L1** is tiny, per-core, split, ~4 cyc (latency-optimized); **L2** per-core unified; **L3** big,
  shared, the coherency point.
- Misses are **compulsory / capacity / conflict** — know which each optimization targets.
- Write-back + store buffers hide write latency but create the reordering that concurrency
  primitives must tame.

**Next:** [05 — Cache addressing & coherency](05-cache-addressing-coherency.md) — how virtual/physical
addressing (VIVT/VIPT/PIPT) interacts with caches, whether a context switch flushes them, and how
MESI keeps multiple cores consistent.
