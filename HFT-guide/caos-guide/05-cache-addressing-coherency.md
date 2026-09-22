# Module 05 — Cache addressing & coherency

Module 04 assumed the cache is indexed by "the address." But which address — the *virtual* one the
program uses, or the *physical* one after the MMU translates it (Module 06)? That choice
(VIVT/VIPT/PIPT) decides whether a context switch must flush the cache, and it's a favorite HFT
interview thread. The second half of this module answers the other classic: when several cores each
cache the same line, how do they stay consistent? That's **cache coherency** (MESI), and it's the
mechanism behind false sharing.

---

## 1. Two addresses, one cache — the core problem

Every load the CPU issues uses a **virtual address (VA)**. The MMU translates it to a **physical
address (PA)** via the page tables + TLB (Module 06). Translation costs time (a TLB lookup, or worse,
a page-table walk). So there's a tension for the L1 cache, which sits on the critical path:

- If the cache is indexed/tagged by **virtual** address, it can start *immediately* — no waiting for
  translation. Fast, but virtual addresses are **ambiguous across processes** (the same VA means
  different memory in different processes; different VAs can alias the same physical memory).
- If it's indexed/tagged by **physical** address, it's unambiguous and correct, but you must
  **finish translation before you can even look up the cache** — adding TLB latency to every access.

The three schemes are the three ways to resolve this tension.

---

## 2. VIVT, VIPT, PIPT — the three schemes

Naming: the first V/P is how the cache is **Indexed** (which set to look in); the second V/P is how
it's **Tagged** (how a hit is confirmed). "VIPT" = **V**irtually **I**ndexed, **P**hysically
**T**agged.

```
Recall the address split:   [ tag | index | offset ]
                                     ▲       ▲
                             which set       byte in line

Timeline of a load (VA in → data out):

VIVT:  VA ─▶ index+tag from VA ─▶ cache hit? ─▶ data      (translation not needed for hit)
             └── fastest, but VA is ambiguous ──┘

PIPT:  VA ─▶ [ TLB translate to PA ] ─▶ index+tag from PA ─▶ cache hit? ─▶ data
             └── translation is SERIAL before lookup: slower ──┘

VIPT:  VA ─▶ index from VA ─────────────┐
       VA ─▶ [ TLB translate to PA ] ──▶ tag from PA
             └── index and TLB run IN PARALLEL, then compare physical tag ──┘
             (best of both: no waiting to start, but correct physical tag)
```

### VIVT — virtually indexed, virtually tagged

Both index and tag come from the VA, so **no translation is on the hit path** — fastest possible.
But virtual addresses are ambiguous, giving two bugs:

- **Homonyms:** the *same* VA in two processes maps to *different* physical memory. After a context
  switch, a VIVT cache could hand process B the data process A left at that VA. → you must **flush
  the whole cache on every context switch** (or tag entries with an address-space ID).
- **Synonyms / aliasing:** two *different* VAs (e.g. a shared library mapped twice, or shared
  memory) map to the *same* PA. Now the same physical byte lives in two cache lines; write one and
  the other is stale.

VIVT is therefore rare for data caches — the flush-on-switch cost is brutal. (It survives in some
niche/embedded designs and historically some I-caches.)

### PIPT — physically indexed, physically tagged

Both index and tag come from the PA. Physical addresses are **globally unambiguous** — one PA is one
piece of memory for everyone — so there are **no homonym/synonym problems and no flush on context
switch**. The cost: you must complete TLB translation *before* indexing, adding latency. This is
fine for **L2/L3**, which are already many cycles away from the core and off the tightest path — so
lower-level caches are essentially always PIPT.

### VIPT — the sweet spot for L1

Index using the **virtual** address (available instantly, no wait), but tag with the **physical**
address (looked up in the TLB *in parallel* with the cache-set read). When both finish, compare the
physical tag against the ways. You get VIVT's speed (index starts immediately) with PIPT's
correctness (physical tag, so no ambiguity).

The catch — the constraint that makes this legal:

> The **index + offset bits must fit entirely within the page offset**, so that they are identical
> in the virtual and physical address (the page offset is *not* translated — only the page number
> is). Then "virtually indexed" and "physically indexed" pick the *same* set, and the aliasing
> problem disappears.

Concretely, for 4 KB pages the page offset is **12 bits**. Recall the Module 04 example: a 32 KB
8-way L1 with 64 B lines has offset 6 + index 6 = **12 bits** — exactly the page offset. Not a
coincidence — L1 caches are *sized* (via associativity) so index+offset ≤ page-offset bits. This is
why L1 is "32 KB, 8-way" and not "32 KB, direct-mapped": bumping associativity keeps the set count
low enough that the index stays inside the page offset. When designers want a bigger VIPT L1 they add
ways, not sets. (If index bits spilled past the page offset you'd get the **page-coloring / cache
aliasing** problem and would need OS help to color pages.)

