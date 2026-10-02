# Module 13 — Scheduling & real-time

Module 10 established *that* the kernel multiplexes the CPU across many runnable threads, and separated
the **scheduler** (policy: who runs next) from the **dispatcher** (mechanism: the context switch). This
module opens the scheduler. It answers the questions interviewers actually probe: *which* runnable task
does Linux pick, by what rule, how do priority and `nice` map to real numbers, what the real-time
scheduling classes (`SCHED_FIFO`/`SCHED_RR`/`SCHED_DEADLINE`) buy you, and — the HFT punchline — why a
trading desk fights to make scheduling a *non-event* on its hot core: pin it, give it a real-time
class, isolate it, and ensure the scheduler and the load balancer never touch it again.

---

## 1. The scheduling problem

At any instant there may be more runnable threads than cores. On an N-core machine at most N threads
are RUNNING (Module 10); every other runnable thread sits in a **run queue** in the READY state. The
scheduler's job is to answer, each time a core becomes free (a thread blocks, exits, or is preempted):
*of the runnable threads, which one gets this core next, and for how long?*

Two sub-decisions hide in that:

- **Selection** — which thread. This is the *policy*: fairness, priority, deadlines.
- **Preemption / time** — for how long before we re-decide. A **time slice** (quantum) bounds how long
  one thread monopolises a core before the timer interrupt (Module 09) forces a re-schedule.

Keep the Module 10 vocabulary straight: the **scheduler picks**, the **dispatcher does** (performs the
register save/restore + CR3 reload). This module is entirely about the *picking*. The cost of the
*doing* — ~1–5 µs, dominated by TLB flush and cold caches — is Module 10's and is exactly why fewer,
rarer scheduling decisions is itself a latency goal.

The scheduler is invoked at well-defined points: on the **timer tick** (periodic preemption check), when
a thread **blocks** or **exits**, when a thread is **woken** (a higher-priority sleeper becoming
runnable may preempt the current one), and on an explicit `sched_yield()`. Between those points your
thread runs undisturbed — which is the whole game for HFT.

---

## 2. Scheduling algorithms — the concepts

Interviewers expect the classic menu, then the real Linux mapping. The classics, briefly:

| Algorithm | Rule | Pro | Con |
|---|---|---|---|
| **FCFS** (first-come-first-served) | run in arrival order, to completion | simple, no starvation | one long job stalls everyone (convoy effect) |
| **SJF / SRTF** (shortest job first) | run the shortest (remaining) burst next | provably minimal average wait | needs the future; starves long jobs |
| **Round-robin (RR)** | each gets one fixed quantum, then back of queue | fair, responsive, no starvation | quantum tuning: too small ⇒ switch overhead; too big ⇒ RR degrades to FCFS |
| **Priority** | highest priority runnable thread runs | lets important work jump the queue | low-priority threads can **starve** (Module 14) |

**Round-robin and the quantum.** RR is priority-free fairness: a circular queue, each thread gets a
quantum (classically 10–100 ms; on Linux the effective slice is derived, see §5), then is preempted to
the tail. The quantum is the central tuning knob — small quanta give snappy interactivity at the cost of
more context switches (each ~µs of overhead), large quanta amortise switch cost but hurt responsiveness.

**Priority: static vs dynamic.**

- **Static priority** never changes once set. Real-time classes work this way — a `SCHED_FIFO` thread at
  priority 80 stays at 80 (§4).
- **Dynamic priority** is recomputed by the kernel from observed behaviour. The classic heuristic:
  **boost I/O-bound threads, penalise CPU-bound ones.** A thread that sleeps a lot (waiting on I/O,
  interactive) earns a priority bonus so it responds fast when its data arrives; a thread that burns its
  whole quantum every time earns a penalty so it doesn't hog the CPU. The old O(1) scheduler literally
  tracked a `sleep_avg` per task (more sleep ⇒ bigger bonus) and docked time for CPU consumption.

That `sleep_avg` bonus/penalty machinery is *gone* in modern Linux — but the *intent* survives in CFS,
which achieves the same "interactive tasks feel fast" outcome through a cleaner mechanism (virtual
runtime, §5) rather than ad-hoc heuristics. When an interviewer asks "how does Linux favour interactive
tasks?", the right answer is CFS/vruntime, with the `sleep_avg` story as the historical motivation.

