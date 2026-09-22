# 12 — Logistic Regression

**Playlist videos 70–75, 79–81.** (Videos 76–78, the classification-metrics trio, are covered
separately in [13 — Classification Metrics](13-classification-metrics.md).) Despite the name,
logistic regression is a **classification** algorithm — it predicts a category, not a continuous
value. This section builds it up in the order the playlist does: start from a purely geometric
idea (the perceptron trick), hit its limits, fix those limits with the sigmoid function, derive a
proper loss function from maximum likelihood, and finish with gradient descent, multiclass
extension, and the practical hyperparameters.

---

## 1. Why not just use linear regression for classification?

Say the target is binary (0/1) — e.g. "will this customer churn." You could fit a straight line
`y = wx + b` and threshold the output at 0.5. Two things break:

- **Unbounded output.** A line's output ranges over all of `(-∞, ∞)`, but a class label — or a
  probability — should live in `[0, 1]`. Nothing about ordinary least squares constrains the
  output to a sensible range.
- **Sensitivity to outliers shifts the boundary.** Because linear regression minimizes squared
  error over the *raw* target values, a single far-away point (e.g. one extreme feature value in
  the positive class) drags the whole line — and therefore the decision threshold — with it, even
  though it shouldn't change which side of the boundary most points fall on.

What we actually want is a **decision boundary** — a line (or hyperplane in higher dimensions)
that separates the two classes — plus, ideally, a probability of belonging to each class. Logistic
regression gets there in two conceptual steps: first find *any* separating line (the perceptron
trick), then turn its output into a calibrated probability (the sigmoid).

---

## 2. The perceptron trick

### The geometric picture

A binary classifier can be represented by a hyperplane: `wᵀx + b = 0`. Points where `wᵟx + b > 0`
fall on one side (predict class 1), points where `wᵟx + b < 0` fall on the other (predict class
0). Training a linear classifier is exactly the problem of finding the `w` and `b` that place this
boundary correctly.

### The trick: nudge the line toward misclassified points

The perceptron trick is an iterative, intuitive way to find such a line, with no calculus:

1. Start with a random line (random `w`, `b`).
2. Pick a random training point.
3. If it's already correctly classified, leave the line alone.
4. If it's misclassified, **nudge the line toward that point** — update the weights by adding (or
   subtracting, depending on which class was misclassified) a small multiple of the point's
   coordinates: `w ← w + η·x` (or `w ← w − η·x`), where `η` is a small learning rate.
5. Repeat over many points/epochs until few or no points are misclassified.

The intuition: each misclassified point "pulls" the boundary toward itself just enough to flip
which side it's on, without needing gradients — you're directly rotating/shifting the line based
on where it's currently wrong.

### Why this isn't enough

Two limitations motivate everything that follows:

- **It only produces *a* boundary, not probabilities.** The output is a hard 0/1 decision. There's
  no notion of "70% confident this is class 1" — often exactly what's needed downstream (ranking,
  risk scores, thresholding for business needs).
- **The step function is not differentiable.** The perceptron trick works by direct geometric
  nudges, not gradient descent, because the underlying decision function is a step function (jump
  from 0 to 1 at the boundary) — it has no useful derivative. To plug this into a proper,
  gradient-based optimization framework (which generalizes far better and is what the rest of ML
  relies on), we need a **smooth, differentiable** stand-in for the step function. That stand-in is
  the sigmoid.

---

## 3. The sigmoid function

$$\sigma(z) = \frac{1}{1 + e^{-z}}$$

where `z = wᵟx + b` is the same linear combination as before (called the **logit**). The sigmoid
takes any real number and squashes it into `(0, 1)` — an S-shaped curve:

- As `z → +∞`, `σ(z) → 1`.
- As `z → -∞`, `σ(z) → 0`.
- At `z = 0`, `σ(z) = 0.5` — the inflection point.

