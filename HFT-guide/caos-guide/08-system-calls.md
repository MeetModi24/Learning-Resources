# Module 08 — System calls

A **system call** is the door from Module 07 in action: the single, controlled way a user program
asks the kernel to do something privileged on its behalf — read a file, send a packet, get more
memory, look at the clock. This module traces one syscall end-to-end at the register and instruction
level, explains why it's expensive, and shows the tricks (vDSO, kernel bypass) HFT uses to avoid
paying that cost on the hot path.

---

## 1. What a syscall is (and isn't)

Your program can't read a file directly — the disk is a shared device, and touching it is privileged
(Module 07). So it asks the kernel: "please read 4096 bytes from fd 3 into this buffer." That request
*is* a system call. The kernel validates it (is fd 3 really yours? is the buffer in your address
space?), does the privileged work, and returns a result.

A syscall is **synchronous, deliberate, and software-initiated**: your running thread executes a
specific instruction (`syscall` on x86-64) at a specific point in its code, on purpose, and control
returns to the *very next* instruction. Hold onto that — Module 09 contrasts it with interrupts,
which are asynchronous and involuntary.

You almost never write a syscall by hand; you call a **libc wrapper** like `read()`, which is a thin
function that marshals arguments and executes the syscall instruction. `read()` the C function ≠
`read` the syscall, though they line up 1:1 here. Some libc functions (`printf`, `malloc`) call *zero
or many* syscalls underneath.

---

## 2. The Linux/x86-64 calling convention

There's a fixed ABI so the kernel knows where to find the request:

| Register | Holds |
|----------|-------|
| `rax`    | **syscall number** (which call: 0=read, 1=write, 2=open, …) — and, on return, the **result** |
| `rdi`    | arg 1 |
| `rsi`    | arg 2 |
| `rdx`    | arg 3 |
| `r10`    | arg 4 (note: **not** `rcx` — `syscall` clobbers `rcx`) |
| `r8`     | arg 5 |
| `r9`     | arg 6 |

So `read(fd, buf, count)` becomes: `rax=0, rdi=fd, rsi=buf, rdx=count`, then execute `syscall`. The
return value (bytes read, or a negative errno) comes back in `rax`. Up to 6 register arguments — no
stack args for syscalls, which keeps the boundary cheap and simple.

---

## 3. End-to-end: one `read()` call, instruction by instruction

```
USER MODE (ring 3)                                      KERNEL MODE (ring 0)
──────────────────                                      ────────────────────
  // read(3, buf, 4096)
  mov  rax, 0          ; syscall number = read
  mov  rdi, 3          ; fd
  mov  rsi, buf        ; buffer
  mov  rdx, 4096       ; count
  syscall  ───────────────────────────────────►  [CPU, atomically:
                                                    - save user RIP  → rcx
                                                    - save RFLAGS    → r11
                                                    - load RIP from MSR_LSTAR
                                                      (kernel entry point)
                                                    - switch CPL 3 → 0
                                                    - (mask flags per MSR_FMASK) ]
                                                        │
                                                   entry_SYSCALL_64:
                                                    - swapgs (get kernel data)
                                                    - switch to kernel stack
                                                    - push user registers
                                                    - range-check rax
                                                    - call sys_call_table[rax]   ── indexes by number
                                                        │
                                                   sys_read(fd, buf, count):
                                                    - validate fd, copy_to_user checks
                                                    - do the privileged I/O
                                                    - return bytes read → rax
                                                    - restore user registers
  next instruction ◄────────  sysret  ◄───────────  [CPU: RIP := rcx, RFLAGS := r11,
  (result in rax)                                    switch CPL 0 → 3]
```

Key points:

- **The `syscall` instruction is the single hardware step** that flips privilege and redirects to the
  kernel's fixed entry point — the "door" from Module 07. It doesn't take a target address from the
  user; the destination is whatever the kernel wrote into the `LSTAR` **model-specific register** at
  boot. You can't point it at your own code.
- **`sys_call_table`** is a big array of function pointers indexed by syscall number. `rax=0` →
  `sys_read`. This table-driven dispatch is why adding a syscall means adding a table entry.
- **The kernel distrusts every argument.** Pointers are checked to be in the user's address space;
  `copy_from_user`/`copy_to_user` do the crossing safely. A syscall is a trust boundary, not a
  function call.
- **`sysret`** (or `iret` in older paths) reverses everything: restores the saved user `RIP`/`RFLAGS`
  and drops back to ring 3, resuming at the instruction after `syscall`.

