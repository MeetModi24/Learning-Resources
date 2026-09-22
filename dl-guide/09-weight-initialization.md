# 09 — Weight Initialization

**Playlist videos 29–30.** What can go wrong if you initialize weights carelessly (all-zero,
too-small, too-large), why randomness is required to break symmetry, and the two standard fixes —
Xavier/Glorot (for sigmoid/tanh) and He (for ReLU) — that scale that randomness correctly.

---

## 1. Why initialization matters

Before the first gradient step is ever taken, someone has to pick the starting values for every
weight matrix. It's tempting to treat this as a minor bookkeeping detail — surely gradient descent
will sort it out? It won't, not reliably. The starting weights determine:

- **Whether training converges at all.**
- **How fast it converges**, if it does.
- **Whether gradients vanish or explode** as they propagate through a deep network — the exact
  failure mode from Section 06, except triggered here by a bad starting point rather than a bad
  activation choice.

Initialization, activation choice (Section 08), and normalization (Section 10) are three different
levers on the same underlying problem: keeping signal (activations going forward, gradients going
backward) at a *usable scale* as it moves through many layers. Get any one of them wrong and you can
reintroduce vanishing/exploding gradients even if you got the other two right.

---

## 2. What NOT to do

### 2.1 Initializing everything to zero (or any constant)

Set every weight in a layer to 0 (or to any single constant `c`). Every neuron in that layer now:

- computes the exact same pre-activation `z` from the same inputs,
- so produces the exact same output `a`,
- so receives the exact same gradient during backprop.

Every neuron in the layer updates **identically**, forever. A layer with `N` neurons behaves as if
it had exactly **one** neuron, just copied `N` times — it can never learn `N` different features. This
is the **symmetry problem**, and it's not fixed by training longer; the symmetry is exact and
gradient descent has no mechanism to break an exact tie between identical neurons. This is precisely
why weights are initialized **randomly** — random values break the symmetry so different neurons
start life computing different things and can diverge into different, useful features. (Biases can
still safely be initialized to zero — it's the weights, which control what each neuron responds to,
that must be randomized.)

### 2.2 Initializing too small

If weights are drawn from a distribution with very small variance (e.g. all values close to 0, like
`N(0, 0.01²)`), each layer's output activations shrink relative to its inputs. Multiply many small
numbers together across many layers and the signal — both the forward-pass activations and the
backward-pass gradients — shrinks geometrically toward zero. Deep layers end up with gradients too
small to produce any meaningful weight update: **vanishing gradients**, the same symptom as Section
06, now caused by the starting scale of the weights instead of a saturating activation.

### 2.3 Initializing too large

The opposite failure: draw weights from a distribution with too much variance, and pre-activations
grow layer over layer. Two things go wrong:

- Activations/gradients can blow up numerically — **exploding gradients**.
- If the activation is sigmoid or tanh, large pre-activations push into the **saturating** region of
  those curves (Section 08), where the local gradient is near zero anyway — so large weights can
  *also* produce vanishing gradients through a different mechanism (saturation rather than
  geometric shrinkage).

### 2.4 The actual goal

Neither too small nor too large — the target is to initialize weights so that the **variance of
activations (and of gradients) stays roughly constant as you move from layer to layer**. Too small
and variance shrinks toward zero going forward (or backward); too large and it grows without bound.
The two standard schemes below are both just principled ways of choosing that variance based on how
many units feed into (`fan_in`) and/or out of (`fan_out`) a layer, so that this "roughly constant
variance" property holds regardless of layer width.

---

## 3. Xavier / Glorot initialization

Designed for **sigmoid and tanh** — symmetric, saturating activations. The idea: scale the random
weights by the number of inputs and outputs of the layer so that variance is preserved as signal
passes through. Two common formulations, both used in practice:

- **Normal:** draw from `N(0, σ²)` with `σ² = 1 / fan_in` (or the more balanced
  `σ² = 2 / (fan_in + fan_out)`).
- **Uniform:** draw from `U(-limit, limit)` with `limit = sqrt(6 / (fan_in + fan_out))`.

