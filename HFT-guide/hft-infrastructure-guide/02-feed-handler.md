# Module 02 — The feed handler in C++ (deep dive)

Module 01 drew the pipeline and gave the feed handler one paragraph. This module spends a whole
chapter on it, because in an HFT infrastructure interview the feed handler is the box you'll be
grilled on hardest — it's where *networking, binary parsing, lock-free data structures, and hard
correctness under packet loss* all meet. It's also the box most beginners hand-wave ("I read the
socket and update the book"), so getting it right is a strong differentiator.

We go **exchange-side first** — concretely, what NSE/BSE actually put on the wire — then build the
handler up in layers: receive → parse → sequence → recover → normalize → hand off. Every layer has a
code sketch, the tradeoffs, and the classic failure modes.

> Prerequisites from earlier modules: struct layout & alignment (15), atomics & lock-free ring buffers
> (16), zero-cost abstractions (17), `std::span`/`<bit>` (18). We'll lean on all of them.

---

## 1. What a feed handler *is* (one sentence)

> A feed handler turns a **fast, lossy, exchange-specific byte stream** on the wire into a
> **reliable, normalized, in-order stream of typed messages** that the rest of your system can trust.

Everything it does serves that sentence. "Fast" → no allocation, no locks, minimal copies on the hot
path. "Lossy" → sequence-gap detection and recovery. "Exchange-specific" → parsing. "Reliable,
in-order, normalized, trust" → the output contract to the order-book builder.

Its responsibilities, in order of the data's journey:

1. **Receive** raw UDP multicast packets off the NIC (ideally via kernel bypass).
2. **Deframe** — a packet may contain a header + *several* messages back-to-back.
3. **Parse** each message from the exchange's binary wire format into a struct.
4. **Sequence** — check the sequence number; detect gaps, duplicates, reordering.
5. **Recover** — on a gap, request retransmission or resync from a snapshot; buffer meanwhile.
6. **Arbitrate** — most exchanges send two identical streams (A/B lines); pick whichever arrives
   first and drop the duplicate.
7. **Normalize** — convert to your internal message type (venue-agnostic).
8. **Hand off** — publish to the order-book builder / strategy, usually across a lock-free ring buffer.

---

## 2. The exchange side, concretely (NSE & BSE)

You cannot design a good feed handler without knowing the shape of what you're parsing. Exact wire
specs are licensed documents (you get them under an exchange membership/NDA), so the layouts below are
**representative** — realistic in structure and field types, close enough to reason and code against,
but you'd bind to the real spec on the job.

### 2.1 Two different worlds: NSE vs BSE

- **NSE** runs its **own** trading platform. Market data comes as UDP multicast; the premium
  order-level feed is **TBT (Tick-By-Tick)**. Order entry is NSE's native binary protocol (**NNF /
  "NEAT" native**), with FIX available for some flows. NSE's older broadcast feeds used **LZO
  compression** on the payload; the low-latency TBT feed is delivered uncompressed for speed.
- **BSE** runs on **Deutsche Börse's T7** platform. So its interfaces are the T7 family:
  **EOBI (Enhanced Order Book Interface)** = the tick-by-tick order-level multicast feed, **EMDI**
  = netted depth, and **ETI (Enhanced Trading Interface)** = order entry. If an interviewer at a BSE-
  heavy desk hears you say "BSE is T7, so EOBI for TBT and ETI for orders," that's an instant depth
  signal.

The mechanics below (multicast, sequence numbers, A/B lines, snapshots) are **common to both** — only
the exact field names and message IDs differ.

### 2.2 How the data physically arrives

- **Transport: UDP multicast** inside the colo. You don't get a stream you `connect()` to; you
  **join a multicast group** (an address like `239.x.x.x` + a port) and packets are pushed to you.
- **Segments have separate groups.** Cash, F&O, currency, and even *partitions within a segment*
  (instruments sharded by a hash) each broadcast on their own multicast group. A busy handler joins
  *many* groups. This sharding is deliberate — it lets a firm subscribe to only the instruments it
  trades and spreads load across NICs/cores.
- **A/B lines (redundancy):** the exchange sends **two identical copies** of every feed on two
  different multicast groups, often over physically separate network paths ("Line A" and "Line B").
  A packet dropped on A is very likely present on B. Your handler listens to both and arbitrates
  (§7).
