# 11 — Optimizers

**Playlist videos 32–38.** Section 07 introduced vanilla gradient descent and its batch/stochastic/
mini-batch variants (see `../ml-guide` for the original derivation). This section is about a
different axis of improvement: not *how much data* each step looks at, but *how the step itself is
computed*. That's what an **optimizer** is — the update rule that turns a gradient into a weight
change. Seven videos, one continuous story: each optimizer below exists to fix a specific, named
weakness in the one before it.

---

## 1. The problem: plain gradient descent isn't good enough

Vanilla (S)GD's update rule is just:

```
w = w - η · ∇L(w)
```

One fixed learning rate `η`, applied uniformly to every parameter, moving directly along the current
gradient. In practice this runs into three recurring failure modes:

- **Ravines (narrow curved valleys in the loss surface).** The gradient points mostly across the
  ravine's steep walls, not along its shallow floor toward the minimum. GD **oscillates** back and
  forth across the walls while creeping slowly along the floor — visually, a jittery zig-zag instead
  of a straight run to the minimum.
- **Saddle points and plateaus.** Regions where the gradient is near zero but you're not at a
  minimum (a flat plateau, or a saddle where the surface curves up in one direction and down in
  another). GD has no signal to push through — it can crawl or stall for a long time.
- **One learning rate for every parameter is a poor fit.** Some parameters (weights tied to rare or
  sparse features) need large updates when they finally do get signal; others (weights that update
  on nearly every batch) need small, careful updates so they don't overshoot. A single global `η`
  can't serve both well — too large and the frequent parameters blow up, too small and the rare ones
  never learn.

**The fix, in general:** make the update rule smarter than "just follow the current gradient at a
fixed rate." Two independent improvements turn out to matter, and the rest of this section is
essentially "add one, then the other, then combine both":

1. **Momentum** — use *history* of past gradients to smooth the path and power through ravines/flat
   regions (Sections 3–4 below).
2. **Adaptive per-parameter learning rates** — give each parameter its own effective `η` based on how
   much it's moved historically (Sections 5–7 below).

Adam (Section 7) is literally "momentum + adaptive learning rates in one update rule," which is why
it's the default almost everywhere today.

---

## 2. EWMA — the mathematical primitive underneath everything below

Before momentum, AdaGrad, RMSProp, or Adam make sense, you need one tool: the **Exponentially
Weighted Moving Average**. It is not itself an optimizer — it's the smoothing mechanism that
Momentum, RMSProp, and Adam are all built from, so it's worth understanding in isolation first.

**The problem it solves:** given a noisy stream of numbers `x_1, x_2, x_3, ...` (e.g. daily
temperatures, or gradients at each training step), how do you get a smoothed "current average" that
updates cheaply with each new value, without storing the whole history?

**The update rule:**

```
v_t = β · v_{t-1} + (1 - β) · x_t
```

- `v_t` — the smoothed average after seeing `x_t`.
- `x_t` — the newest raw value.
- `β` (beta) — a number between 0 and 1 controlling how much weight the past carries.

In words: the new average is a weighted blend of *where the average already was* and *the newest
observation*. Unroll the recursion and you'll see it's a weighted sum over **all** past values, with
weights that decay exponentially the further back you go — hence "exponentially weighted."

**Intuition for β.** A useful rule of thumb: EWMA with parameter `β` behaves like a simple average
over roughly the last `1 / (1 - β)` values.

- `β = 0.9` → averages over roughly the last **10** values.
- `β = 0.98` → averages over roughly the last **50** values.
- `β = 0.5` → averages over roughly the last **2** values (barely smooths at all).

Higher `β` = longer memory = smoother but slower to react to real changes. Lower `β` = shorter
memory = noisier but tracks recent shifts faster. This is exactly the same trade-off as a moving
average window size — EWMA is just the recursive, O(1)-memory way to compute one.

