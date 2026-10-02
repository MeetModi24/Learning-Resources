# CAOS Guide — Index

Computer Architecture & Operating Systems for HFT / low-latency C++ interviews. See
[README.md](README.md) for how to read this and how it relates to the C++ guide. Read top-to-bottom
if you're new; jump by topic if you're revising.

## Part I — Computer Architecture

1. [CA foundations & the ISA](01-ca-foundations-isa.md) — von Neumann, registers, fetch-decode-execute, CISC vs RISC, CPI
2. [Pipelining](02-pipelining.md) — stages, hazards, forwarding, register renaming, reservation stations, ROB/retirement, load-store queues, SIMD
3. [Branch prediction](03-branch-prediction.md) — predictors, BTB, branch-miss penalty, branchless code and SIMD masks
4. [Caches & the memory hierarchy](04-caches-memory-hierarchy.md) — L1/L2/L3, lines, associativity, the 3 C's, working sets, layout, tiling and prefetch
5. [Cache addressing & coherency](05-cache-addressing-coherency.md) — VIVT/VIPT/PIPT, MESI, false sharing, NUMA placement and locality
6. [Virtual memory](06-virtual-memory.md) — MMU, TLB, page-table walk, page faults, demand paging, copy-on-write

## Part II — Operating Systems

7. [Privilege & the OS role](07-privilege-and-os-role.md) — kernel/user mode, the mode bit, dual-mode, what an OS is
8. [System calls](08-system-calls.md) — how a syscall is implemented, the trap, the syscall table, cost
9. [Interrupts & exceptions](09-interrupts-and-exceptions.md) — interrupt flow, IDT/ISR, interrupt vs syscall
10. [Processes & context switching](10-processes-context-switching.md) — PCB, states, fork/exec/COW, context-switch mechanics
11. [Threads & synchronization](11-threads-and-synchronization.md) — join/detach, mutex/spinlock/futex, condition vars, deadlock
12. [Allocators & I/O](12-allocators-and-io.md) — malloc/new/free internals, the I/O path, raw sockets
13. [Scheduling & real-time](13-scheduling-and-realtime.md) — CFS/vruntime, SCHED_FIFO/RR/DEADLINE, nice/priority, RM/EDF, affinity, CPU isolation
14. [Deadlocks, starvation & livelock](14-deadlocks-starvation-livelock.md) — Coffman conditions, RAG, Banker's, prevention, priority inversion, livelock/backoff
15. [IPC fundamentals](15-ipc-fundamentals.md) — pipes/FIFOs, message queues, local/network sockets, shared memory, mapped files, zero-copy, SPSC handoff

## The through-line

> A program is bytes in memory that the CPU fetches, decodes, and executes through a **pipeline**,
> guessing branches ahead of time (**branch prediction**) and hitting **caches** and the **TLB** so
> it rarely waits on DRAM. Addresses are virtual, translated by the **MMU** through page tables
> (**virtual memory**). The **OS**, running in a privileged **kernel mode** the hardware enforces,
> multiplexes that one CPU and that one physical memory across many **processes** and **threads**,
> switching between them on **interrupts**, entering the kernel on **system calls**, and protecting
> them from each other. Those isolated processes exchange data through kernel transports or
> deliberately shared pages (**IPC**). HFT is the art of making every one of those steps predictable
> and fast.