### Three ways to enter, historically

- **`int 0x80`** — the original i386 mechanism: a software interrupt through the **IDT** (Module 09).
  Correct but slow (goes through the interrupt machinery).
- **`sysenter`/`sysexit`** (Intel) — a faster dedicated pair introduced to avoid the IDT overhead.
- **`syscall`/`sysret`** (AMD64, now universal on x86-64) — the modern fast path; uses MSRs
  (`LSTAR`, `STAR`, `FMASK`) instead of the IDT. **This is the important distinction for Module 09:
  the modern `syscall` instruction does *not* go through the IDT, whereas hardware interrupts and
  `int 0x80` do.**

On **AArch64** the equivalent is the **`svc`** (supervisor call) instruction, trapping from EL0 to
EL1; the syscall number is in `x8`, args in `x0`–`x5`, return in `x0`.

---

## 4. Why syscalls are expensive

Even the fast `syscall` path is far more than a function call:

1. **Mode switch** — save/restore user context (`RIP`, `RFLAGS`, and the kernel entry saves the
   general registers), `swapgs`, stack switch.
2. **Pipeline effects** — the transition is a serialising-ish event; speculation across it is limited.
3. **Cache and TLB pollution** — the kernel executes its own code and touches its own data, evicting
   *your* hot code and data from L1/L2. When you return, your next accesses miss. This **indirect
   cost** often dwarfs the direct cost.
4. **KPTI (post-Meltdown)** — with page-table isolation on, entry switches to a minimal kernel page
   table, flushing much of the TLB — a real regression (tens to hundreds of extra cycles).

Ballpark: a null syscall is ~**100–400 ns** on modern hardware including indirect effects — thousands
of cycles. A plain function call is ~1–2 ns. That's a 100×+ gap, and worse, it's **variable** (depends
on cache state, KPTI, contention) — which for HFT is the real sin: jitter.

---

## 5. Avoiding the cost: vDSO and kernel bypass

Because some syscalls are *read-only and harmless*, Linux avoids trapping for them via the **vDSO
(virtual dynamic shared object)** — a small kernel-provided shared library mapped into every process's
address space. Calls like `clock_gettime`, `gettimeofday`, and `getcpu` are implemented in the vDSO:
they read a kernel-maintained page of data (updated by the kernel) *in user mode*, with **no
mode switch at all**. So `clock_gettime(CLOCK_MONOTONIC, …)` is often just a few-nanosecond user-space
read — critical, because HFT timestamps everything and can't afford a syscall per timestamp.

```
   Normal syscall:   user → [trap] → kernel does work → [return] → user      (~hundreds of ns)
   vDSO "syscall":   user reads a kernel-updated page IN user mode           (~few ns, no trap)
```

For the network hot path, HFT goes further with **kernel bypass** (DPDK, Solarflare Onload,
`AF_XDP`): the NIC's queues are mapped into user space, so sending/receiving packets touches device
memory directly and **skips `send`/`recv` syscalls entirely**. The trade-off is burning a core on a
busy-poll loop and giving up the kernel's convenience/safety — worth it to shave microseconds and,
more importantly, to make latency *flat*.

**The general HFT rule:** no syscalls on the hot path. Pre-allocate memory at startup (no `mmap`/`brk`
while trading), use `clock_gettime` via vDSO for timestamps, kernel-bypass for packets, and batch or
offload any logging to a separate thread/core.

---

## Common pitfalls / misconceptions

- **"A libc call is a syscall."** Not necessarily. `memcpy`, `strlen`, arithmetic — pure user space.
  `malloc` usually *no* syscall (it recycles memory it already got); only occasionally `mmap`/`brk`.
  `printf` buffers and may do one `write` for many calls.
- **"`errno` is returned by the syscall."** The kernel returns a negative error code in `rax`; the
  libc wrapper detects the negative range, stores the positive value in `errno`, and returns −1.
- **"`syscall` goes through the interrupt table."** The *modern* `syscall` instruction does **not** —
  it uses MSRs. Only `int 0x80` and hardware interrupts use the IDT. (Common trap in interviews.)
- **"System calls are slow because the kernel code is slow."** The dominant cost for a *cheap* syscall
  is the boundary crossing and cache/TLB pollution, not the handler's own work.

---

## Quiz — tough problems

