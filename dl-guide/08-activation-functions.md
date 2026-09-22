# 08 — Activation Functions

**Playlist videos 27–28.** Why non-linearity is non-negotiable, the classic trio (sigmoid, tanh,
ReLU) and where each breaks, then the ReLU variants (Leaky ReLU, PReLU, ELU, SELU) built to patch
ReLU's one real flaw — the dying ReLU problem.

---

## 1. Why activation functions exist at all

A single neuron computes `z = w·x + b` and then applies an activation function `a = f(z)`. Strip
the activation out — or make it linear, `f(z) = z` — and something bad happens the moment you stack
layers.

Recall the MLP from Section 03: layer 2's input is layer 1's output. If every layer is just an affine
transform (`Wx + b`) with no non-linearity in between, composing them gives you *another* affine
transform:

```
f(x) = W2(W1 x + b1) + b2 = (W2 W1) x + (W2 b1 + b2) = W' x + b'
```

A 50-layer network with linear activations throughout is mathematically identical to **one** linear
layer. All that depth buys you nothing — you could only ever learn a linear decision boundary, exactly
like plain linear/logistic regression. Depth is only useful if each layer can bend the representation
in a genuinely new way, and that requires a **non-linear** activation between layers.

This is the whole point of this section: **non-linearity is what gives depth its power.** Every
activation function below is a different answer to "which non-linearity, and with what trade-offs."

---

## 2. Sigmoid

$$\sigma(x) = \frac{1}{1 + e^{-x}}$$

- **Range:** (0, 1).
- **Pros:** smooth and differentiable everywhere; output reads as a probability, which makes it the
  natural choice for a **binary classification output neuron** (Section 04).
- **Cons:**
  - **Saturates.** For large |x|, σ(x) flattens to ~0 or ~1 and the gradient σ'(x) = σ(x)(1-σ(x))
    goes to ~0. Chain that through many layers during backprop and gradients shrink to nothing —
    exactly the **vanishing gradient problem** from Section 06. Sigmoid hidden layers are the classic
    way to trigger it.
  - **Not zero-centered.** Output is always positive, so gradients flowing back through a layer of
    sigmoid units all push weights in a consistent sign direction, which slows convergence.
  - Computing `e^-x` is comparatively expensive.
- **Verdict:** rarely used in hidden layers today. Still standard for a binary output layer.

## 3. Tanh

$$\tanh(x) = \frac{e^x - e^{-x}}{e^x + e^{-x}}$$

- **Range:** (-1, 1), and it's just a rescaled sigmoid: `tanh(x) = 2σ(2x) - 1`.
- **Pros:** **zero-centered** — unlike sigmoid, outputs can be negative, so gradients aren't
  systematically biased in one direction. This alone made it a better default than sigmoid for
  hidden layers for a long time.
- **Cons:** it **still saturates** at both ends (flattens to ±1), so it still suffers vanishing
  gradients — just a bit less severely than sigmoid in practice.
- **Verdict:** better than sigmoid for hidden layers, but superseded by ReLU for the same underlying
  reason ReLU exists: saturation is still a problem.

## 4. ReLU (Rectified Linear Unit)

$$\text{ReLU}(x) = \max(0, x)$$

- **Range:** [0, ∞).
- **Pros:**
  - **No saturation for positive inputs** — the gradient is exactly 1 for any x > 0, no matter how
    large. This is the direct fix for vanishing gradients that sigmoid/tanh can't offer.
  - **Cheap.** Just a threshold at zero — no exponentials.
  - **Sparse activations.** Any neuron with negative input outputs exactly 0, so at any given time
    only a subset of neurons are "active" — a form of implicit regularization.
  - Converges faster in practice than sigmoid/tanh networks.
  - This combination is why ReLU is the **default activation for hidden layers** in modern
    feedforward and convolutional networks.
- **Cons:**
  - **Not zero-centered** (outputs are ≥ 0).
  - **The dying ReLU problem.** If a neuron's input lands in the negative region for a training
    example, ReLU outputs exactly 0 *and* its gradient is exactly 0 there too. If a large or badly
    scaled gradient update pushes a neuron's weights so that it ends up outputting a negative
    pre-activation for *every* training example, that neuron is stuck: it always outputs 0, its
    gradient is always 0, and no further update can ever revive it. It has permanently "died" and
    contributes nothing to the network for the rest of training.
- **Verdict:** the default choice for hidden layers unless you have a specific reason not to use it.

## 5. ReLU variants — fixing dying ReLU

All four variants below solve the same problem the same way: give the negative side of the function
a **non-zero gradient**, so a neuron that drifts into the negative region can still receive a
gradient signal and recover instead of dying.

### Leaky ReLU

$$f(x) = \begin{cases} x & x \ge 0 \\ \alpha x & x < 0 \end{cases} \quad (\alpha \text{ e.g. } 0.01)$$

Instead of flatlining at 0 for negative inputs, it leaks a small slope through (commonly α = 0.01).
The gradient for x < 0 is now α instead of 0 — small, but never fully dead. α is a fixed
hyperparameter you choose upfront.

### Parametric ReLU (PReLU)

Same shape as Leaky ReLU, but **α is learned** during training via backpropagation instead of being
fixed. This lets the network decide, per layer (or per channel), how much negative slope actually
helps — more flexible than Leaky ReLU, at the cost of one extra learnable parameter per unit.

### ELU (Exponential Linear Unit)

$$f(x) = \begin{cases} x & x \ge 0 \\ \alpha(e^x - 1) & x < 0 \end{cases}$$