- **Recovery channels:** separate from the live feed there's a **retransmission service** (usually
  TCP request/response: "resend me sequences 5000–5050") and a periodic **snapshot feed** (the full
  current book state, so a late joiner or a badly-desynced handler can rebuild from scratch).
- **Heartbeats:** when nothing is trading, the exchange still sends periodic heartbeat messages so
  you can tell "quiet market" apart from "my feed died."

```
        EXCHANGE                                     YOUR COLO RACK
   ┌──────────────────┐   Line A (239.1.1.1:15001)   ┌──────────────────────┐
   │  Market-data     │ ───────────────────────────▶ │  NIC 0 ─▶ feed handler│
   │  publisher       │   Line B (239.1.2.1:15001)   │  NIC 1 ─▶   (A/B      │
   │                  │ ───────────────────────────▶ │           arbitrate) │
   │  Retrans (TCP)   │ ◀───── "resend 5000-5050" ─── │                      │
   │  Snapshot feed   │ ───────────────────────────▶ │  (resync on big gap) │
   └──────────────────┘                              └──────────────────────┘
```

### 2.3 The packet framing (representative)

One UDP datagram is not one message. It's a **packet header** followed by one or more **messages**
packed back-to-back. This matters: your parser loops *within* a packet.

```
UDP payload:
┌───────────────── packet header ─────────────────┬── msg 1 ──┬── msg 2 ──┬─ ... ─┐
│ msgCount | firstSeqNum | sendTime | ...          │  (varlen) │  (varlen) │       │
└──────────────────────────────────────────────────┴───────────┴───────────┴───────┘
        the header tells you how many messages follow and the seq of the first one
```

A representative packet header and a couple of message bodies as C++ structs (network byte order,
tightly packed):

```cpp
#include <cstdint>

#pragma pack(push, 1)              // no padding — the wire has none (see Module 15 on alignment)

struct PacketHeader {
    uint16_t msg_count;            // how many messages follow in this datagram
    uint64_t first_seq;            // sequence number of the first message
    uint64_t send_time_ns;         // exchange send timestamp (epoch ns)
};

// Each message starts with a small common header so the parser can dispatch on type & length.
struct MsgHeader {
    uint16_t length;               // total bytes of this message (incl. this header)
    uint16_t type;                 // message type id (e.g. 1=NewOrder, 2=Modify, 3=Cancel, 4=Trade)
};

struct NewOrderMsg {               // TBT "add order"
    MsgHeader hdr;
    uint64_t  order_id;
    uint32_t  instrument_id;       // token / symbol id (NOT the text symbol — a numeric id)
    char      side;                // 'B' or 'S'
    int64_t   price;               // fixed-point: paise, or price * 10^k. NEVER a float on the wire
    uint32_t  quantity;
    uint64_t  timestamp_ns;
};

struct TradeMsg {                  // a match happened
    MsgHeader hdr;
    uint64_t  buy_order_id;
    uint64_t  sell_order_id;
    uint32_t  instrument_id;
    int64_t   price;
    uint32_t  quantity;
    uint64_t  timestamp_ns;
};

#pragma pack(pop)
```

Five concrete facts a beginner must internalize from this:

1. **Prices are integers on the wire**, in the smallest tick unit (e.g. paise), never floats.
   Floats aren't bit-exact across machines and break equality/ordering. You keep them as `int64_t`
   internally too (Module 5/15).
2. **Instruments are numeric ids** ("tokens"), not `"RELIANCE"`. You map id→symbol via a
   reference-data file loaded at startup.
3. **Everything is fixed-width and packed** — hence `#pragma pack(1)`. The wire has no padding; if
   your struct has padding your offsets are wrong.
4. **Byte order matters.** Wire formats are usually **big-endian** (network order) or a documented
   little-endian; your x86 box is little-endian. You must byte-swap on read (§4.2).
5. **Messages are variable length** and self-describing via `length` — you advance by `hdr.length`,
   never by `sizeof(SomeStruct)`, so you survive spec versions that add trailing fields.

---

## 3. The receive path

### 3.1 Joining a multicast group (the ordinary kernel-socket way)

