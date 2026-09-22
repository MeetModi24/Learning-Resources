# 16 — Support Vector Machines

**Playlist videos 92–96.** SVM is the last of the "classic" supervised algorithms before trees and
ensembles take over. The core idea is a genuinely different way of picking a decision boundary
than logistic regression: instead of maximizing likelihood, SVM maximizes the **margin** — the
buffer zone around the boundary. That single idea drives everything below: the hard-margin
formulation, why real data needs a soft margin, and why the kernel trick lets SVM handle
non-linear boundaries efficiently.

---

## 1. Geometric Intuition — Maximum Margin

For a linearly separable binary classification problem, there are *infinitely many* hyperplanes
that separate the two classes perfectly. Logistic regression finds *some* separating line (whichever
one gradient descent on log-loss converges to) — it has no notion of "how good" that particular
line is beyond fitting the data. SVM asks a sharper question: **of all the lines that separate the
classes, which one generalizes best to new points?**

The answer: the one that is as far as possible from both classes — i.e. the line with the widest
**margin**.

- Draw the separating hyperplane.
- Draw two parallel hyperplanes on either side, pushed out until each just touches the nearest
  point(s) of its class.
- The gap between those two parallel planes is the **margin**. SVM picks the position and
  orientation of the boundary that makes this gap as wide as possible.

The points that lie exactly on those two touching planes are the **support vectors**. They are the
*only* points that determine the boundary — every other point could be deleted (or moved further
into its own region) without changing the solution at all. This is the geometric heart of SVM: the
decision boundary is decided by the hardest, closest, most "borderline" examples, not by the bulk of
the data.

Why maximize the margin at all? Intuitively, a boundary that barely squeezes between the two
classes is fragile — a new point close to the training data can easily land on the wrong side. A
boundary with a wide cushion on both sides is more robust to small perturbations and unseen data.
This is why SVM is often praised for generalizing well, especially in high-dimensional spaces.

**Contrast with logistic regression:** logistic regression's loss (cross-entropy on the sigmoid
output) is driven by *all* points — it keeps pushing to make confident points even more confident,
which doesn't have a margin-maximizing interpretation. SVM's hinge-style objective (below) only
cares about points near or on the wrong side of the margin; points already comfortably classified
contribute nothing to the loss. That's the practical reason support vectors are "only a few points"
— the rest of the training set is irrelevant to the final boundary once fit.

---

## 2. Hard Margin SVM — The Math

Assume the two classes are linearly separable and labels are encoded as **y ∈ {−1, +1}** (not
{0, 1} — this is what makes the margin algebra clean).

The hyperplane is:

```
wᵗx + b = 0
```

`w` is the normal vector to the hyperplane (its direction sets the orientation); `b` is the offset.
The two boundary planes that touch the nearest points on each side are set at:

```
wᵗx + b = +1   (positive-class boundary)
wᵗx + b = −1   (negative-class boundary)
```

(The "+1 / −1" scale is a convenience — `w` and `b` can always be rescaled so the nearest points
land exactly on these planes.)

**Margin width.** The perpendicular distance between those two planes works out to:

```
margin = 2 / ||w||
```

So a *wider* margin means a *smaller* `||w||`. Maximizing `2/||w||` is equivalent to minimizing
`||w||`, and for cleaner calculus we minimize `½||w||²` instead (same minimizer, nicer gradient).

**Constraint.** Every training point must sit on the correct side of its boundary plane (not just
the separating hyperplane) — i.e. correctly classified with margin at least 1:

```
yᵢ(wᵗxᵢ + b) ≥ 1    for every training point i
```

Note the sign trick: because `y ∈ {−1, +1}`, this single inequality covers both classes — for a
positive point (`y=+1`) it means `wᵗx+b ≥ 1`; for a negative point (`y=−1`) it means
`wᵗx+b ≤ −1`. Either way, "correct side, at least margin 1 away."

**Hard margin SVM optimization problem:**

```
minimize   ½||w||²
subject to yᵢ(wᵗxᵢ + b) ≥ 1   for all i
```

This is a **constrained optimization problem** — a convex quadratic objective with linear
inequality constraints. It's solved via **Lagrange multipliers**, converting it into a **dual
problem** where each training point gets a multiplier `αᵢ`. The key structural result (don't worry
about deriving it, just know the consequence): at the optimum, `αᵢ > 0` *only* for the support
vectors — every non-support-vector point has `αᵢ = 0` and drops out of the solution entirely. This
dual formulation is also what makes the kernel trick (Section 4) possible, because the dual only
ever needs dot products between pairs of points, never the raw feature vectors.

