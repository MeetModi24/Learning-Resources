# Module 12 — Allocators & the I/O path

Two questions bottom out here that HFT interviewers love because they force you *below* the language:
"what actually happens when you call `malloc` / `new`?" and "what actually happens when you call
`read()` / `write()`?" Both are journeys from a one-line C call, through the C library, into the
kernel, and down to hardware (DRAM, disk, the NIC). This module traces all three: the heap allocator's
internals, the OS I/O path, and the special case of raw sockets — the layer where market data actually
enters your process. The [C++ guide's Module 3](../cpp-guide/03-dynamic-memory.md) and
[`../guide-os-net`](../../guide-os-net/06-custom-memory-pools.md) show how HFT *avoids* the slow paths
below; this module explains the machinery they're avoiding so you can defend *why*.

---

## 1. What `malloc` actually is, and where its memory comes from

`malloc` is **not** a system call. It's a normal C-library function (in glibc, an implementation
called **ptmalloc2**) that runs entirely in user space *most of the time*. Its whole job is to be a
middleman: it asks the kernel for memory in big chunks, then hands out small pieces of those chunks to
your program on demand, and recycles pieces you `free`. It does this because talking to the kernel is
expensive (a syscall + a page fault — Modules 6, 8), so you want to do it rarely and in bulk.

There are exactly two ways `malloc` gets raw memory from the OS:

- **`brk` / `sbrk`** — grows the **heap segment**, a contiguous region that sits just above your
  program's data segment. `brk` moves a single pointer (the "program break") upward, extending one
  linear region. This is cheap and used for the vast majority of small allocations. The catch: because
  it's one contiguous region grown by moving one boundary, memory in the *middle* can't easily be
  returned to the OS — if you `free` a chunk in the middle, the break can't move down past the
  still-live chunks above it. That freed memory stays owned by your process (available for reuse by
  future `malloc`s, but not given back to the kernel).

- **`mmap`** — asks the kernel for a fresh, independent region of virtual memory somewhere in the
  address space (the "memory-mapping segment", between the heap and the stack). glibc routes **large**
  allocations here — anything `≥ M_MMAP_THRESHOLD`, **128 KB by default** (the threshold is dynamic and
  can grow). Why: an `mmap` region is standalone, so `free`ing it can `munmap` and return the memory to
  the OS *immediately*, independent of everything else. Big allocations also don't fragment the main
  heap.

```
   Process virtual address space (grows as shown)

   high ┌────────────────────────┐
        │  stack   ↓ grows down   │
        ├────────────────────────┤
        │        (gap)            │
        ├────────────────────────┤
        │  mmap segment           │  ← large mallocs (≥128KB), shared libs, file mmaps
        │   ↑ grows toward heap   │
        ├────────────────────────┤
        │        (gap)            │
        ├────────────────────────┤
        │  heap    ↑ grows up     │  ← small mallocs; brk pointer = top of heap
        │  ── program break ──    │
        ├────────────────────────┤
        │  .bss / .data           │
        │  .text (code)           │
   low  └────────────────────────┘
```

**Key mental model:** `malloc` manages a *pool of virtual memory it already owns*. A call to `malloc`
usually just carves a piece out of that pool with no kernel involvement at all. Only when the pool is
exhausted does it call `brk`/`mmap` to get more — and *that* call is the expensive, unpredictable one
(syscall + first-touch page faults). This unpredictability is exactly why HFT hot paths never call
`malloc`.

---

## 2. How the allocator tracks chunks: headers, boundary tags, and free lists

Here's the puzzle that reveals the design: `free(p)` is given **only a pointer** — no size. How does it
know how many bytes to reclaim? The answer: the allocator stores bookkeeping **just before** the
pointer it returns to you.

Every allocation is a **chunk**: a small header immediately followed by the usable region whose address
you receive. In glibc the header holds the chunk's **size** (rounded up to a 16-byte multiple, so the
low bits are always zero) and, in those spare low bits, **flags** — most importantly `PREV_INUSE`
(is the physically-previous chunk free?).

