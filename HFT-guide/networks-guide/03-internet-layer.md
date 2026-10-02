# Module 03 — The internet layer

The network access layer ([Module 02](02-network-access-layer.md)) can only move a frame across one link.
The internet layer is what makes a *network of networks* — it gives every machine a globally meaningful
**IP address** and forwards **packets** hop by hop across arbitrarily many networks to reach any of them.
It's deliberately **best-effort**: connectionless, unreliable, no ordering guarantee. If you want
reliability you build it on top at the transport layer ([Module 04](04-transport-layer.md)). For HFT the
internet layer is where **co-location** (minimizing hops), **multicast** (one-to-many feed distribution),
and **TTL/scoping** decisions live — and where every extra router between you and the matching engine is
latency and jitter you can't take back.

---

## 1. What the internet layer does

Four jobs:

- **Logical addressing** — assign IP addresses to devices.
- **Routing** — determine the best path for a packet across networks.
- **Packet forwarding** — move packets between different networks, hop by hop.
- **Fragmentation** — split a packet that's larger than a link's MTU (and the destination reassembles).

The key promise: it connects devices that are **not** on the same local network. The network access layer
got you across the room; the internet layer gets you across the planet. The dominant protocol is **IP**
(with **ICMP** for control/error signaling riding alongside it).

Best-effort means exactly that: IP will *try* to deliver a packet and will silently drop it on congestion,
corruption, or a TTL expiry. No retransmit, no ordering, no duplicate suppression. Those are TCP's job.

---

## 2. IP addresses — logical, hierarchical, global

An **IP address** is a *logical* address assigned to a device — unlike a MAC, which is physical and burned
into hardware. The contrast matters:

| | MAC (layer 2) | IP (layer 3) |
|---|---------------|--------------|
| Nature | physical, hardware-bound | logical, reassignable |
| Structure | flat | **hierarchical** (network + host) |
| Scope | link-local | **global**, routable |
| Lifetime across a route | rewritten every hop | constant end to end |

**IPv4** is a 32-bit number written as dotted-decimal — four octets 0–255, e.g. `192.168.1.100`. The
hierarchy is the important part: an address splits into a **network portion** (which network) and a **host
portion** (which device on it). That split is what makes routing scalable — routers forward toward a
*network*, not toward billions of individual hosts.

---

## 3. Classes, subnet masks, and CIDR

Historically IPv4 was carved into **classes** by the leading bits:

| Class | Leading bits | Range | Default split | Use |
|-------|-------------|-------|---------------|-----|
| A | `0…` | 1.0.0.0 – 126.255.255.255 | Network.Host.Host.Host | huge nets (~16M hosts) |
| B | `10…` | 128.0.0.0 – 191.255.255.255 | Network.Network.Host.Host | medium (~65K hosts) |
| C | `110…` | 192.0.0.0 – 223.255.255.255 | Network.Network.Network.Host | small (~254 hosts) |
| D | `1110…` | 224.0.0.0 – 239.255.255.255 | — | **multicast** (see §7) |

