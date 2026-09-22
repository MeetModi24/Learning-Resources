# Module 07 — The five systems at a glance (capstone cheat-sheet)

Modules 02–06 built each HFT system component in depth. This module is the **revision sheet**: for
every component, the same five boxes — **functional requirements**, **non-functional requirements**,
**key challenges**, a one-paragraph **complete implementation flow**, and **how it's done the HFT way**
(the concrete techniques: order pools, sharding, lock-free rings, generational handles…). Use it to
prep the "walk me through the whole system" interview and to keep the moving parts straight. Each
section links back to its deep module.

The pipeline, one line:

```text
exchange ─▶ feed handler ─▶ order book ─▶ strategy engine ─▶ OMS / gateway ─▶ exchange
                                                                    │
                              backtester & simulator ◀── replay/validate the whole thing
```

The single organizing idea across all five: **do bounded, allocation-free, deterministic work on one
thread per core; synchronize only at lock-free queue boundaries; recover correctness explicitly.**

---

## 1. Feed handler — *(Module 02)*

> Turns a **fast, lossy, exchange-specific byte stream** into a **reliable, in-order, normalized**
> message stream the rest of the system can trust.

**Functional requirements**
- Receive UDP-multicast packets; deframe (one datagram → many messages).
- Parse each venue message into a typed struct.
- Detect sequence gaps, duplicates, reordering.
- Recover lost messages (retransmission / snapshot).
- Arbitrate the redundant A/B lines.
- Normalize to a venue-agnostic internal message and hand off.

**Non-functional requirements**
- Lowest possible receive latency; no allocation/locks on the hot path.
- No kernel drops under a market-open microburst.
- Correctness over speed while recovering (a stale book must not be traded on).
- Measurable: feed latency (exch-ts vs rx-ts), gap counts, ring fullness.

**Key challenges**
- **Packet loss on UDP** with no built-in recovery.
- **Applying past a gap** (a missed cancel leaves a ghost order → corrupt book).
- **Endianness / struct padding** bugs turning prices into garbage.
- Silent kernel drops; distinguishing "quiet market" from "dead feed."

**Complete implementation flow**
A busy-polling thread pinned to an isolated core pulls a batch of datagrams with `recvmmsg` (or via
Onload/DPDK kernel bypass); for each datagram it reads the packet header, then walks `msg_count`
messages, advancing by each message's own wire `length`. Each message is `memcpy`'d into an aligned
struct, byte-swapped once, and its sequence number classified by the sequencer into
Process/Duplicate/GapAhead/Resync. In-order messages are normalized into a cache-aligned internal
struct and pushed onto a lock-free SPSC ring to the book-builder thread; a gap flips the handler into
recovery (buffer newer messages, request retransmission, or resync from a snapshot) and marks the book
stale so the strategy stops trading until the hole is filled.

**How it's done the HFT way**

| Concern | Technique |
|---|---|
| Receive latency | kernel bypass (Onload/DPDK), **busy-poll** (no wake-up), `recvmmsg` batching, big `SO_RCVBUF` |
| Correct parse | `#pragma pack` structs + `memcpy` into aligned locals + `std::byteswap`; advance by wire length, not `sizeof` |
| In-order guarantee | the **sequencer's 4 verdicts** (Process/Duplicate/GapAhead/Resync) |
| Loss recovery | **A/B arbitration first** (dedup by seq), then retransmission (small gap), then **snapshot** (big gap) |
| Decouple from consumer | lock-free **SPSC ring**, cache-line-aligned normalized messages (no false sharing) |

---

## 2. Order book — *(Module 03)*

> Maintains the live limit order book per instrument with **price-time (FIFO) priority**, so best
> bid/ask and add/cancel/execute are all fast.

**Functional requirements**
- Add / modify / cancel limit orders; match market orders; support stop / stop-limit.
- Maintain price levels sorted; FIFO order within a level.
- O(1) best-bid / best-ask; O(1) lookup of an order by id.
- Report trades on a match.

