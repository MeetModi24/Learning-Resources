# 10 — Gradient Descent

**Playlist videos 57–60.** Section 09 fit linear regression using the **normal equation** — a
closed-form solution you get by setting the cost function's derivative to zero and solving
directly. This section covers the alternative: **gradient descent**, an iterative optimizer that
doesn't solve for the minimum in one shot but *walks toward it* step by step. It's not just an
alternative for linear regression — it's the optimization algorithm underneath almost every ML and
deep learning model you'll meet from here on.

---

## 1. Why Not Just Use the Normal Equation?

The normal equation for linear regression is:

```
θ = (XᵀX)⁻¹Xᵀy
```

This is exact and requires no iteration, no learning rate, no tuning. So why does the rest of ML
use something else?

**Matrix inversion is expensive.** Inverting `XᵀX` (an `n×n` matrix, where `n` is the number of
features) costs roughly **O(n³)** time. For a handful of features this is instant. For thousands
or millions of features — text embeddings, image pixels, one-hot-encoded categoricals — it becomes
computationally infeasible. It also doesn't scale gracefully with the number of *rows*; very large
datasets make even forming `XᵀX` costly, and the matrix may not always be invertible (singular /
near-singular when features are collinear).

**Gradient descent scales instead of inverting.** It replaces one big matrix inversion with many
cheap, small update steps. Its cost per step is roughly linear in the data size, it works when
`XᵀX` isn't practically invertible, and — crucially — it generalizes far beyond linear regression:
logistic regression, SVMs, neural networks, and virtually every model with a differentiable cost
function is trained with some flavor of gradient descent. The normal equation is a one-off trick
that only works for linear regression's specific closed form; gradient descent is the general
workhorse.

---

## 2. The Core Idea

You have a **cost function** `J(θ)` that measures how wrong the model is (for linear regression,
typically **MSE**). Training means finding the parameters `θ` (slope and intercept, or weight
vector) that **minimize** `J(θ)`.

Think of `J(θ)` as a surface — a landscape whose height is the error for each possible setting of
`θ`. For linear regression with MSE, this surface is a **convex bowl**: smooth, with exactly one
minimum, no false valleys. Gradient descent's job is to start somewhere on that bowl and walk
downhill until it reaches the bottom.

**The gradient** `∇J(θ)` is the vector of partial derivatives of the cost with respect to each
parameter — it points in the direction of **steepest increase** of the cost. Since we want to
*decrease* the cost, we step in the **negative** gradient direction: that's why it's called
gradient *descent*.

**The update rule:**

```
θ := θ − α · ∇J(θ)
```

- `α` (alpha) is the **learning rate** — how big a step to take each time.
- `∇J(θ)` is recomputed at the *current* `θ` every step, so the direction adapts as you move.
- One full pass of doing this is one **iteration**; one pass through the entire training set is one
  **epoch** (the exact relationship between "iteration" and "epoch" depends on the variant — see
  Section 5).

Concretely, for linear regression with MSE cost, the gradients with respect to slope `m` and
intercept `b` are:

```
∂J/∂b = -(2/n) Σ (yᵢ - ŷᵢ)
∂J/∂m = -(2/n) Σ (yᵢ - ŷᵢ) · xᵢ
```

and each step updates `b := b − α·∂J/∂b` and `m := m − α·∂J/∂m` simultaneously.

---

## 3. The Learning Rate: the One Knob That Matters Most

`α` controls step size, and getting it wrong breaks convergence in two opposite ways:

- **Too small:** each step barely moves. The algorithm crawls toward the minimum, needing far more
  iterations than necessary — technically correct, practically too slow to be useful.
- **Too large:** each step overshoots the minimum, lands on the other side of the bowl, overshoots
  again, and the cost can oscillate or even **diverge** (grow without bound) instead of converging.
- **Well-chosen:** the cost decreases smoothly and quickly, flattening out as `θ` approaches the
  minimum (the gradient itself shrinks near the bottom of the bowl, so steps naturally get smaller).

There's no universal correct value — it's found by experimentation (or scheduling, see Section 4)
for a given problem and scaling of the data.

**Convergence** is declared, in practice, when the cost stops decreasing meaningfully between
iterations (the change drops below some small tolerance) or after a fixed number of epochs.

---

## 4. Convex vs Non-Convex Surfaces

For **linear regression with MSE**, the cost surface is provably convex — a single bowl. This
means gradient descent, given a reasonable learning rate, is **guaranteed to reach the global
minimum**, regardless of where `θ` starts. There's no risk of getting stuck.

Many other models (neural networks especially) have **non-convex** cost surfaces — landscapes with
multiple **local minima**, saddle points, and plateaus. There, gradient descent can get stuck in a
local minimum that isn't the best possible solution, and where you initialize `θ` starts to matter.
This is one motivation for the noisier variants in Section 6 — a bit of randomness in the descent
path can help hop out of shallow local minima.

---

## 5. Feature Scaling and Convergence Speed

This connects back to Section 04. If features are on very different scales (e.g. "rooms" ranging
1–10 and "square footage" ranging in the thousands), the cost surface becomes a **long, narrow,
elongated bowl** rather than a nicely round one. Gradient descent on an elongated bowl zig-zags
back and forth across the narrow axis instead of moving directly toward the minimum, needing many
more steps to converge — the same learning rate that's stable for one feature's scale can be
wildly too large or too small for another's.

**Scaling features to a comparable range (standardization / min-max scaling) makes the bowl more
circular**, so a single learning rate works well across all dimensions and convergence is much
faster. This is one of the concrete, practical payoffs of feature scaling, not just a theoretical
nicety.

