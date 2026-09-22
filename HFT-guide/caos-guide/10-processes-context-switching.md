# Module 10 — Processes, scheduling & context switching

A **process** is the OS's unit of "a running program." Understanding what the kernel stores per
process (the PCB), how it moves a process between running and waiting, and *exactly* what happens
when the CPU is taken from one process and handed to another (a **context switch**) is the difference
between "I've heard of context switches" and "I know why they cost microseconds." That microsecond
is why HFT systems pin threads to cores and never let the scheduler touch the hot path.

---

## 1. What a process actually is

A program on disk is just bytes — an ELF file with code and initial data. A **process** is that
program *in execution*, together with all the state the OS keeps on its behalf:

- an **address space** — the virtual memory the process sees (code, data, heap, stack, mmap'd
  regions), backed by a page table (Module 06);
- one or more **threads** — the actual schedulable execution contexts (registers + stack);
- **open files** — the file-descriptor table (fd 0/1/2 = stdin/stdout/stderr, plus sockets, files);
- **identity & accounting** — PID, parent PID, user/group IDs, priority, CPU time used, signal
  handlers.

The key mental model: **a process is an address space + a set of resources; a thread is what runs.**
A single-threaded process has exactly one thread. Isolation between processes is enforced by the
hardware — each has its own page table, so process A literally cannot name a byte in process B's
memory (there's no virtual address that translates into B's frames). Threads *inside* one process
share the address space; that's the whole point of Module 11.

```
Process (PID 4021)
┌───────────────────────────────────────────────┐
│  Address space (own page table / CR3)          │
│   ┌──────┬──────┬───────┬─────────┬──────────┐  │
│   │ code │ data │ heap ▲│  ...mmap│ ▼ stack  │  │
│   └──────┴──────┴───────┴─────────┴──────────┘  │
│  fd table: 0→tty 1→tty 2→tty 3→socket ...       │
│  threads: [T1 regs+stack] [T2 regs+stack] ...   │
│  PID 4021, PPID 4000, uid, priority, signals    │
└───────────────────────────────────────────────┘
```

---

## 2. The PCB (Linux: `task_struct`)

The **Process Control Block** is the kernel's per-process record. It's the single data structure the
kernel consults to know everything about a process, and — critically — it's where a process's CPU
state is **saved** when it's not running. On Linux the PCB is `struct task_struct` (a large struct in
`sched.h`); conceptually it holds:

| Field group | What it stores | Why it matters |
|---|---|---|
| **Saved CPU context** | general-purpose registers, program counter (RIP), stack pointer (RSP), flags (RFLAGS) | Restored to resume the process exactly where it stopped |
| **Memory** | pointer to the page-table base (loaded into **CR3** on x86) | Switching this *is* switching address spaces |
| **Kernel stack ptr** | this thread's kernel-mode stack | Where the kernel runs when this thread traps in |
| **Scheduling** | state, priority/nice, scheduling class, CPU time, last CPU | The scheduler's inputs |
| **Identity** | PID, PPID, uid/gid, credentials | Ownership & permissions |
| **Resources** | fd table, signal handlers, namespaces, cgroup | The rest of the process's world |

> Nuance for interviews: on Linux there is no separate "thread control block" — each *thread* is a
> `task_struct` too. Threads of one process are `task_struct`s that **share** the memory descriptor
> (`mm_struct`), fd table, and signal handlers, but have their own saved registers and kernel stack.
> This is the "1:1 threading model" (Module 11). So "PCB per process" is really "one `task_struct`
> per schedulable thread, some of which share an address space."

---

## 3. Process states & the state machine

At any instant a process (thread) is in one of a few states. The scheduler is the machine that moves
it between them:

```
                 admitted            dispatch
    ┌─────┐  ───────────────▶ ┌───────┐ ─────────────▶ ┌─────────┐
    │ NEW │                   │ READY │                │ RUNNING │
    └─────┘                   └───────┘ ◀───────────── └─────────┘
                                  ▲    timer/preempt        │
                          wakeup  │                         │ exit
                    (I/O done,    │        blocks on        ▼
                     lock free)   │       I/O / lock    ┌────────────┐
                              ┌─────────┐ ◀──────────── │ TERMINATED │
                              │ WAITING │               │ (zombie)   │
                              └─────────┘               └────────────┘
```

