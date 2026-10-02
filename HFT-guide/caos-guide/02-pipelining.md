# Module 02 — Pipelining

Pipelining is the single trick that lets a CPU start a new instruction every clock cycle even though
each instruction takes several cycles to finish. It's the reason CPI can approach 1 — and
understanding *where it breaks* (hazards, stalls, mispredicts) is the foundation for everything HFT
cares about: why a mispredicted branch costs ~15 cycles, why a dependent chain of loads is slow, and
why "branchless" code exists. This module builds the pipeline from the fetch–decode–execute cycle of
Module 01 and teaches you to count cycles like an interviewer will ask you to.

---

## 1. The idea: throughput vs latency (the laundry analogy)

One load of laundry goes wash → dry → fold. Say each takes 30 minutes, so one load takes 90 minutes.
If you do 4 loads *sequentially*, that's 360 minutes. But the washer, dryer, and folding table are
three separate machines — so once load 1 leaves the washer, load 2 can enter it. Overlap the stages
and 4 loads finish in 180 minutes, not 360.

Nothing made a *single* load finish faster — that's still 90 minutes (**latency** is unchanged). What
improved is how often a load *completes*: one every 30 minutes instead of one every 90
(**throughput** tripled). A CPU pipeline is exactly this: instructions are the laundry, pipeline
stages are the machines.

```
Non-pipelined (one instruction fully finishes before the next starts):
  I1: F D E W
  I2:         F D E W
  I3:                 F D E W       ← 4 cycles per instruction

Pipelined (a new instruction enters every cycle):
  cycle:  1  2  3  4  5  6  7
  I1:     F  D  E  W
  I2:        F  D  E  W
  I3:           F  D  E  W
  I4:              F  D  E  W        ← after fill, one finishes EVERY cycle
```

**Latency** of one instruction is still 4 cycles; **throughput** is now ~1 instruction/cycle.

---

## 2. The classic 5-stage RISC pipeline

The textbook MIPS-style pipeline has five stages, each handled by dedicated hardware with a register
(a "pipeline latch") between stages holding the in-flight instruction's state:

```
 IF  ─▶  ID  ─▶  EX  ─▶  MEM  ─▶  WB
 │        │       │       │        │
 fetch    decode  ALU     memory   write-
 instr,   & read  compute load/    back to
 PC += 4  regs    or addr store    register
```

1. **IF (Instruction Fetch)** — read the instruction at the PC from the L1 instruction cache; PC += 4.
2. **ID (Instruction Decode / register read)** — decode the opcode, read source registers from the
   register file.
3. **EX (Execute)** — the ALU computes the result, *or* computes a memory address for a load/store,
   *or* evaluates a branch condition.
4. **MEM (Memory access)** — for loads/stores, read/write data cache; other instructions pass through.
5. **WB (Write-Back)** — write the result into the destination register.

With five stages, up to **five instructions are in flight simultaneously**, each in a different stage.

### Ideal CPI and speedup

In the ideal case one instruction retires per cycle, so **CPI = 1**. The speedup over a
non-pipelined design approaches the number of stages (5 here) for a long instruction stream, minus
the startup "fill" (the first instruction still takes 5 cycles before *any* completes) and any
stalls. The clock can also run faster because each stage does less work per cycle — that's the other
half of pipelining's win.

> **Why not 100 stages then?** Deeper pipelines allow a higher clock (less work per stage) but make
> every stall/flush cost *more* cycles (you throw away more in-flight work) and add latch overhead.
> The Pentium 4's 20–31 stage pipeline chased clock speed and paid brutally on mispredicts; the
> industry settled on ~14–20 stages. See Module 03 for the mispredict math.

---

## 3. Hazards: why real CPI > 1

A **hazard** is any situation that prevents the next instruction from executing in its scheduled
cycle. There are three kinds.

### 3a. Structural hazard — two instructions want the same hardware

If the pipeline had a *single* memory port shared by IF and MEM, then in a cycle where I1 is in MEM
(reading data) and I4 is in IF (reading an instruction), both need memory at once — a **structural
hazard**. Solution: duplicate the resource. This is *the* reason for **split L1i/L1d caches** — so
instruction fetch and data access never collide. Structural hazards are mostly designed away in
modern CPUs.

