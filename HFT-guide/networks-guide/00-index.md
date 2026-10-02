# Networks Guide — Index

Computer networking for HFT / low-latency interviews, built bottom-up along the **TCP/IP stack**. See
[README.md](README.md) for how to read this and how it relates to the other guides. Read top-to-bottom if
you're new; jump by topic if you're revising.

## The TCP/IP stack, bottom-up

1. [TCP/IP model overview](01-tcpip-model-overview.md) — the 4 layers, OSI mapping, encapsulation/decapsulation, the end-to-end packet journey, where the microseconds go
2. [The network access layer](02-network-access-layer.md) — physical/link, NIC (DMA/IRQ, kernel bypass, PTP, FPGA), Ethernet frame, MAC addresses, ARP, switches (cut-through), VLANs, MTU/jumbo
3. [The internet layer](03-internet-layer.md) — best-effort IP, addressing/classes/CIDR/subnets, private ranges/NAT, IPv4 header, routing (longest-prefix), fragmentation/PMTU, ICMP, multicast/IGMP, IPv6
4. [The transport layer](04-transport-layer.md) — ports & ranges, the 4-tuple (mux/demux), the socket API, TCP vs UDP tradeoff, the HFT split (feeds → UDP, orders → TCP)
5. [The TCP protocol](05-tcp-protocol.md) — 3-way handshake & state machine, seq/ack + retransmission, head-of-line blocking, sliding-window flow control, congestion control, `TCP_NODELAY`/Nagle
6. [The UDP protocol](06-udp-protocol.md) — connectionless/stateless, 8-byte header, multicast feeds, gap recovery / B-feed, why market data rides UDP

## The through-line

> A message starts as application bytes and is **encapsulated** downward: the **transport layer** adds a
> port + reliability (TCP) or nothing but a port + checksum (UDP); the **internet layer** adds a best-effort
> IP address to cross networks; the **network access layer** frames it with MAC addresses and puts bits on
> the wire. Routers forward it hop-by-hop by **longest-prefix match**, switches forward it inside a LAN by
> **MAC**, and the far NIC **decapsulates** it back up to the right socket. HFT is the art of doing all of
> that in single-digit microseconds — kernel bypass, cut-through switches, hardware timestamps, UDP
> multicast for data, tuned TCP for orders.
