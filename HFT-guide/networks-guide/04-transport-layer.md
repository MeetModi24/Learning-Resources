# Module 04 — The transport layer

The internet layer ([Module 03](03-internet-layer.md)) gets a packet from *host* to *host* — it finds
the right machine on the planet and nothing more. But a machine runs hundreds of programs at once: a
browser, an SSH session, three trading strategies, a logger. The transport layer is the layer that
turns "the packet reached the box" into "the packet reached *this specific process*," and — if you
chose TCP — "in order, exactly once, with nothing lost." It is the first layer that two *application
endpoints* genuinely share; everything below it only cared about moving bits between network cards.

This module is the map of that layer: what it does, how ports and sockets demultiplex traffic to the
right process, and the TCP-vs-UDP decision that governs every networked system — including which half
of an HFT stack uses which. The two protocols then get their own deep dives:
[Module 05 — TCP](05-tcp-protocol.md) and [Module 06 — UDP](06-udp-protocol.md).

---

## 1. What the transport layer is for

The internet layer's contract is deliberately weak: **best-effort, host-to-host, connectionless**. An
IP packet can be dropped, duplicated, reordered, or delayed arbitrarily, and IP addresses only name a
*machine*. The transport layer sits directly on top and adds two things the application actually needs:

1. **Process-to-process delivery (addressing).** IP delivers to a NIC; the transport layer delivers to
   a *program*. It does this with **port numbers** — the "apartment number" on top of IP's "street
   address."
2. **A delivery contract the app can choose.** Either *best-effort* again but lightweight (UDP), or a
   *reliable, ordered byte stream* with flow and congestion control (TCP). The application picks the
   contract by picking the protocol.

```
   Application  ("send these bytes to that service")
        │
   ┌────▼─────────────────────────────────────────────┐
   │ TRANSPORT  — ports + reliability contract          │  TCP / UDP
   │   • demux arriving data to the right socket         │
   │   • TCP: ordering, retransmit, flow/congestion ctrl │
   └────┬─────────────────────────────────────────────┘
        │  hands a segment/datagram down with a port header
   ┌────▼──────┐
   │ INTERNET   │  IP: host-to-host, best-effort (Module 03)
   └───────────┘
```

Everything below the transport layer is the network's job; everything at and above it runs on the two
*end hosts* only. This is the **end-to-end principle**: reliability, if you want it, is implemented at
the endpoints (TCP), not inside the routers. That principle is exactly why HFT can rip the kernel's TCP
stack out and run its own in user space — the network doesn't know or care who implements the contract.

---

## 2. Ports: addressing a process, not a machine

A **port number is a 16-bit identifier** (0–65535) that names a communication endpoint *within* a host.
IP gets you to the machine; the port gets you to the right program on it. Without ports, a host could
hold exactly one conversation at a time; with them it holds up to 65 536 per protocol per peer.

Ports fall into three ranges (IANA):

| Range | Name | Who uses it |
|-------|------|-------------|
| **0 – 1023** | Well-known / system | Standard services: 22 SSH, 80 HTTP, 443 HTTPS, 53 DNS. Binding these usually needs privilege. |
| **1024 – 49151** | Registered | Vendor/app-assigned (e.g. 5432 PostgreSQL). |
| **49152 – 65535** | Dynamic / ephemeral | Short-lived client-side source ports the OS hands out per outbound connection. |

A typical client connection uses a *well-known destination port* (you connect to `:443`) and an
*ephemeral source port* the kernel picks for you. In an HFT order gateway, the exchange publishes the
destination port of its order-entry endpoint; your session's source port is ephemeral and irrelevant to
anyone but your own kernel's demux table.

---

## 3. Sockets, the 5-tuple, and demultiplexing

A **socket** is the OS's handle for one transport endpoint. What uniquely identifies a *connection* —
and so what the OS keys its demux table on — is the **5-tuple**:

```
   ( protocol , source IP , source port , destination IP , destination port )
     TCP/UDP     1.2.3.4     49231         203.0.113.9      443
```

When a segment arrives, the kernel has already used the destination IP to get it to this host (internet
layer). Now the transport layer reads the port fields and looks up *which socket* this belongs to:

- **UDP demux** keys mostly on `(dst IP, dst port)` — a UDP socket bound to a port receives datagrams
  sent to it regardless of source (which is why one UDP socket can receive a whole multicast group).
- **TCP demux** keys on the *full 5-tuple* — this is why one server port (`:443`) can hold thousands of
  simultaneous connections: each is a distinct 5-tuple, each its own socket.

```
   wire ──▶ [internet layer: right host?] ──▶ [transport: which socket?]
                                                   │
            lookup 5-tuple in socket table ────────┤
                                                    ▼
            ┌───────────┬───────────┬───────────┐
            │ socket A  │ socket B  │ socket C   │  → deliver to owning process
            │ strat-1   │ strat-2   │ logger     │
            └───────────┴───────────┴───────────┘
```

This split is **multiplexing** (many processes' data funnelled down onto one IP layer on send) and
**demultiplexing** (one incoming stream fanned back out to the right socket on receive). The socket API
(`socket()`, `bind()`, `connect()`, `send()`/`recv()`) is the boundary where this becomes a syscall into
the kernel — see [Module 08 — System calls](../caos-guide/08-system-calls.md) for the trap cost, and
[Module 12 — Allocators & I/O](../caos-guide/12-allocators-and-io.md) for the socket buffers and the
raw-socket path. Every `recv()` is a kernel round-trip; that cost is precisely what HFT kernel bypass
exists to delete.

---

## 4. Segmentation and terminology

The application hands the transport layer a lump of bytes. The transport layer breaks it into
transport-layer units sized to fit inside IP packets (respecting the path MTU, ~1500 bytes of Ethernet
payload — [Module 02](02-network-access-layer.md)):

- A **TCP segment** is one chunk of the byte stream plus a TCP header. TCP tracks byte offsets so the
  far end can reassemble the stream in order.
- A **UDP datagram** is one self-contained message plus an 8-byte UDP header — one `send` = one datagram
  = (ideally) one packet; there is no stream, no reassembly across messages.

Keep the vocabulary straight — interviewers test it: at the transport layer the unit is a **segment**
(TCP) or **datagram** (UDP); the internet layer wraps it in a **packet**; the link layer wraps that in a
**frame** (see the encapsulation diagram in [Module 01](01-tcpip-model-overview.md)).

---

## 5. The two protocols: TCP vs UDP

The whole layer offers exactly two mainstream contracts, and the choice between them is one of the most
common interview questions in any systems role:

| Feature | **TCP** | **UDP** |
|---------|---------|---------|
| Connection | Connection-oriented (handshake first) | Connectionless (just send) |
| Reliability | Guaranteed delivery (ACK + retransmit) | Best-effort, no retransmit |
| Ordering | Guaranteed in-order byte stream | None — datagrams can arrive reordered |
| Flow control | Yes (receiver window) | No |
| Congestion control | Yes (adapts to the network) | No — you send as fast as you like |
| Data model | Continuous **byte stream** | Discrete **datagrams** (message boundaries preserved) |
| Header size | 20–60 bytes | **8 bytes** |
| Setup cost | 1 RTT handshake before any data | Zero — first packet carries payload |
| Speed / overhead | Slower, more state and bookkeeping | Faster, minimal overhead, stateless |
| Typical use | Web, email, file transfer, **order entry** | Streaming, gaming, DNS, **market-data feeds** |

The mental model: **TCP gives you a reliable phone call** (dial, confirm the other side picked up, then
talk in order, with "say that again?" built in). **UDP gives you postcards** — you drop them in the box,
they're tiny, they go instantly, and nobody tells you if one got lost. Neither is "better"; they're
different contracts for different failure tolerances.

> **The subtle point interviewers want:** TCP's reliability is not free magic — it is *latency and
> jitter*. Every guarantee (handshake, ACKs, retransmit-on-loss, congestion backoff) is a round trip or
> a stall. UDP has none of that machinery, so it has none of that latency — but *you* now own loss and
> ordering. The engineering question is never "which is faster" in the abstract; it's "who should pay
> for reliability, the protocol or my application?"

---

## 6. How an HFT stack splits the two

HFT does not pick one protocol — it uses **both, on purpose, for opposite reasons.** This is the single
most important application of this module and a near-guaranteed interview topic:

```
   EXCHANGE ──── market data (prices, trades) ────▶  UDP MULTICAST  ──▶ strategy
     │            one stream, thousands of listeners, loss-tolerant
     │
   strategy ──── orders / cancels / fills ────────▶  TCP (often)   ──▶ EXCHANGE
                  must not be lost, dropped, or duplicated
```

- **Market data → UDP multicast.** The exchange must push the *same* price updates to every member
  simultaneously and as fast as physically possible. TCP can't do this: it's point-to-point and would
  make the exchange retransmit per-subscriber, adding latency and unfairness. UDP multicast sends each
  update **once** onto the wire and every subscriber's NIC picks it up — minimal header (8 bytes), no
  handshake, no per-receiver state. The feed tolerates occasional loss because exchanges run **A/B
  redundant feeds** and a sequence-number gap triggers a separate recovery/snapshot request — the
  application owns reliability, not the protocol. (Deep dive: [Module 06](06-udp-protocol.md).)
- **Order entry → usually TCP.** You absolutely cannot afford to silently lose an order or send it
  twice — the correctness and risk cost dwarfs the latency cost, and order-entry volume per session is
  low. So the reliable, ordered, exactly-once contract of TCP is worth its overhead here. The latency
  that *does* matter on this path is attacked separately — disabling Nagle's algorithm with
  `TCP_NODELAY` so small order messages aren't buffered, and running the stack in user space to avoid
  kernel round-trips. (Deep dive: [Module 05](05-tcp-protocol.md).)

The headline you say out loud: **"Fan-out, loss-tolerant, latency-critical ⇒ UDP multicast. Low-volume,
must-not-lose, correctness-critical ⇒ TCP. Market data is the first; order entry is the second."**

---

## Common pitfalls / misconceptions

- **"UDP is just a worse TCP."** No — UDP is a *different contract*. For one-to-many fan-out, for
  loss-tolerant real-time data, and for anything where a stall is worse than a drop, UDP is the *correct*
  choice, not a compromise. HFT market data would be unworkable over TCP.
- **"A port is a physical thing."** A port is a 16-bit number in a header and an entry in the kernel's
  socket table. Nothing physical. The NIC/link layer deals in MAC addresses, not ports.
- **"One server port = one client."** One listening port holds thousands of TCP connections because the
  *5-tuple* is the key, not the port alone. Each distinct source IP/port makes a distinct connection.
- **"TCP guarantees low latency because it's reliable."** It guarantees *delivery and order*, which is
  the opposite trade — reliability is bought with retransmits and congestion backoff, both of which can
  spike latency badly. Reliable ≠ fast.
- **"Switching to UDP automatically makes me faster."** Only if your app genuinely tolerates loss and
  reordering. If you then rebuild acks and ordering on top of UDP badly, you've reinvented a slower TCP.
- **"The transport layer routes packets."** No — *routing* is the internet layer (Module 03). The
  transport layer never looks at the network topology; it only addresses *processes* via ports.

---

## Quiz — tough problems

**Q1.** A single server listens on TCP port 443 and is handling 10 000 live connections. How are they
kept apart, and what's the theoretical scaling limit?
**Answer:** Each connection is a distinct **5-tuple** `(TCP, srcIP, srcPort, dstIP, 443)`; the kernel
demuxes on the full tuple, so the shared destination port is no problem. The limit isn't 65 536 — that
would only bound connections *from a single* source IP+port combination. Across many client IPs and
ephemeral source ports the ceiling is really file-descriptor/memory limits, not port count.

**Q2.** Why can one UDP socket receive data from thousands of different senders while a TCP socket
(post-accept) talks to exactly one peer?
**Answer:** UDP is connectionless — its demux keys essentially on the local `(IP, port)`, so any datagram
sent there is delivered; `recvfrom` tells you the sender. A connected TCP socket is bound to a specific
5-tuple established by the handshake, so it only carries that one peer's stream. (This is exactly why
market-data multicast is UDP: one socket, many sources, fan-out.)