```
   what malloc returns to you  ──┐
                                 ▼
   ┌──────────────┬──────────────────────────────────────┐
   │  size | flags │   your usable bytes (payload)        │
   └──────────────┴──────────────────────────────────────┘
   ^ chunk start   ^ p (the pointer you get)
     (p - 8/16)

   free() does:  header = p - sizeof(header);  size = header.size & ~0xF;
```

When a chunk is **free**, its payload is dead space, so the allocator reuses it to store **free-list
pointers** (fd/bk — forward/back links) threading all free chunks of a class together. This is why a
use-after-free that writes to freed memory corrupts the allocator's linked lists (a classic exploit
primitive).

**Boundary tags** make coalescing cheap. When you `free` a chunk, the allocator wants to merge it with
adjacent free chunks to fight fragmentation. It can find the *next* chunk by adding `size`; to find the
*previous* one, a free chunk also stores its size at its **end** (the boundary tag / `prev_size`
field), so the allocator can look just before the current header to see if the previous chunk is free
and how big it is. Merge both neighbours if free → one bigger chunk.

The free chunks are bucketed into **bins** by size for fast lookup:

- **Fastbins** — small chunks (≤ ~128 B), kept in simple LIFO singly-linked lists, *not* coalesced
  immediately (speed over compaction). A `malloc` of a fastbin size is nearly O(1): pop the head.
- **Small bins / large bins** — exact-size (small) or size-range, sorted (large) doubly-linked lists.
- **Unsorted bin** — a temporary holding pen where just-freed chunks land before being sorted into
  small/large bins on a later allocation (a caching optimization).
- **Top chunk** — the "wilderness" at the very top of the heap adjacent to the break; if no bin can
  satisfy a request, malloc carves from the top, and if the top is too small, calls `sbrk`/`mmap`.

Two more operations complete the picture:

- **Splitting** — if the best-fit free chunk is bigger than requested, split it: return the front,
  put the remainder back as a smaller free chunk.
- **Fragmentation** — **internal** (you asked for 40 B, got a 48-B chunk rounded to alignment — waste
  *inside* the chunk) vs **external** (plenty of total free bytes, but scattered in pieces too small for
  the next request — waste *between* chunks). Coalescing and bins fight external fragmentation.

---

## 3. Multithreaded allocators: arenas, tcmalloc, jemalloc

A single global free list shared by all threads would need a lock on every `malloc`/`free` — a
scalability and latency disaster. Modern allocators fix this with **per-thread / per-core structure**:

- **glibc arenas** — glibc keeps multiple **arenas** (independent heap regions each with its own bins
  and lock). The main arena uses `brk`; secondary arenas use `mmap`. A thread is assigned an arena; if
  its arena's lock is contended, it tries another or creates one (up to a cap, typically `8 × ncores`).
  This reduces — but doesn't eliminate — lock contention.

- **tcmalloc** (Google) and **jemalloc** (FreeBSD/Facebook) — the allocators HFT and low-latency shops
  actually reach for. Their key idea: a **per-thread cache** of free objects segregated by **size
  class**. The fast path — allocate/free an object whose size class is cached — touches only
  thread-local memory, so it takes **no lock at all** and is a handful of instructions. Only when the
  thread cache is empty/full does it go to a central heap (with a lock) to refill/flush in a batch.
  Size-class segregation (e.g. round every request up to one of ~80 fixed sizes) also keeps
  fragmentation low and predictable.

```
   tcmalloc fast path (no lock):

   thread A cache          thread B cache
   ┌───────────────┐       ┌───────────────┐
   │ 16B: [][][]    │       │ 16B: []        │
   │ 32B: [][]      │       │ 32B: [][][]    │      ← per-thread, lock-free pop/push
   │ 64B: []        │       │ 64B: [][]      │
   └───────┬───────┘       └───────┬───────┘
           │ refill/flush in batches (rare, locked)
           ▼
   ┌───────────────────────────────────────┐
   │        central heap  (size classes)    │
   └───────────────────────────────────────┘
```