For x < 0, ELU smoothly saturates toward `-α` instead of a straight line — it's continuous and
differentiable at x = 0 (no kink), and its outputs are closer to zero-centered than ReLU/Leaky ReLU
outputs, which tends to help convergence. Costs more compute per activation than ReLU because of the
exponential.

### SELU (Scaled ELU)

A scaled version of ELU (`SELU(x) = λ · ELU(x)` with specific constants for λ and α) with a
**self-normalizing property**: under the right conditions — specifically, weights initialized with
`lecun_normal` and a network built purely from Dense + SELU layers — the mean and variance of
activations tend to stay stable (converge toward mean 0, variance 1) as they propagate through
layers, without needing a separate Batch Normalization step (Section 10). This property only holds
under those specific conditions; drop them (e.g. mix in Dropout naively, or use a different
initializer) and the self-normalizing guarantee breaks down.

### Practical guidance

- **Default: plain ReLU.** It's cheap, fast, and works well in the overwhelming majority of cases.
- **If you suspect dying ReLU** (e.g. training accuracy plateaus early, or you inspect activations
  and see large fractions of dead neurons), try **Leaky ReLU** first — it's a one-line swap with
  negligible extra cost.
- **ELU** is worth trying when you want smoother, more zero-centered activations and can afford the
  extra compute.
- **SELU** is a specialist choice — only pays off in the specific self-normalizing setup (plain
  Dense network, `lecun_normal` init) described above; not a drop-in general replacement.
- **PReLU** is the most flexible but adds parameters and is more prone to overfitting on small
  datasets — reach for it after simpler options.

---

## 6. Summary table

| Function | Formula | Range | Pros | Cons |
|---|---|---|---|---|
| Sigmoid | 1/(1+e⁻ˣ) | (0, 1) | Smooth, probabilistic output | Saturates → vanishing gradients; not zero-centered; costly |
| Tanh | (eˣ−e⁻ˣ)/(eˣ+e⁻ˣ) | (-1, 1) | Zero-centered | Still saturates → vanishing gradients |
| ReLU | max(0, x) | [0, ∞) | No saturation (x>0), cheap, sparse, fast convergence | Dying ReLU; not zero-centered |
| Leaky ReLU | x if x≥0 else αx | (-∞, ∞) | Fixes dying ReLU cheaply | α is a manual hyperparameter |
| PReLU | x if x≥0 else αx (α learned) | (-∞, ∞) | α adapts to the data | Extra learnable params, can overfit |
| ELU | x if x≥0 else α(eˣ−1) | (-α, ∞) | Smooth, closer to zero-centered | More expensive (exponential) |
| SELU | λ·ELU(x), scaled | (-λα, ∞) | Self-normalizing (under conditions) | Conditions are narrow/fragile |

## 7. Hidden layers vs. output layer

Don't use the same activation everywhere — hidden layers and the output layer are solving different
problems:

- **Hidden layers:** default to **ReLU** (or a variant if dying ReLU is a concern). The job here is
  purely to introduce non-linearity and keep gradients flowing.
- **Output layer:** the activation must match what you're predicting (Section 04):
  - **Binary classification →** sigmoid (single neuron, output read as P(class=1)).
  - **Multiclass classification →** softmax (one neuron per class, outputs sum to 1).
  - **Regression →** linear (no activation, or `activation='linear'`) — you want the raw real-valued
    prediction, not something squashed into a bounded range.

---

## 8. Minimal Keras code

```python
from tensorflow import keras
from tensorflow.keras import layers

model = keras.Sequential([
    layers.Dense(64, activation='relu', input_shape=(20,)),   # hidden layer, plain ReLU
    layers.Dense(32, activation='elu'),                        # hidden layer, ELU
    layers.Dense(1, activation='sigmoid'),                     # binary output
])
```

Leaky ReLU and PReLU aren't plain strings you pass to `activation=` — they're layers of their own,
applied after a `Dense` layer with **no** activation set:

```python
model = keras.Sequential([
    layers.Dense(64),                 # no activation here
    layers.LeakyReLU(alpha=0.01),     # activation applied as its own layer
    layers.Dense(32),
    layers.PReLU(),                   # alpha is learned
    layers.Dense(1, activation='sigmoid'),
])
```

---

## Key takeaways

- Non-linear activations are what let depth matter at all — stacked linear layers collapse into one
  linear function, no matter how many you stack (ties directly to the MLP intuition in Section 03).
- Sigmoid and tanh both saturate, causing vanishing gradients (Section 06); tanh is at least
  zero-centered, sigmoid is not. Both are now mostly confined to output layers (sigmoid) or legacy
  use.
- ReLU is the modern default for hidden layers — no saturation on the positive side, cheap, fast
  convergence — but it can suffer **dying ReLU**: a neuron stuck outputting 0 forever once its
  pre-activation goes permanently negative.
- Leaky ReLU, PReLU, ELU, and SELU all fix dying ReLU by giving the negative side a non-zero
  gradient; they differ in whether that negative response is fixed, learned, smooth/exponential, or
  self-normalizing.
- Match the activation to the layer's job: **ReLU family for hidden layers**, and for the output
  layer — **sigmoid** (binary), **softmax** (multiclass), or **linear** (regression), per Section 04.

**Next:** [09 — Weight Initialization](09-weight-initialization.md) — why *how* you initialize
weights interacts directly with which activation you chose, and can cause the same vanishing/exploding
problems all over again if done carelessly.