**Bias correction.** If you initialize `v_0 = 0`, the first several `v_t` values are biased toward
zero (there's no history yet to average over, so the `(1-β)·x_t` term is fighting against a `v_0`
that hasn't "warmed up"). The standard fix, used inside Adam, is to divide by `(1 - β^t)`:

```
v̂_t = v_t / (1 - β^t)
```

Early on (`t` small), `1 - β^t` is small, so this scales `v_t` up to compensate for the cold start.
As `t` grows, `β^t → 0`, so `1 - β^t → 1` and the correction fades away — it only matters in the
first few steps.

Keep this one equation in your head — `v_t = β·v_{t-1} + (1-β)·x_t` — because every optimizer below
is just "apply EWMA to some quantity derived from the gradient."

---

## 3. SGD with Momentum

**What it EWMA-smooths:** the gradients themselves.

Instead of stepping directly along the current gradient, momentum keeps a running "velocity" — an
EWMA of past gradients — and steps along *that*:

```
v_t = β · v_{t-1} + (1 - β) · ∇L(w_t)      (velocity = EWMA of gradients)
w_t = w_{t-1} - η · v_t
```

(Some formulations drop the `(1-β)` scaling factor on the gradient term and fold it into `η`
instead — you'll see both conventions; the behavior is the same.) `β` is typically **0.9**.

**The physical analogy:** think of a ball rolling downhill. Plain GD is a ball with no inertia — it
reacts instantly and only to the current slope, so on a ravine wall it just bounces side to side.
Momentum gives the ball mass and inertia: it *accumulates* velocity in directions the gradient
consistently points, and that accumulated push:

- **Dampens oscillation across ravine walls** — the left-right components of the gradient partially
  cancel out over time (they alternate sign), while the consistent along-the-floor component keeps
  reinforcing itself and grows.
- **Accelerates through consistent-direction regions** — if 10 consecutive gradients all point
  roughly the same way, velocity builds up and the effective step size grows well past what a single
  gradient would give you.
- **Powers through small local minima, saddle points, and plateaus** — a ball with momentum doesn't
  stop the instant the slope flattens or briefly reverses; it coasts through on the velocity it's
  already built up, the same way you'd coast over a small bump if you were rolling fast enough.

The cost: momentum can **overshoot** the minimum, since velocity keeps carrying it forward even
after the local gradient says "slow down" — this is exactly what NAG fixes next.

---

## 4. Nesterov Accelerated Gradient (NAG)

**The weakness in plain momentum:** it computes the gradient at the *current* position, then blends
it with velocity — so the gradient term has no idea the velocity is about to carry the ball
somewhere else. It reacts to overshoot only *after* the fact, one step late.

**NAG's fix — look ahead first, then correct:** compute the gradient not at the current position,
but at the position momentum is *about to move you to* ("look-ahead point"), then use that
look-ahead gradient to correct the update:

```
w_lookahead = w_{t-1} - β · v_{t-1}                 (where momentum alone would take you)
v_t         = β · v_{t-1} + (1 - β) · ∇L(w_lookahead)   (gradient evaluated THERE)
w_t         = w_{t-1} - η · v_t
```

The intuition is a smarter version of the ball-rolling analogy: instead of blindly trusting your
current velocity, you peek ahead to where that velocity is carrying you, check the slope *there*,
and if it's now climbing back uphill (a sign you're about to overshoot), that correction gets
folded into the update *before* you actually move — not one lagging step later. This means NAG
reacts to an approaching minimum sooner and **overshoots less** than plain momentum, converging
faster and with less oscillation in practice, at essentially the same computational cost (one extra
gradient evaluation at a shifted point).

Momentum and NAG both still use one global learning rate `η` for every parameter — that's the
problem the next three optimizers attack.

---

## 5. AdaGrad — adaptive per-parameter learning rates

**The idea:** give each parameter its own effective learning rate, shrinking it for parameters that
have already received large/frequent updates and leaving it larger for parameters that have barely
moved. Track a running **sum of squared gradients** per parameter, and divide the learning rate by
its square root:

```
G_t = G_{t-1} + (∇L(w_t))²                    (elementwise, accumulated forever)
w_t = w_{t-1} - (η / √(G_t + ε)) · ∇L(w_t)
```

`ε` is a tiny constant (e.g. `1e-8`) just to avoid dividing by zero. `G_t` accumulates
**elementwise** — every parameter has its own `G`, so every parameter effectively has its own
learning rate that shrinks based on *its own* update history.

**Why this is genuinely useful:** for a parameter tied to a rare/sparse feature (say, a word that
appears in 1% of training examples), gradients are infrequent but often need to make a real dent when
they do arrive — AdaGrad keeps that parameter's `G` small and its effective learning rate large. For
a parameter that gets a nonzero gradient on nearly every batch (a common feature), `G` grows fast and
its effective learning rate shrinks, preventing it from swinging wildly. This is exactly the "one
learning rate is a poor fit for every parameter" problem from Section 1, solved directly — which is
why AdaGrad shines on sparse-feature problems (its original use case was NLP with sparse word
features).

**The fatal flaw:** `G_t` is a **monotonically increasing sum** — squared terms, added forever, never
decayed. Over a long training run, `G_t` keeps growing without bound for every parameter, which means
`η / √(G_t + ε)` keeps **shrinking toward zero** for every parameter, regardless of whether that
parameter still needs to learn. Training doesn't diverge — it just **grinds to a halt**, often well
before convergence, because the effective learning rate has decayed to something too small to make
progress. This single flaw is why AdaGrad is rarely used today, and it's the exact problem RMSProp
was designed to fix.

---

## 6. RMSProp

**The one-line fix:** replace AdaGrad's ever-growing *sum* of squared gradients with an **EWMA** of
squared gradients — the same primitive from Section 2, applied to `(∇L)²` instead of raw gradients.

```
S_t = β · S_{t-1} + (1 - β) · (∇L(w_t))²
w_t = w_{t-1} - (η / √(S_t + ε)) · ∇L(w_t)
```

`β` is typically **0.9** (sometimes written as `0.9` in Keras's `rho` parameter). Because this is an
EWMA rather than a running sum, `S_t` doesn't accumulate forever — it reflects roughly "the recent
squared-gradient magnitude" (per the `1/(1-β)`-window intuition from Section 2), so it can **grow or
shrink** as training progresses. That's the whole fix: same adaptive-per-parameter idea as AdaGrad,
same benefit for sparse features, but the effective learning rate no longer collapses to zero, so
training doesn't stall. This is a direct, deliberate application of Section 2's tool to Section 5's
problem.

---

## 7. Adam — Momentum + RMSProp

**The idea:** why choose between "smooth the direction with momentum" and "adapt the learning rate
per parameter with RMSProp" when you can EWMA both quantities and combine them? Adam (Adaptive
Moment Estimation) keeps two running EWMAs per parameter:

```
m_t = β1 · m_{t-1} + (1 - β1) · ∇L(w_t)          (1st moment — EWMA of gradients, = momentum)
s_t = β2 · s_{t-1} + (1 - β2) · (∇L(w_t))²       (2nd moment — EWMA of squared gradients, = RMSProp)

m̂_t = m_t / (1 - β1^t)                            (bias-corrected, per Section 2)
ŝ_t = s_t / (1 - β2^t)

w_t = w_{t-1} - (η / (√ŝ_t + ε)) · m̂_t
```

- `m_t` is exactly the momentum velocity from Section 3.
- `s_t` is exactly the RMSProp accumulator from Section 6.
- Both get bias-corrected (Section 2) because both start at 0 and would otherwise be biased low for
  the first several steps — this matters more for Adam than for plain momentum/RMSProp because
  `β1`/`β2` are typically high, so the cold-start bias is more pronounced.
- The final update divides the momentum-smoothed gradient direction (`m̂_t`) by the RMSProp-style
  adaptive scale (`√ŝ_t`) — so you get **direction smoothing and per-parameter step-size adaptation
  in the same update**, which is precisely the two-pronged fix Section 1 called for.

**Typical hyperparameters** (defaults that work almost everywhere and rarely need tuning):
`β1 = 0.9`, `β2 = 0.999`, `ε = 1e-8`, `η = 0.001`. This is why Adam is the de-facto default optimizer
for deep learning today — it inherits momentum's speed through ravines/plateaus and RMSProp's
per-parameter adaptivity, and it just works with default settings across a huge range of problems.

### Minimal Keras usage

```python
# String shorthand — uses Keras's default hyperparameters (lr=0.001, beta_1=0.9, beta_2=0.999)
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])

# Explicit object — use this when you want to tune the learning rate or betas
from tensorflow import keras

opt = keras.optimizers.Adam(learning_rate=0.001, beta_1=0.9, beta_2=0.999)
model.compile(optimizer=opt, loss='binary_crossentropy', metrics=['accuracy'])

# The other optimizers in this section are available the same way, for comparison:
keras.optimizers.SGD(learning_rate=0.01, momentum=0.9)                 # SGD + Momentum
keras.optimizers.SGD(learning_rate=0.01, momentum=0.9, nesterov=True)  # NAG
keras.optimizers.Adagrad(learning_rate=0.01)
keras.optimizers.RMSprop(learning_rate=0.001, rho=0.9)
```

---

## 8. Summary — the lineage and when to reach for each

**The build-up, one fix at a time:**

```
GD/SGD  →  + Momentum  →  + look-ahead (NAG)  →  AdaGrad  →  + EWMA (RMSProp)  →  + Momentum (Adam)
 (07)      fixes ravine    fixes momentum's     fixes one     fixes AdaGrad's      combines both
            oscillation     overshoot            LR-fits-all   LR-collapse          fixes: direction
                                                  problem                           + adaptive LR
```

| Optimizer | Key idea | What it fixes | When to use |
|---|---|---|---|
| **GD / SGD** | Step directly along the current gradient at a fixed `η` | — (baseline) | Simple/convex problems; rarely competitive on deep nets alone |
| **SGD + Momentum** | Step along an EWMA (velocity) of past gradients | Oscillation in ravines; stalling on plateaus/small local minima | When plain SGD is too slow/jittery; a solid, cheap default improvement |
| **NAG** | Evaluate the gradient at the look-ahead point momentum is about to reach, then correct | Momentum's overshoot | Same use cases as momentum, when overshoot/oscillation near the minimum is a problem |
| **AdaGrad** | Divide `η` per-parameter by `√(sum of squared gradients)` | One global `η` poorly fitting every parameter | Sparse features (e.g. classic NLP with bag-of-words); rarely used for long training runs |
| **RMSProp** | Same as AdaGrad but with an EWMA instead of an ever-growing sum | AdaGrad's learning-rate collapse to zero | Non-sparse deep nets needing adaptive LR without stalling; solid default, esp. for RNNs |
| **Adam** | EWMA of gradients (momentum) **+** EWMA of squared gradients (RMSProp), bias-corrected | Needing both smoothed direction *and* adaptive per-parameter LR at once | **The default choice** for most deep learning problems, most of the time |

## Key takeaways

- Plain gradient descent has three concrete weaknesses: it **oscillates in ravines**, it **stalls on
  plateaus/saddle points**, and **one global learning rate** is a poor fit for every parameter.
  Optimizers are just smarter weight-update rules that fix these.
- **EWMA** (`v_t = β·v_{t-1} + (1-β)·x_t`) is the shared mathematical primitive — a cheap, O(1)-memory
  smoothed running average, tuned by `β` (higher `β` ≈ averaging over more recent history, per the
  `1/(1-β)` rule of thumb), with bias correction (`/(1-β^t)`) to fix the cold start. Momentum, RMSProp,
  and Adam are all "apply EWMA to something."
- **Momentum** EWMA-smooths the gradients into a velocity → dampens ravine oscillation, accelerates
  through consistent directions, coasts through small flat regions. **NAG** improves on it by
  computing the gradient at the look-ahead point momentum is about to reach, correcting overshoot
  sooner.
- **AdaGrad** gives each parameter its own learning rate via an accumulated sum of squared gradients
  — great for sparse features, but the sum only grows, so the learning rate eventually collapses to
  zero and training stalls. **RMSProp** fixes this by using an EWMA of squared gradients instead of a
  running sum.
- **Adam** = Momentum's gradient EWMA (1st moment) + RMSProp's squared-gradient EWMA (2nd moment),
  both bias-corrected → adaptive per-parameter learning rate *with* momentum. Defaults
  (`β1=0.9, β2=0.999, ε=1e-8`) work almost everywhere, which is why it's the default optimizer choice
  for deep learning today.

**Next:** [12 — Hyperparameter Tuning](12-hyperparameter-tuning.md) — using Keras Tuner to search
over these optimizer choices (and other hyperparameters) systematically instead of by hand.
