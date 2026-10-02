# Module 01 — The TCP/IP model

Every market-data tick and every order you send is, at the bottom, a pattern of voltages on a wire that
some software had to wrap in headers, hand to a NIC, and unwrap on the far side. This guide is about that
journey. The networking stack is built as **layers**, each solving one problem and handing a clean
service to the layer above — and in HFT the entire game is understanding that stack well enough to know
exactly which layers you can collapse, bypass, or offload to shave microseconds off tick-to-trade.

This networks-guide has six modules, and they climb the stack from the bottom:

1. **The TCP/IP model** (this module) — the four layers, how they nest, and where latency hides.
2. **The network access layer** — Ethernet, MAC addresses, ARP, switching; where kernel-bypass NICs live.
3. **The internet layer** — IP addressing, routing, multicast; how a packet crosses networks.
4. **The transport layer** — ports, sockets, multiplexing; TCP vs UDP as a design choice.
5. **The TCP protocol** — handshake, reliability, flow/congestion control, `TCP_NODELAY`.
6. **The UDP protocol** — datagrams and multicast, the backbone of every exchange feed.

It's the sibling of the [caos-guide](../caos-guide/00-index.md) (CPU/OS internals) and the
[cpp-guide](../cpp-guide); where the OS actually touches the wire — NIC interrupts, DMA, the socket
I/O path — those modules own the detail and this one links across to them.

---

## 1. Why layers at all

A network has to solve a stack of unrelated problems at once: how to turn bits into light on a fibre, how
to find a machine on the other side of the planet, how to make an unreliable path look reliable, how to
get the bytes to the *right program* on the destination box. Cramming all of that into one blob of code
would be unmaintainable and un-evolvable.

The fix is **separation of concerns**: each layer solves one problem, consumes the service of the layer
below as a black box, and exports a clean service upward. You can swap Wi-Fi for fibre at the bottom
without TCP noticing; you can swap TCP for UDP at the transport layer without IP noticing. That
independence is why the Internet could evolve for 40 years without a rewrite.

The cost — and this is the HFT-relevant half — is that **every layer adds a header, a copy, and
processing**. Layering buys flexibility by spending latency. The whole discipline of low-latency
networking is deciding, layer by layer, where that spend is worth it and where you tear it out.

---

## 2. The four layers of TCP/IP

TCP/IP is the model the real Internet runs on. Four layers, top to bottom:

```
 ┌─────────────────────────────────────────────────────────────┐
 │ 4. APPLICATION    HTTP, DNS, FTP, SMTP — and your own         │  "what the bytes mean"
 │                   binary market-data / order protocols        │
 ├─────────────────────────────────────────────────────────────┤
 │ 3. TRANSPORT      TCP (reliable, ordered) / UDP (fast,        │  "to the right PROGRAM,
 │                   connectionless) — ports, end-to-end         │   reliably or not"
 ├─────────────────────────────────────────────────────────────┤
 │ 2. INTERNET       IP — logical addressing + routing across    │  "to the right MACHINE,
 │                   networks, hop by hop                        │   anywhere"
 ├─────────────────────────────────────────────────────────────┤
 │ 1. NETWORK ACCESS Ethernet, Wi-Fi, fibre — frames on one      │  "across one physical
 │   (link + phys)   physical link, MAC addressing, signaling    │   link"
 └─────────────────────────────────────────────────────────────┘
```

Read it as a chain of promises, each building on the one below:

| Layer | Job | Addresses by | Reliable? | Key protocols |
|-------|-----|--------------|-----------|---------------|
| Application | Give user programs a network service / define the wire format | — | depends | HTTP(S), DNS, FTP, SMTP, **custom binary** |
| Transport | Get data to the right *process*; optionally make it reliable | **port** | TCP yes / UDP no | TCP, UDP |
| Internet | Get a packet to the right *machine* across many networks | **IP address** | no (best-effort) | IP, ICMP |
| Network access | Move a frame across *one* physical link | **MAC address** | no | Ethernet, Wi-Fi, 802.11 |