**Non-functional requirements**
- Sub-microsecond add/cancel; no allocation on the hot path.
- Cache-friendly memory layout; predictable (low-jitter) latency.
- Single-threaded per instrument (deterministic).

**Key challenges**
- **Best price in O(1)** while keeping levels ordered.
- **Order cancel in O(1)** (find and unlink an arbitrary resting order).
- Avoiding pointer-chasing cache misses and hot-path `new`/`delete`.
- (In the real repo:) an **uncached AVL height** → inserts drift toward O(M); the differentiator is to fix or replace it.

**Complete implementation flow**
A normalized message arrives from the feed handler's ring. On an add, the book finds (or creates) the
`Limit` for that price, appends the `Order` to the tail of that level's intrusive FIFO list, and
updates the id→order map and the best-price edge pointer if this order improves it. On a cancel, the
id map locates the order in O(1) and it's unlinked from its level's doubly-linked list. A marketable
order walks from the best-price edge, consuming resting orders FIFO until filled, emitting trades and
advancing the edge pointer as levels empty.

**How it's done the HFT way**

| Concern | Technique |
|---|---|
| O(1) best bid/ask | **edge pointers** (`lowestSell`/`highestBuy`) cached, updated on the fly |
| O(1) order cancel | **id→order map** + **intrusive doubly-linked list** at each level → unlink in place |
| FIFO time priority | append to list tail; match from head |
| Price levels | rung-1 **AVL tree**; rung-3 **flat price array + bitset + `std::countr_zero`** (no rebalance) |
| No hot-path alloc | **object pool** for `Order`/`Limit`; rung-3 uses **32-bit indices, not pointers** for cache packing |
| Fast lookup | rung-3: sequential ids → **direct-indexed vector** instead of `unordered_map` |

---

## 3. Strategy engine — *(Module 04)*

> The decision brain: consumes market/fill events, computes signals, decides orders — deterministically
> and fast, at scale (e.g. **100 strategies over 10,000 instruments on 8 cores**).

**Functional requirements**
- Dispatch events (`onBookUpdate`/`onTrade`/`onFill`/`onTimer`) to the strategies subscribed to each instrument.
- Compute signals incrementally (imbalance, microprice, rolling stats).
- Decide target orders and reconcile against live orders (send/modify/cancel).
- Track position/PnL from fills; run identical code live and in backtest.

**Non-functional requirements**
- Deterministic (same events → same orders).
- Low tick-to-decision latency; no allocation/locks on the hot path.
- Scales horizontally across cores; parity between live and backtest.

**Key challenges**
- **Scaling** 100 strategies × 10k instruments without a thread per strategy.
- Staying **lock-free** without sacrificing correctness.
- **Determinism** despite asynchronous events and timers.
- Minimizing order messages (OTR) while staying responsive (adverse selection).

**Complete implementation flow**
Instruments are partitioned into **shards**; each shard is one thread pinned to one core and owns a
disjoint set of instruments *and* the strategies on them. A market-data event lands on the shard's
preallocated event-queue ring; the shard's single thread pops it, and the dispatcher looks up
`subscribers[instrument]` and calls each subscribed strategy's callback **sequentially**. A strategy
updates its incremental signals (ring buffer / EWMA), computes target quotes, and emits order intents
through an `OrderRouter`; the engine mints a ClientOrderId, records `orderOwner[id] = strategy`, and
passes the order to risk → OMS. Fills come back as events tagged with that id and are routed to the
owning strategy's `onFill`. Because only one thread touches a shard's state, no locks are needed;
threads meet only at lock-free queues (feed→shard, shard→gateway, and the off-hot-path work queue).

**How it's done the HFT way**

