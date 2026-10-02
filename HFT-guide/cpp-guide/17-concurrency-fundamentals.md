# Module 17 — Concurrency fundamentals: threads, mutexes, condition variables & spinlocks

Before lock-free programming, you must understand what locks solve. A data race is not merely “the
wrong answer sometimes”; in C++ it is **undefined behavior**. Mutexes, condition variables, and
thread ownership provide the simplest correct model. Atomics come next, after you can state exactly
which shared state is protected and which ordering the program needs.

HFT changes the trade-off, not the correctness rules. Blocking and context switches are dangerous
on a latency-critical path, but a correct mutex on a cold path is better than an incorrect lock-free
structure everywhere.

---

## 1. Processes, threads, and the shared-address-space problem

Threads in one process share code, globals, and heap storage. Each thread has its own registers and
stack:

```text
process address space
┌──────────────────────────────────────────────┐
│ code │ globals │ heap (shared by all threads)│
├──────────────────────────────────────────────┤
│ thread A: registers + stack                  │
│ thread B: registers + stack                  │
└──────────────────────────────────────────────┘
```

Sharing avoids serialization/copying but creates synchronization obligations. A pointer to a local
belongs to one thread's stack; handing it to another thread is only safe if the object outlives every
access and concurrent access is synchronized.

### Launching and joining

```cpp
#include <thread>

void parse_feed(int channel);

std::thread t{parse_feed, 7};
// do independent work
t.join();                         // wait; t is no longer joinable
```

A joinable `std::thread` destroyed without `join()` or `detach()` calls `std::terminate`. Detaching
usually makes lifetime and shutdown harder: the detached thread may outlive every object it uses.
Prefer structured ownership and joining.

C++20's `std::jthread` joins automatically and supports cooperative cancellation:

```cpp
void receiver(std::stop_token stop) {
    while (!stop.stop_requested()) {
        poll_once();
    }
}

std::jthread io{receiver};        // destructor requests stop and joins
```

RAII applies to threads just as it does to files and memory.

---

## 2. Data races and critical sections

`counter++` is a read-modify-write, not an indivisible operation:

```text
Thread A: load 0 ── add 1 ── store 1
Thread B:    load 0 ── add 1 ── store 1
Expected 2, observed 1
```

The formal rule is stronger: a **data race** occurs when two threads access the same memory location,
at least one access writes, the accesses are not atomic, and there is no happens-before ordering.
The whole program then has UB.

“It is one aligned machine word” does not fix this. C++ compiler transformations and hardware
ordering are constrained by the language memory model, not by what happened in one debugger run.

A **critical section** is code that must execute with exclusive access to some shared invariant.
Protect the invariant, not merely one individual field:

```cpp
struct Position {
    std::int64_t bought{};
    std::int64_t sold{};
    std::mutex mutex;

    void fill(bool is_buy, std::int64_t qty) {
        std::lock_guard lock{mutex};
        (is_buy ? bought : sold) += qty;
        // every invariant involving bought/sold remains protected until scope exit
    }
};
```

---

## 3. Mutexes and RAII locking

`std::mutex::lock()` grants mutual exclusion; another thread trying to lock it waits. Never pair raw
`lock()`/`unlock()` across code paths when an RAII guard can do it safely:

```cpp
std::mutex mutex;

void update() {
    std::lock_guard guard{mutex};  // locks now
    mutate_shared_state();
}                                 // unlocks on return or exception
```

The main wrappers:

- **`std::lock_guard`**: simple scoped ownership; lock now, unlock at scope exit.
- **`std::scoped_lock`**: scoped ownership of one or more mutexes; locks multiple mutexes without the
  usual order inversion.
- **`std::unique_lock`**: movable and can defer, unlock, and relock; required by
  `std::condition_variable`.
- **`std::shared_lock` + `std::shared_mutex`**: multiple readers or one writer. Useful only when reads
  are sufficiently long/frequent to repay the more expensive lock machinery.

```cpp
void transfer(Account& from, Account& to, Money amount) {
    if (&from == &to) return;
    std::scoped_lock both{from.mutex, to.mutex};
    // check and update the two-account invariant atomically
}
```

### Keep lock scope explicit and small

Do not hold a mutex while performing I/O, invoking unknown callbacks, allocating unnecessarily, or
waiting on another subsystem. But “small” does not mean splitting one invariant across multiple
locks; correctness comes first.

---

## 4. Deadlock, livelock, starvation, and contention

They are different failure modes:

- **deadlock**: threads wait forever in a cycle of held/requested resources;
- **livelock**: threads keep reacting/retrying but make no useful progress;
- **starvation**: one thread can be postponed indefinitely while others progress;
- **contention**: threads compete for the same synchronization/cache resource and slow down.

Classic deadlock:

```text
Thread A holds M1, waits for M2
Thread B holds M2, waits for M1
```

Prevent it by avoiding nested locks where possible, imposing one global lock order, or using
`std::scoped_lock(m1, m2)`. Never “solve” it with sleeps; timing changes do not establish a proof.

Even an uncontended mutex has bookkeeping cost. A contended mutex may enter the OS, park a thread,
and later reschedule it—large and variable latency. This is why a matching engine usually gives the
book to one thread rather than protecting every book operation with a mutex.

---

## 5. Condition variables: sleep until a state change

A condition variable coordinates waiting for a predicate protected by a mutex. It avoids burning a
core in a polling loop:

```cpp
std::mutex mutex;
std::condition_variable cv;
std::queue<Message> queue;
bool stopping = false;

void consume() {
    for (;;) {
        std::unique_lock lock{mutex};
        cv.wait(lock, [] { return stopping || !queue.empty(); });

        if (stopping && queue.empty()) return;
        Message msg = std::move(queue.front());
        queue.pop();
        lock.unlock();                  // process outside the critical section
        process(msg);
    }
}

void produce(Message msg) {
    {
        std::lock_guard lock{mutex};
        queue.push(std::move(msg));
    }
    cv.notify_one();
}
```

`wait(lock, predicate)` conceptually:

1. checks the predicate while holding the mutex;
2. atomically releases the mutex and sleeps if false;
3. wakes, reacquires the mutex, and checks again.

Always wait with a predicate because wakeups may be **spurious**, and another consumer may take the
work before this one reacquires the mutex. The shared predicate—not the notification—is the truth.

Condition variables are excellent for control planes, worker pools, startup, and shutdown. Their OS
scheduling latency usually makes them unsuitable for a nanosecond-sensitive message hop.

---

## 6. Spinlocks: trading CPU for wake-up latency

A spinlock repeatedly tries an atomic acquisition instead of sleeping:

```cpp
class SpinLock {
    std::atomic_flag held_ = ATOMIC_FLAG_INIT;
public:
    void lock() noexcept {
        while (held_.test_and_set(std::memory_order_acquire)) {
            while (held_.test(std::memory_order_relaxed)) {
                // platform pause/yield or bounded backoff may belong here
            }
        }
    }

    void unlock() noexcept {
        held_.clear(std::memory_order_release);
    }
};
```

The inner read-only loop reduces repeated write ownership requests on the cache line. Production
implementations commonly add a CPU pause instruction and bounded backoff.

A spinlock is reasonable only when:

- the critical section is extremely short and cannot block;
- the owner is guaranteed to run promptly;
- active threads do not greatly exceed available/pinned cores;
- measurements show sleeping/waking costs more than the spin.

It is harmful when the owner is descheduled, the critical section performs I/O/allocation, or many
threads contend. Spinning then wastes a core and creates coherency traffic. It is neither lock-free
nor automatically fair: progress still depends on the lock holder.

---

## 7. Choosing a synchronization primitive

| Need | First choice | Why |
|---|---|---|
| protect a multi-step invariant | mutex + RAII guard | simplest correctness proof |
| lock several mutexes | `std::scoped_lock` | deadlock-avoiding acquisition |
| sleep until work/state change | condition variable + predicate | no busy waiting |
| one independent counter/flag | `std::atomic` | no lock for a single atomic state |
| ultra-short, measured critical section | spinlock, cautiously | avoids park/wake latency |
| one producer → one consumer hot hop | bounded SPSC ring | ownership pattern removes contention |
| complex MPMC ownership | proven queue/library or mutex | reclamation and progress are difficult |

Do not replace a mutex-protected compound invariant with several atomics and assume the combination
became atomic. Atomics protect individual operations; the algorithm must establish relationships
between them (Module 18).

---

## 8. Low-latency architecture: remove sharing before optimizing it

The fastest synchronization is ownership:

```text
network thread ──SPSC──▶ matching thread ──SPSC──▶ publisher
                         owns the order book
```

- The matching thread alone mutates the book: no book mutex, no book data race.
- Bounded SPSC queues connect stages with known capacity and predictable memory.
- Configuration/startup/shutdown may use ordinary mutexes and condition variables off the hot path.
- Threads may be pinned to cores only after measurement and with an operational plan; affinity is
  platform-specific and can hurt when chosen blindly.