---

## 3. The Linux priority model (concrete numbers)

This is a favourite because candidates wave at "priority" without knowing the actual ranges. Linux has
**one internal priority axis, 0–139**, split into two regions:

```
  internal priority:  0 ........... 99 | 100 ................... 139
                      └── real-time ──┘ └──────── normal (CFS) ────────┘
                       higher urgency        nice −20 .......... +19
                                              prio 100 ......... 139
                                                    (120 = nice 0, default)

  LOWER number  =  HIGHER priority  (prio 0 beats prio 139)
```

- **Normal threads (CFS)** occupy internal priority **100–139**, selected by the **nice** value, which
  ranges **−20 (most favoured) to +19 (least favoured)**, default **0**. The mapping is
  `prio = 120 + nice`, so nice 0 ⇒ prio 120 (`DEFAULT_PRIO`), nice −20 ⇒ prio 100, nice +19 ⇒ prio 139.
  Nice is *not* a hard priority — under CFS it scales each thread's share of CPU weight (roughly, each
  nice step is ~1.25× CPU), it does not guarantee one thread beats another.
- **Real-time threads** occupy **0–99** and *always* run before any normal thread. Here priority is
  absolute, not a weight (§4).

Confusingly, two "priority" numbers exist in userland. The `nice`/`setpriority` number is the −20..+19
one (CFS only). The `sched_setscheduler`/`chrt` **real-time priority** is 1..99 and only applies to RT
classes. They are different axes; `nice` has no effect on a `SCHED_FIFO` thread.

```bash
nice -n 10 ./batch_job          # start a process niced down (prio 130)
renice -n -5 -p 4021            # re-nice a running PID up
# programmatic:
setpriority(PRIO_PROCESS, 0, 10);   // nice this process to +10
```

Note `nice` is subject to permission: lowering niceness below 0 (raising priority) needs
`CAP_SYS_NICE`/root. A normal user can only be *nicer*, not greedier.

---

## 4. Linux scheduling classes

Linux doesn't run one algorithm — it runs a **stack of scheduling classes**, consulted in strict
priority order. When picking the next thread for a core, the kernel asks each class in order and takes
the first that has a runnable thread:

```
   pick-next order (highest wins):
   ┌────────────────────────────────────────────────────────┐
   │ stop_sched  (kernel internal, migration/CPU-stop)        │
   │ SCHED_DEADLINE  (EDF — earliest deadline first)          │  real-time
   │ SCHED_FIFO / SCHED_RR  (fixed RT priority 1..99)         │  (prio 0..99)
   │ CFS / SCHED_NORMAL, SCHED_BATCH, SCHED_IDLE  (nice/vruntime) │  normal (100..139)
   └────────────────────────────────────────────────────────┘
```

**A single runnable `SCHED_FIFO` thread beats every CFS thread on that core, always.** That's the
property HFT buys. The classes:

- **CFS (`SCHED_NORMAL`)** — the Completely Fair Scheduler, default for everything. Models an "ideal
  fair CPU" where every runnable thread accrues **virtual runtime (`vruntime`)** at a rate inversely
  weighted by its nice value. All runnable CFS threads live in a **red-black tree keyed by vruntime**;
  the scheduler simply picks the leftmost node — the thread that has run the *least* virtual time — runs
  it, and when it yields/is-preempted its vruntime has advanced, moving it rightward. Lower nice ⇒ smaller
  vruntime increments ⇒ it sits leftward longer ⇒ more CPU. There is no fixed quantum; the slice is
  computed from the number of runnable tasks (a target latency, e.g. ~6 ms, divided among them). This is
  O(log n) to insert/pick. `SCHED_BATCH` (non-interactive, no preemption-on-wake) and `SCHED_IDLE` (runs
  only when nothing else wants the core) are CFS variants.

