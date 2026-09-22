# Module 11 — Threads & synchronization

A thread is the OS's unit of *execution* — the thing the scheduler actually runs — while the process
(Module 10) is the container of memory and resources it runs inside. Because threads of one process
**share memory**, they're cheap to switch between and easy to make cooperate, but that same sharing is
the source of every data race and deadlock. This module covers what a thread is at the kernel level,
its lifecycle (`join`/`detach` and friends), and how synchronization primitives — especially the
mutex — actually work underneath. The C++-API angle lives in
[`../../guide-os-net/02`](../../guide-os-net/02-multithreading.md) and
[`../../guide-os-net/03`](../../guide-os-net/03-cross-thread-safety.md); the lock-free/atomic side is in
[`../cpp-guide/16`](../cpp-guide/16-atomics-lockfree.md). Here we go under the API to the kernel mechanism.

---

## 1. Threads vs processes

A process owns an address space; a thread runs inside it. All threads of a process **share** the code,
heap, global data, and open files, but each thread has its **own**:

- **stack** (local variables, call frames — reserved ~1–8 MB each),
- **registers** (including its own PC and SP),
- **thread-local storage** (`thread_local` / TLS),
- kernel stack and saved context (Module 10).

```
Process address space (one page table, shared by all threads)
┌──────────────────────────────────────────────────────────┐
│  code  │  globals/data  │        heap  (shared)           │
├──────────────────────────────────────────────────────────┤
│  Thread 1 stack   Thread 2 stack   Thread 3 stack   ...    │  ← each thread: own stack + regs + TLS
└──────────────────────────────────────────────────────────┘
        shared ▲ (races possible)        private ▲ (no sync needed)
```

The consequence is the entire subject: **anything on a thread's own stack or in TLS needs no
synchronization** (nobody else can name it), while **anything shared — the heap, globals — is a race
waiting to happen** the moment two threads touch it and one writes.

---

## 2. User-level vs kernel-level threads; the 1:1 model

Historically there were two ways to implement threads:

- **Kernel-level (1:1):** each user thread maps to one kernel-schedulable entity. The kernel sees and
  schedules every thread. This is what **Linux does** — `std::thread`/`pthread_create` calls `clone()`
  with flags to share the address space, producing a new `task_struct`. Pro: true parallelism, one
  thread blocking doesn't block the others. Con: creation and switching go through the kernel.
- **User-level (N:1):** a userspace library multiplexes many "green" threads onto one kernel thread.
  Cheap to create/switch (no kernel), but one blocking syscall stalls *all* of them, and they can't use
  multiple cores. (M:N hybrids exist — Go's goroutines are the modern example — but the OS underneath
  is still 1:1.)

For systems/HFT interviews, know that **Linux threads are 1:1 kernel threads created via `clone()`**,
and that a thread switch is a context switch that skips the CR3/TLB reload (Module 10 §7).

---

## 3. The thread lifecycle and thread functions

A thread is *created* running a function, executes, and *terminates* when that function returns. But
its OS resources (kernel stack, `task_struct`, exit status) aren't reclaimed until it's **joined** or
it was **detached**. This is the single most important lifecycle rule.

```
 create ──▶ runnable ⇄ running ⇄ blocked (I/O, lock, condvar) ──▶ terminated ──▶ joined/reaped
   │                                                                   │
 std::thread t{fn,args}                                        t.join() reclaims it
```

**`join()`** — block the calling thread until the target finishes, then reclaim its resources. Gives
you a synchronization point ("wait here until that work is done"). Each thread can be joined exactly
once.

**`detach()`** — sever the handle; let the thread run independently. Its resources are auto-reclaimed
when it exits. You **can no longer join it** and lose any way to know when it finished. Use for
fire-and-forget background work whose completion you don't need to observe.

**The classic bug:** a `std::thread` that is destroyed while still **joinable** (neither joined nor
detached) calls `std::terminate()` — the whole process aborts. C++ made this loud on purpose: you
*must* decide join-or-detach for every thread. Hence `std::jthread` (C++20), which joins automatically
in its destructor (RAII — the "join or terminate" trap disappears). Prefer `jthread`.

```cpp
std::thread t{work};
// ... if t goes out of scope here without join()/detach() → std::terminate()
t.join();   // or t.detach();   — one of these is mandatory
```

Other useful thread functions (know what they do):

