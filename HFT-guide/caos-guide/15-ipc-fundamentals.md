# Module 15 — IPC fundamentals: pipes, queues, sockets & shared memory

Processes are isolated: each has its own virtual address space and cannot normally dereference
another process's pointers. **Inter-process communication (IPC)** is the set of mechanisms that lets
processes exchange bytes, messages, or shared state without removing that isolation.

The interview skill is not memorizing syscall signatures. It is choosing a mechanism from its
semantics: stream or messages, copied or shared data, local or network-capable, blocking or polling,
who owns each buffer, and what happens when one process crashes.

---

## 1. The two fundamental IPC models

Almost every IPC mechanism is one of two models.

### A. Kernel-mediated message/byte transfer

```text
process A user buffer
        │ write/send
        ▼
kernel pipe/socket/queue buffer
        │ read/receive
        ▼
process B user buffer
```

The kernel owns the transport state, validates access, buffers data, wakes waiters, and cleans up
descriptors. This is simple and robust, but calls into the kernel and data movement add overhead.

Examples: pipes, FIFOs, message queues, Unix-domain sockets, TCP/UDP sockets.

### B. Shared memory

```text
process A virtual address        process B virtual address
           │                                │
           └──────────────┬─────────────────┘
                          ▼
                 same physical pages
```

After setup, both processes load/store the same physical memory directly. Transfer does not require
a syscall per message and payloads need not be copied through a kernel transport buffer. But the
kernel no longer supplies message boundaries, synchronization, backpressure, or crash consistency;
the application must design them.

Shared memory is fast because it removes work, not because it bypasses cache coherency. Cache-line
ownership, memory ordering, false sharing, and NUMA still apply—often more visibly.

---

## 2. The questions that choose an IPC mechanism

Before naming an API, ask:

1. **Communication scope:** related local processes, unrelated local processes, or different hosts?
2. **Data model:** byte stream, datagrams, discrete messages, or shared mutable objects?
3. **Topology:** one-to-one, producer-consumer, fan-out, request-response, or many-to-many?
4. **Volume:** tiny control messages or a continuous high-rate payload stream?
5. **Latency/throughput:** is the bottleneck syscall cost, copying, scheduling, coherency, or network?
6. **Backpressure:** block, drop, overwrite, reject, or queue when the consumer is slow?
7. **Failure model:** what if a peer exits halfway through a write or while owning shared state?
8. **Portability/security:** must it cross machines, users, containers, or operating systems?

The fastest microbenchmark result is not automatically the best design. Semantics and recovery can
save far more engineering time than a few avoided copies on a cold control path.

---

## 3. Pipes and FIFOs: simple byte streams

An **unnamed pipe** is a kernel byte buffer normally created before `fork`, so related processes
inherit its file descriptors. One end writes; the other reads:

```text
producer ── byte stream ──▶ kernel pipe buffer ──▶ consumer
```

A **FIFO** (named pipe) gives the pipe a filesystem name so unrelated local processes can open it.
The transport semantics are still pipe-like.

Properties:

- ordered byte stream—no inherent application message boundaries;
- natural blocking/wakeup and finite-buffer backpressure;
- kernel-managed lifetime/permissions and clear end-of-stream when writers close;
- typically one direction per pipe; use two for request-response;
- copying and kernel crossings on the data path.

POSIX guarantees that writes no larger than `PIPE_BUF` are not interleaved with other writers; that
does not make the pipe message-oriented, and larger writes may interleave.

If the sender writes two structs, the receiver must still frame the stream correctly. Never assume
one `write` equals one `read`; reads may return fewer bytes, signals may interrupt operations, and
larger writes may interleave. Define a fixed record size or length-prefix protocol and loop until the
record is complete.

Use pipes for shell-like pipelines, child-worker control, logs, and straightforward local streaming
where simplicity matters more than the last microsecond.

---

## 4. Message queues: preserve message boundaries

A message queue stores discrete kernel-managed records rather than an undifferentiated stream:

```text
send {type, length, payload} ──▶ [msg][msg][msg] ──▶ receive one message
```

Depending on the API, queues may offer priorities, typed messages, blocking/non-blocking operations,
and persistence beyond one process lifetime. The benefits are explicit framing and simple
producer-consumer coordination; the costs are kernel bookkeeping, copies, bounded capacity, and
platform-specific limits.

Queue **ordering** is not the same as end-to-end correctness. If multiple producers represent one
logical feed, include sequence numbers when gaps, duplicates, or replay matter. A priority queue can
also reorder normal messages and starve low-priority work, so use priority only when the protocol
defines that behavior.

