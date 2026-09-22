# 10 — Batch Normalization

**Playlist video 31.** *Batch Normalization in Deep Learning | How Batch Normalization works.* A
single technique, but one that changed how every deep network since is trained: normalize the
inputs to a layer, per mini-batch, then let the network learn how much of that normalization it
actually wants.

---

## 1. The problem: internal covariate shift

Train a deep network and watch what happens to, say, layer 5's inputs. Layer 5 receives the output
of layer 4. But layer 4's weights are being updated every step by gradient descent — so the
*distribution* of what layer 5 receives (its mean, its spread) keeps shifting as training
progresses. Layer 5 isn't learning to map a fixed input distribution to a target; it's chasing a
moving one, produced by every layer beneath it updating simultaneously.

This was named **internal covariate shift (ICS)**: the change in the distribution of a layer's
inputs caused by updates to the parameters of the *preceding* layers. The deeper the network, the
worse this compounds — a small shift in layer 1's output distribution amplifies as it propagates
through layers 2, 3, 4... By the time it reaches a late layer, that layer's inputs are on a
completely different scale than they were a few steps ago. The layer has to constantly re-adapt to
a target that keeps moving, which:

- slows down training (each layer wastes capacity re-adjusting instead of learning the actual
  mapping),
- forces the use of small learning rates and careful initialization to avoid the whole thing
  diverging (the initialization sensitivity from Section 09, and the vanishing/exploding gradient
  dynamics from Section 06, are both aggravated by this),
- makes deep networks fragile to train in practice.

Batch Normalization (BN), introduced by Ioffe & Szegedy (2015), was proposed as the fix: force each
layer's inputs back to a stable, well-behaved distribution — every mini-batch — so downstream
layers see roughly the same statistics throughout training.

**An honest caveat.** This ICS story is the original motivation and it's still the intuition most
people reach for. But later research (Santurkar et al., 2018) ran controlled experiments and argued
BN's benefit has little to do with reducing covariate shift — in fact they showed BN networks can
have *more* internal covariate shift by some measures and still train better. Their explanation is
that BN **smooths the loss landscape** (makes it more predictably curved, i.e. improves the
Lipschitz properties of the loss and its gradients), which is what actually permits larger learning
rates and faster, more stable convergence. The practical recipe below is identical either way — only
the *why* is contested. Mention both when asked "why does BN work," since the field hasn't fully
settled it.

---

## 2. What it does: normalize per mini-batch, then rescale

For a given layer, BN takes the values flowing through it for the *current mini-batch* and
standardizes them to zero mean, unit variance — the same z-score idea you already know from
feature scaling — then applies a learnable linear transform on top. Four steps, in order, for a
mini-batch of size `m` and one feature/activation `x`:

**Step 1 — mini-batch mean:**

```
μ_B = (1/m) Σ x_i
```

**Step 2 — mini-batch variance:**

```
σ²_B = (1/m) Σ (x_i − μ_B)²
```

**Step 3 — normalize:**

```
x̂_i = (x_i − μ_B) / sqrt(σ²_B + ε)
```

