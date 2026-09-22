# Module 09 — Interrupts & exceptions

Module 08 covered the door the program *chooses* to walk through (the syscall). This module covers the
doors it *doesn't* choose: **interrupts** (asynchronous signals from hardware — a packet arrived, a
timer fired) and **exceptions** (synchronous events the CPU raises mid-instruction — a page fault, a
divide-by-zero). Both trap into the kernel, and the interview payoff is knowing precisely how an
interrupt differs from a syscall at the implementation level.

---

## 1. The taxonomy: three ways control enters the kernel

Control leaves your user code and enters the kernel in exactly three flavours. Getting the vocabulary
straight is half the battle:

| Kind | Source | Timing | Example | Returns to |
|------|--------|--------|---------|------------|
| **(Hardware) interrupt** | External device, via interrupt controller | **Asynchronous** — unrelated to current instruction | NIC packet, timer tick, keyboard | the *interrupted* instruction |
| **Trap** (software interrupt) | The running instruction, deliberately | **Synchronous, intentional** | `syscall`/`int 0x80`, `int3` breakpoint | the *next* instruction |
| **Fault / abort** (exception) | The running instruction, involuntarily | **Synchronous, accidental** | page fault, `#GP`, divide-by-zero | the *faulting* instruction (fault) or nowhere (abort) |

The umbrella term is often "**interrupt**" (broad sense) or "**trap**"; Intel calls the whole set
**interrupts and exceptions**. The crucial axis is **asynchronous (hardware) vs synchronous
(instruction-caused)** — that's what separates a NIC interrupt from a syscall or a page fault.

- **Interrupt** = asynchronous, from *outside* the CPU core, unrelated to what you were executing.
- **Trap** = synchronous, *you asked for it* (`syscall`) — resume after.
- **Fault** = synchronous, *your instruction misbehaved*; the kernel may fix it (demand paging) and
  **re-run the same instruction**, or kill you.
- **Abort** = synchronous, unrecoverable hardware error.

---

## 2. The full hardware-interrupt flow

Say a network packet arrives. Here's the whole path, step by step:

```
 1. NIC finishes DMA-ing the packet into RAM, then raises an IRQ line.
        │
 2. Interrupt controller (local APIC / IO-APIC; old chips: 8259 PIC)
    prioritises pending IRQs and signals the target CPU core with a vector number.
        │
 3. CPU finishes the CURRENT instruction (interrupts are checked at instruction
    boundaries), then — if interrupts are enabled (IF flag set) — accepts it.
        │
 4. CPU uses the vector to index the IDT (Interrupt Descriptor Table):
        IDT[vector] → gate descriptor → handler address + privilege + stack info
        │
 5. CPU switches to kernel mode (ring 0), switches to the kernel stack
    (via TSS / IST), and pushes the interrupted context:
        SS, RSP, RFLAGS, CS, RIP  (+ error code for some exceptions)
        │
 6. Jump to the ISR (Interrupt Service Routine) at IDT[vector].
        │
 7. ISR runs. In Linux it's split:
        TOP HALF (hardirq):  minimal, fast, interrupts often disabled —
                             acknowledge the device, grab the data, schedule work.
        BOTTOM HALF (softirq / tasklet / threaded irq): the heavy lifting
                             (protocol processing) runs later with interrupts enabled,
                             so we don't stay in the critical hardirq context long.
        │
 8. ISR sends EOI (End Of Interrupt) to the APIC so it can deliver the next one.
        │
 9. iret restores the pushed context (RIP, CS, RFLAGS, RSP, SS) and returns
    to the EXACT instruction that was about to run when the interrupt hit.
```

Two design pressures explain the top-half/bottom-half split: while a hardirq handler runs, that IRQ
(often all IRQs on that core) is masked, so anything slow there delays *everything*. So the top half
does the bare minimum (ack, copy descriptor) and defers real work (TCP/IP processing) to a softirq
that runs with interrupts re-enabled. This is why "interrupt handlers can't sleep/block" — they run in
a special context with no backing process to schedule around.

**AArch64** is analogous: the GIC (Generic Interrupt Controller) delivers the IRQ, the CPU takes an
exception to EL1 using the vector table at `VBAR_EL1`, runs the handler, and returns with `eret`.

---

## 3. Why interrupts exist (vs polling)

The alternative to interrupts is **polling**: the CPU repeatedly asks "packet yet? packet yet?" That
wastes cycles when nothing's happening. Interrupts invert it — the device tells the CPU only when
there's something to do, freeing it to run other work meanwhile. It's the hardware equivalent of "don't
call us, we'll call you."