Start here to understand the mechanics; §3.3 replaces it for production speed.

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <cstring>
#include <stdexcept>

int make_multicast_socket(const char* group, uint16_t port, const char* local_nic_ip) {
    int fd = ::socket(AF_INET, SOCK_DGRAM, 0);
    if (fd < 0) throw std::runtime_error("socket");

    // Allow multiple processes/sockets to bind the same port (many handlers, one group).
    int one = 1;
    ::setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));

    // Bind to the multicast port on all local addresses.
    sockaddr_in addr{};
    addr.sin_family      = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port        = htons(port);
    if (::bind(fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr)) < 0)
        throw std::runtime_error("bind");

    // Join the group *on a specific NIC* (imr_interface) — critical in a multi-NIC colo box,
    // where Line A and Line B come in on different cards.
    ip_mreq mreq{};
    mreq.imr_multiaddr.s_addr = ::inet_addr(group);
    mreq.imr_interface.s_addr = ::inet_addr(local_nic_ip);
    if (::setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq)) < 0)
        throw std::runtime_error("join group");

    // A big receive buffer absorbs microbursts so the kernel doesn't drop before you read.
    int rcvbuf = 64 * 1024 * 1024;
    ::setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));
    return fd;
}
```

Note the three things beginners miss: **`SO_REUSEADDR`** (so several handlers can share a group
port), **`imr_interface`** (join on the *right* NIC — A vs B live on different cards), and a **large
`SO_RCVBUF`** (a market-open microburst can deliver tens of thousands of packets in a millisecond; if
the socket buffer is small the *kernel* silently drops them and you get a sequence gap you can't blame
the network for).

### 3.2 Reading packets — batch, don't read one at a time

`recvmmsg` pulls *many* datagrams in one syscall. Syscalls cost ~microseconds; at market open you
can't afford one per packet.

```cpp
#include <sys/socket.h>

constexpr int kBatch = 64;
constexpr int kMTU   = 2048;

struct RxBuffers {
    mmsghdr msgs[kBatch];
    iovec   iov[kBatch];
    char    data[kBatch][kMTU];
    RxBuffers() {
        std::memset(msgs, 0, sizeof(msgs));
        for (int i = 0; i < kBatch; ++i) {
            iov[i].iov_base          = data[i];
            iov[i].iov_len           = kMTU;
            msgs[i].msg_hdr.msg_iov  = &iov[i];
            msgs[i].msg_hdr.msg_iovlen = 1;
        }
    }
};

