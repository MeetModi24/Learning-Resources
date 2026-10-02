# Module 02 — The network access layer

This is the bottom of the stack — the layer that turns bytes into voltages, light, or radio and shoves
them across *one* physical link to a device on the *same* local network. It owns two jobs OSI splits into
Physical (signaling on the wire) and Data Link (framing + hardware addressing). It's also the layer HFT
cares about most physically: this is where the NIC lives, where kernel-bypass hardware plugs in, where
cut-through switches shave nanoseconds, and where co-location cross-connects terminate. Get this layer and
you understand the first and last microseconds of every tick-to-trade path.

---

## 1. What this layer does

The internet layer ([Module 03](03-internet-layer.md)) gives you global addressing and routing, but it
assumes something can actually push bits between two physically-adjacent boxes. That "something" is the
network access layer. It is really two things glued together:

- **Hardware + driver** that serialize digital bits into physical signals and deserialize them back —
  electrical pulses on copper, light on fibre, radio on Wi-Fi.
- **A protocol on top** (almost always **Ethernet** on a wired LAN) that defines a **frame** format, a
  hardware addressing scheme (**MAC**), and the rules for who may transmit on a shared medium (media
  access control).

Its scope is strictly **one link / one LAN**. It cannot get a frame to a machine on a different network —
that's the internet layer's job. Think "across the room," not "across the planet."

```
   bits  ─serialize─▶  signal on medium  ─deserialize─▶  bits
   (NIC + driver)       (copper/fibre/radio)              (peer NIC + driver)
         ▲                                                      │
         └──────── Ethernet frame: addressing + framing ◀───────┘
```

---

## 2. The physical link: serialization and media

**Serialization** = converting bits to a physical signal for transmission; **deserialization** = the
reverse on receive. The medium decides the physics:

| Medium | Signal | HFT note |
|--------|--------|----------|
| Copper (twisted pair) | electrical | cheap, short runs; `10GBASE-T` has ~2µs+ PHY latency — avoided on hot paths |
| **Fibre optic** | light in glass | low, predictable latency; the HFT default inside a datacenter |
| Coaxial | electrical, shielded | legacy |
| Wi-Fi / cellular / Bluetooth | radio | irrelevant for the hot path; microwave/mmWave links matter *between* datacenters |

Two latency facts worth carrying: light in fibre travels at ~**2/3 c** (~5 µs per km), which is why the
Chicago–New York microwave links (radio through air is faster than light through glass) exist and why
physical distance is literally alpha. And **`10GBASE-T` copper PHY encoding adds ~2 µs** vs direct-attach
or fibre — HFT shops pick the media and the SFP for latency, not convenience.

Ethernet speed grades you should recognize: `10BASE-T` (10 Mbps), `100BASE-TX` (100 Mbps), `1000BASE-T`
(1 Gbps), `10GBASE-T` (10 Gbps), and beyond (25/40/100 GbE in modern trading infra).

---

## 3. The NIC — where software meets the wire

The **Network Interface Card** is the hardware that connects a machine to the network. Its functions:

- **Signal conversion** — bits ↔ physical signals.
- **Frame processing** — builds outgoing Ethernet frames, parses incoming ones.
- **Error detection** — computes/checks the **CRC** (FCS) over each frame.
- **Media access control** — manages access to the medium.

On receive in a *normal* kernel, the NIC **DMAs** the frame into a RAM ring buffer and raises an **IRQ**;
the kernel's interrupt handler and NAPI softirq then push it up the stack. That interrupt + driver + copy
path is exactly what [caos Module 09 (interrupts)](../caos-guide/09-interrupts-and-exceptions.md) and
[caos Module 12 (the I/O path)](../caos-guide/12-allocators-and-io.md) dissect — and exactly what HFT
tears out:

- **Kernel-bypass NICs** (Solarflare + Onload, Exablaze/Cisco Nexus SmartNICs, Mellanox/NVIDIA with
  VMA/DPDK) map the NIC's packet rings into user memory so the trading process **busy-polls** them. No
  interrupt, no kernel copy, no syscall per packet — the frame appears in your buffer and you read it.