| Function | Purpose |
|---|---|
| `joinable()` | Is the thread still associated with a running/finished thread of execution? |
| `get_id()` | The thread's unique ID (for logging, maps) |
| `native_handle()` | The underlying `pthread_t` — needed to set affinity/priority (HFT!) |
| `std::this_thread::yield()` | Hint the scheduler to run someone else now |
| `std::this_thread::sleep_for()` | Block this thread for a duration (a WAITING state, Module 10) |
| `std::thread::hardware_concurrency()` | Hint: number of hardware threads (for sizing pools) |
| `pthread_setaffinity_np` / `sched_setaffinity` | **Pin** the thread to a core (via `native_handle`) — the HFT staple |

---

## 4. The core hazard: race conditions & critical sections

A **race condition** exists when the result depends on the timing/interleaving of threads. A **data
race** — two threads access the same memory, at least one writing, with no synchronization — is
**undefined behavior** in C++ (not "a wrong number," but anything: torn reads, compiler miscompiles).

```
counter++  is really:   load counter → add 1 → store counter
Thread A: load(0) .............. store(1)
Thread B: ....... load(0) → +1 → store(1)          result = 1, not 2  (lost update)
```

The region that must run without interference is the **critical section**. Synchronization primitives
exist to enforce that only one thread (or a controlled number) is in it at a time.

---

## 5. How a mutex actually works (fast path / slow path)

This is the interview favorite: *"what happens when you lock a mutex?"* The naive mental model — "it's
a kernel object, locking is a syscall" — is **wrong for the common case**. A modern Linux mutex is a
**futex** ("fast userspace mutex"), and its whole design is to stay out of the kernel when there's no
contention.

The mutex is, at heart, an integer in user-space memory (0 = unlocked, 1 = locked).

```
 lock():
   ┌─ FAST PATH (uncontended) ───────────────────────────────┐
   │  atomic compare-and-swap the word 0 → 1 in USER space.   │   ~tens of ns, NO syscall
   │  success? you hold the lock. return.                     │
   └──────────────────────────────────────────────────────────┘
                     │ CAS failed (someone holds it)
                     ▼
   ┌─ SLOW PATH (contended) ─────────────────────────────────┐
   │  syscall futex(FUTEX_WAIT, &word, expected=1):           │
   │    kernel puts this thread to SLEEP on a wait queue      │   context switch, ~µs
   │    keyed by the futex address; thread → WAITING.         │
   └──────────────────────────────────────────────────────────┘

 unlock():
   atomic store word → 0 (user space).
   if there were waiters: syscall futex(FUTEX_WAKE, &word, 1) → kernel wakes one waiter.
```

The takeaways interviewers want:

- **An uncontended lock/unlock is a couple of atomic ops in user space — tens of nanoseconds, no
  syscall.** This is why "mutexes are slow" is a myth for low-contention code.
- **A contended lock is expensive** — it falls into the kernel via `futex`, sleeps the thread (a
  context switch out, Module 10), and later wakes it (another switch). Now you're paying microseconds
  plus cold caches. Contention, not locking, is the cost.
- This is why HFT hot paths **avoid contention entirely** rather than "avoid mutexes" — a single-
  threaded matcher fed by a lock-free SPSC queue ([`../cpp-guide/16`](../cpp-guide/16-atomics-lockfree.md)) has
  no shared mutable state and thus no locks at all.

---

## 6. Spinlock vs mutex

Both provide mutual exclusion; they differ in what a *waiter* does:

- **Spinlock** — the waiter **busy-waits** (`while (!try_lock()) _mm_pause();`), burning CPU until the
  lock frees. No context switch. Good only when (a) the critical section is *very* short (a few
  instructions) and (b) the waiter has a dedicated core, so spinning doesn't starve the holder. On a
  single core, spinning is a disaster: the waiter burns the whole slice while the holder can't run to
  release the lock.
- **Mutex (blocking)** — the waiter **sleeps** (futex slow path) and yields the CPU. Costs a context
  switch to sleep/wake, but doesn't waste cycles while waiting. Right for longer or contended sections.

Rule of thumb: **spin if the expected wait is shorter than the cost of a context switch (~µs) and you
can spare the core; otherwise block.** Adaptive mutexes spin briefly, then fall back to sleeping.

---

## 7. Condition variables

A mutex protects data; a **condition variable** lets a thread *wait until a condition becomes true*
without busy-waiting, and be woken when another thread changes the state.