- **`SCHED_FIFO`** — real-time, **fixed priority 1..99, no time slice**. A FIFO thread runs until it
  *voluntarily* blocks, yields, or is preempted by a *strictly higher* RT-priority thread. Two FIFO
  threads at the same priority: the running one keeps the CPU until it blocks (the other waits — this is
  the "FIFO" part). This is the classic HFT hot-thread policy: pin it, set FIFO at high priority, and the
  kernel will never preempt it for a normal task.

- **`SCHED_RR`** — like FIFO but with a **time slice**: same-priority RR threads round-robin the quantum
  among themselves. Still beats all CFS; still fixed priority. Use when you have several equal-priority RT
  threads that must share.

- **`SCHED_DEADLINE`** — Linux's **EDF** implementation (§5), the highest-priority class, above FIFO/RR.
  Each thread declares `(runtime, deadline, period)`; the kernel admits it only if the set is schedulable
  and then runs the thread with the **earliest deadline** first. Used for strict periodic real-time work.

```bash
chrt -f 80 ./trading_engine        # SCHED_FIFO, RT priority 80
chrt -r 50 ./worker                # SCHED_RR, RT priority 50
chrt -p 4021                       # query a PID's policy & priority
# programmatic:
struct sched_param p = { .sched_priority = 80 };
sched_setscheduler(0, SCHED_FIFO, &p);   // needs CAP_SYS_NICE
```

> **The RT self-foot-gun.** A `SCHED_FIFO` thread that busy-loops and never blocks will **monopolise its
> core forever**, starving every CFS task there — including kernel threads. If it's the only core, the
> box can wedge. Linux's default guard is **RT throttling** (`sched_rt_runtime_us`: RT may use at most
> 950 ms per 1 s by default, yielding 50 ms to others). HFT setups that pin a busy-polling FIFO thread
> to an *isolated* core typically disable this throttle for that core — the busy-poll is intentional, and
> there's nothing else on the core to starve.

---

## 5. Real-time scheduling theory

"Real-time" does **not** mean "fast." It means **deterministic** — a *bounded, predictable* worst-case
response time, even if the average is slower. For HFT the metric that matters is **jitter** (variance of
latency), not raw throughput: a path that's 800 ns every single time beats one that's 300 ns usually but
5 µs on the 1-in-10000 that gets preempted.

- **Hard real-time** — missing a deadline is a *failure* (flight control, pacemaker). Requires provable
  worst-case bounds.
- **Soft real-time** — missing a deadline degrades quality but is tolerable (video frame drop). HFT is
  effectively soft-RT with money on the line: a missed tick isn't a crash, it's a worse fill or a missed
  trade.

The two classic RT scheduling algorithms, both of which an interviewer may ask you to compare:

```
  Tasks: A(period 50ms), B(period 30ms), C(period 20ms)

  RATE MONOTONIC (RMS) — STATIC priority by period:
     shorter period ⇒ higher priority  ⇒  C > B > A, fixed forever.
     Simple, priorities assigned once. Optimal among *fixed-priority* schemes.
     Schedulable if total utilisation ≤ n(2^(1/n) − 1)  (≈ 69% as n → ∞).

  EARLIEST DEADLINE FIRST (EDF) — DYNAMIC priority by absolute deadline:
     at every instant, run the task whose next deadline is soonest.
     Priorities shuffle as deadlines approach.
     Optimal: schedulable whenever total utilisation ≤ 100%  (uses the CPU fully).
     Cost: more bookkeeping; overload behaviour is worse (a miss can cascade).
```

**Rate Monotonic** assigns priority by period — the more frequently a task must run, the higher its
(static) priority. It's optimal among static-priority schemes and dead simple, but can't use the CPU past
~69% utilisation in the worst case. **EDF** recomputes "who's most urgent" continuously by nearest
absolute deadline, achieving full 100% utilisation, at the cost of runtime overhead and uglier overload
behaviour.

Mapping to Linux: **`SCHED_DEADLINE` *is* EDF** (constant-bandwidth-server EDF, to be precise). The
fixed-priority RT classes `SCHED_FIFO`/`SCHED_RR` are the vehicle for a Rate-Monotonic-style assignment
(you set the periods-to-priorities mapping yourself). **In practice HFT rarely uses `SCHED_DEADLINE`** —
the hot thread isn't periodic, it's an event-driven busy-poll loop that must react to a packet *now*. So
the dominant HFT choice is **`SCHED_FIFO` at a high priority on a dedicated core**: give it absolute
precedence and let it spin.