**Q1.** Why does the syscall ABI use `r10` for the 4th argument instead of `rcx`, when the normal C
function ABI uses `rcx`?
**Answer:** The `syscall` instruction itself clobbers `rcx` — it stashes the return `RIP` there. So the
4th arg is moved to `r10` before executing `syscall`; the libc wrapper does the `rcx`→`r10` shuffle. A
detail that shows you actually know the mechanism.

**Q2.** You call `clock_gettime` a million times in a tight loop and see ~3 ns each — but a colleague
insists every syscall costs hundreds of ns. Who's right?
**Answer:** Both. `clock_gettime` for `CLOCK_MONOTONIC`/`REALTIME` is served by the **vDSO** in user
space with no trap, so ~ns. It's a "syscall" by API but not by mechanism. A real trapping syscall
(e.g. `read`) would indeed be hundreds of ns.

**Q3.** After a syscall returns, your next few memory accesses are unexpectedly slow. Why?
**Answer:** **Cache/TLB pollution.** While in the kernel, its code and data evicted your hot lines from
L1/L2 and its accesses displaced TLB entries (worse with KPTI, which flushes on entry). On return, your
working set is partly cold, so the first accesses miss. This indirect cost is often bigger than the
direct switch cost.

**Q4.** Is it possible to point the `syscall` instruction at your own function to run it in kernel mode?
**Answer:** No. `syscall` loads `RIP` from the kernel-owned `LSTAR` MSR (set at boot, unwritable from
user mode). The destination is fixed by the kernel; you only supply a syscall *number* in `rax`, which
the kernel range-checks before indexing `sys_call_table`. This is exactly the Module 07 guarantee.

**Q5.** How does the kernel prevent you from passing a pointer to *its* memory as a syscall buffer to
read kernel secrets?
**Answer:** The kernel never blindly dereferences user pointers. It uses `copy_from_user`/`copy_to_user`,
which check the pointer lies in the user address range and handle faults safely. Passing a kernel
address yields `-EFAULT`, not a leak.

**Q6.** `write()` returns 100 when you asked to write 4096 bytes. Bug?
**Answer:** No — that's a **short write**, legal for `write`. The syscall may transfer fewer bytes (e.g.
pipe/socket buffer full, signal interruption). Correct code loops, advancing the buffer by the returned
count until done (or handles `EINTR`). A frequent source of real bugs.

**Q7.** Why might replacing 1000 small `write()` calls with one big `writev()`/buffered `write()` be a
huge speedup even though the same bytes move?
**Answer:** Each `write` pays the full boundary-crossing cost (mode switch + cache/TLB pollution),
independent of payload size. 1000 crossings ≈ 1000× that fixed overhead. Batching amortises the fixed
cost over more data — one crossing instead of a thousand. This is why HFT buffers and batches any
unavoidable syscalls.

**Q8.** On x86-64, which of these involve the IDT: `int 0x80`, `syscall`, a NIC interrupt, a page
fault?
**Answer:** `int 0x80` (yes, software interrupt vector 0x80), NIC interrupt (yes), page fault (yes,
vector 14) — all go through the IDT. **`syscall` does not** — it uses `LSTAR`/`STAR` MSRs. This
IDT-vs-MSR split is the crisp answer to "difference between syscall and interrupt implementation."

---

## Indian HFT interview questions

**Q1 (Tower Research, Gurgaon — the headline).** *Walk me through exactly what happens when a program
makes a system call.*
**Model answer:** The libc wrapper loads the syscall number into `rax` and args into `rdi, rsi, rdx,
r10, r8, r9`, then executes `syscall`. The CPU atomically saves user `RIP`→`rcx` and `RFLAGS`→`r11`,
loads `RIP` from the kernel's `LSTAR` MSR, and switches to ring 0. The kernel entry stub does `swapgs`,
switches to the kernel stack, saves registers, range-checks `rax`, and calls `sys_call_table[rax]`. The
handler validates arguments (`copy_from/to_user`), does the privileged work, puts the result in `rax`,
restores registers, and `sysret` drops back to ring 3 at the instruction after `syscall`. Cost is a
mode switch plus cache/TLB pollution — hundreds of ns — which is why we keep them off the hot path.