void receive_loop(int fd, RxBuffers& rx, FeedHandler& handler) {
    while (true) {
        int n = ::recvmmsg(fd, rx.msgs, kBatch, MSG_DONTWAIT, nullptr);  // non-blocking, busy-poll
        if (n <= 0) continue;                                            // nothing right now — spin
        for (int i = 0; i < n; ++i) {
            std::span<const std::byte> pkt{
                reinterpret_cast<const std::byte*>(rx.data[i]),
                rx.msgs[i].msg_len
            };
            handler.on_packet(pkt);   // one datagram → deframe + parse inside
        }
    }
}
```

`MSG_DONTWAIT` + a spin loop is **busy-polling**: the thread never sleeps, so there's no wake-up
latency when a packet lands. You pin this thread to an isolated core (Module 15/16) and it burns 100%
CPU on purpose. That's the HFT trade: a whole core spent to save the ~µs it takes the OS to wake a
sleeping thread.

### 3.3 Kernel bypass (the production answer)

The kernel socket path copies each packet kernel→user and traverses the network stack. HFT feed
handlers skip it with **kernel bypass**: **Solarflare/Onload** (transparently reroutes normal socket
calls into userspace — often a drop-in), or **DPDK** (you own the NIC and poll RX descriptor rings
directly, zero-copy). The *logic* below is identical; only the "where do the bytes come from" layer
changes. Interview line: *"the socket code is for correctness and dev; in prod the receive layer is
Onload or DPDK so packets land in userspace with no kernel copy and no context switch."*

---

## 4. Parsing (deframing + decoding)

### 4.1 The deframe loop

One datagram → header → walk `msg_count` messages, advancing by each message's own `length`:

```cpp
void FeedHandler::on_packet(std::span<const std::byte> pkt) {
    if (pkt.size() < sizeof(PacketHeader)) { stats_.runt_packets++; return; }

    PacketHeader ph;
    std::memcpy(&ph, pkt.data(), sizeof(ph));      // memcpy avoids unaligned-access UB (§4.3)
    ph.first_seq     = bswap(ph.first_seq);        // fix endianness once, up front
    ph.msg_count     = bswap(ph.msg_count);
    ph.send_time_ns  = bswap(ph.send_time_ns);

    size_t off = sizeof(PacketHeader);
    uint64_t seq = ph.first_seq;

    for (uint16_t m = 0; m < ph.msg_count; ++m) {
        if (off + sizeof(MsgHeader) > pkt.size()) { stats_.truncated++; return; }

        MsgHeader mh;
        std::memcpy(&mh, pkt.data() + off, sizeof(mh));
        mh.length = bswap(mh.length);
        mh.type   = bswap(mh.type);

        if (mh.length < sizeof(MsgHeader) || off + mh.length > pkt.size()) {
            stats_.bad_length++; return;           // corrupt/inconsistent — bail on the packet
        }

        // sequence check happens per-message (§5), then dispatch:
        if (sequence_ok(seq)) {
            std::span<const std::byte> body{pkt.data() + off, mh.length};
            dispatch(mh.type, body);
        }
        off += mh.length;                          // advance by the wire length, not sizeof(T)
        ++seq;
    }
}
```

### 4.2 Endianness

Wire is (say) big-endian; x86 is little-endian. C++20 gives you `std::byteswap` (`<bit>`, Module 18):

```cpp
#include <bit>
template <class T> constexpr T bswap(T v) {
    if constexpr (std::endian::native == std::endian::big) return v;   // already matches
    else return std::byteswap(v);
}
```

Forget this and every multi-byte field is garbage — a price of `100` becomes billions. This is one of
the most common "why is my parser producing nonsense" bugs.

### 4.3 Why `memcpy` and not a reinterpret_cast overlay?

Tempting: `auto* m = reinterpret_cast<const NewOrderMsg*>(ptr);`. Two problems: (1) the buffer offset
may be **unaligned** for an `int64_t`, which is undefined behavior on read and a real fault on some
architectures; (2) it's a strict-aliasing violation. `std::memcpy` into a properly-aligned local is
the correct, portable, and (with optimization) *free* idiom — the compiler turns a small fixed-size
`memcpy` into a single load. Beginners fear "memcpy is slow"; here it compiles away.

### 4.4 Dispatch

`type` → the right decode. A `switch` on a small dense integer becomes a jump table — fast and
branch-predictable:

```cpp
void FeedHandler::dispatch(uint16_t type, std::span<const std::byte> body) {
    switch (type) {
        case MSG_NEW_ORDER: on_new_order(decode<NewOrderMsg>(body)); break;
        case MSG_MODIFY:    on_modify   (decode<ModifyMsg>  (body)); break;
        case MSG_CANCEL:    on_cancel   (decode<CancelMsg>  (body)); break;
        case MSG_TRADE:     on_trade    (decode<TradeMsg>   (body)); break;
        case MSG_HEARTBEAT: on_heartbeat();                          break;
        default:            stats_.unknown_type++;                   break;  // forward-compat: skip
    }
}
```

Skipping unknown types (rather than crashing) is how you survive an exchange adding a message type in
a spec bump without redeploying at 3 a.m.

---

## 5. Sequencing — the heart of correctness

The feed is lossy and unordered. The sequence number is your only truth about *"did I see everything,
in order?"* Track the next sequence you expect, and classify each arrival:

```cpp
class Sequencer {
    uint64_t expected_ = 0;
    bool     started_  = false;
public:
    enum class Verdict { Process, Duplicate, GapAhead, Resync };

