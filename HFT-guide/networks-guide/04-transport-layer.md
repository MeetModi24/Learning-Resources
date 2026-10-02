# Module 04 — The transport layer

The internet layer ([Module 03](03-internet-layer.md)) gets a packet to the right *machine*. But a machine
runs dozens of programs — which one gets the bytes? The transport layer answers that with **port numbers**,
and it's also where you choose your fundamental tradeoff: **TCP** (reliable, ordered, connection-oriented,
heavier) or **UDP** (best-effort, connectionless, minimal, fast). This module is the hinge: it defines
ports, multiplexing, and the socket API, then positions TCP vs UDP. The two protocols each get their own
deep module next ([05 — TCP](05-tcp-protocol.md), [06 — UDP](06-udp-protocol.md)). For HFT the headline is
simple and it's decided here: **market data rides UDP, order sessions ride TCP**, and understanding why is
a guaranteed interview question.

---

## 1. What the transport layer does

It provides **end-to-end communication between applications** (processes), not just between machines. Its
jobs:

- **End-to-end communication** — connect a process on one host to a process on another.
- **Port addressing** — identify *which* application via a 16-bit port number.
- **Reliability** — guarantee delivery + ordering (TCP) or not (UDP).
- **Flow control** — stop a fast sender from overwhelming a slow receiver (TCP).
- **Error detection** — a checksum flags corrupted data.

The clean mental model: once IP/MAC have delivered the packet to the machine, the **port number bifurcates
the data to the correct application**. MAC → machine on a link, IP → machine anywhere, **port → program**.

---

## 2. Ports — addressing the application

A **port** is a 16-bit identifier (0–65535) distinguishing applications/services on one host, so many
programs share the network at once. Ranges:

| Range | Name | Use |
|-------|------|-----|
| **0–1023** | well-known | standard services: 80 HTTP, 443 HTTPS, 22 SSH, 53 DNS |
| **1024–49151** | registered | vendor/app-assigned services |
| **49152–65535** | dynamic / ephemeral | temporary client-side ports, allocated per connection |

When your client connects out, the OS assigns it an **ephemeral** source port; the destination (server) port
is the well-known/known one. That asymmetry is what lets one server port (say 443) serve thousands of
clients — each client is distinguished by *its* ephemeral port + IP.

---

## 3. Multiplexing and demultiplexing — the 4-tuple

Multiple conversations share one host and even one server port. The transport layer keeps them separate by
identifying each connection by a **tuple**:

```
 TCP connection identity (the 4-tuple):
   ( source IP , source port , destination IP , destination port )

 UDP demux (less state):
   deliver by ( destination IP , destination port )  → the bound socket
```

**Demultiplexing** on receive: the IP layer hands up a segment; the transport layer reads the destination
port (TCP: the full 4-tuple) and routes it to the matching **socket**. This is why two browser tabs to the
same server don't get crossed — different source ephemeral ports → different 4-tuples → different sockets.

For HFT this matters when you run **multiple sessions** (e.g. several order gateways, or primary/secondary
feeds): each is a distinct tuple → a distinct socket → independently pollable. It's also why a single UDP
multicast group (one dest IP+port) can be `recv`'d by many subscriber processes.

---

## 4. Sockets — the programming interface

A **socket** is the OS abstraction your program uses to speak to the transport layer — the API endpoint
binding `(protocol, IP, port)` to a file descriptor. In Unix a socket **is a file descriptor**, so the same
`read`/`write`/`close` + `epoll` machinery from [caos Module 12 (the I/O path)](../caos-guide/12-allocators-and-io.md)
applies. The canonical flows:

```
 TCP server            TCP client            UDP (both ends)
 ─────────             ─────────             ───────────────
 socket()              socket()              socket()
 bind()                connect() ──┐         bind()        (server side)
 listen()                          │         sendto()/recvfrom()
 accept() ◀── 3-way handshake ─────┘         (no connection, no accept)
 read()/write()        read()/write()
 close()               close()
```

TCP needs `listen`/`accept` and a handshake because it's connection-oriented; UDP just `bind`s and
`sendto`/`recvfrom`s — no connection state. Every one of these calls is a **syscall** trapping into the
kernel ([caos Module 08](../caos-guide/08-system-calls.md)) — which is exactly the per-packet overhead HFT
removes:

- **Kernel-bypass sockets** (Solarflare Onload, DPDK, VMA) present a socket-like API but run the TCP/UDP
  stack in user space and **busy-poll** the NIC — no syscall, no kernel copy, no interrupt per packet.
- Even in-kernel, HFT uses `recvmmsg`/`sendmmsg` (batch many datagrams per syscall), `SO_BUSY_POLL`, and
  `epoll` to amortize or avoid syscall and interrupt cost.