This output is interpreted directly as **P(y = 1 | x)**, the model's estimated probability that
the point belongs to class 1. The classifier still draws a linear boundary in `x`-space (the
boundary is still `wᵟx + b = 0`), but now every point also gets a smooth confidence score, and the
whole pipeline is differentiable end to end.

**Decision rule:** predict class 1 if `σ(z) ≥ 0.5` (equivalently `z ≥ 0`), else class 0. This 0.5
threshold is a *default*, not a law — it can be moved to trade precision for recall (see Section
13).

### Log-odds / logit interpretation

Inverting the sigmoid gives the **logit**:

$$z = \ln\!\left(\frac{p}{1-p}\right)$$

This says logistic regression models the **log-odds of the positive class as a linear function of
the features**. That's the precise sense in which logistic regression is a *linear* model even
though its probability output is a curved S-shape — linearity lives in log-odds space, and the
sigmoid is just the transform that maps log-odds back to a probability.

---

## 4. Loss function: maximum likelihood → binary cross-entropy

### Why not mean squared error?

You could try plugging the sigmoid's output into the same squared-error loss used for linear
regression. The problem: **MSE on top of a sigmoid is non-convex** — it has multiple local minima,
so gradient descent isn't guaranteed to find the global best fit. We need a loss whose surface is
well-behaved (convex) for this model, and the principled way to get one is to derive the loss from
first principles: maximum likelihood.

### Maximum likelihood intuition

Given the model's predicted probability `p` for each point, and knowing the *true* label `y`
(0 or 1), the likelihood of the data under the model is the product, over all points, of "how
probable was the observed label given the model." For a single point:

- If `y = 1`, we want `p` to be large — the likelihood contribution is `p`.
- If `y = 0`, we want `p` to be small, i.e. `1 − p` to be large — the likelihood contribution is
  `1 − p`.

Both cases are captured in one expression: `p^y · (1 − p)^(1−y)`. **Maximum likelihood
estimation** picks the parameters `w, b` that maximize the product of this term across all
training points — i.e. the parameters that make the observed labels as probable as possible under
the model.

### From likelihood to log loss

Maximizing a product of many small numbers is numerically awkward and hard to differentiate
cleanly, so instead we maximize the **log**-likelihood (log turns the product into a sum, and log
is monotonic so the maximizer is unchanged). Then we flip the sign to turn "maximize" into
"minimize," which is the convention for a loss function. The result, per point, is **binary
cross-entropy (log loss)**:

$$\text{L}(y, p) = -\big[y \cdot \log(p) + (1 - y) \cdot \log(1 - p)\big]$$

and the total loss is the average of this over all training points. This is exactly:
**minimizing binary cross-entropy is equivalent to maximizing the likelihood of the data under the
sigmoid model.** Unlike MSE-on-sigmoid, this loss surface is convex in `w, b`, so gradient descent
reliably converges to the global minimum.

### Why it penalizes confident wrong predictions so heavily

Only one of the two terms is active for any given point (the other is multiplied by 0):

- If `y = 1`: loss is `-log(p)`. As `p → 0` (confidently wrong), `-log(p) → ∞`.
- If `y = 0`: loss is `-log(1-p)`. As `p → 1` (confidently wrong), `-log(1-p) → ∞`.

So a prediction that's *wrong and confident* is punished far more severely than one that's wrong
but uncertain (`p` near 0.5). This is a deliberate design goal: it pushes the model away from
being confidently incorrect, which is worse in practice than being unsure.

---

## 5. Derivative of the sigmoid

The sigmoid has an unusually clean derivative:

$$\sigma'(z) = \sigma(z)\,\big(1 - \sigma(z)\big)$$

That is, the derivative is expressible **entirely in terms of the sigmoid's own output** — no need
to recompute `e^{-z}` separately. This is why the sigmoid was historically such a convenient
nonlinearity: once you've done the forward pass and have `σ(z)`, the gradient computation is
almost free. It's also central to why the combination of *sigmoid + binary cross-entropy* produces
a gradient formula for logistic regression that ends up looking remarkably similar to linear
regression's — the messy exponential terms cancel out in the chain rule.

