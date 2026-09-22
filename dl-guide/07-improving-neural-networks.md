# 07 — Improving Neural Networks

**Playlist videos 20–26.** So far the guide has built a network, made it forward-propagate, and made
it learn via backpropagation. In practice a freshly-built network trained with plain gradient descent
rarely performs well out of the box. This section covers the **variants of gradient descent** actually
used to train networks, frames the **general toolkit for improving a network's performance**, and then
works through the first three tools in that toolkit in depth: **early stopping**, **feature scaling**,
and **dropout/regularization**. The rest of the toolkit (activations, initialization, batch norm,
optimizers) gets its own sections (08–11).

---

## 1. Gradient Descent in Neural Networks: Batch vs Stochastic vs Mini-Batch

`../ml-guide` already covers gradient descent as an optimization algorithm — computing a gradient and
stepping downhill. In a neural network, the only new question is: **how much data do you use to
compute one gradient update?** That choice is what separates the three variants.

### Batch Gradient Descent

Compute the loss and gradient over the **entire training set**, then take one weight-update step.

- One **epoch** = one weight update (since the whole dataset is used at once).
- Gradient estimate is exact (no noise) → smooth, stable convergence toward the minimum.
- Extremely slow for large datasets, and requires the **entire dataset to fit in memory** at once
  (impossible for datasets that don't fit in RAM/GPU memory).

### Stochastic Gradient Descent (SGD)

Compute the loss and gradient using **one training sample at a time**, updating weights after every
single sample.

- One epoch = N updates, where N is the number of training samples.
- Very fast per step, and works with data that can't fit in memory (stream one sample at a time).
- The gradient estimate is noisy — the loss bounces around instead of decreasing smoothly. This noise
  is a double-edged sword: it can help the network escape shallow local minima, but it also means the
  loss never fully "settles" (it oscillates around the minimum instead of landing on it).

### Mini-Batch Gradient Descent

Compute the loss and gradient over a **small batch** of samples (typically 32, 64, 128, 256), update,
then move to the next batch.

- One epoch = N / batch_size updates. This ratio is exactly Keras's `steps_per_epoch` /
  "iterations per epoch."
- This is the **practical default** used by virtually every deep learning framework and every model
  in this guide from here on (Keras's `model.fit(batch_size=32)` is doing mini-batch GD).
- Why it wins: batching **vectorizes** the forward/backward pass into matrix operations that GPUs
  execute in parallel, so it's far faster per-sample than SGD despite doing "more work" per step than
  a single sample. It also smooths out the pure-SGD noise (averaging over a batch), while still
  keeping enough noise to help avoid getting stuck, and it doesn't require the whole dataset in
  memory at once like batch GD.

### Comparison

| | Batch GD | Stochastic GD | Mini-Batch GD |
|---|---|---|---|
| **Samples per update** | All N | 1 | Small batch (e.g. 32–256) |
| **Updates per epoch** | 1 | N | N / batch_size |
| **Gradient noise** | None (exact) | High | Moderate |
| **Convergence path** | Smooth | Erratic/bouncy | Smooth-ish, some noise |
| **Speed per epoch** | Slow | Fast per step, high overhead overall | Fast (vectorized) |
| **Memory** | Needs full dataset in memory | One sample at a time | One batch at a time |
| **Can escape local minima** | Hard (no noise) | Yes (a lot of noise) | Yes (some noise) |
| **Used in practice** | Rarely (small datasets only) | Rarely in pure form | **Default choice** |

```python
model.fit(X_train, y_train, epochs=50, batch_size=32)  # mini-batch GD, the default
# batch_size=len(X_train)  -> batch GD
# batch_size=1             -> pure SGD
```

---

## 2. How to Improve the Performance of a Neural Network — the Toolkit

Once a network trains at all, the question becomes: is it *underfitting* or *overfitting*? This is
the same **bias–variance trade-off** from `../ml-guide` — a network with high bias is too simple to
capture the pattern (underfitting); a network with high variance has memorized the training data and
generalizes poorly (overfitting). Every "improve the network" technique is aimed at one side or the
other, and diagnosing which side you're on (compare training loss vs validation loss) always comes
first.

### If the problem is underfitting (high bias) — help the network learn more / converge faster

- **Increase model capacity:** more layers, more neurons per layer.
- **Better activation functions:** e.g. ReLU instead of sigmoid, which suffers from vanishing
  gradients (Section 06) — covered in Section 08.
- **Better weight initialization:** avoid networks starting in a bad region of the loss surface —
  Section 09.
- **Feature/data scaling:** helps gradient descent converge faster — covered below in this section.
- **Better optimizers:** Adam, RMSProp, etc. instead of vanilla GD — Section 11.
- **Train for more epochs.**
- **Batch normalization:** keeps activations well-behaved layer to layer — Section 10.

### If the problem is overfitting (high variance) — help the network generalize

- **More training data** (the single most effective fix, when available).
- **Data augmentation** (especially for images — Section 15) — synthetically grow the dataset.
- **Early stopping:** stop training before the network starts memorizing — covered below.
- **Dropout:** force the network to not rely on any single neuron — covered below.
- **Regularization (L1/L2 / weight decay):** penalize large weights — covered below.
- **Reduce model complexity:** fewer layers/neurons (the opposite lever from underfitting).

This section (07) covers early stopping, scaling, dropout, and L1/L2 — the overfitting side plus
scaling. Sections 08–11 cover the underfitting/convergence side (activations, initialization, batch
norm, optimizers) in depth. Together, sections 07–11 *are* "how to improve a neural network" — this
section is the map for all of them.

---

## 3. Early Stopping

**Idea:** monitor the **validation loss** during training. As long as training continues, training
loss keeps falling — but past some point validation loss stops improving (or starts rising), which is
the signature of overfitting. Early stopping halts training right around that point instead of
running the full, pre-specified number of epochs.

- **Patience:** don't stop the instant validation loss ticks up once (it's noisy). Wait `patience`
  epochs without improvement before actually stopping — this tolerates small fluctuations while still
  catching a genuine trend.
- **Restore best weights:** by the time training stops, the *last* epoch's weights are already a
  bit past the optimum (that's why it stopped). Restore the weights from the epoch with the best
  validation loss, not the final epoch.
- This is a **regularizer that costs nothing extra to compute** — it doesn't change the model or the
  loss function, it just chooses a good stopping time. It directly targets overfitting from training
  too long, which is the same failure mode more epochs would otherwise cause.

```python
from tensorflow.keras.callbacks import EarlyStopping

early_stop = EarlyStopping(
    monitor="val_loss",
    patience=10,
    restore_best_weights=True,
)

model.fit(X_train, y_train, validation_split=0.2, epochs=500, callbacks=[early_stop])
```

---

## 4. Data / Feature Scaling in ANNs

**Why scaling matters more, not less, in neural networks:**

- **Faster, more stable gradient descent.** If features are on wildly different scales (e.g. age in
  years vs income in dollars), the loss surface becomes elongated/stretched in some directions. GD
  then zig-zags and converges slowly. Scaled features make the loss surface closer to circular,
  letting GD head straight for the minimum.
- **No feature dominates by scale alone.** A feature ranging in the thousands would produce much
  larger raw contributions to the weighted sum than a feature ranging in [0,1], even if the
  small-range feature is more predictive. Scaling puts every feature on equal footing so the network
  learns importance from correlation with the target, not from raw magnitude.
- **Keeps activations in a well-behaved range.** Unscaled inputs can push pre-activations `z` to
  extreme values, which for sigmoid/tanh land in the flat, near-zero-gradient tails — directly
  feeding the **vanishing gradient problem** from Section 06. Scaled inputs keep `z` near the
  activation's sensitive middle region.
- Deeper networks compound this: without scaling, the distribution of activations can drift and
  explode/vanish layer over layer purely from scale effects (this is also part of the motivation for
  batch normalization in Section 10).

**Standardization** (zero mean, unit variance — `StandardScaler`) is the default for neural networks;
**normalization** (min-max scaling to `[0, 1]`) is common for image pixel values (`/255.`).

**Leakage rule — same as `../ml-guide`:** fit the scaler on the training set only, then use that
*same fitted* scaler to transform validation and test sets. Fitting on the full dataset (including
test data) leaks test-set statistics into training and gives an optimistic, invalid performance
estimate.

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)   # fit only on train
X_test_scaled = scaler.transform(X_test)          # reuse train's fit
```

---

## 5. Dropout

**Idea:** during training, at each forward pass, **randomly zero out a fraction of neurons** in a
layer (their output is set to 0, so they contribute nothing to that pass, forward or backward). Which
neurons get dropped is re-randomized on every batch.

**Why this regularizes:**

- It prevents **co-adaptation** — neurons can no longer rely on a specific set of other neurons
  always being present to "cover" for their weaknesses. Each neuron has to learn something useful on
  its own.
- Because a different random subset of neurons is active on every pass, training with dropout is
  approximately like training a **large ensemble of smaller sub-networks** that share weights, then
  implicitly averaging their predictions. Ensembles generalize better than any single member — this
  is dropout's core regularization mechanism.

**The dropout rate** (e.g. `Dropout(0.5)`) is the fraction of neurons dropped, not kept. Common values
are 0.2–0.5; higher rates regularize more aggressively but can push the network toward underfitting if
overdone — it's a hyperparameter to tune like any other.

**Training vs inference — the crucial asymmetry:**

- Dropout is **active only during training**. At inference/prediction time, **all** neurons are used
  — no dropping.
- If neurons are dropped during training but all are present at inference, the sum of activations
  flowing into the next layer would be larger at inference than what the network was trained on. To
  correct for this, frameworks use **inverted dropout**: during training, the surviving neurons'
  outputs are scaled up by `1 / (1 - rate)` so the *expected* output magnitude matches what will occur
  at inference (where all neurons are present, unscaled). Keras's `Dropout` layer implements this
  automatically — during training it drops-and-scales, and it's a no-op at inference (`model.predict`
  / `training=False`). You don't need to do this bookkeeping by hand.

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense, Dropout

model = Sequential([
    Dense(128, activation="relu", input_shape=(n_features,)),
    Dropout(0.3),
    Dense(64, activation="relu"),
    Dropout(0.3),
    Dense(1, activation="sigmoid"),   # classification
    # Dense(1)  # regression: linear output, no activation
])
```

Dropout applies the same way to **regression and classification** — it's a layer-level technique
independent of the loss/output layer. The only difference is the usual one: sigmoid/softmax output +
cross-entropy loss for classification, linear output + MSE/MAE for regression. Dropout layers are
typically inserted after the dense layers you want to regularize, not after the final output layer.

---

## 6. Regularization: L1, L2, and Weight Decay

This is the exact same idea as **Ridge (L2) and Lasso (L1)** regression from `../ml-guide`, applied to
a neural network's weights instead of a linear model's coefficients: add a penalty on weight
*magnitude* directly to the loss function, so the optimizer is pushed to keep weights small while
still fitting the data.

- **L2 regularization:** penalty term `λ · Σ w²`. Discourages large weights by shrinking all of them
  smoothly toward (but not exactly to) zero. In deep learning this is very commonly called **weight
  decay** — because in the gradient-descent update, the L2 penalty's gradient literally subtracts a
  small fraction of the current weight at every step (`w ← w - η∇L - ηλw`), i.e. the weight "decays"
  a bit each update independent of the data term.
- **L1 regularization:** penalty term `λ · Σ |w|`. Its gradient has a constant magnitude regardless of
  how large `w` is, which — same as in Lasso — tends to push some weights **exactly to zero**,
  producing a sparse network (some connections effectively pruned). L2's gradient shrinks
  proportionally to `w`, so it shrinks weights but rarely zeroes them out — that's the shrinkage vs.
  sparsity distinction carried straight over from `../ml-guide`'s Ridge vs. Lasso.
- **Why smaller weights generalize better:** large weights let the network fit sharp, high-curvature
  functions that latch onto training-set noise. Constraining weight magnitude keeps the learned
  function smoother and simpler — a direct lever on model complexity, i.e. on the variance side of the
  bias–variance trade-off, same as dropout and early stopping.
- **λ (the regularization strength)** is a hyperparameter: too small has no effect, too large pushes
  the network toward underfitting (all weights crushed toward zero).

```python
from tensorflow.keras.layers import Dense
from tensorflow.keras import regularizers

Dense(128, activation="relu", kernel_regularizer=regularizers.l2(0.01))
Dense(128, activation="relu", kernel_regularizer=regularizers.l1(0.01))
Dense(128, activation="relu", kernel_regularizer=regularizers.l1_l2(l1=0.01, l2=0.01))  # elastic net
```

Regularization, dropout, and early stopping are complementary, not redundant — it's common to use
more than one together (e.g. dropout + L2 + early stopping) since each attacks overfitting differently
(structural, weight-magnitude, and training-duration, respectively).

---

## Key takeaways

- **Mini-batch gradient descent** is the practical default: it vectorizes for GPU speed while keeping
  enough gradient noise to avoid getting stuck, unlike batch GD (exact but slow/memory-heavy) or pure
  SGD (fast per-step but erratic).
- "Improving a network" always starts with diagnosing **underfitting vs. overfitting** (bias–variance,
  `../ml-guide`) — the fix differs by which side you're on. Sections 07–11 are the full toolkit for
  both sides.
- **Early stopping** monitors validation loss and halts training once it stops improving (with
  patience), restoring the best-seen weights — a zero-cost regularizer against training too long.
- **Feature scaling** speeds up GD convergence, stops large-range features from dominating, and keeps
  activations out of vanishing-gradient territory (Section 06). Fit the scaler on train only.
- **Dropout** randomly zeroes neurons during training only (scaled via inverted dropout to keep
  inference consistent) to prevent co-adaptation and approximate an ensemble — applies identically to
  regression and classification networks.
- **L1/L2 regularization** (L2 = "weight decay" in DL) penalizes weight magnitude in the loss, exactly
  mirroring Ridge/Lasso from `../ml-guide`: L2 shrinks, L1 sparsifies.

**Next:** [08 — Activation Functions](08-activation-functions.md) — sigmoid, tanh, ReLU and its
variants, and how the right activation avoids vanishing gradients and speeds up convergence.