---

## 5. TCP vs UDP — the decision

The two transport protocols are opposite points on the reliability-vs-latency tradeoff:

| Feature | TCP | UDP |
|---------|-----|-----|
| Connection | connection-oriented (handshake first) | connectionless |
| Reliability | **guaranteed** delivery (retransmit) | best-effort (may drop) |
| Ordering | **guaranteed** in-order | none |
| Flow control | yes | no |
| Congestion control | yes | no |
| State | per-connection state machine | **stateless** |
| Header | 20–60 bytes | **8 bytes** |
| Speed / latency | slower, more overhead + buffering | faster, minimal |
| Typical use | web, email, file transfer, **order sessions** | streaming, gaming, DNS, **market-data feeds** |

The one-liner: **TCP buys reliability and ordering with latency and state; UDP buys minimal latency by
giving you nothing but a port and a checksum.** Everything TCP does — handshake, ACKs, retransmit,
reordering buffer, congestion window — is latency you don't pay with UDP, and in HFT latency is often worse
than loss.

### The HFT split

- **Market-data feeds → UDP (multicast).** One-to-many fan-out where a retransmit would arrive too late to
  be useful; you'd rather detect a gap (via sequence numbers) and recover from a secondary feed. UDP's tiny
  8-byte header and statelessness are a bonus. (Deep dive: [Module 06](06-udp-protocol.md).)
- **Order entry → TCP.** You cannot "best-effort" an order — you need it to arrive exactly once, in order,
  acknowledged. Exchanges mandate TCP (often FIX or a binary order protocol over TCP) for order gateways,
  and HFT tunes it hard: `TCP_NODELAY` to kill Nagle buffering, pre-warmed connections, kernel-bypass TCP
  stacks. (Deep dive: [Module 05](05-tcp-protocol.md).)

A subtlety interviewers like: UDP being "faster" isn't magic — it's faster *because it does less*. If you
rebuild reliability on top of UDP (sequence numbers, selective retransmit), you can end up re-implementing a
leaner, latency-tuned slice of TCP tailored to your traffic — which is precisely what exchange recovery
protocols do.

---

## 6. Error detection

Both TCP and UDP carry a **checksum** covering the header + payload (and a pseudo-header of the IP
addresses), so a corrupted segment is detected and dropped. TCP then **retransmits** the lost/corrupt
segment; UDP simply drops it and tells no one — recovery, if any, is the application's problem. (UDP
checksum is technically optional on IPv4 but effectively always on.) This is the error-detection half of
reliability; the retransmit half is what separates the two protocols.

---

## Common pitfalls / misconceptions

- **"Ports are a layer-3/IP thing."** Ports are **transport-layer**. IP addresses the machine; ports
  address the application on it.
- **"A connection is identified by the destination port alone."** A TCP connection is the full **4-tuple**
  (src IP, src port, dst IP, dst port). That's how one server port serves thousands of clients.
- **"UDP is faster, so always use it."** UDP is faster because it omits reliability/ordering/flow/congestion
  control. If your app needs those, you either use TCP or re-implement them on UDP (and may lose the
  advantage). Right tool per traffic type.
- **"A socket is a network concept."** A socket is an **OS file descriptor** — same `read`/`write`/`epoll`
  machinery as files/pipes; see [caos Module 12](../caos-guide/12-allocators-and-io.md).
- **"TCP vs UDP is about security."** Neither is encrypted; that's TLS at the application layer. The TCP/UDP
  choice is about reliability vs latency.

---

## Quiz — tough problems

**Q1.** One web server listens on port 443. A thousand clients are connected. How does the transport layer
keep their data separate?
**Answer:** By the **4-tuple** (src IP, src port, dst IP, dst port). All share dst IP + dst port 443, but
each client has a distinct source IP and/or **ephemeral source port**, so every connection is a unique tuple
mapped to its own socket. Demultiplexing routes each segment by its tuple.

**Q2.** Why does UDP have an 8-byte header while TCP's is 20–60 bytes? Name what those extra bytes buy.
**Answer:** UDP carries only source port, dest port, length, checksum (8 bytes) — no connection state. TCP's
extra bytes carry **sequence/acknowledgment numbers** (ordering + retransmit), **window size** (flow
control), **flags** (SYN/ACK/FIN for the connection state machine), and options (MSS, window scaling,
timestamps, SACK). Those bytes *are* reliability, ordering, flow and congestion control.

**Q3.** Your strategy opens one TCP order session and joins one UDP multicast feed. Which transport calls
differ, and why no `accept` for the feed?
**Answer:** TCP order session: `socket→connect→read/write` (client), requiring a 3-way handshake. UDP feed:
`socket→bind→recvfrom` (plus an IGMP join for the group) — no handshake, no connection, so no
`listen`/`accept`. UDP is connectionless: you just bind and receive datagrams.