**Why HFT cares:** the general allocator's problems are all *latency* problems — it takes a lock
(unpredictable stall under contention), it can fault in new pages (a syscall + page-fault spike), and
its latency depends on the current fragmentation state (variable). A per-thread-cache allocator makes
the *common* case lock-free and flat, which is why simply linking `jemalloc`/`tcmalloc` often cuts
tail latency for free. But even that isn't deterministic enough for the true hot path — hence custom
pools/arenas ([`../guide-os-net`](../../guide-os-net/06-custom-memory-pools.md)) that never call `malloc`
at all during trading.

---

## 4. `new`, `delete`, and placement new

C++ `new` is **two operations fused**:

1. **`operator new(size)`** — a plain function that obtains raw memory. The default implementation
   just calls `malloc` (and throws `std::bad_alloc` on failure). It is *replaceable* — you can define
   your own global or per-class `operator new`.
2. **the constructor** — runs in that raw memory, turning bytes into a live object.

`delete` is the mirror: **destructor**, then **`operator delete`** (which calls `free`). So:

```cpp
Order* o = new Order{id, px, qty};
//   ≡  void* m = operator new(sizeof(Order));   // malloc under the hood
//      Order* o = new (m) Order{id, px, qty};    // placement new: construct in m

delete o;
//   ≡  o->~Order();               // destructor
//      operator delete(o);        // free under the hood
```

**Placement new** — `new (ptr) T{...}` — skips allocation entirely and just constructs a `T` in memory
*you* already own. It's the bridge between a raw allocator (a pool that hands back `void*`) and typed
C++ objects, and it's why the object-pool pattern works. You must then call the destructor **manually**
(`o->~Order()`), because you took over memory management.

**Why HFT overrides these:** because `operator new` calling `malloc` on the hot path is exactly the
non-determinism we're eliminating. Shops either override `operator new`/`operator delete` (per-class or
global) to route through a pool/arena, or — more commonly and more surgically — allocate everything
from pools up front with placement new and never touch `new`/`delete` during trading. For a trivial
POD `Order` (no non-trivial constructor/destructor) you can even skip construction/destruction and just
reinterpret pool bytes.

---

## 5. The OS I/O path: what `read()` / `write()` really do

`read(fd, buf, n)` looks like "copy n bytes from the file into buf." Underneath, it's a syscall
(Module 8) that drops into the kernel and traverses several layers. The default (**buffered I/O**) path:

```
   user space          │  kernel space
   ─────────────────── │ ──────────────────────────────────────────────
   read(fd, buf, n)     │
      │ syscall ────────┼──► VFS (generic file layer: fd → struct file → inode)
                        │        │
                        │        ▼
                        │     filesystem (ext4/xfs: which disk blocks?)
                        │        │
                        │        ▼
                        │     PAGE CACHE  ── HIT ──► copy page → user buf ──► return
                        │        │ MISS
                        │        ▼
                        │     block layer / device driver
                        │        │  program DMA, then BLOCK the calling thread
                        │        ▼
   ┌────────────────────┼──  DISK/SSD ──DMA──► fills a page-cache page in DRAM
   │  (device works      │        │  (CPU is free; thread sleeps — Module 10)
   │   on its own)       │        ▼
   └────────────────────┼──► device raises an INTERRUPT (Module 9) on completion
                        │        │  ISR wakes the blocked thread
                        │        ▼
                        │     copy page-cache page → user buf ──► return
```

The **page cache** is the crux: the kernel keeps recently-used file pages in DRAM. A `read` that hits
the cache never touches the disk — it's just a memory-to-memory copy from a kernel page into your
buffer. A miss triggers a **DMA** (Direct Memory Access): the disk controller writes data straight into
a page-cache page in DRAM *without the CPU copying it byte by byte*; meanwhile the kernel **blocks** the
calling thread (a context switch — Module 10), and when the DMA finishes the device raises an
**interrupt** (Module 9) whose handler marks the data ready and wakes the thread.

`write(fd, buf, n)` is even more decoupled: it copies your bytes into page-cache pages, marks them
**dirty**, and returns — the data is still only in DRAM. The kernel's **writeback** flushes dirty pages
to disk later (or on `fsync`). This is why `write` returning does *not* mean the data is durable.