The HFT twist: for the *lowest, most predictable* latency, you flip back to **busy-polling** on a
dedicated core (kernel bypass, Module 08/12). Interrupts add latency (controller → IDT → handler
dispatch) and, worse, **jitter** — they preempt your hot thread at unpredictable moments. So HFT routes
device IRQs to *other* cores (interrupt affinity) and busy-polls the NIC on the trading core. You accept
burning a core to eliminate interrupt-driven unpredictability.

---

## 4. The headline question: interrupt vs syscall implementation

Both an interrupt and a syscall are "traps into the kernel" — both switch to kernel mode via a
kernel-controlled entry, both save context and restore it on return. Interviewers push on the
*differences*, and there are several precise ones:

```
                    SYSTEM CALL                    HARDWARE INTERRUPT
                    ───────────                    ──────────────────
 Trigger            software: the thread           external device raises an IRQ
                    executes `syscall`/`int`        (NIC, timer, disk)
 Timing             synchronous — at a known        asynchronous — arrives between
                    point the code chose            ANY two instructions, unrelated
                                                    to the running code
 Intent             voluntary ("I want I/O")        involuntary (imposed from outside)
 Entry mechanism    modern `syscall`: via MSRs      via the IDT[vector] (or GIC/VBAR)
                    (LSTAR), BYPASSES the IDT;       — always table-indexed by vector
                    `int 0x80`: via IDT
 Vector / number    syscall NUMBER in rax selects   the interrupt VECTOR (fixed per
                    the handler (sys_call_table)     device/IRQ) selects IDT entry
 Returns to         the NEXT instruction            the INTERRUPTED instruction
                    (after `syscall`)                (re-run the one that was pending)
 Context            runs in the calling process's   runs in "interrupt context" — no
                    context; may sleep/block         guaranteed current process; the
                    (it's your thread in kernel)     handler must NOT sleep/block
 Relation to code   caused BY the instruction        unrelated to the current instruction
                    stream                            stream entirely
```

The one-sentence answer that lands: **"A syscall is a synchronous, deliberate software trap — the
running thread executes an instruction to enter the kernel and resumes at the next instruction, in its
own context. A hardware interrupt is asynchronous and involuntary — an external device forces entry
between any two instructions via the IDT, the handler runs in interrupt context with no owning process
and can't block, and control resumes at the interrupted instruction. Also, modern `syscall` uses MSRs
and bypasses the IDT, whereas interrupts always go through it."**

