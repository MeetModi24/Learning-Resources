# Module 16 — `std::atomic`, memory ordering, lock-free basics

Concurrency without locks. This is "table stakes for HFT" and the hardest interview territory in the
whole track — the topic that most reliably separates candidates who *memorized* C++ from those who
*understand* the machine. The goal of this module is a correct mental model plus the one lock-free
structure you'll actually build and be asked to whiteboard: the single-producer/single-consumer ring
buffer. We build directly on Module 15: atomics and memory ordering are the software knobs that ride
on top of the MESI cache-coherency hardware you just met.

---

## 1. Why not just use a mutex?

A `std::mutex` is correct, simple, and the right default for most software. So why does HFT avoid it
on the hot path?

```cpp
std::mutex m;
void push(Message msg) {
    std::lock_guard lk(m);   // acquire lock (may BLOCK)
    queue.push_back(msg);
}                            // unlock on scope exit
```

The problem is not throughput — an uncontended mutex is only tens of nanoseconds. The problem is
**tail latency and determinism**:

- When two threads want the lock at once, one **blocks**. Blocking means the OS may deschedule the
  thread (context switch, ~1–10 µs) — an eternity when your budget is hundreds of nanoseconds.
- A thread can be **preempted while holding the lock** (its time slice ends, or a page fault hits).
  Now *every* other thread waiting on that lock is stuck until the holder is rescheduled. This is
  **priority inversion / convoying**, and it's unbounded and unpredictable — poison for P99.9
  latency.
- Locks serialize; they don't compose well; and they can deadlock.

Lock-free structures let threads **make progress without blocking** — no thread's stall can stall
another indefinitely. The payoff is *bounded, predictable* latency. The price is that they are
subtle, easy to get wrong, and only correct if you reason precisely about memory ordering (below).
A useful hierarchy of guarantees interviewers probe:

- **Wait-free** — every thread finishes its operation in a bounded number of steps, no matter what
  others do (strongest; e.g. a well-designed SPSC queue push).
- **Lock-free** — *some* thread always makes progress system-wide, but an individual thread can be
  starved (retry forever). Typical of CAS-loop structures.
- **Obstruction-free** — a thread makes progress if it runs *alone* for long enough (weakest).

"Lock-free" colloquially means "no mutexes," but know the precise ladder.

---

## 2. `std::atomic` — indivisible operations

An **atomic** operation is one no other thread can observe half-done — no torn reads, no torn
writes, no lost updates.

```cpp
#include <atomic>
std::atomic<int> counter{0};

counter.fetch_add(1);          // atomic read-modify-write increment
int v = counter.load();        // atomic read
counter.store(5);              // atomic write
int old = counter.exchange(9); // atomically set to 9, return previous
```

Why this matters: `++counter` on a plain `int` from two threads is **three steps** — load, add,
store. Two threads can interleave (both load 5, both store 6) and *lose* an update. Worse, in the
C++ memory model, two threads accessing the same non-atomic object where at least one writes, with
no synchronization, is a **data race — undefined behavior**. Not "you get a wrong number" — the
whole program is UB (the compiler may assume it never happens and miscompile everything). `atomic`
makes the operation indivisible *and* well-defined.

`std::atomic<T>` works for any trivially-copyable `T`, but only small types are **lock-free** (the
CPU has an instruction for them). Check with `is_lock_free()` / `is_always_lock_free`:

```cpp
static_assert(std::atomic<std::int64_t>::is_always_lock_free);   // true on x86-64
std::atomic<BigStruct> a;  // if too big, the library uses a HIDDEN lock internally!
assert(a.is_lock_free());  // verify — a "lock-free" algorithm using a locking atomic isn't
```

An `atomic<T>` larger than the CPU's largest atomic instruction (16 bytes on x86-64 with cmpxchg16b)
silently falls back to a **mutex inside the standard library**. Your "lock-free" queue is then
secretly using locks — a classic gotcha.

### Compare-and-swap (CAS) — the foundation of lock-free

```cpp
std::atomic<int> x{0};
int expected = 0;
// Atomically: if x == expected, set x = 10 and return true;
//             else load current x INTO expected, return false.
while (!x.compare_exchange_weak(expected, 10)) {
    // expected now holds the current value; recompute your new value from it and retry
}
```

