# Module 07 — Privilege & the role of the OS

Before you can reason about system calls, interrupts, or context switches, you need one idea straight:
the CPU runs in one of (at least) two **privilege levels**, and the operating system's power comes
entirely from the hardware enforcing that boundary. This module explains what an OS actually *is*,
what "kernel mode vs user mode" means at the silicon level, and the interview-favourite question —
*why can't a user process just flip the bit and promote itself to kernel mode?*

---

## 1. What an operating system actually is

Strip away the marketing and an OS is three things at once:

1. **A resource manager (an arbitrator).** There is one CPU (or a handful of cores), one block of
   physical RAM, one NIC, one disk — and dozens of programs that all want them *now*. The OS decides
   who gets the CPU (the scheduler), who gets which physical memory frames (the memory manager), and
   in what order I/O happens. It multiplexes scarce hardware across many programs and stops them from
   trampling each other.

2. **An abstraction layer (a virtual-machine provider).** Nobody wants to write to raw disk sectors
   or poke NIC registers. The OS hands each program clean, portable abstractions: a **process** (a
   virtual CPU with a private address space), a **file** (a byte stream over messy block storage), a
   **socket** (a reliable-ish pipe over packets), an **address space** (a private, contiguous-looking
   memory even though physical RAM is fragmented). Your code targets these abstractions; the OS maps
   them onto real hardware.

3. **A protection boundary (a referee).** Because programs are mutually distrustful (and buggy), the
   OS — backed by hardware — isolates them: process A cannot read process B's memory, cannot hog the
   CPU forever, cannot talk to the disk directly. This is the part that *requires* privilege levels,
   and it's what the rest of this module is about.

The **kernel** is the core of the OS — the always-resident code that runs in the privileged mode and
implements all three roles. "User programs" (your trading engine, bash, Chrome) run *outside* the
kernel, in the unprivileged mode, and must **ask** the kernel (via system calls, Module 08) whenever
they need something privileged.

```
        ┌──────────────────────────────────────────────┐
        │              USER SPACE (unprivileged)         │
        │   bash   trading-engine   python   nginx  ...  │
        └───────────────┬────────────────────────────────┘
                        │  system-call interface (the ONLY door down)
        ┌───────────────▼────────────────────────────────┐
        │                 KERNEL (privileged)              │
        │  scheduler │ memory mgr │ VFS │ net stack │ drivers│
        └───────────────┬─────────────────────────────────┘
                        │  privileged instructions, MMIO, DMA
        ┌───────────────▼─────────────────────────────────┐
        │   HARDWARE: CPU cores, RAM, MMU, NIC, disk, timer │
        └───────────────────────────────────────────────────┘
```

Everything crosses that middle line through controlled doorways. There is no other way down.

---

## 2. Dual-mode operation: the privilege bit

To make the protection boundary *unforgeable*, protection can't live in software alone — a buggy or
malicious program could just skip the check. So it lives in the **CPU hardware**: the processor
always runs in one of two modes, tracked by a bit (really a small field) in a control register.

- **User mode** (unprivileged): the mode ordinary programs run in. Many instructions are forbidden.
- **Kernel mode** (a.k.a. supervisor / privileged mode): the mode the kernel runs in. Everything is
  allowed.

On **x86-64** this is the **CPL (Current Privilege Level)**, a 2-bit field in the `CS` segment
register giving four "rings" 0–3. In practice only two are used: **ring 0 = kernel**, **ring 3 =
user** (rings 1–2 are historical). On **AArch64** the same idea is called **Exception Levels**:
**EL0 = user**, **EL1 = kernel/OS**, with EL2 (hypervisor) and EL3 (secure monitor) above.

```
x86-64 rings                      AArch64 exception levels
┌───────────────┐                 EL3  secure monitor / firmware
│ ring 0 kernel │  ← the OS       EL2  hypervisor
│ ring 1  (—)   │                 EL1  kernel / OS      ← the OS
│ ring 2  (—)   │                 EL0  user apps        ← your program
│ ring 3  user  │  ← your program
└───────────────┘
```

The CPU checks this field on *every* instruction and *every* memory access. If a user-mode program
tries something privileged, the hardware refuses and raises a fault (a **#GP general-protection
fault** on x86) which traps into the kernel — the kernel then typically kills the offender with
`SIGSEGV`/`SIGILL`.

---

## 3. What counts as "privileged"?

Privileged instructions and operations are exactly the ones that, if a random program could do them,
would let it break isolation or take over the machine:

- **Direct device I/O** — the `in`/`out` port instructions, and access to memory-mapped device
  registers (MMIO). Otherwise any program could reprogram the disk controller.
- **Modifying the MMU / page tables** — loading `CR3` (the page-table base) on x86. If a user could
  repoint the page tables, it could map *any* physical memory into its address space and read every
  other process's secrets.
- **Halting the CPU** (`HLT`), and **disabling/enabling interrupts** (`CLI`/`STI`). A user program
  that could disable interrupts could stop the scheduler's timer and hog the CPU forever, defeating
  preemptive multitasking.
- **Loading descriptor tables** (`LGDT`, `LIDT`, `LTR`) — the tables that define memory segments and
  interrupt handlers.
- **Reading/writing many control and model-specific registers** (`CR0`, `CR4`, most `MSR`s via
  `wrmsr`).
- **Changing the privilege level itself.** This is the crux of the next section.

Everything else — arithmetic, loads/stores to *your own* mapped memory, ordinary jumps and calls —
runs fine in user mode with no kernel involvement. That's why user code is fast: 99.9% of it never
touches the kernel at all.

---

## 4. Why a user process can *not* set the mode bit itself

This is a classic interview question, and the answer reveals the whole design.

Naively you'd think: "kernel mode is just a bit in a register — why can't my program set it and gain
root-level power?" The answer is that **there is no user-accessible instruction that only sets the
privilege bit.** The privilege level can only change through hardware-defined **controlled transition
points**, and each of those does *two things atomically and inseparably*:

1. **Raises the privilege level** (user → kernel), **and simultaneously**
2. **Redirects execution to a fixed, kernel-chosen address** — a handler whose location the *kernel*
   installed ahead of time (in the IDT, or in the `LSTAR` MSR for the `syscall` instruction).

You never get to "be in kernel mode running your own code." The instant you're in kernel mode, the
program counter is already inside the kernel's entry stub — code you don't control. The kernel then
validates what you asked for and does it on your behalf, or rejects you.

```
   USER MODE                              KERNEL MODE
   ─────────                              ───────────
   ...your code...
   syscall / int / (interrupt)  ──────►   [CPU atomically: CPL 3→0
                                            AND  RIP := kernel entry point]
                                                  │
                                           kernel entry stub (KERNEL's code)
                                                  │  validates, dispatches
                                           ...does privileged work...
   ...next instruction  ◄──────  sysret / iret  [CPL 0→3, RIP := saved user RIP]
```

Contrast the "attack" that the design forbids:

```
   BAD (impossible):  set CPL=0 ; jmp my_own_evil_code   ← no such instruction exists
```

The two steps are welded together in hardware. If they *weren't* — if a program could raise
privilege and then keep executing its own instructions — protection would be meaningless: any
program could promote itself and do anything. So the rule is: **the only way up is through a door the
kernel built, and walking through it drops you inside the kernel's house, not yours.** Even the
instructions that *do* change privilege as a side effect (`syscall`, `int`, `iret`, `sysret`) either
force the destination (`syscall`/`int`) or are themselves privileged / only meaningful when returning
*down* (`iret`, `sysret`).

> **One-liner for the interview:** "You can't set the mode bit because there's no instruction that
> merely sets it — privilege can only rise through a trap/syscall/interrupt, and that transition
> *simultaneously* jumps to a kernel-defined handler address. You enter kernel mode already running
> the kernel's code, never your own."

---

## 5. Putting it together: the OS as an enforced monopoly

The scheduler can preempt you because a **timer interrupt** (privileged to configure) fires
periodically and traps into the kernel — a user program can't disable it. Memory isolation holds
because only the kernel can load `CR3`, so each process is stuck inside the page tables the kernel
built for it. I/O is safe because only the kernel can touch device registers. Every one of the OS's
three roles (§1) is ultimately backed by "this instruction is privileged, and you can only reach
privileged code through a door I control." Modules 08 (system calls) and 09 (interrupts) are simply
the two kinds of doors.

---

## Common pitfalls / misconceptions

- **"Kernel mode = a separate CPU or process."** No — it's the *same* core running the *same*
  instruction stream capability, just with the privilege field flipped and the program counter inside
  kernel code. A syscall doesn't switch cores; it switches modes on the current core.
- **"root/administrator = kernel mode."** No. `root` is a *user-space* identity (uid 0) checked by
  the kernel's permission logic. A root process still runs in ring 3 / EL0 and still traps into the
  kernel for privileged work. Privilege level (hardware) and user identity (kernel policy) are
  orthogonal.