`ε` is a tiny constant (e.g. `1e-3` in Keras' default) added purely for numerical stability — it
stops division by zero when a batch happens to have near-zero variance for that unit.

**Step 4 — scale and shift:**

```
y_i = γ · x̂_i + β
```

`γ` (scale) and `β` (shift) are **learnable parameters**, one pair per feature/channel, trained by
backpropagation exactly like weights and biases — gradients flow through the normalization step
into them.

### Why gamma and beta matter — don't skip step 4

If BN stopped at step 3, every layer's inputs would be forced to exactly zero mean and unit
variance, always. That's a strong, rigid constraint — sometimes the network's optimum genuinely
needs inputs shifted or scaled differently (for example, a sigmoid or tanh activation might want
inputs with some deliberate spread to use its non-saturating range effectively, not squashed to
unit variance). Without a way to undo the normalization, BN would be actively removing
representational power the network relies on.

`γ` and `β` give the network an escape hatch: gradient descent can learn `γ = sqrt(σ²_B + ε)` and
`β = μ_B` if the best answer is "don't normalize this one, thanks" — recovering the original
distribution exactly. In practice it learns something in between, whatever scale and shift actually
helps that layer. This is why BN is a normalization *followed by* a learned affine transform, not
normalization alone — the learnable part is what makes it safe to insert almost anywhere.

---

## 3. Where it goes, and the batch-size dependency

**Placement.** BN is inserted between the layer's linear transform and its activation function —
i.e. `Dense/Conv → BatchNorm → Activation`. The intuition: normalize the raw pre-activation values
(`z = Wx + b`) before they hit a nonlinearity, so the activation always sees inputs on a consistent
scale, which is exactly the ICS problem BN targets. (Applying BN after the activation is also seen
in practice and works reasonably well too — the original paper places it before; either is a minor
architectural choice, but before-activation is the common default and the one to reach for first.)

**Batch-size dependency.** Because `μ_B` and `σ²_B` are computed *from the current mini-batch*, BN's
quality depends directly on batch size. A batch of 256 gives a reasonably stable estimate of the
true population mean/variance for that layer's inputs. A batch of 4 gives a noisy, high-variance
estimate — the normalization statistics jitter wildly from step to step, which can hurt rather than
help training. This is the practical reason BN is known to struggle with very small batch sizes
(and part of why alternatives like Layer Normalization, used in Transformers — Section 20 — compute
statistics per-sample instead of per-batch, sidestepping this dependency entirely).

---

## 4. Training vs. inference — the gotcha

At training time, every mini-batch has its own `μ_B`, `σ²_B`, computed on the fly. That's fine
because there's always a batch present.

At **inference time** this breaks down: you might be predicting on a single example, so there's no
"batch" to compute statistics from — and even if you batch requests, you don't want a prediction to
depend on which other examples happen to be in the same batch as it (that would make output
non-deterministic and batch-composition-dependent, which is clearly wrong for a deployed model).

The fix: during training, BN also maintains a **running (exponential moving) average** of the mean
and variance it has seen across all mini-batches:

```
running_mean = momentum · running_mean + (1 − momentum) · μ_B
running_var  = momentum · running_var  + (1 − momentum) · σ²_B
```

At inference, BN switches to using these accumulated `running_mean` / `running_var` instead of
computing fresh batch statistics — giving deterministic, batch-independent predictions. This
training/inference mode switch is automatic in Keras (`model.fit` vs. `model.predict`/`evaluate`),
but it's the single most common source of confusing BN bugs when people hand-roll training loops or
freeze layers incorrectly — forgetting to put a model in inference mode leaves it using batch
statistics on data it was never meant to see, or (worse) leaves the running statistics frozen at
their random initial values if training never ran long enough to accumulate them.

---

## 5. Why it helps — tying back to earlier sections

- **Faster, more stable convergence, higher learning rates.** Whether you attribute it to reduced
  ICS or a smoother loss landscape (Section 1's caveat), the empirical result is the same: BN
  networks tolerate learning rates several times larger than unnormalized ones and converge in
  fewer epochs.
- **Less sensitivity to weight initialization.** Section 09 covered how a bad initial scale for `W`
  can blow up or shrink activations layer by layer before training even starts. BN re-normalizes
  after every layer regardless of how the incoming weights were scaled, which meaningfully lowers
  the stakes of getting initialization exactly right (though good initialization is still worth
  doing — BN reduces the penalty for imperfection, it doesn't make initialization irrelevant).
- **Mitigates vanishing/exploding gradients.** Section 06's vanishing/exploding gradient problem is
  driven largely by activations (and their gradients) growing or shrinking multiplicatively across
  layers. By repeatedly re-centering and re-scaling activations to a consistent range at every BN
  layer, the signal is kept in a well-behaved zone as it propagates, which keeps gradients from
  compounding out of control across depth.
- **A mild regularization effect.** Because `μ_B`/`σ²_B` are computed from a *randomly sampled*
  mini-batch rather than the true population, each example gets normalized slightly differently
  depending on which other examples land in its batch — this injects a small amount of noise into
  training, similar in spirit to what Dropout (Section 07) does deliberately. It's not a substitute
  for dropout in general, but in practice networks with BN often need less dropout (or none) to
  reach the same generalization, since some of dropout's job is already being done for free.

---

## 6. Minimal Keras usage

```python
from tensorflow import keras
from tensorflow.keras.layers import Dense, BatchNormalization, Activation

model = keras.Sequential([
    Dense(128, input_shape=(784,)),   # linear transform first (no activation here)
    BatchNormalization(),             # normalize the pre-activations
    Activation('relu'),               # activation last

    Dense(64),
    BatchNormalization(),
    Activation('relu'),

    Dense(10, activation='softmax'),
])

model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
```

`BatchNormalization()` handles all four steps and the train/inference statistics switch
automatically — `momentum` (default `0.99`) and `epsilon` (default `1e-3`) are its two most relevant
constructor arguments if you ever need to tune them.

---

## Key takeaways

- BN was proposed to fix **internal covariate shift** — the constant reshuffling of a layer's input
  distribution as earlier layers' weights update — though later work argues the real benefit is a
  **smoother loss landscape**; the mechanism is undisputed even if the "why" isn't fully settled.
- Four steps per mini-batch, per feature: **mean → variance → normalize (÷ by std, with ε for
  stability) → scale & shift** with learnable `γ`, `β`.
- `γ`/`β` are essential, not optional polish — without them, BN would force every layer into a rigid
  zero-mean-unit-variance shape and could destroy representational power the network needs.
- Placed between the linear transform and the activation (`Dense/Conv → BN → Activation`); its
  statistics get noisier as batch size shrinks, so BN is unreliable with very small batches.
- **Training** uses live mini-batch statistics; **inference** uses a running (moving) average of
  mean/variance accumulated during training — this train/inference split is the classic BN gotcha.
- Net effect: faster and more stable convergence, tolerance for higher learning rates, reduced
  sensitivity to weight initialization (Section 09), mitigation of vanishing/exploding gradients
  (Section 06), and a mild regularizing effect that can reduce the need for dropout (Section 07).

**Next:** [11 — Optimizers](11-optimizers.md) — EWMA, momentum, NAG, AdaGrad, RMSProp, and Adam: the
algorithms that decide *how* gradient descent actually steps.