---

## 3. "Do we need to flush the cache when switching between processes?"

A signature HFT interview question. The precise answer:

- **Data/unified caches (L1d, L2, L3) are PIPT (or VIPT with physical tags).** They're keyed by
  *physical* address, which is unambiguous across processes. **So NO — you do not flush the data
  caches on a context switch.** Process B simply won't find process A's physical lines relevant, and
  if they happen to share physical memory (shared lib, shared mem), that sharing is *correct*. Not
  flushing is also what makes leftover hot lines available if you switch back quickly.
- **What you DO flush (or avoid flushing) is the TLB**, because the TLB *is* virtually keyed
  (VA→PA). Two processes have different page tables, so process A's translations are wrong for B.
  Classic hardware flushed the whole TLB on `CR3` reload (the page-table-base switch). Modern CPUs
  avoid even that with **ASIDs / PCIDs** (address-space identifiers) that tag each TLB entry with an
  owning address space, so entries from different processes coexist and no flush is needed.
- **Only a virtually-*tagged* cache (VIVT) would force a data-cache flush on switch** — which is
  precisely why real designs avoid virtual tags for data caches.

So the crisp interview answer: *"No — modern data caches are physically tagged, so they need no flush
across processes; physical addresses disambiguate. What's address-space-specific is the TLB, and even
that is handled by ASIDs/PCIDs rather than a full flush. A full cache flush would only be needed for
a virtually-tagged cache, which is why we don't build data caches that way."*