Message queues suit control events, job dispatch, and modest-rate structured messages. They are not
the default for bulk market-data payloads on the hottest local path.

---

## 5. Sockets: one interface for local or remote peers

Sockets are communication endpoints. The important distinction is semantic:

- **Unix-domain sockets**: local machine, filesystem/abstract name, stream or datagram semantics;
- **TCP**: reliable ordered byte stream between hosts, with connection and flow control;
- **UDP**: connectionless datagrams; preserves message boundaries but may drop, duplicate, or reorder.

For local IPC, Unix-domain sockets usually avoid IP routing but still use kernel socket buffers and
syscalls. They are attractive when you want bidirectional communication, credential/permission
checks, standard event-loop integration, and a familiar request-response protocol.

TCP does not preserve application messages: it may combine or split sends, just like a pipe. UDP
preserves each datagram boundary, but the application owns sequencing, loss detection, recovery, and
maximum-datagram decisions. “TCP is reliable” also does not mean the remote application processed a
message; it means the byte stream was delivered to the peer's transport stack or a failure surfaced.

Choose sockets when process placement may cross a host boundary or operational simplicity and
standard tooling outweigh local-copy overhead. This chapter deliberately stops at those semantics;
the networking guide covers socket APIs and tuning.

---

## 6. Shared memory: the fastest path with the largest contract

The OS can map the same shared-memory object into multiple address spaces. POSIX shared memory and a
shared file mapping differ in naming/backing/lifetime details, but the resulting principle is the
same: virtual addresses in both processes translate to common physical pages.

### Pointers are process-relative

The mapping may appear at different virtual addresses:

```text
process A maps region at 0x7000... ─┐
                                    ├─ same physical page
process B maps region at 0x4a00... ─┘
```

A raw pointer stored inside the region is generally invalid in the other process. Store **offsets or
indices from the mapping base**, then reconstruct a local pointer:

```cpp
auto* local = base + record.payload_offset;
```

### Define a stable layout

A shared region is an ABI/protocol. State explicitly:

- magic number and version;
- region size and capacity;
- fixed-width field representations and byte order if portability matters;
- alignment of atomics and cache-line-separated producer/consumer state;
- initialization state/generation so a new process can detect stale memory;
- ownership and cleanup rules.

Do not place `std::string`, `std::vector`, ordinary virtual objects, process-local mutexes, or raw
heap pointers in shared memory and expect another process to use them. Their internal pointers and
allocator/runtime state belong to the creating process.

### Synchronization is still required

Two processes concurrently accessing shared pages face the same data-race and ordering problems as
threads. Use an OS-documented process-shared synchronization primitive or a deliberately supported
lock-free layout. Do not assume every `std::atomic<T>` implementation or every library mutex is
automatically valid across processes merely because its bytes are mapped twice; verify the platform
contract, lock-free property, construction, alignment, and recovery behavior.

### Crash consistency

If a producer dies after writing half a record, the consumer needs a way to distinguish incomplete
data. Common techniques include publish-last sequence/generation fields, checksums for persistent
records, ownership epochs, and rebuilding from a known snapshot/log. A process-shared mutex also
needs a robust-owner-death plan or it can remain apparently locked forever.

---

## 7. Memory-mapped files: file data through virtual memory

`mmap`-style mapping makes file-backed pages part of a process's virtual address space. Loads and
stores then go through the normal MMU/page-cache path; a first access may fault a page in, and dirty
pages are eventually written back.

This is useful for large read-mostly datasets, append/replay logs, indexes, and sharing file-backed
data between processes. It avoids an explicit `read` call and user-buffer copy for every access, but
it does **not** mean “the whole file is already in RAM” or “access cannot block.” Major page faults,
writeback, truncation, and storage errors still exist.

Important lifetime rules:

- mapping length and file size must agree;
- pointers/views into the mapping die when it is unmapped or remapped;
- a concurrent file truncation can invalidate accesses;
- visibility and durability are different—another mapping seeing dirty cache pages does not prove
  bytes are on stable storage;
- random access to a mapping can still thrash the page cache/TLB.

Use mapped files when the file-as-memory access pattern is helpful, not merely to replace every
`read` call syntactically.

---

## 8. Zero-copy: count ownership transitions and copies

“Zero-copy” means eliminating avoidable payload copies; it rarely means zero data movement. Compare:

```text
traditional send:
storage/page cache -> user buffer -> socket/NIC path

shared/mapped handoff:
producer writes shared buffer -> consumer reads same bytes

kernel-assisted file send:
page cache -> socket/NIC path (no application payload buffer)
```

Every copy consumes CPU, memory bandwidth, and cache capacity. Avoiding a copy is valuable for large
payloads or high throughput, but small control messages may be faster and simpler to copy than to set
up reference-counted/shared ownership.

The hard part becomes **buffer lifetime**: the producer must not reuse a buffer until every consumer
is finished, and slow consumers need a policy. Reference counts, completion rings, generations, or
fixed ownership stages solve that problem at a cost. Zero-copy transfers bytes; it does not remove
framing, synchronization, backpressure, or failure handling.

---

## 9. The fundamental low-latency pattern: shared SPSC ring

For one producer and one consumer, a bounded ring buffer is the simplest shared-memory hot path:

```text
slots: [0][1][2][3][4][5][6][7]
             ^ read/head        ^ write/tail

producer owns tail and slot publication
consumer owns head and slot reclamation
```

Protocol:

1. Producer checks that advancing `tail` would not catch `head` (full).
2. Producer writes the complete payload into its free slot.
3. Producer **publishes** the new tail with release semantics.
4. Consumer reads tail with acquire semantics; seeing it makes the payload visible.
5. Consumer reads the slot, then releases the new head so it can be reused.

Place head and tail on separate cache lines to avoid false sharing. Use fixed capacity and define
full behavior—reject/drop/backpressure—rather than allocate unexpectedly. The detailed C++ memory
ordering and implementation are in C++ Module 18; for cross-process use, validate the operating
system and toolchain's shared-atomic contract.

Do not generalize SPSC code to multiple producers or consumers by adding `fetch_add`. MPSC/MPMC
queues need per-slot publication, contention handling, and often memory-reclamation machinery. Use a
proven implementation unless that algorithm itself is the interview topic.

---

## 10. Trade-off table

| Mechanism | Data model | Copies / kernel work | Main strength | Main risk/cost |
|---|---|---|---|---|
| unnamed pipe | byte stream | kernel buffered | simplest related-process stream | no message boundaries |
| FIFO | byte stream | kernel buffered | unrelated local processes by name | lifecycle/blocking quirks |
| message queue | discrete messages | kernel queue/bookkeeping | framing, priorities | limits and overhead |
| Unix socket | stream/datagram | socket buffers/kernel | bidirectional, credentials, tooling | copies/syscalls |
| TCP socket | reliable byte stream | full transport path | cross-host transparency | framing + variable network latency |
| UDP socket | datagrams | transport path | message boundaries, low protocol overhead | loss/reorder/recovery |
| shared memory | application-defined | direct loads/stores after setup | lowest-copy local data path | synchronization, ABI, crash recovery |
| mapped file | file-backed pages | page faults/page cache | large datasets and replay | faults, truncation, durability subtleties |

Do not attach universal nanosecond numbers to this table. Payload size, contention, wakeup policy,
NUMA placement, page state, kernel version, and hardware can reverse a simplistic ranking. Benchmark
the actual topology and report distribution/tail latency.

---

## Common pitfalls / misconceptions

- **“Shared memory needs no synchronization.”** Same physical pages mean conflicting access is more
  direct, not automatically ordered.
- **Storing raw pointers in shared memory.** Different processes can map the region at different
  virtual addresses; use offsets/indices.
- **Treating a pipe/TCP stream as messages.** Reads and writes do not preserve application framing.
- **Treating UDP delivery as reliable or ordered.** Add sequence/gap/recovery semantics when needed.
- **Calling `mmap` zero-latency I/O.** First touches can page-fault; dirty pages eventually need
  writeback.
- **Assuming zero-copy solves ownership.** Buffer reuse and slow-consumer policy become the central
  correctness problem.
- **No crash protocol.** A writer can die mid-record or while owning a process-shared lock.
- **Unbounded buffering.** It hides overload temporarily, then turns it into memory growth and huge
  tail latency. Backpressure/drop policy must be explicit.
- **Using benchmark folklore.** IPC cost depends heavily on message size, scheduling, NUMA, and
  contention; measure the deployed system.

---

## Quiz — tough problems

**Q1.** Two processes map one shared region at different virtual addresses. Why is a pointer stored
by process A unsafe in process B, and what should be stored instead?
**Answer:** A pointer is a virtual address meaningful in A's page tables. B's mapping base may differ.
Store an offset/index relative to the region base and reconstruct a local pointer after bounds checks.