- **Hardware timestamping (PTP, IEEE 1588)** — the NIC stamps each frame's arrival at the wire with
  nanosecond precision, so you measure true wire-to-wire latency instead of lying software clocks.
- **FPGA NICs** parse Ethernet/IP/UDP *in silicon* and can fire an order in tens of nanoseconds with no
  CPU in the loop.

This is why "the network access layer" is not boring plumbing to an HFT engineer — it's where the biggest
single latency win (bypassing the kernel) physically happens.

---

## 4. The Ethernet frame

Ethernet needs surprisingly little to work. The frame:

```
 Ethernet Frame  (14-byte header + payload + 4-byte trailer)
 ┌──────────────┬──────────────┬───────────┬───────────────────┬────────┐
 │ Dest MAC     │ Src MAC      │ EtherType │ Payload           │  FCS   │
 │ (6 bytes)    │ (6 bytes)    │ (2 bytes) │ (46–1500 bytes)   │ (4B)   │
 ├──────────────┼──────────────┼───────────┼───────────────────┼────────┤
 │AA:BB:CC:DD:EE:FF 11:22:33:44:55:66  0x0800   [ IP packet … ]   CRC32  │
 └──────────────┴──────────────┴───────────┴───────────────────┴────────┘
   (preamble + SFD, 8 bytes, precede this on the wire for clock sync)
```

- **Destination / Source MAC** — who on this link the frame is to/from.
- **EtherType** — what's inside: `0x0800` = IPv4, `0x86DD` = IPv6, `0x0806` = ARP, `0x8100` = 802.1Q VLAN
  tag. This is how the receiver's NIC knows which upper-layer handler to call (decapsulation).
- **Payload** — the IP packet. Minimum **46 bytes** (padded if smaller → 64-byte minimum frame), maximum
  **1500 bytes** (the standard MTU).
- **FCS (CRC32)** — error-detection trailer; a corrupted frame is silently dropped by the receiver's NIC.

### MTU and jumbo frames

The **MTU** is the largest payload a link will carry. Standard Ethernet = **1500 bytes**; **jumbo frames**
= **9000 bytes** on tuned networks. Jumbo frames cut per-byte header overhead and interrupt count for bulk
throughput — but HFT messages are tiny (tens of bytes), so jumbo frames rarely help the order path; they
matter for bulk/replay/reference-data transfer. The real MTU lesson for HFT is to keep every message
**inside one frame** so it's never fragmented ([Module 03](03-internet-layer.md)).

---

## 5. MAC addresses

A **MAC address** is a 48-bit (6-byte) identifier burned into every network interface — the device's
"physical address" on a local network, written as `AA:BB:CC:DD:EE:FF`.

```
 MAC = 48 bits = 6 bytes
 ┌──────────────────────────┬──────────────────────────┐
 │ OUI (24 bits)            │ Device ID (24 bits)       │
 │ manufacturer (IEEE-     │ unique per NIC from that  │
 │ assigned prefix)         │ manufacturer              │
 ├──────────────────────────┼──────────────────────────┤
 │ AA:BB:CC                 │ DD:EE:FF                  │
 └──────────────────────────┴──────────────────────────┘
```

- **OUI** (first 24 bits) — Organizationally Unique Identifier, identifies the manufacturer.
- **Device ID** (last 24 bits) — unique per card from that vendor.

Three delivery types encoded in the destination MAC:

- **Unicast** — one specific device.
- **Multicast** — a group (first bit of the address set); the hardware mechanism market-data feeds ride.
- **Broadcast** — `FF:FF:FF:FF:FF:FF`, every device on the LAN (used by ARP requests).

