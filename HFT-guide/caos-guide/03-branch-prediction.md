# Module 03 — Branch prediction

A pipeline (Module 02) only stays full if the CPU knows what to fetch next — but a conditional branch
doesn't reveal its destination until it executes, several stages deep. Rather than stall on every
`if`, the CPU **guesses** the outcome and speculatively runs down the predicted path. Get it right
(and modern predictors are >95% accurate) and branches are nearly free; get it wrong and you flush
~15–20 cycles of work. This module explains how the guessing works, exactly what a mispredict costs,
the classic **"branch miss vs cache miss — which is slower?"** question, and why HFT hot paths are
written to be *predictable* or *branchless*.

---

## 1. Why a branch is a problem for a pipeline

Recall the pipeline fetches a new instruction every cycle from the address in the PC. A conditional
branch like `je target` needs two things the front-end doesn't have yet:

1. **Direction** — taken or not-taken? That depends on the flags, which may not be computed until the
   branch reaches **EX** (Module 01: `cmp` sets flags, `je` reads them).
2. **Target address** — if taken, *where* to? For a direct branch the target is encoded, but for an
   indirect branch (`jmp rax`, virtual call, `ret`) even the target is data-dependent.

So in the cycles *between* fetching the branch and resolving it, the fetch unit faces a dilemma: it
must fetch *something* every cycle to keep the pipeline full, but it doesn't know the right something.

```
cycle:      1   2   3   4   5
branch je : F   D   E   ...          ← direction known only at EX (cycle 3)
   ????   :     F   D   ...          ← what do we fetch in cycles 2,3 ?
```

Stalling until EX (a "fetch bubble") would waste `stages_to_resolve` cycles on *every branch* — and
branches are ~15–20% of all instructions. Unacceptable. Enter prediction: **guess** the direction and
target immediately at fetch, run speculatively, and fix up only if wrong.

---

## 2. Static prediction: the cheap first guess

The simplest schemes need no history:

- **Predict not-taken** — just keep fetching the fall-through instructions. Cheap; wrong for loops.
- **Predict taken** — assume every branch jumps. Needs the target early.
- **BTFN (Backward-Taken, Forward-Not-taken)** — the clever static heuristic: **backward** branches
  (negative offset) are predicted *taken*, **forward** branches predicted *not-taken*. Why? A loop's
  back-edge branches backward and is taken on every iteration but the last — so "backward = taken" is
  right ~(n−1)/n of the time. Forward branches are often error/exit paths taken rarely, so
  "forward = not-taken" is a good default.

Static prediction is used as a fallback when dynamic history isn't available yet (a branch seen for
the first time). It's decent, but real workloads need history.

---

## 3. Dynamic prediction: learning from the past

Dynamic predictors record how each branch behaved recently and bet it'll behave the same way. This is
where the accuracy comes from.

### 3a. 1-bit predictor — remember the last outcome

Store one bit per branch: "last time, taken or not?" Predict the same again.

