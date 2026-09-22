# Module 01 — CA foundations & the ISA

Before you can reason about *why* code is fast or slow, you need a mental picture of the machine that
runs it: what a CPU actually does, where your data lives, and what an "instruction" is. This module
builds that picture from zero and then tackles the single most common architecture warm-up in an HFT
interview — **CISC vs RISC** — which is really a question about how the ISA choice ripples into
pipelining, decode cost, and instruction throughput (the subjects of the next two modules).

---

## 1. The von Neumann model: one memory for code *and* data

A modern computer is, at its core, astonishingly simple. There is:

- **Memory** — one giant, flat array of bytes, each with a numeric address (`0, 1, 2, … `). It holds
  *both* your program's instructions *and* the data those instructions operate on. This "stored
  program" idea — code is just data you can load, jump into, and even generate — is the **von Neumann
  model**, and it's what makes a general-purpose computer possible.
- **A CPU** — which repeatedly reads an instruction from memory, figures out what it means, and does
  it.
- **A bus** — the wires connecting them.

```
        ┌──────────────────────────── CPU ────────────────────────────┐
        │  Registers        Control Unit            ALU                │
        │  ┌────────┐       ┌──────────┐        ┌──────────┐           │
        │  │ RAX RBX│       │ decode & │        │ add, sub │           │
        │  │ RCX ...│◀────▶ │ sequence │ ─────▶ │ and, cmp │           │
        │  │ RIP RSP│       └──────────┘        └──────────┘           │
        │  └────────┘             ▲                                    │
        └─────────────────────────┼────────────────────────────────────┘
                                  │  address / data bus
                                  ▼
        ┌───────────────────────────────────────────────────────────┐
        │  MEMORY (one flat byte array — code AND data live here)     │
        │  0x…00: 48 89 e5   (mov rbp, rsp)   ← instructions          │
        │  0x…10: 00 00 00 05                 ← data (an int = 5)      │
        └───────────────────────────────────────────────────────────┘
```

The famous consequence is the **von Neumann bottleneck**: because code and data share one path to
memory, the CPU is perpetually starved for that bus. *Everything* in modern architecture — caches,
pipelines, prefetchers, out-of-order execution — exists to hide the fact that memory is far slower
than the CPU. Keep that framing: the CPU is fast; memory is the enemy.

> **A note on "Harvard".** Real CPUs cheat: the L1 cache is *split* into an L1-instruction cache and
> an L1-data cache (a "modified Harvard" arrangement), so instruction fetch and data access don't
> fight over the same port. But main memory is still one unified von Neumann space.

---

## 2. Registers: the CPU's scratchpad, and why they're ~free

Memory is slow, so the CPU keeps a tiny set of ultra-fast storage slots *on the chip itself*:
**registers**. On x86-64 there are 16 general-purpose 64-bit registers (`RAX`, `RBX`, `RCX`, `RDX`,
`RSI`, `RDI`, `RBP`, `RSP`, `R8`–`R15`) plus special ones:

- **`RIP`** — the *instruction pointer* (a.k.a. the program counter, PC): the address of the next
  instruction to fetch. Executing a program is essentially "advance RIP, do what's there, repeat."
- **`RSP`** — the *stack pointer*: top of the call stack.
- **`RFLAGS`** — the *flags* register: single bits set by the last operation (Zero, Sign, Carry,
  Overflow). This is how `cmp` then `je` works — `cmp` sets flags, `je` reads them.

Why do we care? **Latency.** A register access is essentially 0 extra cycles — the value is already
inside the execution unit. Compare the hierarchy you'll meet in Module 04:

```
register      : ~0 cycles   (it's already in the CPU)
L1 cache      : ~4 cycles
L2 cache      : ~12 cycles
L3 cache      : ~40 cycles
main memory   : ~200+ cycles (~60–100 ns)
```

This is why compilers work so hard to keep hot variables in registers, and why `register`-pressure
(too many live values, forcing "spills" to the stack) hurts. An HFT hot loop that fits its working
set in registers and L1 can be *two orders of magnitude* faster than one that keeps touching DRAM —
same Big-O, wildly different wall-clock.

---

## 3. The fetch–decode–execute cycle

