# 11 — Regularization

**Playlist videos 61–69.** Linear regression (Section 09) fits a straight line/hyperplane. This
section extends it to curved relationships (polynomial regression), formalizes *why* models fail —
too simple or too complex — as the bias–variance trade-off, and then introduces the fix for
too-complex models: penalizing large coefficients (Ridge, Lasso, ElasticNet).

---

## 1. Polynomial Regression

Linear regression assumes a straight-line (or flat-hyperplane) relationship between `X` and `y`.
Real relationships are often curved. **Polynomial regression** handles this without abandoning
linear regression at all: you engineer new features that are powers/interactions of the originals,
then run ordinary linear regression on the *expanded* feature set.

For one feature `x`, instead of fitting `y = β₀ + β₁x`, you construct `x², x³, ...` as extra
columns and fit:

```
y = β₀ + β₁x + β₂x² + β₃x³ + ...
```

The key insight: this is still called **linear** regression, because "linear" refers to
linearity *in the coefficients* `β`, not in `x`. The model is a linear combination of features —
it doesn't matter that one of those features happens to be `x²`. The fitting procedure (normal
equation or gradient descent) is unchanged; only the design matrix `X` changes.

With multiple original features, `PolynomialFeatures` also generates **interaction terms**
(`x₁x₂`, `x₁²x₂`, etc.), not just pure powers — this lets the model capture combined effects
between features, not only individual curvature.

```python
from sklearn.preprocessing import PolynomialFeatures
from sklearn.linear_model import LinearRegression
from sklearn.pipeline import make_pipeline

model = make_pipeline(PolynomialFeatures(degree=2, include_bias=False), LinearRegression())
model.fit(X_train, y_train)
```

**Degree is the knob.** `degree=1` is plain linear regression. As degree increases, the model can
bend through more of the training points — flexibility goes up. But flexibility is a double-edged
sword: past some degree, the curve starts snaking through individual noisy points rather than the
underlying trend. This is the direct on-ramp to the next topic — polynomial degree is the cleanest
possible knob for the bias–variance trade-off, because you can watch a single line change shape as
you turn it.

---

## 2. Bias–Variance Trade-off (Overfitting and Underfitting)

This is the central diagnostic concept for *every* model in this guide, not just linear ones.

### Defining bias and variance

Think of a model's expected error as coming from two independent sources (plus noise you can't
remove):

- **Bias** — error from the model's assumptions being *too simple/wrong* for the true
  relationship. A high-bias model can't capture the pattern even with unlimited data, because its
  functional form isn't rich enough. Symptom: **underfitting** — poor performance on training data
  *and* test data alike.
- **Variance** — error from the model being *too sensitive* to the specific training sample it saw.
  A high-variance model fits the noise/quirks of one training set, so a different training set
  would produce a noticeably different model. Symptom: **overfitting** — great performance on
  training data, poor performance on test/validation data (the gap between the two is the tell).

A useful mental image: bias is "I'm using a straight ruler to trace a curve" (wrong regardless of
data); variance is "I'm connecting every dot exactly" (perfect on this data, wrong on the next
sample).

### The decomposition

For a given point, expected squared prediction error decomposes as:

```
Total Error = Bias² + Variance + Irreducible Error
```

- **Bias²** — systematic error from an overly simple model.
- **Variance** — error from the model's sensitivity to the particular training sample.
- **Irreducible error** — noise inherent to the problem itself (measurement error, randomness);
  no model, however good, removes this.

You cannot make both Bias² and Variance zero simultaneously with a fixed amount of data — reducing
one generally increases the other. That tension *is* the trade-off.

### The U-shaped curve

Plot test error against model complexity (polynomial degree, tree depth, number of features,
whatever complexity knob a model has):

- At **low complexity**: high bias, low variance → high test error (underfitting side).
- At **high complexity**: low bias, high variance → high test error again (overfitting side).
- Somewhere in the **middle**: the sum Bias² + Variance is minimized → the sweet spot.