**MAC vs IP — the distinction interviewers probe:** MAC is **flat** and **link-local** (not routable — a
router won't forward based on MAC); IP is **hierarchical** and **global** (routable). That's precisely why,
as a frame crosses the Internet, the **IP stays constant end to end** while the **MAC is rewritten at every
hop** — each hop is a new link with new link-local endpoints. (See [Module 01 §5](01-tcpip-model-overview.md)
and [Module 03](03-internet-layer.md).)

---

## 6. ARP — bridging IP to MAC

Here's the gap most overviews gloss: to put a frame on the wire you need the **destination MAC**,
but the application gave you a **destination IP**. **ARP (Address Resolution Protocol)** resolves an IPv4
address to a MAC on the local link:

```
 Host A wants to send to 10.0.0.5 but doesn't know its MAC:
   1. A broadcasts an ARP REQUEST (dst MAC = FF:FF:FF:FF:FF:FF):
        "Who has 10.0.0.5? Tell 10.0.0.2 (AA:BB:CC:...)"   → every device sees it
   2. Only 10.0.0.5 replies, UNICAST ARP REPLY:
        "10.0.0.5 is at DD:EE:FF:00:11:22"
   3. A caches IP→MAC in its ARP cache (ageing timer), then frames and sends.
```

Subsequent frames skip the broadcast and use the cache. **Gratuitous ARP** (an unsolicited announcement of
your own IP→MAC) is how a box refreshes neighbors' caches — and how **failover** works: a backup takes over
an IP and gratuitously ARPs so switches/hosts re-point to its MAC. For a destination on a *different*
network, A doesn't ARP for the final host at all — it ARPs for the **default gateway's** MAC and sends the
frame there ([Module 03](03-internet-layer.md)).

HFT relevance: ARP resolution and cache misses add a one-time latency and, worse, jitter. Trading boxes
**pre-populate static ARP entries** for the exchange gateways and peers so the hot path never stalls on an
ARP round-trip, and never risks an ARP-cache-miss spike mid-session.

---

## 7. Switches — intelligent frame forwarding

Early Ethernet put every device on one shared cable (**bus topology**) — every transmission collided with
every other, and the medium was a shared broadcast domain. A **switch** fixes this by giving each device a
dedicated port and forwarding each frame only where it needs to go.

**How a switch learns and forwards (MAC/CAM table):**

```
 LEARN:    frame arrives on port 3 with src MAC X  →  record  X → port 3  in CAM table
 FORWARD:  frame arrives with dst MAC Y
             table has Y → port 7   →  send ONLY out port 7   (unicast forward)
             Y not in table         →  FLOOD out all ports except the source (then learn from the reply)
```

So a switch builds its MAC→port table by watching **source** MACs, and forwards by looking up **destination**
MACs. Unknown destination → flood; broadcast/multicast → flood (modulo IGMP snooping, below). Switch
functions: **learning, forwarding, filtering** (don't bother ports that don't need the frame), and
**loop prevention** via Spanning Tree Protocol (STP) so redundant links don't create broadcast storms.

**Collision vs broadcast domains:** each switch port is its own collision domain (full-duplex, no
collisions); a broadcast/ARP still floods the whole LAN (one broadcast domain) unless you segment with
**VLANs**.

### Latency-critical switching: cut-through vs store-and-forward

This is the HFT heart of the switching discussion:

| Mode | Behavior | Latency |
|------|----------|---------|
| **Store-and-forward** | buffer the *entire* frame, verify CRC, then forward | higher; grows with frame size |
| **Cut-through** | start forwarding as soon as the destination MAC is read (first ~6 bytes) | ~tens–hundreds of **ns**; independent of frame size |

HFT uses **cut-through** low-latency switches (Arista 7130/Metamako, Exablaze/Cisco Nexus) that forward in
**single-digit nanoseconds** and offer **layer-1 replication** (a passive fan-out of the market-data feed to
many subscribers with near-zero added latency). The tradeoff: cut-through can forward a frame that later
fails CRC (it didn't wait to check) — accepted, because the receiver drops the bad frame anyway and the
latency win dominates.

### VLANs

A **VLAN** (802.1Q) partitions one physical switch into multiple logical broadcast domains by inserting a
4-byte tag (EtherType `0x8100`) carrying a 12-bit VLAN ID. It isolates traffic (market data vs orders vs
management) without separate hardware — useful for segmentation and limiting broadcast scope.

---

## 8. Topologies (quick reference)

| Topology | Shape | Fault tolerance | Reality |
|----------|-------|-----------------|---------|
| Bus | one shared cable | poor (single point of failure, collisions) | obsolete |
| **Star** | all devices → central switch | switch is the single point; otherwise scalable/reliable | **the modern default** |
| Ring | circular, data flows around | one break can halt it (unless dual-ring) | token-passing legacy |
| **Mesh** | every device ↔ every device | **best** (maximum redundancy) | expensive; critical systems only |

Modern LANs are **star** (devices around a switch). Mesh gives the best fault tolerance but is expensive and
complex — reserved for critical cores. HFT datacenters are effectively star-of-switches with carefully
engineered, minimal-hop paths from feed → switch → trading box → switch → gateway.

---

## Common pitfalls / misconceptions

- **"A switch routes packets."** No — a switch forwards *frames* at layer 2 by **MAC**; a router forwards
  *packets* at layer 3 by **IP**. "L3 switches" blur the line, but the exam answer: switch = L2/MAC,
  router = L3/IP.
- **"A hub and a switch are the same."** A hub is a dumb repeater that floods every bit to every port
  (shared collision domain); a switch learns MAC→port and forwards only where needed. Switches won.
- **"MAC addresses are routable like IPs."** MACs are **flat and link-local** — they don't survive past the
  first router. IP is the global, routable address.
- **"The source MAC/IP both change at each hop."** Only the **MAC** pair is rewritten per hop; the **IP**
  pair is end-to-end constant (no NAT).
- **"Jumbo frames always help latency."** They help *throughput/overhead* for bulk transfer. HFT order
  messages are tiny; jumbo frames don't help the hot path and a too-large message risks fragmentation.
- **"Cut-through switches verify every frame."** They don't — they forward after reading the destination
  MAC, so a bad-CRC frame can be forwarded. The receiver discards it; HFT accepts this for the latency win.

---

## Quiz — tough problems

**Q1.** A frame crosses host → switch → router → switch → host on another subnet. At how many points is the
destination MAC rewritten, and does the destination IP ever change?
**Answer:** The MAC is link-local, so it's set/rewritten on **each link segment** — host→router (dest MAC =
router's), then router→host (dest MAC = final host's). The switch does *not* rewrite MACs; it only forwards
by them. The destination **IP never changes** (no NAT) — it's end-to-end.

**Q2.** Why does a switch learn from *source* MACs but forward by *destination* MACs?
**Answer:** When a frame arrives on a port, its **source** MAC proves "that device is reachable out this
port," so the switch records src→port. To deliver a frame it needs the port for the **destination**, which
it looks up in that same table. Learning is a side effect of every frame; forwarding is the lookup. Unknown
dest → flood, then it learns the location from the reply.

**Q3.** You see a one-off ~1 ms latency spike on the first order to a counterparty each morning. Network-
access cause and fix?
**Answer:** Likely an **ARP cache miss** — the first frame to that IP triggers an ARP broadcast + reply
round-trip (and possibly a gateway resolution) before the frame can go out. Fix: install **static ARP
entries** for the exchange gateways/peers so the hot path never performs ARP at runtime.

**Q4.** Minimum and maximum standard Ethernet frame payload, and why a 10-byte order still occupies more.
**Answer:** Payload min **46 bytes**, max **1500 bytes** (standard MTU). A 10-byte order is **padded to 46**
so the total frame reaches the 64-byte minimum. Tiny messages don't save proportional wire time — another
reason micro-messaging is latency-inefficient and why co-lo bandwidth is rarely the binding constraint,
latency is.

**Q5.** Why do HFT shops prefer cut-through switches, and what do they give up?
**Answer:** Cut-through forwards as soon as it reads the **destination MAC** (~first 6 bytes), giving
single-digit-to-tens-of-nanosecond, frame-size-independent latency vs store-and-forward's buffer-whole-
frame-then-CRC delay. The tradeoff: a frame that later fails CRC may already be forwarded — but the
receiving NIC drops bad-CRC frames anyway, so the latency win dominates.

**Q6.** What does kernel bypass actually change at the network-access layer, mechanically?
**Answer:** The NIC's receive/transmit rings are mapped into the trading process's user memory. Instead of
the NIC raising an **IRQ** → kernel driver → softirq → copy into a socket buffer → `recv` syscall, the app
**busy-polls** the ring and reads the frame directly. It removes the interrupt (and its jitter), the kernel
copy, and the per-packet syscall — the dominant network-access overhead on the hot path.

---

## Indian HFT interview questions

**Q1 (Optiver — fundamentals + HFT).** *What's the difference between a MAC and an IP address, and why does
it matter when a packet is routed across the Internet?*
**Model answer:** A MAC is a 48-bit flat, link-local hardware address burned into the NIC; it's only
meaningful on one physical link and isn't routable. An IP is a hierarchical, globally routable logical
address. When a packet crosses the Internet, the IP source/destination stay constant end to end because
they name the ultimate endpoints, while the MAC pair is rewritten at every hop because each hop is a new
link with new link-local devices — the router re-frames for the next link. ARP is what maps the next-hop IP
to the MAC we actually put in the frame. In HFT we pin static ARP entries so that mapping never costs a
runtime round-trip.

**Q2 (Tower Research, Gurgaon — latency).** *Trace a market-data frame from the exchange's wire to your
strategy, and name every place you'd cut latency at this layer.*
**Model answer:** The feed arrives as a UDP-multicast Ethernet frame. Normally the NIC DMAs it to a ring,
raises an IRQ, the driver + NAPI softirq run, it's copied into a socket buffer, and `recvmmsg` returns it
to user space. Cuts: use a **kernel-bypass NIC** (Onload/exanic) mapping the ring into user space and
**busy-poll** it — no interrupt, no copy, no syscall; **pin** the polling thread to an isolated core so
nothing preempts it; put the box one **cut-through** switch hop from the feed with **layer-1 replication**;
use **fibre/direct-attach** not `10GBASE-T` copper to avoid PHY latency; **hardware-timestamp** at the NIC
to measure the real wire-to-wire number. On the extreme end, parse the feed in an **FPGA NIC** and fire the
order in silicon.

**Q3 (Graviton — switching).** *Explain cut-through vs store-and-forward and when each is right.*
**Model answer:** Store-and-forward buffers the whole frame and verifies CRC before forwarding — latency
grows with frame size and it never forwards a corrupt frame, good for general-purpose or lossy links.
Cut-through forwards as soon as the destination MAC is read, giving fixed, nanosecond-scale latency
regardless of frame size — the HFT choice — at the cost of possibly forwarding a frame that later fails CRC.
Since the destination NIC discards bad-CRC frames anyway, cut-through's latency win is almost always worth
it inside a trading datacenter.

**Q4 (Quadeye — mechanism).** *What is ARP, when does it happen, and why do HFT systems disable it on the
hot path?*
**Model answer:** ARP resolves an IPv4 address to a link-layer MAC: the sender broadcasts "who has this IP",
the owner unicasts its MAC, and the sender caches it. It happens on the first frame to an un-cached
neighbor (or the default gateway for off-subnet destinations) and on cache expiry. On the hot path a cache
miss injects a broadcast round-trip of latency and jitter at an unpredictable moment, so trading boxes
install **static ARP entries** for all gateways and peers — the mapping is always resident and the first
order of the session is as fast as the millionth.

---

## Key takeaways

- The network access layer moves **frames** across **one link/LAN**, converting bits ↔ signals (physical)
  and framing + MAC-addressing them (data link). Scope: "same local network" only.
- The **NIC** serializes/deserializes, frames, and CRC-checks; normally it DMAs + interrupts into the
  kernel — the exact path HFT replaces with **kernel bypass** (user-space ring polling) and **FPGA** NICs.
- **Ethernet frame** = 14B header (dst MAC, src MAC, EtherType) + 46–1500B payload + 4B CRC; MTU 1500,
  jumbo 9000. EtherType selects the upper layer (`0x0800` IPv4, `0x0806` ARP, `0x8100` VLAN).
- **MAC** = 48-bit flat, link-local, non-routable (OUI + device ID); **IP** = hierarchical, global,
  routable. MAC is rewritten per hop; IP is end-to-end constant.
- **ARP** maps IP→MAC on a link (broadcast request, unicast reply, cached). HFT pins **static ARP** to kill
  runtime resolution jitter.
- **Switches** learn src MAC→port into a CAM table and forward by dst MAC (flood if unknown). HFT uses
  **cut-through** switches (nanosecond, frame-size-independent) with layer-1 replication.
- Modern LANs are **star**; mesh gives best fault tolerance at high cost.

**Next:** [03 — The internet layer](03-internet-layer.md)
