# Module 05 — The OMS / order gateway (deep dive)

The feed handler (Module 02) was **ingress**: it turned fast, lossy, exchange-specific bytes into a
trustworthy stream. The OMS is its **mirror image — egress**: it turns your strategy's trustworthy
*order intents* into orders live on the exchange, and tracks the fate of every one of them until it
fills, cancels, or dies. If the feed handler is where you're grilled on *networking + parsing +
correctness under loss*, the OMS is where you're grilled on *state machines + protocol + correctness
under failure — with real money on the line*.

That last clause changes everything. A feed handler that mis-processes a message costs you a trading
opportunity. An OMS that mis-processes a message can **send a duplicate order, lose track of a live
position, or fail an audit** — the errors are asymmetric and expensive, and a regulator (SEBI in
India) is watching. So the OMS carries a second obligation the feed handler doesn't: **never lose an
order, ever, even across a crash or disconnect.** Low latency *and* provable correctness at once.

This module goes **Part I — core concepts** (feed-handler depth: what it is, the order lifecycle,
the exchange session protocol, the anatomy), then **Part II — design problems** worked from a naive
map-based version to the HFT form, each one leaning hard on the C++/OS/CA modules so the theory
actually gets used, not just name-dropped.

> Prerequisites we'll actively use: object lifetime & the heap (Modules 01, 03), RAII & Rule of 5
> (07), move semantics (09), templates/concepts (10), inheritance vs CRTP (11, 17), the STL &
> `unordered_map` cost model (12), smart pointers/ownership (13), `noexcept` & error handling without
> exceptions (14), cache/alignment/false-sharing (15), `std::atomic` & lock-free rings (16), zero-cost
> abstraction/CRTP (17), `<bit>`/`std::span`/`std::byteswap` (18). And from this folder: the
> lock-free-ring hand-off and TCP-vs-UDP reasoning of Module 02.

This module realizes, end to end, what a real Indian-HFT engineer's work looks like:

> *"Architected an OMS from the ground up (core trading workflows + infrastructure); built market-data
> adapters and order execution engines for firing orders to the exchange; designed an Order Gateway
> Manager for intelligent routing to the right execution gateways; engineered a real-time pipeline for
> live trade monitoring and centralized storage."*

Each of those four clauses is a section below.

---

# Part I — Core concepts

## 1. What an OMS *is* (one sentence)

> An OMS turns a strategy's **abstract order intents** ("buy 100 @ 20050") into **concrete, sequenced,
> exchange-specific messages on a reliable session**, and maintains — durably and in real time — the
> **authoritative state of every order and position** as the exchange reports back.

Read it against the feed handler's one-sentence job and the symmetry is exact:

| | Feed handler (Module 02) | OMS (this module) |
|---|---|---|
| Direction | Ingress (exchange → you) | Egress (you → exchange) + the acks back |
| Transport | UDP multicast, lossy, fan-out | **TCP**, reliable, ordered, per-session |
| Input | Raw venue bytes | Internal order intents |
| Output | Normalized market-data stream | Orders on the book + authoritative order/position state |
| Hard property | *Correct-but-late beats fast-but-wrong* | *Never lose or duplicate an order; account for every one* |
| Failure cost | A missed trade | Real money + a regulatory audit finding |

