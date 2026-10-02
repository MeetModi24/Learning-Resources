# Module 05 — The TCP protocol

TCP is how you get **reliable, ordered, exactly-once** delivery over an internet layer that promises none
of it ([Module 03](03-internet-layer.md) — IP is best-effort). The best way to understand TCP is as a case
study in *building reliability on an unreliable foundation*: start with a bare datagram service that can
lose, reorder, and duplicate, then add — one mechanism at a time — connection setup, acknowledgments,
sequence numbers, flow control, and congestion control until you have a reliable byte stream. Every one of
those mechanisms costs latency, which is why HFT uses TCP only where it must (order entry) and tunes it
aggressively. This module builds TCP up mechanism by mechanism, then covers the HFT tuning that matters.

> **One correction up front:** TCP is *not* built on top of UDP — TCP and UDP are **peers**, both riding
> directly on IP. The useful framing is that TCP adds reliability to the same kind of *best-effort datagram
> service* UDP exposes raw. "What do we add to a bare datagram for this?" is the right question at each step.

---

## 1. The problem: reliability on unreliability

IP gives you a datagram that may be **lost** (congestion, corruption, TTL expiry), **reordered** (different
paths), or **duplicated**. Applications like file transfer, email, databases — and **order entry** — can't
tolerate that. TCP layers on the missing guarantees:

| Problem | TCP's mechanism |
|---------|-----------------|
| Packets get lost | acknowledgments + timeout + **retransmission** |
| Packets arrive out of order | **sequence numbers** + receiver reordering buffer |
| Duplicates arrive | sequence numbers → ignore already-seen |
| Fast sender floods slow receiver | **flow control** (sliding window) |
| Senders flood the *network* | **congestion control** (adaptive window) |
| Need a known starting state | **connection** (3-way handshake) + state machine |

---

## 2. The TCP header

```
  0               8              16                            31
 ┌──────────────────────────────┬────────────────────────────────┐
 │ Source Port (16)             │ Destination Port (16)          │
 ├──────────────────────────────┴────────────────────────────────┤
 │ Sequence Number (32)                                           │  byte-offset of this
 ├────────────────────────────────────────────────────────────────┤  segment in the stream
 │ Acknowledgment Number (32)                                     │  next byte expected
 ├──────┬─────────┬──────────────┬────────────────────────────────┤
 │ Data │ Rsvd    │ Flags        │ Window Size (16)               │  flags: SYN ACK FIN
 │Offset│         │ (SYN/ACK/..) │ (receiver's free buffer)       │  RST PSH URG
 ├──────┴─────────┴──────────────┼────────────────────────────────┤
 │ Checksum (16)                │ Urgent Pointer (16)            │
 ├──────────────────────────────┴────────────────────────────────┤
 │ Options (MSS, window scale, SACK, timestamps) … + Padding      │
 └────────────────────────────────────────────────────────────────┘
```

The three fields that carry the reliability: **Sequence Number** (where this segment's bytes sit in the
stream), **Acknowledgment Number** (the next byte the receiver expects — a cumulative ACK), and **Window
Size** (how much more the receiver can buffer — flow control). The **flags** drive the connection state
machine (SYN opens, FIN closes, RST aborts, ACK acknowledges, PSH says "deliver now"). Header is 20 bytes
without options, up to 60 with — vs UDP's 8 ([Module 06](06-udp-protocol.md)).

---

## 3. Connection establishment — the 3-way handshake

Before any data, both sides synchronize initial sequence numbers and exchange parameters (MSS, window
scaling). Three segments:

```
  Client                                   Server
    │  ── SYN  seq=x ───────────────────▶   │   client: CLOSED → SYN_SENT
    │                                        │   server: LISTEN → SYN_RECEIVED
    │  ◀───── SYN seq=y, ACK=x+1 ─────────   │
    │  ── ACK  ack=y+1 ─────────────────▶   │   client: → ESTABLISHED
    │                                        │   server: → ESTABLISHED
    │  ===== data flows both ways =====      │
```

Why three and not two: both sides must prove they can **send and receive**, and both must agree on the
other's **initial sequence number**. The cost that matters for HFT: a connection setup is **one full
round-trip (1 RTT) before the first byte of data** — which is why order sessions are opened and **kept warm**
well before the trading session, never established on the critical path.

### Teardown — the 4-way handshake

```
  A ── FIN ──▶ B      A: → FIN_WAIT
  A ◀─ ACK ── B
  A ◀─ FIN ── B       B finishes sending, then closes
  A ── ACK ─▶ B       A: → TIME_WAIT (waits 2·MSL before fully closing)
```