**Q3.** You replace TCP with UDP for an order-entry link to "cut latency." What breaks?
**Answer:** You lose guaranteed delivery, ordering, and exactly-once semantics. A dropped order
vanishes silently; a duplicated/reordered datagram could double-fill or mis-sequence. For order entry
the correctness and risk cost vastly outweighs TCP's latency, so this is the wrong trade — latency on
that path is cut with `TCP_NODELAY` and kernel bypass instead, not by abandoning reliability.

**Q4.** Header overhead: how many bytes does the transport layer add for TCP vs UDP, and why does it
matter for a market-data feed?
**Answer:** TCP 20–60 bytes (options push it up); UDP a flat **8 bytes**. On a feed pushing millions of
tiny updates, the per-message header is a real fraction of the packet and of the serialization time on
the wire — another reason feeds favour UDP's minimal header.

**Q5.** Where does the transport layer live — in the routers along the path, or only on the end hosts?
Why does the answer matter to HFT?
**Answer:** Only on the **end hosts** (end-to-end principle); routers forward IP packets and never touch
ports or TCP state. That's precisely what lets HFT replace the kernel's transport implementation with a
user-space / kernel-bypass stack — the network is agnostic to who implements the transport contract.

---

## Indian HFT interview questions

**Q1 (Optiver / Graviton — the classic).** *Why does market data use UDP but order entry uses TCP?*
**Model answer:** Market data is a one-to-many broadcast of the same price updates to every member,
must be as fast as possible, and tolerates rare loss — so exchanges use **UDP multicast**: each update
hits the wire once, every subscriber's NIC receives it, 8-byte header, no handshake, no per-subscriber
state. Loss is handled above the protocol via A/B redundant feeds and sequence-gap-triggered recovery.
Order entry is low-volume and *must not* lose, duplicate, or reorder an order — the correctness/risk
cost dominates latency — so it uses **TCP** for reliable, ordered, exactly-once delivery, with
`TCP_NODELAY` and a bypass stack to claw back the latency that matters.