The upward progression of "who are we addressing" is the clean way to remember it: **MAC** gets you across
one link, **IP** gets you to any machine on the planet, **port** gets you to the right program on that
machine, and the **application protocol** tells that program what the bytes mean.

### The lower-three in one sentence each

- **Network access** converts bytes to signals (voltage/light/radio) and moves a *frame* between two
  devices on the same local network. Covered in [Module 02](02-network-access-layer.md).
- **Internet** gives every device a globally meaningful **IP address** and forwards *packets* hop by hop
  between different networks — this is what lets two machines that aren't on the same LAN talk at all.
  Covered in [Module 03](03-internet-layer.md).
- **Transport** adds **ports** so the data reaches the right *application* (not just the right machine),
  and — if you chose TCP — detects loss, retransmits, orders, and flow-controls. Covered in
  [Module 04](04-transport-layer.md), with TCP and UDP each getting their own module (05, 06).

### The application layer and "roll your own"

The application layer is where the bytes finally mean something. HTTP, DNS, SMTP are the public examples.
But the sentence from the source material that matters most for us: **"A lot of real-world low-latency
programming requires you to create your own protocols."** Exchanges don't send you JSON over HTTP — they
send tight **binary** messages (fixed-width fields, no parsing, no text) over UDP multicast, because every
byte and every parse step is latency. Your feed handler ([infra Module 02](../hft-infrastructure-guide/02-feed-handler.md))
*is* an application-layer protocol implementation. Designing that wire format — fixed offsets, little vs
big endian, sequence numbers for gap detection — is application-layer engineering.

---

## 3. TCP/IP vs the OSI 7-layer model

OSI is the older, more granular academic reference. You'll be asked to map it to TCP/IP:

```
   OSI (7 layers)                  TCP/IP (4 layers)
 ┌──────────────────┐
 │ 7. Application    │ ┐
 │ 6. Presentation   │ ├──────────▶  Application
 │ 5. Session        │ ┘
 ├──────────────────┤
 │ 4. Transport      │ ───────────▶  Transport
 ├──────────────────┤
 │ 3. Network        │ ───────────▶  Internet
 ├──────────────────┤
 │ 2. Data Link      │ ┐
 │ 1. Physical       │ ┴──────────▶  Network Access
 └──────────────────┘
```

The three collapses to remember: TCP/IP's **Application** = OSI's Application + Presentation + Session;
TCP/IP's **Network Access** = OSI's Data Link + Physical. Transport and Network/Internet map 1:1. OSI is
useful vocabulary ("that's a layer-2 switch", "layer-3 routing", "an L7 load balancer") even though the
Internet actually runs the 4-layer model.

---

## 4. Encapsulation and decapsulation

As data goes *down* the stack, each layer wraps it in that layer's header (and the link layer adds a
trailer too). This nesting is **encapsulation**. On the receiver, each layer strips its own header on the
way *up* — **decapsulation**.

```
 SEND (down the stack)                    the PDU name at each layer
 ─────────────────────                    ─────────────────────────
   Application data  (your GET / your order message)      "data"
          │  + Application semantics
          ▼
   [ TCP hdr | data ]                                     "segment"   (UDP: "datagram")
          │  + source/dest PORT, seq/ack
          ▼
   [ IP hdr | TCP hdr | data ]                            "packet"
          │  + source/dest IP, TTL, protocol
          ▼
   [ Eth hdr | IP hdr | TCP hdr | data | Eth FCS ]        "frame"
          │  + source/dest MAC, EtherType, CRC trailer
          ▼
        bits on the wire                                  "bits / symbols"
```

Each header is literally "the function arguments the next hop needs to see." The source material frames it
well: you can't just send `sendData(src, dst, data, len)` as a bare call across the world — the far side
and every router in between must be able to *read* src/dst, so you serialize those arguments into bytes at
the front of the message. That's all a header is: the layer's parameters, on the wire.