Each direction is closed independently (TCP is full-duplex). The lingering **TIME_WAIT** state (holding the
4-tuple for ~2× max segment lifetime so stray old segments don't corrupt a new connection) is a real
operational concern for servers that churn many short connections — HFT avoids it by using **long-lived**
sessions rather than reconnecting.

### Connection state machine (the lifecycle)

```
 CLOSED ─(open)→ SYN_SENT ─┐
 LISTEN ─(SYN)→ SYN_RECEIVED ├→ ESTABLISHED ─(FIN handshake)→ FIN_WAIT/CLOSE_WAIT → TIME_WAIT → CLOSED
```

You see these exact names in `netstat`/`ss`. Interviewers probe whether you know `ESTABLISHED` is the
data-transfer state and `TIME_WAIT` is the post-close lingering state.

---

## 4. Reliability — sequence numbers, ACKs, retransmission

Each byte in the stream has a **sequence number**. The receiver sends **cumulative acknowledgments**: "I
have everything up to byte N, send N next." Mechanics:

- **Loss detection:** the sender keeps unacknowledged segments and a **retransmission timer (RTO)**, derived
  from measured RTT. Timer fires with no ACK → **retransmit**.
- **Fast retransmit:** three **duplicate ACKs** (receiver repeatedly asking for the same missing byte)
  signal loss *before* the timer, so the sender resends immediately.
- **Ordering:** out-of-order segments are **buffered** by the receiver and delivered to the app only once the
  gap is filled — this guarantees in-order delivery but causes **head-of-line (HOL) blocking** (below).
- **Duplicates:** a segment whose sequence range was already received is discarded.
- **SACK** (selective ACK, an option) lets the receiver say exactly which ranges it has, so the sender
  resends only the true gap instead of everything after it.

### Head-of-line blocking — the HFT gotcha

Because TCP guarantees in-order delivery, **one lost segment stalls every byte behind it** until the
retransmit arrives — even if those later bytes are sitting in the receiver's buffer. For a market-data
stream that's fatal: a single drop would freeze *all* subsequent updates for a full retransmit RTT. This is
a core reason feeds use **UDP** (no HOL blocking — you just skip the gap and keep going;
[Module 06](06-udp-protocol.md)). For order entry it's an accepted cost because correctness dominates.

---

## 5. Flow control — the sliding window

A fast sender can overflow a slow receiver's buffer. TCP prevents it with the **sliding window**: the
receiver advertises its free buffer space in the **Window Size** field; the sender may have at most that
many **unacknowledged** bytes in flight.

```
 sent & ACKed │ sent, not yet ACKed │ may send now │ cannot send yet
 ─────────────┼─────────────────────┼──────────────┼────────────────
              └──────── window (receiver-advertised) ──────┘  slides right as ACKs arrive
```

As ACKs come in, the window **slides** forward. If the receiver's buffer fills, it advertises a smaller
(even zero) window — **backpressure** that paces the sender. Flow control is strictly about **receiver**
capacity (end-to-end), distinct from congestion control (network capacity).

---

## 6. Congestion control — adapting to the network

Flow control protects the *receiver*; **congestion control** protects the *network* from being flooded by
all senders collectively. TCP maintains a **congestion window (cwnd)** separate from the receiver window and
sends the **minimum** of the two. The classic algorithm:

- **Slow start:** begin with a tiny cwnd, **double it each RTT** (exponential) until a threshold or loss.
- **Congestion avoidance (AIMD):** past the threshold, grow cwnd **linearly** (+1 MSS/RTT) — additive
  increase.
- **On loss:** treat as congestion → **multiplicative decrease** (halve cwnd, or drop to 1 on timeout). This
  Additive-Increase/Multiplicative-Decrease is what makes TCP flows share a link **fairly** and converge.

```
 cwnd
  │        /\        /\          slow start: exponential ramp
  │       /  \      /  \         loss → cut (multiplicative decrease)
  │      /    \    /    \        avoidance: slow linear climb (additive)
  │_____/      \__/      \____▶ time
```

### Why this hurts HFT

**Slow start** means a freshly-opened or recently-idle TCP connection sends *slowly* and ramps up over
several RTTs — exactly wrong for a burst of orders at the open. Mitigations: keep connections **warm and
busy** (don't let them go idle and reset cwnd — `TCP_SLSTART_AFTER_IDLE=0`), tune initial cwnd, and in some
stacks disable the idle reset. Congestion control also means a single loss can **halve** your throughput —
another reason the low-loss, controlled co-lo environment matters.

---

## 7. The HFT TCP tuning checklist

Order entry must be TCP, so HFT makes TCP as low-latency as possible:

- **`TCP_NODELAY` (disable Nagle):** Nagle's algorithm buffers small writes until the previous small segment
  is ACKed, to reduce tiny-packet overhead — which adds up to an RTT of delay to a small order. **Always set
  `TCP_NODELAY`** on an order socket so each write goes out immediately. The single most important TCP knob
  in HFT.
- **Delayed ACK interaction:** the receiver's delayed-ACK (waiting ~40ms to piggyback ACKs) can interact
  pathologically with Nagle; disabling Nagle (and sometimes `TCP_QUICKACK`) avoids the stall.
- **Keep connections warm:** open and authenticate order sessions before trading; never pay the 1-RTT
  handshake or slow-start on the hot path; avoid idle so cwnd doesn't reset.
- **Kernel-bypass TCP:** user-space TCP stacks (Solarflare Onload, and dedicated low-latency TCP libs) run
  the whole state machine in your process with busy-polling — no per-packet syscall/interrupt
  ([Module 04](04-transport-layer.md), [caos Module 08](../caos-guide/08-system-calls.md)).
- **Pin + isolate** the socket's thread/IRQs ([caos Module 09](../caos-guide/09-interrupts-and-exceptions.md))
  so ACK processing isn't preempted.
- **Right-size buffers** and use `SACK`/window scaling appropriately.

---

## Common pitfalls / misconceptions

- **"TCP runs on top of UDP."** No — both are peers over IP. TCP adds reliability to the same best-effort
  datagram service; it doesn't use UDP.
- **"The 3-way handshake transfers data."** It only synchronizes sequence numbers + parameters. Data starts
  after it (barring TCP Fast Open). It costs a full RTT before the first byte.
- **"Flow control and congestion control are the same."** Flow control = don't overflow the **receiver**
  (advertised window). Congestion control = don't overflow the **network** (congestion window). The sender
  obeys the **min** of both.
- **"TCP is slow, period."** TCP is slower than UDP because of its guarantees, but most HFT latency pain is
  specific and fixable: **Nagle** (set `TCP_NODELAY`), **slow start** (keep warm), **handshake** (pre-open),
  **HOL blocking** (unavoidable — hence UDP for feeds).
- **"A dup ACK means the ACK was duplicated by the network."** It means the receiver got an out-of-order
  segment and is re-asking for the still-missing byte; three of them trigger **fast retransmit**.

---

## Quiz — tough problems

**Q1.** Why does TCP need a *three*-way handshake and not two?
**Answer:** Each side must confirm both its send and receive paths work and must learn the other's **initial
sequence number**. SYN (client ISN) → SYN+ACK (server ISN + ack of client's) → ACK (ack of server's). Two
messages would leave one side's ISN unacknowledged and its receive path unverified. It costs ~1 RTT before
data.

**Q2.** A market-data stream over TCP drops one segment. What happens to the 500 segments already received
behind it, and why is this disqualifying for feeds?
**Answer:** They sit in the receiver's buffer **undelivered** — TCP guarantees in-order delivery, so the app
sees nothing past the gap until the lost segment is retransmitted (**head-of-line blocking**), ~1 RTT of
total stall on *all* subsequent updates. For latency-critical feeds that's fatal; UDP has no HOL blocking —
you skip the gap and keep processing, recovering the one lost message out of band.

**Q3.** You send a 20-byte order and it goes out ~40ms later than expected on an otherwise-idle link. Likely
cause and fix?
**Answer:** **Nagle's algorithm** buffering the small write (possibly interacting with the peer's
**delayed-ACK** timer, ~40ms). Fix: set **`TCP_NODELAY`** on the socket so small segments are sent
immediately; consider `TCP_QUICKACK` on the receive side. This is the canonical HFT TCP bug.

**Q4.** Distinguish the receiver window from the congestion window, and say what the sender actually uses.
**Answer:** The **receiver (advertised) window** is the receiver's free buffer — flow control, end-to-end.
The **congestion window (cwnd)** is the sender's estimate of what the *network* can take — congestion
control. The sender may have in flight at most **min(rwnd, cwnd)** unacknowledged bytes.

**Q5.** Why does a freshly reconnected TCP order session send slowly at first, and how do you avoid it on the
open?
**Answer:** **Slow start** — cwnd begins small and doubles each RTT, so early throughput is low; an idle
connection also resets cwnd. Avoid it by keeping the session **open, warm, and non-idle** before the trading
window (and disabling the idle cwnd reset), so you never pay the ramp on the burst of opening orders.

**Q6.** What are three duplicate ACKs telling the sender, and what does it do?
**Answer:** The receiver keeps getting out-of-order segments and re-requesting the same missing byte → likely
a single lost segment (not general congestion). The sender does **fast retransmit** — resends the missing
segment immediately without waiting for the RTO timer — and (with SACK) resends only the actual gap.

---

## Indian HFT interview questions

**Q1 (Optiver — the staple).** *Explain TCP's three-way handshake and why order entry can't just use UDP.*
**Model answer:** The handshake synchronizes initial sequence numbers and parameters over one RTT: SYN
(client ISN), SYN+ACK (server ISN + ack), ACK — both ends reach ESTABLISHED and prove bidirectional
connectivity. Order entry needs TCP because an order must arrive **exactly once, in order, acknowledged** —
UDP's best-effort delivery could silently drop or reorder a trade, which is unacceptable. We pay TCP's cost
but minimize it: pre-open and keep the session **warm** so the handshake and slow start never hit the hot
path, and set `TCP_NODELAY` so orders aren't Nagle-buffered.

**Q2 (Tower Research, Gurgaon — tuning).** *You see ~40ms stalls on small orders over a TCP session. Diagnose
and fix.*
**Model answer:** Classic **Nagle + delayed-ACK** interaction: Nagle holds a small write until the prior
small segment is ACKed, and the peer's delayed-ACK waits ~40ms to piggyback the ACK, so the two deadlock for
that timer. Fix: **`TCP_NODELAY`** on the order socket to send small segments immediately; optionally
`TCP_QUICKACK` on the receive side. More broadly, keep the connection warm to dodge slow start, pin the
thread and NIC IRQs, and consider a kernel-bypass TCP stack to remove per-packet syscall/interrupt cost.

**Q3 (Graviton — HOL blocking).** *Why do exchanges deliver market data over UDP multicast instead of TCP,
given TCP is reliable?*
**Model answer:** TCP's in-order guarantee causes **head-of-line blocking** — one lost segment stalls every
later update for a retransmit RTT, and TCP is per-connection unicast with slow start and congestion control
cutting throughput on loss. For a one-to-many, latency-critical feed that's exactly wrong. UDP **multicast**
sends each update once, the network replicates it to all subscribers, there's **no HOL blocking** (skip the
gap and keep going), and recovery is handled out of band via **sequence-number gap detection** + a secondary
feed or retransmit service. Latency beats loss for data; correctness beats latency for orders — hence the
split.

