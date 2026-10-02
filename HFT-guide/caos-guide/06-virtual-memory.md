# Module 06 — Virtual memory

Every address your program uses is a lie. When you print a pointer as `0x7ffe...`, that's a
**virtual address** — a per-process fiction the hardware translates to a real **physical address** in
DRAM on every single access. This module explains why that indirection exists, the machinery that
performs it (MMU, TLB, page tables), the **complete step-by-step flow of a single load from virtual
address to data**, and how it powers page faults, demand paging, and **copy-on-write**. This is the
module that ties Modules 04–05 (caches) to Part II (the OS that manages all this).

---

## 1. Why virtual memory exists

Give every process the illusion of its own private, contiguous address space (0 up to 2⁴⁸-ish),
regardless of how much physical RAM exists or what else is running. Three problems it solves at once:

- **Isolation / protection:** process A literally *cannot name* process B's memory — their virtual
  addresses map to different physical frames (or are simply unmapped). One process can't corrupt or
  spy on another. The kernel enforces this in the page tables the hardware consults.
- **Relocation / simplicity:** the compiler/linker can assume a fixed layout (code at low addresses,
  stack high, heap growing up) without knowing where in physical RAM the program will actually land.
  The same binary runs anywhere; the OS maps its virtual pages to whatever physical frames are free.
- **Overcommit / more-than-physical:** the sum of all processes' virtual memory can exceed physical
  RAM. Pages that aren't currently needed live on disk (swap) and are brought in on demand. You can
  also memory-map files larger than RAM.

The cost of all this is **translation on every access** — which is why the hardware (MMU) and a
translation cache (TLB) exist to make it nearly free in the common case.

---

## 2. Pages, frames, and the address split

Memory is managed in fixed-size chunks called **pages** (virtual side) and **frames** (physical
side), classically **4 KB**. A virtual address splits into a **virtual page number (VPN)** and a
**page offset**; translation maps the VPN → a **physical frame number (PFN)**, and the offset is
copied through unchanged (the page is the unit of mapping, so the position *within* a page is
identical in VA and PA):

```
Virtual address (e.g. 48-bit):
  ┌───────────────────────────────┬───────────────────┐
  │      virtual page number      │    page offset     │   (offset = 12 bits for 4 KB)
  └───────────────┬───────────────┴──────────┬─────────┘
                  │ translate (page tables)   │ copied through unchanged
                  ▼                           ▼
  ┌───────────────────────────────┬───────────────────┐
  │    physical frame number      │    page offset     │
  └───────────────────────────────┴───────────────────┘
Physical address
```

Because the offset passes through untranslated, the low 12 bits of a VA and its PA are identical —
**exactly the fact that makes VIPT L1 caches work** (Module 05). Larger pages exist too — **huge
pages** (2 MB, 1 GB) — which reduce the number of translations and TLB pressure; HFT setups often
enable them.

---

## 3. Page tables — multi-level, and why

A page table maps every VPN to a PFN (plus permission bits: present, writable, user/kernel,
no-execute, dirty, accessed). But a flat table would be enormous: a 48-bit space with 4 KB pages has
2³⁶ pages; at 8 bytes/entry that's 512 GB *per process* — absurd, and mostly empty.

The fix is a **multi-level (hierarchical, radix) page table**: the VPN is split into several indices,
each selecting an entry in a table that points to the *next* table, and only the tables actually
needed are allocated. x86-64 uses **4 levels** (PML4 → PDPT → PD → PT); the CPU register **CR3**
points at the top-level table of the current process.

```
Virtual address (4 KB pages, 48-bit):
 [ PML4 idx |  PDPT idx |  PD idx  |  PT idx  |  page offset (12) ]
   9 bits      9 bits     9 bits     9 bits       12 bits

CR3 ─▶ PML4 ──idx──▶ PDPT ──idx──▶ PD ──idx──▶ PT ──idx──▶ PTE ─▶ physical frame
       (table)       (table)      (table)     (table)     (entry with PFN + flags)
```

Each level is one 4 KB table of 512 8-byte entries (9 index bits = 512). Sparse address spaces cost
almost nothing: a process using a little code + stack + heap needs only a handful of tables, not 512
GB. The price is that a translation from scratch is a **4-memory-access walk** (one load per level) —
which is why the TLB exists.

---

## 4. The MMU and the TLB

- The **MMU (Memory Management Unit)** is the hardware block that performs translation. On a TLB
  miss, its **hardware page-table walker** reads CR3 and walks the 4 levels automatically (on x86;
  some ISAs like older MIPS trap to software instead).