Classes were wasteful (a Class B's 65K hosts were too many for most, too few in Class C), so the modern
scheme is **CIDR** (Classless Inter-Domain Routing): a **subnet mask** of arbitrary length marks where the
network portion ends.

**Subnet mask** = the bits that are network vs host. Example:

```
 IP Address :  192.168.1.100     = 11000000.10101000.00000001.01100100
 Subnet Mask:  255.255.255.0  /24  = 11111111.11111111.11111111.00000000
                                      └──────── network ───────┘└─ host ─┘
 Network    :  192.168.1.0       (host bits zeroed)
 Host       :  100               (the host bits)
 Broadcast  :  192.168.1.255     (host bits all ones)
 Usable hosts: .1 – .254         (254 addresses)
```

The `/24` is CIDR notation for "24 network bits." Worked subnetting: split `192.168.1.0/24` into two `/25`s
→ `192.168.1.0/25` (hosts .1–.126) and `192.168.1.128/25` (hosts .129–.254). Each borrowed host-bit halves
host count and doubles subnet count. Interviewers love "how many usable hosts in a /26?" → 2^(32−26) − 2 =
**62** (minus network + broadcast).

**Private ranges** (RFC 1918, not routable on the public Internet): `10.0.0.0/8`, `172.16.0.0/12`,
`192.168.0.0/16`. **NAT** (Network Address Translation) maps many private addresses behind one public
address by rewriting the IP (and port) in the header — which is also why "the source IP can change in
transit" is true *only* when a NAT is in the path. HFT co-lo networks are typically flat private subnets
with no NAT on the hot path (NAT adds a translation + state lookup = latency).

---

## 4. The IPv4 header

Everything a router needs to move the packet lives in the 20-byte (minimum) IPv4 header:

```
  0               8              16                            31
 ┌────┬──────┬──────────────┬──────────────────────────────────┐
 │Ver │ IHL  │ TOS/DSCP     │ Total Length (16b)                │  Ver=4, IHL=header words,
 │(4) │ (4)  │ (8)          │ (whole packet, header+data)       │  TOS=QoS/priority
 ├────┴──────┴──────────────┼──────────┬────────────────────────┤
 │ Identification (16)      │ Flags(3) │ Fragment Offset (13)   │  ID+flags+offset =
 │                          │ DF/MF    │                        │  fragmentation control
 ├───────────┬──────────────┼──────────────────────────────────┤
 │ TTL (8)   │ Protocol (8) │ Header Checksum (16)              │  TTL -1 per hop;
 │ hop limit │ 6=TCP 17=UDP │                                   │  Protocol selects L4
 │           │ 1=ICMP       │                                   │
 ├───────────┴──────────────┴──────────────────────────────────┤
 │ Source IP Address (32)                                       │
 ├──────────────────────────────────────────────────────────────┤
 │ Destination IP Address (32)                                  │
 ├──────────────────────────────────────────────────────────────┤
 │ Options (optional) …          │           Payload (L4 segment)│
 └──────────────────────────────────────────────────────────────┘
```

Fields worth knowing cold:

- **Version** — 4 for IPv4.
- **IHL** — header length in 32-bit words (so the payload offset is known).
- **TOS / DSCP** — QoS/priority marking (used in some networks to prioritize traffic classes).
- **Total Length** — entire packet size including header (max 65,535).
- **Identification / Flags / Fragment Offset** — the fragmentation machinery (§6). `DF` = Don't Fragment.
- **TTL** — decremented by **every router**; at 0 the packet is dropped and an ICMP "time exceeded" is
  returned. Prevents infinite routing loops; also the engine behind `traceroute`.
- **Protocol** — what's in the payload: **6 = TCP, 17 = UDP, 1 = ICMP**. This is how the receiver's IP
  layer knows which transport handler to call (decapsulation).
- **Header Checksum** — covers the header only (not payload); recomputed at each hop because TTL changes.
- **Source / Destination IP** — the end-to-end endpoints; constant across the route (barring NAT).

Note the header checksum covers only the header, and IPv6 drops it entirely — a deliberate simplification
to speed per-hop processing (§8).

---

## 5. Routers and routing

A **router** connects multiple networks and forwards packets between them by **IP** — contrast the switch,
which forwards within one network by **MAC**:

| Feature | Router | Switch |
|---------|--------|--------|
| Layer | Internet (L3) | Network access (L2) |
| Addressing | IP | MAC |
| Function | connect *different* networks | connect devices on *same* network |
| Broadcast domain | **separates** them | single broadcast domain |
| Decision | routing-table lookup | CAM-table lookup |

**Router operation per packet:** receive on an interface → read destination IP → **route lookup** in the
routing table → pick outgoing interface + next hop → **decrement TTL, recompute checksum, re-frame** for the
outgoing link (new MACs!) → forward.

**Routing table** — how to reach each destination network:

```
 Destination Network | Subnet Mask     | Next Hop     | Interface | Metric
 192.168.1.0         | 255.255.255.0   | 0.0.0.0      | eth0      | 1    ← directly connected
 10.0.0.0            | 255.0.0.0       | 192.168.1.1  | eth0      | 2    ← via gateway
 0.0.0.0             | 0.0.0.0         | 192.168.1.1  | eth0      | 3    ← DEFAULT route
```

The `0.0.0.0/0` entry is the **default gateway** — "everything I don't have a specific route for, send
here." Lookups use **longest-prefix match**: the most specific (longest mask) matching entry wins, so a
`/24` route beats the `/0` default for an address inside it.

**Static routing** (manually configured, simple, stable — common in small/controlled networks like a
trading co-lo) vs **dynamic routing** (protocols like OSPF/BGP learn and adapt — needed in large,
changing networks). HFT prefers **static** routes on the hot path: deterministic, no reconvergence pauses,
no surprise path changes mid-session.

---

## 6. Fragmentation (and why HFT avoids it)

If a packet exceeds a link's **MTU** (1500 bytes on standard Ethernet), IP **fragments** it: split into
MTU-sized pieces, each with its own IP header carrying the shared **Identification**, the **More Fragments**
flag, and a **Fragment Offset** so the destination can **reassemble** in order. Only the *destination*
reassembles — routers don't.

Fragmentation is latency- and reliability-hostile:

- Lose **one** fragment and the **whole** packet is undeliverable (TCP/app must resend everything).
- Reassembly costs buffering + CPU at the receiver.
- It's a jitter source and a historical attack surface.

So the discipline is **Path MTU Discovery**: set the **DF (Don't Fragment)** bit and let any too-small link
bounce an ICMP "fragmentation needed" telling you the MTU, then size packets to fit. HFT's practical rule
is simpler: keep **every message inside a single frame** (well under 1500 bytes — order and tick messages
are tens of bytes anyway) so fragmentation never happens on the hot path.

---

## 7. ICMP and multicast

**ICMP** (Protocol 1) is IP's control/error channel — not for data, for signaling:

- **Echo request/reply** → `ping` (reachability + round-trip time).
- **Time Exceeded** (TTL hit 0) → the mechanism behind **`traceroute`**: send packets with TTL = 1, 2, 3…
  and each router in turn returns a Time-Exceeded, revealing the hop-by-hop path. This is how you *measure*
  how many routers sit between you and the exchange — directly relevant when every hop is latency.
- **Destination Unreachable**, **Fragmentation Needed** (the PMTU signal above).

**Multicast** (Class D, `224.0.0.0/4`) is **the** internet-layer feature for HFT. A multicast group has one
IP; a single packet sent to it is delivered to **every** subscriber, with the network (switches via **IGMP
snooping**, routers via IGMP/PIM) replicating it only along paths that have subscribers. This is exactly how
exchanges distribute **market data**: the matching engine sends each update **once** to a group address, and
every participant's feed handler receives it simultaneously — no per-client unicast, minimal and *fair*
latency across subscribers.

- **IGMP** (Internet Group Management Protocol) is how a host **joins/leaves** a multicast group; the switch
  snoops IGMP to forward the group only to ports that asked for it (so non-subscribers aren't flooded).
- **TTL / scoping** bounds how far a multicast travels (TTL small = stays local), which exchanges use to
  keep a feed inside the co-lo and prevent it leaking across routers.
- HFT consequence: your feed handler ([infra Module 02](../hft-infrastructure-guide/02-feed-handler.md))
  `IGMP-join`s the exchange's groups and must handle **gaps** (dropped multicast packets are not
  retransmitted — you detect a sequence-number gap and recover from a secondary feed or a retransmit
  service), which is the whole reason UDP+multicast is chosen over TCP for feeds ([Module 06](06-udp-protocol.md)).

---

## 8. IPv6 (briefly)

IPv4's 32 bits (~4.3B addresses) ran out; **IPv6** uses **128 bits** (written as eight hex groups,
`2001:db8::1`). Beyond address space, it simplifies the header for faster per-hop processing: **no header
checksum**, **no in-network fragmentation** (hosts must do PMTU discovery; routers never fragment),
fixed 40-byte header, built-in support for autoconfiguration. For HFT the IPv6/IPv4 choice is dictated by
the exchange; the latency-relevant takeaways are the ones that reduce per-hop work (no checksum, no router
fragmentation).

---

## Common pitfalls / misconceptions

- **"IP is reliable."** No — IP is **best-effort**: connectionless, may drop/reorder/duplicate. Reliability
  is TCP's job one layer up.
- **"The source IP changes as it's routed."** Constant end to end **unless a NAT** is in the path. It's the
  **MAC** that's rewritten per hop.
- **"A router and a switch are interchangeable."** Router = L3, forwards between networks by IP, separates
  broadcast domains, slower (more processing). Switch = L2, forwards within a network by MAC.
- **"TTL is a time in seconds."** It's a **hop count** — decremented per router, not per second. At 0 the
  packet dies (and emits ICMP Time Exceeded).
- **"Fragmentation is harmless."** Losing one fragment kills the whole packet; it adds reassembly cost and
  jitter. HFT keeps messages inside one frame and sets DF.
- **"Multicast is just broadcast."** Broadcast hits *every* device on the LAN; multicast hits only group
  **subscribers**, with the network replicating along subscribed paths (IGMP). It's targeted fan-out.

---

## Quiz — tough problems

**Q1.** `192.168.1.100/26` — what's the network address, broadcast address, and number of usable hosts?
**Answer:** `/26` = mask `255.255.255.192`; host bits = 6. The .100 falls in the block `.64–.127`:
**network `192.168.1.64`**, **broadcast `192.168.1.127`**, usable `.65–.126` = **62 hosts** (2^6 − 2).

**Q2.** A packet leaves with TTL 64 and arrives with TTL 57. What happened, and what would TTL 0 cause?
**Answer:** It crossed **7 routers** (each decremented TTL by 1). If TTL reached 0 at some router, that
router would **drop** the packet and send an **ICMP Time Exceeded** back to the source — which is exactly
how `traceroute` enumerates the path (deliberately sending TTL=1,2,3,…).

**Q3.** Why does a router recompute the IP header checksum at every hop, but your TCP checksum is computed
once end to end?
**Answer:** The router modifies the header each hop (it decrements **TTL**), so the header checksum — which
covers only the header — must be recomputed. The **TCP** checksum covers the transport segment and payload,
which routers never touch, so it's computed by the source and verified only by the destination (transport is
end-to-end).