The CPU's entire life is one loop, the **instruction cycle**:

```
        ┌────────────────────────────────────────────────┐
        │                                                  │
        ▼                                                  │
   ┌─────────┐   ┌─────────┐   ┌──────────┐   ┌─────────┐  │
   │  FETCH  │──▶│ DECODE  │──▶│ EXECUTE  │──▶│WRITEBACK│──┘
   │ read    │   │ what op?│   │ do it in │   │ store   │
   │ [RIP],  │   │ which   │   │ the ALU  │   │ result, │
   │ RIP += n│   │ operands│   │ /mem     │   │ set flag│
   └─────────┘   └─────────┘   └──────────┘   └─────────┘
```

1. **Fetch** — read the instruction bytes at `RIP`, advance `RIP` past them.
2. **Decode** — work out the operation (`add`? `load`? `jump`?) and its operands (which registers,
   which memory address, what immediate constant).
3. **Execute** — the ALU computes (`add`, `sub`, `and`, `cmp`, …), or the address is sent to memory
   for a load/store.
4. **Write-back** — put the result in the destination register, update flags.

Done naïvely, one instruction finishes before the next begins. That wastes hardware: while the ALU
runs, the fetch unit sits idle. **Pipelining** (Module 02) overlaps these stages so a new instruction
starts every cycle. But you can only pipeline cleanly if instructions are *easy to decode* — which is
exactly where the ISA design choice bites.

---

## 4. What an ISA is

The **Instruction Set Architecture** is the contract between hardware and software: the list of
instructions the CPU understands, the registers it exposes, the addressing modes, the memory model
(Module 05/16 territory), and the binary encoding of each instruction. It is the boundary the
compiler targets. Two chips with the *same* ISA (say two x86-64 CPUs) run the same binaries even
though their internal *microarchitecture* (pipeline depth, cache sizes, predictors) differs wildly.

Keep **ISA vs microarchitecture** straight — interviewers love this distinction:

- **ISA** = the visible programming model (x86-64, AArch64, RISC-V). Stable for decades.
- **Microarchitecture** = how a specific chip *implements* that ISA (Intel Golden Cove, Apple
  Firestorm, AMD Zen 4). Changes every generation.

---

## 5. CISC vs RISC

Two philosophies for designing an ISA:

**CISC (Complex Instruction Set Computer)** — e.g. **x86 / x86-64**. Many instructions, some very
complex, **variable length** (an x86 instruction is 1–15 bytes). A single instruction can do a lot:
`add rax, [rbx + rcx*4 + 8]` reads memory with a computed address *and* adds *and* writes back, all
in "one instruction." Historically motivated by expensive memory (fewer, denser instructions = smaller
programs) and hand-written assembly.

**RISC (Reduced Instruction Set Computer)** — e.g. **ARM/AArch64, RISC-V, MIPS**. Fewer, simpler
instructions, **fixed length** (typically 4 bytes), a **load-store architecture**: only explicit
`load`/`store` instructions touch memory; everything else operates register-to-register. That same
add-from-memory becomes three instructions: `load` the value, `add` in registers, `store` back.

```
CISC (x86-64)                          RISC (AArch64, load-store)
────────────────                       ─────────────────────────
add rax, [rbx+rcx*4+8]   ; 1 insn      ldr x3, [x1, x2, lsl #2]  ; load
                                        add x0, x0, x3            ; add
                                        ; (store separately if needed)
variable length (1–15 B)               fixed length (4 B)
few registers historically (8→16)      many registers (32)
```

### Key contrasts

| Dimension            | CISC (x86-64)                    | RISC (AArch64/RISC-V)          |
|----------------------|----------------------------------|--------------------------------|
| Instruction length   | Variable (1–15 B)                | Fixed (4 B)                    |
| Memory operands      | Yes, in arithmetic ops           | Load-store only                |
| Instruction count    | Large, complex                   | Small, simple                  |
| Registers            | 16 GP                            | 31–32 GP                       |
| Decode               | Hard (must find boundaries)      | Trivial (align on 4 B)         |
| Code size            | **Smaller**                      | Larger                         |
| CPI (naïve)          | Higher (complex insns)           | Lower (uniform, ~1)            |