| Concern | Technique |
|---|---|
| Scale (100×10k on 8 cores) | **sharding**: one shard = one thread/core owning disjoint instruments + their strategies (not one thread per strategy) |
| Lock-free | **single-threaded per shard** → `position += …` needs no mutex/atomic; sync only at **lock-free ring** boundaries |
| Event publish→consume | preallocated **ring-buffer event queue**; dispatcher fans to `subscribers[instrument]`; strategies run **sequentially** |
| Determinism | **event-time virtual clock** (never wall clock); total order by `(timestamp, seq)`; no `unordered_map`-iteration decisions; seeded RNG |
| Incremental signals | **ring buffer + running sum** or **EWMA**, not recompute-over-window |
| Live/backtest parity | events + `OrderRouter` are the only interfaces; swap the source, keep the binary |
| Correlated instruments | **co-locate** in one shard (a pairs strategy needs both legs on the same thread) |
| Slow work | pushed **off the hot path** over a lock-free queue to a background thread |

---

## 4. OMS / order gateway — *(Module 05)*

> Turns order intents into **sequenced messages on a reliable session** and maintains the
> **authoritative state of every order and position** — never losing or duplicating one.

**Functional requirements**
- Drive the order lifecycle state machine (PendingNew→Working→Partial/Filled, +Cancel/Replace, terminals).
- Serialize orders to the venue wire format; decode exec reports.
- Route orders across sessions; throttle to exchange limits; fail over.
- Track positions; persist every event for audit; reconcile after disconnect.

**Non-functional requirements**
- **Never lose or duplicate an order**; idempotent, durable, auditable (SEBI).
- Low order-path latency; no I/O on the hot path.
- Single writer per session (no locks); deterministic.

**Key challenges**
- **Exchange fills vs order cancel — the cancel/fill race** (a fill and a cancel cross on the wire).
- **Late/duplicate exec reports** (resend) must not double-count or use-after-free.
- Respecting **rate limits / OTR** without adding latency or head-of-line blocking.
- **Recovery** after a disconnect with live orders in an unknown state.

**Complete implementation flow**
A strategy's intent arrives on a lock-free MPSC ring into the gateway manager, which routes it by
instrument to the correct gateway; that gateway's **single writer thread** performs the pre-trade risk
check, allocates an order slot from a pool (its ClOrdID *is* a generational handle into that pool),
sets state `PendingNew`, and — if the session's token bucket allows — serializes the order into packed
bytes and writes it to the TCP socket (`TCP_NODELAY`, or Onload). Exec reports return on the same
session: an Ack flips the order to Working; a Fill clamps to `leavesQty` and updates cumulative
quantity/position idempotently; the cancel/fill race resolves because the exchange is treated as truth
(a Fill during PendingCancel just reduces leaves, and the following CancelReject settles the state).
Every event is dropped onto an SPSC ring to a drainer thread that writes the audit log, the time-series
store, and the monitoring feed — never blocking the order path. On disconnect, the session resends,
and the OMS reconciles its store against an exchange snapshot / drop-copy before resuming.

**How it's done the HFT way**

| Concern | Technique |
|---|---|
| O(1) order lookup, no hash | **object pool + generational-handle ClOrdID** used as a **direct array index**; venue echoes ClOrdID |
| Late/dup report safety | **generation bits** → stale handle returns `nullptr` (safe drop, no use-after-free) |
| Cancel/fill race | **exchange-as-truth + idempotent** fills (clamp to `leavesQty`); Fill wins, CancelReject settles |
| No locks | **single writer per session** owns socket + seqnum + token bucket + order store |
| Cross-core hand-off | strategies → gateways over **lock-free MPSC rings**, cache-line padded |
| Routing / throttling | **direct-indexed** routing table; **token bucket** (integer, no sleep) per session; OTR-aware |
| No HOL blocking / failover | one ring + writer **per gateway**; failover flips the routing entry to a backup session |
| Wire encoding | packed structs + `byteswap` + `memcpy` + `std::span`; **CRTP** encoder for multi-venue (no vtable) |
| Telemetry off hot path | event → **SPSC ring** → drainer → append-only **audit log** + batched **QuestDB** (PG-wire) + monitoring |
| Recovery | **exchange-is-truth**: session resend, reconcile via snapshot/drop-copy, cancel-on-disconnect |

---

## 5. Backtester & exchange simulator — *(Module 06)*

> Two replay tools with different questions: **backtester** = *"would the strategy have made money?"*;
> **simulator** = *"would the trading system have behaved correctly?"*