- The **TLB (Translation Lookaside Buffer)** is a small, fast, fully/highly-associative cache of
  recent **VPN → PFN** translations. It's the "cache for the page tables." A TLB *hit* gives the
  frame in ~1 cycle; a *miss* triggers the page-table walk (4 dependent memory accesses, ~tens to
  100+ cycles — though those table lines are often themselves cached).

The TLB is **virtually indexed by VPN and address-space-specific**, so it's what a context switch
must handle (flush, or use **ASIDs/PCIDs** to tag entries per process) — contrast this with the
physically-tagged data caches, which do *not* need flushing (Module 05). There are separate iTLB
(instruction) and dTLB (data), and multi-level TLBs (L1 TLB, L2 TLB) just like data caches.

---

## 5. The full read/write flow — virtual address to data

This is the centerpiece — the "explain the full flow of how we read/write data" question. Follow a
single `load` of a virtual address, from the instruction to the bytes arriving in a register:

```
   CPU executes:  mov rax, [VA]
        │
        ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │ 1. TLB lookup (VPN → PFN)                                          │
 │     hit ─▶ get PFN in ~1 cyc ─────────────────────────────┐       │
 │     miss ▼                                                 │       │
 │   2. Page-table walk (MMU walker reads CR3→PML4→PDPT→PD→PT)│       │
 │        ├─ PTE present & perms OK ─▶ get PFN, fill TLB ─────┤       │
 │        └─ PTE not present / perm fail ─▶ PAGE FAULT (trap  │       │
 │           to kernel; §6) ─▶ kernel fixes ─▶ retry          │       │
 └───────────────────────────────────────────────────────────┼──────┘
                                                               ▼
             physical address = PFN : page-offset   (offset copied through)
                                                               │
   For L1 VIPT this actually overlaps: the cache is INDEXED    │
   from the VA's offset+index bits WHILE the TLB translates,   │
   then the PHYSICAL tag from the PFN is compared. (Module 05) │
                                                               ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │ 3. Cache lookup with the physical address                         │
 │     L1 hit  (~4 cyc) ─▶ return the bytes ─────────────────┐       │
 │     L1 miss ▼                                              │       │
 │     L2 hit  (~12 cyc) ─▶ fill L1, return ─────────────────┤       │
 │     L2 miss ▼                                              │       │
 │     L3 hit  (~40 cyc) ─▶ fill L2/L1, return ──────────────┤       │
 │     L3 miss ▼                                              │       │
 │   4. DRAM access (~200 cyc): controller reads the 64B line,│      │
 │      fills L3/L2/L1, returns the bytes ───────────────────┤       │
 └───────────────────────────────────────────────────────────┼──────┘
                                                               ▼
                                            bytes land in rax; instruction retires
```

A **write** follows the same translation path, then (for a write-back cache, Module 04) updates the
L1 line, marks it **dirty**, and posts through the **store buffer**; the value drains to L1
asynchronously and is written back down the hierarchy only on eviction. Coherency (MESI, Module 05)
ensures other cores see a consistent value; a write to a page marked read-only triggers a protection
fault — which is the hook copy-on-write uses (§7).

The key takeaway: a *single* memory access can be anywhere from ~4 cycles (TLB + L1 hit) to ~hundreds
(TLB miss → walk → DRAM miss) to *milliseconds* (page fault to disk). HFT work is largely about
keeping every access at the top of that range.

---

## 6. Page faults, demand paging, swapping

A **page fault** is a CPU exception (trap to the kernel, Module 09) raised when a translation can't
complete: the PTE is marked not-present, or the access violates permissions. The kernel's page-fault
handler decides what happened:

- **Minor (soft) fault:** the page is in physical RAM but not mapped in this process's tables yet
  (e.g. it's in the page cache, or a shared page, or a COW page). The kernel just fixes the mapping —
  cheap (no disk).
- **Major (hard) fault:** the page is on disk (swap, or a not-yet-loaded file page). The kernel must
  read it from disk into a free frame, update the PTE, and retry — *very* expensive (µs–ms). Fatal
  for latency; HFT boxes disable swap and pre-fault/lock memory (`mlockall`) to avoid this entirely.
- **Invalid fault:** the address isn't mapped at all / bad permission → SIGSEGV.

**Demand paging** is the policy of *not* loading pages until first touched: `malloc`/`mmap` just
create mappings (or reserve address space); the physical frame is allocated lazily on the first
access via a minor fault. That's why a huge `mmap` returns instantly and why the *first* write to
fresh memory is slower than later ones (the "first-touch" effect, also central to NUMA placement).

---

## 7. Copy-on-write (COW) — how it's implemented

COW is the classic "explain how X is implemented" question, and `fork()` is its poster child. A naïve
`fork()` would copy the entire parent address space to the child — gigabytes, most of it never used
(the child usually `exec`s immediately). COW makes `fork()` nearly free by **sharing physical frames
and copying lazily, only on the first write.**

Mechanism, step by step:

1. On `fork()`, the kernel copies the parent's **page tables** (cheap — just the mapping structures),
   not the underlying pages. Both parent and child PTEs now point at the **same physical frames**.
2. Every shared, writable page is marked **read-only** in *both* processes' PTEs, and a **reference
   count** on each shared frame is bumped (now 2).
3. Reads by either process hit the shared frame directly — no copy, fully shared.
4. When either process **writes** a shared page, the read-only bit triggers a **protection page
   fault** (trap to kernel). The handler recognizes it as a COW page (the VMA is logically writable),
   so it:
   - allocates a **fresh physical frame**,
   - **copies** the shared page's contents into it,
   - remaps the *writing* process's PTE to the new frame, marked **writable**,
   - decrements the old frame's refcount. If the count drops to 1, the remaining owner's page can be
     marked writable again (no more sharing → no need to fault next time).
5. The faulting write is retried and now succeeds on the private copy.

```
After fork (before any write):          After child writes page P:
 parent PTE ─┐                            parent PTE ─▶ [frame A]  (RO→RW, refcount 1)
             ├─▶ [frame A]  RO, refc=2     child  PTE ─▶ [frame B]  (fresh copy, RW)
 child  PTE ─┘                                            (refcount of A now 1)
```

So the only pages actually copied are the ones actually written. COW is also used for `mmap` private
mappings and for zero pages (all fresh anonymous memory maps to one shared read-only **zero page**
until first written). The interview one-liner: *"fork shares frames read-only and copies a page only
on the first write, triggered by a protection fault — so the cost is proportional to pages modified,
not pages mapped."*

---

## 8. Tying it back: TLB vs cache on a context switch

Now the two "flush?" facts sit side by side cleanly (a favorite follow-up):

- **Data caches (L1d/L2/L3):** physically tagged → **not flushed** on a context switch (Module 05).
- **TLB:** virtually keyed and per-address-space → its entries are wrong for the next process, so
  historically flushed on the CR3 reload; modern CPUs avoid the flush with **ASIDs/PCIDs** that tag
  entries by address space. The *kernel's* own global (kernel-space) translations are marked
  **global** so they survive switches regardless.

So a context switch's memory cost is mostly TLB-related (and cold caches after the other process
runs), not a data-cache flush.

---

## 9. Page-replacement — which frame to evict

Physical RAM is finite. When every frame is occupied and a new page must come in (a major fault, §6),
the kernel must **evict a victim** to free a frame. The policy that picks the victim decides how often
you fault next — a bad choice evicts a page you're about to touch, forcing another expensive disk read.

The algorithms, from naïve to practical:

- **FIFO** — evict the page that's been resident longest, by load order. Trivial (one queue), but
  ignores *usage*: a hot page loaded early gets evicted while cold recent pages stay. Worse, it suffers
  **Belady's anomaly** — giving it *more* frames can *increase* faults, which no sane policy should do.
  Essentially never used alone.
- **LRU** — evict the **least-recently-used** page, betting the past predicts the future (temporal
  locality). Near-optimal hit rate, but exact LRU means timestamping or reordering a list on *every
  memory access* — far too expensive to do in hardware/software on the hot path. Theoretically ideal,
  practically unaffordable.
- **Clock / Second-Chance** — the real-world approximation of LRU, and what Linux-family kernels
  actually use. It exploits the **reference (accessed) bit** the MMU already sets in the PTE for free
  on each access. Frames sit in a circular list with a "clock hand":

```
         ┌───────── clock hand sweeps ──────────┐
         ▼                                       │
   [P0 r=1] → [P1 r=0] → [P2 r=1] → [P3 r=0] → [P4 r=1] → (wraps to P0)
         │                                       │
   On eviction need: hand advances —
     r == 0 ?  evict THIS page (victim found).
     r == 1 ?  clear r to 0 (give it a "second chance"), advance hand, keep scanning.
```

A page referenced since the last sweep (r=1) survives one pass but gets its bit cleared; if it isn't
touched again before the hand returns, it's evicted. This gives LRU-like behavior — recently used pages
survive — at O(1) amortized cost and **zero per-access bookkeeping** (the hardware sets the bit). Linux
refines it into two LRU lists (active/inactive) with the same reference-bit mechanism.