### The twist: modern x86 is CISC on the outside, RISC on the inside

This is the fact that separates a memorized answer from an understood one. A modern x86 core does
**not** execute those complex variable-length instructions directly. Its front-end *decodes* each
x86 instruction into one or more **micro-operations (µops)** — small, fixed-format, RISC-like
internal instructions — and the fast RISC-style back-end (pipeline, reorder buffer, execution ports)
runs *those*. `add rax, [mem]` becomes roughly a load-µop plus an add-µop internally.

To avoid paying the expensive x86 decode over and over in a hot loop, the core also keeps a **µop
cache** (decoded-stream buffer) that stores already-decoded µops, so tight loops skip the legacy
decoder entirely. So the "CISC decode is slow" penalty is real but *amortized*.

---

## 6. CPI, and the two classic interview questions

**CPI = Cycles Per Instruction** — the average number of clock cycles each instruction takes.
Execution time is:

```
time = instruction_count × CPI × clock_period
     = instruction_count × CPI / clock_frequency
```

This one equation frames both interview questions. There are three levers: how *many* instructions,
how *many cycles each*, and how *fast the clock*. RISC and CISC trade these against each other.

### Q: "Compare CPI in CISC and RISC."

Naïvely, **RISC has a lower (better) CPI** because every instruction is simple and uniform, so a
clean pipeline approaches the ideal CPI ≈ 1. **CISC has a higher CPI** per *architected* instruction
because a single complex instruction does more work (and older CISC implementations even microcoded
them over many cycles).

*But* — and this is the point — CISC does the same *job* in **fewer instructions**. So comparing raw
CPI is misleading; you must compare `instruction_count × CPI` (total cycles), not CPI alone. And on a
modern x86, since instructions are cracked into µops, the meaningful metric is **µops per cycle /
IPC (instructions per cycle)** of the back-end, where x86 and ARM are broadly competitive.

### Q: "CISC code is smaller than RISC, yet RISC runs as fast as (or faster than) CISC — why?"

Because **code size and execution speed are different things**, and speed is dominated by how well
the pipeline stays full, not by how many bytes the program occupies:

1. **Fixed-length decode is pipeline-friendly.** A RISC fetcher knows every instruction is 4 bytes,
   so it can fetch and decode *several in parallel* trivially and never has to hunt for instruction
   boundaries. x86's variable length makes parallel decode genuinely hard — the decoder must
   determine where instruction *k* ends before it knows where *k+1* begins. Simpler decode → wider,
   deeper, higher-clock pipelines → more instructions retired per cycle.
2. **RISC's extra instructions are individually cheaper and overlap.** Three simple RISC instructions
   that each take ~1 pipelined cycle can retire in ~3 cycles of *throughput*; the one equivalent CISC
   instruction still has to do all that work internally (as µops) and doesn't magically finish
   sooner.
3. **Modern x86 neutralizes much of its own disadvantage** with µop caches and aggressive
   out-of-order back-ends — but it pays in front-end complexity, power, and silicon area.
4. **The cost of smaller code:** denser code is easier on the **instruction cache** (Module 04) —
   more of the program fits in L1i. So CISC's compactness *is* a real advantage for i-cache
   footprint; it just doesn't translate into raw per-instruction speed. HFT shops care about i-cache
   pressure precisely because a hot path that spills out of L1i stalls the front-end.

The honest interview answer: *"Code size affects i-cache footprint, not throughput. RISC's fixed-length
encoding makes the front-end simple and easy to widen, so it keeps the pipeline fuller; and modern x86
gets the best of both by decoding its CISC instructions into RISC-like µops and caching them. In
practice the ISA label matters far less than the microarchitecture."*

---

## Common pitfalls / misconceptions

- **"RISC has fewer instructions in the *program*."** No — RISC has a smaller *instruction set*, but
  its programs contain *more* instructions (load-store forces extra ops). Don't confuse set size with
  program size.
- **"CISC is obsolete / slow."** x86-64 dominates servers and desktops and is extremely fast; the
  CISC/RISC war ended in a synthesis (CISC ISA, RISC-style core).
- **"CPI alone tells you which is faster."** Never compare CPI without instruction count and clock —
  the `IC × CPI / freq` product is what matters.
