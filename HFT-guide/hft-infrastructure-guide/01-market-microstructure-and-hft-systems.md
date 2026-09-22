# Module 01 — Market microstructure & the HFT system pipeline

Modules 1–19 taught you to *build fast things*. This module teaches you *what you're building and
why* — the domain knowledge an HFT infrastructure interview probes once it's satisfied you can code.
Most candidates can write a fast order book but go vague the moment someone asks "where does the book
*come from*?" or "backtester vs simulator — what's the difference?". This module closes that gap.

It's written India-first (NSE / BSE / prop-shop context) because that's the market you're
interviewing into. Read it as the *systems map* that your Module 03 order book plugs into: the order
book is one box in a much larger pipeline, and this module draws the whole pipeline.

> This is **Part 1 — the conceptual map**. Subsequent parts (planned) will expand each box into its
> own deep-dive with C++ design: the feed handler, the matching-engine simulator, the backtester's
> fill model, and the options pricing/greeks engine. For now: the complete picture, end to end.

---

## 1. The exchange side — what you connect to

In India the venues are **NSE** (dominant: equities, F&O, currency), **BSE** (fast-growing in index
derivatives since ~2023 via Sensex/Bankex weeklies), **MCX** (commodities), **NCDEX**. As an HFT you
do not "connect over the internet." You rent rack space *inside the exchange's colocation (colo) data
center* and cross-connect to the exchange gateways over fibre. This single physical fact defines the
whole game:

> Everyone is within tens of microseconds of the matching engine, so competition is about shaving the
> **last nanoseconds** — cable length, switch hops, and how little work your code does per packet.

Concepts you must be fluent in:

- **Matching engine** — a price–time priority FIFO order book *per instrument* (exactly your Module
  19 project, running on the exchange's side). Best price served first; ties broken by arrival time.
- **Order types** — limit, market, **IOC** (immediate-or-cancel: the HFT workhorse), stop-loss.
  HFTs almost never send *market* orders (uncontrolled slippage); they send priced IOC/limit.
- **Tick size, lot size, circuit limits, price bands** — India has per-scrip price bands and index
  circuit breakers (10/15/20%). Your risk logic must respect them.
- **Sessions** — pre-open call auction (09:00–09:08), continuous (09:15–15:30), close. Your system
  behaves differently in each; e.g. no continuous matching during the call auction.

---

## 2. The data that flows out of the exchange

Two directions. Get this crisp — it's the most common weak spot in interviews.

**Inbound (market data → you):**

- Broadcast over **UDP multicast** inside the colo. One publisher fans out to hundreds of members at
  no per-member cost — that's why multicast, not TCP.
- The premium product is **TBT (Tick-By-Tick)**: every order add / modify / cancel and every trade,
  in sequence. This lets you rebuild the *full* limit order book to any depth. It is the crown jewel.
- The cheaper feed is **snapshot / Level-2 depth** (top 5 or 20 levels, periodically refreshed) —
  lossy, insufficient for serious HFT.
- Feeds carry **sequence numbers**. UDP drops and reorders packets. You *must* detect gaps and
  recover — NSE provides a **TCP retransmission / recovery channel** plus periodic snapshots to
  resync.

**Outbound (orders → exchange) and back:**

- Orders go over a **TCP session** to the exchange trading gateway using the exchange's binary
  order-entry protocol (NSE's native **NNF**, and **FIX** for some flows). You send New / Modify /
  Cancel; you receive **execution reports** — acks, fills, rejects.
- Every order carries required regulatory tags: your SEBI-**approved algo ID**, user/dealer ID, etc.

The one-line mental picture:

```
  fast, lossy UDP multicast IN  ─▶  your logic  ─▶  reliable TCP order session OUT
                                        ▲                     │
                                        └──── execution reports (fills/rejects) ◀──┘
```

Two clocks must stay consistent: **the market's book** (built from the feed) and **your live orders**
(tracked from execution reports). Most correctness bugs live in the gap between those two views.

---

## 3. The full pipeline — every box and its job

```
                     ┌─────────────── EXCHANGE (colo) ────────────────┐
                     │  Matching engine • Multicast MD • Order gateway │
                     └───────▲──────────────────────────┬─────────────┘
                             │ orders (TCP)              │ market data (UDP multicast)
                             │                           ▼
   ┌─────────┐   ┌───────────┴────┐   ┌──────────────────────────┐
   │  Risk   │◄──│ Order Gateway  │   │   Feed Handler(s)         │
   │ (pre-   │   │  / OMS         │   │  parse • sequence •       │
   │ trade)  │──►│                │   │  gap recovery • normalize │
   └─────────┘   └───────▲────────┘   └──────────┬───────────────┘
                         │                        ▼
                         │              ┌──────────────────────┐
                         │              │  Order Book Builder  │  
                         │              │  rebuild LOB from TBT│
                         │              └──────────┬───────────┘
                         │                         ▼
                         │              ┌──────────────────────┐
                         └──────────────┤  Strategy / Signal    │
                                        │  Engine (alpha logic) │
                                        └──────────┬────────────┘
                                                   │
        ┌──────────────┬───────────────┬──────────┴──────┐
        ▼              ▼               ▼                  ▼
  Market Data     Backtester      Simulator          TCA / PnL /
  Recorder        (historical)    (sim exchange)     Position keeper
```

**Feed handler.** Reads raw multicast packets, decodes the exchange wire format, checks sequence
numbers, requests retransmission on gaps, and emits *normalized* internal messages (a clean struct,
not exchange-specific bytes). The hottest path — usually **kernel bypass** (Solarflare/Onload, DPDK)
and busy-polling, not interrupt-driven sockets. *Interview probe:* "a gap of 50 messages appears —
what do you do?" → buffer, recover via the retransmit channel or a snapshot resync, and mark the book
**stale** until resynced (don't trade on a book you can't trust).

**Order book builder.** Consumes TBT and maintains the full LOB in memory: price levels + per-level
FIFO order queues (price–time priority). *This is your Module 03 project.* The classic data-structure
question — array/vector indexed by price ticks for O(1) level access, plus intrusive linked lists of
orders per level, plus an object pool so the hot path never allocates. Connect it explicitly in
interviews: "the thing I built is exactly the book builder in this pipeline."

**Strategy / signal engine.** Reads book + trades, computes signals (order-book imbalance,
microprice, short-term momentum, stat-arb spreads, options mispricing), decides to send / modify /
cancel. Prop-HFT strategy families: **market making** (quote both sides, earn the spread, manage
inventory), **arbitrage** (cash–futures basis, index vs constituents, cross-exchange NSE/BSE),
**liquidity-taking / latency** (react faster than others). Kept single-threaded per instrument-group
by design — threads add locks and destroy latency determinism.

**Risk (pre-trade checks).** *Every* order passes checks *before* the wire: max order size, max
position, price sanity (fat-finger bands), order-rate limits, max loss / kill-switch. In India this
is **mandatory** (SEBI-mandated pre-trade controls; the broker's risk layer also sits between you and
the exchange). Key insight: risk is *on the critical path* — it must be nanosecond-fast yet
unbypassable.

**Order gateway / OMS.** Owns the TCP session(s), translates internal order intents to the exchange
protocol, tracks each order's **state machine** (New → Acked → PartiallyFilled → Filled / Cancelled /
Rejected), reconciles fills, maintains true position, throttles to rate limits, reconnects. The hard
part is *state correctness under failure*: after a disconnect you **query** the real order state, you
never assume it.

**Market data recorder.** Captures the raw feed to disk with hardware timestamps for replay. The
foundation of everything offline. The right design is "record raw bytes + timestamps, normalize on
replay" — never record only normalized data (you lose information and can't reproduce parser bugs).

**Backtester.** Replays recorded historical data through your *actual* strategy code to estimate PnL.
The whole intellectual difficulty is **fill modelling**: you didn't really trade, so you must *model*
whether your order would have filled — did you cross the spread? were you at the front of the queue?
did your own order move the market (**market impact**)? Naive "assume perfect fills" backtests lie.
Also nail **look-ahead bias** (using data you couldn't have known yet) and **survivorship bias**.

**Simulator (sim exchange / matching-engine simulator).** A fake exchange running the *same* matching
logic so your *whole stack* — feed handler, strategy, OMS — trades against simulated flow with
realistic latency. Two flavours: market-replay (match your orders against recorded order flow) and
synthetic/agent-based. The distinction to state crisply:

> **Backtester** answers *"would this alpha have made money?"*
> **Simulator** answers *"does my whole system behave correctly and fast, end to end?"*

**TCA / analytics.** Post-trade: slippage, fill rates, adverse selection, latency percentiles, PnL
attribution. Feeds back into strategy tuning. In India, costs (STT, fees, stamp duty) are large enough
that TCA often decides whether a strategy is viable at all.

---

## 4. Designing one in an interview

When asked *"design an HFT trading system"* or *"design a matching engine / order book,"* drive it in
this order:

1. **Clarify scope & NFRs first.** Segment (equities / F&O)? Strategy class (making vs taking)?
   Latency target (single-digit-µs tick-to-trade)? Throughput (NSE F&O bursts to millions of
   msg/sec on expiry)? Asking this shows maturity.
2. **State the critical path** and that you optimise **tail latency (p99.9), not average** — a slow
   outlier is a missed trade or a bad fill.
3. **Draw the §3 pipeline**, then justify the separations: feed handler vs strategy (isolation,
   testability); risk on the hot path (correctness + speed); OMS owning session state (recovery).
4. **Go deep where pushed** — usually the book data structure, the feed handler's gap recovery, or
   how you avoid allocation/GC on the hot path.
5. **Fast path / slow path split**: hot path = zero allocation, no locks, no syscalls (kernel
   bypass), no inline disk logging (write to a lock-free ring buffer; a separate thread drains it).
   Slow path = recovery, logging, config.
6. **Mechanical-sympathy vocabulary** (all from Modules 15–17): cache-line alignment & false
   sharing, NUMA pinning, isolated cores & busy-polling, lock-free SPSC ring buffers, kernel bypass
   (DPDK / Onload), branch prediction, cache warming, PTP-synced hardware timestamping, FPGA for the
   fastest feed decode. You needn't have built all of it — know *why each exists*.
7. **Design for testing** unprompted: the recorder + simulator + backtester triad is how you develop
   safely.

---

## 5. India-specific facts that signal you actually know the market

These separate serious candidates from people who read a US-centric blog:

- **Colo & SEBI fair-access rules** — SEBI regulates colo and TBT access (the 2015 NSE colo scandal
  is why). Equal, fair access is mandated.
- **Algo approval regime** — each strategy needs an **exchange-approved algo ID**; brokers certify.
  SEBI's algo-trading framework (incl. retail algo rules) is tightening — say you're aware.
- **Order-to-Trade Ratio (OTR) penalties** — NSE penalises excessive order messages per trade (to
  curb quote-stuffing). This **directly constrains market-making design**: you can't spam quotes for
  free. A strong detail to drop.
- **Product mix** — Indian HFT money is heavily in **index options** (NIFTY, BANKNIFTY weeklies; now
  Sensex on BSE) and **cash–futures / calendar arbitrage**, not US-style equity making. Options making
  needs a **pricing / greeks engine** (vol surface, real-time greeks) bolted onto the strategy engine.
- **Expiry-day microstructure** — weekly expiries create huge message bursts and volatility; your
  throughput and risk systems are sized for these peaks.
- **Costs** — STT, exchange fees, stamp duty are large; *this is why TCA matters so much in India*.
- **Clearing & margin** — trades clear through NSE Clearing (NCL); SPAN + exposure margins (now
  upfront intraday) constrain holdings — your position/risk system tracks **margin**, not just
  notional.

---

## 6. The two traps to rehearse

1. **"Where does the book come from?"** — Many can code a fast order book but can't explain how it
   gets *populated* from a lossy UDP feed with gaps. Learn the feed-handler / sequence-gap / recovery
   story cold (§2, §3).
2. **"Backtesting = replaying prices?"** — No. The entire difficulty is **fill modelling** and
   avoiding look-ahead bias (§3, Backtester). Miss this and you look naive.

Nail these two, draw the §3 pipeline from memory, and drop 3–4 §5 India facts, and you read as
someone who understands *the business* — not just the data structures.

---

## The mental through-line

> Data leaves the matching engine as a fast lossy multicast feed. A feed handler turns bytes into a
> trustworthy stream; the book builder (Module 03) turns that stream into a live order book; the
> strategy turns the book into intent; risk gates it; the OMS turns intent into orders and tracks
> their fate. Everything offline — recorder, backtester, simulator, TCA — exists so you can develop
> and validate that hot path *without lighting money on fire.* The C++ you learned is how each box
> stays fast; this module is why each box exists.

---

## What's next (the rest of Part B)

The modules follow the data's own journey through the pipeline — feed in, book built, orders out,
then the offline tooling that lets you develop safely:

- **02 — The feed handler in C++** *(written)*: multicast decode, sequence-gap state machine, kernel
  bypass, the normalize step, marking the book stale.
- **03 — Order-book architecture** *(written)*: the limit order book & matching engine — your
  flagship portfolio project.
- **04 — The strategy engine** *(written)*: the decision brain — event dispatch, incremental signals,
  the market-making quoter, backtest/live parity, worked LLD & HLD design problems.
- **05 — The OMS / order gateway** *(written)*: the exchange session, the order state machine, fills
  reconciliation, position keeping, gateway routing/throttling, telemetry, recovery after disconnect.
- **06 — The backtester & the exchange simulator** *(written)*: deterministic event-time replay,
  queue-position fill models, the simulated exchange behind the gateway protocol (reusing the Module 03
  engine), latency modelling, and fault injection to validate the OMS.
- **07 — The five systems at a glance** *(written)*: a capstone cheat-sheet — functional/non-functional
  requirements, key challenges, implementation flow, and the concrete HFT techniques for each of the
  five components above.
- **08 — Options pricing / greeks engine** *(planned)*: vol surface, real-time greeks, the extra box
  for index-option market making.

*(This sequence may still shift as modules are written.)*