Test error traces a **U-shape** against complexity — that U is the single most important picture in
applied ML. Training error, by contrast, keeps *falling* monotonically as complexity increases,
because a more complex model can always fit training data better (or at least no worse).

### Diagnosing with train vs. validation error

This is the practical, everyday use of the concept:

| Train error | Validation error | Diagnosis |
|---|---|---|
| High | High (≈ train) | **Underfitting** (high bias) — model too simple |
| Low | High (>> train) | **Overfitting** (high variance) — model too complex |
| Low | Low (≈ train) | Good fit — near the sweet spot |

The gap between train and validation error is a direct proxy for variance; the absolute level of
train error is a direct proxy for bias.

### Where regularization fits in

If a model is overfitting (high variance), the fix is to constrain its flexibility. **Regularization
does exactly this: it deliberately introduces a small amount of bias in exchange for a large
reduction in variance**, moving you back toward the middle of the U. That trade is the entire
justification for everything in the rest of this section.

---

## 3. Regularization: The Core Idea

An overfit linear/polynomial model typically has **large-magnitude coefficients** — it swings wildly
to chase every training point. Regularization modifies the loss function to also penalize
coefficient *magnitude*, not just prediction error:

```
Regularized Loss = Original Loss (e.g. MSE) + λ · (penalty on coefficients)
```

- `λ` (often written `alpha` in scikit-learn) is a hyperparameter controlling **penalty strength**.
  - `λ = 0` → no penalty → ordinary linear regression.
  - `λ → ∞` → coefficients pushed toward zero → an increasingly flat/simple model.
- The penalty discourages the optimizer from choosing large coefficients unless the reduction in
  prediction error truly justifies it, which curbs the model's sensitivity to noise — i.e. it
  reduces variance, per the trade-off above.

The two classic choices differ only in *how* they measure coefficient magnitude: Ridge uses the sum
of squares (L2), Lasso uses the sum of absolute values (L1).

---

## 4. Ridge Regression (L2)

### Formulation

Ridge adds the sum of squared coefficients to the loss:

```
Loss = Σ(yᵢ − ŷᵢ)² + λ Σ βⱼ²
```

### Closed-form solution

Ordinary least squares has the normal equation `β = (XᵀX)⁻¹Xᵀy`. Ridge's closed form is:

```
β = (XᵀX + λI)⁻¹ Xᵀy
```

The `λI` term (identity matrix scaled by `λ`) is doing double duty:

1. It shrinks coefficients toward zero (the actual regularization effect).
2. It makes `XᵀX + λI` **invertible even when `XᵀX` is singular or near-singular** — which happens
   under multicollinearity (highly correlated features) or when there are more features than
   samples. Plain OLS can fail or become numerically unstable in that case; adding `λI` guarantees
   a well-conditioned matrix to invert. This is why Ridge is the standard fix when features are
   correlated.

### Geometric intuition

Minimizing the penalized loss is equivalent to minimizing the original loss (MSE) subject to a
**constraint** on coefficient size: `Σβⱼ² ≤ t` for some `t` tied to `λ`. In 2D coefficient space,
this constraint region is a **circle** (a sphere in higher dimensions) centered at the origin. The
unconstrained MSE loss has elliptical contours centered at the OLS solution. Ridge's solution is
where the smallest loss ellipse touches the circular constraint boundary.

Because a circle has no corners, that tangent point can land anywhere on its boundary — generally
at a point where **no coordinate is exactly zero**. Coefficients shrink toward the origin, but
smoothly, and essentially never hit it exactly (this is the key geometric contrast with Lasso —
Section 6).

### Effect of increasing λ

- `λ = 0`: identical to OLS.
- As `λ` increases: coefficients shrink continuously toward zero (never reaching it), the model
  gets flatter/simpler, bias increases, variance decreases.
- Very large `λ`: coefficients ≈ 0, the model approaches predicting the mean of `y` regardless of
  `X` — heavily underfit.

