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

**Next:** [07 — Privilege & the OS role](07-privilege-and-os-role.md) — kernel vs user mode, the mode
bit that makes all this protection enforceable, and why a user process can't just grant itself
kernel privileges.