Because market data fans out to hundreds of subscribers, it's UDP multicast. Order entry is a
**private, bidirectional conversation** between your firm and the exchange — so it's **TCP**: one
reliable, in-order, sequenced session per login. (Interview line: *"market data is UDP multicast for
lossy fan-out; order entry is TCP because it's a reliable point-to-point session where losing a
message is unacceptable."*)

## 2. OMS vs EMS vs gateway vs router — the terminology (and the prop-firm blur)

The industry splits the egress side into named boxes; small prop HFT firms often fuse them into one
process, but you must know the distinctions:

- **OMS (Order Management System)** — owns the **state of every order** and the resulting position.
  The source of truth. "What orders do I have working, at what price, how much is filled?"
- **EMS / Order Execution Engine** — the **firing path**: serialize an order into the venue's wire
  format and push it down the session; decode the exec reports coming back. "How do bytes get to the
  exchange and back?"
- **Order Gateway (session)** — one authenticated **connection to one exchange segment** (e.g. NSE
  F&O, NSE cash, BSE derivatives). Owns that session's sequence numbers and socket.
- **Order Gateway Manager / Router** — sits above many gateways and does **intelligent routing**:
  which order goes to which gateway, per-session throttling, failover, load-balancing.

Map to the resume clauses: *"architected the OMS"* = §4's state machine + §7's guarantees; *"order
execution engines for firing orders"* = the EMS (Problem 2); *"Order Gateway Manager for intelligent
routing"* = the manager (Problem 3); *"real-time pipeline for monitoring + storage"* = the telemetry
path (Problem 4).

## 3. Where it sits, and the data flow

```
                         intents                     ┌──────── OMS process ─────────┐
 strategy (Module 04) ───────────▶ pre-trade RISK ──▶│  order store + state machine │
   (one per core)      new/mod/cxl   (fat-finger,    │            (Problem 1)       │
        ▲                            pos/price limits,│              │              │
        │ fills/position             kill switch)     │              ▼              │
        │                                             │  Gateway Manager (Prob 3)   │
        │                                             │   routes to a gateway…      │
        │                                             │     │        │        │     │
        │                                             │  [GW NSE-FO][GW NSE-CM][GW BSE]  ── TCP ──▶ EXCHANGE
        │                                             │   exec engine / serialize (Prob 2)   ◀── acks/fills ──
        │                                             └───────┬──────────────────────┘
        │                                                     │ every event (out of band, non-blocking)
        └──────── exec reports routed back ◀──────────  telemetry & storage pipeline (Problem 4)
                                                        └─▶ QuestDB / monitoring / SEBI audit trail
```

Two flows matter: the **hot outbound path** (intent → risk → store → gateway → wire) whose latency is
part of your tick-to-trade, and the **inbound path** (ack/fill → update store → notify strategy)
whose correctness is sacrosanct. Telemetry hangs off the side and must **never** be on either critical
path (Problem 4).

## 4. The order lifecycle — the state machine at the heart of the OMS

This is the single most important concept in the module, and the thing interviewers probe first.
An order is **asynchronous**: you *send* a request and only *later* learn its fate. So the OMS models
an order as a state machine with explicit **in-flight ("pending") states** for "I've asked but don't
yet know."

```
                 send()                 exchange Ack
   [Created] ───────────▶ [PendingNew] ─────────────▶ [Working] ─────────┐
      │                        │                         │  ▲            │ partial Fill
      │                        │ Reject                  │  │ ReplaceAck │ (leaves>0)
      │                        ▼                          │  │            ▼
      │                    [Rejected]◀── Reject ──────────┤  └──── [PendingReplace] ◀── replace()
      │                     (terminal)                    │              │ ReplaceReject → back to Working
      │                                                   │              │
      │                                    cancel()       │ CancelReject (too late) → stay Working
      │                                 ┌─────────────────┤
      │                                 ▼                 │  Fill (leaves==0)
      │                          [PendingCancel] ──CancelAck──▶ [Cancelled]      ▼
      │                                                   │      (terminal)   [Filled] (terminal)
      └───────────────────────────────────────────────────────────────────────────────
                                                                    Expire → [Expired] (terminal)
```

The states:

| State | Meaning | You're waiting for |
|---|---|---|
| **Created** | Built internally, not yet sent | — |
| **PendingNew** | Sent to the exchange, no response yet | Ack or Reject |
| **Working** | Acked, live on the book (a.k.a. New/Acknowledged) | Fill / your cancel / your replace |
| **PartiallyFilled** | Some qty filled, remainder still working | more Fills / cancel |
| **PendingCancel** | You asked to cancel; not confirmed | CancelAck or CancelReject |
| **PendingReplace** | You asked to amend (price/qty); not confirmed | ReplaceAck or ReplaceReject |
| **Filled** | `leavesQty == 0` | terminal |
| **Cancelled** | Cancel confirmed | terminal |
| **Rejected** | Exchange refused the new order | terminal |
| **Expired** | Time-in-force elapsed (e.g. IOC remainder, day order at close) | terminal |

The transitions come from **two sources**, and that's what makes it hard:

- **From the strategy (outbound):** `send`, `cancel`, `replace`.
- **From the exchange (inbound, asynchronous):** `Ack`, `Fill`/`PartialFill`, `CancelAck`,
  `ReplaceAck`, `Reject`, `CancelReject`, `ReplaceReject`, `Expire`.

The subtle, interview-critical part is the **races between the two**, because the exchange doesn't
know what you're about to do and you don't know what it's already done:

- **Cancel/fill race:** you send a cancel while the order is resting; before your cancel reaches the
  matching engine, a `Fill` for that order is already on its way back. You must accept the fill (it
  really happened) and then get a `CancelReject` ("too late, already traded"). Applying them in the
  wrong order corrupts your position.
- **Ack-after-cancel:** you fire a new order and immediately a cancel; the Ack and the CancelAck can
  arrive in either order relative to your sends.
- **Replace races:** an amend and a fill cross on the wire; the replace may reject because the order
  just filled.

The discipline: **every inbound exec report is applied idempotently to the order's current state, and
the exchange is the source of truth on quantities.** You never assume your request succeeded — you
wait for the report. (Time-in-force types you'll model: **DAY**, **IOC** — immediate-or-cancel,
**FOK** — fill-or-kill, **GTC**. India intraday desks live on IOC and DAY.) We implement this state
machine and its races in **Problem 1**.

## 5. The exchange session & order-entry protocol (concretely)

Just as you can't design a feed handler without knowing the wire, you can't design an OMS without
knowing the **order-entry session**.

### 5.1 It's a TCP session with sequence numbers

You **logon** (authenticate: comp/session id, password, sometimes per-message HMAC), then exchange
**heartbeats** to keep it alive and detect death, and every message carries a **session sequence
number** so both sides can detect loss and request a **resend / gap-fill** — conceptually the same
sequence discipline as the feed handler (Module 02 §5), but now *bidirectional and reliable* because
TCP already guarantees order; the app-level seqnum exists for **recovery across a reconnect** (§Problem
5), not for reordering.

### 5.2 FIX vs native binary

Two worlds, exactly like NSE-proprietary vs BSE-T7 on the market-data side:

- **FIX (Financial Information eXchange)** — an ASCII, tag=value protocol: `35=D|11=ORD123|54=1|38=100|44=20050|...`
  fields separated by the SOH (`0x01`) byte, terminated by a checksum (tag `10` = sum of all bytes
  mod 256). Human-readable, ubiquitous, forgiving — and **slow**: string formatting, variable-length
  parsing, a checksum over every byte. Used for slower flows and as a lingua franca.
- **Native binary** — fixed-layout packed structs, like the market-data messages of Module 02 §2.3,
  but outbound. Orders of magnitude faster to encode/decode. This is what a latency-sensitive desk
  uses. **India:** NSE's native order-entry interface over TCP (the binary protocol members' front-ends
  — CTCL/NNF — bind to); **BSE** uses **Deutsche Börse T7's ETI (Enhanced Trading Interface)** for order
  entry (the same T7 whose EOBI you parsed for market data). Global analogues: NASDAQ **OUCH**, native
  binaries everywhere latency matters.

We encode both in **Problem 2**, and the FIX-vs-native tradeoff is a guaranteed interview question.

### 5.3 Identity: ClOrdID vs exchange OrderID

- **ClOrdID (Client Order ID)** — *you* generate it, unique per session per day, monotonically
  increasing. It's how you recognize which of *your* orders an exec report refers to. Most native
  protocols **echo your ClOrdID back on every exec report**, which — as Problem 1 shows — lets you skip
  a whole hash map.
- **Exchange OrderID** — the *exchange* assigns it on Ack; it's the market-wide identity of the resting
  order. Some venues key certain messages (e.g. unsolicited fills) by exchange OrderID, so you may need
  a secondary lookup.
- **Cancel/replace chaining** — to amend, you send a new request that carries a **new ClOrdID** plus
  the **original ClOrdID** (`OrigClOrdID` in FIX). Each mutation gets a fresh ClOrdID; you chain them so
  you can follow the order's history. This is why ClOrdIDs are burned quickly and must be cheap to
  generate and look up.

### 5.4 Drop copy & cancel-on-disconnect

- **Drop copy** — a separate, read-only session that receives a *copy* of every exec report for your
  firm, used by an independent risk/reconciliation process (and often mandated). Your safety net if the
  primary session's state is ever in doubt.
- **Cancel-on-disconnect (COD)** — an exchange feature (T7 has it; NSE offers it) that **auto-cancels
  your resting orders if your session drops**. It turns "I have unknown live orders after a crash" into
  "the exchange already flattened me" — a huge simplifier for recovery (§Problem 5).

## 6. The anatomy — four sub-systems

Putting §2's boxes onto the flow, the OMS is four cooperating pieces, each a design problem below:

1. **Order store + state machine** (Problem 1) — the authoritative record; O(1) lookup; correct under
   races; single-writer-per-session so it needs no locks.
2. **Execution engine** (Problem 2) — serialize intents to the wire, decode exec reports back; the
   market-data adapter's outbound twin.