`λ` is tuned via cross-validation, watching the U-shaped validation-error curve from Section 2.

### Gradient descent version

Ridge doesn't require the closed form — it also works with gradient descent, since the penalty term
is differentiable. The gradient of the loss w.r.t. `βⱼ` picks up an extra `2λβⱼ` term compared to
plain linear regression:

```
∂Loss/∂βⱼ = −2Σxᵢⱼ(yᵢ − ŷᵢ) + 2λβⱼ
```

Each update step therefore shrinks `βⱼ` a little (proportional to its current value) in addition to
the usual error-driven update — this is sometimes called "weight decay." Gradient descent scales to
large feature counts where forming/inverting `(XᵀX + λI)` is expensive.

```python
from sklearn.linear_model import Ridge

model = Ridge(alpha=1.0)   # alpha == λ
model.fit(X_train, y_train)
```

### Key points on Ridge (summary)

1. Penalizes squared coefficient magnitude (L2).
2. Shrinks coefficients toward zero but **never exactly to zero** — keeps all features.
3. Closed form `(XᵀX + λI)⁻¹Xᵀy` is always invertible, unlike plain OLS — handles multicollinearity.
4. `λ` trades bias for variance; tune via cross-validation.
5. Also solvable via gradient descent for scale.

---

## 5. Lasso Regression (L1)

Lasso ("Least Absolute Shrinkage and Selection Operator") swaps the penalty to the sum of *absolute
values* of the coefficients:

```
Loss = Σ(yᵢ − ŷᵢ)² + λ Σ |βⱼ|
```

Everything about the motivation (shrink coefficients, trade bias for variance, tune `λ` by
cross-validation) carries over from Ridge. The one crucial behavioral difference: **Lasso can drive
coefficients to exactly zero**, effectively removing those features from the model. This makes
Lasso a built-in **feature selection** method — after fitting, inspect which coefficients are zero
to see which features the model decided were not worth keeping.

```python
from sklearn.linear_model import Lasso

model = Lasso(alpha=0.1)   # alpha == λ
model.fit(X_train, y_train)
```

---

## 6. Why Lasso Regression Creates Sparsity

This is the one mechanism worth understanding carefully rather than memorizing as a fact.

### Geometric explanation

Just like Ridge, Lasso's penalized loss is equivalent to minimizing MSE subject to a constraint on
coefficient magnitude — but the constraint region is now `Σ|βⱼ| ≤ t`, which in 2D coefficient space
is a **diamond** (an axis-aligned polytope in higher dimensions), not a circle.

The critical geometric fact: **a diamond has corners, and those corners sit exactly on the axes**
(e.g. at `(t, 0)` and `(0, t)` in 2D — points where one coefficient is zero). A circle has no such
special points; every boundary point is geometrically equivalent.

When you inflate the elliptical MSE loss contours outward from the OLS optimum until they first
touch the constraint region, that first contact is disproportionately likely to occur **at a
corner** of the diamond — because corners stick out and are more "exposed" to contours arriving
from a wide range of angles/orientations than a flat edge is. A contact point at a corner means one
or more coordinates are exactly zero. With Ridge's circle there are no corners to land on, so the
solution almost always lands where every coordinate is nonzero.

### The gradient-magnitude intuition

A second, complementary way to see it: the penalty term's gradient behaves differently near zero.

- For Ridge, `d/dβ (β²) = 2β` — the pull toward zero **shrinks in proportion to `β` itself**. As
  `β` gets small, the shrinking force gets small too, so it asymptotically approaches zero but the
  force never overwhelms a matching small residual-error gradient — `β` settles near but not at
  zero.
- For Lasso, `d/dβ |β| = sign(β)` — a **constant-magnitude** pull toward zero (`+λ` or `−λ`),
  regardless of how small `β` already is. Even when `β` is tiny, the penalty keeps pushing it
  toward zero with the same constant force. If the corresponding reduction in error from keeping
  that coefficient nonzero is smaller than this constant pull, the optimizer's best move is to snap
  the coefficient **exactly to zero** and leave it there.

