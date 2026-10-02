# Module 14 — Deadlocks, starvation & livelock

Module 11 introduced the primitives — mutex, condvar, semaphore — and gave deadlock a one-screen
treatment (the four Coffman conditions, lock ordering, `std::scoped_lock`). This module is the deep
dive the interviewer actually drills into: *how do you detect, prevent, avoid, and recover from a
deadlock; what is starvation and how is it different; what is priority inversion and how was it fixed
on an actual Mars lander; and what is livelock — the bug where every thread is busy and nothing gets
done.* These are **liveness** failures: the system doesn't crash or corrupt, it simply stops making
progress. In HFT the hot path sidesteps all of them by owning its data single-threaded (Module 11
§10), but the control plane — OMS, risk engine, config reload, order-state machines shared across
threads — is exactly where they bite, and where interviewers probe.

---

## 1. What a deadlock is

A **deadlock** is a set of two or more threads, each **blocked waiting for a resource that another
member of the set holds**, so none of them can ever proceed. It is permanent: without outside
intervention (a timeout, a kill, a reboot) the set is stuck forever.

The canonical example is two threads and two locks acquired in opposite order:

```cpp
// Thread 1                     // Thread 2
lock(A);                        lock(B);
    lock(B);   // ← blocks          lock(A);   // ← blocks
    ...                             ...
    unlock(B);                      unlock(A);
unlock(A);                      unlock(B);
```

Walk the timeline — the deadlock needs a specific interleaving, which is why it's intermittent and
survives testing:

```
 time │ Thread 1              Thread 2
 ─────┼───────────────────────────────────────────
   t0 │ lock(A)  ✓
   t1 │                       lock(B)  ✓
   t2 │ lock(B)  … blocks (T2 holds B)
   t3 │                       lock(A)  … blocks (T1 holds A)
   t4 │ ─────────── both blocked forever ──────────
```

Draw the "who-waits-for-whom" graph and the problem is a **cycle**:

```
        holds              wants
   A ───────────▶ T1 ───────────▶ B
   ▲                               │
   │ wants                   holds │
   └────────────── T2 ◀────────────┘

   T1 → B → T2 → A → T1   (a cycle ⇒ deadlock)
```

If the wait graph has no cycle, there is no deadlock; every cycle among single-instance resources
*is* a deadlock. (With multiple instances of a resource, a cycle is *necessary but not sufficient* —
§3.)

---

## 2. The four Coffman conditions — and how to break each

Deadlock requires **all four** of these to hold **simultaneously**. That's the lever: negate *any one*
and deadlock becomes impossible. (Module 11 lists them; here is each with its break.)

| # | Condition | Meaning | How to break it |
|---|-----------|---------|-----------------|
| 1 | **Mutual exclusion** | A resource is non-shareable — only one holder at a time | Make it shareable / read-only (e.g. immutable config read by all) |
| 2 | **Hold and wait** | A thread holds ≥1 resource while blocking for another | Acquire *all* needed resources at once, or hold none while requesting |
| 3 | **No preemption** | A held resource can't be forcibly taken back | Allow preemption + rollback (take the lock, roll the victim back) |
| 4 | **Circular wait** | A cycle exists in the wait-for graph | **Total ordering** of resources: always acquire in ascending order |

```
   ┌─ Mutual exclusion ─┐   ┌─ Hold & wait ─┐   ┌─ No preemption ─┐   ┌─ Circular wait ─┐
   │  only-one-holder   │ + │ keep + ask     │ + │ can't take back │ + │  cycle in graph │  = DEADLOCK
   └────────────────────┘   └────────────────┘   └─────────────────┘   └─────────────────┘
        break any single box  ⇒  no deadlock possible
```

The three on the left are usually **intrinsic to the resource** — a lock is mutually exclusive by
definition, you can't make a counter "shareable for writes," and most locks can't be yanked back
safely. So in practice the engineering lever is almost always **#4, circular wait**, killed by a
**global lock-ordering discipline** (§4). That's why Module 11's advice reduces to "acquire locks in a
fixed order."

---

## 3. Detection — the Resource Allocation Graph & the detection algorithm

If you don't prevent deadlock, you can **let it happen and detect it**. The model is the **Resource
Allocation Graph (RAG)**:

```
   ○  process (thread)          □  resource type
   □ ──▶ ○   assignment edge  (resource is held BY the process)
   ○ ──▶ □   request edge     (process is WAITING for the resource)
```

Example of a deadlocked RAG (the §1 two-lock case):

```
        A □ ──────▶ ○ T1 ──────▶ □ B
          ▲                        │
          │                        ▼
          └──────── ○ T2 ◀─────────┘
   edges:  A→T1 (held)   T1→B (want)   B→T2 (held)   T2→A (want)   ⇒ cycle
```

**Single instance per resource:** a cycle in the RAG ⇔ deadlock. Just run cycle detection (DFS,
colour nodes) on the wait-for graph.

**Multiple instances per resource** (e.g. a semaphore with N permits, a pool of 5 connections): a
cycle is *necessary but not sufficient* — some holder outside the cycle might release an instance and
unblock it. You need a reduction / **detection algorithm** (Banker-style, work/finish vectors):

```
  Available[]           # free instances of each resource type
  Allocation[i][]       # what thread i currently holds
  Request[i][]          # what thread i is currently waiting for

  Work  = Available                       # what we can hand out right now
  Finish[i] = (Allocation[i] == 0)        # threads holding nothing can't be deadlocked

  loop:
    find an i with Finish[i]==false AND Request[i] <= Work      # its ask can be met now
    if found:
        Work += Allocation[i]             # pretend it finishes and frees everything
        Finish[i] = true
        repeat
    else:
        break

  # any thread with Finish[i]==false is deadlocked.
```

The intuition: optimistically assume every thread *whose current request can be satisfied* will run,
finish, and release. Sweep until no more can run. Whoever's left stranded is in the deadlock set. Run
it periodically or when a wait exceeds a threshold. HFT control-plane systems rarely run a formal
detector — they use **watchdog timeouts** (§9) as a cheap, practical stand-in: "if this lock isn't
acquired in N ms, something is wrong, log + abort."

---

## 4. Prevention — negate a condition by construction

**Prevention** means designing so a Coffman condition can *never* hold — a static guarantee, no
runtime checks.

**Break hold-and-wait — request everything at once.** A thread acquires *all* locks it will need in
one atomic step, or none:

```cpp
// no hold-and-wait: grab both or neither (scoped_lock orders internally)
std::scoped_lock lk{mtxA, mtxB};   // acquires both deadlock-free, or blocks until it can
```

Downside: you must know the full lock set up front, and you hold locks longer (worse concurrency).

**Break no-preemption — allow rollback.** If a thread can't get a lock, it releases everything it
holds and retries. `try_lock` with back-off is the practical form (§9). Downside: wasted work on
rollback, risk of livelock if everyone keeps retrying in lockstep.

**Break circular wait — total resource ordering.** The workhorse. Assign every lock a global rank and
**always acquire in ascending rank order**:

```cpp
// Global order: account_lock(1) < position_lock(2) < order_lock(3)
// EVERY code path acquires low→high. A cycle is now impossible:
// no thread can hold a higher-ranked lock while waiting on a lower-ranked one.
std::lock_guard a{account_lock};     // rank 1
std::lock_guard p{position_lock};    // rank 2   ✓ ascending
```

Why it works: a cycle requires some thread holding a *higher* lock while waiting for a *lower* one —
impossible if everyone goes low→high. This is the single most important deadlock-prevention technique
in real multi-lock codebases, and the answer interviewers want. Address-ordering (`&m1 < &m2`) is a
cheap way to get a total order when locks have no natural rank.

> **HFT framing:** *multiple trading algorithms competing for shared market-data feeds* is the
> textbook circular-wait setup — strat X holds feed-A's lock and wants feed-B's while strat Y holds B
> and wants A. The fix is the same two tools: a **total ordering on the feed locks** (every strat
> locks feeds in the same canonical order) plus **timeouts** so a stuck acquisition is detected and
> backed out rather than hanging the engine.

---

## 5. Avoidance — the Banker's algorithm

**Avoidance** is softer than prevention: it allows the Coffman conditions but **refuses any individual
allocation that would move the system into an unsafe state.** It needs each thread to declare its
**maximum** resource claim up front.