---

## 6. Gradient descent for logistic regression

Plugging the binary cross-entropy loss and the sigmoid into the chain rule, the gradient of the
loss with respect to each weight turns out to be:

$$\frac{\partial L}{\partial w_j} = \frac{1}{n}\sum_{i=1}^{n} \big(p_i - y_i\big)\, x_{ij}$$

This has the **same shape as the linear regression gradient** — "(prediction − actual) × feature,"
averaged over the batch — even though `p_i = σ(wᵟx_i + b)` is a nonlinear function of the
weights. That structural similarity is the elegant payoff of choosing log loss over MSE: the
sigmoid's derivative and the log loss's derivative combine to cancel out, leaving this simple
form. Training then proceeds exactly like linear regression's gradient descent (Section 10):
initialize `w, b`, repeatedly step opposite the gradient (`w ← w - η · ∂L/∂w`) until convergence,
optionally in batch/stochastic/mini-batch flavors.

```python
import numpy as np

def sigmoid(z):
    return 1 / (1 + np.exp(-z))

def train_logistic_regression(X, y, lr=0.01, epochs=1000):
    n, m = X.shape
    w, b = np.zeros(m), 0.0
    for _ in range(epochs):
        p = sigmoid(X @ w + b)
        dw = (X.T @ (p - y)) / n
        db = np.sum(p - y) / n
        w -= lr * dw
        b -= lr * db
    return w, b
```

---

## 7. Softmax / multinomial logistic regression

Plain logistic regression is binary. For **more than two classes**, the natural generalization is
**softmax regression** (a.k.a. multinomial logistic regression):

- Instead of one weight vector producing one logit, the model learns **one weight vector per
  class**, producing a raw score (logit) `z_k` for each class `k`.
- The **softmax function** converts the vector of raw scores into a valid probability
  distribution over all classes:

$$\text{softmax}(z_k) = \frac{e^{z_k}}{\sum_{j} e^{z_j}}$$

  Every output is in `(0, 1)` and all outputs sum to 1 — exactly what's needed to interpret them as
  class probabilities. Softmax is the natural multiclass generalization of the sigmoid (in fact,
  softmax over two classes reduces to the sigmoid).
- The loss generalizes analogously: instead of binary cross-entropy, it's **categorical
  cross-entropy**, which only looks at `-log(p_correct_class)` — again heavily penalizing confident
  wrong predictions, now across many classes.

**Alternative: One-vs-Rest (OvR).** Rather than jointly modeling all classes with softmax, train
`K` independent binary logistic regression classifiers, one per class, each asking "is it this
class or not." At prediction time, run all `K` classifiers and pick the class whose classifier
gives the highest probability. Softmax is usually preferred for a cleaner probabilistic
interpretation (probabilities across classes are jointly normalized), but OvR is simple and
sometimes competitive, especially when classes are separable independently.

```python
from sklearn.linear_model import LogisticRegression

# multi_class handles the >2-class case internally
model = LogisticRegression(multi_class="multinomial")   # softmax
model_ovr = LogisticRegression(multi_class="ovr")        # one-vs-rest
```

---

## 8. Polynomial features → non-linear decision boundaries

Logistic regression's decision boundary is inherently **linear** in its input features (recall:
it's `wᵟx + b = 0`). But real data often isn't linearly separable. The same trick used for
polynomial regression (Section 11) applies here: **engineer polynomial/interaction features**
(`x²`, `x1·x2`, etc.) before fitting.

The model is still, technically, "linear" — a linear combination of the *engineered* features —
but because those features are nonlinear functions of the original inputs, the resulting boundary
**in the original feature space** becomes curved (circles, parabolas, more complex shapes). This
is exactly how logistic regression can separate classes that aren't linearly separable, without
switching to a fundamentally different algorithm.

```python
from sklearn.preprocessing import PolynomialFeatures
from sklearn.pipeline import make_pipeline

model = make_pipeline(
    PolynomialFeatures(degree=2, include_bias=False),
    LogisticRegression()
)
```