| Policy | Hit rate | Cost per access | Used in practice? |
|--------|----------|-----------------|-------------------|
| FIFO | poor (Belady) | O(1), queue only | no |
| LRU (exact) | near-optimal | high — update on every access | no (too costly) |
| Clock / Second-Chance | near-LRU | ~zero (HW ref bit) | **yes** (Linux active/inactive) |

HFT relevance is inverted: on a correctly tuned trading box **page replacement should never run at
all** — the working set is locked (`mlockall`) and swap is off, so there's never a victim to pick. The
algorithm matters precisely because you want to guarantee it stays idle; any eviction is a latency
event. Knowing *why* Clock is cheap is also a clean interview demonstration of the reference bit.

---

## 10. Thrashing & the working set

Push overcommit too far — too many active processes, too little RAM — and the system enters
**thrashing**: it spends more time servicing page faults (reading pages from disk) than running real
work. It's a collapse, not a slowdown, because of a vicious cycle: not enough frames → high fault rate
→ processes block on disk I/O → CPU looks idle → scheduler admits/runs *more* processes → even less RAM
per process → even higher fault rate.

```
  Healthy:              fault rate ~ few/sec        CPU util ~ 90%+   (working set fits)
  Thrashing:            fault rate ~ 100s–1000s/sec CPU util collapses to ~single digits
                        (everyone blocked on disk; the CPU starves while the disk saturates)
```

The **working-set model** names the fix: a process's working set W(t, Δ) is the set of pages it
touched in the last Δ of execution. If the sum of all processes' working sets exceeds physical RAM,
thrashing is guaranteed. So the kernel should admit only as many processes as their working sets fit —
and if it can't, **suspend** (swap out whole) some processes rather than let everyone thrash.

Prevention: fewer concurrent processes, more RAM, or lock the hot pages resident. This is the exact
pathology that `mlockall` + swap-off + pre-faulting (§6) exist to make **structurally impossible** on a
trading box: with the working set pinned in RAM and no swap device, there is no page to evict and no
disk to fault to, so the thrashing cycle can't even begin. Thrashing is the disease; memory locking is
the vaccine.

---

## 11. Huge pages & TLB reach

The TLB holds only a few entries (§7) — say **64** per level for a given page size. The memory those
entries can cover without a miss is the **TLB reach**:

```
  reach = (TLB entries) × (page size)

  4 KB pages:  64 × 4 KB  = 256 KB   ← tiny; any working set past 256 KB thrashes the TLB
  2 MB pages:  64 × 2 MB  = 128 MB   ← 512× more memory covered by the same 64 entries
  1 GB pages:  64 × 1 GB  = 64 GB    ← entire datasets with near-zero TLB misses
```

Worked example — an **8 GB** working set (a market-data book, a tick database in RAM):

- With **4 KB** pages it spans 8 GB / 4 KB ≈ **2 million** distinct translations. A 64-entry TLB caches
  0.003% of them, so you miss the TLB almost constantly, and every miss is a 4-level page-table walk
  (§3) — up to 4 dependent memory accesses on the hot path.
- With **2 MB** pages the same 8 GB is 8 GB / 2 MB ≈ **4096** translations. Still more than 64, but now
  table walks are rare *and* each walk is shorter (a 2 MB page skips the last level: PML4→PDPT→PD→**PTE
  points straight at the 2 MB frame**, a 3-level walk). TLB misses and walk cost both plummet.

That's why huge pages are standard HFT tuning: they shrink both TLB pressure and page-walk depth for
large resident structures. Setup (2 MB pages):

```sh
# 1. Reserve huge pages at boot (GRUB), e.g. 1024 × 2 MB = 2 GB:
GRUB_CMDLINE_LINUX="hugepagesz=2M hugepages=1024"

# 2. (optional) mount hugetlbfs for file-backed huge-page allocations:
mount -t hugetlbfs nodev /mnt/huge
```

```c
/* 3. Allocate anonymous memory backed by huge pages: */
void *p = mmap(NULL, size,
               PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
               -1, 0);
```

