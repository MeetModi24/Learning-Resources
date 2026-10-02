# Computer Networking — HFT Interview Guide (Networks)

A theory-first, interview-focused guide to the networking layer under every trading system: **how a message
actually crosses the wire** from one process to another, built bottom-up along the TCP/IP stack. This is the
layer HFT interviews (Tower Research, Optiver, Graviton, Quadeye, AlphaGrep, IMC, Jump, HRT, Da Vinci,
Squarepoint, NK Securities, Mansard, Qube, Millennium) grill after C++ and systems — because in trading the
network *is* the race: the fastest path from the exchange's matching engine to your strategy and back wins.

The goal is **understanding, not memorization**: every topic is built from zero for a beginner, then climbed
to the depth an interviewer probes, with ASCII diagrams of what's on the wire, tough quizzes with answers,
and the exact India-desk interview questions that map to it. Where the networking matters for latency, the
HFT angle is folded in — kernel bypass, cut-through switching, hardware PTP timestamps, UDP multicast feeds,
tuned TCP order sessions.

> **Relationship to the other guides.** The [`../caos-guide`](../caos-guide/) guide covers the machine and
> the OS (syscalls, interrupts, the I/O path, raw sockets) — the layer *below* the socket. This guide picks
> up at the socket and goes *out onto the network*. The [`../hft-infrastructure-guide`](../hft-infrastructure-guide/)
> covers the deployment angle (co-location, feed handlers, exchange connectivity). Read this one when the
> words "packet", "ARP", "multicast", "handshake", or "`TCP_NODELAY`" feel like magic.

## How to read this

- **If you're new:** read top to bottom. The stack is built bottom-up — access layer (bits + MAC) →
  internet layer (IP + routing) → transport layer (ports + sockets) → TCP and UDP in depth. Each layer
  stands on the one below it, so the order matters.
- **If you're revising for an interview:** each module is self-contained and ends with a **quiz** and an
  **interview questions** section (with model answers). Jump to the topic you're weak on — but the TCP-vs-UDP
  decision ([04](04-transport-layer.md)) and the handshake/HOL-blocking material ([05](05-tcp-protocol.md),
  [06](06-udp-protocol.md)) are the ones that come up most.
- **The through-line:** *application bytes are encapsulated down the stack (port → IP → MAC → bits), forwarded
  hop-by-hop by routers (longest-prefix) and switches (MAC), and decapsulated back up to the right socket on
  the far side.* Every module is one layer of that sentence.

## Table of contents

| # | File | Topics |
|---|------|--------|
| 01 | [tcpip-model-overview.md](01-tcpip-model-overview.md) | The **4-layer TCP/IP model** and its OSI mapping, PDU names, **encapsulation/decapsulation**, the end-to-end journey of a packet, and **where the microseconds go** (kernel bypass preview) |
| 02 | [network-access-layer.md](02-network-access-layer.md) | Physical + link layer, serialization, the **NIC** (DMA/IRQ, **kernel bypass**, PTP timestamping, FPGA), the **Ethernet frame**, **MAC addresses**, **ARP**, **switches** (CAM table, **cut-through vs store-and-forward**, L1 replication), VLANs, MTU/jumbo frames |
| 03 | [internet-layer.md](03-internet-layer.md) | Best-effort **IP**, MAC vs IP, addressing/classes/**CIDR**/subnet masks, private ranges/**NAT**, the **IPv4 header**, **routers & routing tables** (longest-prefix match, static vs dynamic), **fragmentation/PMTU**, **ICMP** (ping/traceroute), **multicast/IGMP**, IPv6 |
| 04 | [transport-layer.md](04-transport-layer.md) | **Ports** & ranges, the **4-tuple** (multiplexing/demultiplexing), the **socket API** (and kernel-bypass sockets), **TCP vs UDP** tradeoff table, and the **HFT split** (feeds → UDP multicast, orders → TCP), error detection/checksum |
| 05 | [tcp-protocol.md](05-tcp-protocol.md) | The **3-way handshake** & connection state machine, the TCP header, **seq/ack + retransmission** (fast retransmit, SACK), **head-of-line blocking**, **sliding-window flow control**, **congestion control** (slow start, AIMD), and the **HFT tuning checklist** (`TCP_NODELAY`/Nagle, warm connections, kernel-bypass TCP) |
| 06 | [udp-protocol.md](06-udp-protocol.md) | **Connectionless/stateless** delivery, the **8-byte header**, why there's no handshake/ordering/flow-control, **multicast** market-data feeds, **sequence-number gap recovery** and the **B-feed**, and why latency-critical data rides UDP |

## Notation & conventions

- **ASCII diagrams** show what's on the wire (frame/packet/segment layouts) and the end-to-end path.
- **`**Qn.**` + immediate `**Answer:**`** in the quiz sections — try before reading.
- **Firm-attributed** interview questions are the real India-desk style for that topic; model answers follow.
- **HFT framing** is folded into each module where latency is at stake, not siloed — the point is to see
  *why* the networking fact matters for a trading system.
