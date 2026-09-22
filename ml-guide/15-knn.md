# 15 — K-Nearest Neighbors

**Playlist video 91.** A single video, a single algorithm — but it's the canonical example of the
**instance-based / lazy learning** paradigm introduced in Section 01.4, so it earns its own section.
KNN does almost no work up front and all of its work at prediction time — the opposite of every
model-based algorithm covered so far (linear/logistic regression, Naive Bayes).

---

## 1. The core idea

KNN's assumption is simple: **similar points have similar labels/values.** To predict for a new
point, look at the training points closest to it and let them decide the answer.

There is no "training" in the usual sense — **fit** just stores the training data (`X`, `y`)
verbatim. All the computation happens in **predict**:

1. Compute the distance from the new point to *every* stored training point.
2. Take the `K` closest ones (the "K nearest neighbors").
3. Aggregate their labels/values:
   - **Classification:** majority vote — the new point gets the class that appears most often
     among its K neighbors.
   - **Regression:** average (mean) of the K neighbors' target values.

That's the whole algorithm. This is exactly the instance-based learner from Section 01: "the data
*is* the model." Training is `O(1)` (just store); prediction is `O(n)` per query (compare against
every stored point, then sort/select the K smallest distances).

---

## 2. Distance metrics

"Nearest" requires a distance function over the feature space. The common choices, all special
cases of one general form:

- **Euclidean distance** (straight-line distance, most common default):
  `d(x, y) = sqrt(Σ (x_i − y_i)²)`
- **Manhattan distance** (sum of absolute differences, moves along axes like city blocks):
  `d(x, y) = Σ |x_i − y_i|`
- **Minkowski distance** (the general form both of the above are instances of):
  `d(x, y) = (Σ |x_i − y_i|^p)^(1/p)`
  - `p = 1` → Manhattan.
  - `p = 2` → Euclidean.
  - `p → ∞` → Chebyshev (max coordinate-wise difference).

`p` (in scikit-learn, the `p` parameter alongside `metric='minkowski'`) is itself a hyperparameter
you can tune, though Euclidean (`p=2`) is the default and usually fine as a starting point.

### Scaling is not optional

Distance-based algorithms are dominated by whichever feature has the largest numeric range. If one
feature is "income" (range: thousands to millions) and another is "age" (range: 0–100), the
Euclidean distance is almost entirely determined by income — age barely moves the number regardless
of how meaningful it is. This directly reuses the scaling problem from Section 04: **always scale
features (standardization or min-max) before running KNN.** Unlike tree-based models, KNN has no
internal mechanism to correct for feature-range imbalance — the distance formula treats raw
magnitude as importance.

---

## 3. Choosing K

`K` is the central hyperparameter, and it directly controls the **bias–variance trade-off** (full
formal treatment in Section 11):

- **Small K (e.g. K=1):** the prediction follows individual points very closely — **low bias, high
  variance.** The decision boundary is jagged/noisy, highly sensitive to outliers and mislabeled
  points near the boundary (classic overfitting).
- **Large K:** the prediction is an average/vote over many points — **high bias, low variance.**
  The decision boundary becomes smooth, possibly ignoring real local structure (underfitting). At
  the extreme (K = number of training points), every prediction is just the global majority
  class/mean, regardless of the input.
- **Odd K for binary classification:** with an even K, a vote can tie (e.g. 2 vs 2 out of K=4).
  Using an odd K avoids ties by construction.
- **Selecting K in practice:** don't guess — sweep a range of K values and pick the one with the
  best **cross-validated** score (Section 25 covers systematic hyperparameter search). This is the
  standard way to find the sweet spot between the two failure modes above.

As K increases from 1 upward, the decision boundary visibly smooths out — this is the same
underfitting/overfitting spectrum you'll see again with tree depth, regularization strength, etc.

---

## 4. Curse of dimensionality

KNN is the algorithm that suffers most visibly from the **curse of dimensionality** (Section 08).
As the number of features grows, the volume of the feature space grows exponentially, and points
that were "close" in a couple of dimensions end up roughly equidistant from each other in high
dimensions — distance stops being a meaningful signal of similarity. Practically: KNN degrades
sharply on high-dimensional data unless you first reduce dimensionality (PCA, feature selection) or
have enough data density to compensate (which usually isn't feasible at high dimension counts).

---

## 5. Pros and cons

**Pros:**
- Conceptually simple, easy to implement and explain.
- No training phase — new labeled data can be "learned" just by adding it to the stored set.
- Naturally handles multiclass classification (majority vote works the same for any number of
  classes — no one-vs-rest machinery needed, unlike logistic regression).
- Makes no assumption about the shape of the decision boundary — it can model arbitrarily
  non-linear boundaries because it's entirely local.

**Cons:**
- **Slow at prediction time:** `O(n)` distance computations per query, which is the opposite of
  most algorithms (expensive training, cheap prediction). Gets worse as the dataset grows.
- **Memory-heavy:** the entire training set must be stored and available at prediction time.
- **Sensitive to feature scale** (Section 2 above) and to **irrelevant/noisy features** — every
  feature contributes to the distance whether or not it's actually predictive.
- **Sensitive to class imbalance:** if one class dominates the training set, it dominates the
  neighborhoods too, biasing the vote toward the majority class regardless of the new point's true
  neighborhood.
- **Curse of dimensionality** (Section 4 above).

### Weighted KNN

A common refinement: instead of every one of the K neighbors getting an equal vote, weight each
neighbor's contribution by the **inverse of its distance** — closer neighbors count more, farther
ones less. This softens the effect of a "borderline" K-th neighbor that's much farther away than
the others and often improves accuracy without any extra hyperparameter beyond enabling it
(`weights='distance'` in scikit-learn, vs. the default `weights='uniform'`).

---

## 6. Minimal code

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import make_pipeline

# Scaling is mandatory for KNN — bake it into a pipeline so it's never forgotten.
model = make_pipeline(
    StandardScaler(),
    KNeighborsClassifier(n_neighbors=5, weights="distance")
)
model.fit(X_train, y_train)      # just stores X_train, y_train (scaled)
preds = model.predict(X_test)    # all the work happens here
```

Swap in `KNeighborsRegressor` for a continuous target — same API, averages instead of votes.

---

## Key takeaways

- KNN is the canonical **instance-based / lazy learner**: fit stores the data; predict does all the
  work by finding the K nearest training points and voting (classification) or averaging
  (regression).
- Distance metrics — Euclidean, Manhattan, and the general Minkowski form — require **scaled
  features**, or the largest-range feature silently dominates every distance calculation.
- K controls bias–variance: small K overfits (jagged boundary, high variance), large K underfits
  (smooth boundary, high bias). Use odd K for binary classification to avoid ties, and pick K via
  cross-validation.
- High-dimensional data breaks KNN's core assumption — distances stop discriminating (curse of
  dimensionality) — so pair KNN with dimensionality reduction when feature counts are large.
- No training cost, expensive prediction cost — the inverse of most algorithms in this guide.
  Weighted KNN (inverse-distance weighting) is a simple, usually-beneficial refinement.

**Next:** [16 — SVM](16-svm.md) — geometric margin-based classification, hard/soft margins, and the
kernel trick.