    Verdict classify(uint64_t seq) {
        if (!started_) { expected_ = seq; started_ = true; return Verdict::Process; }
        if (seq == expected_)       { ++expected_;      return Verdict::Process;   } // in order
        if (seq <  expected_)       {                   return Verdict::Duplicate; } // old (A/B dup or retrans)
        /* seq > expected_ */       {                   return Verdict::GapAhead;  } // we missed some
    }
    uint64_t expected() const { return expected_; }
    void resync_to(uint64_t seq) { expected_ = seq; started_ = true; }
};
```

The four cases and what each means:

| Verdict     | Condition           | Meaning                                   | Action                                            |
|-------------|---------------------|-------------------------------------------|---------------------------------------------------|
| `Process`   | `seq == expected`   | Exactly the next message                  | Decode, hand off, `expected++`                     |
| `Duplicate` | `seq <  expected`   | Already saw it (B line, or a retransmit)  | Drop silently (this is *normal*, not an error)     |
| `GapAhead`  | `seq >  expected`   | We missed `[expected, seq)` messages      | Enter recovery: buffer new msgs, request the gap   |
| `Resync`    | gap too big / stale | Recovery can't catch up in time           | Abandon incremental; rebuild from a snapshot       |

The subtlety that separates good from great: on `GapAhead` you do **not** just skip to `seq`. The
missed messages might be a `NewOrder` you now need, or a `Cancel` for an order already in your book.
Applying `seq` before the missing ones corrupts the book. So on a gap you **stop applying to the book,
buffer incoming messages, and recover the hole** — see §6.

---

## 6. Recovery — what to do on a gap

Two mechanisms, chosen by gap size:

**A. Retransmission (small gap).** Ask the exchange's retrans service to resend `[expected, seq)`.
Meanwhile buffer live messages (which have seq ≥ `seq`) so you can apply everything in order once the
hole is filled.

```cpp
void FeedHandler::on_gap(uint64_t have_expected, uint64_t got_seq,
                         std::span<const std::byte> current_msg) {
    state_ = State::Recovering;
    mark_book_stale();                                  // §8 — strategy must NOT trade now
    buffer_.push(got_seq, current_msg);                 // hold the future msg
    retrans_.request(instrument_group_, have_expected, got_seq - 1);  // "resend the hole"
    stats_.gaps++;
}

void FeedHandler::on_retrans_message(uint64_t seq, std::span<const std::byte> body) {
    if (seq != seq_.expected()) return;                 // out of order retrans arrivals get dropped
    apply_to_book(body);
    seq_.resync_to(seq + 1);
    drain_buffer_if_contiguous();                       // now replay buffered live msgs in order
    if (buffer_.empty()) { state_ = State::Live; mark_book_fresh(); }
}
```

**B. Snapshot resync (large gap or startup).** If the gap is huge (you were down, or lost thousands of
messages in a burst), replaying is hopeless. Wait for the next periodic **snapshot** (the full current
book), load it, and set `expected` to the snapshot's sequence number, then resume incrementals from
there. This is also exactly how you **cold-start** mid-session: you can't replay the whole day, so you
snapshot-then-follow.

```
   normal ──seq gap──▶ Recovering ──retrans fills hole──▶ normal
                          │
                          └── gap too big ──▶ WaitSnapshot ──load snapshot──▶ normal
```

The core discipline: **while recovering, the book is untrusted.** Better to not trade for 5 ms than to
quote off a book missing a cancel. "Correct-but-late beats fast-but-wrong" is the feed handler's
entire ethos — say that in the interview.

---

## 7. A/B line arbitration

The exchange sends two identical streams. You listen to both and merge them so a drop on one line is
covered by the other, *without* processing everything twice. Because both carry the same sequence
numbers, the sequencer's `Duplicate` verdict already does the dedup for you — arbitration is just
"feed both lines into one sequencer, first copy of each seq wins":

```cpp
// Two receive threads (A and B), one sequencer guarded so only the first copy of each seq is applied.
void FeedHandler::on_line_message(uint64_t seq, std::span<const std::byte> body) {
    switch (seq_.classify(seq)) {
        case Verdict::Process:   apply_to_book(body); drain_buffer_if_contiguous(); break;
        case Verdict::Duplicate: stats_.ab_dedup++;   break;      // the other line already had it
        case Verdict::GapAhead:  on_gap(seq_.expected(), seq, body); break;
        case Verdict::Resync:    begin_snapshot_resync(); break;
    }
}
```

Now a packet lost on Line A but present on Line B produces *no* gap — B's copy arrives, is `Process`,
and A's later duplicate (if it ever comes) is dropped. You only fall back to retrans/snapshot when
**both** lines miss the same sequence. This is why exchanges bother sending two lines: independent
drop probabilities multiply, so `P(lose on both) ≈ P(A) × P(B)` — tiny.

(Concurrency note: two threads hitting one `Sequencer` need synchronization. Common designs: a
per-line lock-free queue merged by a single arbitration thread, or one thread that polls both sockets.
Keep the shared-state surface tiny — Module 16.)

---

## 8. Normalize & hand off

The book builder must not know or care that this is NSE-TBT vs BSE-EOBI. So the last step converts the
parsed venue message into your **internal, venue-agnostic** message and publishes it.

```cpp
enum class MdType : uint8_t { Add, Modify, Cancel, Trade };