### 3b. Data hazard — an instruction needs a result that isn't ready

The common and important one. Consider:

```asm
add r1, r2, r3     ; I1: r1 = r2 + r3   (result ready at end of EX/WB)
sub r4, r1, r5     ; I2: needs r1  — but I1 hasn't written it back yet!
```

This is a **RAW (Read-After-Write)** hazard — a true data dependency. Without help, I2 reaches ID
(where it reads registers) before I1 reaches WB (where it writes r1), so I2 would read a stale r1.

The three data-hazard types (know the names):

- **RAW (true dependency):** read after write — I2 reads what I1 writes. *Real* dependency; can't be
  removed, only bridged.
- **WAR (anti-dependency):** write after read — I2 writes a register I1 still needs to read. A naming
  artifact, removable by register renaming.
- **WAW (output dependency):** write after write — two instructions write the same register; order
  must be preserved. Also removable by renaming.

Only RAW is a *fundamental* dependency; WAR/WAW are "false" dependencies that out-of-order cores
eliminate with **register renaming** (Section 5).

### 3c. Control hazard — we don't know what to fetch next

A branch (`jmp`, `je`, `call`, `ret`) changes the PC, but the *new* PC often isn't known until the
branch is evaluated in EX. Meanwhile IF has already fetched the 1–3 instructions sitting right after
the branch. If the branch goes somewhere else, those fetched instructions are wrong and must be
discarded — a **control hazard**. This is what branch prediction (Module 03) exists to solve.

---

## 4. Fixing data hazards: forwarding, and the load-use stall

### Forwarding (bypassing)

The naïve fix is to **stall** — freeze I2 in ID for a couple of cycles (inserting "bubbles" — no-op
gaps) until I1's WB completes. That works but wastes cycles.

The clever fix is **forwarding (bypassing)**: the result of I1's `add` is actually *computed* at the
end of EX — it just hasn't been written to the register file yet. So route it directly from the EX
output back to the EX input of I2 via a bypass wire. I2 gets the value one cycle after it's computed,
no stall:

```
add r1,r2,r3 :  F  D  E  W
sub r4,r1,r5 :     F  D  E  W
                        ▲
                        └── r1 forwarded from I1's EX output straight
                            into I2's EX input — no bubble needed
```

Forwarding handles most ALU→ALU dependencies with zero stalls.

### The load-use hazard — the one forwarding *can't* fully hide

A load produces its value in **MEM**, not EX. So if the very next instruction uses that loaded value,
forwarding from MEM→EX arrives one cycle too late — you must insert **one bubble** (a load-use stall):

```asm
ld  r1, [r2]       ; value available only after MEM
add r3, r1, r4     ; needs r1 immediately → 1-cycle stall (bubble)
```

```
ld  r1,[r2] : F  D  E  M  W
add r3,r1,r4:    F  D  ⊘  E  W        ⊘ = bubble; add waits one cycle
                       ▲
                       forward from MEM (too late for back-to-back EX)
```