Variations that change the copy count — and why they matter for latency:

- **Direct I/O (`O_DIRECT`)** — bypass the page cache; DMA straight between device and the user buffer.
  One fewer copy and no cache pollution, but you lose caching and must obey alignment rules. Databases
  and some low-latency loggers use it.
- **`mmap`** — map the file's pages into your address space; accessing them page-faults the data in
  (Module 6). Reads then avoid the explicit read-syscall copy (you touch page-cache pages directly) —
  one fewer copy, at the cost of page-fault jitter.
- **Zero-copy** (`sendfile`, `splice`) — move data from a file to a socket entirely inside the kernel,
  never copying into user space at all. The standard "serve a file over the network fast" answer.

The through-line: **every copy and every syscall is latency.** The count of copies (device→page
cache→user→socket→…) is the thing all these mechanisms are trimming.

---

## 6. Raw sockets: which packets you actually receive

A normal socket hands you *payloads*: `socket(AF_INET, SOCK_STREAM, 0)` gives you a TCP byte stream —
the kernel has already done IP reassembly, TCP sequencing, ACKs, and stripped every header. A **raw
socket** taps in *lower*, letting you see (and forge) headers the kernel normally manages. There are two
distinct flavors, and interviewers want you to distinguish them precisely:

```
   where each socket taps the RX stack:

   wire ──► NIC ──► [Ethernet frame] ──► [ IP packet ] ──► [ TCP/UDP ] ──► app payload
                         ▲                    ▲                 ▲              ▲
                         │                    │                 │              │
                  AF_PACKET raw          AF_INET raw        (kernel owns    normal socket
                  (SOCK_RAW, L2)         (SOCK_RAW, L3)      TCP/UDP here)   (SOCK_STREAM/DGRAM)
                  whole frame,           IP payload of a                     payload only
                  incl. Ethernet hdr     matched protocol,
                                         incl. IP header
```

- **`AF_INET` + `SOCK_RAW` (a raw *IP* socket)** — you specify a protocol number (e.g. `IPPROTO_ICMP`
  for ping, `IPPROTO_UDP`, or a custom protocol). You receive IP-layer packets **of that matching
  protocol**, **including the IP header** (and, with `IP_HDRINCL`, you supply it on send too). Requires
  `CAP_NET_RAW` (root). This is how `ping` and `traceroute` craft ICMP. Crucially, you **don't** get
  traffic the kernel's own stack owns — e.g. established TCP connections are handled by the kernel; a
  raw socket only sees the protocol you asked for, and delivering to a raw socket doesn't stop the
  normal stack from also processing it.

- **`AF_PACKET` + `SOCK_RAW` (a raw *link-layer* socket, Linux)** — you receive **entire link-layer
  frames**, Ethernet header and all, straight off the interface. With `SOCK_DGRAM` instead of
  `SOCK_RAW` the link-layer header is cooked/stripped for you. Put the interface in **promiscuous mode**
  and you receive **every** frame on the wire, not just those addressed to you — this is exactly what
  `tcpdump`/**libpcap** use to sniff. This is the lowest tap in the standard stack.

**What you *don't* get / common gotchas:** a raw IP socket won't magically give you all TCP/UDP traffic
(the kernel stack still owns those ports); you're responsible for checksums and header fields you
supply; and everything above requires elevated privileges. Filtering (e.g. **BPF**/`SO_ATTACH_FILTER`)
is how tools avoid drowning in irrelevant frames.

**HFT connection (opposite direction):** raw sockets still go *through* the kernel — they're for
inspection/crafting, not speed. HFT does the reverse: **kernel bypass** (DPDK, Solarflare/Onload,
Mellanox VMA) pulls the NIC's packets straight into user memory with **no kernel stack, no syscall, no
interrupt**, and busy-polls for them. Same goal of "get raw packets," opposite mechanism and motivation
— see [`../../guide-os-net/07-networking.md`](../../guide-os-net/07-networking.md).

---

## Common pitfalls / misconceptions

- **"`malloc` is a syscall."** No — it's a user-space library function that only *occasionally* calls
  `brk`/`mmap`. The whole point is to amortize kernel trips.