Both explanations point at the same underlying fact: L1's penalty is "flat-rate" per unit of
coefficient magnitude (rewarding zero specifically), while L2's penalty is proportional and softens
near zero (never fully rewarding exact zero over a tiny nonzero value).

---

## 7. ElasticNet Regression

ElasticNet combines both penalties in one loss function:

```
Loss = Σ(yᵢ − ŷᵢ)² + λ₁ Σ|βⱼ| + λ₂ Σβⱼ²
```

Equivalently, scikit-learn parameterizes it with a total penalty strength `alpha` and a mixing ratio
`l1_ratio ∈ [0, 1]` that splits that strength between L1 and L2 (`l1_ratio=1` → pure Lasso,
`l1_ratio=0` → pure Ridge).

**Why combine them:** Lasso's feature-selection behavior gets unstable when features are highly
correlated — among a correlated group, Lasso tends to arbitrarily pick one and zero out the rest,
and that pick can change with small data perturbations. Ridge handles correlated features
gracefully (shrinks them together) but never eliminates any. ElasticNet's L2 component stabilizes
the selection among correlated features while its L1 component still zeroes out genuinely
irrelevant ones — giving you sparsity *and* stability in the presence of multicollinearity.

```python
from sklearn.linear_model import ElasticNet

model = ElasticNet(alpha=0.1, l1_ratio=0.5)   # 0.5 = even split between L1 and L2
model.fit(X_train, y_train)
```

---

## 8. When to Use Which

- **Ridge** — default choice when you believe most/all features carry some signal and you mainly
  want to tame overfitting and multicollinearity without discarding features.
- **Lasso** — when you suspect many features are irrelevant and want automatic feature selection /
  a sparse, more interpretable model. Be cautious if features are strongly correlated (unstable
  selection).
- **ElasticNet** — a safer general default when you want feature selection *and* have correlated
  features (common in real, wide datasets) — gets sparsity without Lasso's instability.

All three require `λ`/`alpha` (and, for ElasticNet, `l1_ratio`) to be tuned via cross-validation
against the U-shaped validation-error curve from Section 2 — there's no way to pick it analytically
for real data.

---

## Key takeaways

- Polynomial regression fits curves by adding polynomial/interaction features, then running
  ordinary linear regression on the expanded feature set — it's still linear *in the coefficients*.
  Degree is a direct complexity knob.
- Bias–variance trade-off: total error = Bias² + Variance + irreducible error. High bias →
  underfitting (poor train *and* validation). High variance → overfitting (good train, poor
  validation). Test error vs. complexity is U-shaped; the goal is the minimum of that U.
- Regularization adds a penalty on coefficient magnitude to the loss, trading a little bias for a
  large reduction in variance — moving an overfit model back toward the sweet spot.
- Ridge (L2, `Σβ²`): shrinks coefficients smoothly toward zero, never exactly zero; closed form
  `(XᵀX + λI)⁻¹Xᵀy` is always invertible, which is why it handles multicollinearity; also trainable
  via gradient descent.
- Lasso (L1, `Σ|β|`): can zero out coefficients exactly → automatic feature selection. Sparsity
  comes from the diamond-shaped constraint region's corners landing on the axes, and from L1's
  constant-magnitude gradient near zero, versus Ridge's circular region (no corners) and
  proportional gradient (softens near zero).
- ElasticNet mixes L1 + L2 to get Lasso's sparsity with Ridge's stability under correlated
  features.
- Ridge for general shrinkage/multicollinearity, Lasso for sparsity/feature selection, ElasticNet
  when you want both and features are correlated. Tune `λ` (and `l1_ratio`) by cross-validation.

**Next:** [12 — Logistic Regression](12-logistic-regression.md) — moving from regression to
classification.