CAS is the primitive from which almost all lock-free algorithms are built: read a value, compute a
new one, and *atomically* install it **only if nobody changed it in the meantime**. If someone did,
the CAS fails, hands you the fresh value, and you retry — a **CAS loop**.

```
CAS loop:
   ┌────────────────────────────────────────┐
   │ read expected = x.load()                │
   │ compute desired = f(expected)           │
   │ CAS(x, expected, desired) ── success ───┼──► done
   │            │ fail (someone else wrote)   │
   │            └───────── retry ─────────────┘
   └────────────────────────────────────────┘
```

**`compare_exchange_weak` vs `strong`:** `weak` may fail *spuriously* (report failure even when
`x == expected`) on some architectures (LL/SC on ARM/POWER), but is cheaper — use it *inside a
loop* where a spurious retry is harmless. `strong` never fails spuriously — use it when you're
*not* looping and a false failure would be a bug.

---

## 3. Memory ordering — the genuinely hard part

Here is the mind-bending fact: **the order you write memory operations in your source is not the
order other threads see them happen.** Both the *compiler* (reordering instructions during
optimization) and the *CPU* (out-of-order execution, store buffers) reorder memory operations for
speed. Within a single thread this is invisible — the hardware guarantees your own reads see your
own prior writes ("as-if" single-threaded). But *across* threads, another core can observe your
writes in a **different order** than you issued them. Memory ordering is the set of knobs that
constrains which reorderings are allowed.

The canonical example — publishing data with a flag:

```cpp
std::atomic<bool> ready{false};
int data = 0;                     // plain (non-atomic) data

// Producer thread:
data = 42;                                          // (1) plain write
ready.store(true, std::memory_order_release);       // (2) RELEASE

// Consumer thread:
while (!ready.load(std::memory_order_acquire)) {}   // (3) ACQUIRE (spin until true)
use(data);                                          // (4) guaranteed to see 42
```

Why it's correct — the **publish/consume** contract, drawn as a timeline:

```
 PRODUCER (core 0)                     CONSUMER (core 1)
 ─────────────────                     ─────────────────
 data = 42            ─┐
                       │ release-store PUBLISHES
 ready.store(true,     │ everything sequenced
   release) ───────────┼───────────►  ready.load(acquire) sees true
                       │               (acquire-load CONSUMES)
                       └───────────►  use(data)  ← sees 42, guaranteed
       ▲                                    ▲
   nothing above (2) can                reads below (3) can't
   move BELOW the release              move ABOVE the acquire
```

- A **release** store acts as a one-way barrier: no memory operation *before* it can be reordered
  to *after* it. It "publishes" everything the thread wrote beforehand.
- A matching **acquire** load is the other one-way barrier: no operation *after* it can be reordered
  to *before* it. When an acquire-load reads the value a release-store wrote, everything the
  producer did before the release becomes visible to the consumer. This is a
  **synchronizes-with / happens-before** relationship.

Note `data` is a *plain* int and it's still safe — because the acquire/release pair establishes
happens-before, transitively covering the non-atomic write. Without it, `use(data)` racing `data=42`
would be UB.

### The five orderings, weakest → strongest

- **`relaxed`** — atomic (no torn values, no lost RMWs) but **no ordering** relative to other memory
  ops. Fastest. Use only where order genuinely doesn't matter: a statistics counter you read later,
  or your *own* index in an SPSC queue (only you write it).
- **`consume`** — a weaker acquire that orders only *dependent* reads. Notoriously hard to specify;
  in practice everyone uses `acquire` instead. Know it exists; don't use it.
- **`acquire`** (loads) / **`release`** (stores) — the workhorse pair. Release publishes; acquire
  consumes. This is how you hand data between threads on the hot path.
- **`acq_rel`** — both at once, for read-modify-write ops (`fetch_add`, CAS) that must publish *and*
  consume.
- **`seq_cst`** (the default) — sequential consistency: on top of acquire/release, there's a single
  global total order all threads agree on. Easiest to reason about, **strongest, slowest** (often a
  full memory fence, e.g. `mfence`/`lock`-prefixed op on x86).