- **"`free` returns memory to the OS."** Usually not. `brk`-backed chunks stay in the process for
  reuse; only `mmap`-backed (large) allocations are actually `munmap`ped back on `free`.
- **"`write()` returning means it's on disk."** No — it's in the page cache (dirty), flushed later.
  Durability needs `fsync`/`O_SYNC`.
- **"Bigger `malloc` is always slower."** Not linearly — a fastbin hit is ~O(1); the expensive cases
  are cold paths (`sbrk`/`mmap` growth, first-touch faults, lock contention), which is why *latency
  variance*, not average, is the HFT concern.
- **Placement-new without a manual destructor** leaks the object's resources; forgetting it on
  non-trivial pool objects is a classic bug.
- **"A raw IP socket sees all traffic."** No — it sees only its matched protocol; only `AF_PACKET` +
  promiscuous mode sees everything.
- **Use-after-free "just reads garbage."** Worse — freed chunks store the allocator's own fd/bk
  pointers; writing to them corrupts the heap metadata (undefined behavior, and an exploit vector).

---

## Quiz — tough problems

**Q1.** You `malloc(16)`, then `free` it, then `malloc(16)` again. Roughly how many system calls did
this sequence make (assume the heap already has room)?
**Answer:** Zero. The first `malloc` carves from an existing heap region (or top chunk), `free` pushes
the 16-B chunk onto a fastbin, and the second `malloc` pops that exact chunk back. All user-space; no
`brk`/`mmap`. Syscalls only happen when the allocator must grow its pool.