struct alignas(64) NormalizedMsg {     // 64-byte aligned: one per cache line, no false sharing (M15/16)
    MdType   type;
    char     side;        // 'B'/'S'
    uint32_t instrument;  // internal instrument id (mapped from the venue token)
    int64_t  price;       // internal fixed-point
    uint32_t qty;
    uint64_t order_id;
    uint64_t exch_ts_ns;  // exchange timestamp
    uint64_t rx_ts_ns;    // when *we* received it (hardware timestamp) — for latency measurement
};
```

Then publish across a **lock-free SPSC ring buffer** (Module 16) to the book-builder thread. The feed
handler is the single producer; the book builder is the single consumer. No locks, no allocation — the
ring is a fixed array filled at startup.

```cpp
// feed handler thread (producer)
NormalizedMsg m = normalize(parsed);
m.rx_ts_ns = hw_timestamp();          // ideally the NIC's hardware RX timestamp, PTP-synced
if (!ring_.try_push(m)) stats_.ring_full++;   // backpressure: consumer fell behind — a red flag

// book builder thread (consumer), elsewhere:
NormalizedMsg m;
while (ring_.try_pop(m)) book_.apply(m);
```

Two design points worth stating aloud:

- **The rx timestamp is captured as early as possible** (ideally by the NIC in hardware, PTP-synced to
  the exchange clock). The difference between `exch_ts_ns` and `rx_ts_ns` is your *feed latency*; the
  difference between `rx_ts_ns` and when your order hits the wire is your *tick-to-trade*. You can't
  optimize what you don't measure.
- **The ring decouples parsing from book-building** so a momentary slow consumer doesn't stall the
  receive thread (which would cause kernel drops → gaps). If the ring fills, that's a signal your
  consumer is too slow — an alert, not something to paper over.

---

## 9. Tradeoffs & design decisions (interview fuel)

| Decision | The tradeoff |
|---|---|
| **Kernel socket vs bypass (Onload/DPDK)** | Sockets are portable and simple; bypass saves a copy + context switch (~single-digit µs) but costs money, pins a core, and complicates dev. Have both: sockets for tests, bypass in prod. |
| **Busy-poll vs blocking/epoll** | Busy-poll removes wake-up latency but burns a whole core. Worth it on the hot feed; wasteful for a low-rate feed where you'd use epoll. |
| **`recvmmsg` batching** | Fewer syscalls (huge at open) but adds a touch of latency for the last packet in a batch. Small batch sizes balance it. |
| **Copy-on-parse vs zero-copy** | Parsing into structs is clean; truly zero-copy (interpret in place) is faster but fragile (alignment/aliasing). The `memcpy`-of-fixed-struct middle ground optimizes away. |
| **Single sequencer vs per-partition** | One sequencer is simple; per-partition/per-group sequencers parallelize across cores but multiply state and complicate ordering guarantees across instruments. |
| **Retrans vs snapshot threshold** | Retrans is faster for small gaps but adds request/response round-trips; snapshot is heavier but bounded. Pick a gap-size threshold; too eager on snapshots wastes time, too eager on retrans floods the service. |
| **Trust vs speed during recovery** | Marking the book stale costs trading opportunities but prevents trading on a wrong book. Non-negotiable: correctness wins. |

---

## 10. Problems & failure modes (the war stories interviewers love)

- **Silent kernel drops at open.** Small `SO_RCVBUF` → the *kernel* drops during the open microburst,
  you see a gap, and you blame the network. Fix: big buffers, batching, bypass, and *alert on gap
  counts*.
- **Endianness / packing bugs.** Forgot `bswap`, or a struct picked up padding → every field after
  the misalignment is garbage. Fix: `#pragma pack`, `static_assert(sizeof(T) == N)`, `memcpy` reads.
