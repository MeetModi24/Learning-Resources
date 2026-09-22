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

Real high-performance cores go beyond the simple in-order 5-stage pipe:

- **Superscalar** — multiple parallel pipelines (execution ports) so *more than one* instruction can
  issue per cycle. A modern x86 core can retire ~4–6 µops/cycle, i.e. **CPI < 1** (IPC > 1). It has
  several ALUs, load/store units, etc.
- **Out-of-order (OoO) execution** — instructions execute as soon as their inputs are ready, not in
  program order. A **reorder buffer (ROB)** tracks them and *retires* them in program order (so the
  visible result is correct), while a scheduler dispatches ready ops to free ports. This hides latency:
  while a cache-missing load is outstanding, independent later instructions run ahead.
- **Register renaming** — the ~16 architectural registers are mapped onto a much larger pool of
  physical registers, which dissolves WAR/WAW false dependencies (two instructions "writing r1" get
  different physical registers).
- **Speculative execution** — the core executes down the *predicted* branch path before the branch
  resolves, squashing the work if the prediction was wrong (Module 03; also the root of Spectre).

The mental model for HFT: the machine is a wide, deep, speculative, out-of-order engine that is
*brilliant* at hiding latency **when there's independent work to overlap**, and helpless when there
isn't (long dependency chains, unpredictable branches, cache misses with nothing else to do).

---

## 6. Counting cycles (the "pipeline question")

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

---

## Key takeaways

- Pipelining overlaps the fetch/decode/execute stages so a new instruction can start every cycle:
  **throughput** rises toward 1/cycle; single-instruction **latency** is unchanged.
- The classic 5-stage pipe is IF→ID→EX→MEM→WB; ideal CPI = 1, real CPI > 1 because of hazards.
- Three hazards: **structural** (resource conflict, solved by duplication like split L1i/L1d),
  **data** (RAW true / WAR & WAW false), **control** (branch — the domain of Module 03).
- **Forwarding** bridges ALU→ALU dependencies with no stall; the **load-use** hazard still costs one
  bubble because a load's value appears a stage later — the root of slow pointer-chasing.
- Modern cores are **superscalar + out-of-order + register-renamed + speculative**: they hide latency
  brilliantly when there's independent work, and stall on dependency chains and unpredictable branches.
- Deeper pipelines buy clock speed but pay a larger flush penalty on mispredict — there's a sweet spot.
- Cycle-counting: `(stages−1) + N`, then add load-use bubbles and branch penalties.

**Next:** [03 — Branch prediction](03-branch-prediction.md) — how the CPU guesses which way a branch
goes so the pipeline never drains, and what it costs when it guesses wrong.