Note that a **page fault** sits interestingly between them: it's synchronous like a syscall (caused by
your instruction) but *involuntary* like an interrupt (you didn't ask), and it re-runs the faulting
instruction after the kernel fixes the mapping — a great follow-up to test whether you really
understand the axes rather than memorised a table.

---

## 5. Where each vector lives

The IDT has 256 entries. The low ones are CPU-defined exceptions; the rest are for interrupts:

- **0–31: CPU exceptions** (fixed by Intel/AMD): `#DE` divide error (0), `#BP` breakpoint (3), `#GP`
  general protection (13), `#PF` page fault (14), etc.
- **32–255: external/software interrupts**: device IRQs mapped here by the APIC; `0x80` historically
  for the legacy syscall trap.

So "vector 14" is *always* a page fault; "vector 0x80" is the legacy syscall; a NIC's vector is assigned
dynamically. The `syscall` instruction, again, doesn't consult this table at all.

---

## Common pitfalls / misconceptions

- **"Interrupts and exceptions are the same thing."** They share the trap machinery but differ on the
  key axis: interrupts are asynchronous/external; exceptions are synchronous/instruction-caused.
- **"An interrupt handler runs in the interrupted process."** It runs on that process's core using a
  kernel stack, but it's **not** logically that process — it's *interrupt context*, borrowing the CPU.
  It can't assume anything about `current` and must not sleep.
- **"`syscall` is just `int 0x80` with a nicer name."** No — `int 0x80` goes through the IDT like an
  interrupt; `syscall` uses MSRs and skips it. Different mechanisms, big latency difference.
- **"The CPU jumps to a handler the instant an interrupt fires."** It waits for the current instruction
  to retire (and honours the interrupt-enable flag / masking); interrupts are taken at instruction
  boundaries, not mid-instruction.

---

## Quiz — tough problems

**Q1.** Classify each and say where control resumes: (a) `syscall`, (b) NIC packet arrival, (c) page
fault on a valid but not-yet-present page, (d) divide by zero.
**Answer:** (a) trap — synchronous, voluntary; resume at *next* instruction. (b) interrupt —
asynchronous; resume at the *interrupted* instruction. (c) fault — synchronous, involuntary; kernel maps
the page and **re-runs the faulting instruction**. (d) fault (`#DE`) — synchronous; typically fatal
(`SIGFPE`) unless handled.

**Q2.** Why must a hardware-interrupt top-half handler not call a blocking/sleeping function?
**Answer:** It runs in interrupt context with no backing process to schedule, often with IRQs disabled.
Sleeping would try to context-switch away from "nothing," deadlocking or hanging the core and delaying
all further interrupts. Heavy/blocking work is deferred to a softirq/threaded-irq/workqueue that runs in
a schedulable context.

**Q3.** Both a syscall and a timer interrupt end up in the kernel on the same core. Name three
implementation differences.
**Answer:** (1) Entry: `syscall` via `LSTAR` MSR bypassing the IDT; the timer via IDT[vector]. (2) Timing:
syscall is synchronous at a chosen instruction; the timer is asynchronous, arriving between arbitrary
instructions. (3) Return target: syscall resumes at the next instruction; the interrupt resumes at the
interrupted one. (Bonus: context — syscall runs in the caller's context and may block; the interrupt
runs in interrupt context and can't.)

**Q4.** A page fault is caused by your instruction, yet you didn't "ask" for it. Is it a trap or a
fault, and how does return differ from a syscall?
**Answer:** A **fault** — synchronous but involuntary. Unlike a syscall (which resumes *after* the
instruction), a fault resumes by **re-executing the faulting instruction**, now that the kernel has made
the access valid (e.g. demand-paged the page in, resolved COW). If it can't be fixed, `SIGSEGV`.

**Q5.** Why route device interrupts away from the trading core in HFT?
**Answer:** An interrupt on the trading core preempts the hot thread at an unpredictable moment
(jitter) and pollutes its cache/TLB. Setting IRQ affinity to other cores keeps the trading core running
the strategy uninterrupted; combined with busy-polling the NIC, the hot path sees neither interrupts nor
their jitter.

**Q6.** What is EOI and what breaks if the handler forgets it?
**Answer:** End Of Interrupt — the ISR signals the APIC that it's done, so the controller can deliver the
next (equal/lower priority) interrupt. Forgetting EOI means the controller thinks the interrupt is still
in service and won't deliver more of that line — the device effectively stops being serviced (a hung
IRQ).

**Q7.** Why are interrupts checked at instruction boundaries rather than mid-instruction?
**Answer:** So the saved context (`RIP` etc.) is a clean, restartable state. Interrupting mid-instruction
would leave partial architectural effects that couldn't be cleanly resumed with `iret`. Some long
instructions are made interruptible/restartable specifically to bound interrupt latency, but the
principle is: take the interrupt only at a well-defined boundary.

**Q8.** True/false: adding a new syscall requires adding an IDT entry.
**Answer:** False. Syscalls are dispatched by *number* through `sys_call_table`, not by IDT vector. You
add a table entry and a handler; the single `syscall` entry point (`LSTAR`) already routes all of them.
IDT entries are for exceptions and device/software interrupts.

---

## Indian HFT interview questions

**Q1 (Tower Research, Gurgaon — the classic).** *What is the difference between an interrupt and a
system call, in terms of implementation?*
**Model answer:** A syscall is synchronous, voluntary, and software-initiated — the thread executes
`syscall`, which (on modern x86-64) enters via the `LSTAR` MSR and **bypasses the IDT**, dispatches by
number through `sys_call_table`, runs in the caller's process context (can block), and returns to the
*next* instruction. A hardware interrupt is asynchronous and involuntary — an external device raises an
IRQ; the APIC signals the core, which enters via **IDT[vector]** between arbitrary instructions, runs the
ISR in interrupt context (can't sleep), sends EOI, and `iret`s back to the *interrupted* instruction.
Same "trap to kernel" skeleton; opposite on timing, intent, entry mechanism, context, and return target.

**Q2 (Optiver — jitter).** *You see occasional latency spikes on your trading thread that correlate with
network load. What's happening and how do you fix it?*
**Model answer:** Incoming packets raise NIC interrupts that preempt the trading thread and pollute its
cache/TLB — classic interrupt jitter. Fixes: set IRQ affinity so NIC interrupts land on other cores,
isolate and pin the trading core (`isolcpus`/`nohz_full`), and move to kernel-bypass busy-polling so the
hot path receives packets without interrupts at all. Also route the timer/RCU work off the core with
`nohz_full`.

**Q3 (Graviton — full flow).** *Trace what happens from a packet hitting the NIC to your handler running,
in a normal (non-bypass) kernel.*
**Model answer:** NIC DMAs the frame into a ring buffer in RAM and raises an IRQ → APIC signals a core
with the device's vector → CPU finishes the current instruction, switches to ring 0 and a kernel stack via
the IDT gate, saves context → the top-half hardirq acknowledges the NIC and schedules NAPI/softirq →
returns quickly → later the softirq (bottom half) does IP/TCP processing with IRQs enabled and delivers
data to the socket → the userspace `recv` returns. Bypass removes almost all of this by mapping the ring
into user space and polling.

**Q4 (IMC — recoverability).** *Which exceptions can the kernel recover from and continue, and which are
fatal?*
**Model answer:** **Recoverable faults** re-run the instruction after fixing state — a page fault for a
demand-paged or COW page (map/copy the page, retry), an FPU-disabled trap (enable, retry). **Usually
fatal to the process** — invalid page fault (`SIGSEGV`), `#GP` (`SIGSEGV`/`SIGILL`), divide error
(`SIGFPE`) — the kernel signals the process. **Aborts** (machine-check, double fault) are unrecoverable
hardware/kernel errors and may panic the machine. The dividing line is whether the kernel can make the
faulting instruction succeed on retry.

**Q5 (AlphaGrep — precision).** *Does the modern `syscall` instruction use the interrupt descriptor
table?*
**Model answer:** No. `syscall` reads its entry point from the `LSTAR` MSR (and `STAR`/`FMASK` for
segments/flags) and jumps there directly, bypassing the IDT — that's why it's faster than the legacy
`int 0x80`, which *does* go through IDT vector 0x80. Hardware interrupts and CPU exceptions still use the
IDT. This is the crisp mechanism-level distinction.

**Q6 (Squarepoint — context rules).** *Why can't you take a mutex that might sleep inside a hardirq
handler?*
**Model answer:** A hardirq runs in interrupt context with no owning task and often IRQs disabled; there's
nothing schedulable to switch to if it blocks, so sleeping deadlocks/hangs the core and stalls further
interrupts. You use non-sleeping primitives (spinlocks with IRQs disabled) in the top half and defer any
work that might sleep to a threaded IRQ, softirq, or workqueue that runs in process context.

**Q7 (HRT — design intuition).** *Why does Linux split interrupt handling into top and bottom halves?*
**Model answer:** While the top half runs, the interrupt (and often others) is masked, so time spent there
adds latency to all interrupt handling and can drop events. So the top half does the minimum — ack the
device, grab the descriptor, schedule deferred work — and the bottom half (softirq/tasklet/threaded irq)
does the expensive protocol processing later with interrupts re-enabled. It bounds the critical hardirq
window and keeps interrupt latency low.

---

## Key takeaways

- Control enters the kernel three ways: **hardware interrupts** (asynchronous, external),
  **traps/software interrupts** (synchronous, voluntary — e.g. `syscall`, `int3`), and **faults/aborts**
  (synchronous, involuntary — e.g. page fault). The key axis is asynchronous-vs-synchronous.
- The interrupt flow: device → APIC → CPU finishes current instruction → **IDT[vector]** → ring-0 stack
  switch + context save → **ISR** (top half fast, bottom half deferred) → **EOI** → **`iret`** back to
  the interrupted instruction.
- **Syscall vs interrupt:** syscall = synchronous, voluntary, resumes at the *next* instruction, runs in
  the caller's context, modern `syscall` **bypasses the IDT** (MSRs). Interrupt = asynchronous,
  involuntary, resumes at the *interrupted* instruction, runs in interrupt context (can't sleep), goes
  **through the IDT**.
- A **page fault** is synchronous-but-involuntary and **re-runs** the faulting instruction — the perfect
  example that the interrupt/trap/fault axes are real, not cosmetic.
- Interrupt handlers **can't sleep**; heavy work is deferred (top/bottom half) to keep the masked hardirq
  window tiny.
- HFT treats interrupts as **jitter**: route device IRQs to other cores and busy-poll the NIC so the hot
  path never takes one.

**Next:** [10 — Processes & context switching](10-processes-context-switching.md)