**Q4.** Market data arrives as UDP multicast and you occasionally miss an update. Why doesn't the network
just resend it, and how do you cope?
**Answer:** Multicast/UDP is **best-effort** — IP doesn't retransmit and UDP adds no reliability, by design,
because waiting for a retransmit would be worse than a gap for a latency-critical fan-out feed. You cope by
putting **sequence numbers** in the application protocol: detect the gap, and recover from a **secondary
(B) feed** or a dedicated **retransmit/recovery service**, rather than stalling the primary.

**Q5.** Why does HFT prefer static routing and avoid NAT on the hot path?
**Answer:** **Static routing** is deterministic — no dynamic-protocol reconvergence pauses, no surprise path
changes mid-session, predictable latency. **NAT** adds a per-packet address/port rewrite plus a connection-
state lookup (latency + jitter + a stateful box that can fail), so co-lo hot paths use flat routable private
subnets with no NAT between the strategy and the gateway.

**Q6.** Both a switch flood and an IP broadcast "go everywhere." How is multicast different, and why does it
matter for feeds?
**Answer:** Flood/broadcast reach **every** device on the segment regardless of interest. **Multicast** is
targeted: hosts **IGMP-join** a group, switches snoop IGMP and forward the group **only to subscribed
ports**, and the matching engine sends each update **once** to be replicated along subscribed paths. For a
market-data feed that means one send reaches all participants with minimal, *fair* latency and without
drowning non-subscribers.