Problem: it mispredicts **twice** per loop. Consider a loop that iterates 10 times (branch taken 9×,
not-taken once at exit). On the exit iteration the bit flips to "not-taken." On the *next entry* to
the loop, the first branch is now mispredicted (bit says not-taken, but it's taken), then flips back.
So a loop nested in an outer loop mispredicts on entry *and* exit every time — the 1-bit predictor is
too twitchy.

### 3b. 2-bit saturating counter — the workhorse

Add hysteresis: use a 2-bit counter (four states) so a *single* anomaly doesn't flip the prediction.
You must be wrong **twice in a row** to change your bet.

```
        taken            taken            taken
   ┌──────────▶┌──────────▶┌──────────▶┌
 ┌─┴─┐  not   ┌┴──┐  not  ┌┴──┐  not  ┌┴──┐
 │ 00│◀───────│01 │◀──────│10 │◀──────│11 │
 │Strong      │Weak       │Weak       │Strong
 │ N-T │      │ N-T │     │ Taken│    │ Taken│
 └───┘        └───┘       └───┘       └───┘
  predict      predict     predict     predict
  NOT-taken    NOT-taken   TAKEN       TAKEN

 00 = strongly not-taken   10 = weakly taken
 01 = weakly not-taken     11 = strongly taken
 Taken outcome → count up (saturate at 11); not-taken → count down (saturate at 00).
 Prediction = high bit (states 10/11 → taken; 00/01 → not-taken).
```

Now the loop-exit anomaly only nudges a "strongly taken" counter to "weakly taken" — the prediction is
*still taken*, so the next loop entry predicts correctly. A 2-bit counter gets a simple loop to just
**one** mispredict (the exit), not two. This is the baseline dynamic predictor; a table of these
counters indexed by branch-address bits is a **bimodal predictor**.

### 3c. Correlating / two-level predictors — patterns across branches

Many branches are correlated: `if (x) …; if (!x) …;` — the second outcome is perfectly predictable
from the first. A bimodal predictor can't see that. **Two-level (correlating) predictors** add a
**history register** (a shift register of the last *h* branch outcomes, the *global history*) and use
`(branch address ⊕ history)` to index the counter table. This lets the predictor learn "when the last
3 branches went T,N,T, this branch is taken." **gshare** is the classic scheme; modern CPUs use
sophisticated hybrids (e.g. **TAGE**, which keeps multiple history-length tables and picks the
best-matching one). Accuracy on typical code exceeds **95–98%**.

- **Global history** — one shared history of all recent branches (captures cross-branch correlation).
- **Local history** — per-branch history (captures a single branch's own pattern, e.g. a branch that
  goes T,T,N,T,T,N…).
- Real predictors combine both (tournament/hybrid predictors pick whichever is more accurate per
  branch).

### 3d. The Branch Target Buffer (BTB) and Return Address Stack (RAS)

Direction is half the problem; you also need the **target address** *at fetch time*. The **BTB** is a
small cache mapping `branch instruction address → predicted target address`, so the moment the fetch
unit sees a known branch, it can redirect fetch to the predicted target the very next cycle. For
**indirect** branches (`jmp rax`, virtual dispatch) the BTB predicts the most likely target; a
mispredicted indirect branch is a real cost for polymorphic C++ (a reason HFT avoids `virtual` on hot
paths — see the C++ guide's inheritance module). For **returns**, a dedicated **Return Address Stack**
predicts `ret` targets by mirroring the call stack — near-perfect since calls and returns nest.

---

## 4. The misprediction penalty

When the branch finally resolves and the guess was wrong, every speculatively-fetched instruction on
the wrong path must be **squashed** (discarded) and the front-end **redirected** to the correct PC and
refilled from scratch.

```
correct path fetched speculatively... then at resolve: WRONG
  ┌────────────── wrong-path work, all squashed ──────────────┐
  F  D  E  M  W   ← these ~15 in-flight instructions thrown away
                  ▲
             branch resolves here → flush → refetch correct path from empty pipe
```

The penalty ≈ the **pipeline depth from fetch to branch-resolution** — on modern x86 that's roughly
**15–20 cycles** (~5–7 ns at 3 GHz). Every mispredict is a full pipeline refill.

**Why it dominates on branchy code:** with a 95% accurate predictor and branches ~20% of
instructions, the mispredict rate per instruction is `0.20 × 0.05 = 1%`. At ~15 cycles each, that adds
`0.15` to CPI — a 15% slowdown from branches alone. On unpredictable data (e.g. branching on random
input), accuracy can crater to 50%, turning branches into a catastrophe.

---

## 5. Branch miss vs cache miss — which is slower?

A favourite interview question, and the honest answer is *"it depends, and you should compare on two
axes: per-event cost and frequency."*

**Per-event cost:**

```
branch mispredict :  ~15–20 cycles   (pipeline refill)
L1 miss / L2 hit  :  ~12 cycles
L2 miss / L3 hit  :  ~40 cycles
L3 miss / DRAM    :  ~200–300 cycles  (~60–100 ns)
```

So a **single DRAM cache miss (~200–300 cycles) is far more expensive than a single branch mispredict
(~15–20 cycles)** — often 10–20× worse. On the "which one stall is bigger" reading, **the cache miss
wins**.

**But frequency and hideability matter:**

- Branch mispredicts can be *very frequent* (a branch every ~5 instructions) and are hard for
  out-of-order execution to hide, because a wrong direction poisons *all* subsequent speculative work.
- A cache miss, though individually huge, can be *partly hidden* by out-of-order execution and
  memory-level parallelism (Module 02) if there's independent work to overlap — and hardware
  prefetchers often fetch the line before you need it.
- Conversely, a mispredicted branch that *guards* a load creates both problems at once.

**The interview-grade answer:** *"Per event, a DRAM cache miss (~200+ cycles) is much costlier than a
branch mispredict (~15–20 cycles). But branch mispredicts are more frequent and harder to hide with
OoO execution, and a cache miss can often be overlapped or prefetched. So 'slower' depends on the
workload: I profile which one dominates. On HFT hot paths I attack both — keep the working set in
cache to avoid misses, and make branches predictable or branchless to avoid flushes."*

---

## 6. Writing predictable and branchless code

Since you can't make the predictor smarter, you make the code easier to predict — or remove the branch:

- **Make branches predictable.** Sort data before a filtering loop; a branch on sorted data is
  almost always taken-then-not-taken (predictor nails it), while the same branch on shuffled data
  mispredicts ~50% of the time. (The famous "why is processing a sorted array faster" result.)
- **Branchless / predication.** Replace a control-flow branch with a data operation the CPU always
  executes: `cmov` (conditional move), arithmetic, or bit tricks. Example — clamp without a branch:
  ```cpp
  // branchy:                         // branchless (compiler emits cmov):
  int m = (a < b) ? a : b;            int m = a + ((b - a) & -(b < a));   // min via bit trick
  ```
  Branchless has *no* mispredict penalty but *always* does the work — a win only when the branch is
  unpredictable. A *well-predicted* branch is often faster than branchless because the not-taken side
  is skipped for free.
- **Hint the compiler.** `[[likely]]`/`[[unlikely]]` (C++20) and `__builtin_expect(cond, 0)` bias
  code layout so the hot path is the fall-through (better for the front-end / static prediction and
  i-cache), and steer branch-hinting. They tune *layout and static hints*, not the dynamic predictor
  directly.
- **Remove indirect branches on the hot path.** Virtual calls and function-pointer tables are
  BTB-predicted indirect branches; a mispredict costs the full penalty. HFT code favors CRTP,
  `if constexpr`, or `std::variant` dispatch over `virtual` (see the C++ guide, module 17).

The governing rule: **an unpredictable branch on the hot path is a latency bug.** Either make it
predictable (sort, restructure) or eliminate it (branchless, table lookup).

---

## Common pitfalls / misconceptions

- **"Branches are always expensive."** A *well-predicted* branch is nearly free (~0–1 cycle). Only
  *mispredicts* cost. The enemy is *unpredictability*, not branching per se.
- **"Branchless is always faster."** Branchless removes the mispredict risk but always executes both
  sides' work; for a highly predictable branch it can be *slower*. Measure.
- **"A cache miss and a branch miss are the same order of cost."** No — a DRAM miss (~200+ cycles) is
  ~10× a branch mispredict (~15–20 cycles) per event. They differ in frequency and hideability too.
- **"`[[likely]]` speeds up the predictor."** It guides code layout and static hints; the dynamic
  predictor learns from runtime behavior regardless.
- **"Virtual calls are fine because the BTB predicts them."** Monomorphic virtual calls predict well;
  *polymorphic* ones (target varies) mispredict and cost the full penalty — hence HFT's aversion.
- **Forgetting speculation's security angle.** Speculative execution down a mispredicted path can leave
  microarchitectural traces (Spectre/Meltdown) — worth *naming* even if you don't detail it.

---

## Quiz — tough problems

**Q1.** A tight loop runs 1000 iterations. How many mispredicts does a 1-bit predictor make vs a 2-bit
saturating counter (assume the loop is entered once)?
**Answer:** The loop branch is taken 999×, not-taken once (exit). A **2-bit** counter mispredicts only
the final exit → **1 mispredict**. A **1-bit** predictor also mispredicts the exit; if the loop is
entered only once that's just 1 too — but *nested* in an outer loop it mispredicts on both exit and
re-entry each time → **2 per outer iteration**. The 2-bit's hysteresis is what saves the re-entry.

**Q2.** Estimate the CPI contribution of branches: 20% of instructions are branches, predictor
accuracy 96%, mispredict penalty 16 cycles.
**Answer:** Mispredict rate per instruction = `0.20 × 0.04 = 0.008`. Added CPI = `0.008 × 16 = 0.128`.
So branches add ~0.13 to CPI — roughly a 13% slowdown versus a perfect predictor.

**Q3.** Two versions of the same filter loop over an array: one over sorted data, one over shuffled
data. Same instructions, same cache behavior. Why is sorted ~5× faster?
**Answer:** Branch prediction. On sorted data the `if (x < threshold)` branch is taken for a long run
then not-taken for the rest — the predictor is right ~100% of the time. On shuffled data it's
essentially a coin flip, ~50% mispredict, and each mispredict costs a ~15–20-cycle flush. Data layout
changed predictability, not the algorithm.

**Q4.** Per event, rank: L1 hit, branch mispredict, DRAM access. And which is easiest for the CPU to
hide?
**Answer:** Cost: L1 hit (~4 cyc) < branch mispredict (~15–20 cyc) < DRAM (~200+ cyc). Easiest to hide:
the **DRAM access**, via out-of-order execution overlapping independent work and hardware prefetch. A
**branch mispredict** is hardest to hide because it invalidates all downstream speculative work.

**Q5.** Why is a mispredicted `ret` rare but a mispredicted polymorphic virtual call common?
**Answer:** Returns are predicted by the **Return Address Stack**, which mirrors the nesting of
calls/returns almost perfectly. A polymorphic virtual call is an **indirect** branch whose target
(the actual overriding function) varies at runtime by object type; the BTB can only cache one/few
targets, so when the type varies the target mispredicts.

**Q6.** When is branchless code (cmov) *slower* than the branchy version?
**Answer:** When the branch is **highly predictable**. A predicted branch skips the untaken side's work
for ~free, while `cmov` unconditionally computes both inputs and has a data dependency on the
condition (it can't be speculated past). So for predictable branches, branchy wins; branchless only
wins when misprediction would otherwise be frequent.

**Q7.** You add `__builtin_expect(rare_error, 0)` around an error check. What actually changes in the
generated code, and does it help the dynamic predictor?
**Answer:** It biases **code layout** — the compiler moves the unlikely error-handling block out of
line (off the hot fall-through path), improving i-cache density and front-end fetch on the common path,
and sets static branch hints. It does **not** train the dynamic predictor (that learns from runtime
outcomes); it optimizes layout and the first-encounter/static guess.

**Q8.** Explain how a 2-bit saturating counter handles a branch pattern of T,T,T,N,T,T,T,N,… (period-4).
**Answer:** Mostly counting up toward "strongly taken," the lone N nudges it down one state (to "weakly
taken") but not across the prediction boundary, so it keeps predicting taken and mispredicts *only the
N* each period — 1 miss per 4 branches. A correlating/history predictor could learn the period-4
pattern and predict the N too, approaching 0 misses.

**Q9.** Why do very deep pipelines make branch prediction *more* important?
**Answer:** The mispredict penalty ≈ pipeline depth (all the wrong-path in-flight work flushed). A
deeper pipeline runs a higher clock but flushes more cycles per mispredict, so the *same* mispredict
rate costs more wall-clock. Deep-pipeline designs therefore invest heavily in predictor accuracy to
keep the (rate × penalty) product low.

---

## Indian HFT interview questions

**Q: Explain branch prediction in depth. (Tower Research Gurgaon, Optiver — very common)**
"A conditional branch's direction and target aren't known until it executes deep in the pipeline, but
the fetch unit must fetch every cycle. So the CPU predicts: statically (BTFN — backward taken, forward
not-taken) for unseen branches, then dynamically once it has history. The dynamic core is a table of
2-bit saturating counters indexed by branch address (bimodal), upgraded to correlating/two-level
predictors (gshare, TAGE) that use global/local branch history to catch cross-branch patterns —
reaching >95% accuracy. Targets come from the BTB (indirect) and the Return Address Stack (returns). On
a correct prediction the branch is ~free; a mispredict flushes ~15–20 cycles."

**Q: Branch miss vs cache miss — which is slower? (Graviton, Quadeye, IMC — classic)**
"Per event, a DRAM cache miss (~200–300 cycles) is far costlier than a branch mispredict (~15–20
cycles) — roughly 10–20×. But branch mispredicts are more frequent and harder for out-of-order
execution to hide, since a wrong direction poisons all downstream speculation, whereas a cache miss can
be overlapped with independent work and prefetched. So the honest answer is workload-dependent: I'd
profile which dominates. On the hot path I minimize both — stay in cache, keep branches predictable or
branchless."

**Q: Why is iterating a sorted array faster than a shuffled one for the same code? (AlphaGrep, Jump)**
"Pure branch prediction. A threshold branch over sorted data is a long run of taken then a run of
not-taken — perfectly predictable. Over shuffled data it's ~50/50, so ~half the branches mispredict at
~15–20 cycles each. Same instructions, same cache footprint; predictability alone gives a multiple-x
difference."

**Q: How would you eliminate an unpredictable branch on the hot path? (HRT, Da Vinci)**
"If the branch is data-dependent and unpredictable, make it branchless: `cmov`, arithmetic, or bit
tricks so the CPU always executes a fixed sequence with no direction to mispredict; or replace control
flow with a table lookup. If the branch is *predictable*, I'd leave it — a predicted branch is cheaper
than always doing both sides. For dispatch I avoid polymorphic `virtual` (indirect-branch mispredicts)
in favor of CRTP/`if constexpr`/`variant`. And I'd sort/partition data upstream so branches become
predictable."

**Q: What does `[[likely]]`/`__builtin_expect` actually do? (Squarepoint, NK Securities)**
"They're layout and static-hint tools, not dynamic-predictor controls. They tell the compiler which
path is hot so it keeps that path as the straight-line fall-through and moves cold code (error
handlers) out of line — better i-cache density and front-end behavior — and set architectural branch
hints for the first/static prediction. The runtime dynamic predictor still learns from actual
outcomes."

**Q: Why does HFT code avoid virtual functions on the hot path, in branch-prediction terms? (Mansard, Qube)**
"A virtual call is an indirect branch resolved through the vtable; its target is predicted by the BTB.
If the call site is monomorphic it predicts well, but a polymorphic site (target varies by object type)
mispredicts and eats the full ~15–20-cycle flush plus the vtable load. On a nanosecond-budget hot path
that's unacceptable, so we use compile-time dispatch (CRTP, `if constexpr`, templates) that the compiler
resolves to a direct, inlinable call — no indirect branch, no mispredict."

**Q: What is speculative execution and what's its downside? (IMC, Optiver — follow-up)**
"The core executes instructions down the *predicted* branch path before the branch resolves, squashing
them if wrong — this is what makes prediction pay off. Downsides: wasted energy/work on mispredicts,
and security — speculatively-executed wrong-path instructions can leave microarchitectural side effects
(cache state) that Spectre/Meltdown-class attacks read out, which is why mitigations sometimes cost
performance."

---

## Key takeaways

- A branch's direction/target resolve deep in the pipeline, so the CPU **predicts** and runs
  speculatively to avoid stalling the front-end on every `if`.
- **Static:** BTFN (backward-taken, forward-not-taken). **Dynamic:** 2-bit saturating counters
  (hysteresis beats the twitchy 1-bit), upgraded to correlating/two-level (gshare, TAGE) using
  global/local history — >95% accurate. **Targets:** BTB (indirect) and Return Address Stack (returns).
- A **mispredict costs ~15–20 cycles** (full pipeline refill); at ~20% branches and 96% accuracy that's
  already ~13% of CPI, and it collapses on unpredictable data.
- **Branch miss vs cache miss:** per event the DRAM miss (~200+ cycles) is much bigger, but branch
  misses are more frequent and harder to hide — the "which is slower" answer is workload-dependent.
- Make hot-path branches **predictable** (sort/partition data) or **branchless** (cmov, bit tricks,
  table lookup); branchless only wins when the branch would otherwise mispredict often.
- Avoid polymorphic **indirect branches** (virtual calls) on the hot path — they mispredict; prefer
  compile-time dispatch. `[[likely]]`/`__builtin_expect` tune layout/static hints, not the dynamic
  predictor.

**Next:** [04 — Caches & the memory hierarchy](04-caches-memory-hierarchy.md) — the L1/L2/L3 machinery
that keeps the pipeline fed and turns the ~200-cycle DRAM wall into a ~4-cycle L1 hit most of the time.