- **NEW** — being created (resources allocated, not yet runnable).
- **READY (runnable)** — able to run, waiting for a CPU. Sits in a run queue.
- **RUNNING** — currently executing on a CPU. On an N-core machine at most N processes are RUNNING.
- **WAITING (blocked)** — cannot proceed until an event: disk I/O completes, a lock is released, data
  arrives on a socket, a `sleep` expires. **A blocked process consumes zero CPU** — but it costs a
  context switch to put it to sleep and another to wake it.
- **TERMINATED / zombie** — finished executing; its exit status lingers until the parent `wait`s.

The two transitions that matter for latency: **RUNNING → READY** (the kernel *preempted* you — your
time slice expired or a higher-priority task woke) and **RUNNING → WAITING** (you *blocked* yourself
on a syscall). Both trigger a context switch. HFT hot threads are engineered to do neither: they busy-
poll (never block on I/O) and run on an isolated core with nothing else to preempt them.

---

## 4. Creating and replacing processes: `fork`, `exec`, `wait`

Unix builds new processes with a deliberately odd split:

**`fork()`** creates a near-exact **copy** of the calling process. It's the famous call that
"returns twice": once in the parent (returning the child's PID) and once in the child (returning 0).
The child gets a *copy* of the address space, fd table, and registers.

```c
pid_t pid = fork();
if (pid == 0)      { /* child:  pid==0 */ }
else if (pid > 0)  { /* parent: pid==child's PID */ }
else               { /* fork failed */ }
```

Copying the entire address space eagerly would be absurdly expensive (imagine `fork`-ing a process
using 8 GB). It isn't copied — it's **copy-on-write (COW)**, exactly the mechanism from Module 06:
parent and child share the same physical frames, all marked **read-only** in both page tables. Reads
are free. The first *write* to a shared page traps into the kernel (a protection fault), which
allocates a fresh frame, copies that one page, and remaps it writable for the writer. So `fork` is
cheap and pages are duplicated lazily, only as they're actually modified.

**`exec()`** (the `execve` family) does the opposite: it **replaces** the current process image with a
new program — new code, data, heap, stack — while keeping the same PID and (by default) open fds. It
does *not* return on success (there's nothing to return to; the old image is gone).

The idiom is `fork` then `exec`: fork to get a new process, exec in the child to become the new
program. This is how a shell launches commands. Because COW makes the fork cheap and exec throws the
copied pages away anyway, the pair is efficient — though Linux also offers `posix_spawn`/`vfork` and
`clone` for cases where you want to skip even the COW setup.

**`wait()`/`waitpid()`** lets a parent collect a terminated child's exit status. Until it does, the
child stays a **zombie** (dead but its PCB lingers to hold the exit code). If the parent dies first,
the child is **orphaned** and re-parented to `init`/`systemd` (PID 1), which reaps it. Zombie leaks
(a parent that never `wait`s) are a real bug: PCBs and PIDs are finite.

---

## 5. Scheduler vs dispatcher

These get conflated; interviewers separate them:

- The **scheduler** decides *which* READY process runs next — the policy. Linux's default is **CFS**
  (Completely Fair Scheduler), which approximates giving each runnable task a fair share of CPU time
  using a red-black tree keyed by virtual runtime; real-time classes (`SCHED_FIFO`/`SCHED_RR`) sit
  above it for latency-sensitive threads.
- The **dispatcher** is the *mechanism* that actually hands the CPU to the chosen process — it
  performs the context switch: saves the old state, loads the new, switches to user mode, and jumps to
  the new process's PC. **Dispatch latency** is the time this takes.

One-liner: *the scheduler picks, the dispatcher does.*

---

## 6. Exactly how a context switch happens

This is the money question. A **context switch** is the act of saving one process's (thread's) CPU
state and restoring another's so the CPU can run someone else. Step by step, triggered by either a
**timer interrupt** (preemption — your slice is up) or a **blocking syscall** (you asked for I/O and
must wait):

```
 Process A running in user mode
        │
   (1)  │ timer interrupt fires  OR  A calls a blocking syscall
        ▼
   [ CPU switches to kernel mode, jumps to handler, uses A's kernel stack ]   ← Module 08/09
        │
   (2)  │ SAVE A's context: push GP registers, RIP, RSP, RFLAGS into A's
        │                    kernel stack / task_struct
        ▼
   (3)  │ SCHEDULER runs: pick next runnable task → chooses B
        ▼
   (4)  │ SWITCH ADDRESS SPACE: load B's page-table base into CR3
        │   └─▶ this INVALIDATES the TLB (unless entries are PCID/ASID-tagged)
        ▼
   (5)  │ RESTORE B's context: pop B's saved registers, RSP, RFLAGS, RIP
        ▼
   (6)  │ return to USER mode (iret/sysret) at B's saved PC
        ▼
 Process B running in user mode  (resumes exactly where it left off)
```

The visible cost is steps (2) and (5): saving and restoring a few dozen registers — a few hundred
nanoseconds. But the *real* cost is hidden:

- **Step (4) — the TLB flush.** Loading CR3 switches the page table, so every cached virtual→physical
  translation is now potentially wrong and gets invalidated. Process B starts with a **cold TLB** and
  eats a burst of page-walk misses (Module 06) as it re-populates it. Modern CPUs mitigate this with
  **PCID/ASID** tags (each address space gets an ID stamped on TLB entries, so entries survive the
  switch and don't need flushing) — but the effect is only partly hidden.
- **Cold caches.** Process B's code and data aren't in L1/L2 anymore — A evicted them, or B was away
  long enough that its lines aged out. B runs slowly for thousands of cycles while its working set is
  pulled back from L3/DRAM. This "cache pollution" is usually the biggest cost and doesn't show up in
  any single instruction — it's spread across B's first many microseconds.
- **Kernel-mode transition** itself (Module 08) and scheduler bookkeeping.

Add it up and a full process context switch costs on the order of **1–5 microseconds** of direct work
plus a tail of cache/TLB warm-up. In HFT terms that's an eternity — a tick-to-trade budget might be a
few hundred nanoseconds total. Hence: **pin the hot thread to an isolated core** (`isolcpus`, cpuset,
`sched_setaffinity`), give it a real-time priority or the whole core to itself, and never let it block
— so the scheduler simply never switches it out.

---

## 7. Process switch vs thread switch — why threads are cheaper

Switching between two **threads of the same process** skips the most expensive step: they share the
same address space, so **CR3 doesn't change and the TLB isn't flushed**. You still save/restore the
registers and stack pointer, and the caches may still be somewhat cold if the threads touch different
data — but you avoid the address-space switch entirely.

```
 Thread switch (same process):   save regs → schedule → restore regs → return   (no CR3 reload)
 Process switch (diff process):  save regs → schedule → RELOAD CR3 (TLB flush!) → restore → return
```

This is a concrete reason multithreading-within-a-process can beat multi-process designs for latency,
and why "how much does a context switch cost?" should always be answered with a question back: *thread
or process?*

---

## Common pitfalls / misconceptions

- **"A context switch is just swapping registers."** The register swap is the cheap part; the TLB
  flush and cold caches dominate, and they're invisible if you only look at the switch code.
- **"`fork` copies the whole address space."** No — copy-on-write; frames are shared read-only and
  duplicated lazily on the first write.
- **"Blocked processes waste CPU."** A blocked (WAITING) process consumes zero CPU. Busy-waiting
  (spinning) wastes CPU; blocking does not — but blocking costs two context switches (sleep + wake).
- **"Thread switch and process switch cost the same."** Thread switches within a process skip the
  CR3/TLB flush and are meaningfully cheaper.
- **"The scheduler does the context switch."** The scheduler *chooses*; the dispatcher *performs* the
  switch. Different responsibilities.
- **Zombie ≠ orphan.** A zombie is dead but not yet reaped by its parent; an orphan is alive but its
  parent died (re-parented to PID 1).

---

## Quiz — tough problems

**Q1.** A process calls `fork()`. Right after, before either process writes anything, how much
physical memory has been copied?

**Answer:** Essentially none of the *user* pages — COW shares all frames read-only. The kernel does
allocate a new `task_struct`, kernel stack, and a copy of the page tables (the page-table *structures*
themselves are copied/set up, not the pages they point to). Actual data pages copy lazily, one per
first-write fault.

---

**Q2.** `fork()` is called in a loop `for (int i = 0; i < 3; i++) fork();`. How many processes exist
at the end (including the original)?

**Answer:** 2³ = **8**. Each iteration doubles the count: after i=0 there are 2, after i=1 there are 4,
after i=2 there are 8. Every existing process (parent and children) executes the remaining iterations.

---

**Q3.** True or false: switching from thread T1 to thread T2 in the *same* process flushes the TLB.

**Answer:** **False.** Same process ⇒ same page table ⇒ CR3 is unchanged ⇒ no TLB flush. That's why
thread switches are cheaper than process switches. (Switching to a thread in a *different* process
reloads CR3 and, absent PCID tagging, flushes the TLB.)

---

**Q4.** You measure that a workload doing frequent `read()`s on a socket has terrible tail latency,
even though CPU usage is low. Give the systems explanation.

**Answer:** Each blocking `read()` that finds no data puts the thread to **WAITING** — a context
switch out, then another back in when data arrives, plus cold caches/TLB each time and the syscall
cost itself. Low CPU usage is the symptom of blocking, not health. The HFT fix is **busy-polling** the
socket (or kernel bypass, Module 12) on a pinned core: burn CPU to eliminate the context switches and
keep caches hot, trading throughput/efficiency for deterministic latency.

---

**Q5.** Why can't you make context switches "free" just by adding more registers to save/restore
faster?

**Answer:** Because the dominant cost isn't the register save/restore — it's the **TLB flush and cache
coldness** that follow an address-space switch. Those are properties of the memory hierarchy, not the
register file; no amount of faster register spilling addresses them. (PCID/ASID tagging attacks the
TLB part directly, which is the real lever.)

---

**Q6.** A parent forks 100 children and never calls `wait()`. The children all `exit()` immediately.
What's the problem?

**Answer:** 100 **zombies** accumulate — each dead child keeps its PCB and PID alive to hold its exit
status until reaped. This leaks kernel resources and PIDs. Fixes: `wait()`/`waitpid()` for them, reap
them in a `SIGCHLD` handler, or `signal(SIGCHLD, SIG_IGN)` / double-fork so they're auto-reaped.

---

**Q7.** Two designs for a market-data handler: (a) one process per symbol, (b) one process with one
thread per symbol. Which has cheaper context switches between symbol handlers, and why?

**Answer:** **(b).** Threads within one process share an address space, so switching between them skips
the CR3 reload and TLB flush. (In practice HFT would prefer *neither* — one pinned thread polling all
symbols, avoiding switches altogether — but between the two, threads win on switch cost.)

---

**Q8.** During a context switch, whose kernel stack does the saving code run on, and why does each
thread need its own kernel stack?

**Answer:** It runs on the **outgoing thread's** kernel stack (entered when the trap/interrupt raised
privilege). Each thread needs its own kernel stack because a thread can be mid-syscall (blocked in the
kernel) when switched out; its in-kernel call chain and saved state must persist independently of every
other thread's. A shared kernel stack would be clobbered the moment two threads were both in the
kernel.

---

**Q9.** `vfork()` vs `fork()`: what does `vfork` promise and why was it invented if COW already makes
`fork` cheap?

**Answer:** `vfork` creates a child that **shares** the parent's address space (no page-table copy at
all) and **suspends the parent** until the child `exec`s or `_exit`s. It was invented before COW, when
`fork` eagerly copied everything, specifically for the `fork`-then-`exec` case where copying is pure
waste. With COW, `fork`'s advantage shrank, but `vfork`/`posix_spawn` still avoid even the page-table
duplication and COW fault overhead, which can matter for very large or very frequently spawning
parents.