(You *do* explicitly flush lines in special cases — `clflush`/`wbinvd` for DMA coherency with a
device, or persistent-memory ordering — but that's not process switching.)

---

## 4. Cache coherency — the multi-core problem

Now the other axis. Each core has its own private L1/L2. If two cores both cache the same physical
line and one writes it, the other must not keep reading the stale copy. **Cache coherency** is the
hardware protocol that maintains the illusion of a single shared memory. The dominant family is
**MESI** (and extensions MOESI/MESIF), named after the four states a line can be in *within one
core's cache*:

- **M — Modified:** this core has the only copy, and it's *dirty* (differs from memory). Must write
  back before anyone else reads it.
- **E — Exclusive:** this core has the only copy, and it's *clean* (matches memory). Can be written
  silently (→ M) without telling anyone.
- **S — Shared:** multiple cores may hold this line, all clean/read-only. Reads are free; a write
  requires invalidating the others first.
- **I — Invalid:** this line is stale/absent; a read/write here misses.

The rule: **at most one core in M or E; any number in S; a write requires exclusive ownership.**

### A two-core walkthrough

```
Line L, initially in memory only. Core0, Core1 each have L = I.

1. Core0 reads L      → miss, fetch from mem, no other copies → Core0: E
2. Core1 reads L      → miss; Core0 sees the read, downgrades E→S; both now: S
3. Core0 writes L     → needs ownership. Issues Read-For-Ownership /
                        invalidate. Core1's copy → I. Core0: M (dirty).
4. Core1 reads L      → miss (its copy is I). Core0 must supply the modified
                        data (cache-to-cache / write-back), then downgrade M→S.
                        Both: S again.
```

The expensive step is 3→4: a write that others hold **Shared** triggers an **invalidate broadcast**
(a **Read-For-Ownership**), and the next reader forces a write-back/transfer. These coherency
messages travel over the interconnect and cost tens to hundreds of cycles — *even though the program
did nothing "wrong."*

### False sharing — the killer consequence

Coherency operates at **cache-line granularity**, not variable granularity. If two threads on two
cores write *different variables that happen to share one 64 B line*, every write ping-pongs the line
between the two cores' caches (M on one → invalidate the other → M on the other → …):

```
        line (64B): [ counterA (core0) | counterB (core1) | ... ]
   core0: write A → line becomes M in core0, I in core1
   core1: write B → invalidate core0, line M in core1, I in core0
   core0: write A → invalidate core1 again ... ping-pong forever
```

The variables are logically independent, but the hardware sees one line and serializes them. This is
**false sharing**, and it can make "parallel" code slower than single-threaded. Fix: pad/align each
hot per-thread datum to its own line (`alignas(64)`), exactly as the SPSC-queue head/tail are aligned
in the C++ guide.

---

## Common pitfalls / misconceptions

- **"Context switches flush the caches."** They don't flush *data caches* (physically tagged). They
  affect the *TLB*, and even that is handled by ASIDs/PCIDs, not a blanket flush.
- **"VIPT is virtual, so it has aliasing problems."** No — VIPT tags *physically*, and the index bits
  are constrained to the (untranslated) page offset, so it's alias-free like PIPT while being fast
  like VIVT.
- **"Coherency keeps my variables consistent, so I don't need atomics."** Coherency keeps *lines*
  consistent (single-value visibility), but it does **not** give you atomic read-modify-write or
  ordering across multiple variables — that's what `std::atomic`/fences are for. Coherency ≠
  synchronization.
- **"False sharing is a correctness bug."** It's purely a *performance* bug — the results are
  correct, just slow.
- **Confusing the TLB with the cache.** The TLB caches *address translations* (VA→PA); the data
  cache caches *data*. Different structures, different flush rules.

---

## Quiz — tough problems

**Q1.** Why is L1 typically VIPT while L2/L3 are PIPT?
**Answer:** L1 is on the tightest latency path, so it can't afford to wait for TLB translation before
indexing — VIPT lets it index (from the VA) in *parallel* with the TLB lookup, then tag-check with
the PA. L2/L3 are already many cycles out and are accessed after L1 (and after translation), so PIPT
is fine and simplest — no aliasing worries at all.

**Q2.** For a VIPT L1 with 4 KB pages, 64 B lines, what's the maximum number of sets before you hit
the aliasing constraint, and hence the max size at a given associativity?
**Answer:** index+offset must fit in the 12-bit page offset. Offset = 6 bits → index ≤ 6 bits → ≤ 64
sets. Max size = sets × ways × line = 64 × ways × 64 B = **4 KB × ways**. So 8-way → 32 KB, 16-way →
64 KB. To grow a VIPT L1 you must add *ways*, not sets — which is exactly why L1 caches are highly
associative.

**Q3.** A VIVT cache doesn't flush on context switch and has no ASID tagging. What goes wrong?
**Answer:** **Homonym bug:** the same virtual address in the new process indexes/tags to the old
process's cached line, so the new process reads stale/foreign data → correctness violation and a
security leak. That's why VIVT caches must flush (or tag with ASIDs) on every switch.

**Q4.** Two threads increment two separate `int` counters 100M times each, pinned to different cores.
The counters are adjacent fields in a struct. Why is it slow, and what's the one-line fix?
**Answer:** **False sharing** — both `int`s live in one 64 B line, so each write invalidates the
other core's copy, ping-ponging the line over the interconnect (RFO every increment). Fix: put each
counter on its own line, e.g. `alignas(64) int a; alignas(64) int b;` (or pad the struct).

**Q5.** In MESI, a core holds a line in **E**. It writes to it. What coherency traffic is generated?
**Answer:** **None.** Exclusive means it's the only clean copy, so the write silently transitions
E→M with no bus messages. That's the whole point of the E state — it lets a private read followed by
a write avoid an invalidate broadcast (which the S state would require).

**Q6.** Line is **S** in two cores. Core0 writes. Walk the states.
**Answer:** Core0 must gain ownership: it issues a Read-For-Ownership / invalidate. Core1's copy goes
**S→I**. Core0 goes **S→M** (dirty). Any later read by Core1 misses (I), forcing Core0 to supply the
modified data and both settle to **S** (or Core0→I if it's a write by Core1). The write's cost is the
invalidate round-trip.

**Q7.** Does cache coherency (MESI) make `x++` thread-safe across cores?
**Answer:** No. MESI guarantees a consistent *value* of the line, but `x++` is load-modify-store —
three operations. Two cores can both load the old value before either stores. You need an *atomic*
RMW (`lock xadd` / `std::atomic::fetch_add`), which uses coherency (RFO to get the line in M) plus
locking of the operation. Coherency ≠ atomicity.

**Q8.** You add `clflush` after writing a buffer that a DMA device will read. Why — isn't coherency
automatic?
**Answer:** Core-to-core is coherent via MESI, but a **DMA device** may not participate in cache
coherency (or you want to force ordering to memory). `clflush`/`clwb` writes the dirty line back so
the device reads current data. This is a *device*-coherency flush, unrelated to process switching.

**Q9.** Same physical page mapped at two different virtual addresses in one process (a synonym). On a
PIPT cache, is there an aliasing problem?
**Answer:** No — PIPT indexes and tags by *physical* address, so both virtual aliases resolve to the
same set and same physical tag → one line, always consistent. The synonym problem only bites
virtually-*indexed* caches whose index bits exceed the page offset (uncontrolled VIPT/VIVT).

**Q10.** Why does adding cores sometimes *slow down* a shared-counter microbenchmark?
**Answer:** The single shared line bounces between all cores' caches (each increment = RFO +
invalidate of the others). More cores = more contention on that one line = more coherency traffic,
which serializes and adds latency. This is *true* sharing contention (the false-sharing mechanism,
but on genuinely shared data) — fixed by per-core counters summed at the end, or sharded atomics.

---

## Indian HFT interview questions

**Q: Do we need to flush the cache when switching between processes?** (Very common — Tower Research,
Graviton, HRT.)
**A:** No, not the data caches. Modern L1/L2/L3 are physically tagged (L1 is VIPT with a physical
tag, L2/L3 PIPT), and physical addresses are unambiguous across processes, so cached lines stay valid
— and useful if you switch back quickly. What's address-space-specific is the **TLB** (it maps
VA→PA), and even that isn't fully flushed anymore: **ASIDs/PCIDs** tag TLB entries per address space.
A full cache flush would only be needed for a *virtually-tagged* cache — which is exactly why we
don't design data caches that way.