- **Safe state** — there exists at least one ordering of the threads (a "safe sequence") in which each
  can get its maximum, run, and release, so *all* finish. No deadlock is reachable.
- **Unsafe state** — no such sequence is guaranteed. Unsafe ≠ deadlocked, but it *may* lead there, so
  avoidance forbids entering it.

The **Banker's algorithm** (Dijkstra): on each request, pretend to grant it, then check the result is
still safe; if not, make the requester wait.

```
  Need[i] = Max[i] - Allocation[i]      # what i could still demand

  isSafeState():
      Work = Available
      Finish[i] = false for all i
      loop:
        find i with Finish[i]==false AND Need[i] <= Work   # can finish with what's free
        if none: break
        Work += Allocation[i]; Finish[i] = true            # it finishes, frees its holdings
      return all Finish[i] == true                         # everyone could finish ⇒ safe

  requestResource(i, req):
      if req > Need[i]:        error   # asked beyond declared max
      if req > Available:      wait     # not enough free right now
      # tentatively allocate
      Available   -= req;  Allocation[i] += req;  Need[i] -= req
      if isSafeState():      grant
      else:                  rollback the tentative allocation and make i wait
```

Honest take: the Banker's algorithm is **textbook, almost never used in production** — it needs
every thread's maximum claim in advance (unrealistic), and the safety check is O(threads² ×
resources) per request. Real systems prevent (lock ordering) or detect-and-recover. But it's a
**standard interview question**, so know safe-vs-unsafe and be able to run `isSafeState` on a small
matrix by hand.

---

## 6. Recovery — once you're already deadlocked

Detection (§3) tells you a set is stuck. Two ways out:

**Process/thread termination:**

- **Abort all** deadlocked threads — fast, simple, maximally destructive (lose all their work).
- **Abort one at a time**, re-running detection after each kill, until the cycle breaks — less waste,
  but repeated detection passes.
- **Minimum-cost victim** — kill the one cheapest to restart (least work done, lowest priority,
  fewest resources held).

**Resource preemption:**

- **Select a victim** holding a needed resource and forcibly take it.
- **Rollback** that victim to a safe checkpoint (it can't just continue mid-operation with a resource
  yanked away — hence you need checkpointing or an idempotent operation).
- **Avoid starvation** — don't pick the *same* victim every time, or it never completes (ties into
  §7); factor the number of prior rollbacks into victim selection.

In an HFT control plane you rarely do formal preemption/rollback — the practical "recovery" is a
**watchdog that kills and restarts** the stuck component, which is why those components are built to
restart cleanly (state in a journal / recoverable from the exchange).

---

## 7. Starvation — and how it differs from deadlock

**Starvation** is a thread being **indefinitely denied** a resource because the policy keeps favouring
others — it's not blocked in a cycle, it's just always last in line.

- **CPU starvation** — a low-priority thread never scheduled because higher-priority threads are
  always runnable (a strict-priority scheduler problem — Module 13).
- **Resource starvation** — e.g. a writer waiting on a reader-writer lock while a steady stream of
  readers keeps the lock in shared mode forever (writer starvation, flagged in Module 11 §8).

The distinction interviewers want:

| | **Deadlock** | **Starvation** |
|---|---|---|
| Thread state | **Blocked** (waiting in a cycle) | **Ready/runnable**, just never chosen |
| The resource | **Held** by another in the set | **Available**, but keeps going to others |
| Duration | **Permanent** (no external help) | **Indefinite**, but *could* resolve any time |
| Cause | Circular dependency | Unfair scheduling / allocation policy |
| Fix | Break a Coffman condition | **Aging**, fair policy |

**Fixes:**