---

## Indian HFT interview questions

**Q1 (Optiver — addressing fundamentals).** *Difference between an IP and a MAC address, and which one
survives end to end across the Internet?*
**Model answer:** A MAC is a flat, hardware-bound, link-local address; an IP is a logical, hierarchical,
globally routable one. The **IP** source/destination are the true endpoints and stay constant end to end
(unless NAT rewrites them), while the **MAC** pair is rewritten at every hop because each hop is a fresh
link. Routers forward by IP (longest-prefix match in the routing table); switches forward by MAC. ARP
([Module 02](02-network-access-layer.md)) bridges the two by resolving the next-hop IP to the MAC we frame.

**Q2 (Tower Research, Gurgaon — latency topology).** *Why does co-location matter at the internet layer, and
how would you measure the hops to the exchange?*
**Model answer:** Every router between us and the matching engine adds store/forward latency and, worse,
**jitter**, and each hop re-frames and re-checksums the packet. Co-location puts us on the same subnet or
one controlled hop from the gateway, so routing is a trivial directly-connected lookup with static routes —
deterministic and minimal. To measure the path I'd use `traceroute`, which exploits **TTL** + ICMP Time
Exceeded to reveal each router; in production I'd rely on **NIC hardware timestamps** for the true
wire-to-wire number rather than ICMP, which is deprioritized by routers.