---

## Indian HFT interview questions

**"Walk me through exactly what happens on a context switch."**
Trigger (timer interrupt or blocking syscall) → trap into the kernel, switch to kernel mode and the
thread's kernel stack → save the outgoing thread's registers/PC/SP/flags into its `task_struct` → the
scheduler picks the next runnable thread → if it's a different process, reload CR3 (which flushes the
TLB unless PCID-tagged) → restore the incoming thread's saved state → `iret`/`sysret` back to user
mode at its PC. Then stress the hidden costs: TLB flush, cold caches, ~µs total — which is why we pin
and avoid switches on the hot path.

**"How much does a context switch cost, and what dominates?"**
Order of a microsecond of direct work, but the dominant cost is the **cache/TLB coldness** afterward,
not the register swap. Always split thread-switch (cheap, no CR3 reload) from process-switch
(expensive, TLB flush). Mention PCID/ASID as the hardware mitigation for the TLB part.

**"How is `fork()` implemented so cheaply?"**
Copy-on-write: parent and child share physical frames marked read-only; the page tables are set up but
data pages aren't copied. The first write to any page faults into the kernel, which copies just that
page and remaps it writable. So cost is proportional to *pages actually modified*, not total memory.
(Cross-reference Module 06 for the COW page-fault mechanics.)

