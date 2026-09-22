# Computer Architecture & Operating Systems — HFT Interview Guide (CAOS)

A theory-first, interview-focused guide to the two systems layers under every low-latency C++
program: **how the machine actually executes** (computer architecture) and **how the OS manages it**
(operating systems). This is the layer HFT interviews (Tower Research, Optiver, Graviton, Quadeye,
AlphaGrep, IMC, Jump, HRT, Da Vinci, Squarepoint, NK Securities, Mansard, Qube, Millennium) grill
hardest after C++ itself — because latency is decided *here*, not in your algorithm's Big-O.

The goal is **understanding, not memorization**: every topic is built from zero for a beginner,
then climbed to the depth an interviewer probes. Nothing is "jotted down without context" — each
mechanism is explained (why it exists, how it works in memory/hardware, what it costs), with ASCII
diagrams of what's happening at the hardware/kernel level, tough quizzes with answers, and the exact
India-desk interview questions that map to it.

> **Relationship to the other guides.** The [`../guide`](../cpp-guide/) C++ guide's Module 15
> (cache/memory) and Module 16 (atomics/lock-free) touch this material from the *language* side;
> [`../guide-os-net`](../../guide-os-net/) covers the HFT-*tuning* angle (isolcpus, NUMA, kernel bypass,
> networking). This guide is the **foundational systems layer** they both stand on — read it first if
> the hardware/kernel mechanisms feel like magic.

## How to read this

- **If you're new:** read top to bottom. Part I (architecture) builds the mental model of the machine
  — pipeline → branch prediction → caches → virtual memory — so that Part II (OS) has something
  concrete to manage. Almost every "why is this slow?" answer bottoms out in Part I.
- **If you're revising for an interview:** each module is self-contained and ends with a **quiz** and
  an **interview questions** section (with model answers). Jump to the topic you're weak on.
- **The through-line:** *A program is bytes in memory that the CPU fetches, decodes, and executes
  through a pipeline, hitting caches and the TLB on the way; the OS, running in a privileged mode,
  multiplexes that CPU and that memory across processes using interrupts, system calls, and context
  switches.* Every module is one piece of that sentence.

## Table of contents

### Part I — Computer Architecture (how the machine runs your code)

| # | File | Topics |
|---|------|--------|
| 01 | [ca-foundations-isa.md](01-ca-foundations-isa.md) | von Neumann model, ISA, registers, the fetch-decode-execute cycle, **CISC vs RISC**, CPI, why RISC keeps up despite larger code |
| 02 | [pipelining.md](02-pipelining.md) | The classic 5-stage pipeline, throughput vs latency, **hazards** (structural/data/control), forwarding, stalls, superscalar & out-of-order execution, CPI math |
| 03 | [branch-prediction.md](03-branch-prediction.md) | Why branches stall the pipeline, static vs **dynamic prediction**, 2-bit counters, BTB, global/local history, the **branch-miss penalty**, branch-miss vs cache-miss, branchless code |
| 04 | [caches-memory-hierarchy.md](04-caches-memory-hierarchy.md) | SRAM/DRAM, the memory wall, **L1/L2/L3** (and how they differ), cache lines, associativity, replacement, write-back vs write-through, the 3 C's of misses |
| 05 | [cache-addressing-coherency.md](05-cache-addressing-coherency.md) | VIVT / **VIPT / PIPT** caches, aliasing & homonyms, why VIPT is the sweet spot, **MESI coherency**, and **do we flush caches on a context switch?** |
| 06 | [virtual-memory.md](06-virtual-memory.md) | The **full software→hardware read/write flow** (MMU, TLB, page-table walk, cache, DRAM), multi-level page tables, page faults, demand paging, **copy-on-write** |

### Part II — Operating Systems (how the OS manages the machine)

| # | File | Topics |
|---|------|--------|
| 07 | [privilege-and-os-role.md](07-privilege-and-os-role.md) | Kernel vs user mode, the **mode bit** and why a user process can't set it, dual-mode operation, traps into the kernel, what an OS actually is (the "intro slides") |
| 08 | [system-calls.md](08-system-calls.md) | **How a syscall is implemented** end-to-end (`syscall`/`sysenter`/`int 0x80`), the trap, the syscall table, argument passing, libc wrappers, and syscall cost |
| 09 | [interrupts-and-exceptions.md](09-interrupts-and-exceptions.md) | The **interrupt flow** (IRQ → IDT → ISR → EOI), interrupts vs traps vs faults vs aborts, and the **difference between an interrupt and a syscall** at the implementation level |
| 10 | [processes-context-switching.md](10-processes-context-switching.md) | The PCB, process states, `fork`/`exec` (+ COW), the scheduler & dispatcher, and **exactly how a context switch happens** (register save/restore, TLB, cache effects) |
| 11 | [threads-and-synchronization.md](11-threads-and-synchronization.md) | Threads vs processes, the thread lifecycle (`join`/`detach` and friends), **how a mutex actually works** (spinlock → futex → kernel), condition variables, deadlock, atomics |
| 12 | [allocators-and-io.md](12-allocators-and-io.md) | **`malloc`/`new`/`free` internals** (`brk`/`mmap`, free lists, bins, arenas, tcmalloc/jemalloc), the OS **I/O path** (VFS → page cache → DMA), and **raw sockets** |

## Notation & conventions

- **Diagrams** are ASCII and read top-to-bottom / left-to-right in time or address order; a `───▶`
  arrow means "points to" or "flows to". Latencies are given in CPU **cycles** and rough **ns**
  (assume a ~3 GHz core, so ~1 cycle ≈ 0.3 ns) — treat them as orders of magnitude, not exact.
- **Answers** to quiz problems appear immediately after each question on a `**Answer:**` line.
- Code is C/C++ or pseudo-assembly (x86-64 / AArch64 where the ISA matters); it's illustrative but
  written to be correct.

**Start here:** [01 — CA foundations & the ISA](01-ca-foundations-isa.md).
