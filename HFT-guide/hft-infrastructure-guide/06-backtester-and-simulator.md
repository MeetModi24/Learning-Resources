# Module 06 — The backtester & the exchange simulator (deep dive)

These two components are easy to confuse — both replay historical data — but they answer **different
questions**:

> **Backtester** = *"Would this strategy have made money?"*
> **Simulator** = *"Would the strategy/order system actually have behaved correctly in a realistic
> exchange environment?"*

The backtester is a **research** tool: feed a strategy old market data and measure its P&L. The
simulator is a **system-validation** tool: stand in for a real exchange so your whole stack — strategy
→ OMS → gateway — can be exercised against ACKs, fills, rejects, cancels, delays, and disconnects
without touching a real market. This module builds both, following the same shape as Module 05:
**Part I — core concepts**, then **Part II — design problems** worked brute→HFT with the C++/OS/CA
tools you've built actually put to use.

> Prerequisites we lean on: object lifetime & the heap (01/03), templates/concepts & CRTP (10/17),
> the STL cost model (12), `noexcept` (14), cache/alignment (15), lock-free rings (16),
> `<bit>`/`std::span`/`byteswap` (18); and from this folder: the order book & matching engine (Module
> 03 — the simulator *reuses* it), the strategy engine's determinism + `OrderRouter` boundary (Module
> 04 §1.3–1.4, §4.2), and the OMS order state machine + cancel/fill race (Module 05 §4, Problem 1).

---

# Part I — Core concepts

## 1. The core distinction (memorize this first)

| | **Backtester** | **Simulator** |
|---|---|---|
| Main purpose | Evaluate the **strategy** | Test the **trading system** |
| Input | Historical market data | Orders **+** market data |
| Main focus | Signals, fills, P&L | OMS / gateway / exchange behavior |
| "Exchange" | A **simplified fill model** | A **simulated exchange** (matching engine) |
| Tests cancel/fill races | Sometimes | **Yes** |
| Tests disconnect/recovery | Usually no | **Yes** |
| Tests order state machine | Simplified | **Yes** |
| Tests strategy profitability | **Yes** | Yes, but not its primary purpose |
| Latency modeling | Usually basic | **Important** |
| Matching engine | Usually simplified | More realistic |

Everything below elaborates this table. The mental shortcut: the **backtester replaces the exchange
with a fill assumption**; the **simulator replaces the exchange with an exchange** (a fake one that
talks the same protocol).

## 2. The backtester

A backtester takes historical market data and runs a strategy against it to evaluate its performance.
Suppose you have historical AAPL data:

```text
10:00:00.001  AAPL 250.00
10:00:00.002  AAPL 250.05
10:00:00.003  AAPL 250.10
...
```

The backtester replays these events in timestamp/sequence order and pipes them through the strategy:

```text
Historical Data
      ↓
  Replay Engine
      ↓
   Strategy
      ↓
 BUY / SELL decisions
      ↓
 Simulated fills
      ↓
 P&L / positions / statistics
```

Instead of connecting your strategy to a real exchange, you give it old market data and ask: *"If I
had run this strategy yesterday, what would have happened?"*