**Q2 (Tower Research — fundamentals).** *What exactly identifies a connection, and how does the kernel
route an incoming segment to the right socket?*
**Model answer:** The **5-tuple**: protocol, source IP, source port, destination IP, destination port.
On receive, the internet layer has already delivered to this host by destination IP; the transport
layer then reads the port fields and looks the 5-tuple up in the socket table to find the owning socket
(and process). TCP keys on the full tuple (so one port holds many connections); UDP keys on the local
IP+port (so one socket can receive a whole multicast group).

**Q3 (Jump / IMC — tradeoff reasoning).** *Is UDP always lower latency than TCP? When would TCP actually
be the lower-latency choice?*
**Model answer:** UDP has no handshake, no ACK/retransmit, no congestion backoff, so in the common case
it's lower latency and lower jitter. But if the application genuinely needs reliability and you rebuild
acks/ordering on top of UDP, you can end up slower and buggier than TCP's decades-tuned implementation.
And on a lossy path, TCP's single fast retransmit can beat a naive UDP app that detects loss slowly. The
honest answer: UDP wins when the app truly tolerates loss; otherwise reliability has to be paid for
*somewhere*, and TCP often pays it more efficiently than hand-rolled code.

---

## Key takeaways

- The transport layer adds **process-to-process addressing** (ports) and a **chosen delivery contract**
  (TCP reliable stream vs UDP best-effort datagram) on top of IP's host-to-host best-effort service.
- A **port** is a 16-bit process identifier; a **socket/connection** is identified by the **5-tuple**
  `(proto, srcIP, srcPort, dstIP, dstPort)`, which is how the kernel demultiplexes arriving data.
- **TCP** = connection-oriented, reliable, ordered, flow/congestion-controlled byte stream, 20–60-byte
  header, higher latency. **UDP** = connectionless, best-effort, unordered datagrams, 8-byte header,
  minimal latency. Pick by failure tolerance, not by "which is faster."
- Reliability is a cost (round trips, retransmits, backoff), not free magic — the end-to-end principle
  puts that cost on the end hosts, which is what lets HFT run its own user-space transport.
- HFT uses **both**: **UDP multicast for market data** (fan-out, speed, loss-tolerant with sequence
  recovery) and **TCP for order entry** (must not lose/duplicate), tuned with `TCP_NODELAY` and bypass.
- The deep mechanics of each contract — TCP's handshake/state machine/congestion control and UDP's
  datagram model and multicast — are the next two modules.

**Next:** [05 — The TCP protocol](05-tcp-protocol.md)
