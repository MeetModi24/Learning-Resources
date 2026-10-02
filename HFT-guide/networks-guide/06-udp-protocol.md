# Module 06 — The UDP protocol

UDP is the transport you reach for when **speed beats guarantees** — and in trading that means
**market data**: a continuous firehose of quotes and trades where a *late* tick is worthless and the
only thing that matters is getting the *freshest* one to your strategy as fast as physically possible.
UDP is deliberately almost nothing: a thin 8-byte header over IP ([Module 03](03-internet-layer.md))
that hands your datagram to the right port and gets out of the way. No handshake, no connection state,
no ACKs, no retransmission, no ordering, no flow or congestion control. The whole design philosophy is
**fire-and-forget** — and understanding *why* that's a feature, not a deficiency, is the heart of this
module.

---

## 1. The UDP philosophy: fire-and-forget

Four principles, and every one maps to a latency win:

| Principle | What it means | Why HFT wants it |
|---|---|---|
| **Fire-and-forget** | send and don't track the result | no ACK wait, no retransmit stall |
| **No state** | each datagram independent | no per-connection memory, no setup |
| **Minimal overhead** | 8-byte header, no management | less to build, parse, and copy |
| **Best-effort** | network delivers what it can | no congestion-control throttle |

TCP ([Module 05](05-tcp-protocol.md)) is a stateful engine that *promises* delivery; UDP is a stateless
courier that *attempts* it. For a high-rate feed where you'd rather skip a lost packet than wait for it,
the courier wins every time.

## 2. The UDP header — 8 bytes, fixed

```
 UDP header (8 bytes, fixed)
 ┌───────────────────────────────┬───────────────────────────────┐
 │ Source port (16)              │ Destination port (16)         │
 ├───────────────────────────────┼───────────────────────────────┤
 │ Length (16)  (header+payload) │ Checksum (16)  (optional)     │
 ├───────────────────────────────┴───────────────────────────────┤
 │ Payload (variable) — your market-data message                 │
 └───────────────────────────────────────────────────────────────┘
```

Only four fields, and that's genuinely all the protocol needs:
- **Source port** — the sending application's port (can be 0 if no reply expected).
- **Destination port** — demux target on the receiving host (how the feed handler gets its data).
- **Length** — total datagram size (header + payload).
- **Checksum** — integrity check over header + payload + an IP pseudo-header. **Optional in IPv4**
  (zero = "not computed"); **mandatory in IPv6**. If the checksum fails, the datagram is **silently
  dropped** — never repaired, never retransmitted.

Compare TCP's 20–60 bytes driving sequence numbers, ACKs, windows, and options. UDP's 8 bytes reflect
the absence of all that machinery. IP handles getting it across the network ([Module 03](03-internet-layer.md));
Ethernet handles the last hop ([Module 02](02-network-access-layer.md)); UDP just drops the payload on
the right port.

## 3. Datagram boundaries — one send, one receive

Unlike TCP's byte stream, UDP **preserves message boundaries**: one `sendto()` of N bytes becomes
exactly one datagram, and the receiver's `recvfrom()` returns exactly that N-byte message (or nothing).
No framing logic, no "loop until a full message arrives." Each market-data message is self-contained.

The flip side: if the payload exceeds the path MTU ([Module 03](03-internet-layer.md)), IP fragments it,
and if *any* fragment is lost the *entire* datagram is dropped (UDP has no partial recovery). So HFT
feeds keep each message **well under the MTU** — one message per unfragmented datagram.

## 4. What you give up — and why it's fine for data

| You lose | Consequence | Why acceptable for market data |
|---|---|---|
| Delivery guarantee | packets may vanish | a dropped tick is recovered via sequence gaps, not retransmit |
| Ordering | packets may reorder | sequence numbers in the payload reorder/detect at the app |
| Flow control | can overwhelm a slow receiver | receiver is a tuned feed handler sized for peak rate |
| Congestion control | can flood the network | colo links are provisioned for the feed; exchange paces it |
| Error recovery | corrupt datagram dropped | gap recovery handles it like a loss |

