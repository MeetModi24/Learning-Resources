# 08 — Dimensionality Reduction & PCA

**Playlist videos 45–49.** Two ideas here: first, that you can often *build* better features
by hand (feature construction/splitting) before ever touching a model; second, that having too
many features is itself a problem (the curse of dimensionality) that motivates squeezing many
features down into a few via **Principal Component Analysis (PCA)**.

---

## 1. Feature Construction

Feature construction means **creating new features from existing ones**, using domain knowledge,
because the new feature carries signal the raw columns don't expose directly.

Common patterns:

- **Combining columns.** `Family_Size = SibSp + Parch + 1` (Titanic dataset) turns two weak
  columns into one that more directly captures "traveling alone vs. with family," which correlates
  with survival better than either input alone.
- **Ratios.** `BMI = weight / height²`, `price_per_sqft = price / area`. A ratio can encode a
  relationship a model would otherwise have to learn indirectly (and linear models can't learn a
  ratio from two separate columns at all — they can only take linear combinations).
- **Binary/indicator flags from domain knowledge.** `is_alone = 1 if Family_Size == 1 else 0`.
- **Aggregates.** Total spend, average order value, count of prior visits — derived from grouping
  raw transaction rows.

Why this can dramatically boost models: a model can only ever combine the features you give it in
the ways its own structure allows (linear models: linear combinations; trees: axis-aligned splits).
If the *true* signal is a ratio, a product, or a domain-specific combination, no amount of extra
data or model complexity substitutes for just handing the model that feature directly. This is
also why feature construction is one of the highest-leverage, lowest-cost steps in the MLDLC
(Section 01) — it's cheap and can beat switching algorithms entirely.

## 2. Feature Splitting

The reverse move: a single column actually bundles **multiple pieces of information**, and pulling
them apart lets the model use each independently.

- A `Name` column like `"Braund, Mr. Owen Harris"` bundles surname and a **title** (`Mr.`, `Mrs.`,
  `Miss.`, `Dr.`) — and the title alone is a strong proxy for age/sex/social status, which is
  exactly the kind of thing that predicts Titanic survival.
- A `Date` column bundles year, month, day, day-of-week, is-weekend, is-holiday — splitting it out
  lets a model see "this was a Saturday" instead of an opaque timestamp integer it can't relate to
  seasonality.
- An `Address` column bundles street, city, state, zip — city/state alone might be the useful
  signal, buried inside a high-cardinality string.

The mechanism is the same for both construction and splitting: **make the useful information
explicit as its own column** instead of leaving it implicit or buried, so the model doesn't have
to reverse-engineer it from raw, noisy, or compound inputs.

---

## 3. The Curse of Dimensionality

"Curse of dimensionality" is the umbrella term for a set of geometric facts that make **high-
dimensional data behave very differently from low-dimensional intuition**, and mostly for the
worse.

- **Volume grows exponentially with dimensions.** A unit hypercube in `d` dimensions has volume 1
  regardless of `d`, but to keep the same *density* of points as `d` grows, you need exponentially
  more data — the amount of data required to cover the space at a fixed resolution scales as
  `k^d` for some `k > 1`. Fixed-size datasets get sparser and sparser as dimensions increase.
- **Data becomes sparse.** With a fixed number of points spread over more and more dimensions, the
  points end up far apart from each other — "nearest neighbors" stop being meaningfully near.
- **Distances become less meaningful.** As `d` grows, the ratio between the distance to the
  nearest point and the distance to the farthest point tends toward 1 — in other words, in very
  high dimensions almost every pair of points looks roughly equidistant. This directly hurts
  distance-based methods like KNN (Section 15) and K-Means (Section 23), whose entire logic
  depends on "close" meaning something.
- **Overfitting risk rises.** More features means more ways for a model to find spurious patterns
  that fit the training data's noise, especially when the number of features approaches or exceeds
  the number of training examples. To keep learning reliable, the amount of data needed grows
  roughly exponentially with dimensionality — which in practice you never have.

The practical upshot: adding more features isn't free, and past some point it actively hurts.
This motivates **dimensionality reduction** — deliberately cutting the number of features while
keeping as much of the useful signal as possible.

### Feature selection vs. feature extraction