**Hard margin's fatal flaw:** it requires the data to be *perfectly* linearly separable with *no*
noise. One mislabeled point, or one outlier inside the "wrong" class's territory, and there is no
`w, b` that satisfies every constraint — the problem becomes infeasible. Real-world data is almost
never this clean, which is exactly why soft margin exists.

---

## 3. Soft Margin SVM — Allowing Violations

Soft margin SVM keeps the same spirit (maximize margin) but **tolerates some points violating the
margin, or even being misclassified**, in exchange for a penalty.

For each point, introduce a **slack variable ξᵢ ≥ 0** measuring how far that point is on the wrong
side of its margin boundary (`ξᵢ = 0` means the point satisfies the original hard-margin
constraint; `ξᵢ > 0` means it's intruded into the margin or crossed the boundary — larger ξᵢ is a
worse violation).

The constraint relaxes to:

```
yᵢ(wᵗxᵢ + b) ≥ 1 − ξᵢ ,   ξᵢ ≥ 0
```

The objective now balances two competing goals — wide margin vs. few/small violations:

```
minimize   ½||w||² + C · Σ ξᵢ
subject to yᵢ(wᵗxᵢ + b) ≥ 1 − ξᵢ,  ξᵢ ≥ 0
```

**`C` is the trade-off knob**, and it's the main hyperparameter you tune in practice:

- **Large `C`** → violations are penalized heavily → the optimizer prioritizes classifying every
  point correctly, shrinking the margin to do so. Behaves like hard margin; higher variance, more
  prone to **overfitting** (chasing outliers/noise).
- **Small `C`** → violations are cheap → the optimizer accepts more misclassified/margin-violating
  points in exchange for a **wider, smoother margin**. Higher bias, more tolerant of noise, better
  generalization up to a point (too small and it underfits).

This is exactly the same bias–variance knob you've seen as `λ` in Ridge/Lasso (Section 11) — `C`
is SVM's regularization strength, just parameterized inversely (large `C` = *less* regularization,
unlike `λ` where large = *more*). Soft margin is what actually gets used in practice; "hard margin"
is really just the `C → ∞` limit of soft margin.

---

## 4. Kernel Trick — Geometric Intuition

Soft margin handles *noisy* but still roughly linear data. It does nothing for data that is
**fundamentally non-linearly separable** — e.g. one class forming a ring around the other; no
straight line in the original 2-D space separates them, no matter how much slack you allow.

**The idea:** if the classes aren't separable in the current feature space, project the data into a
**higher-dimensional space** where they *become* linearly separable. A classic example: points
arranged in concentric circles in 2-D become separable by a flat plane once you add a third feature
like `z = x² + y²` — the two classes now sit at different heights and a hyperplane slices cleanly
between them.

The naive way to do this is to explicitly engineer/compute the transformed features (`φ(x)`) and
run linear SVM in that new, larger space. The problem: useful transformations often blow up the
dimensionality massively (or go to infinite dimensions, as with RBF below), making the explicit
transform computationally infeasible.

**The trick:** recall from the dual optimization problem (Section 2) that SVM's solution only ever
needs the **dot product** between pairs of points, `xᵢᵗxⱼ` — never the individual feature vectors
in isolation. The kernel trick replaces that dot product with a **kernel function**
`K(xᵢ, xⱼ)` that computes *what the dot product would have been in the higher-dimensional space*,
**without ever explicitly constructing that space**. You get the benefit of a rich, non-linear
feature space at the computational cost of working in the original one.

Common kernels:

- **Linear:** `K(xᵢ, xⱼ) = xᵢᵗxⱼ` — no transformation, equivalent to plain linear SVM. Good
  default when you already suspect the classes are roughly linearly separable, or when the feature
  count is very high (text data).
- **Polynomial:** `K(xᵢ, xⱼ) = (xᵢᵗxⱼ + c)^d` — implicitly expands into polynomial feature
  interactions up to degree `d`, without ever materializing them.