The theme: UDP gives you a blank slate, and the application layer adds back *exactly* the guarantees it
needs — and nothing more. A feed doesn't need TCP's full reliability; it needs **loss detection** and a
way to recover *selectively*, which it builds itself (Section 6).

## 5. UDP multicast — one send, many receivers

This is why exchanges use UDP, not merely *that* they do. **Multicast** is one-to-many: a single packet
sent to a multicast group address (IPv4 Class D, 224.0.0.0–239.255.255.255, [Module 03](03-internet-layer.md))
is delivered to **every host that joined that group** — the switch/router replicates it, not the sender.

```
                        ┌─▶ subscriber A (feed handler)
   exchange ──1 packet──┤─▶ subscriber B
   (multicast group)    ├─▶ subscriber C
                        └─▶ subscriber D       (network replicates, sender sends once)
```

Why this is the only sane design for an exchange feed:
- **The sender sends once** regardless of subscriber count — constant work whether 10 or 10,000 firms
  listen. Unicasting to each would be O(subscribers) and unfair (early recipients get data first).
- **No per-receiver state** — the exchange tracks nobody; hosts join/leave via **IGMP** without the
  sender knowing.
- **Fairness** — all subscribers get the same packet at essentially the same instant, which is a
  regulatory and competitive requirement.
- **The key point:** if there were a mechanism for the sender to confirm every
  receiver got every packet, processing all those responses would impose huge overhead on the sender and
  **delay transmission for everyone** — so by design it doesn't ask. Fire-and-forget *is* the fairness
  guarantee.

## 6. Reliability on top of UDP — gap recovery and A/B feeds

"Best-effort" doesn't mean "lose data silently." Exchange feeds build their *own* lightweight
reliability in the payload, keeping UDP's speed:

- **Sequence numbers in the payload** — every message carries a monotonically increasing seq. The feed
  handler tracks the next expected seq; a jump (…, 1000, 1001, **1004**) means 1002–1003 were lost.
- **A/B (redundant) feeds** — the exchange publishes the *same* data on two independent multicast groups
  over separate network paths. The handler **arbitrates**: take whichever message arrives first per seq,
  fill gaps on A from B. A packet lost on one path is almost always present on the other, so you recover
  **without any retransmit round-trip**.
- **Recovery / replay channel** — for a gap missing on *both* feeds, a separate (often TCP)
  retransmission or snapshot channel re-requests the range. This is the slow path, used rarely, never on
  the hot path.
- **Snapshots + incrementals** — periodic full-book snapshots let a late joiner or a badly-gapped
  handler resynchronize without replaying the whole day.

That's the HFT reliability model: **detect loss cheaply (seq), recover from redundancy (A/B), fall back
to replay only when forced** — all while never making the hot path wait for an ACK.

## 7. HFT summary: UDP, exploited

- **Market data → UDP multicast**: one send fans out, minimal header, no handshake, no per-receiver
  state, no head-of-line blocking. Freshest tick wins.
- **Loss handled in the app**: payload sequence numbers + A/B feed arbitration + a rarely-used replay
  channel. Never a retransmit stall on the hot path.
- **Kernel-bypass the receive path**: NIC DMAs the datagram to a user-space ring; a busy-polling thread
  parses IP+UDP itself — sub-microsecond wire-to-callback, no syscall, no socket copy
  ([Module 04 §6](04-transport-layer.md)).
- **Keep datagrams unfragmented** (under MTU) so one lost packet is one lost message, not a whole
  fragmented datagram.
- **Size receive buffers / rings for peak burst** — UDP has no flow control, so a slow receiver just
  drops; the feed handler must drain fast enough to never overflow at the open or on a news spike.

---

## Common pitfalls / misconceptions