**Decapsulation on arrival** runs the diagram in reverse: the NIC/link layer checks the frame's dest MAC
and CRC and strips the Ethernet header; IP checks the dest IP and reads the `protocol` field to know it's
TCP; TCP reads the dest port to find the socket; the application reads its own header to know what the
payload is. Each layer peels its own wrapper and hands the core upward.

### The nesting is also the cost

Count the overhead on a tiny message: Ethernet (14B + 4B FCS) + IPv4 (20B) + TCP (20B) = **~58 bytes of
header** before a single byte of payload. For a 16-byte order that's ~78% overhead. More importantly,
every header is a *parse* and the payload is often a *copy* across the kernel/user boundary. That is the
raw material HFT optimizes: fewer layers (UDP not TCP → drop 20 header bytes and the whole reliability
state machine), fewer copies (kernel bypass → DMA straight to user memory), fewer parses (fixed-offset
binary formats).

---

## 5. The end-to-end journey of a packet

Putting it together — an application on host A sends to an application on host B across the Internet:

```
 HOST A                         ROUTERS (hop by hop)                      HOST B
 app write()                                                             app read()
   │ encapsulate down                                                      ▲ decapsulate up
   ▼                                                                       │
  TCP seg ─▶ IP pkt ─▶ Eth frame ─▶ [wire] ─▶ R1 ─▶ R2 ─▶ … ─▶ [wire] ─▶ Eth ─▶ IP ─▶ TCP ─▶ data
                                      │        │                            check   check  find
                                   each router reads only the IP header,    MAC/CRC  dstIP  socket
                                   decrements TTL, looks up the next hop,
                                   re-frames for the outgoing link, forwards
```

The key insight for interviews: **routers in the middle only touch layers 1–3.** A router reads the IP
header to decide the next hop and rewrites the layer-2 (Ethernet) framing for each link, but it never
looks at your TCP header or payload — transport is *end-to-end*, meaningful only to the two endpoints. The
MAC/Ethernet framing is *link-local* and is rebuilt at every hop; the IP addresses stay constant end to
end while the MAC addresses change every hop. (Why the IP survives but the MAC doesn't is the heart of
[Module 02](02-network-access-layer.md) and [Module 03](03-internet-layer.md).)

The OS side of this — the `write()` syscall trapping into the kernel, the data copied into socket buffers,
the NIC DMA, the receive interrupt on host B — is the subject of [caos Module 08 (syscalls)](../caos-guide/08-system-calls.md),
[Module 09 (interrupts)](../caos-guide/09-interrupts-and-exceptions.md), and
[Module 12 (the I/O path)](../caos-guide/12-allocators-and-io.md). The networking stack rides on top of all of it.

---

## 6. Where the microseconds go — and what HFT bypasses

Layering tells you exactly where latency accumulates, which is why it's the map for optimization:

| Layer | Normal-path cost | HFT move |
|-------|------------------|----------|
| Application | parse/format, business logic | fixed-offset **binary** formats, no allocation, no text parsing |
| Transport | TCP state machine, ACKs, congestion control, Nagle buffering | prefer **UDP**; if TCP, set `TCP_NODELAY`; often **kernel-bypass TCP** stacks |
| Internet | routing lookups, per-hop store/forward | **co-location** — be one subnet from the matching engine, minimize hops |
| Network access | kernel driver, interrupt, copy to user | **kernel-bypass NIC** (Solarflare/Onload, Exablaze), **DPDK**, busy-poll the ring, FPGA offload |

The headline technique is **kernel bypass**: the standard path routes a received frame through NIC
interrupt → driver → IP → TCP/UDP → socket buffer → copy to user space, costing single-digit-to-tens of
microseconds and, worse, unpredictable **jitter** from interrupts and scheduling. Kernel-bypass stacks
(Solarflare Onload, DPDK, exanic) map the NIC's packet rings directly into user memory and let the trading
process **busy-poll** them — the frame lands in your buffer with no syscall, no interrupt, no kernel copy.
You're still speaking Ethernet/IP/UDP, but the layer *implementations* now run in your process at user
level instead of in the kernel. Understanding the layer model is precisely what lets you reason about what
such a stack is and isn't doing for you.