```
relaxed  <  acquire/release  <  acq_rel  <  seq_cst
 (fastest, fewest guarantees)          (slowest, one global order)
```

**Rule of thumb:** write it with the default `seq_cst` first (correct by construction, easy to
reason about). Only relax to `acquire`/`release` on a proven-hot path where you've *shown* it's
correct. Relaxing ordering is the #1 source of "works on x86, breaks on ARM" bugs — see next.

### x86 (TSO) vs ARM (weak) — why testing on one isn't enough

Different CPUs have different **hardware memory models**:

- **x86-64 is TSO (Total Store Order)** — a *strong* model. Loads aren't reordered with loads,
  stores aren't reordered with stores; the only relaxation is that a store can be delayed behind a
  later load (store buffer). Practically, on x86 an ordinary load already behaves almost like
  acquire and a store almost like release — so **buggy code with too-weak ordering often still works
  on x86 by accident.**
- **ARM/AArch64 and POWER are weak** — they reorder aggressively. The *same* mis-ordered code
  *breaks* here.

So a lock-free structure that passes every test on your x86 dev box can corrupt data on an ARM
server. You cannot test your way to correctness here — you must *reason* about the ordering. This is
a favorite interview point precisely because it's a real production trap.

---

## 4. The SPSC ring buffer (single-producer, single-consumer)

This is the lock-free structure you'll build and be asked to whiteboard — the simplest genuinely
useful one. Exactly **one** thread pushes and **one** thread pops. Because each index has a single
writer, you need *no CAS at all* — just acquire/release. It's wait-free on both ends.

```cpp
template <typename T, std::size_t N>       // N MUST be a power of two
class SpscQueue {
    // Separate cache lines → no false sharing between producer and consumer (Module 15).
    alignas(64) std::atomic<std::size_t> head_{0};  // consumer owns (writes) this
    alignas(64) std::atomic<std::size_t> tail_{0};  // producer owns (writes) this
    T buffer_[N];
public:
    bool push(const T& v) {                         // called by producer thread ONLY
        auto t = tail_.load(std::memory_order_relaxed);   // my own index — relaxed is fine
        auto next = (t + 1) & (N - 1);
        if (next == head_.load(std::memory_order_acquire)) // acquire: see consumer's progress
            return false;                                   // full
        buffer_[t] = v;
        tail_.store(next, std::memory_order_release);       // release: PUBLISH the write above
        return true;
    }
    bool pop(T& out) {                              // called by consumer thread ONLY
        auto h = head_.load(std::memory_order_relaxed);    // my own index — relaxed
        if (h == tail_.load(std::memory_order_acquire))    // acquire: see producer's publish
            return false;                                   // empty
        out = buffer_[h];
        head_.store((h + 1) & (N - 1), std::memory_order_release);
        return true;
    }
};
```

State picture (N = 8):

```
 index:   0   1   2   3   4   5   6   7
        ┌───┬───┬───┬───┬───┬───┬───┬───┐
        │ a │ b │ c │   │   │   │   │   │
        └───┴───┴───┴───┴───┴───┴───┴───┘
          ▲           ▲
        head_=0     tail_=3          (3 items queued: a,b,c)
   consumer pops here   producer pushes here
   empty ≡ head_ == tail_   ;   full ≡ (tail_+1)&(N-1) == head_
   (one slot always left empty to distinguish full from empty)
```

Every optimization from earlier modules shows up here:

- **`alignas(64)`** on `head_`/`tail_` prevents **false sharing** (Module 15) — otherwise the two
  indices share a line and the producer/consumer ping-pong it on every op. This alone can be a 10×.
- **Power-of-two `N`** lets `& (N-1)` replace the expensive `% N` (Module 18 bit tricks).
- **`relaxed` for your own index** (only you write it) but **`acquire`/`release` to synchronize with
  the other thread** — the release-store of `tail_` publishes the `buffer_[t] = v` write; the
  consumer's acquire-load of `tail_` consumes it, so `out = buffer_[h]` safely sees the data.
- **Local index caching** (the rigtorp trick, a further optimization): each side caches the *other*
  thread's index and only re-loads the real atomic when its cache says the queue is full/empty.
  This slashes cache-coherency traffic — you avoid touching the other core's line on most ops.