**"Difference between the scheduler and the dispatcher?"**
Scheduler = policy (which task runs next — CFS, real-time classes). Dispatcher = mechanism (performs
the actual context switch and mode change). Scheduler picks, dispatcher does; dispatch latency is the
dispatcher's overhead.

**"Why does HFT pin threads to cores and use real-time scheduling?"**
To eliminate context switches on the latency-critical path. A pinned, isolated core (`isolcpus` +
`sched_setaffinity`, often `SCHED_FIFO`) means nothing else runs there, the scheduler never preempts
the hot thread, and its caches/TLB stay warm. Combined with busy-polling (never blocking), the thread
simply never gets switched out — deterministic, sub-microsecond behavior.

**"Process vs thread — from the kernel's point of view on Linux?"**
Both are `task_struct`s created by `clone()`. A "thread" is a `clone()` that shares `mm_struct`
(address space), fd table, and signal handlers with its creator; a "process" is a `clone()`/`fork()`
that copies them (COW for memory). There's no fundamental "thread object" — sharing flags decide.

**"What's a zombie process and how do you avoid leaking them?"**
A terminated child whose exit status hasn't been collected; its PCB lingers. Reap with
`wait`/`waitpid`, handle `SIGCHLD`, or ignore `SIGCHLD` so the kernel auto-reaps. Leaking them exhausts
PIDs/PCBs.

---

## Key takeaways

- A **process** = address space + resources; a **thread** = the schedulable execution context. On
  Linux both are `task_struct`s; threads share the `mm_struct` (address space).
- The **PCB/`task_struct`** holds saved registers, the page-table base, the kernel stack pointer,
  scheduling state, identity, and resources — everything needed to pause and resume a process.
- Process **states** (ready/running/waiting) are moved by the scheduler; RUNNING→READY (preempt) and
  RUNNING→WAITING (block) each cost a context switch. Blocked ≠ CPU-wasting; spinning is.
- `fork` = COW copy (returns twice), `exec` = replace image (doesn't return), `wait` = reap the child
  (avoid zombies). `fork`+`exec` is the standard launch idiom.
- The **scheduler picks, the dispatcher switches.** A context switch saves/restores registers, reloads
  CR3 (flushing the TLB for a process switch), and returns to user mode.
- The register swap is cheap; the **TLB flush + cold caches** dominate — ~µs total. **Thread switches
  skip the CR3/TLB flush** and are cheaper than process switches.
- HFT eliminates switches on the hot path: **pin to isolated cores, real-time priority, busy-poll,
  never block.**

**Next:** [11 — Threads & synchronization](11-threads-and-synchronization.md)