- **Aging** — increase a waiter's effective priority the longer it waits, so eventually it outranks
  everyone and must be picked. (Linux's CFS achieves the same end via `vruntime` fairness — Module 13.)
- **Fair scheduling / FIFO fairness** — a `fair`/`phase-fair` reader-writer lock blocks new readers
  once a writer is waiting, bounding writer wait.

Starvation is the "softer" liveness bug: the system *can* recover on its own, a deadlock cannot.

---

## 8. Priority inversion — and the Mars Pathfinder story

**Priority inversion**: a *high*-priority thread is blocked on a lock held by a *low*-priority thread,
and a *medium*-priority thread (which needs no lock) preempts the low one — so the medium thread,
indirectly, runs ahead of the high one. The high-priority thread is stuck behind a lock its holder
can't release because the holder isn't being scheduled.

```
  H (high)  ── wants lock L ──▶ blocked …………………………… (waiting on Low)
  M (med)   ── needs no lock ─▶ RUNS, preempts Low  ← inversion: M effectively outranks H
  L (low)   ── holds lock L  ─▶ preempted by M, can't release L
```

**Fixes:**

- **Priority inheritance** — while L holds a lock that H wants, L **temporarily inherits H's
  priority**, so M can't preempt it; L runs, releases, drops back. (`PTHREAD_PRIO_INHERIT` mutex
  attribute.)
- **Priority ceiling** — each lock has a ceiling = the highest priority of any thread that can take
  it; a thread holding it runs at that ceiling. Prevents inversion *and* bounds it, and prevents
  deadlock among ceiling-ordered locks.
- **Immediate (ceiling) inheritance** — raise to the ceiling the instant the lock is taken, not only
  when a higher thread blocks.

**The Mars Pathfinder bug (1997)** — the lander kept **resetting on Mars**. A high-priority bus-
management task shared a mutex with a low-priority meteorological task; a busy medium-priority
communications task kept preempting the low task while it held the mutex, so the high-priority task
missed its deadline and a watchdog **reset the system**. VxWorks *had* priority inheritance available
but it was disabled; JPL reproduced it on the ground, uploaded a patch flipping the flag on for that
mutex, and the resets stopped. The canonical real-world lesson: on a real-time system, **enable
priority inheritance on any lock a high-priority thread can wait for.** (Module 11 §9 names this bug;
this is the full story.)

---

## 9. Livelock — busy, but going nowhere

A **livelock** is the sneaky cousin of deadlock: the threads are **not blocked** — they're actively
running and *changing state* — but their state changes cancel each other out so **no real progress is
made.** CPU usage is high (unlike deadlock, where blocked threads use none).

The classic picture: two people meet in a corridor, both step the same way to let the other pass, both
step back, both step the same way again — politely, forever. In code it's a release-and-retry loop
reacting to each other:

```cpp
// Both threads try to be "polite" to avoid deadlock — and livelock instead.
while (!done) {
    lock(A);
    if (!try_lock(B)) {    // can't get B
        unlock(A);         // politely back off…
        continue;          // …and immediately retry — in lockstep with the other thread
    }
    do_work();
    unlock(B); unlock(A); done = true;
}
```

Both grab A, both fail to get B, both release A, both retry at the same instant — repeat. No cycle, so
a deadlock detector finds nothing; everyone's "running," so it's **harder to spot than deadlock.**

| | **Deadlock** | **Livelock** |
|---|---|---|
| Thread state | **Blocked** / static | **Running** / state constantly changing |
| CPU usage | ~none (threads asleep) | **High** (threads spinning/retrying) |
| Detection | Cycle in wait-graph — detectable | No cycle — **harder** to detect |
| Trigger | Opposite-order acquisition | Overly-polite back-off in lockstep |

**Fixes — break the symmetry:**

- **Randomized back-off** — each thread waits a *random* interval before retrying, so they stop
  colliding in lockstep.
- **Exponential back-off** — grow the wait after each failure, capped: `delay = min(delay * 2,
  maxDelay)`. Standard in networking (Ethernet CSMA/CD), lock retries, and exchange reconnect logic.
- **Timeouts** — bound the retry loop; after N attempts, give up and escalate (log, abort, alert)
  rather than spin forever.
- **Asymmetry / priority** — give threads different ranks so one yields and the other proceeds
  (equivalent to the lock-ordering fix for deadlock).

---

## 10. Best practices & the HFT stance

**Control-plane discipline (where locks live):**

- **Global lock hierarchy** — rank every lock, acquire ascending (§4). Document and enforce it; some
  shops add a debug-build checker that asserts acquisition order.
- **Avoid nested locks** — hold one lock at a time where possible; the fewer locks held at once, the
  fewer cycles can form.
- **Timeouts over indefinite waits** — `try_lock_for`; a stuck acquisition should be *detected*, not
  silent. Pair with watchdogs that kill+restart.