- **Applying past a gap.** Skipping to the newest seq without recovering corrupts the book (a missed
  cancel leaves a ghost order). Fix: the buffer-and-recover state machine (§5–6).
- **Duplicate-as-error.** Treating B-line duplicates or retransmits as errors floods logs and can
  trigger false recovery. Fix: `Duplicate` is a *normal* verdict, counted, dropped.
- **Ring overflow ignored.** Consumer stalls, ring fills, you drop normalized messages *after*
  parsing them correctly — a gap the sequencer can't even see. Fix: size the ring for bursts, alert on
  fullness, keep the consumer lean.
- **Clock confusion.** Mixing exchange timestamps with your host clock (unsynced) makes latency
  numbers meaningless and can mis-order events. Fix: PTP-sync, keep exch vs rx timestamps distinct.
- **Snapshot/incremental seam bugs.** Off-by-one when splicing the snapshot sequence to the live
  stream → you drop or double-apply one message at the boundary. Fix: the snapshot carries the seq it
  reflects; set `expected = snapshot_seq + 1` and buffer live msgs from before it arrived.

---

## 11. Discussion — likely interview questions

- *"Feed uses UDP — how do you not lose data?"* → Sequence numbers + gap detection; A/B line
  arbitration first (covers most drops), then retransmission for small gaps, snapshot resync for big
  ones; mark the book stale while recovering.
- *"A gap of 50 messages appears mid-session. Walk me through it."* → Classify `GapAhead`; stop
  applying to the book; buffer incoming (seq ≥ got); request retrans `[expected, got-1]`; on retrans
  fill, apply in order, drain the buffer, mark fresh; if the gap were huge, snapshot-resync instead.
- *"Why not TCP for market data?"* → TCP is per-connection (can't fan out to hundreds cheaply), and
  its in-order guarantee means head-of-line blocking: one lost packet stalls *everything* behind it.
  Multicast + app-level recovery lets you keep processing good data and recover only the hole.
- *"Where do you avoid allocation?"* → Everywhere on the hot path: preallocated RX buffers, structs on
  the stack, a fixed lock-free ring, an object pool for orders in the book. No `new`/`malloc` after
  startup.
- *"How do you parse safely and fast?"* → `#pragma pack` structs, `memcpy` into aligned locals (no
  aliasing/alignment UB, compiles to a load), byte-swap once, advance by wire `length`, `switch`
  dispatch, skip unknown types.
- *"How do you measure the handler's latency?"* → Hardware NIC RX timestamp vs exchange send
  timestamp = feed latency; track p50/p99/p99.9, not the mean; watch gap counts and ring fullness as
  health metrics.
- *"NSE vs BSE feeds?"* → NSE proprietary (TBT multicast, NNF order entry); BSE runs Deutsche Börse
  T7 (EOBI for TBT, ETI for orders). Mechanics — multicast, sequence numbers, A/B, snapshots — are the
  same; only field layouts differ, which is why you normalize behind an internal message type.

---

## The mental through-line

> The wire gives you fast, lossy, exchange-specific bytes. The feed handler's whole job is to hand the
> book builder a stream it can *trust*: received with the least latency (bypass, busy-poll, batching),
> parsed correctly (packed structs, byte-swap, memcpy), verified in-order (the sequencer's four
> verdicts), healed on loss (A/B arbitration → retrans → snapshot), and delivered venue-agnostically
> across a lock-free ring. Every earlier module shows up here — alignment, atomics, spans, zero-cost
> parsing — which is why this is the box that proves you can actually build HFT infrastructure, not
> just talk about it.

---

## Where this connects

- **Module 15/16** — the aligned structs, false-sharing avoidance, and the lock-free SPSC ring are
  exactly those tools.
- **Module 18** — `std::byteswap`, `std::span`, `std::endian` are the parsing primitives used above.
- **Module 03** — the "hand off" of §8 is precisely the input to your order-book builder.
- **Next: Module 03** — the order-book builder, which consumes exactly the normalized, in-order
  stream this handler produces and turns it into a live limit order book.
