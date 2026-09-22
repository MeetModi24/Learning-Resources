# 09 — Linear Regression

**Playlist videos 50–56.** The first real predictive model in the guide. This section covers
simple linear regression (fitting a line), its closed-form math, how to actually score a
regression model, the jump to multiple features (a hyperplane), and the assumptions that justify
trusting the model in the first place.

---

## 1. Simple Linear Regression — Intuition & Code

The setup: one input feature `x`, one continuous target `y`, and a belief that the relationship
between them is roughly a straight line:

```
y = mx + b        (equivalently written  y = β0 + β1·x)
```

- `m` (or `β1`) — the **slope**: how much `y` changes per unit change in `x`.
- `b` (or `β0`) — the **intercept**: the predicted `y` when `x = 0`.

"Fitting" the model means choosing the specific `m` and `b` that make the line track the data as
closely as possible. For any candidate line, each point has a **residual** — the vertical gap
between the actual `yi` and the line's prediction `ŷi`:

```
residual_i = yi - ŷi = yi - (m·xi + b)
```

**Least squares** is the fitting rule this section uses: pick the line that minimizes the **sum of
squared residuals**. Squaring does two things — it makes residuals positive (so overshoots and
undershoots don't cancel out) and it penalizes big misses much more than small ones. Among the
infinite lines you could draw through a scatter of points, least squares picks the one unique line
minimizing that sum — and it has a closed-form answer (Section 2).

```python
from sklearn.linear_model import LinearRegression

model = LinearRegression()
model.fit(X_train, y_train)      # X_train shape: (n_samples, 1) for simple regression
print(model.coef_, model.intercept_)   # m, b
preds = model.predict(X_test)
```

---

## 2. Mathematical Formulation (OLS from scratch)

### The cost function

To fit the line we need a number that scores "how good" a given `(m, b)` is. That's the **cost
function**, and for linear regression it's the **Mean Squared Error (MSE)**:

```
J(m, b) = (1/n) · Σ (yi - (m·xi + b))²
```

This is exactly the least-squares idea from Section 1, averaged over `n` points. Training = finding
the `(m, b)` that **minimizes** `J`. Because `J` is a smooth bowl-shaped (convex) function of `m`
and `b`, it has a single minimum — no local-minima traps for simple linear regression.

### The closed-form (OLS) solution

Taking the derivative of `J` with respect to `m` and `b`, setting both to zero, and solving gives
**Ordinary Least Squares (OLS)**:

```
m = Σ(xi - x̄)(yi - ȳ) / Σ(xi - x̄)²

b = ȳ - m·x̄
```

where `x̄` and `ȳ` are the means of `x` and `y`. Read the slope formula as *"the covariance of `x`
and `y`, normalized by the variance of `x`"* — it tells you how much `y` tends to move for a unit
move in `x`, on average, across the whole dataset. Once you have `m`, the intercept just forces the
line through the point `(x̄, ȳ)` — the line always passes through the mean of the data.

This is a direct, algebraic solution — no iteration, no guessing. Plug in the data, get `m` and `b`.
That's the whole "from scratch" implementation: compute the two sums, divide, done. (Section 10
introduces gradient descent, an *iterative* way to minimize the same cost function — necessary once
a closed form gets expensive or doesn't exist, as with multiple features at scale.)

---

## 3. Regression Metrics

Once a model produces predictions, you need a number that says how good they are. Each metric below
answers a slightly different question — picking the wrong one can hide the wrong kind of failure.

### MAE — Mean Absolute Error

```
MAE = (1/n) · Σ |yi - ŷi|
```

Average of the absolute residuals. **Same units as the target**, easy to explain ("predictions are
off by $X on average"), and **robust to outliers** — a single huge miss contributes only its actual
size, not its square. Downside: it's not differentiable at zero, which matters for some optimization
methods (not OLS, but relevant elsewhere).

### MSE — Mean Squared Error

```
MSE = (1/n) · Σ (yi - ŷi)²
```

Squares the residuals before averaging, so **large errors are penalized disproportionately more**
than small ones — a residual of 10 contributes 100× a residual of 1. This is also the OLS cost
function itself (Section 2), and it's smooth and differentiable everywhere, which is why it's the
default target for optimization. Downside: the units are squared (e.g. "dollars²"), which makes it
hard to interpret directly, and it's more sensitive to outliers than MAE.

### RMSE — Root Mean Squared Error

```
RMSE = √MSE
```

Just the square root of MSE, which brings the units back to the original scale of `y` — so it's
interpretable like MAE ("off by $X on average") while still keeping MSE's heavier penalty on large
errors. In practice RMSE is often the default reporting metric for regression because it combines
interpretability with outlier-sensitivity.

### R² — Coefficient of Determination

```
R² = 1 - SS_res / SS_tot

SS_res = Σ (yi - ŷi)²          (residual sum of squares — the model's error)
SS_tot = Σ (yi - ȳ)²           (total sum of squares — error of just predicting the mean)
```

R² is the **proportion of variance in `y` explained by the model**, relative to the naive baseline
of always predicting `ȳ`. Interpretation:

- `R² = 1` → the model explains all variance (perfect fit).
- `R² = 0` → the model is no better than predicting the mean every time.
- `R² < 0` → the model is **worse** than just predicting the mean — this can happen (e.g. a badly
  overfit model evaluated on new data), and it's a real warning sign, not a bug.

R² is scale-free (unlike MAE/MSE/RMSE), which makes it easier to compare across different datasets
or targets — but it says nothing about the absolute size of the error.

### Adjusted R²

Plain R² has a structural flaw: **adding any feature to the model — even pure noise — can never
decrease R², and usually increases it slightly**, because the model gains one more degree of
freedom to fit the training data. That makes R² unreliable for comparing models with different
numbers of features, or deciding whether a new feature actually helped.

```
Adjusted R² = 1 - [ (1 - R²)(n - 1) / (n - k - 1) ]
```

where `n` = number of samples, `k` = number of features. Adjusted R² adds a penalty term that grows
with `k`: a useless feature that barely improves `SS_res` can actually **decrease** Adjusted R²,
because the penalty for the extra parameter outweighs the tiny fit improvement. This is the correct
metric when comparing models that don't have the same feature count — plain R² is not.

### Which to use when

- Need something in the same units as the target, robust to a few bad outliers → **MAE**.
- Optimizing a model (need differentiability) or care a lot about large errors → **MSE**.
- Want an interpretable, outlier-aware number to report → **RMSE**.
- Want "how much of the variance did I explain" on one model → **R²**.
- Comparing models with different feature counts → **Adjusted R²**.

```python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

mae = mean_absolute_error(y_test, preds)
mse = mean_squared_error(y_test, preds)
rmse = mse ** 0.5
r2 = r2_score(y_test, preds)

n, k = X_test.shape[0], X_test.shape[1]
adj_r2 = 1 - (1 - r2) * (n - 1) / (n - k - 1)
```

---

## 4. Multiple Linear Regression — Geometric Intuition & Code

Simple linear regression has one feature and fits a line in 2D. **Multiple linear regression**
extends this to `n` features:

```
y = β0 + β1·x1 + β2·x2 + ... + βn·xn
```

- `β0` is still the intercept.
- Each `βi` is that feature's own slope — how much `y` changes per unit change in `xi`, **holding
  all other features fixed**.

### Geometric picture

With one feature, the model is a line in 2D (`x, y`). With two features, it's a **plane** in 3D
(`x1, x2, y`) — instead of tracing a line through a scatter, you're fitting a flat plane through a
3D point cloud, minimizing the same sum of squared vertical distances. With `n` features, the model
generalizes to a **hyperplane** in `(n+1)`-dimensional space. You can't visualize it past 3
dimensions, but the idea is identical: a flat surface, one dimension higher than the feature space,
positioned to minimize squared vertical distance to every point.

```python
from sklearn.linear_model import LinearRegression

model = LinearRegression()
model.fit(X_train, y_train)      # X_train shape: (n_samples, n_features)
print(model.coef_)               # one β per feature
print(model.intercept_)          # β0
```

---

## 5. Multiple Linear Regression — Mathematical Formulation (Normal Equation)

With multiple features, tracking individual sums per feature gets unwieldy, so the closed form is
written in matrix notation. Stack the features into a matrix `X` (with a column of 1s prepended for
the intercept) and the targets into a vector `y`. The same least-squares idea — minimize the sum of
squared residuals `‖y - Xβ‖²` — has a closed-form solution called the **normal equation**:

```
β = (XᵀX)⁻¹ Xᵀy
```

Conceptually: `Xᵀy` correlates each feature with the target, and `(XᵀX)⁻¹` "undoes" the
feature-to-feature correlation structure so that each `βi` reflects that feature's independent
contribution. It's the direct multi-feature generalization of the simple-regression OLS formula in
Section 2 — same idea (set the derivative of the MSE cost to zero and solve), just in matrix form.

**Why this isn't always what you want in practice:**

- **Computationally expensive at scale.** Inverting `XᵀX` is roughly `O(k³)` in the number of
  features `k` (and involves an `n × k` matrix multiply first). Fine for tens or hundreds of
  features; painful for very large `k` or very large `n`.
- **`XᵀX` can be singular (non-invertible).** This happens when features are perfectly or near
  perfectly correlated (multicollinearity — see Section 6) or when there are more features than
  samples. No inverse means no closed-form solution.

These two limitations are exactly why Section 10 introduces **gradient descent** — an iterative
method that minimizes the same MSE cost function without ever forming or inverting `XᵀX`, and scales
to far larger problems.

---

## 6. Multiple Linear Regression — Code from Scratch

Implementing the normal equation directly, without `sklearn`, makes the math concrete:

```python
import numpy as np

class LinearRegressionScratch:
    def fit(self, X, y):
        # prepend a column of 1s to X for the intercept term
        X = np.insert(X, 0, 1, axis=1)
        betas = np.linalg.inv(X.T @ X) @ X.T @ y
        self.intercept_ = betas[0]
        self.coef_ = betas[1:]
        return self

    def predict(self, X):
        return self.intercept_ + X @ self.coef_
```

This is a direct translation of `β = (XᵀX)⁻¹Xᵀy`: `np.insert` adds the intercept column, `@` is
matrix multiplication, `np.linalg.inv` computes the matrix inverse. In production code, prefer
`np.linalg.pinv` or `np.linalg.lstsq` over `np.linalg.inv` — they're numerically more stable and
degrade gracefully instead of exploding when `XᵀX` is close to singular; `sklearn.LinearRegression`
uses this kind of stable solver internally rather than a naive inverse.

---

## 7. Assumptions of Linear Regression (Top 5)

Linear regression's coefficients, p-values, and confidence intervals are only trustworthy if the
data reasonably satisfies these assumptions. Violating them doesn't always break predictions
outright, but it undermines the *inferences* you draw from the model (which features matter, how
confident to be) and can degrade prediction quality.

1. **Linearity.** The relationship between the features and the target must actually be linear.
   If the true relationship is curved, a linear model will systematically under/overshoot in
   different regions.
   *Check:* plot `y` vs. each `x` (simple case), or plot residuals vs. predicted values — a
   linear fit should show residuals scattered with no visible curve or pattern.

2. **Little to no multicollinearity.** The features shouldn't be strongly correlated with each
   other. When two features carry almost the same information, `XᵀX` becomes close to singular
   (Section 5), and the model can't reliably tell which feature is "responsible" for the effect —
   coefficients become unstable and can swing wildly (even flip sign) with small data changes.
   *Check:* correlation matrix between features, or **VIF (Variance Inflation Factor)** — a VIF
   above ~5–10 for a feature signals problematic collinearity with the others.

3. **Homoscedasticity.** The residuals' spread (variance) should stay roughly constant across all
   levels of the predicted value — no funnel or fan shape. If error variance grows with the
   predicted value (**heteroscedasticity**), the model's confidence intervals are wrong even if the
   point predictions are okay.
   *Check:* plot residuals vs. predicted values — look for a horizontal band of roughly constant
   spread. A widening/narrowing "cone" shape indicates heteroscedasticity.

4. **Normality of residuals.** The residuals should be approximately normally distributed. This
   matters for the validity of statistical inference (confidence intervals, hypothesis tests on
   coefficients) built on top of the model — not for the point predictions themselves.
   *Check:* a **Q-Q plot** of the residuals — points should fall roughly along the diagonal line.
   Systematic deviation (curving away, heavy tails) means residuals aren't normal.

5. **Independence of errors (no autocorrelation).** One observation's residual shouldn't predict
   another's. This is most commonly violated in **time-series data**, where consecutive errors are
   correlated (today's error resembles yesterday's), which silently violates the model's core
   assumption that errors are independent noise.
   *Check:* plot residuals in the order the data was collected and look for patterns/trends; for
   time series specifically, autocorrelation plots or the Durbin-Watson statistic are the standard
   tools.

Why any of this matters: violating these assumptions doesn't necessarily mean the model is useless
for prediction, but it does mean you can't trust the *story* the coefficients tell you, and standard
errors / p-values / confidence intervals become unreliable. Always check residual plots — they're
the fastest, most direct diagnostic for #1, #3, #4, and #5 at once.

---

## Key takeaways

- Simple linear regression fits `y = mx + b` by minimizing the sum of squared residuals (least
  squares); the OLS closed form gives `m` and `b` directly from the data's means, variances, and
  covariance.
- MAE, MSE, RMSE, R², and Adjusted R² each answer a different question — same-unit robustness
  (MAE), penalizing big errors (MSE/RMSE), variance explained (R²), and fair comparison across
  differently-sized feature sets (Adjusted R²). Plain R² always non-decreases as you add features,
  which is exactly why Adjusted R² exists.
- Multiple linear regression is the same least-squares idea extended to many features: a
  hyperplane instead of a line, fit via the normal equation `β = (XᵀX)⁻¹Xᵀy`.
- The normal equation is exact but gets expensive and can fail (singular `XᵀX`) with many/
  correlated features or huge datasets — the motivation for gradient descent next.
- The five assumptions (linearity, low multicollinearity, homoscedasticity, normal residuals,
  independent errors) don't have to hold for the model to predict *at all*, but violating them
  undermines how much you can trust the coefficients and any inference built on them. Residual
  plots, VIF, and Q-Q plots are the standard diagnostics.

**Next:** [10 — Gradient Descent](10-gradient-descent.md) — the iterative alternative to the normal
equation, and how it scales to problems the closed form can't handle.