**Intuition:** sigmoid and tanh are symmetric around zero and saturate at both ends. If you keep the
variance of each layer's pre-activations in a stable, moderate range, activations stay in the
*non-saturating* middle of the curve for longer, so gradients keep flowing instead of vanishing due
to saturation. Using `fan_in` alone stabilizes the forward pass; averaging `fan_in` and `fan_out`
balances stabilizing the forward pass (activations) against stabilizing the backward pass
(gradients), since the same weight matrix is used (transposed) in both directions.

---

## 4. He initialization

Designed for **ReLU and its variants** (Leaky ReLU, PReLU, ELU — Section 08). Same idea as Xavier,
different scale:

- **Normal:** draw from `N(0, σ²)` with `σ² = 2 / fan_in`.
- **Uniform:** draw from `U(-limit, limit)` with `limit = sqrt(6 / fan_in)`.

**Intuition:** ReLU zeroes out roughly half its inputs (everything negative). That halves the
variance of the activations that actually pass through, compared to a symmetric activation like
tanh. Xavier's scaling assumes a symmetric activation and doesn't account for that loss — used with
ReLU it under-scales, letting variance shrink layer after layer. He initialization compensates by
doubling the variance (the factor of **2** in `2 / fan_in`) exactly to offset ReLU zeroing out about
half the signal. This is why He, not Xavier, is **the practical default for ReLU-family networks**.

---

## 5. Practical guidance

| Activation in the layer | Recommended initializer | Keras `kernel_initializer` |
|---|---|---|
| Sigmoid, tanh | Xavier / Glorot | `'glorot_normal'` or `'glorot_uniform'` (Keras default) |
| ReLU, Leaky ReLU, PReLU, ELU | He | `'he_normal'` or `'he_uniform'` |
| SELU | LeCun (as noted in Section 08) | `'lecun_normal'` |

Both Xavier and He come in a **normal** and a **uniform** flavor — same target variance, different
sampling distribution. Neither flavor is consistently better than the other in practice; either is a
reasonable choice as long as it matches the activation.

Match the initializer to the activation actually used in that layer — mixing them (e.g. He weights
feeding into a tanh layer) reintroduces the mismatch each scheme was designed to avoid.

---

## 6. Minimal Keras code

```python
from tensorflow import keras
from tensorflow.keras import layers

model = keras.Sequential([
    layers.Dense(64, activation='relu', kernel_initializer='he_normal', input_shape=(20,)),
    layers.Dense(32, activation='relu', kernel_initializer='he_uniform'),
    layers.Dense(16, activation='tanh', kernel_initializer='glorot_normal'),
    layers.Dense(1, activation='sigmoid'),   # Keras default (glorot_uniform) is fine for the output layer
])
```

Keras already defaults `kernel_initializer` to `'glorot_uniform'` on `Dense` layers, which is why
older ReLU networks that never set an initializer explicitly can still train reasonably — but
setting `he_normal`/`he_uniform` explicitly on ReLU-family hidden layers is the correct, deliberate
choice rather than relying on a default tuned for a different activation.

---

## Key takeaways

- Initialization is not a minor detail — bad starting weights can stall convergence or reintroduce
  the vanishing/exploding gradient problem from Section 06 before a single useful gradient step is
  taken.
- **Never initialize all weights to zero or any constant** — every neuron in a layer computes the
  same thing and gets the same gradient forever (the symmetry problem); a layer of `N` neurons
  collapses to behave like one. Random initialization exists specifically to **break this symmetry**.
- **Too small →** vanishing signal/gradients; **too large →** exploding gradients, or saturation if
  the activation is sigmoid/tanh. The goal is to keep the **variance of activations and gradients
  roughly constant** across layers, which is what `fan_in`/`fan_out`-based scaling achieves.
- **Xavier/Glorot** (`1/fan_in` or `2/(fan_in+fan_out)`) is designed for sigmoid/tanh. **He**
  (`2/fan_in`) is designed for ReLU — the extra factor of 2 compensates for ReLU zeroing out about
  half the activations — and is the practical default for ReLU-family networks.
- Initialization, activation choice (Section 08), and batch normalization (Section 10) are three
  complementary fixes for the same underlying vanishing/exploding gradient problem (Section 06) —
  getting one wrong can undo the benefit of the other two.

**Next:** [10 — Batch Normalization](10-batch-normalization.md) — normalizing layer inputs during
training so the network stays well-scaled even as weights update, rather than relying solely on a
good starting point.