- **Priority inheritance** on any lock a latency-critical thread can block on (§8), and **aging / fair
  locks** to bound starvation (§7).
- **Monitor** — track lock wait times and contention; a creeping p99 lock-wait is the early warning of
  a liveness problem.

**The hot path (where the real answer is: no locks at all).** Everything above manages liveness bugs;
HFT's tick-to-trade path **eliminates the possibility** by removing shared mutable state (Module 11
§10):

- **Single-threaded ownership** — the matching engine owns its data; with no shared lock there is no
  deadlock, no priority inversion, no livelock on that path.
- **Lock-free SPSC queues** — threads hand data off without locks, so no cycle can form and latency
  stays bounded ([`../cpp-guide/18`](../cpp-guide/18-atomics-lockfree.md)).
- **Immutability / `thread_local`** — data never shared-and-mutated needs no coordination.

The interview summary: *deadlock, starvation, priority inversion and livelock are all liveness bugs of
**shared contended state**; the control plane manages them with lock ordering, timeouts, priority
inheritance and aging; the hot path dodges them entirely by not sharing mutable state.*

---

## Common pitfalls / misconceptions

- **"A livelock is just a deadlock."** No — deadlocked threads are **blocked** (CPU idle); livelocked
  threads are **running** and burning CPU while making no progress. A deadlock detector (cycle search)
  misses livelock entirely.
- **"Starvation and deadlock are the same."** No — a starved thread is **runnable**, the resource is
  **available**, and it *could* get it any moment; a deadlocked thread is **blocked** on a held
  resource in a cycle, permanently.