- **"Registers are just fast memory."** They're addressed differently (by name in the instruction,
  not by a memory address), can't be pointed to, and are the ALU's direct inputs — that's why they're
  ~0-cost.
- **Confusing ISA with microarchitecture.** Two x86 chips share an ISA but have totally different
  performance because the microarchitecture differs.

---

## Quiz — tough problems

**Q1.** Program A on a CISC machine has 1.0 B instructions at CPI 2.5, clock 3 GHz. Program B (same
task) on a RISC machine has 1.6 B instructions at CPI 1.1, clock 3 GHz. Which is faster, and by how
much?
**Answer:** A: `1.0e9 × 2.5 / 3e9 = 0.833 s`. B: `1.6e9 × 1.1 / 3e9 = 0.587 s`. RISC is ~1.42× faster
*here* — despite 60% more instructions — because its CPI is so much lower. The lesson: instruction
count went up, total cycles went down. Compare `IC × CPI`, never CPI alone.

**Q2.** True or false: reducing instruction count always reduces execution time.
**Answer:** False. If the "fewer" instructions have a much higher CPI (e.g. a microcoded complex CISC
instruction taking 20 cycles), total cycles can rise. Time depends on the *product* `IC × CPI`, plus
second-order effects like i-cache and pipeline behavior.

**Q3.** Why can a RISC front-end decode 4 instructions per cycle far more easily than an x86
front-end?
**Answer:** Fixed 4-byte instructions mean boundaries are known without decoding — instruction *k*
starts at `base + 4k`, so all 4 can be decoded fully in parallel. x86's variable length (1–15 B)
means you can't locate instruction *k+1* until you've partly decoded *k*, making wide parallel decode
expensive (Intel uses complex length-predecoders and a µop cache to work around it).

**Q4.** What is a µop, and why does the µop cache matter for a hot loop?
**Answer:** A micro-operation is the internal RISC-like instruction x86 decodes each complex
instruction into; the RISC-style back-end executes µops. The µop cache stores already-decoded µops so
a tight loop replays them without re-running the expensive x86 legacy decoder every iteration — it
turns decode from a per-iteration cost into a one-time cost.

**Q5.** You have `cmp rax, rbx` immediately followed by `je label`. How does `je` "know" the result
of the comparison — they share no operand?
**Answer:** Through the **flags register (RFLAGS)**. `cmp` computes `rax - rbx` and sets the Zero flag
(and Sign, Carry, Overflow) without storing the difference. `je` ("jump if equal") reads the Zero
flag. The dependency is *implicit*, carried in the flags — a hidden data dependency the pipeline must
respect.

**Q6.** On a load-store RISC machine, how many instructions does `x = arr[i] + 5; arr[i] = x;`
(x, arr, i already in registers) take, minimum, and why?
**Answer:** Roughly three: a `load` from the computed address `arr + i*elem`, an `add` of the
immediate 5 in registers, and a `store` back. Arithmetic can't touch memory directly, so the two
memory accesses must be explicit load/store instructions surrounding the register-only add.

**Q7.** Two servers run the *identical* x86-64 binary; one is 2× faster on your hot loop. If the ISA
is identical, what changed?
**Answer:** The **microarchitecture** — pipeline width/depth, cache sizes, branch predictor quality,
µop cache size, clock speed, memory latency. The ISA is the software contract; performance is a
property of the implementation, not the instruction set.