- **"Kernel code is a library I link against."** No — the kernel lives in its own address space
  region (mapped into every process's high addresses but only accessible in kernel mode). You reach
  it only via the syscall door, not a normal `call`.
- **"More rings = more security."** In practice everyone collapsed to two (0 and 3); the middle rings
  added complexity without enough benefit. Virtualization added EL2/ring -1-ish concepts instead.

---

## Quiz — tough problems

**Q1.** A user-space program executes `cli` (clear interrupt flag, meant to disable interrupts). What
happens?
**Answer:** On x86 `cli` is privileged when the program's IOPL doesn't permit it (normal case), so the
CPU raises a **#GP fault**, which traps into the kernel; the kernel typically delivers `SIGSEGV` and
the process dies. Crucially the interrupt flag is *not* changed — the whole point is that a user
program can't disable the timer that lets the scheduler preempt it.

**Q2.** Why does the kernel need to run in the *same* address space region as every user process
(mapped high) rather than a totally separate space?
**Answer:** So the user→kernel transition on a syscall/interrupt doesn't require switching page tables
(reloading `CR3`), which would flush the TLB and be slow. The kernel is mapped into every process's
upper half but marked kernel-only, so the trap just flips the privilege bit and jumps — no
address-space switch. (Meltdown forced *page-table isolation*/KPTI which partially undid this for
security, at a latency cost — a good follow-up.)

**Q3.** True/false: entering kernel mode always means switching to a different process.
**Answer:** False. A syscall or interrupt runs kernel code *in the context of the currently running
process* (using a per-task kernel stack) without changing which process is "current." A context
switch to another process is a *separate*, later decision the scheduler may make (Module 10).

**Q4.** If privilege is enforced by a bit in a register, and registers are just memory-ish state, why
can't malware overwrite that register?
**Answer:** Because the instructions that write it (`mov` to `CS`/control regs, `wrmsr`, etc.) are
themselves privileged or constrained — writing them in user mode faults. The bit is protected by the
same mechanism it enforces; the bootstrap is that the CPU starts in kernel mode at power-on and the
kernel sets up the doors before ever dropping to user mode.

**Q5.** Why are rings 1 and 2 essentially unused on modern OSes?
**Answer:** The classic design imagined device drivers in ring 1–2 (some privilege but not full), but
managing multiple gradations added complexity, and page-based protection (not segment/ring-based)
became the real isolation mechanism. So OSes use just ring 0 (kernel, drivers included) and ring 3
(everything else). Hypervisors later reused the extra levels conceptually (VMX root mode, EL2).

**Q6.** A researcher claims their user-space library "runs in kernel mode for speed." Plausible?
**Answer:** Not as stated. User-space code cannot run in kernel mode. What they likely mean is either
(a) a kernel module they wrote, or (b) kernel-bypass techniques (DPDK/Onload) that map device memory
into *user* space so the fast path avoids the kernel entirely — still ring 3, just talking to the NIC
directly. Both are "fast because they avoid the transition," not "user code promoted to ring 0."

**Q7.** On AArch64, an app at EL0 wants to change the page tables. What must happen?
**Answer:** It must trap to EL1 via an `svc` (supervisor call) — the kernel at EL1 owns the translation
tables (`TTBR0/1_EL1`). EL0 has no instruction to write those registers; attempting one traps. Same
principle as x86: page-table control is privileged and reachable only through the syscall door.

**Q8.** Why is *preemptive* multitasking impossible without hardware privilege support?
**Answer:** Preemption relies on a periodic **timer interrupt** that yanks control back into the
kernel regardless of what the user thread is doing. Configuring that timer and handling its interrupt
are privileged; disabling interrupts is privileged. If user code could disable interrupts or reprogram
the timer, it could refuse to be preempted — cooperative multitasking at best. Hardware privilege is
what makes "the OS is always in charge" true.

---

## Indian HFT interview questions

**Q1 (Graviton / Quadeye — fundamentals).** *What is the difference between user mode and kernel mode,
and why do we need two modes at all?*
**Model answer:** Two hardware privilege levels: user mode forbids privileged instructions (I/O, MMU
config, interrupt control, halting), kernel mode allows everything. We need the split for **protection
and multiplexing** — so a buggy/malicious program can't touch other processes' memory, monopolise the
CPU, or corrupt devices. The CPU checks the privilege field on every instruction and access; violations
fault into the kernel. Without it there's no isolation and no enforceable scheduler.

**Q2 (Tower Research, Gurgaon — the classic).** *Why can't a user process set the mode bit and enter
kernel mode on its own?*
**Model answer:** There is no instruction that merely sets privilege. Privilege can only rise through a
controlled transition — `syscall`, a software interrupt, or a hardware interrupt/fault — and that
transition atomically raises the level *and* redirects the program counter to a kernel-installed
handler address. So the moment you're privileged, you're already executing the kernel's code, not your
own. If you could raise privilege and keep running your own instructions, isolation would collapse.

**Q3 (Optiver — practical latency).** *Given that entering the kernel is expensive, how does that shape
an HFT hot path?*
**Model answer:** Every user→kernel transition (syscall, page fault, interrupt) costs a mode switch plus
cache/TLB pollution and is a jitter source. So we keep the hot path entirely in user mode: kernel-bypass
networking (map the NIC into user space, busy-poll, no `recv` syscall), pre-allocated + `mlock`ed memory
(no page-fault traps, no `mmap`), pinned isolated cores with interrupts routed elsewhere (no timer
preemption or device IRQs), and no logging syscalls inline. The whole discipline is "don't cross the
privilege boundary while trading."

**Q4 (IMC — depth).** *When a user program dereferences a null pointer, walk me through what the CPU and
OS do.*
**Model answer:** The load/store to address 0 goes to the MMU; there's no valid mapping, so the CPU
raises a **page fault** (a synchronous exception) which traps to ring 0 and jumps through the IDT to the
kernel's page-fault handler — an involuntary user→kernel transition. The handler inspects the faulting
address and access type, sees it's an illegal access (not a demand-paging or COW case), and delivers
`SIGSEGV` to the process; absent a handler, the process is terminated. Note the transition was *not*
requested by the program — it's a fault, but it uses the same door mechanism as a syscall.

**Q5 (AlphaGrep — conceptual clarity).** *Is `root` the same as kernel mode?*
**Model answer:** No. Kernel vs user mode is a **hardware** privilege level. `root` (uid 0) is a
**software** identity the kernel uses for permission checks. A root process still runs in ring 3 and
still traps into the kernel for privileged operations; it just passes the kernel's permission checks more
often. You can be root and never in kernel mode; the kernel is in kernel mode regardless of which user's
process it's servicing.

**Q6 (HRT — systems reasoning).** *Why is the kernel mapped into every process's address space instead
of living entirely separately?*
**Model answer:** To make syscalls and interrupts cheap: the kernel occupies the high half of every
process's virtual address space (marked kernel-only via page permissions), so a trap into the kernel
flips the privilege bit and jumps to an address that's *already mapped* — no `CR3` reload, no full TLB
flush. If the kernel lived in a separate address space, every syscall would pay an address-space switch.
(Follow-up: Meltdown forced KPTI, which *does* switch to a minimal kernel page table on entry, reintroducing
some of that cost — a real latency regression HFT shops cared about.)

**Q7 (Squarepoint — precision).** *What exactly is "privileged" about an instruction like `cli` or a
write to `CR3`?*
**Model answer:** These control machine-wide invariants: `cli` disables interrupts (could stop
preemption and interrupt-driven I/O); `CR3` is the page-table base (could remap physical memory and
break isolation). The CPU only executes them at CPL 0 (or per IOPL for `cli`); attempting them in user
mode raises `#GP`. They're privileged precisely because letting user code run them would let it escape
the OS's control over CPU time and memory.

---

## Key takeaways

- An OS is simultaneously a **resource manager**, an **abstraction layer**, and a **protection
  boundary**; the last one is what requires hardware privilege.
- The CPU always runs in **user** or **kernel** mode, tracked by a hardware field (x86 CPL rings 0/3,
  AArch64 EL0/EL1); it's checked on every instruction and memory access.
- **Privileged operations** — device I/O, MMU/`CR3` changes, interrupt control, `HLT`, descriptor-table
  loads — are exactly those that could break isolation or CPU control if user code could do them.
- A user process **cannot set the privilege bit** because no instruction merely sets it: privilege only
  rises through a trap/syscall/interrupt that *atomically* raises the level and jumps to a
  kernel-defined handler — you're always running the kernel's code the instant you're privileged.
- `root` (identity) ≠ kernel mode (hardware privilege); they're orthogonal.
- Preemptive multitasking, memory isolation, and safe I/O all reduce to "privileged instruction +
  kernel-controlled entry door" — the two doors being system calls (Module 08) and interrupts (Module 09).

**Next:** [08 — System calls](08-system-calls.md)