**Q4.** An interviewer says "UDP is just TCP minus reliability, so you can always bolt reliability back on
for free." Push back.
**Answer:** You can add sequence numbers and selective retransmit on UDP, and exchanges do — but it's not
free: you re-implement buffering, loss detection, and ordering, and if you add full reliability you
reintroduce much of TCP's latency. The win comes from building a *leaner* reliability tuned to your traffic
(e.g. gap-detect + recover from a B feed instead of in-order blocking retransmit), not from getting TCP's
guarantees at UDP's cost.

**Q5.** Where does the per-packet syscall cost come from on the receive path, and how does HFT remove it?
**Answer:** Each `recvfrom`/`read` traps into the kernel ([caos Module 08](../caos-guide/08-system-calls.md)),
and the normal path also takes a NIC interrupt + kernel copy. HFT removes it with **kernel-bypass** sockets
(Onload/DPDK) that run the UDP/TCP stack in user space and **busy-poll** the NIC — no syscall, no interrupt,
no copy — or amortizes it with batched `recvmmsg` and `SO_BUSY_POLL`.

---

## Indian HFT interview questions

**Q1 (Optiver — the classic).** *Why does an exchange send market data over UDP but require TCP for order
entry?*
**Model answer:** Market data is a one-to-many, latency-critical fan-out where a retransmit would arrive too
late to matter — better to send once via **UDP multicast**, let subscribers detect a sequence-number gap,
and recover from a secondary feed or retransmit service. UDP's statelessness and 8-byte header also minimize
overhead. Order entry is the opposite: an order must arrive **exactly once, in order, acknowledged** — you
can't best-effort a trade — so it rides **TCP**, which guarantees delivery and ordering. We then tune TCP
with `TCP_NODELAY`, pre-warmed connections, and often a kernel-bypass TCP stack to claw back its latency.

**Q2 (Tower Research, Gurgaon — mechanism).** *How does the transport layer get data to the right
application, and how do you run multiple sessions without crossing streams?*
**Model answer:** Via **port numbers** and, for TCP, the full **4-tuple** (src IP, src port, dst IP, dst
port). The IP layer delivers to the machine; the transport layer demultiplexes by port/tuple to the matching
**socket**. Running multiple order gateways or primary/secondary feeds just means distinct tuples → distinct
sockets, each independently pollable — a single UDP multicast group can also be received by many subscriber
sockets. No crossing because every stream has a unique tuple.

**Q3 (Graviton — sockets + latency).** *What is a socket, and where's the latency in the socket API on the
hot path?*
**Model answer:** A socket is the OS endpoint binding `(protocol, IP, port)` to a file descriptor — in Unix
it's a normal fd, so `read`/`write`/`epoll` apply. The latency is that every `recv`/`send` is a **syscall**
into the kernel, plus a NIC interrupt and a kernel→user copy on the standard path. On the hot path we
**kernel-bypass** (Onload/DPDK/VMA): the UDP/TCP stack runs in user space, the NIC rings map into our
process, and we **busy-poll** — eliminating the syscall, the interrupt, and the copy — or we batch with
`recvmmsg`/`sendmmsg` and enable `SO_BUSY_POLL` when staying in-kernel.

---

## Key takeaways

- The transport layer provides **end-to-end, application-to-application** communication, addressing the app
  via a 16-bit **port**: MAC → machine on a link, IP → machine anywhere, **port → program**.
- Port ranges: **0–1023** well-known, **1024–49151** registered, **49152–65535** ephemeral (client side).
- A connection is identified by the **4-tuple** (src IP, src port, dst IP, dst port) — how one server port
  serves many clients. The transport layer **demultiplexes** segments to sockets by this.
- A **socket** is an OS file descriptor binding `(proto, IP, port)`; TCP needs
  `socket/bind/listen/accept` + handshake, UDP just `socket/bind/sendto/recvfrom`. Each call is a syscall —
  the per-packet cost HFT removes via **kernel bypass** and batched `recvmmsg`.
- **TCP** = connection-oriented, reliable, ordered, flow + congestion controlled, 20–60B header, slower.
  **UDP** = connectionless, best-effort, stateless, 8B header, fast. TCP buys guarantees with latency; UDP
  gives latency by doing less.
- The HFT split, decided here: **feeds → UDP multicast** (latency > loss), **orders → TCP** (exactly-once,
  ordered). Detailed in [Module 05](05-tcp-protocol.md) and [Module 06](06-udp-protocol.md).

**Next:** [05 — The TCP protocol](05-tcp-protocol.md)