---

## 9. Key hyperparameters (`sklearn.linear_model.LogisticRegression`)

- **`penalty`** — the regularization type applied to the weights: `'l2'` (default, Ridge-style,
  shrinks weights smoothly), `'l1'` (Lasso-style, can zero out weights → feature selection), or
  `'elasticnet'` (a mix of both, needs `l1_ratio`). Same underlying idea as Section 11's
  regularization, applied to the logistic loss instead of squared error.
- **`C`** — the **inverse** of regularization strength. This is the one to watch: unlike `alpha` in
  Ridge/Lasso, **smaller `C` means stronger regularization**, and larger `C` means weaker
  regularization (closer to unregularized logistic regression). It's easy to get this backwards.
- **`solver`** — the optimization algorithm used to minimize the log loss (e.g. `'lbfgs'`,
  `'liblinear'`, `'saga'`). Different solvers support different `penalty` options and scale
  differently with dataset size — `'liblinear'`/`'saga'` support `l1`, `'lbfgs'` does not (l2 only).
- **`multi_class`** — how multiclass problems are handled: `'multinomial'` (softmax, jointly
  normalized) vs `'ovr'` (One-vs-Rest, Section 7).
- **`max_iter`** — cap on solver iterations; logistic regression has no closed-form solution, so
  it's fit iteratively (like gradient descent) and can fail to converge within the default budget
  on harder or unscaled data — a common source of `ConvergenceWarning`.
- **`class_weight`** — reweights the loss contribution of each class (e.g. `'balanced'` upweights
  the minority class). Important for imbalanced classification (Section 24) where the default
  0.5 threshold and equal-weighted loss favor the majority class.

```python
from sklearn.linear_model import LogisticRegression

model = LogisticRegression(
    penalty="l2",
    C=1.0,
    solver="lbfgs",
    max_iter=1000,
    class_weight="balanced",
)
model.fit(X_train, y_train)
probs = model.predict_proba(X_test)   # calibrated class probabilities
preds = model.predict(X_test)         # thresholded at 0.5 by default
```

---

## Key takeaways

- Logistic regression is a **classification** algorithm that fits a **linear decision boundary**
  (`wᵟx + b = 0`); linear regression is unsuitable because its output is unbounded and it's overly
  sensitive to outliers in the raw target scale.
- The perceptron trick finds a separating line by nudging it toward misclassified points, but
  produces no probabilities and relies on a non-differentiable step function.
- The **sigmoid** `σ(z) = 1/(1+e^{-z})` fixes both: it's smooth/differentiable and maps any real
  number to `(0, 1)`, interpreted as `P(y=1|x)`. Its inverse is the log-odds, which is linear in
  the features — the sense in which the model is "linear."
- The loss is derived from **maximum likelihood**, not squared error (MSE on a sigmoid is
  non-convex). The result is **binary cross-entropy / log loss**, which heavily penalizes
  confident wrong predictions.
- `σ'(z) = σ(z)(1-σ(z))` is a clean, self-referential derivative that makes gradient descent's
  formula for logistic regression mirror linear regression's `(prediction - actual) × feature`
  shape.
- **Softmax regression** generalizes sigmoid + binary cross-entropy to `K` classes via a jointly
  normalized probability distribution and categorical cross-entropy; **One-vs-Rest** is a simpler
  alternative using `K` independent binary classifiers.
- **Polynomial features** let logistic regression carve non-linear boundaries in the original
  feature space, while remaining linear in the engineered feature space.
- Watch `C` (inverse regularization strength — smaller `C` = stronger regularization), `penalty`,
  `solver` compatibility, `multi_class`, `max_iter` (convergence), and `class_weight` (imbalance).

**Next:** [13 — Classification Metrics](13-classification-metrics.md) — confusion matrix,
precision/recall/F1, and ROC–AUC, to actually judge how good a classifier is.