- **"UDP is just a worse TCP."** UDP is *deliberately* minimal. For latency-critical, loss-tolerant data
  its lack of handshake/ACK/ordering is the *feature* — it's why market data uses it.
- **"UDP loses data, so you can't trust a feed."** The feed adds its own reliability (sequence numbers,
  A/B redundancy, replay). You get loss *detection* and selective recovery without TCP's latency.
- **"The UDP checksum guarantees correct data."** It only *detects* corruption; a bad datagram is
  **silently dropped**, not repaired or resent. And in IPv4 the checksum can even be disabled (zero).
- **"Multicast means the sender transmits to each subscriber."** No — the sender transmits **once**; the
  network (switches/routers) replicates to all group members. That's the whole efficiency and fairness
  argument.
- **"UDP preserves order like it preserves boundaries."** It preserves *message boundaries* (one
  datagram = one message) but **not order** across datagrams — reordering is the app's job via sequence
  numbers.
- **"Bigger UDP messages are fine."** Over the MTU, IP fragments them, and losing one fragment drops the
  whole datagram. Keep feed messages small and unfragmented.

## Quiz — tough problems

**Q1.** Your feed handler sees sequence numbers 1000, 1001, 1004. What happened, and how do you recover
*without* a retransmit round-trip?
**Answer:** Messages **1002 and 1003 were lost** on this feed (a gap). You recover from the **B feed**:
the exchange publishes identical data on a second multicast group over a separate path, so you take 1002
and 1003 from B via **A/B arbitration** — no round-trip. Only if *both* feeds are missing the range do
you fall to the (slow) replay/snapshot channel.

**Q2.** Why does an exchange serving thousands of firms use UDP multicast instead of a TCP connection to
each?
**Answer:** With multicast the exchange **sends each packet once** and the network replicates it to all
group members — constant sender work regardless of subscriber count, no per-receiver state, and every
firm gets the same packet at essentially the same instant (fairness). Thousands of TCP connections would
cost O(subscribers) work, per-connection state, retransmit tracking, and would deliver to early
connections first — unfair and unscalable.

**Q3.** A UDP datagram is corrupted in flight. What happens, and how does the feed handle it?
**Answer:** The checksum fails and the datagram is **silently discarded** — UDP never repairs or
retransmits it. To the feed handler it looks exactly like a lost packet: it shows up as a **sequence-
number gap**, recovered the same way (A/B feed, then replay). There's no automatic retransmission and no
buffering for later.

**Q4.** Why keep market-data datagrams under the MTU?
**Answer:** If a datagram exceeds the path MTU, IP **fragments** it into multiple packets, and UDP has
no partial recovery — if **any single fragment** is lost, the **entire** datagram is dropped. Keeping
each message small and unfragmented means one lost packet costs one message (a single seq gap), not a
whole multi-fragment datagram, and avoids reassembly cost and jitter.

**Q5.** UDP has no flow control. What breaks if your feed handler is too slow, and how do you prevent it?
**Answer:** With no backpressure, the sender/network keeps delivering at full rate; a slow receiver's
socket/NIC ring buffer **overflows and silently drops** packets (seen as seq gaps, worst exactly at the
busy open or a news spike when you can least afford it). Prevent it by draining fast — kernel-bypass +
busy-poll, generously sized receive rings/buffers, a lean hot path, and pinned isolated cores so the
handler never falls behind peak burst rate.

**Q6.** Why is UDP's lack of ordering acceptable for market data when order clearly matters for a book?
**Answer:** Ordering matters, but UDP doesn't have to *provide* it — the **payload carries sequence
numbers**, so the feed handler reorders and detects gaps itself at the application layer. This gives the
exact ordering the book needs without TCP's cost of *blocking all later data* until a missing earlier
packet is retransmitted (head-of-line blocking). The app gets ordering on its own terms, cheaply.

## Indian HFT interview questions