3. **Gateway manager** (Problem 3) — route across sessions, throttle to respect exchange limits,
   fail over.
4. **Telemetry & persistence pipeline** (Problem 4) — capture everything for monitoring, TCA, and the
   SEBI audit trail, entirely off the hot path.

## 7. Non-functionals — the governing ethos

Where the feed handler's creed was *"correct-but-late beats fast-but-wrong,"* the OMS adds three:

- **Never lose an order.** Every order and every exec report must be recoverable across a process
  crash and a session disconnect. This forces durable, append-only event logging (Problem 4) and
  reconciliation (Problem 5).
- **Idempotency & determinism.** Applying the same exec report twice (it happens on resend) must not
  double-count a fill. Same inputs → same state, always — for reproducibility and audit.
- **Single writer per session.** One thread owns a gateway's socket, its sequence number, and the
  orders on it. No shared mutable order state across threads → **no locks on the hot path** (the same
  SPSC discipline as the feed handler's ring). Concurrency comes from *sharding sessions across cores*,
  not locking one.

Plus the shared HFT non-functionals: bounded, allocation-free work on the order path; latency measured
at p50/p99/**p99.9** (the tail is what a market-maker's adverse fills live in); pre-trade risk on the
critical path (correctness *and* speed, §Problem 3 intro).

---

# Part II — Design problems

Each problem is posed the way an interview would, then reasoned **naive → critique → HFT**, with the
relevant module called out where its tool is used. The naive version is always *correct and
map-based, indifferent to latency*; the optimized version earns every nanosecond.

## Problem 1 (LLD) — the order store & state machine

### 1.1 Statement

> Design the data structure and logic that holds **every live order** and drives the §4 state machine.
> Requirements: **O(1)** lookup of an order by **ClOrdID** (to apply an exec report) and, where the
> venue needs it, by **exchange OrderID**; correct transitions for all events including the
> **cancel/fill and ack/cancel races**; and it must run with **no allocation and no locks** on the hot
> path. Support partial fills, cancel, and replace.

### 1.2 How to think about it

Clarifying questions that score: *Does the venue echo my ClOrdID on every exec report?* (If yes, the
exchange-OrderID map is almost unnecessary.) *Are ClOrdIDs monotonic per session?* (Yes — so they can
be an array index, not a hash key.) *How many orders can be live at once?* (Bounded — so preallocate.)
*Who calls this — one thread or many?* (One per session — so no locks.)

Then state the shape: "It's a fixed pool of order records, indexed directly by ClOrdID; the state
machine is a small explicit switch on (state, event); the races are handled by treating the exchange
as truth on quantities and applying every report idempotently."

### 1.3 Naive version (correct, map-based, latency-agnostic)

```cpp
enum class St { PendingNew, Working, PendingCancel, PendingReplace, Filled, Cancelled, Rejected };

struct Order {
    uint64_t clOrdId, exchOrderId = 0;
    uint32_t instrument, origQty, leavesQty, cumQty = 0;
    int64_t  price, avgPx = 0;
    uint8_t  side; St state;
};

class NaiveStore {
    std::unordered_map<uint64_t, Order*> byClId_;     // heap-allocated orders
    std::unordered_map<uint64_t, Order*> byExchId_;
    std::mutex mtx_;
public:
    Order* create(...) {
        std::lock_guard g(mtx_);
        auto* o = new Order{...};                      // ← heap alloc on the hot path
        byClId_[o->clOrdId] = o;
        return o;
    }
    void onFill(uint64_t clId, uint32_t qty, int64_t px) {
        std::lock_guard g(mtx_);
        auto it = byClId_.find(clId);                  // ← hash + pointer chase (cache miss)
        if (it == byClId_.end()) return;
        Order* o = it->second;
        o->cumQty += qty; o->leavesQty -= qty;
        if (o->leavesQty == 0) { o->state = St::Filled; /* who deletes o? */ }
    }
    // ... onAck, onCancelAck, onReject each lock, find, mutate
};
```

**Critique — say all of this out loud:**

- **Heap allocation per order on the hot path** (`new Order`) — the exact cost Modules 01/03 warn
  about; unpredictable latency, fragmentation, and a `delete` lifetime question you can see leaking in
  the comment.
- **Two hash maps** — each lookup is a hash + a pointer chase into a heap node → a **cache miss**
  (Module 15). At market-open message rates that dominates.
- **A mutex** on a single-writer path — pure waste, plus contention and priority-inversion risk
  (Module 16). You don't need it: one thread owns the session.
- **Lifetime is undefined** — who `delete`s a filled order, and what happens when a *late* duplicate
  fill arrives for an order you already freed? **Use-after-free** (Modules 07/13).
- No explicit handling of the races (§4) — just field mutation.

### 1.4 HFT version

Fix each point with a concept you've already built:

**(a) An object pool, not `new`/`delete`** — the identical trick as the order book's `OrderPool`
(Module 03 §7.1, backed by Modules 03/13). All order records live in one preallocated array; "create"
hands out a free slot, "retire" returns it. Zero hot-path allocation, contiguous memory.

**(b) A generational handle instead of a raw pointer** — the killer detail for the late-fill bug. Pack
the ClOrdID (or a companion handle) as `{ slotIndex : generation }`. Lookup is a direct array index
(no hash, no pointer chase — Modules 12/15); the generation bits detect a **stale** report for a slot
that's been recycled, turning a use-after-free into a safe, counted drop:

```cpp
struct Handle { uint32_t slot; uint32_t gen; };        // fits in 64 bits; also usable as the ClOrdID

class OrderStore {
    static constexpr uint32_t N = 1 << 16;             // max live orders — sized to peak
    std::array<Order,   N> pool_;                      // contiguous, cache-friendly (Module 15)
    std::array<uint32_t,N> gen_{};                     // generation per slot
    std::vector<uint32_t>  freeList_;                  // slot indices available
public:
    OrderStore() { freeList_.reserve(N);
                   for (uint32_t i = N; i-- > 0; ) freeList_.push_back(i); }