```cpp
std::mutex m;
std::condition_variable cv;
std::queue<Task> q;

// consumer
std::unique_lock lk{m};
cv.wait(lk, []{ return !q.empty(); });   // atomically: release m, sleep; on wake re-acquire m, recheck
Task t = std::move(q.front()); q.pop();
```

Two non-negotiable facts:

- **Always use the predicate form** (`cv.wait(lk, pred)`), because a wait can return **spuriously**
  (without any notify) and because a notify sent *before* you waited would otherwise be **lost**. The
  predicate + internal while-loop re-checks the actual condition, so a spurious/lost wakeup is
  harmless.
- **`wait` atomically releases the mutex and sleeps**, then re-acquires it on wake. That atomicity is
  exactly why a condvar needs an associated mutex: without it, there'd be a window between "check the
  condition" and "go to sleep" where a notify could slip through and be missed (the lost-wakeup race).

---

## 8. Semaphores, reader-writer locks, and friends

- **Semaphore** — a counter with atomic `acquire` (wait/P) and `release` (post/V). A counting
  semaphore of N permits N concurrent holders (resource pool); a binary semaphore (count 1) resembles
  a mutex but, unlike a mutex, has **no ownership** — any thread can post it, which makes it useful for
  signaling between threads (producer posts, consumer waits).
- **Reader-writer lock** (`std::shared_mutex`) — many concurrent **readers** *or* one exclusive
  **writer**. Ideal for read-heavy, write-rare data (e.g. a config read on every request, reloaded
  occasionally). Watch for writer starvation under constant readers.