- **"Unsafe state means deadlocked."** No — unsafe (Banker's) means *no guaranteed* safe sequence
  exists; the system may still get lucky. Deadlock is the actual stuck state.
- **"`try_lock` + retry eliminates deadlock, full stop."** It removes the *blocking* deadlock but can
  introduce **livelock** if everyone backs off and retries in lockstep. You need randomized/exponential
  back-off too.
- **"Priority inheritance is on by default."** It is **not** — it's a mutex attribute you must enable
  (`PTHREAD_PRIO_INHERIT`). Mars Pathfinder shipped with it available but off.
- **"Lock ordering fixes all deadlocks."** It kills circular-wait deadlocks among *locks you control
  and can rank*. It doesn't help with resources you can't order (e.g. acquiring locks in callback order
  dictated by external code) — there you need `scoped_lock`/try-lock.

---

## Quiz — tough problems

**Q1.** T1 does `lock(A); lock(B);`, T2 does `lock(B); lock(A);`. Give the exact interleaving that
deadlocks, name the Coffman condition exploited, and give the one-line structural fix.

**Answer:** Interleaving: T1 `lock(A)` ✓, T2 `lock(B)` ✓, T1 `lock(B)` blocks (T2 holds B), T2
`lock(A)` blocks (T1 holds A) → both stuck. Exploits **circular wait**. Fix: **global lock ordering** —
both acquire A before B (or `std::scoped_lock{A, B}`, which orders internally).

---

**Q2.** A deadlock detector reports no cycle, yet two threads spin at 100% CPU and never finish. What's
the bug and why did the detector miss it?

**Answer:** **Livelock.** The detector looks for a cycle in the *wait-for* graph, but livelocked
threads aren't *waiting* — they're running and repeatedly changing state (lock → fail → release →
retry) in lockstep. No blocked-wait edge, so no cycle. Fix: randomized/exponential back-off to break
the symmetry.

---

**Q3.** Why is "unsafe state" not the same as "deadlock"?

**Answer:** Unsafe (Banker's) means there is **no guaranteed safe sequence** in which every thread can
reach its declared maximum and finish — it *might* lead to deadlock depending on future requests.
Deadlock is the realized stuck state. Avoidance refuses to enter unsafe states precisely because they
*risk* deadlock, not because they *are* deadlock.

---

**Q4.** Run the detection algorithm. Available = [0], three threads, Allocation = [1, 1, 0], Request =
[0, 1, 1]. Deadlocked?

**Answer:** Work = [0]. T1 holds 1, requests 0 ≤ Work → T1 can finish: Work += 1 → [1]. Now T2 requests
1 ≤ [1] → finishes: Work += 1 → [2]. T3 requests 1 ≤ [2] → finishes. All finish → **no deadlock**. The
key move: a thread requesting nothing (or whose request fits what's free) finishes and frees its
holdings, unblocking others.

---

**Q5.** A high-priority thread misses its deadline whenever a particular medium-priority thread is
busy, even though neither touches the other directly. Diagnose and fix.

**Answer:** **Priority inversion.** A *low*-priority thread holds a lock the high thread needs; the
medium thread (needing no lock) preempts the low holder, so the lock is never released and the high
thread stalls behind medium work. Fix: **priority inheritance** — the low holder temporarily runs at
the high thread's priority so medium can't preempt it (the Mars Pathfinder fix).

---

**Q6.** You replace blocking `lock()` with `try_lock()`-and-release-on-failure to "fix a deadlock."
Tests pass but production occasionally pins two cores at 100% with no progress. What happened?

**Answer:** You traded deadlock for **livelock.** Both threads grab the first lock, fail the second,
release, and retry at the same cadence — forever, busy. Add **randomized or exponential back-off**
(`delay = min(delay*2, max)`) and/or a retry **timeout** so the symmetry breaks and a stuck loop
escalates.

---

**Q7.** Writers on a `std::shared_mutex` never acquire it under steady read traffic. Name the problem
and two fixes.

**Answer:** **Writer starvation** — a continuous stream of shared-mode readers keeps the lock from ever
being free for the exclusive writer. Fixes: (1) a **fair / write-preferring** reader-writer lock that
blocks *new* readers once a writer is queued; (2) **aging** the writer's priority; or structurally, an
**atomic pointer swap to an immutable snapshot** (RCU-style) so writers never block readers at all.

---

**Q8.** Why is the Banker's algorithm almost never used in real low-latency systems, despite being a
standard interview topic?

**Answer:** It requires every thread to **declare its maximum resource claim in advance** (usually
unknowable), and the safety check is **O(threads² × resources) on every request** — unacceptable
overhead on a hot path. Real systems **prevent** via lock ordering or **detect-and-recover** via
timeouts/watchdogs. Know it to answer the question, not to deploy it.

---

## Indian HFT interview questions

**Q1 (Tower Research, Gurgaon — the staple).** *Explain deadlock and the four conditions, then tell me
how you'd prevent it in a large codebase with many locks.*
**Model answer:** Deadlock = a set of threads each blocked on a resource another in the set holds,
forming a cycle; it needs all four Coffman conditions — mutual exclusion, hold-and-wait, no preemption,
circular wait — simultaneously, so breaking any one prevents it. In practice the first three are
intrinsic to locks, so you kill **circular wait** with a **global lock ordering**: rank every lock,
always acquire ascending (`std::scoped_lock` for multi-lock sites, which orders internally). Add
`try_lock`-with-timeout and avoid nested locks for the paths you can't statically order.

**Q2 (Optiver — liveness taxonomy).** *What's the difference between deadlock, livelock, and
starvation?*
**Model answer:** **Deadlock** — threads blocked in a cycle, CPU idle, permanent. **Livelock** —
threads running and changing state (e.g. retry-and-back-off in lockstep) but making no progress, CPU
pinned, no cycle so a detector misses it. **Starvation** — a runnable thread indefinitely denied a
resource that *is* available because the policy keeps favouring others; it can resolve on its own,
fixed by aging/fair scheduling. Deadlock = structural cycle; starvation = unfair policy; livelock =
busy non-progress.

**Q3 (Graviton — real-time war story).** *What is priority inversion and how do you fix it?*
**Model answer:** A high-priority thread blocks on a lock held by a low-priority thread, and a
medium-priority thread preempts the low holder so the lock is never released — the high thread stalls
behind medium work. Fix with **priority inheritance** (the holder temporarily runs at the waiter's
priority) or **priority ceiling** (holder runs at the lock's ceiling). This is the **Mars Pathfinder**
bug — resets on Mars from a mutex with inheritance disabled; JPL patched the flag on remotely. On an
RT trading box, enable `PTHREAD_PRIO_INHERIT` on any lock a hot thread can wait on.

**Q4 (Quadeye — the practical defense).** *Two of your strategies both consume feed A and feed B and
occasionally the whole engine hangs. What's happening and how do you fix it?*
**Model answer:** Classic **circular-wait deadlock** — strat 1 holds A's lock and wants B's; strat 2
holds B's and wants A's. Fix: impose a **canonical ordering on the feed locks** so every strategy
acquires feeds in the same order (A before B everywhere), eliminating the cycle; back it with
**acquisition timeouts** so a stuck grab is detected and backed out rather than hanging the engine.
Better still, don't share the feeds behind locks — fan out via **lock-free SPSC queues** so each strat
reads its own copy.

**Q5 (IMC — detection vs prevention).** *When would you detect-and-recover from deadlock instead of
preventing it?*
**Model answer:** Prevention (lock ordering) is cheap and static, so it's the default. You
detect-and-recover when you *can't* impose an order — e.g. locks acquired in an order dictated by
external callbacks or plugin code. Then run a periodic/ threshold-triggered **detection pass**
(work/finish reduction over the allocation graph) or, pragmatically, a **watchdog timeout** that
kills+restarts the stuck component. That's why control-plane components are built to restart cleanly
with recoverable state.

**Q6 (HRT — Banker's).** *Walk me through the Banker's algorithm and tell me if you'd use it.*
**Model answer:** Each thread declares a **max** claim; `Need = Max − Allocation`. On a request,
tentatively allocate, then run `isSafeState`: repeatedly find a thread whose `Need ≤ Work`, assume it
finishes and add its `Allocation` back to `Work`; if all finish, the state is **safe** and you grant,
else roll back and make it wait. I **wouldn't deploy it** — it needs maximum claims up front and is
O(n²·m) per request. It's a conceptual tool; production uses prevention or detection.

**Q7 (Squarepoint — subtle).** *You "fixed" a deadlock with try-lock and back-off. What new failure
mode did you just risk, and how do you close it?*
**Model answer:** **Livelock** — if both threads back off and retry in lockstep they spin forever at
100% CPU with no cycle for a detector to find. Close it with **randomized / exponential back-off**
(`delay = min(delay*2, maxDelay)`) to break symmetry, plus a **retry cap/timeout** that escalates
(log/abort/alert) instead of spinning indefinitely.

**Q8 (AlphaGrep — starvation on the control plane).** *Your config-reload writer never gets the lock
under load. Why, and what do you change?*
**Model answer:** **Writer starvation** on a reader-writer lock — a steady stream of readers keeps it
in shared mode so the exclusive writer never runs. Switch to a **write-preferring/fair** `shared_mutex`
(new readers block once a writer waits), or avoid the lock entirely with an **atomic swap to an
immutable config snapshot** (RCU-style) so readers never block and the writer publishes with one atomic
store.

---

## Key takeaways

- **Deadlock** = threads blocked in a cycle, each holding what another needs; needs **all four Coffman
  conditions** (mutual exclusion, hold-and-wait, no preemption, circular wait) — break any one.
- The practical lever is **circular wait**, killed by a **global lock ordering** (acquire ascending);
  `std::scoped_lock` orders multi-lock sites internally (Module 11).
- **Detection**: single-instance → cycle in the Resource Allocation Graph; multi-instance → work/finish
  reduction. In practice, **timeouts/watchdogs** stand in for formal detectors.
- **Banker's algorithm** = avoidance via safe-vs-unsafe states; textbook and interview-standard but
  almost never deployed (needs max claims up front, O(n²·m) per request).
- **Starvation** ≠ deadlock: a runnable thread indefinitely denied an *available* resource by unfair
  policy; fixed by **aging** / fair scheduling. It can resolve on its own; deadlock can't.
- **Priority inversion**: high blocked behind a low holder preempted by medium work; fix with
  **priority inheritance** / **ceiling** — the **Mars Pathfinder** lesson.
- **Livelock**: threads running and changing state but making no progress, high CPU, no cycle (so
  detectors miss it); break symmetry with **randomized / exponential back-off** and timeouts.
- HFT's hot path **avoids all of these** by removing shared mutable state — single-threaded ownership,
  lock-free SPSC queues, immutability. Liveness bugs live in the **control plane**, where lock ordering,
  timeouts, priority inheritance, and aging keep them in check.

**Next:** [Module 15 — IPC fundamentals](15-ipc-fundamentals.md)