**Q3 (Graviton — multicast).** *Explain how an exchange distributes market data at the IP layer and what
your feed handler must do.*
**Model answer:** The exchange publishes to **multicast** group addresses (Class D). Each update is sent
**once**; switches use **IGMP snooping** and routers IGMP/PIM to replicate it only toward subscribers, so
every participant receives it near-simultaneously and fairly. My feed handler **IGMP-joins** the groups,
reads via a kernel-bypass socket, and — because multicast/UDP is best-effort with no retransmit — tracks
**sequence numbers** to detect gaps and recovers from the **B feed** or a retransmit service. We also respect
multicast **TTL/scoping** so the feed stays within the co-lo.

**Q4 (Quadeye — fragmentation).** *What is IP fragmentation and why do low-latency systems go out of their
way to avoid it?*
**Model answer:** When a packet exceeds a link's MTU, IP splits it into fragments (shared Identification,
More-Fragments flag, Fragment Offset) that the destination reassembles. It's avoided because losing a single
fragment destroys the entire packet, reassembly costs buffering and CPU, and it injects jitter. The fix is
to set the **DF** bit and do Path-MTU discovery, but in practice our order and tick messages are tens of
bytes — far under 1500 — so we simply guarantee every message fits in **one frame** and fragmentation never
occurs on the hot path.

---

## Key takeaways

- The internet layer provides **logical addressing + routing** to connect *different* networks; it's
  **best-effort** (connectionless, unreliable) — reliability is TCP's job above.
- **IP** is logical, hierarchical (network + host), global, routable — opposite of the flat, link-local MAC.
  IPv4 = 32 bits dotted-decimal; **subnet mask / CIDR** (`/24`) marks the network/host split.
- The **IPv4 header** carries TTL (hop count, −1 per router, 0 → drop + ICMP), Protocol (6 TCP / 17 UDP /
  1 ICMP), fragmentation fields, header-only checksum, and the end-to-end src/dst IPs.
- **Routers** forward by IP via routing-table **longest-prefix match**, re-framing (new MACs) and
  decrementing TTL each hop; the **default route** `0.0.0.0/0` catches the rest. HFT uses **static** routes.
- **Fragmentation** is latency/reliability-hostile; set **DF**, do PMTU, and keep every HFT message in one
  frame.
- **ICMP** powers ping/traceroute (via TTL). **Multicast** (Class D + **IGMP**) is how feeds fan out — one
  send, network-replicated to subscribers; best-effort, so the app uses **sequence numbers** for gap
  recovery.
- **IPv6** = 128-bit, no header checksum, no in-network fragmentation — simpler per-hop processing.

**Next:** [04 — The transport layer](04-transport-layer.md)