**Q: Explain PIPT and VIPT caches.** (Explicitly on the user's list — Quadeye, Optiver, Jump.)
**A:** PIPT = physically indexed, physically tagged: correct and unambiguous, but you must translate
(TLB) before you can index, so translation is serial on the access path — fine for L2/L3. VIPT =
virtually indexed, physically tagged: you index from the virtual address *immediately* (in parallel
with the TLB lookup) but confirm the hit with the physical tag — as fast to start as virtual, as
correct as physical. VIPT is the L1 sweet spot, valid only because the index+offset bits fit inside
the untranslated page offset (which is why L1 is small and highly associative).

**Q: What's the difference between the TLB and the cache?** (Optiver, AlphaGrep.)
**A:** The TLB caches *address translations* (virtual page → physical frame) to avoid a page-table
walk; the data cache caches *data* (64 B lines). The TLB is virtually keyed and address-space-
specific (hence ASIDs); data caches are physically tagged and process-agnostic. A load can hit the
TLB but miss the cache, or vice versa (miss TLB → walk → then hit cache).

**Q: What is false sharing and how do you fix it?** (Ubiquitous — everyone asks this.)
**A:** Coherency works at 64 B line granularity, so two threads writing different variables in the
same line invalidate each other's cached line on every write, ping-ponging it over the interconnect —
huge slowdown with no logical sharing. Fix: align/pad each hot per-thread variable to its own cache
line (`alignas(64)`), e.g. SPSC-queue head and tail on separate lines.

**Q: Walk me through MESI when two cores share a line and one writes.** (HRT, Millennium.)
**A:** Both start in Shared. The writer needs ownership, so it issues a Read-For-Ownership /
invalidate; the other core's copy goes to Invalid; the writer's line goes to Modified (dirty). When
the other core reads again, it misses, the writer supplies the modified data (cache-to-cache) and
both settle to Shared. The invalidate round-trip is the cost of a write to shared data.

**Q: Does cache coherency remove the need for locks/atomics?** (Conceptual trap — Jump, Da Vinci.)
**A:** No. Coherency guarantees a consistent *single value* per line and read-your-writes visibility,
but not atomic read-modify-write, not ordering across multiple locations, and not mutual exclusion.
You still need `std::atomic`/`lock`-prefixed RMW and the right memory ordering. Coherency is a
building block that atomics use, not a replacement for them.

**Q: Why is L1 32 KB and 8-way rather than, say, 128 KB direct-mapped?** (Graviton, Quadeye.)
**A:** Two reasons converge. (1) Latency: L1 must answer in ~4 cycles, which caps its size. (2) VIPT:
to index virtually without aliasing, index+offset must stay within the 12-bit page offset → ≤64 sets;
the only way to add capacity while keeping the index small is to add *ways*. So 8-way, 32 KB. Direct-
mapped would also suffer heavy conflict misses.

---

## Key takeaways

- Caches must choose whether to index/tag by **virtual** or **physical** address; that choice (VIVT
  / VIPT / PIPT) trades speed against ambiguity.
- **VIPT** is the L1 sweet spot: index from the VA in parallel with TLB translation, tag with the PA
  — fast *and* alias-free, provided index+offset ≤ page-offset bits (why L1 is small + associative).
- **PIPT** (L2/L3) is unambiguous across processes → **data caches are NOT flushed on a context
  switch**; only virtually-*tagged* caches would need that.
- What's address-space-specific is the **TLB**, handled by **ASIDs/PCIDs** instead of full flushes.
- **MESI** keeps per-core caches consistent (M/E/S/I); a write to a Shared line costs an invalidate
  broadcast (RFO).
- Coherency is **line-granular**, which creates **false sharing** — a performance bug fixed by
  padding hot data to separate lines.
- Coherency ≠ synchronization: you still need atomics/fences for RMW and ordering.

**Next:** [06 — Virtual memory](06-virtual-memory.md) — the MMU, TLB, and page tables that produce the
physical addresses this module assumed, the full read/write flow from a virtual address to DRAM, and
how copy-on-write is implemented.