**Q4 (Quadeye — flow vs congestion).** *Difference between flow control and congestion control, and which
one makes a reconnecting session slow?*
**Model answer:** Flow control is the **sliding window** protecting the receiver from overflow — the receiver
advertises free buffer and the sender caps in-flight bytes to it. Congestion control protects the **network**
via the congestion window, using **slow start** (exponential ramp) then **AIMD**. The sender sends
min(rwnd, cwnd). A reconnecting or idled session is slow because of **slow start / idle cwnd reset** — the
congestion window, not flow control. We keep sessions warm and disable the idle reset to avoid it.

---

## Key takeaways

- TCP builds **reliable, ordered, exactly-once** delivery on IP's best-effort datagrams (it is a **peer** of
  UDP over IP, *not* layered on UDP).
- Header carries **sequence** (byte offset), **ACK** (next byte expected, cumulative), **window** (flow
  control), and **flags** (SYN/ACK/FIN/RST) driving the connection state machine.
- **3-way handshake** (SYN / SYN+ACK / ACK) sets up a connection in ~1 RTT; **4-way FIN** tears it down,
  leaving **TIME_WAIT**. HFT keeps sessions **long-lived and warm** to avoid both on the hot path.
- Reliability = sequence numbers + cumulative ACKs + RTO timeout + **retransmission**, with **fast
  retransmit** on 3 dup-ACKs and **SACK** for precise gaps. In-order delivery causes **head-of-line
  blocking** — disqualifying for feeds.
- **Flow control** = sliding window (don't overflow the **receiver**); **congestion control** = cwnd with
  **slow start + AIMD** (don't overflow the **network**); sender uses **min(rwnd, cwnd)**.
- HFT TCP tuning: **`TCP_NODELAY`** (kill Nagle — the #1 knob), keep connections warm (dodge slow start +
  handshake), kernel-bypass TCP, pin/isolate threads + IRQs.

**Next:** [06 — The UDP protocol](06-udp-protocol.md)