- **`std::latch`/`std::barrier`** — one-shot / reusable rendezvous ("all N threads wait here until
  everyone arrives").
- **`thread_local`** — per-thread storage; sidesteps sharing entirely, so no synchronization needed.
  Often the fastest "fix" for a race: don't share at all.

---

## 9. Deadlock — the four Coffman conditions

A **deadlock** is a set of threads each waiting for a resource another holds, so none can proceed.

```
 Thread 1: holds A, wants B          A ──held by──▶ T1 ──wants──▶ B
 Thread 2: holds B, wants A          B ──held by──▶ T2 ──wants──▶ A     (a cycle → stuck forever)
```

All **four Coffman conditions** must hold simultaneously; break any one to prevent deadlock:

1. **Mutual exclusion** — resources aren't shareable.
2. **Hold and wait** — a thread holds one resource while waiting for another.
3. **No preemption** — resources can't be forcibly taken back.
4. **Circular wait** — a cycle in the "who waits for whom" graph.

Practical prevention:

- **Lock ordering** — always acquire multiple locks in a fixed global order (kills circular wait).
  Simplest, most common.
- **`std::scoped_lock{m1, m2}`** — locks multiple mutexes with a deadlock-avoidance algorithm.
- **Try-lock with backoff / timeouts** — don't hold-and-wait; if you can't get all locks, release and
  retry.
- **Avoid nested locks** entirely where possible.

Beyond deadlock, know **livelock** (threads keep changing state in response to each other but make no
progress) and **priority inversion** (a low-priority thread holds a lock a high-priority thread needs;
fixed by priority inheritance — famously the Mars Pathfinder bug).

---

## 10. The HFT stance: avoid the problem

Every primitive above has a cost, and the fastest synchronization is **none**. HFT designs therefore
lean on:

- **Single-threaded hot path** — the matching engine owns its data exclusively; no locks, always
  cache-hot, deterministic ([`../cpp-guide/16`](../cpp-guide/16-atomics-lockfree.md)).
- **Message passing over shared state** — hand data between threads through a **lock-free SPSC queue**
  rather than sharing a structure behind a mutex. No contention, bounded latency.
- **Immutability & `thread_local`** — data never written after publication needs no locks;
  per-thread state isn't shared at all.
- **Atomics for the rare shared flag** — a single `std::atomic` counter/flag is lock-free and
  nanosecond-scale, far cheaper than a mutex (memory-ordering details in `../cpp-guide/16`).

Locks and condvars still appear — but in the **control plane** (config reload via `shared_mutex`,
shutdown coordination, the thread pool for backtests), never on the tick-to-trade path.

---

## Common pitfalls / misconceptions

- **"Mutexes are slow."** Only when *contended*. An uncontended lock is a user-space CAS (tens of ns,
  no syscall). Blame contention, not the mutex.
- **"Locking always enters the kernel."** No — the futex fast path stays in user space; only the
  contended slow path calls `futex()`.
- **Forgetting to join/detach.** A joinable `std::thread` destroyed → `std::terminate()`. Use
  `jthread`.
- **`cv.wait` without a predicate.** Spurious and lost wakeups will bite you. Always pass the
  predicate.
- **A binary semaphore is not a mutex.** A mutex has ownership (only the locker unlocks); a semaphore
  can be posted by any thread. Using a semaphore as a lock loses the ownership invariant.
- **Spinning on a single core (or an oversubscribed one).** The spinner starves the lock holder; you
  spin the entire time slice for nothing.
- **"More threads = faster."** Past the core count, extra threads just add context-switch and
  contention overhead (oversubscription). Amdahl's law caps speedup by the serial fraction.

---

## Quiz — tough problems

**Q1.** You lock and unlock an uncontended `std::mutex` a million times in a tight loop. Roughly how
many syscalls does that make?

**Answer:** **Roughly zero.** Uncontended lock/unlock is an atomic CAS and an atomic store in user
space (the futex fast path). No `futex` syscall is issued unless a lock is contended (a waiter must
sleep) or an unlock must wake a waiter. This is why the "mutex = syscall" mental model is wrong.

---

**Q2.** A thread does `cv.wait(lk)` (no predicate). Another thread calls `cv.notify_one()` *before*
the first thread reaches `wait`. What happens?

**Answer:** The notify is **lost** — condition variables don't remember notifications. The waiter may
sleep forever. The fix is the predicate form `cv.wait(lk, pred)`: because the mutex is held while
checking `pred` and the state was already set, the predicate returns true immediately and the thread
never sleeps. (The predicate form also handles spurious wakeups.)

---

**Q3.** Is a spinlock ever *worse* than a blocking mutex on a single-core machine? Explain.

**Answer:** Almost always worse. On one core, if T1 holds the lock and T2 spins, T2 burns its entire
time slice spinning — but T1 can't run to release the lock because T2 has the core. You waste a full
slice guaranteeing the lock *won't* be released. A blocking mutex would sleep T2, letting T1 run and
release. Spinlocks pay off only with multiple cores and very short critical sections.

---

**Q4.** Two threads: T1 does `lock(A); lock(B);`, T2 does `lock(B); lock(A);`. Name the bug, the
condition it exploits, and the simplest fix.

**Answer:** **Deadlock**, exploiting **circular wait** (T1 holds A wants B; T2 holds B wants A).
Simplest fix: **global lock ordering** — make both acquire A before B (or use `std::scoped_lock{A, B}`,
which orders internally to avoid the cycle).

---

**Q5.** What's the difference in resource cleanup between a joined thread and a detached thread?

**Answer:** For a **joined** thread, the joining thread blocks until it finishes and then reclaims its
resources (and can read its result / know it's done). For a **detached** thread, resources are reclaimed
**automatically** when it exits, but you have no handle and cannot wait for or observe it. Both prevent
the resource leak / `std::terminate`; they differ in whether you retain a rendezvous point.

---

**Q6.** Why does a condition variable require a mutex, when its whole job is to make a thread sleep?

**Answer:** To close the **lost-wakeup race** between checking the condition and going to sleep.
`wait()` **atomically** releases the mutex and blocks; because the state is only ever changed under the
same mutex, there's no window where a notifier can change the state and signal *between* your check and
your sleep. Without the mutex, a notify could slip into that gap and be missed forever.

---

**Q7.** A binary semaphore initialized to 1 is used like a mutex: thread A `acquire()`s, does work,
`release()`s. Thread B (which never acquired it) also calls `release()`. What breaks?

**Answer:** The semaphore's count goes to 2, so **two** threads can now `acquire()` simultaneously —
mutual exclusion is destroyed. A semaphore has **no ownership**: any thread can post it. A real mutex
would reject an unlock by a non-owner (or it's UB). This is exactly why a semaphore ≠ a mutex.

---

**Q8.** You have a config struct read by 50 worker threads on every request and rewritten roughly once
an hour. Which primitive, and why not a plain mutex?

**Answer:** A **reader-writer lock** (`std::shared_mutex`): the 50 readers take it in **shared** mode
concurrently (no serialization on the common path), while the rare writer takes it **exclusively**. A
plain mutex would serialize all 50 readers against each other for no reason, killing throughput on the
hot path. (For truly hot data, an atomic pointer swap to an immutable new config — RCU-style — avoids
locking readers entirely.)

---

**Q9.** Thread switch vs process switch cost from the synchronization designer's view — why does it
push HFT toward threads-in-one-process plus lock-free queues rather than multiple processes with
shared memory?

**Answer:** Thread switches skip the CR3 reload/TLB flush (Module 10 §7), so they're cheaper, and
threads share an address space *directly* (no shared-memory setup, no separate mappings). That makes a
**lock-free SPSC queue between two pinned threads** the cheapest possible hand-off: no address-space
switch, no kernel, no lock. Multiple processes would add IPC/shared-memory overhead and more expensive
switches for no benefit on the hot path.

---

## Indian HFT interview questions

**"What actually happens when you lock a `std::mutex`? Is it a syscall?"**
Not usually. It's a **futex**: the fast path is an atomic CAS on a user-space word (tens of ns, no
syscall) when uncontended. Only on **contention** does it call `futex(FUTEX_WAIT)` to sleep the thread
in the kernel, and unlock calls `FUTEX_WAKE` if waiters exist. So cost scales with contention, not with
locking frequency.

**"`join` vs `detach` — and what happens if you do neither?"**
`join` blocks until the thread finishes and reclaims it (gives a rendezvous point). `detach` lets it
run independently with auto-reclamation but no way to join. Do neither and let the `std::thread`
destruct while joinable → `std::terminate()`. `std::jthread` auto-joins (RAII) to make this safe.

**"Spinlock or mutex — how do you choose?"**
Spin if the critical section is a handful of instructions *and* you have a spare/dedicated core, so the
wait is shorter than a context switch (~µs) and spinning doesn't starve the holder. Otherwise block
(mutex/futex) so the waiter yields the CPU. On a single core, prefer blocking. Adaptive locks spin
briefly then sleep.

**"Why does `cv.wait` take a predicate?"**
To survive **spurious wakeups** (wait can return with no notify) and **lost wakeups** (notify sent
before you waited). The predicate re-checks the real condition under the mutex, so a stray or missed
signal is harmless.

**"Explain deadlock and how you'd prevent it in a codebase with many locks."**
Four Coffman conditions (mutual exclusion, hold-and-wait, no preemption, circular wait); break one.
Practically: enforce a **global lock ordering**, use `std::scoped_lock` for multi-lock acquisition,
avoid nested locks, and prefer designs that don't hold multiple locks at once. Mention priority
inversion + priority inheritance if pushed.

**"How does HFT get away with barely using locks?"**
By eliminating shared mutable state on the hot path: a **single-threaded** matcher owns its data,
threads communicate via **lock-free SPSC queues**, and shared reads use immutable snapshots or a lone
atomic flag. Locks live only in the control plane. The fastest synchronization is not needing any.

**"Threads vs processes — and why is a thread switch cheaper?"**
Threads share the address space (heap/globals/fds), own only stack/registers/TLS; a process is the
isolated container. A thread switch keeps the same page table, so **no CR3 reload and no TLB flush**
(Module 10) — cheaper than a process switch. That sharing is also what makes data races possible.

---

## Key takeaways

- A **thread** = execution context (own stack/registers/TLS) inside a process's shared address space;
  Linux threads are **1:1 kernel threads** created via `clone()`.
- Every thread must be **`join`ed or `detach`ed**; a joinable `std::thread` destroyed calls
  `std::terminate()`. Prefer `std::jthread` (auto-join, RAII).
- A **mutex is a futex**: uncontended lock/unlock is a user-space atomic (no syscall, tens of ns); only
  **contention** enters the kernel (sleep/wake, ~µs). Contention is the cost, not locking.
- **Spin** for very short waits on a spare core; **block** otherwise. Spinning on a single/oversubscribed
  core starves the lock holder.
- **Condition variables** need a mutex and a **predicate** to avoid lost/spurious wakeups; `wait`
  atomically releases-and-sleeps, then re-acquires.
- **Deadlock** needs all four Coffman conditions; break one — usually via **global lock ordering** or
  `std::scoped_lock`.
- HFT's real answer is to **avoid shared mutable state**: single-threaded hot path, lock-free SPSC
  queues, immutability/`thread_local`, atomics for the rare flag. Locks stay in the control plane.

**Next:** [12 — Allocators & I/O](12-allocators-and-io.md)