**Q2 (Optiver — latency focus).** *How do you get a timestamp millions of times per second without
killing latency?*
**Model answer:** Use `clock_gettime(CLOCK_MONOTONIC)` served by the **vDSO** — the kernel maps a page
of clock data into user space and updates it, so the call reads it in user mode with no trap, ~few ns.
For even tighter needs, read the **TSC** (`rdtsc`/`rdtscp`) directly and convert with a calibrated
frequency, but you must account for TSC scaling and per-core sync (`rdtscp` gives the core id). Never
call a trapping syscall per timestamp.

**Q3 (Graviton — mechanism precision).** *What's the difference between `int 0x80` and `syscall`?*
**Model answer:** `int 0x80` is a software interrupt: it goes through the **IDT** (vector 0x80), like
any interrupt, which is slower. `syscall` (AMD64) is a dedicated fast instruction that uses
model-specific registers (`LSTAR` for the entry point, `STAR` for segments, `FMASK` for flags) and
bypasses the IDT entirely, saving `RIP`/`RFLAGS` in `rcx`/`r11`. Modern x86-64 Linux uses `syscall`.
This IDT-vs-MSR distinction is also the core of how syscalls differ from hardware interrupts.

**Q4 (IMC — why it costs).** *Why is a system call so much more expensive than a function call if it's
"just" a jump into kernel code?*
**Model answer:** Direct cost: privilege switch, `swapgs`, stack switch, saving/restoring registers,
and with KPTI a page-table switch that flushes TLB. Indirect cost (usually larger): while in the kernel,
its instructions and data evict your hot L1/L2 lines and TLB entries, so your code runs cold on return.
A function call touches none of that. And the cost is *variable*, which for us is worse than its
magnitude.

**Q5 (AlphaGrep — systems design).** *Your strategy needs to send an order. Describe the difference
between the syscall path and kernel bypass.*
**Model answer:** Syscall path: build the packet, call `send()` → trap into the kernel → the network
stack copies the buffer, does TCP/IP processing, hands it to the driver → return. Multiple crossings and
copies, µs-scale and jittery. Kernel bypass (DPDK/Onload/`AF_XDP`): the NIC's TX ring is mapped into
user space, so we write the descriptor and ring the doorbell directly from user mode — no `send`
syscall, no kernel copy, no stack traversal. Cost: we busy-poll and reimplement the transport, but
latency drops and flattens. HFT uses bypass on the wire-facing hot path.

**Q6 (Squarepoint — correctness).** *A junior writes `write(fd, buf, n)` once and assumes all `n` bytes
are written. What do you tell them?*
**Model answer:** `write` can do a **short write** — return fewer than `n` (buffer full, signal, socket
pressure) — and can return `-1` with `errno==EINTR`. Correct code loops: advance `buf` and decrement `n`
by the returned count, retry on `EINTR`. Same for `read`. Assuming a full transfer is a real bug that
shows up under load — exactly when it hurts.

**Q7 (HRT — depth).** *Why does the kernel copy data with `copy_to_user` instead of just dereferencing
the user pointer you passed?*
**Model answer:** The user pointer is untrusted: it could point at kernel memory (to leak secrets),
be unmapped (to crash the kernel), or race with another thread unmapping it. `copy_to_user`/`copy_from_user`
verify the address is in the user range and use fault-tolerant accessors that turn a bad access into
`-EFAULT` instead of a kernel oops. It's the enforcement of the trust boundary a syscall represents.

---

## Key takeaways

- A syscall is the **synchronous, deliberate, software-initiated** door into the kernel: put a number
  in `rax`, args in `rdi/rsi/rdx/r10/r8/r9`, execute `syscall`.
- The hardware atomically switches to ring 0 and jumps to the **kernel-owned** entry point
  (`LSTAR`); the kernel then dispatches through **`sys_call_table[rax]`** and returns the result in
  `rax`. You can never redirect it to your own code.
- Modern `syscall`/`sysret` **bypasses the IDT** (uses MSRs); only `int 0x80` and hardware interrupts
  use the IDT — the crisp "syscall vs interrupt implementation" distinction.
- Syscalls cost **hundreds of ns and are jittery**, dominated by the mode switch plus cache/TLB
  pollution (worse under KPTI) — not by the handler's work.
- The **vDSO** serves harmless read-only calls (`clock_gettime`) in user mode with no trap; **kernel
  bypass** removes packet syscalls entirely. Both exist to keep the boundary off the hot path.
- HFT rule: **no trapping syscalls while trading** — pre-allocate, vDSO/TSC for time, kernel-bypass
  for packets, batch/offload logging.

**Next:** [09 — Interrupts & exceptions](09-interrupts-and-exceptions.md)