**Requirements** (each one is a design constraint we'll honor in Part II):

- **Deterministic replay** — the same historical input must produce the same result every time
  (Module 04 §1.4). Without this, a P&L number is meaningless and bugs are unreproducible.
- **Correct event ordering** — events replayed in their original sequence, ties broken by sequence
  number.
- **Fast execution** — you may want to replay *months* of data far faster than real time.
- **Realistic transaction costs** — fees, spread, slippage. In India this is large: STT, stamp duty,
  exchange fees, GST, SEBI turnover fee — enough to flip a "profitable" strategy negative (the TCA
  point from Module 01).
- **Position / P&L tracking.**
- **Reproducibility** — the ability to replay one exact run for debugging.

The strategy sees *exactly the same sequence of events it would have seen live* — which is only true if
you get the next point right.

## 2.1 Parity: the same strategy code, live and backtest

You do **not** want the backtester to use a different strategy API from production — otherwise you test
one implementation and trade with another. Only the **source of events** should change:

```text
              Strategy                       (identical code both sides)
                 │
       ┌─────────┴─────────┐
       ↓                   ↓
    Live Feed          Backtest Replay
       │                   │
Exchange→FeedHandler   Historical file→Replay Engine
       ↓                   ↓
     Strategy           Strategy
```

This is exactly the `OrderRouter` / event-source boundary from Module 04 §1.3: swap what feeds the
event queue, keep the strategy binary fixed.

## 2.2 The replay engine

A simplified version:

```cpp
while (reader.next(event)) {
    clock.advance(event.timestamp);   // the backtest CONTROLS time
    strategy.onMarketData(event);
}
```

```text
10:00:00.001 → Quote AAPL 250    → strategy.onQuote()
10:00:00.002 → Trade AAPL 250.05 → strategy.onTrade()
10:00:00.003 → Quote AAPL 250.10 → strategy.onQuote()
```

The crucial detail: **the backtest controls time.** The strategy must never call
`std::chrono::system_clock::now()` (that returns *today's* wall clock); it calls `simulatedTime.now()`,
which returns the timestamp of the historical event currently being processed. Same discipline as the
strategy engine's "event time, not wall clock" (Module 04 §1.4, points 21–22) — here it's enforced by
the replay engine owning the clock.

## 3. The hard problem in HFT backtesting: fills

Imagine the strategy says `BUY 100 AAPL @ 250.00` and the historical data shows `Best Ask = 250.00`.
Can we just say *"100 shares filled"*? **Not necessarily** — maybe there were already **5,000 shares
ahead of you** in the exchange queue.

This is why an HFT backtester needs a **fill model**. The naive model:

```text
if (market ask <= my price) → fill        // overly optimistic!
```

A better model considers: price, available quantity, **trade prints**, **queue position**, order
arrival time, latency, and cancel activity. So the real question shifts from:

> *"Would my signal have been correct?"* → *"Could I realistically have gotten this fill?"*

That difference is enormous, and it's the single biggest source of backtest over-optimism. We build a
queue-position fill model in **Problem 2**.

## 4. The simulator

The simulator tests the **trading system and order/exchange interaction**, not merely strategy
profitability. It *pretends to be an exchange*:

```text
Backtester:   Historical market data → Strategy → P&L
Simulator:    Strategy → OMS → Gateway → Simulated Exchange
                                              ↑ realistic fills
```

```text
Strategy → BUY 100 AAPL @ 250 → OMS → Gateway → SIMULATED EXCHANGE → ACK → FILL / REJECT / CANCEL
```

This lets you test the **entire order lifecycle** (Module 05 §4) without sending real orders.

## 5. What the simulated exchange actually does

Your system sends:

```text
NEW  ClOrdID=1001  BUY 100 AAPL @ 250
```

The simulator responds like a real venue:

```text
ACK  OrderID=EX5001
```

Then, as market conditions change, it generates fills:

```text
FILL  ClOrdID=1001  Qty=40  Price=250
FILL  ClOrdID=1001  Qty=60  Price=250
```

Your **real OMS** sees exactly the sort of messages it would receive from a real exchange — which is
the whole point. And it does this over the **same gateway protocol interface** (Module 05 Problem 2/3),
so the gateway can't tell whether the other side is real or simulated.

## 6. Why a simulator if we already have a backtester?

Because they test different things. The backtester answers *"does my strategy make money?"*; the
simulator answers *"does my trading infrastructure behave correctly?"* — and the second lets you
deliberately exercise the ugly cases that a fill-model backtester can't:

```text
cancel/fill race • partial fills • duplicate exec reports • exchange rejects •
sequence gaps • disconnect/reconnect • cancel-on-disconnect • rate limits •
order state transitions • failover • recovery/reconciliation
```

Every one of those is a Module 05 correctness concern. The simulator is where you prove the OMS state
machine survives them — safely, before real money is at stake.

## 7. How they work together

In a mature stack you run both off the same recorded data:

```text
              Historical Data
          ┌─────────┴─────────┐
          ↓                   ↓
     Market Replay       Market Replay
          ↓                   ↓
      Strategy             Strategy
          ↓                   ↓
      Backtester          OMS/Gateway
          ↓                   ↓
         P&L          Exchange Simulator
                              ↓
                       Execution Reports
                              ↓
                             P&L
```

The first path gives **fast strategy research**; the second gives **realistic end-to-end system
testing**. Same strategy code, same OMS interfaces — only the market/exchange implementation is
swapped.

---

# Part II — Design problems

Same method as Module 05: pose it, reason naive → critique → HFT, and use the modules where their tools
fit.

## Problem 1 (LLD) — the deterministic replay engine + virtual clock

### 1.1 Statement

> Build the engine that reads a recorded market-data file and drives a strategy through it: replay in
> `(timestamp, sequenceNumber)` order, let the **backtest control time**, run **months of data faster
> than real time**, and be **bit-for-bit reproducible**. The strategy binary must be the *same* one
> used live.

### 1.2 Naive version

```cpp
std::vector<Event> events = parseCsv("day.csv");         // whole file into a vector of heap objects
std::sort(events.begin(), events.end(), byTime);
for (const Event& e : events) {
    auto now = std::chrono::system_clock::now();          // ← wall clock! non-deterministic
    strategy.onMarketData(e);
}
```

**Critique:** loads and heap-allocates the whole day (Modules 01/03); parses CSV (slow, lossy for
prices — Module 02 §2.3 on integer prices); reads the **system clock** so runs aren't reproducible and
the strategy "sees" today's time (Module 04 §1.4); and it's a different code path from production.

### 1.3 HFT version

**(a) Recorded data is a binary file of packed records** — the same format the feed handler parses
(Module 02 §2.3): fixed-width, integer prices, `#pragma pack`. **Memory-map** it (`mmap`) and iterate
in place as a `std::span` (Module 18) — no per-event allocation, no copy, the OS pages it in lazily:

```cpp
std::span<const RecordedMsg> recs = mmap_records(fd);     // zero-copy view over the file (M18)
```

If a day spans multiple instrument files, do a **k-way merge** by `(ts, seq)` with a small min-heap —
never load-then-sort the world.

**(b) A virtual clock the strategy reads instead of the wall clock.** The replay engine owns "now":

```cpp
class VirtualClock {
    uint64_t now_ns_ = 0;
public:
    void advance(uint64_t t) noexcept { now_ns_ = t; }    // set by the replay loop, per event
    uint64_t now() const noexcept { return now_ns_; }
};

for (const RecordedMsg& r : recs) {
    clock_.advance(r.ts_ns);                              // event time, not system_clock (M04 §1.4)
    strategy_.onMarketData(decode(r));                    // decode = memcpy+byteswap, M02 §4 / M18
}
```

Because the loop runs as fast as the CPU allows and time only advances via `clock_.advance`, you get
**months-in-minutes** replay *and* determinism for free. Timers the strategy sets are events on this
same clock, so they fire reproducibly.

**(c) Parity via the interface, zero-cost.** The strategy takes its market data through the same
callback and acts through the same `OrderRouter` (Module 04 §1.3) in both worlds. To avoid a virtual
call per event on the hot replay path, template the engine on the concrete strategy / event source
(CRTP, Module 17) so it inlines:

```cpp
template <class Strat, class Source>                      // Source = LiveFeed or FileReplay
class Engine { Strat strat_; Source src_; VirtualClock clock_; /* run() inlines onMarketData */ };
```

**(d) Reproducibility knobs.** Any randomness (e.g. a stochastic fill model, Problem 2) uses a
**seeded** PRNG stored in the run config; log the seed with the results so a run replays exactly. No
`unordered_map` iteration driving decisions (Module 04 §4.2 point 22).

### 1.4 Tradeoffs

| Decision | Tradeoff |
|---|---|
| **mmap binary vs parse CSV** | Binary+mmap is zero-copy and fast but needs the recorded format; CSV is human-readable but slow and float-lossy. Record binary, keep a CSV exporter for eyeballing. |
| **Pre-sort file vs k-way merge at replay** | Pre-sorted single file → simplest, fastest loop; k-way merge → flexible across many instrument files but a heap-pop per event. Pre-sort when you can. |
| **CRTP engine vs virtual `Strategy`** | CRTP inlines (fast, but a template instantiation per strategy type); virtual is flexible for running many strategy types in one binary. Research harnesses often accept virtual; the hottest loops use CRTP. |
| **Replay as fast as possible vs paced** | Max speed for research throughput; real-time pacing only when a human is watching a replayed session. |

## Problem 2 (LLD) — the backtester's fill model (queue position)

### 2.1 Statement

> Decide, deterministically, **whether and how much** of a resting order fills as historical events go
> by — accounting for the **liquidity ahead of you** in the queue, not just "price touched my level."

### 2.2 Naive version

```cpp
if (order.side == Buy && bestAsk <= order.price) order.filled = order.qty;  // instant, full fill
```

**Critique:** ignores queue position entirely — assumes you're always first in line. This is the
classic reason a backtest looks brilliant and live trading loses: you model fills you'd never actually
get.

### 2.3 HFT version — a queue-reactive model

Track your order's **volume ahead** at its price level, and consume it using the historical **trade
prints** and cancels:

```cpp
struct RestingOrder {
    int64_t  price;
    uint32_t qty;            // your remaining size
    uint64_t volumeAhead;    // shares queued in front of you at this price (from book state on arrival)
};

// On a historical TRADE print at your price level:
void onTradeAtLevel(RestingOrder& o, uint64_t tradedQty) noexcept {
    if (tradedQty <= o.volumeAhead) { o.volumeAhead -= tradedQty; return; }  // all ate into the queue ahead
    uint64_t reachesYou = tradedQty - o.volumeAhead;         // spilled past the queue to you
    o.volumeAhead = 0;
    uint32_t fill = std::min<uint64_t>(reachesYou, o.qty);
    o.qty -= fill;                                            // ← a realistic partial fill
    // emit Fill(fill @ o.price) to the strategy
}
```

Refinements that make it faithful: when a **cancel** ahead of you is observable in the data, decrement
`volumeAhead`; set `volumeAhead` from the **book depth at the moment your order arrived** (which needs
the book state — reuse the Module 03 order book to reconstruct it from the recorded stream); and apply
**latency** (Problem 3's model) so your order joins the queue at `arrival = decisionTime + latency`, by
which point the queue may differ. The model turns *"could I have gotten this fill?"* into a number
instead of an optimistic assumption.

### 2.4 Tradeoffs

| Model | Realism | Cost |
|---|---|---|
| Touch-price → full fill | Worst (over-optimistic) | trivial |
| Pro-rata / fraction of prints | Better | cheap |
| **Queue-position (volume-ahead)** | Good — models being late in line | needs book reconstruction + trade prints |
| Full order-book simulation (Problem 3) | Best | it's the simulator |

The lesson mirrors the strategy engine's signal problem: the naive answer assumes; the good answer
*maintains queue state incrementally* from the data you actually have.

## Problem 3 (HLD) — the exchange simulator behind the gateway protocol

### 3.1 Statement

> Build a fake exchange that plugs in **behind the gateway's protocol interface** (Module 05), accepts
> orders, and emits ACK / Fill / Reject / Cancel messages a real OMS would recognize — with a
> **matching engine**, a **latency model**, and **deterministic** scheduling.

### 3.2 How to think about it

Two big reuses: the **gateway protocol interface** is already an abstraction (Module 05 Problem 2/3), so
the simulator is just *another implementation of the exchange side* — the gateway never knows the
difference. And the **matching engine is Module 03** — the simulator *is* your order book with a
message front-end. The new pieces are a **latency model** and **deterministic event scheduling**.

```text
                   Historical Market Data
                           ▼
                    Market Data Replay        (Problem 1)
                           ▼
                       Strategy               (same binary, Module 04)
                           ▼
                         OMS                   (Module 05)
                           ▼
                     Order Gateway             (same protocol interface, Module 05)
                           ▼
                 ┌─────────────────────┐
                 │  Exchange Simulator │
                 │  Matching Engine    │  ← Module 03 order book
                 │  Order Book         │
                 │  Latency Model      │
                 │  Fill Model         │
                 │  Reject Model       │
                 └──────────┬──────────┘
                            ▼
                    Execution Reports  → OMS
```

### 3.3 The simulated exchange needs an order book

For realistic HFT behavior the simulator maintains a book (reconstructed from the replayed market data
plus your own orders):

```text
ASK  251.00 × 200 / 250.50 × 100 / 250.00 × 500
BID  249.50 × 300 / 249.00 × 400 / 248.50 × 100
```

Your `BUY 100 @ 250` either crosses (immediate fill against resting asks) or **rests** and gets a queue
position:

```text
250.00:  existing 500  +  your 100     → you are behind 500
```

When a historical `Trade 200 @ 250` occurs, the simulator **consumes the 500 ahead of you first** —
exactly Problem 2's queue logic, but now driven by the real matching engine (Module 03), which is far
more faithful than "price touched 250 → fill me."

### 3.4 The latency model

In HFT you cannot pretend everything is instant. There's latency at every hop:

```text
Market event → Strategy → Order generation → Queue → Gateway → Network → Exchange → Matching engine
```

The simulator assigns each hop a cost, e.g. `md=5µs, strat=2µs, gw=1µs, net=10µs, exch=3µs`. So an
order decided on a market event at `12:00:00.000000` reaches the simulated matching engine at
`12:00:00.000021` — and **the market may have moved during those 21µs**, which is precisely the effect
that separates a realistic sim from a toy.

*Implementation:* a **min-heap priority queue of timestamped events**, keyed by simulated arrival time.
The simulator pops the earliest event, advances the virtual clock (Problem 1) to it, processes it
(match, or deliver an exec report), and pushes any consequent events with their own future timestamps.
Deterministic because ordering is total on `(time, seq)` and any stochastic latency draws from a
**seeded** PRNG.

```cpp
struct SchedEvent { uint64_t at_ns; uint64_t seq; EvKind kind; /* payload */ };
std::priority_queue<SchedEvent, std::vector<SchedEvent>, ByTimeThenSeq> pq_;   // deterministic order
```

### 3.5 Tradeoffs

| Decision | Tradeoff |
|---|---|
| **Reuse Module 03 engine vs a bespoke sim book** | Reuse = one matching implementation for prod-sim fidelity and less code; bespoke = simpler but risks sim/real divergence. Reuse. |
| **Same gateway protocol vs a sim-only API** | Same protocol means the OMS/gateway are tested for real; a sim-only API is easier but tests less. Use the real protocol. |
| **Latency model fidelity** | Constant per-hop latency is cheap but crude; distributions (seeded) capture jitter/tail but need tuning to measured numbers. Start constant, refine with real p99.9 data. |
| **Determinism vs randomness** | Seeded PRNG gives reproducibility *and* variety (re-run with new seeds for a distribution of outcomes). Never use an unseeded/global RNG. |

## Problem 4 (design) — fault injection: proving the OMS survives the ugly cases

### 4.1 Statement

> Use the simulator to **deterministically reproduce** the failure modes a real exchange throws, and
> verify the OMS state machine (Module 05 §4, Problem 1) handles each correctly.

### 4.2 The approach

Because the simulator owns the exec-report stream and a deterministic scheduler (Problem 3), you can
**script** ugly situations that are almost impossible to trigger on demand against a real venue:

```text
ACK delay • Fill delay • Duplicate fill • Out-of-order message • Sequence gap •
Reject • Disconnect • Reconnect • Cancel/fill race
```

The headline case — the **cancel/fill race** (Module 05 §4): script the simulator to let a Fill arrive
*before* the CancelReject, then assert the OMS walks the right path:

```text
Strategy → Cancel
Simulator ├── Fill arrives first
          └── CancelReject arrives later

OMS must do:  PendingCancel → Fill → Filled → CancelReject → ignore/settle
```

Other high-value scripts: a **duplicate fill** (assert idempotency — the clamp-to-`leavesQty` of Module
05 Problem 1 must not double-count); a **disconnect mid-order** then reconnect (assert reconciliation,
Module 05 Problem 5, and cancel-on-disconnect behavior); a **reject** on an over-limit order (assert the
risk gate); a **sequence gap** (assert session resend handling). Each is a deterministic, repeatable
test — *"much safer than discovering that bug with real money."*

### 4.3 Why this belongs in the simulator, not the backtester

The backtester's simplified fill model has no order state machine to break — it just computes P&L. Only
the simulator carries the full OMS/gateway/exchange conversation, so only it can exercise the
transitions, races, and recovery paths. This is the concrete reason both tools exist.

---

# Interview Q&A

**Q. "Backtester vs simulator — what's the difference?"** The backtester replays historical market data
through the strategy and measures P&L with a *fill model* — it answers "would this strategy have made
money?". The simulator stands in for the exchange behind the real gateway protocol, running a matching
engine + latency model, and answers "does my order system behave correctly?" — cancel/fill races,
rejects, disconnects, recovery. Different questions, different fidelity.

**Q. "How do you keep a backtest deterministic?"** The replay engine owns a virtual clock; the strategy
reads event time, never `system_clock::now()`. Events replay in `(timestamp, seq)` order, any RNG is
seeded and the seed logged, and no decision depends on `unordered_map` iteration order. Same input →
same P&L, always.

**Q. "Why is a naive fill model dangerous?"** "Price touched my level → I'm filled" assumes you're
first in the queue. In reality thousands of shares may be ahead of you. A queue-position model consumes
the volume ahead using trade prints (and cancels) before filling you — the difference between a
backtest that looks great and live trading that loses.

**Q. "How does the simulator produce realistic fills?"** It reuses the Module 03 matching engine over a
real order book reconstructed from the replayed data plus your orders; your resting order gets a queue
position and only fills when trades consume the liquidity ahead of it — plus a latency model so your
order arrives µs after your decision, by which point the book may have moved.

**Q. "How do you test a cancel/fill race without real money?"** Script the simulator's deterministic
scheduler to deliver the Fill before the CancelReject, then assert the OMS walks PendingCancel → Fill →
Filled → (ignore CancelReject). Repeatable every run.

**Q. "How do the same strategy and OMS run in live, backtest, and sim?"** Everything hangs off two
interfaces: the event source and the `OrderRouter`/gateway protocol. Live plugs in the feed + real
exchange; backtest plugs in file replay + a fill model; sim plugs in file replay + a simulated exchange
on the same protocol. The strategy and OMS binaries don't change.

**Q. "Why binary + mmap for the recorded data?"** Zero-copy, no per-event allocation, integer prices
(no float drift), and it's the same packed format the feed handler parses — so replay reuses the decode
path. CSV is only for human inspection.

---

# Key takeaways

- **Two tools, two questions.** Backtester → *"would the strategy have made money?"* (research, fill
  *model*). Simulator → *"would the system have behaved correctly?"* (validation, simulated
  *exchange*). Don't conflate them.
- **Both live or die by determinism and parity.** A virtual clock the engine controls (event time, not
  wall clock), `(timestamp, seq)` ordering, seeded RNG, and the *same* strategy/OMS interfaces as live —
  otherwise you test one thing and trade another.
- **The backtester's crux is the fill model.** Queue position (volume-ahead consumed by trade prints),
  not "price touched → filled." This is the #1 source of backtest over-optimism.
- **The simulator's crux is fidelity through reuse.** It plugs in behind the real gateway protocol
  (Module 05) and reuses the real matching engine (Module 03), adds a latency model (a seeded,
  deterministic min-heap scheduler), and can *inject* the ugly cases (cancel/fill race, duplicate fills,
  disconnects) to prove the OMS state machine (Module 05).
- **Every optimization is a concept you built:** mmap + `std::span` + packed structs + byteswap (M18/M02)
  for zero-copy replay; CRTP (M17) to keep the hot replay loop virtual-call-free; the Module 03 book as
  the sim matching engine; seeded PRNG + `(time,seq)` total order for reproducibility.
- **Realistic costs matter** — in India especially (STT, stamp duty, fees, GST); a backtester that
  ignores them lies.

---

# Where this connects

- **Module 03 (order book)** — the simulator's matching engine *is* your LOB; the backtester reuses it
  to reconstruct book state for queue-position fills.
- **Module 04 (strategy engine)** — the determinism rules (event time, `(ts,seq)` order) and the
  event-source / `OrderRouter` parity boundary are exactly what make replay valid; the same strategy
  binary runs live/backtest/sim.
- **Module 05 (OMS)** — the simulator plugs in behind the gateway protocol and drives the OMS state
  machine; fault injection (Problem 4) validates its cancel/fill race, idempotency, and recovery paths.
- **Modules 17/18** — CRTP for the zero-cost replay loop; `std::span`/`byteswap`/mmap for zero-copy
  binary replay of the recorded feed.
- **QuantStream stack** — recorded market data and run results live in the intraday-retention store
  (QuestDB via PG-wire batched inserts, per the project's ingestion note); results/P&L are what TCA
  (Module 01) consumes.

---

# The mental through-line

> Both tools replay history, but they interrogate different layers. The **backtester** swaps the
> exchange for a *fill assumption* and asks whether the *strategy* is any good — so its honesty lives
> entirely in the fill model and the transaction costs. The **simulator** swaps the exchange for a
> *fake exchange that talks the real protocol* and asks whether the *system* is correct — so its value
> lives in the matching engine, the latency model, and its ability to inject the failures that would
> otherwise only show up with real money on the line. Build both on a deterministic, event-time replay
> with the same strategy and OMS interfaces as production, and you can research fast *and* deploy with
> confidence.

The interview-ready summary:

> "I'd separate the backtester from the exchange simulator. The backtester is optimized for research:
> it replays historical market events deterministically and evaluates strategy decisions, fills, costs,
> and P&L. The simulator is optimized for system validation: it behaves like an exchange, accepting
> orders and generating ACKs, fills, rejects, cancels, delays and disconnects. Both use deterministic
> event-time replay and avoid allocations on the hot path. Ideally the same strategy and OMS interfaces
> are used in live trading, backtesting, and simulation, with only the market/exchange implementation
> swapped."

---

**Next:** Module 07 — [the five systems at a glance](07-systems-at-a-glance.md): a capstone cheat-sheet
summarizing every component's requirements, challenges, flow, and HFT techniques (Module 08, the
options/greeks engine, is planned). Return to the [index](../00-index.md).