**Q2.** `free(p)` receives only `p`. How does it know the allocation's size?
**Answer:** The size is stored in a header in the bytes *immediately before* `p` (`p - sizeof(header)`).
The allocator masks off the low flag bits to recover the rounded size. Boundary tags (a size copy at
the chunk's end) additionally let it find and coalesce the previous chunk.

**Q3.** A long-running server's RSS keeps climbing even though it `free`s everything it allocates, and
there's no leak in the code. What allocator behavior explains this?
**Answer:** External fragmentation plus `brk`'s inability to shrink. Freed small chunks stay in the
process (not returned to the OS), and if live chunks sit above freed ones the program break can't move
down. Scattered free space can't satisfy larger requests, so the heap grows. Mitigations: an allocator
like jemalloc with better return-to-OS behavior, size-class segregation, or pools.

**Q4.** Why does glibc route allocations ≥128 KB to `mmap` instead of `brk`?
**Answer:** An `mmap` region is independent, so `free` can `munmap` it and return the memory to the OS
immediately, regardless of other allocations — impossible with the single-boundary `brk` heap. It also
keeps large, long-lived blocks from fragmenting the main heap.

**Q5.** `new Foo` throws inside `Foo`'s constructor. Is the memory from `operator new` leaked?
**Answer:** No. If the constructor throws, C++ automatically calls the matching `operator delete` to
release the memory `operator new` obtained (the object was never fully constructed, so no destructor
runs, but the raw allocation is reclaimed). This pairing is guaranteed by the standard.

**Q6.** You place a POD `Order` into pool memory with placement new and later `pool.release(p)` without
calling `~Order()`. Bug or fine?
**Answer:** Fine *for a trivial type* — a POD has a trivial destructor that does nothing, so skipping it
is legal and idiomatic. For a non-trivial type (owns a `std::string`, a file, etc.) it's a resource
leak: you must call `p->~T()` before releasing the memory.

**Q7.** A `read()` of a file you just read a millisecond ago returns almost instantly; the first read
took milliseconds. Why, and what would `O_DIRECT` change?
**Answer:** The first read missed the **page cache** and waited on disk DMA + interrupt; the second hit
the page cache (a pure DRAM copy). `O_DIRECT` bypasses the page cache, so *both* reads would DMA from
the device — losing the cache speedup but avoiding cache pollution and the double copy, which some
low-latency/database workloads prefer.

**Q8.** After `write(fd, buf, n)` returns successfully, you pull the power cord. Is the data safe on
disk?
**Answer:** Not guaranteed. `write` only copied into the page cache and marked pages dirty; writeback
to disk happens later. Only after a successful `fsync(fd)` (or `O_SYNC`) is durability guaranteed.

**Q9.** You open an `AF_INET`/`SOCK_RAW` socket with `IPPROTO_ICMP`. A TCP segment and an ICMP echo
arrive. Which do you receive, and does the ICMP still reach `ping`?
**Answer:** You receive the ICMP packet (matching protocol), with its IP header included; you do *not*
receive the TCP segment (kernel stack owns it, and it doesn't match your protocol). Delivery to your
raw socket doesn't consume the packet — the kernel's own ICMP handling still runs, so a concurrently
running `ping` can also see its replies.

**Q10.** Why does simply linking `jemalloc`/`tcmalloc` (no code change) often cut a service's p99
latency?
**Answer:** Their per-thread caches make the common alloc/free path **lock-free** and constant-time,
eliminating the arena-lock contention and variable bin-search cost of the default allocator. The
average may barely move, but the *tail* — dominated by lock waits and cold paths — shrinks, which is
exactly what p99 measures.

---

## Indian HFT interview questions

**Q: Walk me through `malloc` and `free` in depth.** (Tower Research, Optiver, Graviton)
`malloc` is a user-space library allocator (glibc's ptmalloc2), not a syscall. It owns a pool of virtual
memory obtained via `brk` (small, contiguous heap) or `mmap` (large, ≥128 KB, independently
returnable). It carves requests from that pool, tracking each allocation as a **chunk** with a header
(size + flags) just before the returned pointer — that's how `free` knows the size. Free chunks go into
**bins** (fastbins for small LIFO, small/large/unsorted bins) and store fd/bk links in their now-dead
payload. `free` pushes onto a bin and coalesces with physically-adjacent free chunks via boundary tags.
Multi-threading is handled with **arenas** (per-region locks). It only calls the kernel (`sbrk`/`mmap`)
when the pool runs dry — the expensive, unpredictable case HFT avoids.

**Q: Difference between `new` and `malloc`?** (AlphaGrep, Quadeye)
`malloc` returns raw bytes and reports failure by returning `nullptr`. `new` is `operator new` (which
usually *calls* `malloc`) **plus** running the constructor, and it throws `std::bad_alloc` on failure.
`delete` runs the destructor then `operator delete` (→ `free`). `new` is type-aware and object-aware;
`malloc` is byte-aware. You can also override `operator new`/`operator delete` to control where objects
come from — the hook HFT uses to route allocations through pools.

**Q: What is placement new and why does HFT use it?** (IMC, Jump Trading)
`new (ptr) T{...}` constructs a `T` in memory you already own instead of allocating. It's how you turn
raw pool/arena bytes into typed objects without touching `malloc` on the hot path; you then destroy with
an explicit `p->~T()`. Combined with pre-allocated pools, it means "no allocation during trading" — all
memory reserved and page-faulted at startup, objects built in place with placement new.

**Q: Explain the full flow of a `read()` from software to hardware.** (Tower Research, Squarepoint)
`read()` is a syscall that traps into the kernel → VFS resolves the fd to a file/inode → the filesystem
maps the offset to disk blocks → checks the **page cache**. On a hit, it's a DRAM copy into your buffer.
On a miss, the block layer programs a **DMA** so the device fills a page-cache page directly, the kernel
**blocks** the thread (context switch), and when the DMA completes the device raises an **interrupt**
whose handler wakes the thread; then the page is copied to your buffer. `write` copies into the page
cache, marks it dirty, and returns — writeback flushes later. (Ties together Modules 6, 8, 9, 10.)

**Q: What's the page cache and how does it interact with `mmap`?** (HRT, Da Vinci)
The page cache is the kernel's DRAM cache of file pages, so repeated reads hit memory not disk. `mmap`
maps a file's pages into your address space; touching an unmapped page **page-faults** it in from the
page cache (or disk), after which you access it as normal memory — avoiding the explicit read-syscall
copy. Trade-off: fewer copies but page-fault jitter, so it's great for throughput, less so for
worst-case latency.

**Q: What packets does a raw socket receive?** (NK Securities, Mansard)
Depends on the flavor. `AF_INET`/`SOCK_RAW` with a protocol number delivers IP packets of *that
protocol* including the IP header (e.g. ICMP for ping) — not TCP/UDP that the kernel stack owns.
`AF_PACKET`/`SOCK_RAW` (Linux) delivers *entire link-layer frames* including the Ethernet header, and in
**promiscuous mode** every frame on the wire — this is what libpcap/tcpdump use. Both need `CAP_NET_RAW`.

**Q: If you can get raw packets with a raw socket, why does HFT use DPDK/Onload?** (Optiver, Graviton)
Because raw sockets still go through the kernel — syscalls, the network stack, copies, and interrupts,
all sources of unpredictable microsecond latency. Kernel bypass (DPDK, Solarflare Onload, Mellanox VMA)
maps the NIC into user space and **busy-polls** it, so packets DMA straight into user memory with no
syscall, no kernel stack, no interrupt. Same "raw packets," but for latency, not inspection.

**Q: Why is a general-purpose allocator unsuitable for the hot path, and what do you use instead?**
(Jump Trading, Quadeye)
Because its latency is *variable and unbounded on the tail*: it can take a lock (arena contention), hit
a cold path that calls `sbrk`/`mmap` (syscall + page faults), and its cost depends on current
fragmentation. HFT replaces it with pre-allocated **pools/arenas** ([`../guide-os-net`](../../guide-os-net/06-custom-memory-pools.md))
built and page-faulted at startup — O(1), lock-free (single-threaded per core), zero fragmentation, no
syscalls during trading. `jemalloc`/`tcmalloc` are a good default *off* the critical path.

**Q: What's the difference between internal and external fragmentation?** (IMC, AlphaGrep)
Internal: waste *inside* an allocation because sizes are rounded up (ask for 40 B, get a 48-B chunk).
External: enough total free memory, but split into pieces too small for the next request, so it can't be
used — waste *between* allocations. Size-class allocators trade a bit of internal fragmentation to
nearly eliminate external fragmentation; coalescing fights external fragmentation in bin-based
allocators.

---

## Key takeaways

- `malloc` is a **user-space** allocator that owns a pool of memory from `brk` (small, contiguous, not
  easily returned) and `mmap` (large ≥128 KB, independently returnable); syscalls happen only when the
  pool grows — the unpredictable case HFT avoids.
- Each allocation is a **chunk** with a size+flags header *before* your pointer (so `free` knows the
  size); free chunks live in **bins** and store their own linked-list pointers; boundary tags enable
  **coalescing**.
- Scalable allocators (**tcmalloc/jemalloc**) use **per-thread caches** by size class → lock-free,
  constant-time fast path → smaller tail latency; still not deterministic enough for the true hot path.
- `new` = `operator new` (≈ `malloc`) + constructor; `delete` = destructor + `operator delete`;
  **placement new** builds objects in memory you already own — the bridge to pools.
- `read`/`write` traverse VFS → filesystem → **page cache**; misses use **DMA** + **interrupt** +
  a blocking context switch; `write` only dirties the cache (durability needs `fsync`). Every **copy**
  and **syscall** is latency — `O_DIRECT`, `mmap`, and zero-copy trim them.
- **Raw sockets:** `AF_INET`/`SOCK_RAW` = IP packets of a matched protocol with the IP header;
  `AF_PACKET`/`SOCK_RAW` = full link-layer frames (all frames in promiscuous mode, à la tcpdump). They
  go *through* the kernel — the opposite of HFT kernel bypass.

**Next:** back to the [index](00-index.md) — you've reached the last module. For the language-side view
of memory and concurrency, see the C++ guide's [Module 15 (cache/memory)](../cpp-guide/15-memory-cache.md)
and [Module 16 (atomics/lock-free)](../cpp-guide/16-atomics-lockfree.md); for the HFT-tuning layer (custom
pools, isolcpus/NUMA, kernel-bypass networking) see [`../guide-os-net`](../../guide-os-net/00-index.md).