- **RBF / Gaussian:** `K(xᵢ, xⱼ) = exp(−γ ||xᵢ − xⱼ||²)` — the default go-to for non-linear
  problems. Intuition: it's a **similarity score based on distance**. Two points that are close
  together get a kernel value near 1 (very similar); points far apart get a value near 0
  (unrelated). This corresponds to an implicit mapping into an **infinite-dimensional** space —
  something you could never compute explicitly, but the kernel function evaluates in closed form.
  - **`gamma` (γ)** controls how far a single point's influence reaches: **high γ** → narrow
    reach, only very close points are considered "similar" → a wigglier, tighter decision boundary
    that hugs individual points (overfitting risk). **Low γ** → wide reach, far-apart points still
    count as similar → a smoother, more generalized boundary (underfitting risk if too low). `gamma`
    plays a role analogous to `C` — both are complexity/overfitting knobs, tuned together.

Why this is efficient: without the trick, using a kernel equivalent to a high-dimensional mapping
would require computing and storing that mapping for every point. With the trick, you only ever
compute pairwise `K(xᵢ, xⱼ)` values in the *original* feature space — cheap, and the dimensionality
of the implicit space (even infinite, for RBF) never has to be materialized.

---

## 5. Minimal Code Example

```python
from sklearn.svm import SVC
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline

# RBF is the typical default for non-linear data; C and gamma are the two
# hyperparameters worth tuning (grid search / cross-validation in practice).
model = make_pipeline(
    StandardScaler(),                      # SVM is distance/margin-based -> always scale features
    SVC(kernel="rbf", C=1.0, gamma="scale")
)
model.fit(X_train, y_train)
preds = model.predict(X_test)
```

`kernel="linear"` or `kernel="poly"` (with `degree=`) swap in the other kernels from Section 4.
Feature scaling matters *a lot* here — SVM's margin geometry is distance-based, so unscaled features
with very different ranges will silently dominate the boundary.

---

## 6. Pros, Cons, and SVR

**Pros:**

- Effective in **high-dimensional spaces**, even when the number of features exceeds the number of
  samples (common in text/genomics data).
- **Memory-efficient at prediction time** — the model is defined entirely by the support vectors
  (a subset of training points), not the whole dataset.
- **Versatile** via the kernel trick — the same algorithm handles linear and highly non-linear
  boundaries just by swapping the kernel.

**Cons:**

- **Slow on large datasets** — training complexity for kernelized SVM scales poorly (roughly
  quadratic-to-cubic in the number of samples), which is why SVM has fallen out of favor for very
  large tabular datasets in practice (tree ensembles, Sections 18–22, tend to win there).
- **Sensitive to hyperparameters** — `C`, `gamma`, and the kernel choice all need tuning; a bad
  combination underfits or badly overfits.
- **No direct probability estimates** — unlike logistic regression's sigmoid output, raw SVM only
  gives a class + a distance from the boundary. Probabilities (`predict_proba` in scikit-learn) are
  a separate, more expensive calibration step (Platt scaling), not a natural byproduct of training.

**SVR (Support Vector Regression):** the same margin-based machinery extends to regression — instead
of maximizing the gap between classes, SVR fits a tube of fixed width (`ε`) around the regression
line and only penalizes points that fall *outside* that tube. Not covered in depth by this playlist
block, but worth knowing it exists as SVM's regression counterpart.

---

## Key takeaways

- SVM's defining idea is **maximizing the margin**, not just finding *a* separating boundary — the
  boundary is decided entirely by the closest points (**support vectors**); everything else is
  irrelevant to the solution.
- **Hard margin**: minimize `½||w||²` subject to `yᵢ(wᵗxᵢ+b) ≥ 1`; margin = `2/||w||`. Requires
  perfectly separable, noise-free data — solved as a constrained optimization via Lagrange
  multipliers / the dual problem.
- **Soft margin**: adds slack variables `ξᵢ` and a penalty `C·Σξᵢ` to tolerate violations on real,
  noisy data. `C` is SVM's regularization knob — large `C` = less tolerance (higher variance),
  small `C` = wider margin, more tolerance (higher bias).
- **Kernel trick**: implicitly maps data to a higher-dimensional space where it *is* linearly
  separable, by computing the dot product in that space directly via a kernel function
  `K(xᵢ, xⱼ)` — never materializing the transformed features. RBF's `gamma` controls how far each
  point's similarity reach extends, paired with `C` as the two main tuning knobs.
- SVM shines in high dimensions and with clear margins, but scales poorly to large datasets and
  needs careful `C`/`gamma`/kernel tuning; probabilities aren't native output. SVR extends the same
  idea to regression.

**Next:** [17 — Decision Trees](17-decision-trees.md) — a completely different way of building a
decision boundary: recursive splitting instead of margin geometry.