**Q2.** Producer calls one `write` per message on a pipe. Can the consumer call one `read` and assume
it got exactly one message?
**Answer:** No. A pipe is a byte stream; reads may split/combine writes and can be short. Define
framing and accumulate bytes until a complete record is available.

**Q3.** Why does shared memory often beat a pipe for a high-rate local feed?
**Answer:** After mapping, payload handoff uses ordinary coherent loads/stores instead of per-message
kernel crossings and transport-buffer copies. It pays with application-owned synchronization,
backpressure, ABI, and recovery complexity.

**Q4.** Is a memory-mapped file guaranteed not to block during a load?
**Answer:** No. The page may not be resident, producing a page fault and storage I/O. Mapping changes
the access interface; it does not abolish the storage hierarchy.

**Q5.** In an SPSC ring, why publish the tail only after writing the payload?
**Answer:** Tail is the publication marker. A release store after the payload paired with the
consumer's acquire load ensures that observing the new tail also makes the completed slot visible.

**Q6.** What does “zero-copy” fail to solve?
**Answer:** Buffer ownership/lifetime, synchronization, message framing, backpressure, slow consumers,
error handling, and crash recovery. It only removes selected data copies.

**Q7.** When is a Unix-domain socket preferable to shared memory?
**Answer:** When bidirectional messages, credentials, kernel-managed blocking/cleanup, debugging, and
simplicity matter more than eliminating the local transport copies—and the measured rate is adequate.

---

## HFT interview questions

**Q: Compare pipes, sockets, message queues, and shared memory.**
**A:** “Pipes and TCP/Unix stream sockets carry ordered bytes, so the application frames messages;
queues preserve records and may add priority; sockets can cross host boundaries; shared memory gives
the lowest-copy local path but leaves synchronization, backpressure, ABI, and recovery to the
application. I choose from semantics first, then benchmark.”

**Q: Design IPC from a market-data process to a strategy process on one host.**
**A:** “For the hot one-producer/one-consumer path, a preallocated shared-memory SPSC ring with fixed
records, sequence numbers, cache-line-separated indices, release/acquire publication, and an explicit
full policy. A socket/pipe can handle setup and control. I add heartbeats/generation state so restart
cannot consume stale partial data.”

**Q: What makes shared memory difficult?**
**A:** “It shares bytes, not a protocol: relative addressing, stable layout/versioning, memory
ordering, cache/NUMA effects, backpressure, initialization, permissions, and owner-death recovery all
become application responsibilities.”

**Q: What is zero-copy?**
**A:** “Removing avoidable payload copies—for example sharing/mapping the same pages or letting the
kernel move file-backed pages toward a socket without an application buffer. Bytes still move through
caches/memory/devices, and buffer lifetime replaces copying as the main contract.”

**Q: Why use sequence numbers even with a local ring?**
**A:** “They detect gaps, overwrite/restart generations, and stale or partial observations; they also
give monitoring/recovery a protocol-level truth instead of relying only on mutable indices.”

---

## In the trading system

- **Hot local market-data handoff:** bounded shared-memory SPSC ring when the topology is fixed and
  the latency gain justifies operational complexity.
- **Control/configuration/health:** Unix-domain socket, pipe, or message queue—clarity and recovery
  dominate a cold path.
- **Cross-host exchange/gateway traffic:** sockets/network protocols; shared memory cannot cross the
  machine boundary.
- **Capture and replay:** append log plus memory mapping/read-ahead where it matches the workload;
  page residency is controlled during latency-sensitive replay.
- **Process restart:** version/generation, heartbeat, sequence, and ownership state are protocol
  fields, not afterthoughts.

---

## Key takeaways

- IPC is either kernel-mediated transfer or shared pages; the second removes copies/syscalls from the
  hot data path but transfers coordination responsibility to the application.
- Streams need framing; datagrams/messages preserve boundaries but differ in reliability and scope.
- Shared memory requires relative addressing, a stable layout, synchronization, backpressure, and a
  crash/restart protocol.
- Memory mapping uses virtual memory/page cache and can still page-fault; visibility is not durability.
- Zero-copy removes selected copies, not data movement or buffer-ownership problems.
- A bounded shared SPSC ring is the foundational low-latency local pattern; MPSC/MPMC is a different,
  much harder algorithm.
- Choose by semantics and failure model, then benchmark the real message size, topology, NUMA
  placement, load, and tail latency.

**Next:** Return to the [CAOS roadmap](00-index.md), or continue with the networking/tuning material
for deeper socket, kernel-bypass, and deployment-specific details.