---

## 6. The Three Variants: Batch, Stochastic, Mini-Batch

All three use the exact same update rule (`θ := θ − α·∇J(θ)`); what differs is **how much data is
used to compute the gradient at each step**.

### Batch Gradient Descent (BGD)

Uses the **entire training set** to compute the gradient before making a single update. One epoch
= one update.

- **Convergence path:** smooth and direct — since the gradient is exact (computed over all data),
  each step reliably moves toward the minimum.
- **Cost:** every step requires a full pass over the data, which is slow and memory-heavy for large
  `n` (all the data typically needs to fit in memory to vectorize the computation).
- Guaranteed to converge to the global minimum for convex problems (given a suitable `α`), and to
  *a* local minimum for non-convex ones.

### Stochastic Gradient Descent (SGD)

Uses **one randomly chosen sample** to compute the gradient and update `θ`, then moves to the next
random sample. One epoch = `n` updates (one per training example, typically after a shuffle).

- **Convergence path:** noisy and erratic — since each gradient is a rough estimate based on a
  single point, the path zig-zags around the minimum rather than approaching it smoothly. It rarely
  settles exactly at the minimum; it fluctuates around it.
- **Upside of the noise:** for non-convex problems, this randomness can help the optimizer jump out
  of shallow local minima that would trap batch GD.
- **Speed:** each individual update is very cheap (one sample), so it can start improving
  immediately rather than waiting for a full pass — well suited to huge or streaming datasets that
  don't fit in memory (an "online learning" fit, tying back to Section 01 §4).
- **Practical caveat:** because of the noise, SGD usually needs a **learning rate schedule** (start
  larger, shrink `α` over epochs) to actually settle near the minimum rather than perpetually
  bouncing around it.

### Mini-Batch Gradient Descent

Uses a **small, fixed-size batch** of samples (commonly 32, 64, 128, ...) to compute each gradient
update. One epoch = `n / batch_size` updates.

- **Convergence path:** less noisy than pure SGD (averaging over a batch smooths out the estimate)
  but still has some useful stochasticity, and it's far cheaper per epoch than full-batch GD.
- This is the **practical default** used almost everywhere, especially in deep learning: batches are
  large enough to vectorize efficiently on hardware (GPU-friendly), small enough to avoid loading
  the whole dataset into memory, and the noise level is a tunable knob (via batch size) between the
  extremes of BGD and SGD.

---

## 7. Comparison Table

| | Batch GD | Stochastic GD | Mini-Batch GD |
|---|---|---|---|
| Data used per update | Entire dataset | 1 random sample | Small batch (e.g. 32–256) |
| Updates per epoch | 1 | n | n / batch_size |
| Speed per update | Slow | Very fast | Fast |
| Convergence path | Smooth, direct | Noisy, erratic | Moderately smooth |
| Memory | High (needs full dataset) | Very low | Low–moderate |
| Escapes shallow local minima | No | Yes (helps) | Somewhat |
| Typical use | Small/medium datasets | Huge/streaming data | Default in practice, esp. deep learning |

---

## 8. Minimal Code

Conceptual update loop (illustrates the core mechanics without a library):

```python
import numpy as np

def gradient_descent(X, y, lr=0.01, epochs=100, batch_size=None):
    n, n_features = X.shape
    theta = np.zeros(n_features)   # start anywhere on the bowl
    bs = batch_size or n           # bs == n -> batch GD; bs == 1 -> SGD; else mini-batch

    for _ in range(epochs):
        idx = np.random.permutation(n)
        for start in range(0, n, bs):
            batch_idx = idx[start:start + bs]
            Xb, yb = X[batch_idx], y[batch_idx]

            y_pred = Xb @ theta
            grad = -(2 / len(yb)) * Xb.T @ (yb - y_pred)   # ∇J(θ) for MSE
            theta -= lr * grad                              # θ := θ − α·∇J(θ)

    return theta
```

In practice you'd reach for scikit-learn's `SGDRegressor`, which implements (mini-batch/stochastic)
gradient descent with configurable learning-rate schedules:

```python
from sklearn.linear_model import SGDRegressor

model = SGDRegressor(learning_rate="constant", eta0=0.01, max_iter=1000)
model.fit(X_train, y_train)   # note: scale X first (Section 04) — SGD is sensitive to feature scale
```

---

## Key takeaways

- The normal equation is exact but requires inverting `XᵀX` — roughly `O(n³)`, infeasible for large
  feature counts. Gradient descent trades an exact one-shot solve for cheap iterative steps, and is
  the general-purpose optimizer behind almost all of ML/DL.
- Core update: `θ := θ − α·∇J(θ)` — step in the direction opposite the gradient (steepest descent)
  to reduce the cost.
- Learning rate `α` is the critical hyperparameter: too small → slow convergence; too large →
  overshoot or diverge.
- Linear regression's MSE cost surface is convex (one bowl) — gradient descent is guaranteed to
  find the global minimum. Non-convex surfaces (e.g. neural nets) can trap it in a local minimum.
- Feature scaling turns an elongated bowl into a round one, letting one learning rate work well
  across all features and converge much faster.
- Batch GD: full dataset per update — smooth but slow/memory-heavy. Stochastic GD: one sample per
  update — fast, noisy, good for huge/streaming data, needs a learning-rate schedule. Mini-Batch
  GD: small batches — the practical default, balancing stability, speed, and memory.

**Next:** [11 — Regularization](11-regularization.md) — polynomial regression, bias–variance, and
Ridge/Lasso/ElasticNet.