**1 GB pages** exist too (`hugepagesz=1G`), reserved only at boot (they can't be allocated once memory
is fragmented) — used for very large, static datasets where even 2 MB TLB reach isn't enough. Caveats:
huge pages are not swappable (fine — you've disabled swap anyway) and waste memory if sparsely used
(internal fragmentation — a 2 MB page for 4 KB of data wastes 2044 KB). **Transparent huge pages (THP)**
let the kernel promote pages automatically, but HFT shops often *disable* THP — the background
`khugepaged` compaction and promotion cause unpredictable latency spikes; explicit `MAP_HUGETLB` gives
determinism instead.

---

## 12. Address-space layout & protection

A process's virtual space isn't uniform — it's divided into **segments**, each with distinct contents
and permissions (the full `brk`/`mmap` picture is in Module 12; here's the layout and the *why*):

```
  high addr ┌──────────────────┐
            │   stack          │  grows DOWN ↓   (RW, NX)
            │        ↓         │
            │        ·         │  ← one shared gap →
            │        ↑         │
            │   mmap region    │  (shared libs, big mallocs, file maps)
            │   heap           │  grows UP ↑     (RW, NX)
            │   .bss / .data   │  globals        (RW, NX)
            │   .text (code)   │  machine code   (R-X — readable, executable, NOT writable)
  low addr  └──────────────────┘
```

**Why stack and heap grow toward each other** from opposite ends: they share one gap of free address
space between them. Neither needs a pre-committed fixed size — the stack can grow deep (recursion)
while the heap stays small, or vice versa, and only their *sum* is bounded by the gap. A fixed split
would waste space on whichever grew less. Opposite growth = maximally flexible use of one shared pool.

**Per-segment protection** — the PTE permission bits (§3) are set per segment to enforce **W^X**
("write XOR execute", a page is never both writable and executable):

- `.text` is **R-X**: executable but read-only, so a bug can't overwrite your own code.
- heap, stack, data are **RW + NX** (no-execute): writable but *not* executable, so an attacker who
  injects bytes into a buffer can't jump to and run them as code. The **NX bit** (PTE bit 63 on x86-64)
  is the hardware enforcement; it defeats classic code-injection exploits.

**ASLR (Address Space Layout Randomization)** randomizes the base addresses of segments (stack, heap,
mmap, and with PIE the code too) each run, so an attacker can't predict where anything lives to build a
reliable exploit. The HFT footnote: ASLR adds a small amount of **nondeterminism** — addresses differ
run to run, which complicates reproducing a latency anomaly or a crash against a fixed core dump — so
some shops disable it on dedicated, isolated trading boxes (`setarch -R`, or
`kernel.randomize_va_space=0`) to get byte-for-byte reproducible layouts for debugging. It's a security
trade made only because the box is already physically and network-isolated.

---

## Common pitfalls / misconceptions

- **"A pointer is a physical memory address."** No — it's a *virtual* address, translated on every
  access. Two processes can hold the identical pointer value pointing at completely different memory.
- **"`malloc` gives you physical RAM."** It gives *virtual* mappings; physical frames are allocated
  lazily on first touch via minor page faults (demand paging).
- **"`fork()` copies all the memory."** COW means it copies page tables + marks pages read-only;
  actual page copies happen only on write. Cheap unless the child writes a lot.
- **"TLB miss = page fault."** No. A TLB miss just means "walk the page tables" — usually the mapping
  *exists*, it just wasn't cached. A *page fault* is when the walk finds no valid mapping (or a
  permission violation) and traps to the kernel.
- **"Bigger address space needs a bigger page table."** Multi-level tables allocate only the branches
  you use, so a sparse 48-bit space costs a few KB of tables.

---

## Quiz — tough problems

**Q1.** How many memory accesses does a single load take on a **TLB miss** with a 4-level page table,
if none of the page-table lines are cached and the data itself then misses to DRAM?
**Answer:** 4 accesses to walk the levels (PML4, PDPT, PD, PT) + 1 for the data = **5 memory
accesses**. In practice the table lines are often cached, so it's cheaper — but worst case it's a
5-deep dependent chain, which is why TLB misses hurt and huge pages (fewer levels/entries) help.

**Q2.** Why does the first write to a freshly `malloc`'d 1 GB buffer take much longer per byte than
the second pass over it?
**Answer:** Demand paging — `malloc`/`mmap` only reserved virtual mappings; the first touch of each
page triggers a **minor page fault** that allocates and maps a physical frame (and often zeroes it).
The second pass finds everything mapped and cached → far faster. This "first-touch" effect also
decides NUMA placement (the frame is allocated on the writing thread's node).

**Q3.** A process `fork()`s and the child immediately `exec()`s a new program. How much memory was
copied?
**Answer:** Essentially none of the *data* pages — COW shared them read-only, and `exec` throws the
whole address space away before any write forces a copy. Only the page tables were duplicated on
fork (and freed at exec). This is why the common fork+exec pattern is cheap despite "copying" a large
process.

**Q4.** With 4 KB pages and 8-byte PTEs, how many entries per page-table node, and how many index
bits per level?
**Answer:** 4096 / 8 = **512 entries** per node → **9 index bits** per level. Four levels × 9 = 36
VPN bits + 12 offset bits = 48-bit virtual addresses. (This is exactly x86-64's 4-level scheme.)

**Q5.** Is a TLB flushed on a context switch on a modern CPU? What about the L1 data cache?
**Answer:** The TLB is address-space-specific; classically flushed on switch, but modern CPUs use
**PCIDs/ASIDs** to tag entries per process and avoid the flush. The **L1 data cache is NOT flushed** —
it's physically tagged, so its lines remain valid across processes.

**Q6.** Explain what happens on a write to a copy-on-write page, in order.
**Answer:** The page is mapped read-only, so the write raises a **protection page fault** → trap to
kernel → handler sees a COW page → allocates a new frame, copies the old page's bytes into it,
remaps the writer's PTE to the new frame as writable, decrements the old frame's refcount → retries
the write, which now succeeds privately. Other sharers keep the original.

**Q7.** Why can't we just make the TLB huge to eliminate walks?
**Answer:** Like L1, the TLB is on the critical path and must be fast (~1 cyc), which caps its size
and entry count; it's also highly/fully associative (expensive per entry). So it can only hold a few
hundred to a couple thousand translations. Huge pages help by covering more memory per entry (one
2 MB entry replaces 512 4 KB entries), reducing TLB pressure instead of enlarging the TLB.

**Q8.** A load's virtual and physical addresses share their low 12 bits. Why, and what does it enable?
**Answer:** The 12-bit page offset isn't translated — only the page number is — so it's identical in
VA and PA. This lets an **L1 VIPT** cache index using the VA's low bits (available immediately) while
the TLB translates in parallel, then tag-check with the physical frame. (Module 05.)

**Q9.** A process maps a 100 GB file with `mmap` on a machine with 32 GB RAM and it succeeds. How?
**Answer:** `mmap` only creates a **virtual mapping**; no physical frames are allocated until pages
are touched (demand paging). Accessed pages are paged in (major faults from the file) and unused/old
ones evicted, so resident memory stays bounded. You can address more virtual memory than physical RAM.

**Q10.** After `fork()`, the parent writes to a global variable. Does the child see the change?
**Answer:** No. The write triggers COW: the parent gets a private copy of that page and modifies it;
the child still sees the original shared frame. `fork()` gives separate address spaces — post-fork
writes are private to each process (that's the whole point vs. threads, which *do* share memory).

**Q11.** Exact LRU has the best hit rate of any practical replacement policy — so why don't real
kernels use it, and what do they use instead?
**Answer:** Exact LRU requires updating ordering **on every single memory access** (timestamp or move
to list head), which is far too expensive on the hot path. Kernels approximate it with **Clock /
Second-Chance**, which rides the **reference bit** the MMU sets *for free* on each access: a circular
scan evicts a page with r=0 and clears r=1 pages (second chance) instead of evicting them. Near-LRU
behavior at ~zero per-access cost. Linux splits it into active/inactive LRU lists on the same idea.

**Q12.** FIFO replacement can get *worse* when you add RAM. Name the effect and why it disqualifies
FIFO.
**Answer:** **Belady's anomaly** — for FIFO, more frames can yield *more* page faults on some
reference strings, because FIFO ignores usage and evicts purely by load order. A sane policy must have
the "stack property" (more frames never increases faults), which LRU and Clock satisfy and FIFO does
not. That's one reason FIFO is never used alone.

**Q13.** A box has a 64-entry dTLB. With 4 KB pages, how much memory can it map without a miss, and how
does switching an 8 GB working set to 2 MB huge pages change the picture?
**Answer:** TLB reach = 64 × 4 KB = **256 KB** — an 8 GB working set (≈2 million 4 KB translations)
thrashes the TLB, missing almost every access and paying a 4-level walk each time. With 2 MB pages the
8 GB needs only ≈4096 translations, reach jumps to 64 × 2 MB = **128 MB**, and each walk is one level
shorter (the PTE points straight at the 2 MB frame). TLB misses and walk depth both collapse — the
reason huge pages are standard HFT tuning for large resident structures.

**Q14.** Why do stack and heap grow toward each other from opposite ends of the address space?
**Answer:** They share a single gap of free virtual address space between them. Growing from opposite
ends means neither needs a fixed pre-committed size — only their *sum* is bounded by the gap, so a
deep-recursion/small-heap process and a small-stack/huge-heap process both use the same layout
optimally. A fixed partition would waste whichever side grew less.

**Q15.** What is the NX bit and the W^X policy, and what attack class do they defeat?
**Answer:** **NX (no-execute)** is a PTE permission bit marking a page non-executable. **W^X** sets it
so no page is simultaneously writable and executable: code (`.text`) is R-X (executable, read-only),
while heap/stack/data are RW+NX (writable, non-executable). This defeats classic **code injection** —
an attacker who writes shellcode into a buffer can't execute it, because that page is NX. (It pushed
attackers toward return-oriented programming instead, countered by ASLR + stack canaries.)

**Q16.** Thrashing: define it, and explain why it's a collapse rather than a gradual slowdown.
**Answer:** Thrashing is when the system spends more time paging (reading pages from disk) than doing
useful work, because the combined working sets exceed RAM. It's a *collapse* due to a positive-feedback
loop: too few frames → high fault rate → processes block on disk → CPU looks idle → scheduler runs more
processes → even less RAM each → even higher fault rate. CPU utilization crashes toward zero while the
disk saturates. The fix is admission control via the working-set model (or, in HFT, pinning the working
set so it can never start).

---

## Indian HFT interview questions

**Q: Explain the full flow of how we read/write data — from software to hardware.** (On the user's
list — Tower Research, HRT, Jump.)
**A:** The CPU issues a load with a *virtual* address. First it translates: check the **TLB** for the
VPN→PFN mapping; on a hit you have the frame in ~1 cycle, on a miss the **MMU walks the multi-level
page table** (CR3→PML4→PDPT→PD→PT) and fills the TLB — or raises a **page fault** to the kernel if
there's no valid mapping. With the physical address formed (offset copied through), it queries the
**caches**: L1 (~4 cyc) → L2 (~12) → L3 (~40) → **DRAM** (~200) on misses, filling each level on the
way up. On L1 (VIPT) the cache indexing overlaps the TLB lookup. A write does the same translation,
updates the L1 line, marks it dirty, posts via the store buffer, and relies on MESI coherency for
other cores; it writes back to memory only on eviction. Net: one access ranges from ~4 cycles to
hundreds (or a page fault to disk) — HFT is about keeping it at the top.

**Q: How is copy-on-write implemented?** (On the user's list — Optiver, Graviton, Quadeye.)
**A:** On `fork()` the kernel copies only the page tables and points both processes at the same
physical frames, marked **read-only**, with a per-frame **refcount**. Reads share freely. The first
**write** hits the read-only bit → **protection page fault** → the kernel allocates a fresh frame,
copies the page, remaps the writer's PTE writable, and drops the old frame's refcount. So only
modified pages are ever copied — `fork` cost is proportional to pages *written*, not pages mapped.
Also used for private `mmap` and the shared zero-page.

**Q: What is the TLB and what happens on a TLB miss?** (AlphaGrep, Da Vinci.)
**A:** The TLB caches recent virtual→physical page translations so most accesses skip the page-table
walk. On a miss, the hardware page-table walker reads CR3 and walks the levels (up to 4 dependent
memory accesses on x86-64), installs the translation in the TLB, and the access proceeds — unless no
valid mapping exists, which raises a page fault. Huge pages and keeping the working set's page count
small reduce TLB misses.

**Q: Why does virtual memory exist / what problem does it solve?** (Conceptual — Optiver.)
**A:** Isolation (each process has a private space; it can't name another's memory — the kernel
enforces via page tables), relocation (programs assume a fixed layout; the OS maps virtual pages to
any free physical frames), and overcommit (total virtual memory can exceed RAM; unused pages live on
disk, brought in on demand). The cost is per-access translation, which the MMU+TLB make near-free.

**Q: What's a page fault, and what kinds are there?** (HRT, Millennium.)
**A:** A CPU exception when a translation can't complete. *Minor*: page is in RAM but unmapped here
(remap it — cheap). *Major*: page is on disk/swap or a file not yet loaded (read from disk — µs-ms,
latency-fatal). *Invalid*: no mapping/permission violation → SIGSEGV. HFT boxes disable swap and
`mlockall` memory to eliminate major faults, and pre-fault pages at startup.

**Q: How is a multi-level page table better than a single flat one?** (Quadeye, Jump.)
**A:** A flat table for a 48-bit space (2³⁶ pages × 8 B ≈ 512 GB) would be enormous and almost
entirely empty. A multi-level (radix) table only allocates the branches you actually use, so a
typical sparse process needs a few KB of tables. The trade-off is a deeper walk on a TLB miss (one
memory access per level), which the TLB and cached table lines mitigate.

**Q: On a context switch, what memory state has to change — TLB, cache, page table base?** (Advanced
— Tower Research, HRT.)
**A:** The **page-table base register (CR3)** is reloaded to the new process's tables. The **TLB**
holds the old process's virtual→physical mappings, so entries are flushed — or, on modern CPUs, kept
and disambiguated by **PCID/ASID**; kernel *global* pages are preserved. The **data caches are NOT
flushed** (physically tagged, unambiguous across processes). So the switch's real memory cost is TLB
churn plus running cold on the new process's working set — not a cache flush.

**Q: What are huge pages and why would a low-latency system use them?** (Optiver, Graviton, NK
Securities.)
**A:** Huge pages (2 MB or 1 GB vs the default 4 KB) make each TLB entry cover far more memory —
**TLB reach** goes from 64 × 4 KB = 256 KB to 64 × 2 MB = 128 MB with the same 64 entries. For a large
resident structure (an 8 GB tick DB), 4 KB pages mean ~2 million translations and near-constant TLB
misses, each triggering a multi-level page-table walk; 2 MB pages cut that to ~4096 translations and a
shorter walk (the PTE points straight at the 2 MB frame). You reserve them at boot
(`hugepagesz=2M hugepages=N`) and map with `MAP_HUGETLB`. Most shops disable *transparent* huge pages
(THP) though — `khugepaged`'s background promotion causes latency jitter; explicit `MAP_HUGETLB` is
deterministic.

**Q: What is thrashing and how do you prevent it on a trading box?** (Millennium, Quadeye.)
**A:** Thrashing is when combined working sets exceed RAM, so the system spends more time paging from
disk than computing — and it's a feedback collapse (more faults → processes block → scheduler runs more
→ worse). General fixes are admission control via the working-set model and more RAM. On a trading box
you make it *impossible*: `mlockall` pins the working set resident, swap is disabled, and pages are
pre-faulted at startup — with no swap device and nothing evictable, there's no page to fault on and the
cycle can't start.

**Q: Walk me through the virtual address-space layout of a process.** (Da Vinci, Jump — conceptual.)
**A:** Low to high: `.text` (code, R-X), `.data`/`.bss` (globals, RW+NX), heap (grows up, RW+NX), the
mmap region (shared libs, large allocations, file maps), and the stack at the top (grows down, RW+NX).
Stack and heap grow toward each other so they share one gap and neither needs a fixed size. Permissions
enforce **W^X** (code is executable-not-writable, data is writable-not-executable via the NX bit) to
block code injection, and **ASLR** randomizes segment bases for security — though some HFT boxes
disable ASLR for reproducible layouts when debugging latency anomalies, since the box is already
isolated.

---

## Key takeaways

- Programs use **virtual addresses**; the **MMU** translates every access to a **physical address**
  via page tables, giving isolation, relocation, and overcommit.
- Translation is **VPN → PFN**; the page **offset passes through untranslated** (which is what makes
  VIPT L1 caches possible).
- Page tables are **multi-level** (x86-64: 4 levels, CR3 at the root) so sparse address spaces cost
  little; a TLB miss walks them (up to 4 accesses).
- The **TLB** caches translations; a **page fault** is when translation finds no valid mapping and
  traps to the kernel (minor = cheap remap, major = disk = latency death).
- The **full access flow** is TLB → (walk / fault) → physical address → L1/L2/L3 → DRAM, ranging from
  ~4 cycles to hundreds; keeping accesses at the top is the whole game.
- **Copy-on-write** shares frames read-only and copies a page only on the first write (via a
  protection fault + refcount), making `fork()` cheap.
- On a context switch the **TLB** is flushed/ASID-tagged and **CR3** reloaded, but **data caches are
  not flushed**.
- When RAM is full, the kernel evicts a victim by **page replacement**: FIFO (bad — Belady's anomaly),
  LRU (ideal but too costly), **Clock/Second-Chance** (the practical choice — approximates LRU via the
  free MMU reference bit).
- **Thrashing** is a paging-induced collapse when working sets exceed RAM; the **working-set model**
  (and, in HFT, `mlockall` + no swap) prevents it.
- **Huge pages** multiply **TLB reach** (64 × 2 MB = 128 MB vs 64 × 4 KB = 256 KB) and shorten page
  walks — standard HFT tuning for large resident structures via `MAP_HUGETLB`.
- The address space is **segmented** (code R-X, data/heap/stack RW+NX); stack and heap grow toward each
  other to share one gap; **W^X/NX** blocks code injection and **ASLR** randomizes bases (sometimes
  disabled on trading boxes for reproducibility).

**Next:** [07 — Privilege & the OS role](07-privilege-and-os-role.md) — kernel vs user mode, the mode
bit that makes all this protection enforceable, and why a user process can't just grant itself
kernel privileges.