Two different strategies fall under "dimensionality reduction," and it's important to keep them
distinct:

- **Feature selection** — pick a *subset* of the original features and discard the rest (e.g.
  drop low-variance or highly-correlated columns, or use a model's feature importances). The
  surviving features are still the original, interpretable columns.
- **Feature extraction** — build a *smaller set of new features* that are combinations of the
  original ones, capturing most of the same information in fewer dimensions. The new features are
  generally not directly interpretable as any single original column. **PCA is feature
  extraction.**

---

## 4. PCA — Geometric Intuition

Picture a 2-D scatter plot of correlated data — say height and weight — plotted along the
original x/y axes. The cloud of points is elongated along some diagonal direction rather than
aligned with either axis. **PCA finds new axes — the principal components — that are rotated to
align with the directions the data actually varies along**, rather than using the arbitrary
original axes.

The core idea: **variance = information.** A direction along which the data spreads out a lot is
a direction carrying a lot of distinguishing information about the points; a direction along which
all points look almost the same carries almost no information (you couldn't tell the points apart
along it). So:

- **The first principal component (PC1)** is the direction of **maximum variance** in the data —
  the single axis you could project every point onto while preserving the most spread (and
  therefore the most information).
- **The second principal component (PC2)** is the direction of the next-most variance, **subject
  to being orthogonal (perpendicular) to PC1** — it captures whatever spread PC1 missed, without
  duplicating it.
- Each subsequent PC continues this: orthogonal to all previous ones, capturing the most remaining
  variance.

Because the components are ordered by how much variance they capture, you can keep only the
**top k** components and drop the rest, projecting the original `d`-dimensional data down onto a
`k`-dimensional subspace (`k < d`) — reducing dimensionality while throwing away as little
information (variance) as possible.

---

## 5. PCA — Problem Formulation & Step-by-Step Solution

Formally, PCA is looking for a new orthogonal basis (the principal components) such that
projecting the data onto it maximizes variance along the first axis, then the second, and so on.
This is equivalent to two other framings that all arrive at the same answer: (1) maximizing the
variance of the projections, and (2) minimizing the **reconstruction error** — the squared
distance between original points and their projections back from the reduced space. Maximizing
spread along a direction and minimizing what's lost by summarizing points onto that direction are
two sides of the same coin.

The actual computation:

1. **Standardize the data.** Subtract the mean (center at 0) and divide by standard deviation for
   every feature. This is mandatory (see Section 6) — PCA is driven entirely by variance, and
   unscaled features with larger numeric ranges would dominate the components for no meaningful
   reason.
2. **Compute the covariance matrix.** For `d` features this is a `d × d` symmetric matrix where
   entry `(i, j)` is the covariance between feature `i` and feature `j` — it fully describes how
   every pair of features varies together.
3. **Eigen-decompose the covariance matrix.** This yields `d` **eigenvectors** and `d` matching
   **eigenvalues**.
   - Each **eigenvector** is a direction in the original feature space — these directions are
     exactly the principal components.
   - Each **eigenvalue** is the amount of variance captured *along* its eigenvector. Bigger
     eigenvalue = more information along that direction.
4. **Sort by eigenvalue, descending.** The eigenvector with the largest eigenvalue is PC1, the
   next is PC2, and so on. Eigenvectors of a symmetric matrix are automatically orthogonal, which
   is exactly the "each PC perpendicular to the last" property from the geometric picture.
5. **Pick the top `k` eigenvectors** and stack them as columns of a projection matrix.
6. **Project the data** by multiplying the standardized data by that projection matrix, giving a
   new dataset with only `k` columns — the **principal component scores**.

**SVD connection (high level):** in practice, libraries compute PCA via **Singular Value
Decomposition (SVD)** of the (centered) data matrix rather than literally forming the covariance
matrix and eigen-decomposing it — SVD is more numerically stable and avoids ever materializing a
potentially large `d × d` matrix. The singular values relate directly to the eigenvalues (variance
explained), and the right singular vectors are the same principal component directions. You don't
need to hand-derive this to use PCA, but know that "eigen-decomposition of the covariance matrix"
and "SVD of the data matrix" are the two equivalent routes to the same components — scikit-learn's
`PCA` uses SVD internally.

### Choosing `k`: explained variance ratio and the scree plot

Each eigenvalue's share of the total eigenvalue sum is that component's **explained variance
ratio** — the fraction of the dataset's total variance it accounts for. Plotting explained
variance ratio (or its cumulative sum) against component number gives a **scree plot**. Two common
ways to pick `k` from it:

- Pick the smallest `k` whose **cumulative** explained variance crosses a target, e.g. 95%.
- Look for the **"elbow"** in the scree plot — the point where additional components start adding
  only marginal variance — and cut there.

---

## 6. Practical Notes

- **Always scale features before PCA.** PCA maximizes variance, and variance is scale-dependent —
  a feature measured in the thousands (income) will swamp a feature measured in single digits (age)
  purely due to units, not actual signal. Use `StandardScaler` first, every time.
- **PCA is unsupervised.** It never looks at the target `y` — it only looks at the spread of `X`.
  It can therefore discard variance that happens to be exactly what predicts `y`, if that variance
  is small relative to other directions. It's a general-purpose compression, not a
  supervised-signal-preserving one.
- **Components are not interpretable original features.** PC1 is some linear combination like
  `0.6*height + 0.5*weight - 0.3*age + ...` — it doesn't correspond to any single real-world
  quantity, which is the price paid for feature *extraction* over feature *selection*.
- **Common uses:** visualizing high-dimensional data in 2D/3D (plot PC1 vs PC2), speeding up
  downstream model training by cutting feature count, and noise reduction (low-variance components
  are often mostly noise, so dropping them can act as a filter).
- **Limitations:** PCA is a **linear** method — it only finds straight-line (linear combination)
  directions of variance, so it can't capture curved/nonlinear structure in the data (that's what
  nonlinear techniques like t-SNE or kernel PCA are for, outside this playlist's scope). And once
  you're in PCA space, you've traded interpretability for compactness.

## 7. Minimal Code

```python
from sklearn.preprocessing import StandardScaler
from sklearn.decomposition import PCA

# 1. Scale first — mandatory
X_scaled = StandardScaler().fit_transform(X)

# 2. Fit PCA, e.g. reduce to 2 components for visualization
pca = PCA(n_components=2)
X_pca = pca.fit_transform(X_scaled)

# 3. How much variance did we keep?
print(pca.explained_variance_ratio_)        # e.g. [0.62, 0.24] -> 86% retained in 2D
print(pca.explained_variance_ratio_.sum())

# Scree plot to pick k on a real dataset:
pca_full = PCA().fit(X_scaled)
import matplotlib.pyplot as plt
plt.plot(range(1, len(pca_full.explained_variance_ratio_) + 1),
         pca_full.explained_variance_ratio_.cumsum(), marker="o")
plt.xlabel("Number of components")
plt.ylabel("Cumulative explained variance")
```

---

## Key takeaways

- **Feature construction** builds new, more informative columns from existing ones (sums, ratios,
  domain-driven flags); **feature splitting** pulls a compound column apart into its useful parts
  (name → title, date → year/month/day-of-week). Both are cheap, high-leverage MLDLC steps.
- **Curse of dimensionality:** more features → sparser data, less meaningful distances,
  exponentially more data needed to avoid overfitting. This motivates dimensionality reduction.
- **Feature selection** keeps a subset of original (interpretable) features; **feature extraction**
  builds new combined features. **PCA is extraction.**
- PCA finds orthogonal directions (**principal components**) of **maximum variance**, ranked by
  how much variance (information) each explains; projecting onto the top `k` compresses
  dimensions while keeping most of the signal.
- Mechanically: standardize → covariance matrix → eigen-decomposition (eigenvectors = component
  directions, eigenvalues = variance explained) → sort by eigenvalue → keep top `k` → project.
  Equivalent to maximizing variance / minimizing reconstruction error; computed via SVD in
  practice. Use the **explained variance ratio** / **scree plot** to choose `k`.
- PCA is unsupervised, requires scaled input, produces non-interpretable components, and is
  linear-only — useful for visualization, speed, and noise reduction, not a free lunch.

**Next:** [09 — Linear Regression](09-linear-regression.md) — the first predictive model, and
where regression metrics and assumptions come from.