---

## 6. CPU affinity & cache locality

By default the scheduler is free to run a thread on *any* core and to **migrate** it between cores. For
HFT that freedom is poison. **CPU affinity** pins a thread to a specific core (or set):

```c
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(3, &set);                         // only core 3
sched_setaffinity(0, sizeof(set), &set);  // 0 = this thread
```

```bash
taskset -c 3 ./trading_engine     # launch pinned to core 3
taskset -cp 3 4021                # pin running PID 4021 to core 3
```

**Why it matters — the cache argument (Module 04/05).** Each physical core has its *own* private L1 and
(usually) L2 cache. A thread that has been running on core 3 has its hot working set — order book, state,
code — resident in core 3's L1/L2. If the scheduler migrates it to core 7, **none of that is there**: core
7's caches are cold for this thread. It eats a burst of L1/L2 misses, each pulling lines from L3 (~40
cycles) or DRAM (~200+ cycles, ~60–100 ns), for thousands of cycles until the working set is re-warmed.
Worse, it's **jitter** — the migration happens at an unpredictable moment, so your tail latency spikes
exactly when you can't predict. There's also a TLB cost analogous to the context-switch story in Module
10. Pinning eliminates the migration entirely: the thread stays where its caches are warm.

(This stacks with the interrupt-affinity argument from Module 09: route device IRQs *away* from the
trading core so interrupts don't preempt or pollute it either.)

---

## 7. Load balancing — and why HFT turns it off

With per-core run queues, the scheduler runs a **load balancer**: periodically (and on certain events) it
compares queue lengths across cores and **migrates** runnable threads from busy cores to idle ones, so no
core sits idle while another has a backlog. Good for a general server maximising throughput. Catastrophic
for a latency-critical thread, for exactly the §6 reasons — the balancer can yank your hot thread to a
cold core, or schedule *other* work onto the trading core "to balance load."

The HFT counter-move is to **remove the trading core from the scheduler's purview entirely**:

```
   boot: isolcpus=2,3  nohz_full=2,3  rcu_nocbs=2,3

   ┌──────── general cores 0,1 ────────┐   ┌─── isolated cores 2,3 ───┐
   │ CFS, load balancer, kernel threads │   │ ONLY the pinned FIFO      │
   │ timer ticks, RCU callbacks, IRQs   │   │ trading threads. No        │
   │ everything the OS normally does    │   │ balancing, no ticks, no    │
   └────────────────────────────────────┘   │ RCU, IRQs routed elsewhere │
                                             └────────────────────────────┘
```

- **`isolcpus=2,3`** — removes cores 2,3 from the scheduler's load-balancing domains. The balancer will
  never place a thread there on its own; only explicit `sched_setaffinity`/`taskset` pinning puts a
  thread on them. Your hot thread lands there and nothing else does.
- **`nohz_full=2,3`** — **tickless** operation: with a single runnable thread on the core, stop the
  periodic ~1000 Hz timer interrupt. That timer tick is otherwise a guaranteed source of jitter (a
  scheduler-tick every ms, each preempting your thread briefly). Removing it means the thread runs
  genuinely uninterrupted.
- **`rcu_nocbs=2,3`** — move RCU callback processing off the isolated cores so RCU housekeeping doesn't
  run there.
- Plus **IRQ affinity** (Module 09) to route NIC/timer interrupts to cores 0,1, and often disabling RT
  throttling for the isolated core (§4).

**The HFT scheduling stance, in one breath:** give the hot thread a *whole isolated core to itself*
(`isolcpus`), pin it there (`sched_setaffinity`), put it in a real-time class so nothing normal can ever
preempt it (`SCHED_FIFO` high priority), strip the periodic tick (`nohz_full`), route interrupts and RCU
away — so the scheduler and the load balancer effectively **never run** on that core. Scheduling, the
thing this whole module is about, becomes a *non-event* on the path that matters. The thread busy-polls
the NIC (Module 09/12), never blocks (Module 10), never migrates, never gets preempted — deterministic,
sub-microsecond, zero jitter from the OS.

---

## Common pitfalls / misconceptions

- **"Real-time means fast."** No — it means *deterministic / bounded*. An RT thread may have higher
  average latency than a tuned normal thread; what it guarantees is a bounded *worst case* and low jitter.
- **"`nice` sets priority like RT priority."** `nice` (−20..+19) only weights CPU *share* under CFS; it
  can't make one normal thread strictly beat another, and it has zero effect on an RT thread. RT priority
  (1..99 via `chrt`) is a different, absolute axis.
- **"Lower priority number = less important."** Backwards in Linux's internal scale: **lower number =
  higher priority** (prio 0 is the most urgent).
- **"CFS uses time slices / round-robin."** CFS has no fixed quantum; it tracks **virtual runtime** and
  always runs the least-run thread (leftmost in a red-black tree). The slice is derived from the runnable
  count.
- **"A `SCHED_FIFO` thread is safe to busy-loop anywhere."** It will starve everything of lower priority
  on its core — including kernel threads — unless RT throttling catches it or the core is isolated with
  nothing else to starve.
- **"Pinning is just an optimisation."** For the hot path it's correctness-of-latency: without it,
  migration gives you cold-cache jitter spikes exactly when you can't afford them.
- **"The scheduler does the context switch."** (Module 10) The scheduler *selects*; the dispatcher
  *performs* the switch. This module is only the selection half.

---

## Quiz — tough problems

**Q1.** A thread is `SCHED_FIFO` priority 90 on core 3. A `SCHED_FIFO` priority 95 thread on core 3
becomes runnable. What happens? What if the newcomer were priority 85 instead?

**Answer:** At 95 it **preempts** the running 90 immediately (strictly higher RT priority wins, and FIFO
is fully preemptive by higher RT priority). At 85, nothing — the running 90 keeps the core until it
blocks or yields; the 85 waits. FIFO only yields to *strictly higher* RT priority.

---

**Q2.** You set your trading thread to `SCHED_FIFO` priority 99 but *didn't* isolate the core, and the box
occasionally freezes for ~50 ms. Explain.

**Answer:** The busy-polling FIFO thread monopolises the core; **RT throttling** kicks in
(`sched_rt_runtime_us` = 950 ms/s default), forcibly giving ~50 ms/s back to CFS so kernel/other threads
aren't starved — that 50 ms is your freeze. Fix: isolate the core (`isolcpus`), route other work off it,
and disable RT throttling for it so the intentional busy-poll isn't throttled.

---

**Q3.** Rate Monotonic vs EDF: which can schedule a task set at 90% CPU utilisation, and what's the
trade-off?

**Answer:** **EDF** can (it's optimal, schedulable up to 100% utilisation). Rate Monotonic's worst-case
guaranteed bound is ~69% (n(2^(1/n)−1)); above that it *may* still work but isn't guaranteed. Trade-off:
RM is static (assign priorities once, trivial, predictable under overload), EDF is dynamic (recompute
nearest-deadline continuously — more overhead, and under overload a single miss can cascade).

---

**Q4.** Why does CFS use a red-black tree keyed by virtual runtime instead of a sorted list?

**Answer:** It needs the minimum-vruntime thread (leftmost) in O(1)/O(log n) and insert/remove in
O(log n) as threads sleep/wake and their vruntime advances. A red-black tree gives balanced O(log n)
ops; a sorted array would be O(n) to insert. "Pick leftmost, reinsert after it runs" is the whole CFS
loop.

---

**Q5.** Two normal threads, nice 0 and nice +10, both CPU-bound, on one core. Does the nice +10 thread
ever run, and roughly what share does it get?

**Answer:** Yes — nice is a *weight*, not a hard priority, so it isn't starved (unlike RT). Each nice step
is ~1.25× CPU weight; +10 steps ⇒ ~1.25^10 ≈ 9.3× less weight, so the nice 0 thread gets ~90% and the
nice +10 thread ~10%. (Contrast with RT, where a lower-priority thread would get **zero** while a higher
one is runnable.)

---

**Q6.** Your hot thread shows p99.9 latency spikes correlated with nothing in your code. `taskset` shows
it pinned. What OS sources of jitter remain, and how do you kill them?

**Answer:** Even pinned, the core can still take: (1) the **periodic timer tick** (~1000 Hz scheduler
tick) — kill with `nohz_full`; (2) **device interrupts** landing on the core — kill with IRQ affinity
(Module 09); (3) **RCU callbacks** — move off with `rcu_nocbs`; (4) the **load balancer** migrating other
work onto it — kill with `isolcpus`; (5) RT throttle hiccups — disable for the core. Full isolation
removes all five.

---

**Q7.** Why is a thread switch within a process cheaper to *schedule* and *dispatch* than a process
switch, and does the scheduler care which it is?

**Answer:** The *selection* cost is the same (the scheduler just picks a `task_struct`). The *dispatch*
cost differs (Module 10): same-process threads share CR3, so no TLB flush — cheaper. The scheduler
doesn't special-case it at selection time, but the dispatcher's work is smaller. HFT avoids both by never
switching the hot thread at all.

---

## Indian HFT interview questions

**Q1 (Optiver — the policy question).** *What scheduling policy do you put your trading thread on, and
why not just `nice -20`?*
**Model answer:** `SCHED_FIFO` at a high real-time priority, pinned to an isolated core. `nice -20` only
re-weights CPU *share* within CFS — a nice-20 thread can still be preempted by other CFS threads and by
the scheduler tick; it's a weight, not a guarantee. `SCHED_FIFO` is in a strictly higher scheduling class
than *all* normal threads, so no CFS task can ever preempt it, and FIFO has no time slice — it runs until
it blocks (which, busy-polling, it never does). Combined with `isolcpus`/`nohz_full`, the scheduler
effectively never fires on that core.

**Q2 (Tower Research, Gurgaon — numbers).** *What's the Linux priority range, where does `nice` sit, and
what's the default?*
**Model answer:** Internal priority is 0–139, lower = higher priority. 0–99 is real-time (set via `chrt`,
RT priority 1–99), 100–139 is normal/CFS, selected by `nice` −20..+19 with `prio = 120 + nice`. Default
nice is 0 ⇒ prio 120 (`DEFAULT_PRIO`). RT always beats normal. The `nice` axis and the `chrt` RT-priority
axis are different things.

**Q3 (Graviton — CFS internals).** *How does CFS decide who runs next?*
**Model answer:** It tracks each runnable thread's **virtual runtime** — CPU time consumed, scaled by nice
weight (lower nice ⇒ slower vruntime growth). All runnable threads sit in a red-black tree keyed by
vruntime; the scheduler picks the **leftmost** (least vruntime), runs it for a slice derived from the
runnable count (target latency / n), then its advanced vruntime reinserts it rightward. No fixed quantum;
fairness emerges from "always run whoever's run least." Interactive/I-O-bound threads sleep often, keep
low vruntime, so they get scheduled promptly on wake.

**Q4 (Quadeye — RT theory).** *Explain Rate Monotonic vs EDF, and which maps to a Linux policy.*
**Model answer:** Both are RT algorithms. **Rate Monotonic**: static priority by period — shorter period
= higher priority, assigned once; optimal among fixed-priority schemes, guaranteed only to ~69%
utilisation. **EDF**: dynamic — always run the task with the earliest absolute deadline; optimal to 100%
utilisation but more overhead and worse overload behaviour. Linux's `SCHED_DEADLINE` *is* EDF (with
constant-bandwidth admission control); `SCHED_FIFO`/`SCHED_RR` are the fixed-priority vehicles you'd use
for an RM-style assignment. HFT usually uses plain `SCHED_FIFO`, since the hot path is event-driven, not
strictly periodic.

**Q5 (IMC — affinity).** *Why pin threads to cores? Walk through the cost of not doing it.*
**Model answer:** Each core has private L1/L2. A pinned thread keeps its working set hot in that core's
caches. Without pinning, the load balancer can migrate the thread to another core, where its caches are
**cold** — it eats a storm of L1/L2 misses re-fetching from L3 (~40 cyc) / DRAM (~200+ cyc) for thousands
of cycles, plus TLB cost — and crucially this happens at an *unpredictable* time, so it shows up as tail
jitter. Pinning (`sched_setaffinity`/`taskset`) eliminates migration; add `isolcpus` so the balancer
won't place other work there either.

**Q6 (Jump — isolation stack).** *Beyond pinning, what boot/config changes make a core "quiet" for a hot
thread?*
**Model answer:** `isolcpus=<core>` to pull it out of load-balancing domains; `nohz_full=<core>` for
tickless operation (drop the ~1000 Hz scheduler tick when one thread is runnable); `rcu_nocbs=<core>` to
move RCU callbacks off it; IRQ affinity to route device/timer interrupts to other cores (Module 09);
`SCHED_FIFO` + pin for the thread itself; and disabling RT throttling for that core so an intentional
busy-poll isn't clawed back. The goal: the scheduler, load balancer, timer, and interrupts never touch
that core.

**Q7 (HRT — the trap).** *Someone sets a busy-looping thread to `SCHED_FIFO` 99 on a non-isolated
single-socket box and it hangs. What happened?*
**Model answer:** A FIFO thread yields only to strictly higher RT priority; busy-looping, it never blocks,
so it starves every lower-priority thread on its core — including kernel housekeeping threads. On a shared
core that wedges things. Linux's RT throttle (950 ms/1 s default) is the safety valve that forces 50 ms/s
back to others, which itself shows as a periodic stall. Correct setup: isolate the core so there's nothing
to starve, pin the thread there, and disable the throttle for it.

**Q8 (Squarepoint — determinism).** *Your average latency is great but p99.99 is terrible. Scheduling
angle?*
**Model answer:** Average being good means the fast path works; the tail means something *occasionally*
preempts or migrates the thread — the scheduler tick, an interrupt, the load balancer, RT throttling, or a
page fault / NUMA miss. The scheduling fixes: `SCHED_FIFO` + pin to an isolated, tickless core with IRQs
and RCU routed away. Real-time scheduling is about killing that tail (jitter), not improving the mean.

---

## Key takeaways

- The scheduler answers *which runnable thread runs next, and for how long*; it's invoked on timer ticks,
  blocks, wakes, exits, and yields. **Scheduler picks, dispatcher switches** (Module 10).
- Classic algorithms: FCFS, SJF/SRTF, **round-robin** (quantum-based fairness), **priority** (static or
  dynamic). Dynamic priority historically boosted I/O-bound and penalised CPU-bound tasks (`sleep_avg`);
  modern Linux achieves the same via CFS/vruntime.
- Linux internal priority **0–139, lower = higher**: **0–99 real-time**, **100–139 normal/CFS** via
  **nice −20..+19** (`prio = 120 + nice`, default 120). Nice is a CFS *weight*; RT priority (1–99 via
  `chrt`) is a separate absolute axis.
- Scheduling **classes** in strict order: `SCHED_DEADLINE` (EDF) > `SCHED_FIFO`/`SCHED_RR` (fixed RT prio,
  FIFO has no slice) > **CFS** (`SCHED_NORMAL`, red-black tree keyed by vruntime, no fixed quantum). A
  runnable RT thread always beats every normal thread.
- **Real-time = deterministic, not fast.** Jitter (variance), not mean, is the HFT metric. **Rate
  Monotonic** = static priority by period (≤~69% util); **EDF** = dynamic by deadline (≤100% util);
  `SCHED_DEADLINE` ≈ EDF, but HFT usually runs `SCHED_FIFO` for the event-driven hot thread.
- **CPU affinity** (`sched_setaffinity`/`taskset`) keeps the working set hot in a core's private L1/L2;
  migration ⇒ cold-cache miss storm ⇒ tail jitter.
- HFT makes scheduling a **non-event** on the hot core: `isolcpus` (no load balancing) + `nohz_full`
  (no tick) + `rcu_nocbs` + IRQ affinity + pinned `SCHED_FIFO`. The scheduler and load balancer never
  touch it; the thread busy-polls, never blocks, never migrates, never gets preempted.

**Next:** [14 — Deadlocks, starvation & livelock](14-deadlocks-starvation-livelock.md)