---

## 5. The ABA problem (interview keyword)

In a CAS loop, a value can go **A → B → A** between your read and your CAS. Your CAS sees "still A"
and *succeeds*, but the world changed and changed back underneath you — the pointer you're about to
reuse may have been freed and a different object recycled into the same address.

```
 thread 1: read ptr = A ............................. CAS(ptr, A→X)  ← SUCCEEDS (ptr is A again!)
 thread 2:            pop A (free it), push new node B, pop B, push recycled node at addr A
 time ────────────────────────────────────────────────────────────►
                                                   but "A" is now a DIFFERENT object → corruption
```

CAS compares *values*, not *identity/history* — it can't tell "unchanged" from "changed back." This
bites lock-free stacks/queues built on raw pointers. Fixes:

- **Tagged pointers / version counters** — pack a monotonically increasing version alongside the
  pointer (double-width CAS `cmpxchg16b`); A-with-tag-5 ≠ A-with-tag-7.
- **Hazard pointers** — threads publish which pointers they're using so memory isn't reclaimed
  underneath them.
- **Epoch-based reclamation (EBR)** — defer freeing until no thread could still hold the old value.

Know the *name*, the *mechanism*, and at least one fix even if you never implement it. (SPSC queues
above sidestep ABA entirely: indices only ever increase, and there's a single writer per index.)

---

## 6. Common pitfalls & UB

- **Data race = UB.** Any non-atomic object touched by 2+ threads with ≥1 writer and no
  synchronization is undefined behavior — not merely "wrong value."
- **`volatile` is not for threading.** `volatile` prevents compiler optimization of an access
  (memory-mapped I/O) but provides **no atomicity and no cross-thread ordering.** Using it as a
  thread-sync primitive is a classic C-brain mistake. Use `std::atomic`.
- **Assuming x86 behavior is portable.** Too-weak ordering "works" on x86 (TSO) and breaks on ARM.
- **`atomic<BigStruct>` silently locking.** If it's not `is_lock_free()`, your lock-free design uses
  a hidden mutex.
- **`compare_exchange` and the `expected` reload.** On failure, CAS *overwrites* `expected` with the
  current value — recompute `desired` from it, don't reuse the stale one.
- **`weak` vs `strong` misuse.** `weak` outside a loop can falsely fail; `strong` inside a hot loop
  wastes cycles.
- **Forgetting release/acquire pairing.** A release with no matching acquire (or vice versa)
  publishes nothing — the "flag" is atomic but the *data* behind it races.
- **ABA in raw-pointer CAS structures.**

---

## 7. Quiz — tough problems

**Q1.** Producer: `data = 42; ready.store(true, relaxed);` Consumer: `while(!ready.load(relaxed)); use(data);`. Both atomics are relaxed. On ARM, can `use(data)` read a stale `data`?
**Answer:** **Yes.** `relaxed` gives no ordering, so on a weak model the consumer can see `ready==true`
*before* the `data=42` write is visible → reads garbage. Needs `release`/`acquire`. On x86 (TSO) it
usually *happens* to work — the dangerous illusion.

**Q2.** In the SPSC `push`, why is loading `tail_` (my own index) `relaxed` but loading `head_`
`acquire`?
**Answer:** Only the producer writes `tail_`, so no other thread's ordering matters for it — relaxed
is enough. `head_` is written by the *consumer*; the acquire-load ensures the producer observes the
consumer's freeing of slots (its release-store) so "full" is computed against up-to-date state.

**Q3.** Is `int x=0; std::atomic<int> flag; ... x++ ; flag.store(1,release);` in thread A, and
`if(flag.load(acquire)) x++;` in thread B, a data race on `x`?
**Answer:** It depends on control flow. The acquire/release synchronizes-with *only if* B's load
reads the value A's store wrote. If B reads `1`, happens-before is established and the accesses are
ordered — but they still both *write* `x` with no mutual exclusion of the increment itself; if both
run, `x++` in each is ordered (A before B) so no race, result 2. If B reads `0`, it skips — no race.
The subtlety: acquire/release orders A-before-B *when the flag is observed*, preventing the race.

**Q4.** Why must `N` be a power of two in the ring buffer, and what breaks if it isn't (with `&`)?
**Answer:** `& (N-1)` computes `% N` only when `N` is a power of two (then `N-1` is all-ones in the
low bits). With a non-power-of-two, `& (N-1)` is *not* modulo — indices wrap wrong, you overwrite
live data / skip slots → corruption. Either keep N a power of two or use real `% N` (slower).

**Q5.** You benchmark your "lock-free" `atomic<Order>` queue (Order = 40 bytes) and it's slower than
a mutex version. Why?
**Answer:** 40 bytes > 16-byte max atomic on x86 → `atomic<Order>` is **not lock-free**; the library
uses a hidden internal mutex, *plus* atomic overhead on top. `is_lock_free()` returns false. Store
indices/pointers atomically and copy the payload separately (as the SPSC queue does).

**Q6.** Two threads increment `std::atomic<int> c{0}` a million times each with `fetch_add(1, relaxed)`. Final value? Is relaxed safe here?
**Answer:** Exactly **2,000,000** — relaxed still guarantees atomicity of the RMW (no lost updates),
just no ordering vs *other* memory. Since we only care about the final count, not ordering relative
to other data, relaxed is correct and fastest.

**Q7.** Describe an ABA failure in a lock-free Treiber stack and one concrete fix.
**Answer:** Thread 1 reads `top = A` (next = B), stalls. Thread 2 pops A, pops B, pushes A back
(now next = C). Thread 1's `CAS(top, A, B)` sees `top==A`, succeeds, sets `top=B` — but B was freed;
the stack is corrupt. Fix: tagged pointer (version counter in the high bits, double-width CAS) so
A@v1 ≠ A@v3, making the stale CAS fail.

**Q8.** When is `compare_exchange_weak` preferable to `strong`, and why is `weak` cheaper on ARM?
**Answer:** Prefer `weak` inside a retry loop, where a spurious failure just costs one more
iteration. On ARM/POWER, CAS is built from LL/SC (load-linked/store-conditional); the SC can fail
spuriously (e.g. an interrupt between LL and SC). `weak` maps directly to one LL/SC attempt (cheap);
`strong` must add a loop to mask spurious failures (extra cost).

**Q9.** Your producer and consumer indices are `std::atomic<size_t> head_, tail_;` declared
adjacently with no `alignas`. Throughput is terrible under contention. Diagnose and fix.
**Answer:** `head_` and `tail_` share a cache line → **false sharing** (Module 15): every producer
write to `tail_` invalidates the consumer's line and vice versa, ping-ponging on every op. Fix:
`alignas(64)` each (or `hardware_destructive_interference_size`) so each owns a line.

**Q10.** Is `seq_cst` ever *necessary*, or can everything be done with acquire/release?
**Answer:** Acquire/release only orders operations along a single release→acquire chain; it does
*not* give a single global order across *independent* atomics. Some algorithms (e.g. Dekker-style
mutual exclusion, the "store-load" litmus where two threads each store then load the other's var)
genuinely need the global total order of `seq_cst` to be correct. So yes — `seq_cst` is sometimes
required, which is another reason it's the safe default.

---

## 8. Indian HFT interview questions

Atomics and lock-free are *the* deep-end of C++ HFT interviews at Tower Research (Gurgaon), Optiver,
IMC, Graviton, Quadeye, AlphaGrep, WorldQuant, Da Vinci, Squarepoint, Jump, HRT, NK Securities,
Mansard, Qube, and Millennium desks. Expect to be pushed until you break.

- **"Explain acquire and release precisely."** — Release-store: a one-way barrier that publishes all
  prior writes; nothing before it moves after it. Acquire-load: the matching barrier; when it reads
  a released value, all the producer's prior writes become visible; nothing after it moves before
  it. Together they form a happens-before / synchronizes-with edge. (Say "publish/consume.")

- **"Difference between lock-free, wait-free, and obstruction-free?"** — Lock-free: system-wide
  progress guaranteed, individual thread may starve. Wait-free: *every* thread finishes in bounded
  steps (strongest). Obstruction-free: progress only if a thread runs in isolation (weakest).

- **"Implement / walk through an SPSC queue."** — The code in Section 4: power-of-two ring, `head_`/
  `tail_` on separate cache lines, relaxed for own index + acquire/release across the boundary, one
  empty slot to distinguish full/empty, optional local index caching. Be ready to justify every
  memory_order.

- **"What is the ABA problem and how do you solve it?"** — Value goes A→B→A between read and CAS; CAS
  succeeds on stale identity → corruption. Fix: tagged pointers/version counters, hazard pointers,
  or epoch-based reclamation.

- **"Why isn't every `std::atomic<T>` lock-free?"** — Only types ≤ the CPU's widest atomic
  instruction (16 B on x86-64) have hardware support; larger `T` falls back to an internal library
  mutex. Check `is_lock_free()` / `is_always_lock_free`.

- **"Difference between `volatile` and `std::atomic`?"** — `volatile` stops the *compiler* from
  optimizing away/reordering an access (for MMIO) but gives **no** atomicity and **no** inter-thread
  ordering. `atomic` gives both. `volatile` is not a threading tool.

- **"When would you use `relaxed`?"** — When you need atomicity but not ordering: independent
  counters/statistics, or your own single-writer index in an SPSC queue. Never for publishing data
  another thread must then read.

- **"Why do lock-free bugs appear on ARM but not x86?"** — x86 is TSO (strong): loads act
  ~acquire, stores act ~release by default, hiding missing barriers. ARM is weakly ordered and
  reorders freely, exposing them. You must reason, not test.

- **"What's the cost of a CAS / atomic RMW?"** — It needs exclusive ownership of the cache line
  (a coherency transaction) plus, for `seq_cst`, a fence — tens of ns under contention, and it
  *serializes* competing cores. Under heavy contention a CAS loop can livelock-ish (lots of retries),
  which is why SPSC (no CAS) beats MPMC when you can arrange single writers.

- **"Mutex vs spinlock vs lock-free — when each?"** — Mutex: long critical sections, oversubscribed
  threads (blocking yields the core). Spinlock: very short sections, dedicated cores, low contention
  (busy-wait beats a syscall). Lock-free: predictable tail latency, no thread should ever block —
  the HFT default on the hot path.

---

## 9. In the order book

An `SpscQueue<Message, N>` decouples the network/input thread (**producer**) from the matching
thread (**consumer**): the input thread parses inbound messages off the wire and pushes; the
matching thread pops and processes — no lock, no blocking, bounded and predictable latency.
Crucially, **the matching engine itself is single-threaded**: one thread owns the book. Sharing the
book across cores would mean locks or complex lock-free structures on the hottest data — usually
*slower* (coherency traffic, false sharing, ABA) than a single core running the whole match with the
book resident in its L1/L2. So the concurrency lives at the *edges* (lock-free queues in and out),
not the core. The producer's `tail_` and consumer's `head_` get `alignas(64)` (Module 15) so the two
threads never fight over a cache line — the exact false-sharing fix, applied where it earns its keep.

## Key takeaways

- Mutexes are correct but **block** → unpredictable tail latency; lock-free structures make progress
  without blocking → bounded latency, at the cost of subtlety.
- **Atomic** = indivisible + well-defined; a non-atomic data race is **UB**. `volatile` is *not* a
  threading tool.
- **CAS** (with `weak` inside loops, `strong` outside) is the lock-free primitive; the **ABA
  problem** is its classic hazard (fix: tags/hazard pointers/epochs).
- **Memory ordering:** `relaxed` (atomic, no order) < `acquire`/`release` (publish/consume — the
  workhorse) < `acq_rel` < `seq_cst` (global order, default, slowest). Reason, don't test — x86 TSO
  hides bugs that ARM's weak model exposes.
- The **SPSC ring buffer** needs no CAS: power-of-two size, indices on separate cache lines, relaxed
  own-index + acquire/release across the boundary, one empty slot; optional local index caching cuts
  coherency traffic.
- Not every `std::atomic<T>` is lock-free — verify with `is_lock_free()`; a big `T` hides a mutex.
- HFT design: concurrency at the **edges** (lock-free queues), single-threaded matching core with
  the book cache-resident.

**Next:** [17 — Zero-cost abstraction, CRTP, branch elimination](17-zero-cost-crtp.md) — making
high-level abstractions compile down to the same fast code you'd write by hand.