**Q1 (Optiver / Quadeye — the core design question).** *Why is market data distributed over UDP
multicast, and how do you deal with the fact that UDP can lose packets?*
**Model answer:** Market data is high-rate, one-to-many, and latency-critical, and a late tick is
worthless — so it goes over UDP multicast: the exchange sends each packet once and the network
replicates it to every subscriber, giving constant sender cost, no per-receiver state, no handshake, no
head-of-line blocking, and fairness (everyone gets the same packet simultaneously). Loss is handled
*above* UDP: every message carries a sequence number so the handler detects gaps instantly; the exchange
runs redundant **A/B feeds** on separate paths so a packet lost on one is recovered from the other with
no round-trip; and a separate replay/snapshot channel covers the rare double-loss. So we keep UDP's
speed and add back only the reliability we actually need.

**Q2 (Graviton — UDP vs TCP tradeoff).** *If TCP is reliable and UDP isn't, why would anyone choose UDP
for the most important data a trading firm consumes?*
**Model answer:** Because reliability and latency are a tradeoff, and for market data latency wins. TCP's
guarantees cost a handshake, ACK waits, retransmit timers, and — fatally — head-of-line blocking: one
lost segment stalls *all* later data until it's retransmitted, during which fresher prices queue up
unprocessed. A tick recovered a round-trip late is already stale and worthless. UDP lets us skip the gap,
keep consuming fresh prices, and recover the lost one out-of-band from the A/B feed. We don't need TCP's
"every byte, in order, no matter how late" — we need "the newest price, right now," which is exactly what
fire-and-forget UDP gives.

**Q3 (Jump — the gap-recovery mechanics).** *Walk me through what your feed handler does the instant it
detects a sequence gap on the A feed.*
**Model answer:** It doesn't block the hot path. It notes expected-seq vs received-seq to identify the
missing range, then checks the **B feed**, which carries identical data over a separate path — almost
always the missing messages are already there (or arrive momentarily), so it fills the gap by
arbitration and keeps processing in order. Meanwhile it keeps consuming newer messages rather than
stalling. Only if the range is missing on *both* feeds does it issue a request on the (slower)
retransmission/replay channel, or resynchronize from the next periodic snapshot. Throughout, it never
waits on a retransmit before processing fresher data.

**Q4 (Tower Research — receive-path performance).** *UDP has no flow control. How do you make sure you
don't drop packets at the busiest moment of the day?*
**Model answer:** Drops happen when the receive ring/socket buffer overflows because the handler can't
drain fast enough — and that's worst exactly at the open or a news spike. So we remove every source of
delay on the receive path: kernel-bypass (Onload/ef_vi/DPDK) so the NIC DMAs straight to a user-space
ring with no syscall or socket copy, a busy-polling thread so there's no interrupt latency, generously
sized rings/buffers to absorb bursts, a lean branch-predictable hot path, and the handler pinned to an
isolated core with interrupts and other work kept off it. The goal is that peak burst rate still drains
faster than it arrives, so the buffer never fills.

## Key takeaways

- UDP is **connectionless, best-effort, fire-and-forget**: an 8-byte header over IP with ports, length,
  and an (IPv4-optional) checksum — no handshake, state, ACKs, ordering, flow, or congestion control.
- It **preserves message boundaries** (one send = one datagram = one receive), unlike TCP's byte stream;
  keep datagrams **under the MTU** to avoid fragment-loss amplification.
- A corrupt or lost datagram is **silently dropped** — never repaired or retransmitted; the application
  adds back exactly the guarantees it needs.
- **Multicast** makes UDP the right tool for exchange feeds: the sender transmits **once**, the network
  replicates to all group members (joined via IGMP) — constant cost, no per-receiver state, fair
  simultaneous delivery.
- HFT reliability without TCP's latency: **payload sequence numbers** for gap detection, **A/B redundant
  feeds** for round-trip-free recovery, and a rarely-used **replay/snapshot** channel as the fallback —
  all on a kernel-bypassed, busy-polled receive path.

**Next:** back to the [index](00-index.md)