**Functional requirements**
- *Backtester:* replay recorded market data through the strategy; model fills; compute P&L/positions/stats.
- *Simulator:* stand in for the exchange behind the real gateway protocol; emit ACK/Fill/Reject/Cancel; run a matching engine + latency model; inject faults.
- Both: reproduce an exact run for debugging.

**Non-functional requirements**
- **Deterministic** event-time replay; reproducible (seeded).
- Fast (months of data ≫ real time); no allocation on the replay hot path.
- **Parity**: same strategy/OMS binaries as live — swap only the market/exchange side.

**Key challenges**
- **Fill realism** in the backtester (queue position, not "price touched → filled").
- **Fidelity** in the simulator (a real matching engine + latency, not instant fills).
- Keeping runs **bit-for-bit reproducible**.
- Faithfully reproducing **ugly cases** (cancel/fill race, dup fills, disconnects).

**Complete implementation flow**
The replay engine memory-maps a binary file of packed, timestamped records and iterates them in
`(timestamp, seq)` order; before each event it advances a **virtual clock** that the strategy reads
instead of the wall clock, then calls the strategy through the same callbacks used live. In the
backtester, resulting orders hit a **fill model** that tracks the volume ahead of the order at its
price and consumes it using historical trade prints, producing realistic partial fills and P&L (net of
real costs). In the simulator, orders instead flow through the real OMS and gateway into a fake
exchange that reuses the Module 03 matching engine over a reconstructed book; a seeded **min-heap
scheduler** applies per-hop latency so the order reaches the matching engine microseconds after the
decision, and exec reports flow back to the OMS — and the scheduler can be scripted to deliver a Fill
before a CancelReject (or duplicate/out-of-order/disconnect events) to validate the OMS state machine.

**How it's done the HFT way**

| Concern | Technique |
|---|---|
| Determinism | **virtual clock** (event time, not `system_clock`); total order by `(ts, seq)`; **seeded** RNG (seed logged) |
| Zero-copy fast replay | **mmap** binary records + `std::span`; packed structs + `byteswap`; **CRTP** engine (no per-event vtable) |
| Live/backtest/sim parity | same strategy + `OrderRouter`/gateway interfaces; swap only event source + exchange impl |
| Backtester fill realism | **queue-position model**: volume-ahead consumed by trade prints; realistic costs (STT, fees, GST) |
| Simulator fidelity | **reuse the Module 03 matching engine** behind the **real gateway protocol** |
| Latency modeling | **seeded min-heap scheduler** of timestamped events (per-hop latency) |
| System validation | **fault injection**: script cancel/fill race, duplicate fills, rejects, disconnects → assert OMS transitions |

---

## The through-line (say this if asked to tie it all together)

> Market data leaves the exchange fast and lossy; the **feed handler** makes it trustworthy and hands
> it off over a lock-free ring. The **order book** turns that stream into a live, FIFO-priority book
> with O(1) best price and cancel. The **strategy engine** — sharded one-thread-per-core, deterministic,
> lock-free except at ring boundaries — reads the book, computes incremental signals, and emits intents.
> The **OMS** risk-checks them, drives an asynchronous order state machine where the exchange is truth,
> routes across single-writer gateway sessions with token-bucket throttling, and logs everything off the
> hot path for audit and recovery. The **backtester and simulator** replay recorded data deterministically
> to answer, respectively, *is the strategy profitable?* and *is the system correct?* — using the same
> strategy and OMS binaries as production. The recurring pattern everywhere: **bounded allocation-free
> deterministic work on one core, synchronize only at lock-free queues, and recover correctness
> explicitly.**

---

**Next:** Module 08 — the options pricing / greeks engine *(planned)*. Return to the
[index](../00-index.md), or revisit any deep module: [02 feed handler](02-feed-handler.md) ·
[03 order book](03-order-book-project.md) · [04 strategy engine](04-strategy-engine.md) ·
[05 OMS](05-oms-order-gateway.md) · [06 backtester & simulator](06-backtester-and-simulator.md).