    Handle create(const NewOrder& n) noexcept {        // noexcept: no throwing on the hot path (M14)
        uint32_t s = freeList_.back(); freeList_.pop_back();
        Order& o = pool_[s];
        o = {}; o.clOrdId = pack(s, gen_[s]);          // ClOrdID encodes the handle → echoed back by venue
        o.instrument = n.instrument; o.price = n.price;
        o.origQty = o.leavesQty = n.qty; o.side = n.side;
        o.state = St::PendingNew;
        return {s, gen_[s]};
    }
    Order* find(Handle h) noexcept {                   // O(1), one array read, no hash (M12/M15)
        if (h.slot >= N || gen_[h.slot] != h.gen) return nullptr;   // stale/late report → safe drop
        return &pool_[h.slot];
    }
    void retire(uint32_t s) noexcept { ++gen_[s]; freeList_.push_back(s); }  // bump gen → old handles go stale
};
```

Because the venue **echoes your ClOrdID** on every exec report (§5.3) and the ClOrdID *is* the handle,
you look orders up by direct index — **the exchange-OrderID hash map disappears entirely** for the
common path. You keep a *small* `exchOrderId → Handle` map only if the venue sends unsolicited messages
keyed solely by exchange id.

**(c) The state machine as an explicit, race-aware transition** — no locks (single writer), the
exchange authoritative on quantities, idempotent:

```cpp
void OrderStore::onExecReport(const ExecReport& r) noexcept {
    Order* o = find(r.handle);
    if (!o) { stats_.staleReport++; return; }          // late/duplicate for a recycled slot — safe drop
    switch (r.type) {
      case Ack:
        if (o->state == St::PendingNew) { o->state = St::Working; o->exchOrderId = r.exchId; }
        break;                                          // Ack after we've moved on → ignore (idempotent)
      case Fill: {
        // Exchange is truth: clamp so a duplicate/resent fill can't double-count (idempotency, §7)
        uint32_t q = std::min(r.qty, o->leavesQty);
        o->cumQty += q; o->leavesQty -= q;
        o->avgPx = (o->avgPx*(o->cumQty-q) + r.px*q) / o->cumQty;
        if (o->leavesQty == 0) { o->state = St::Filled; publishFillThenRetire(o); }
        else                    o->state = St::Working;  // partial: still live, even if PendingCancel
        break; }                                          // ← the cancel/fill race: fill wins, cancel will reject
      case CancelReject:                                  // "too late" — the order already filled/traded
        if (o->state == St::PendingCancel)
            o->state = (o->leavesQty ? St::Working : St::Filled);
        break;
      case CancelAck:  o->state = St::Cancelled; publishThenRetire(o); break;
      case Reject:     o->state = St::Rejected;  publishThenRetire(o); break;
      // ReplaceAck / ReplaceReject analogous…
    }
}
```

The **cancel/fill race** falls out naturally: a `Fill` arriving while `PendingCancel` reduces
`leavesQty` (the fill really happened); the subsequent `CancelReject` just settles the state. No data
structure gymnastics — the discipline "exchange is truth, apply idempotently" does the work.

**(d) Retire only after publishing.** `retire()` bumps the generation, so any *later* stray report for
that slot is caught by `find()` returning `nullptr` — the use-after-free of the naive version is now
structurally impossible.

### 1.5 Tradeoffs

| Concern | Naive | HFT |
|---|---|---|
| Allocation | `new` per order | preallocated pool, zero hot-path alloc (M03/13) |
| Lookup | 2× hash + pointer chase (cache miss) | direct array index, one read (M12/15) |
| exchId map | always | only for unsolicited-by-exchId venues |
| Concurrency | mutex (contention) | single writer, lock-free (M16) |
| Late/dup report | use-after-free risk | generational handle → safe drop |
| Races | ad-hoc field writes | explicit idempotent transition, exchange-as-truth |

## Problem 2 (LLD) — the execution engine: encoding orders & decoding exec reports

### 2.1 Statement

> Design the path that turns an internal order into **bytes on the session socket** and turns inbound
> **exec-report bytes** back into events for Problem 1. Minimize latency and allocation. Support both a
> **FIX** venue and a **native binary** venue behind one interface. (This is the resume's *"market-data
> adapters and order execution engines for firing orders."*)

### 2.2 How to think about it

It's the feed handler's parser (Module 02 §4) run **backwards and outbound**, over **TCP**. Same
tools — packed structs, `memcpy`, `byteswap`, `std::span` — plus TCP-specific knobs (Nagle, batching)
and a multi-venue abstraction question (how to support FIX *and* native without virtual-call cost on
the hot path).

### 2.3 Naive version

```cpp
void sendFix(const Order& o, int fd) {
    std::string m;                                     // ← heap allocation, formatting cost
    m += "35=D\x01";
    m += "11=" + std::to_string(o.clOrdId) + "\x01";   // ← more allocs, integer→string
    m += "54=" + std::string(1, o.side)   + "\x01";
    m += "38=" + std::to_string(o.origQty)+ "\x01";
    m += "44=" + std::to_string(o.price)  + "\x01";
    int sum = 0; for (char c : m) sum += (unsigned char)c;   // checksum over every byte
    m += "10=" + fmt3(sum % 256) + "\x01";
    ::send(fd, m.data(), m.size(), 0);                 // ← one syscall per order
}
```

**Critique:** heap allocation and integer→string formatting per order (Modules 01/03); a byte-by-byte
checksum; **a syscall per order** (~µs each — the same lesson as the feed handler's `recvmmsg`
batching, Module 02 §3.2); and it hard-codes FIX.

### 2.4 HFT version

**(a) Prefer the native binary encoder — packed structs + byteswap** (Modules 15/18), the mirror of
Module 02 §2.3. Serialize by `memcpy` into a **fixed, thread-local buffer**, no allocation, and the
fixed-size copy compiles to a few stores:

```cpp
#pragma pack(push, 1)
struct WireNewOrder {
    uint16_t length; uint16_t type;                    // = MsgType::NewOrder
    uint64_t clOrdId; uint32_t instrument;
    uint8_t  side; int64_t price; uint32_t qty; uint8_t tif;
    uint64_t seqNum;                                   // session sequence (single-writer owns it)
};
#pragma pack(pop)
static_assert(sizeof(WireNewOrder) == 38);             // catch accidental padding (M15)