The deepest version — **FPGA NICs** — implement the parsing of layers 1–4 *in hardware* so a market-data
message can trigger an order in tens of nanoseconds without a CPU in the loop at all. That's still the same
four layers; they've just been pushed into silicon.

---

## Common pitfalls / misconceptions

- **"OSI is what the Internet uses."** No — the Internet runs the **4-layer TCP/IP** model. OSI is a
  reference/vocabulary model. Interviewers like watching you keep them straight.
- **"A router looks at my TCP/port."** A plain router operates at layer 3 (IP) and rewrites layer 2 per
  hop; it does *not* read your transport header or payload. (NAT boxes, firewalls, and L4/L7 load balancers
  *do* peek higher — but a vanilla router doesn't.)
- **"Encapsulation means encryption."** No. Encapsulation = wrapping data in a layer's header. Encryption
  (TLS) is a separate application-layer concern.
- **"The IP address changes as the packet is routed."** The *IP* src/dst stay constant end to end (barring
  NAT); it's the *MAC* addresses that are rewritten at every hop. Mixing these up is the classic giveaway.
- **"More layers = cleaner, so always good."** Layering trades latency for flexibility. In HFT that trade
  is often reversed on purpose — you bypass or collapse layers to win microseconds.

---

## Quiz — tough problems

**Q1.** A packet travels A → R1 → R2 → B. Which addresses in the headers change along the way, and which
stay constant?
**Answer:** The **IP** source/destination stay constant end to end (no NAT). The **MAC** source/destination
are rewritten at *every* hop — each link's framing is between that link's two endpoints (A→R1, R1→R2,
R2→B). The TTL in the IP header decrements by one per router. Payload and TCP header are untouched by the
routers.

**Q2.** You send a 16-byte order over TCP/IPv4/Ethernet. How many bytes actually hit the wire, roughly, and
what's the overhead ratio?
**Answer:** ~14B Ethernet header + 4B FCS + 20B IPv4 + 20B TCP = ~58B of headers around 16B of payload, so
~74 bytes on the wire — **~78% overhead**. Plus the minimum Ethernet frame is 64B, so a tiny order is
padded anyway. This is one reason tiny, chatty messaging is latency-hostile and why binary batching and UDP
(saves the 20B TCP header + state machine) matter.

**Q3.** Which layers does a store-and-forward router process, and which does it ignore?
**Answer:** It processes layers 1–3: receives the frame (L1/L2), checks/strips Ethernet, reads the **IP**
header (L3) to pick the next hop, decrements TTL, re-frames for the outgoing link, forwards. It does **not**
process layer 4 (TCP/UDP) or the application payload — transport is end-to-end.

**Q4.** Map the OSI "Session" and "Presentation" layers onto TCP/IP.
**Answer:** Both fold into TCP/IP's single **Application** layer (along with OSI's Application layer). TCP/IP
doesn't give session/presentation their own layer — those concerns (serialization, encryption, session
state) live inside the application protocol.

**Q5.** Give the PDU name at each layer for the same piece of data going down the stack.
**Answer:** Application → **data/message**; Transport (TCP) → **segment** (UDP → **datagram**); Internet →
**packet**; Network access → **frame**; Physical → **bits/symbols**. Interviewers use the right noun as a
shibboleth: say "frame" for layer 2, "packet" for layer 3, "segment" for TCP.

**Q6.** Kernel bypass is often described as "removing the kernel from the data path." Which layers' *logic*
still runs — just somewhere else?
**Answer:** All of them still run — Ethernet framing, IP, and UDP/TCP are still parsed/built. Kernel bypass
moves those layer *implementations* out of the kernel and into the user-space process (or into the NIC's
silicon on an FPGA), and lets the app **busy-poll** the NIC rings instead of taking an interrupt + kernel
copy. The protocol stack doesn't disappear; its code just stops living in ring 0.