Compilers reorder code to fill that slot with an independent instruction ("load delay slot
scheduling"). This is why chains of dependent loads (pointer-chasing a linked list) are slow *even in
L1*: every load feeds the next, and you eat the load-use latency (and cache latency) with nothing to
overlap. It's a core reason HFT prefers contiguous arrays over pointer-linked structures (Module 04).

---

## 5. Going wider and out-of-order

The five-stage diagram is an **in-order** model: if an early instruction stalls, everything behind it
waits even when some later instructions are independent. Consider a deliberately simplified stream:

```text
A: r1 = load [price]       # long latency
B: r2 = r1 + 1             # depends on A
C: r3 = 5 * 2              # independent
D: r4 = r2 + r3            # depends on B and C
```

An in-order core cannot pass stalled `B`, so `C` waits unnecessarily. An **out-of-order (OoO)** core
starts `A`, notices that `C` is ready, and executes `C` while the load is outstanding. `B` runs when
`A` produces `r1`, then `D` runs when both inputs exist. The program has not been randomly shuffled:
the CPU follows the dependency graph and uses otherwise-idle execution units.

This exposes two kinds of parallelism:

- **Superscalar execution:** several ready operations can issue in one cycle to different execution
  ports—integer ALUs, vector units, branch units and load/store units. A modern wide core can retire
  several µops per cycle, so aggregate **IPC can exceed 1** (and average CPI can be below 1).
- **Out-of-order execution:** an independent younger operation may execute before an older stalled
  operation. This hides latency; it does not reduce the latency of the stalled operation itself.

### 5.1 The OoO journey: rename, wait, execute, retire

A modern implementation is descended conceptually from **Tomasulo's algorithm**. Exact structures
vary, but this is the useful mental model:

```text
fetch -> decode into µops -> rename + allocate ROB entries
                              |
                              v
                    scheduler / reservation stations
                     (wait for operands and a free port)
                              |
                              v
                  execute -> write result / wake dependants
                              |
                              v
                    retire from ROB head in program order
```

1. **Fetch and decode.** The front end fetches the predicted instruction path and translates complex
   instructions into simpler internal µops.
2. **Rename.** Architectural register names are mapped to physical registers. Sources point to the
   physical version that will produce their value; a destination receives a fresh physical register.
3. **Allocate and dispatch.** Each µop gets an entry in the **reorder buffer (ROB)** and is placed in
   a scheduler or **reservation station**. It may wait there for an operand, an execution port, or
   both. A reservation station does not contain only ready instructions—it also tracks which inputs
   are still pending.
4. **Issue and execute.** When all inputs are ready and the required port is free, the scheduler
   issues the µop, regardless of whether older unrelated µops have completed.
5. **Write back and wake up.** The result is written to a physical register and dependent µops are
   notified that an operand is now ready.
6. **Retire (commit).** Completed µops leave the head of the ROB in original program order. Only here
   do their results become architecturally committed.

The ROB is therefore not simply a queue of instructions waiting to execute. It preserves program
order around an out-of-order engine, records completion and exception state, and provides a recovery
point. If an older instruction faults, or a predicted branch was wrong, younger speculative work can
be discarded. This produces **precise exceptions**: software observes a state as if every older
instruction completed and no younger instruction did.

### 5.2 Register renaming removes false dependencies

Suppose the program reuses architectural register `r1`:

```text
A: r1 = load [x]
B: r2 = r1 + 1       # true RAW dependency on A
C: r1 = 10           # name reuse; must not destroy the value B still needs
D: r3 = r1 * 2       # needs C's version of r1
```

Renaming might create:

```text
A: p7  = load [x]
B: p8  = p7 + 1
C: p12 = 10
D: p13 = p12 * 2
```

Now `B` reads the old version (`p7`) while `C` creates a new version (`p12`). The apparent WAR/WAW
hazards disappear, so `C` can execute without corrupting `B`. The RAW edge from `A` to `B` remains:
renaming creates storage, not a value that has not yet been computed.

### 5.3 Loads, stores and speculation

Registers are comparatively easy because their dependencies are explicit. Memory operations use
calculated addresses, so a younger load may initially not know whether it overlaps an older store.
A **load/store queue** tracks in-flight memory operations. The core may predict that addresses do not
alias and let the load run; if the prediction was wrong, it replays the affected work. It can also
forward data from an older store directly to a younger load of the same address.

This speculation exposes **memory-level parallelism**: several independent cache misses can be in
flight together. It cannot rescue pointer chasing:

```cpp
node = node->next;       // next address is unknown until this load finishes
node = node->next;       // no independent address to send to the cache yet
```

The ROB, scheduler, physical-register file, load/store queues and miss-handling resources are all
finite. A sufficiently old cache miss can fill the instruction window; once no more work can enter,
the front end stalls even if useful independent work exists farther ahead in the program.

### 5.4 Speculation preserves architectural state, not every side effect

Branch prediction keeps the instruction window full by executing a predicted path before the branch
resolves. Correct predictions hide the control dependency. On a misprediction, the core restores the
rename state and discards younger results from the ROB (Module 03).

Architectural results are rolled back, but microarchitectural effects such as warmed cache lines can
remain. That distinction is the basis of Spectre-style side channels. For ordinary performance work,
the immediate costs are wasted execution bandwidth, energy, and a pipeline refill.

### 5.5 Hardware reordering is not permission for a C++ data race

OoO execution preserves the single-threaded illusion. Another core, however, can observe memory
operations under the machine's memory-order rules, and the compiler may also reorder code when the
C++ abstract machine permits it. Plain shared variables therefore do not become safe merely because
the ROB retires instructions in order.

```cpp
int data = 0;
std::atomic<bool> ready{false};

// Producer
data = 42;
ready.store(true, std::memory_order_release);

// Consumer
if (ready.load(std::memory_order_acquire)) {
    assert(data == 42);
}
```

The release/acquire pair establishes a **happens-before** relationship: after the consumer observes
`true`, it also observes the earlier write to `data`. Use C++ atomics and their memory orders rather
than imaginary `memory_barrier()` calls; Module 18 of the C++ guide develops that topic fully.

### 5.6 When OoO helps—and how code exposes more independent work

OoO helps when the instruction window contains independent operations, execution ports are
available, and latency—not memory bandwidth—is the bottleneck. It helps little when there is a long
dependency chain, frequent branch recovery, saturated bandwidth, or more outstanding work than the
hardware can track.

A reduction illustrates the programmer-visible consequence. One accumulator forms a serial chain:

```cpp
std::int64_t sum = 0;
for (std::size_t i = 0; i < n; ++i)
    sum += values[i];              // each add waits for the previous sum
```

Several partial sums create independent chains that the OoO engine—and often the vectorizer—can
overlap:

```cpp
std::int64_t s0 = 0, s1 = 0, s2 = 0, s3 = 0;
for (std::size_t i = 0; i + 3 < n; i += 4) {
    s0 += values[i];
    s1 += values[i + 1];
    s2 += values[i + 2];
    s3 += values[i + 3];
}
const auto sum = s0 + s1 + s2 + s3; // handle any tail separately
```

Do not mechanically rewrite every loop this way: optimizing compilers commonly unroll and
vectorize reductions, integer overflow semantics matter, and extra live values increase register
pressure. Inspect optimized assembly and hardware counters, then measure the actual workload.

The mental model for HFT is a wide, deep, speculative engine that dynamically schedules a bounded
window of work. It is brilliant at overlapping independent latency and helpless when the hot path is
one long dependency chain.

---

## 6. Vector execution: SIMD without the intrinsics rabbit hole

Superscalar execution runs several *different scalar instructions* at once. **SIMD** (Single
Instruction, Multiple Data) makes one instruction apply the same operation to several lanes:

```text
scalar add:   a0 + b0 -> c0

8-lane SIMD: [a0 a1 a2 a3 a4 a5 a6 a7]
            +[b0 b1 b2 b3 b4 b5 b6 b7]
            --------------------------------
             [c0 c1 c2 c3 c4 c5 c6 c7]
```

On x86, SSE uses 128-bit registers, AVX/AVX2 256-bit registers, and AVX-512 512-bit registers. A
256-bit AVX register can hold eight `float`s or four `double`s; width is potential parallelism, not a
guaranteed speedup.

### Auto-vectorization first

Compilers can vectorize a regular loop when iterations are independent and memory accesses are
predictable:

```cpp
for (std::size_t i = 0; i < n; ++i) {
    notionals[i] = prices[i] * quantities[i];
}
```

The common blockers are:

- a loop-carried dependency (`a[i] = a[i - 1] + x`);
- possible pointer aliasing—the compiler cannot prove input and output do not overlap;
- unpredictable control flow or function calls it cannot inline;
- gathers/scatters through random indices;
- too few iterations to repay setup and tail handling.

Contiguous Structure-of-Arrays data is often easier to vectorize than an Array-of-Structures when a
loop touches one field across many records. That is an access-pattern decision, not a universal rule:
processing every field of one order can still favor AoS (Module 04).

### Alignment, tails, and masks

Modern x86 supports unaligned vector loads, but crossing cache-line or page boundaries can cost more.
Aligned allocation and predictable strides make both vectorization and caching easier. When `n` is
not a multiple of the lane count, the compiler emits a scalar cleanup loop or masked tail. AVX-512
mask registers can also express per-lane conditions without a branch.

### Why wider is not automatically faster

- A memory-bound loop cannot consume data faster than caches/DRAM deliver it.
- Gather/scatter instructions still pay for the underlying irregular cache accesses.
- Horizontal reductions introduce dependencies between lanes.
- AVX-512 availability varies, and sustained wide-vector use can reduce clock frequency on some
  processors; dispatch and benchmark on the production CPU.
- More instructions/code paths can pressure the instruction cache.

The HFT rule: let the compiler vectorize regular batch analytics, risk calculations, decoding, and
checksums; do not force SIMD into a short branchy matching decision. Inspect the optimization report
and assembly, then measure end-to-end tail latency—not just arithmetic throughput.

---

## 7. Counting cycles (the "pipeline question")

Interviewers give you a code snippet and ask for total cycles or stall count. Method:

1. Baseline: `cycles ≈ (stages − 1) + N` for N instructions with no hazards (the `stages−1` is the
   fill).
2. Add stalls: **+1 bubble per load-use** dependency; **+branch-penalty** per taken/ mispredicted
   branch (if no prediction, ~stages−1 per control hazard).
3. Subtract nothing for forwarding on ALU→ALU (assume it exists on a modern core unless told
   otherwise).

**Worked example** (5-stage, full forwarding, no branch prediction):

```asm
1: ld  r1, [r2]        ; load
2: add r3, r1, r4      ; uses r1 (load-use → 1 bubble)
3: sub r5, r3, r6      ; uses r3 (ALU→ALU, forwarded, no stall)
4: st  r5, [r7]        ; uses r5 (forwarded)
```

Baseline no-stall: `(5−1) + 4 = 8` cycles. The load-use between (1) and (2) adds **1 bubble** →
**9 cycles**. (3)→(2) and (4)→(3) are ALU dependencies covered by forwarding, no penalty.

---

## Common pitfalls / misconceptions

- **"Pipelining makes each instruction faster."** No — it improves *throughput*, not single-instruction
  *latency*. One instruction still traverses all stages.
- **"Forwarding removes all stalls."** It removes ALU→ALU stalls but *not* the load-use stall (load
  result appears a stage later than an ALU result).
- **"WAR/WAW are real dependencies."** They're naming artifacts; register renaming removes them. Only
  RAW is fundamental.
- **"Deeper pipeline is always faster."** Deeper → higher clock but bigger flush penalty on
  mispredict; there's a sweet spot (~14–20 stages), and the Pentium 4 overshot it.
- **"CPI can't go below 1."** On superscalar OoO cores it does — IPC > 1 because multiple instructions
  retire per cycle.
- **"Out of order means random order."** The scheduler issues only operations whose operands are
  ready, so true dependencies are preserved; the ROB still commits architectural state in order.
- **"In-order retirement makes plain shared variables safe."** It preserves one thread's
  architectural behavior, not inter-thread synchronization. Use C++ atomics and the required memory
  order.
- **Ignoring the fill/drain.** For short sequences the `stages−1` startup cost dominates; don't forget
  it in cycle-counting questions.

---

## Quiz — tough problems

**Q1.** A non-pipelined CPU takes 4 ns/instruction. You split it into a balanced 4-stage pipeline.
What's the best-case throughput and the latency of a single instruction?
**Answer:** Each stage now takes ~1 ns, so throughput approaches **1 instruction/ns** (4× the old
0.25/ns), but single-instruction **latency stays ~4 ns** (it still traverses 4 stages). Throughput up
4×, latency unchanged — the essence of pipelining. (Real latches add a little overhead, so slightly
worse than the ideal.)

**Q2.** Classify each dependency:
```
add r1,r2,r3   (A)
sub r2,r5,r6   (B)
mul r1,r7,r8   (C)
or  r9,r1,r2   (D)
```
**Answer:** B writes r2 which A read → **WAR** (anti). C writes r1 which A wrote → **WAW** (output).
D reads r1 (from C) and r2 (from B) → two **RAW** (true) deps. Only the RAWs are fundamental; WAR/WAW
vanish under register renaming.

**Q3.** With full forwarding on a 5-stage pipe, how many bubbles here?
```
ld  r1,[r0]
ld  r2,[r1]
add r3,r2,r2
```
**Answer:** **Two** bubbles. `ld r2,[r1]` needs r1 from the first load (load-use → 1 bubble), and
`add` needs r2 from the second load (load-use → 1 bubble). Dependent pointer-chasing loads can't be
forwarded away — this is why linked-list traversal stalls even in cache.

**Q4.** Why does a split L1i/L1d cache eliminate a structural hazard?
**Answer:** IF (instruction fetch) and MEM (data access) can occur in the same cycle for different
in-flight instructions. With one unified memory port they'd contend (structural hazard); separate
instruction and data caches give each its own port so both proceed simultaneously.

**Q5.** A 5-stage and a 20-stage pipeline both mispredict a branch. Which loses more cycles and why?
**Answer:** The 20-stage one. The misprediction penalty ≈ the number of pipeline stages of wrong-path
work that must be flushed and refetched — roughly `stages − 1`. Deeper pipeline → more in-flight
speculative instructions squashed → bigger penalty. This is the deep-pipeline tradeoff.

**Q6.** On an out-of-order core, a load misses to DRAM (~200 cycles). Why might the program barely
slow down?
**Answer:** OoO execution + a large reorder buffer let the core keep executing *independent* later
instructions while the load is outstanding (memory-level parallelism). If there's enough independent
work — and no later instruction depends on the missing load — the ~200 cycles are largely hidden. If
the next instruction *needs* the loaded value, the stall is fully exposed.

**Q7.** Compute cycles: 5-stage, full forwarding, no branch prediction (assume a taken branch flushes
`stages−1 = 4`... actually the 2 wrongly-fetched instructions after it, use 2), for:
```
1: add r1,r2,r3
2: ld  r4,[r1]
3: add r5,r4,r6      ; load-use
4: beq r5, r0, L     ; taken branch
5: (fall-through insns fetched then squashed)
```
**Answer:** Baseline `(5−1)+4 = 8` for the 4 real instructions. +1 bubble for the load-use between
(2)→(3). +2 for squashing the two instructions fetched after the taken branch before its target is
known. Total ≈ **11 cycles**. (Exact branch penalty depends on which stage resolves the branch; state
your assumption — interviewers care that you *account* for it.)

**Q8.** Why do compilers reorder independent instructions between a load and its use?
**Answer:** To fill the load-use delay slot with useful work instead of a bubble. If an independent
instruction runs in the cycle the dependent one would have stalled, the load-use latency is hidden and
CPI stays near 1. This is "instruction scheduling."

**Q9.** Why might an eight-lane SIMD loop achieve much less than an 8× speedup?
**Answer:** Vector width accelerates arithmetic only. Loads may be bandwidth/cache limited, the loop
may need gathers, masks, reductions, and a scalar tail, and the wider instruction mix may lower clock
frequency. The slowest resource determines the speedup.

**Q10.** Why can `a[i] = a[i - 1] + b[i]` not be vectorized like `c[i] = a[i] + b[i]`?
**Answer:** The first has a loop-carried RAW dependency: iteration `i` needs the result of `i-1`, so
lanes cannot execute independently. The second iteration set is independent and maps naturally to
SIMD lanes.

**Q11.** An instruction has finished executing but an older instruction will raise a page fault. Why
does the younger instruction not retire immediately?
**Answer:** Retirement occurs from the head of the ROB in program order. Holding the younger result
allows the CPU to present a **precise exception**: all older instructions appear complete and no
younger instruction appears to have committed when the fault handler runs.

**Q12.** Why can four partial accumulators outperform one accumulator even though both perform nearly
the same number of additions?
**Answer:** One accumulator creates a single loop-carried dependency chain. Four accumulators create
four independent chains, allowing several adds to overlap on a superscalar OoO core. The final merge
adds a small fixed cost; register pressure and compiler vectorization still need to be checked.

---

## Indian HFT interview questions

**Q: What is the difference between throughput and latency in a pipeline? (Optiver, AlphaGrep)**
"Latency is how long one instruction takes end-to-end — pipelining doesn't reduce it; it still crosses
every stage. Throughput is how often instructions *complete* — pipelining raises it toward one per
cycle by overlapping stages. HFT cares about both: throughput for sustained work, latency for the
critical dependency chain on the hot path, which pipelining does *not* shorten."

**Q: Walk me through the hazards in a pipeline and how each is handled. (Tower Research, Graviton)**
"Structural — two instructions want the same unit; solved by duplicating hardware, e.g. split L1i/L1d.
Data — RAW (true, bridged by forwarding, but load-use still costs a bubble), WAR/WAW (false, removed by
register renaming). Control — the next PC after a branch is unknown; handled by branch prediction and
speculative execution, with a flush penalty on mispredict."

**Q: Why can't forwarding eliminate the load-use stall? (Quadeye, IMC)**
"An ALU result is ready at the end of EX, so it forwards to the next EX with no gap. A load's result
isn't ready until the end of MEM — one stage later — so a back-to-back dependent instruction still has
to wait one cycle. Compilers schedule an independent instruction into that slot; if they can't, you eat
the bubble. It's why dependent load chains (pointer chasing) are slow."

**Q: What does out-of-order execution buy you, and what's its limit? (Jump, HRT)**
"It executes instructions as their inputs become ready rather than in program order, retiring them in
order via the reorder buffer, so it overlaps independent work with long-latency operations (cache
misses) — memory-level parallelism. Its limit is dependency chains: if each instruction needs the
previous one's result, there's nothing independent to run ahead, and OoO can't help. Latency-bound hot
paths are exactly this case."

**Q: Walk through an instruction on an out-of-order CPU.**
"The front end fetches and decodes it into µops. Rename maps architectural sources and a fresh
physical destination, then the CPU allocates a ROB entry and dispatches the µop to the scheduler. It
waits until its inputs and an execution port are available, executes, writes back its result and wakes
dependants. It may finish before older instructions, but it retires from the ROB in program order. The
ROB makes branch recovery and precise exceptions possible."

**Q: Why isn't a 40-stage pipeline just better? (Da Vinci, Squarepoint)**
"Deeper stages mean less work per stage, so a higher clock — but the misprediction/flush penalty scales
with depth (you throw away `~stages` cycles of speculative work), latch overhead grows, and hazards get
costlier. Past ~20 stages the flush cost dominates; the Pentium 4 proved this. Modern cores balance
depth (~14–20) against predictor accuracy."

**Q: On modern hardware, can CPI be below 1? (NK Securities, Mansard)**
"Yes — superscalar cores issue and retire multiple instructions per cycle across several execution
ports, so IPC > 1 (CPI < 1). A good x86 core sustains ~3–4 IPC on friendly code. The architectural CPI
= 1 is the single-issue in-order ideal; real cores beat it when there's enough independent work, and
fall far short of it on dependency- or branch-bound code."

**Q: What stops a loop from auto-vectorizing?**
"Usually a loop-carried dependency, uncertain pointer aliasing, irregular gather/scatter access,
unvectorizable calls, or control flow whose masking cost exceeds the benefit. I check the compiler's
vectorization report, then the generated assembly and counters; I don't infer SIMD from `-O3`."

---

## Key takeaways

- Pipelining overlaps the fetch/decode/execute stages so a new instruction can start every cycle:
  **throughput** rises toward 1/cycle; single-instruction **latency** is unchanged.
- The classic 5-stage pipe is IF→ID→EX→MEM→WB; ideal CPI = 1, real CPI > 1 because of hazards.
- Three hazards: **structural** (resource conflict, solved by duplication like split L1i/L1d),
  **data** (RAW true / WAR & WAW false), **control** (branch — the domain of Module 03).
- **Forwarding** bridges ALU→ALU dependencies with no stall; the **load-use** hazard still costs one
  bubble because a load's value appears a stage later — the root of slow pointer-chasing.
- Modern cores are **superscalar + out-of-order + register-renamed + speculative**: reservation
  stations schedule ready µops while the ROB restores in-order retirement, precise exceptions and
  recovery. They hide latency brilliantly when independent work exists and stall on dependency chains.
- A load/store queue tracks speculative memory operations; memory-level parallelism overlaps
  independent misses, but dependent pointer chasing exposes the full latency.
- In-order retirement does not provide inter-thread ordering. Correct shared-memory communication
  still requires C++ atomics and suitable acquire/release semantics.
- **SIMD** exploits data-level parallelism across vector lanes. It works best on independent,
  contiguous, regular loops; dependencies, gathers, bandwidth, and tails cap the gain.
- Deeper pipelines buy clock speed but pay a larger flush penalty on mispredict — there's a sweet spot.
- Cycle-counting: `(stages−1) + N`, then add load-use bubbles and branch penalties.

**Next:** [03 — Branch prediction](03-branch-prediction.md) — how the CPU guesses which way a branch
goes so the pipeline never drains, and what it costs when it guesses wrong.