std::size_t encode(const Order& o, uint64_t seq, std::span<std::byte> buf) noexcept {  // M18 span, M14 noexcept
    WireNewOrder w{ .length=sizeof(w), .type=uint16_t(MsgType::NewOrder),
                    .clOrdId=o.clOrdId, .instrument=o.instrument, .side=o.side,
                    .price=o.price, .qty=o.origQty, .tif=o.tif, .seqNum=seq };
    to_wire(w);                                         // byteswap each field once if venue is big-endian (M18)
    std::memcpy(buf.data(), &w, sizeof(w));             // fixed-size memcpy → compiles to stores
    return sizeof(w);
}
```

**(b) Batch writes; disable Nagle.** Set **`TCP_NODELAY`** so the kernel doesn't sit on your small
order waiting to coalesce (Nagle's algorithm is death for latency). When you *do* have several messages
ready (a quote update = cancel + new), gather them and issue **one `writev`** (scatter-gather) instead
of N `send`s — the outbound analog of `recvmmsg`. In production the socket itself is a **kernel-bypass
TCP** stack (Onload), same as Module 02 §3.3.

**(c) Multi-venue without virtual-call cost — CRTP** (Module 17). A FIX venue and a native venue have
different encoders; you don't want a vtable indirection per order on the hot path. Template the engine
on a concrete encoder (a `Protocol` concept, Module 10), so the call is inlined and zero-cost:

```cpp
template <class Enc>                                   // Enc = NativeEncoder or FixEncoder
class ExecEngine {
    Enc enc_; Session& sess_;
public:
    void fire(const Order& o) noexcept {
        std::byte buf[64];
        auto n = enc_.encode(o, sess_.nextSeq(), buf); // inlined, no virtual dispatch (M17)
        sess_.write({buf, n});                          // TCP_NODELAY socket / Onload
    }
};
```

Use runtime polymorphism (a `virtual Encoder`) only at the cold configuration boundary where you pick
the venue once at startup — never per order.

**(d) Decoding exec reports** is literally Module 02 §4 inbound: `memcpy` into an aligned struct,
byteswap, `switch` on type into the Problem-1 `onExecReport`. Reuse the exact discipline.

### 2.5 Tradeoffs

| Decision | Tradeoff |
|---|---|
| **FIX vs native** | FIX: universal, debuggable, forgiving, but string-heavy and slow. Native: fast fixed structs, but venue-specific and needs the licensed spec. Latency desks: native; connectivity/back-office: FIX. |
| **Per-message vs batched `writev`** | Batching cuts syscalls but can add a hair of latency to the first message; batch only what's already ready (e.g. a cancel+new pair). |
| **`TCP_NODELAY` on** | Removes Nagle coalescing delay (essential) at the cost of more, smaller packets — fine in colo. |
| **CRTP vs virtual encoder** | CRTP inlines to zero cost but fixes the venue at compile time (one binary per protocol, or a template instantiation each); virtual is flexible but adds an indirection per order. Pick CRTP on the hot path. |
| **Kernel TCP vs Onload** | Sockets for dev/tests; bypass in prod for the µs and the jitter reduction. |

## Problem 3 (HLD) — the Order Gateway Manager: routing, throttling, failover

### 3.1 Statement

> Design the manager that sits above many gateway sessions. **K strategies** (each pinned to its own
> core) emit orders; the manager must **route** each to the correct gateway (by instrument/segment),
> enforce each exchange session's **rate limits** (messages/sec and order-to-trade ratio), **fail over**
> to a backup session if one dies, all **without head-of-line blocking** and **preserving per-instrument
> order ordering**. (The resume's *"Order Gateway Manager for intelligent routing."*)

### 3.2 How to think about it

The core tension: **many producer cores, one writer per session.** Strategies live on different cores;
each gateway session must be written by exactly one thread (§7, single-writer). So the design is a
**cross-core hand-off** problem — precisely what lock-free rings are for (Module 16) — plus a routing
table, a rate limiter, and a failover policy. And you must not let a slow/throttled session stall
orders bound for a *different*, healthy session (no head-of-line blocking).

### 3.3 Naive version

```cpp
class NaiveManager {
    std::map<uint32_t, Gateway*> route_;               // instrument → gateway (tree lookup)
    std::mutex mtx_;
public:
    void submit(const Order& o) {
        std::lock_guard g(mtx_);                        // ← every strategy core contends on one lock
        Gateway* gw = route_[o.instrument];             // ← O(log n) tree walk, cache-unfriendly
        while (!gw->rateOk()) std::this_thread::sleep_for(1ms);  // ← sleeping = latency + blocks others
        gw->send(o);
    }
};
```

**Critique:** one global mutex all strategy cores fight over (contention + false sharing, Modules
15/16); a `std::map` (node-per-entry, pointer-chasing tree — Module 12); rate-limiting by **sleeping**
(catastrophic latency, and it blocks orders for *other* gateways — head-of-line blocking); no failover.

### 3.4 HFT version

**(a) One single-writer thread per gateway, pinned to a core** (Module 15). Strategies never touch a
socket; they publish to a per-gateway **lock-free MPSC ring** (multiple producer cores, one consumer =
the gateway writer thread — Module 16, with correct `acquire`/`release` ordering). No mutex anywhere:

```
 strat core 0 ┐
 strat core 1 ┼─push→ [MPSC ring → GW NSE-FO writer] ─(TCP_NODELAY / Onload)→ exchange
 strat core 2 ┘        (each ring & each writer's hot state is cache-line padded — M15 false sharing)
 strat core 0 ─push→ [MPSC ring → GW NSE-CM writer] ─→ exchange   (independent — no HOL blocking)
```

Because each gateway has its **own** ring and writer, a throttled NSE-FO session **cannot** stall
orders headed for NSE-CM — head-of-line blocking is gone by construction. Pad each ring's head/tail and
each writer's hot fields to a cache line so producers on different cores don't ping-pong the same line
(Module 15 false sharing).

**(b) Routing table = direct-indexed array, not a map.** Instruments are dense numeric tokens (Module
02 §2.3), so `gatewayId = route_[instrument]` is one array read (Module 12/15), no tree, no hash:

```cpp
std::array<uint16_t, kMaxInstruments> route_;          // instrument → gatewayId, one L1 hit
```

**(c) Rate limiting = a token bucket, integer, no sleeping.** Each session refills tokens at its
permitted rate; if empty, the order **waits in that gateway's ring** (back-pressure local to that
gateway) rather than sleeping a shared thread:

```cpp
struct TokenBucket {                                    // per session; owned by its single writer → lock-free
    int64_t tokens, capacity, refillPerNs; uint64_t lastNs;
    bool allow(uint64_t nowNs) noexcept {               // event-time, not wall-clock → deterministic (M04 §1.4)
        tokens = std::min(capacity, tokens + int64_t(nowNs - lastNs) * refillPerNs);
        lastNs = nowNs;
        if (tokens >= SCALE) { tokens -= SCALE; return true; }
        return false;                                   // no token → leave it queued, serve next tick
    }
};
```

Track the **order-to-trade ratio** too (India: exchanges penalize a high OTR): the writer counts
messages vs fills and, as it approaches the cap, tightens its own quoting via back-pressure to the
strategy rather than getting fined — the same OTR lever the quoter tuned in Module 04 §2.5, now
enforced centrally.

**(d) Failover.** Heartbeat each session (§5.1); on death, the writer marks its gateway `Down`, and the
routing entry flips to the **backup session** for that segment. In-flight orders with no ack move to an
**unknown** state handled by reconciliation (Problem 5). If **cancel-on-disconnect** is enabled, the
exchange flattens the dead session's resting orders for you.

**(e) Ordering guarantee.** Per-instrument order sequence is preserved because all orders for an
instrument route to the **same gateway** whose **single writer** drains its ring FIFO — no reordering
possible. (If you ever split one instrument across sessions for throughput, you lose this; usually you
don't.)

### 3.5 Tradeoffs

| Decision | Tradeoff |
|---|---|
| **Per-gateway ring + writer vs one shared queue** | Independent rings kill HOL blocking and locks, at the cost of more threads/cores. Worth it. |
| **Token bucket vs leaky bucket** | Token bucket allows short bursts up to `capacity` (good for quote storms); leaky bucket is strictly smooth. Bucket suits OTR-bounded bursts. |
| **Back-pressure (queue) vs drop vs sleep** | Queue preserves orders and ordering but adds latency under load; dropping loses orders (unacceptable for real orders); sleeping is worst (latency + HOL). Queue + back-pressure to the strategy. |
| **Failover aggressiveness** | Fast failover risks flapping/duplicate orders; slow failover risks dead time. Gate on missed heartbeats + a hysteresis. |
| **One instrument, one gateway** | Preserves ordering; caps a single instrument's throughput to one session. Fine for equities/options; revisit only for extreme cases. |

### 3.6 The complete lifecycle of one order (memorize this)

Everything in Problems 1–3 comes together in the journey of a *single* order from a strategy's decision
to its terminal state. This is the sequence to have burned into memory:

```text
                    MARKET DATA
                        ↓
                    Strategy                         (Module 04 — decides)
                        ↓
                 "BUY AAPL 100"
                        ↓
                  Command Queue                      (lock-free MPSC, §3.4a / M16)
                        ↓
                Gateway Manager
                        ↓
                route[AAPL]                           (direct-index routing, §3.4b)
                        ↓
                Gateway Queue
                        ↓
                Gateway Thread                        (single writer, owns everything below)
                        ↓
                  Pre-trade Risk                      (§3.intro / §7 — unbypassable)
                        ↓
                   OrderStore                         (Problem 1 — pool + generational handle)
                        ↓
               state = PendingNew
                        ↓
                 Token available?                     (token bucket, §3.4c — no sleep)
                   /          \
                 yes           no
                  ↓             ↓
             send exchange    queue  (back-pressure, local to this gateway)
                  ↓
               Exchange                               (native/FIX encode, Problem 2)
                  ↓
                 ACK
                  ↓
            state = Working
                  ↓
                FILL 40
                  ↓
          cum=40, leaves=60                            (exchange-as-truth, idempotent, Problem 1)
                  ↓
              Strategy
                  ↓
           "Cancel order"
                  ↓
             Command Queue
                  ↓
            Gateway Thread
                  ↓
        state = PendingCancel
                  ↓
            send CANCEL
                  ↓
               Exchange
               /       \
         Cancel wins   Fill wins                       (the cancel/fill race, §4 / Problem 1)
             ↓             ↓
         Cancelled       Filled
```

Notice how every hop maps to a component you designed: the two lock-free queue boundaries (command
queue in, gateway queue), the single-writer gateway thread that serializes risk → store → send, and
the terminal fork that *is* the cancel/fill race resolving itself.

### 3.7 The threading picture (the most important diagram)

Where do threads actually live, and why is almost nothing locked?

```text
       CORE 0             CORE 1             CORE 2
         │                  │                  │
        S1                 S2                 S3        (strategies — one per core, Module 04)
         │                  │                  │
         └────────────┬─────┴───────┬──────────┘
                      │             │
                      ▼             ▼
                  MPSC Queue 0   MPSC Queue 1           (lock-free hand-off — M16)
                      │             │
                      ▼             ▼
                 GW Thread 0    GW Thread 1             (one single writer per session)
                      │             │
                 owns session 0  owns session 1
                      │             │
                      ▼             ▼
                  Exchange A      Exchange B
```

Inside one gateway thread, that single writer owns *all* the mutable state:

```text
                 GW Thread 0
                     │
       ┌─────────────┼──────────────┐
       ↓             ↓              ↓
    socket       OrderStore     TokenBucket
       │             │
       │             ↓
       │          Order 1
       │          Order 2
       │          Order 3
       ↓
    Exchange
```

Because **only GW Thread 0 ever touches those things**:

```text
OrderStore      → no mutex
TokenBucket     → no mutex
Sequence number → no mutex
Socket          → no mutex
```

**The synchronization is primarily at the queue boundary** (the MPSC rings) — nowhere else. This is
the whole payoff of the single-writer-per-session principle (§7): concurrency comes from *sharding
sessions across cores*, and the only place two threads meet is the lock-free queue between a strategy
core and a gateway thread. (Compare the strategy engine's identical reasoning in Module 04 §4.2, points
8/14/23: single-threaded ≠ lock-free; lock-free only at the boundaries.)

### 3.8 The one sentence that ties Problem 3 together

If an interviewer asks *"how does the system work?"*, this is the answer to deliver:

> "Strategies run independently on different cores and publish order commands to lock-free queues. The
> Gateway Manager routes each command to the gateway responsible for that instrument/segment. Each
> gateway has a **single writer thread** that owns its socket, sequence numbers, rate limiter, and order
> state; it processes commands sequentially, performs risk checks, updates the OrderStore, and sends to
> the exchange. Independent queues and gateway threads prevent one throttled or failed session from
> blocking another, while failover redirects new orders and reconciliation resolves orders whose outcome
> is unknown."

## Problem 4 (HLD) — the real-time telemetry & persistence pipeline

### 4.1 Statement

> **Every** order event (send, ack, fill, cancel, reject) must be durably stored for the **SEBI audit
> trail**, streamed to **live trade monitoring**, and landed in **centralized storage** for TCA — with
> **zero impact on the order path's latency**. (The resume's *"real-time pipeline for live trade
> monitoring and centralized storage."*)

### 4.2 How to think about it

The hot path (Problems 1–3) does integer math and one socket write. Logging, DB inserts, and monitoring
involve **syscalls, disk, and locks** — the very things that blow up tail latency and jitter (Module
15). So the rule is absolute: **the hot path may not do I/O.** It may only drop a fixed-size event into
a lock-free queue; a separate thread does the slow work. This is the *exact* SPSC hand-off pattern the
feed handler used to decouple parsing from book-building (Module 02 §8) — reused on the egress side.

### 4.3 Naive version

```cpp
void onEvent(const OrderEvent& e) {
    std::lock_guard g(logMtx_);                         // ← lock on the hot path
    logFile_ << format(e) << '\n';                      // ← disk write / syscall on the hot path
    db_.insert(e);                                      // ← network + DB round-trip inline (!!)
    monitor_.publish(e);
}
```

**Critique:** a lock, a disk write, and a **synchronous DB insert** on the critical path — each is
microseconds-to-milliseconds of latency and, worse, *unpredictable* (a disk hiccup or a DB stall
becomes an order-path stall). This is the single most common way a well-built engine gets ruined.

### 4.4 HFT version

**(a) Hot path → lock-free SPSC ring only.** Each producer (a gateway writer) owns one ring to one
drainer; it pushes a small POD event and returns immediately (Module 16). The ring is a fixed array
allocated at startup — no allocation, no lock, cache-line-padded head/tail (Module 15):

```cpp
struct alignas(64) OrderEvent {                         // one per cache line — no false sharing (M15)
    uint64_t ts_ns, clOrdId; uint32_t instrument;
    int64_t  price; uint32_t qty; uint8_t type, side; uint64_t seq;
};
// gateway writer (producer): fire-and-forget, ~nanoseconds
if (!telemetry_.try_push(evt)) stats_.telemetryDropped++;  // policy decision below
```

**(b) A drainer thread on a non-critical core** batches events out of the ring and does all the slow
work: append to a durable **event log** (append-only, the audit source of truth), batch-insert into the
**centralized time-series store**, and fan out to the **monitoring** feed:

```cpp
void drainLoop() {                                      // pinned to a spare core, not a trading core
    std::array<OrderEvent, 512> batch;
    while (running_) {
        std::size_t n = telemetry_.try_pop_bulk(batch); // amortize per-event cost over a batch
        if (!n) { cpu_relax(); continue; }
        auditLog_.append(batch, n);                      // durable, append-only (WAL-style)
        questdb_.insertBatch(batch, n);                  // centralized storage — see note
        monitor_.publish(batch, n);                      // live dashboards / risk
    }
}
```

**(c) Centralized storage = a time-series DB, written in batches over the right protocol.** In the
QuantStream stack this is **QuestDB**, and there's a concrete gotcha to respect: ingest via the
**PG-wire JDBC** path in batches, **not** the ILP client (the ILP client has a Java-17 crash — recorded
in project memory). Batching is what makes the DB keep up with a burst without back-pressuring the
drainer.

**(d) The ring-full policy — a genuine design decision, not an afterthought.** For *monitoring* you can
drop-oldest under extreme load (a dashboard tolerates a gap). For the **audit trail you cannot drop** —
so size the ring for the worst-case open burst, and if it still saturates, the drainer must be fast
enough (batching + a dedicated core) that it never does; as a backstop, a full audit ring is a
**critical alert**, and some designs give the audit path its own larger ring or a spill-to-local-file
fallback so durability never depends on the DB keeping up.

**(e) Durability & compliance.** The append-only event log is the legal record: immutable, timestamped
(hardware/PTP timestamps as in Module 02 §8), and replayable — which doubles as the input to the
**backtester** (Module 07) and reconciliation (Problem 5). This is why "just log to the DB" isn't
enough: the DB is for querying; the append-only log is for *truth and recovery*.

### 4.5 Tradeoffs

| Decision | Tradeoff |
|---|---|
| **Inline vs ring + drainer** | Inline is simple but puts I/O on the hot path (fatal). Ring+drainer adds a thread and a tiny hand-off cost but keeps the order path clean. Non-negotiable: decouple. |
| **Drop vs block vs overwrite when full** | Monitoring: drop-oldest is fine. Audit: never drop → size for peak + alert + spill fallback; blocking the hot path is unacceptable. Different rings, different policies. |
| **Batch size** | Bigger batches amortize DB/disk cost but add latency to when an event becomes visible/durable. Tune to the DB's sweet spot. |
| **Append-log + DB vs DB only** | The log gives immutable truth + replay + recovery; the DB gives queryability. You want both; they serve different masters. |
| **Storage retention** | Full tick+order retention is huge; tier it (hot in-memory/recent in QuestDB, older rolled off) — the intraday lookback model, not full backfill. |

## Problem 5 (design) — recovery & reconciliation after a disconnect

### 5.1 Statement

> Your order session's TCP connection drops mid-day while you have **live orders resting on the
> exchange** and **in-flight orders with no ack yet**. On reconnect, restore correct state **without
> losing an order, double-counting a fill, or sending a duplicate.**

### 5.2 How to think about it

The instant the session drops, **the exchange — not you — is the source of truth** about what's live.
Your job is to re-synchronize your Problem-1 store to the exchange's reality. Three buckets of orders:
(1) confirmed working before the drop, (2) **in-flight** (sent, no ack — you genuinely don't know if
they made it), (3) terminal (already done). Bucket (2) is the dangerous one.

### 5.3 The approach

1. **Session resend / gap-fill.** On reconnect you re-logon and negotiate sequence numbers; the
   exchange resends the exec reports you missed while disconnected (the reliable-session analog of the
   feed handler's retransmission, Module 02 §6). Apply them **idempotently** (Problem 1's clamp-to-truth
   fill handling means a resent fill can't double-count).
2. **Reconcile against ground truth.** Query the exchange's **open-orders / order-and-trade** snapshot,
   or consume the **drop-copy** session (§5.4), and diff it against your store: any order the exchange
   shows that you don't → adopt it; any order you think is working that the exchange doesn't show →
   mark terminal.
3. **Resolve the in-flight bucket.** For "sent but never acked": either you find it in the reconciled
   snapshot (it made it → adopt) or you don't (it never landed → mark rejected). **Cancel-on-disconnect**
   (§5.4), if enabled, collapses this: the exchange already cancelled everything on the dead session, so
   after reconnect you start clean and simply re-quote.
4. **Never blindly resend.** Re-firing an in-flight order without reconciling risks a **duplicate live
   order** — the expensive mistake. Reconcile first, act second.
5. **The append-only event log** (Problem 4) lets you rebuild your *intended* state after even a full
   process crash; the exchange snapshot corrects it to *actual* state. Intent + truth = recovered.

### 5.4 Tradeoffs

| Decision | Tradeoff |
|---|---|
| **Cancel-on-disconnect vs persist-and-reconcile** | COD makes recovery trivial (start flat) but you lose queue priority on every resting order at every blip — bad for a passive market-maker. Persist-and-reconcile keeps your book but is far more complex and must be flawless. Takers lean COD; makers lean reconcile. |
| **Query snapshot vs drop-copy** | A snapshot query is on-demand but heavier; drop-copy is a continuous independent truth stream but another session to run. Serious desks run drop-copy. |
| **Trust local log vs exchange** | The local log is your intent and is instantly available; the exchange is authoritative but slower to query. Rebuild from the log, then correct against the exchange — never the log alone. |

---

# Interview Q&A — OMS

**Q. "Feed is UDP; why is order entry TCP?"** Market data fans out to many subscribers and tolerates
app-level loss recovery, so multicast wins. Order entry is a private, bidirectional session where a
lost message is unacceptable and there's exactly one counterparty — TCP's reliability and ordering are
exactly right, and you add an app-level sequence number for recovery across reconnects.

**Q. "Walk me through an order's states."** Created → PendingNew (sent) → Working (acked) →
PartiallyFilled/Filled, with PendingCancel/PendingReplace as in-flight branches and Rejected/Cancelled/
Expired terminals. Transitions come from my sends *and* from asynchronous exec reports; the exchange is
truth on quantities and I apply every report idempotently.

**Q. "A cancel and a fill cross on the wire — what happens?"** The fill really occurred, so I reduce
`leavesQty` regardless of my PendingCancel; then the CancelReject ("too late") just settles the state.
Because I treat the exchange as authoritative and apply idempotently, the race resolves correctly with
no special-casing.

**Q. "How do you look up an order in O(1) without a hash map?"** ClOrdIDs are monotonic and the venue
echoes them on every exec report, so the ClOrdID *is* a direct array index into a preallocated order
pool. I add generation bits to the handle so a late/duplicate report for a recycled slot is detected
and safely dropped instead of corrupting a reused order.

**Q. "Where's the allocation and where are the locks?"** Neither is on the hot path. Orders come from a
preallocated pool; each gateway session is single-writer so it needs no lock; strategies hand orders to
gateways over lock-free MPSC rings; telemetry leaves the hot path via an SPSC ring to a drainer thread.

**Q. "How do you respect exchange rate limits without adding latency?"** A per-session token bucket
using event-time, not sleeping. When a session is out of tokens, orders wait in *that* gateway's ring
(local back-pressure) so a throttled session never blocks a healthy one — no head-of-line blocking.

**Q. "How do you log every order for the SEBI audit without slowing trading?"** The hot path only
pushes a fixed-size event into a lock-free ring; a drainer on a spare core batches it into an
append-only audit log plus the time-series store (QuestDB, via PG-wire in batches) and the monitoring
feed. The audit ring is sized for peak and alerts rather than dropping.

**Q. "Your session dies with live orders — how do you recover?"** The exchange is now the truth. Re-
logon and take the session resend, then reconcile against an open-orders snapshot or drop-copy, resolve
in-flight orders by presence in that snapshot, and never blindly resend (duplicate risk). With cancel-
on-disconnect, the exchange has already flattened me, so I start clean.

**Q. "FIX or native?"** Native binary on a latency desk — fixed packed structs, byteswap, memcpy, no
string formatting; FIX where universality/debuggability matters. I hide the choice behind a CRTP
encoder so there's no virtual-call cost per order.

**Q. "NSE vs BSE order entry?"** NSE uses its native binary order-entry protocol over TCP (the CTCL/NNF
interface); BSE runs Deutsche Börse T7, so order entry is ETI — the same T7 whose EOBI I parse for
market data. I normalize both behind one internal order model, exactly as I normalize market data.

---

# Key takeaways

- The OMS is the feed handler's **mirror image**: egress over a reliable TCP session, with a second,
  heavier obligation — **never lose or duplicate an order, and account for every one** (money +
  regulator). "Correct-but-late" becomes "correct-and-durable."
- The **order lifecycle state machine** is the core. Model in-flight (Pending) states explicitly, treat
  the **exchange as truth on quantities**, and apply every exec report **idempotently** — and the ugly
  cancel/fill and ack/cancel races resolve themselves.
- **Every optimization is a concept you already built:** object pool + generational handles (M03/13/15)
  for the store; packed structs + `byteswap` + `memcpy` + `std::span` (M15/18) and CRTP (M17) for the
  wire; lock-free MPSC/SPSC rings + cache-line padding (M15/16) for the gateway hand-off and telemetry;
  `noexcept` on hot paths (M14); `TCP_NODELAY`/`writev`/kernel-bypass (M02) for the socket.
- **Single writer per session** is the organizing principle — it deletes locks from the hot path;
  scale by sharding sessions across cores, never by locking one.
- The **gateway manager** wins by giving each session its own ring + writer (no head-of-line blocking),
  a direct-indexed routing table, and a token-bucket rate limiter that back-pressures instead of
  sleeping.
- **Telemetry never touches the hot path**: fixed-size event → lock-free ring → drainer → append-only
  audit log + batched QuestDB + monitoring. The append-only log is legal truth *and* the backtester's
  input *and* the seed for recovery.
- **Recovery = the exchange is truth.** Resend, reconcile against a snapshot/drop-copy, resolve in-
  flight orders by presence, never blindly resend; cancel-on-disconnect trades queue priority for a
  trivial restart.

---

# Where this connects

- **Module 02 (feed handler)** — the OMS reuses its every tool inverted: TCP-vs-UDP reasoning, packed-
  struct (de)serialization, `byteswap`, `memcpy`-into-aligned, `switch` dispatch, the lock-free-ring
  hand-off, and hardware timestamps for latency/audit.
- **Module 03 (order book)** — the object-pool + direct-index + generational-handle pattern of Problem 1
  is the same allocation-killing move as the book's `OrderPool` and `orderMap → vector` optimization.
- **Module 04 (strategy engine)** — its order intents are this module's input; its OTR lever (§2.5) is
  enforced here by the gateway's token bucket; its `OrderRouter` interface is realized by the gateway
  manager, and its fills come back from Problem 1.
- **Modules 15/16** — cache-line alignment/false-sharing and lock-free rings + memory ordering are the
  spine of the store, the gateway hand-off, and the telemetry path.
- **Modules 17/10/14** — CRTP + concepts give zero-cost multi-venue encoding; `noexcept` keeps the hot
  path exception-free.
- **QuantStream stack** — the centralized store is QuestDB via **PG-wire batched inserts** (not the ILP
  client), and retention follows the intraday lookback model, not full backfill.

---

# The mental through-line

> The strategy decides; **the OMS makes it real and keeps the books.** It takes an abstract intent,
> serializes it into a venue's exact bytes, fires it down a single reliable session, and then tracks
> that order through an asynchronous state machine where the exchange — not you — is the truth. Every
> nanosecond is earned with the same tools the feed handler used (pools, packed structs, byteswap,
> lock-free rings, cache discipline, CRTP), but with a stricter creed: **never lose an order, never
> double it, account for every one, survive a disconnect.** That's why the OMS is the box that proves
> you can build infrastructure that touches real money — not just move data fast.

---

**Next:** Module 06 — [the backtester & the exchange simulator](06-backtester-and-simulator.md): the
OMS's append-only event log and recorded market data are what make deterministic replay trustworthy,
and the simulator plugs in behind this module's gateway protocol to validate the whole order lifecycle.
Return to the [index](../00-index.md).