Partitioning by symbol/venue can scale the same principle across multiple matching threads, provided
cross-partition work and sequencing rules are explicit.

---

## Common pitfalls & UB

- Destroying a joinable `std::thread` calls `std::terminate`; join it or use `std::jthread`.
- Capturing a local by reference in a thread that outlives the scope creates a dangling reference.
- A non-atomic read concurrent with a write is a data race even if the word is naturally aligned.
- Manually unlocking on only the normal path leaks the lock on exceptions/early returns; use RAII.
- Waiting without a predicate mishandles spurious wakeups and stolen work.
- Locking the same non-recursive mutex twice on one thread deadlocks.
- Calling user code while holding a lock may re-enter or acquire locks in an unknown order.
- A spinlock around blocking work can stall the system and burn every available core.
- Several atomic fields do not automatically form one consistent snapshot.

---

## Quiz — tough problems

**Q1.** One thread only reads an `int`; another only writes it. The `int` is aligned. Is that safe?
**Answer:** No. Concurrent conflicting non-atomic accesses without happens-before form a data race,
which is UB. Alignment does not create synchronization.

**Q2.** Why must `condition_variable::wait` use a predicate/loop?
**Answer:** Wakeups can be spurious, and a different thread may consume the state before this waiter
reacquires the mutex. The predicate must be rechecked while holding the lock.

**Q3.** Is a spinlock lock-free?
**Answer:** No. If the holder stops, every waiter stops making progress. “It uses atomics” does not
imply a lock-free progress guarantee.

**Q4.** Why is `std::scoped_lock{a, b}` preferable to `a.lock(); b.lock();`?
**Answer:** It uses deadlock-avoiding multi-lock acquisition and releases both via RAII. Independent
manual ordering can form cycles and is not exception-safe.

**Q5.** Can you replace one mutex guarding `{head, tail, size}` with three atomics?
**Answer:** Not mechanically. Readers could observe a combination that never represented one valid
state. You need a proven algorithm and memory-order relationships, not merely atomic fields.

**Q6.** When is detaching a thread dangerous?
**Answer:** Its lifetime becomes disconnected from the objects it accesses and from orderly shutdown.
It can outlive captured references, logging, allocators, or the process subsystem that owns it.

---

## HFT interview questions

- **“What exactly is a data race?”** Two potentially concurrent conflicting accesses to one memory
  location, at least one a write, neither properly atomic/ordered; in C++ the result is UB.
- **“Mutex vs spinlock?”** A mutex can park and reschedule, saving CPU but adding variable latency; a
  spinlock burns CPU and only wins for tiny, bounded holds when the owner keeps running.
- **“Why does `condition_variable` need `unique_lock`?”** Waiting must atomically unlock, sleep, then
  relock; `unique_lock` supports that changing ownership state.
- **“Name the four Coffman deadlock conditions.”** Mutual exclusion, hold-and-wait, no preemption,
  and circular wait; breaking any one prevents deadlock.
- **“How would you synchronize a matching engine?”** Single-owner book, bounded SPSC queues at the
  edges, immutable/read-only snapshots for observers; avoid sharing the hot state.
- **“Why can fine-grained locking be slower?”** More lock operations, a larger state space, cache-line
  bouncing, and higher risk of deadlock; theoretical concurrency may not repay those costs.

---

## In the order book

The network thread parses messages and publishes them to a bounded SPSC queue. One pinned matching
thread owns all mutable book state and processes messages sequentially, preserving price-time order
without locks. A second queue carries fills outward.

Mutexes and condition variables remain useful for cold control-plane work: loading configuration,
starting workers, coordinating snapshots, and stopping cleanly. A lock-free structure is not a
badge applied everywhere; it is a targeted answer to measured blocking/jitter on a specific path.

---

## Key takeaways

- Threads share heap/globals but own stacks; shared mutable state needs synchronization and valid
  lifetime.
- A C++ data race is UB. “Atomic on my CPU” and “worked in testing” are not arguments.
- Use RAII lock wrappers, protect whole invariants, and prevent deadlock with ownership/order or
  multi-lock tools.
- Condition variables sleep on a mutex-protected predicate; always recheck that predicate.
- Spinlocks exchange scheduler latency for CPU and coherency traffic; they fit only tiny, bounded,
  measured critical sections.
- Low-latency design first removes sharing through single ownership and queues. Atomics are the next
  layer, not the starting point.

**Next:** [Module 18 — `std::atomic`, memory ordering & lock-free basics](18-atomics-lockfree.md) —
the C++ memory-model tools behind the bounded SPSC queue.