---

## Indian HFT interview questions

**Q1 (Optiver — fundamentals).** *Walk me through what happens, layer by layer, when your strategy sends an
order to the exchange.*
**Model answer:** The strategy produces an order message (application layer) in a fixed binary format. The
transport layer wraps it — TCP if the gateway requires a reliable ordered session, with `TCP_NODELAY` so
it isn't buffered by Nagle — adding source/dest ports. IP wraps that with our co-lo source address and the
gateway's destination, picking the route (one hop in co-lo). Ethernet frames it with our NIC's MAC and the
first-hop switch/router's MAC, appends the FCS. The NIC serializes it to the wire. On a kernel-bypass
setup the TCP/UDP + IP + Ethernet encapsulation runs in our user process and we DMA straight to the NIC —
no syscall, no kernel copy — which is where most of the saved microseconds come from.

**Q2 (Tower Research, Gurgaon — model clarity).** *Why does the networking stack use layers, and what does
that cost you in HFT?*
**Model answer:** Layers give separation of concerns and interchangeable implementations — you can change
the physical medium without touching TCP, swap TCP for UDP without touching IP. The cost is that each layer
adds a header, often a copy, and processing, and the standard implementations live in the kernel behind
syscalls and interrupts. In HFT that cost is the enemy: we co-locate to kill routing hops, use UDP
multicast to drop TCP's header and state machine for market data, and kernel-bypass the network-access and
transport layers so the per-layer processing happens in user space with busy-polling instead of in the
kernel with interrupts and jitter.

**Q3 (Graviton — precision).** *A packet crosses three routers. What changes in its headers at each hop and
what stays the same, and why?*
**Model answer:** IP source/destination stay constant end to end because IP addressing is logical and
global — it identifies the ultimate endpoints. The TTL decrements by one per router (loop protection; hits
zero → ICMP time-exceeded, which is how traceroute works). The layer-2 Ethernet source/destination MACs are
rewritten at every hop because MAC addressing is link-local — each physical link's frame is addressed
between that link's two devices, so the router re-frames for the next link. Transport header and payload
are untouched; routers don't operate above layer 3.

**Q4 (Quadeye — design judgment).** *The exchange sends market data as custom binary over UDP, not JSON
over HTTP. Explain the layering choices.*
**Model answer:** Application layer: a custom fixed-offset binary format, so parsing is pointer arithmetic,
not text tokenizing, and there's no allocation. Transport layer: UDP, because market data is a one-to-many
broadcast where retransmission latency is worse than loss — you'd rather detect a gap via sequence numbers
and recover from a secondary feed than wait for a TCP retransmit; UDP also drops TCP's 20-byte header and
its congestion-control buffering. It's typically **multicast** so one send reaches every subscriber. HTTP/
JSON/TCP would add handshakes, head-of-line blocking, text parsing, and per-client unicast — all fatal for
a low-latency fan-out feed.

---

## Key takeaways

- The Internet runs the **4-layer TCP/IP model**: Network Access (frames, MAC) → Internet (packets, IP) →
  Transport (segments, ports) → Application (your protocol). Addressing climbs MAC → IP → port → app.
- **OSI's 7 layers** map onto it: App = App+Presentation+Session; Network Access = Data Link + Physical;
  Transport and Network/Internet are 1:1.
- **Encapsulation** adds a header per layer going down (data → segment → packet → frame → bits);
  **decapsulation** strips them going up. A header is just that layer's parameters serialized onto the wire.
- Routers operate at **layers 1–3**: IP stays constant end to end, MAC is rewritten per hop, TTL decrements
  per hop. Transport is **end-to-end** — only the two endpoints read it.
- Layering trades latency for flexibility. HFT reverses the trade deliberately: **co-locate** (fewer hops),
  **UDP multicast** (fewer transport bytes/state), **kernel bypass / FPGA** (fewer copies, no interrupts),
  **binary formats** (fewer parses).

**Next:** [02 — The network access layer](02-network-access-layer.md)