**Q8.** Why is the von Neumann model called a "bottleneck," and name two hardware features that exist
to hide it.
**Answer:** Code and data share one memory and one bus, so the CPU (much faster than memory)
constantly waits on that shared path. Features that hide it: **caches** (keep hot code/data on-chip),
**pipelining/out-of-order execution** (do useful work while a memory access is outstanding),
**prefetchers** (fetch data before it's asked for), and the **split L1i/L1d** caches (so code fetch
and data access don't collide).

---

## Indian HFT interview questions

**Q: Compare CPI in CISC and RISC. (Optiver, Graviton — architecture warm-up)**
"RISC targets CPI ≈ 1 because instructions are simple, fixed-length, and pipeline cleanly; CISC has
higher CPI per architected instruction because each does more work (and legacy microcoded ops span
many cycles). But that's an unfair comparison — CISC needs fewer instructions for the same task, so
you must compare total cycles `IC × CPI`, not CPI. On modern x86 the point is moot: instructions are
decoded into µops and the meaningful figure is back-end IPC, where x86 and ARM are comparable."

**Q: CISC code is smaller but RISC is just as fast — explain. (Tower Research Gurgaon, Quadeye)**
"Code size is about *bytes* (i-cache footprint); speed is about *pipeline utilization*. RISC's
fixed-length encoding makes decode trivial and easy to parallelize/widen, so the pipeline stays
fuller and retires more instructions per cycle, which cancels out its larger instruction count.
Meanwhile x86 recovers most of its decode penalty with µop caches. Smaller CISC code does help
i-cache pressure — which HFT cares about — but that's a footprint win, not a throughput win."

**Q: What's the difference between ISA and microarchitecture? (IMC, AlphaGrep)**
"ISA is the visible contract — instructions, registers, memory model — that binaries target and that
stays stable for decades (x86-64, AArch64). Microarchitecture is how a specific chip implements that
ISA: pipeline depth, cache sizes, predictors, execution ports. Same ISA runs the same binary; the
microarchitecture decides how fast."

**Q: Why is register access effectively free compared to memory? (Graviton, NK Securities)**
"Registers are on-die storage wired directly into the execution units and named by the instruction
encoding, so there's no address computation, no cache lookup, no bus transaction — the value is
already where the ALU needs it. A memory access, even an L1 hit, costs ~4 cycles; DRAM is 200+. So
compilers fight to keep hot values in registers and minimize spills."

**Q: What is the von Neumann bottleneck and how does modern hardware fight it? (Jump, Squarepoint)**
"One shared memory/bus for code and data means the CPU is throttled by memory bandwidth/latency.
Hardware hides it with caches (L1/L2/L3), split L1i/L1d, hardware prefetchers, deep out-of-order
execution that keeps working while loads are outstanding, and store buffers. Low-latency code is
mostly the art of staying in cache so the bottleneck never bites."

**Q: Modern x86 is said to be 'CISC outside, RISC inside' — what does that mean? (Da Vinci, HRT)**
"The programmer-visible ISA is CISC (complex, variable-length x86 instructions), but the front-end
decodes each into one or more fixed-format RISC-like µops, and a RISC-style out-of-order back-end
executes those µops across multiple ports. A µop cache stores decoded µops so hot loops skip the
costly x86 decoder. So you get x86 compatibility and dense code with a RISC-efficient core."

**Q: Where does the flags register fit into control flow? (Mansard, Qube)**
"Comparison and arithmetic instructions set condition-code bits (Zero, Sign, Carry, Overflow) in
RFLAGS; conditional branches and `cmov` read them. It creates an implicit data dependency from a
`cmp` to the following `je`/`jne`, which the pipeline must honor and the branch predictor tries to
resolve early."

---

## Key takeaways

- A computer is a CPU that loops fetch→decode→execute over instructions stored in one flat memory
  (von Neumann); memory is slow, so the CPU is designed around *hiding* memory latency.
- Registers are on-chip, named, ~0-cost scratch slots; the memory hierarchy (register → L1 → L2 → L3
  → DRAM) spans ~0 to ~200+ cycles — the central fact of performance.
- The ISA is the hardware/software contract; microarchitecture is the implementation. Two x86 chips
  share an ISA but differ vastly in speed.
- CISC (x86) = complex, variable-length, memory operands, small code; RISC (ARM/RISC-V) = simple,
  fixed-length, load-store, bigger code but easy decode and low CPI.
- `time = IC × CPI / clock` — never compare CPI alone. RISC's low CPI can beat CISC despite more
  instructions.
- Modern x86 is CISC-ISA / RISC-core: it decodes complex instructions into µops (cached in a µop
  cache) and runs them on a RISC-style out-of-order back-end.
- Code size affects i-cache footprint, not raw throughput — a distinction HFT interviewers probe.

**Next:** [02 — Pipelining](02-pipelining.md) — how overlapping the fetch/decode/execute stages gets
us toward CPI ≈ 1, and the hazards that get in the way.
