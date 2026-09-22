# 19 — Random Forest

**Playlist videos 108–114.** Bagging (Section 18) built ensembles of trees by bootstrapping rows.
Random Forest adds one more layer of randomness — random *feature* subsets at every split — and
that single addition is what makes it one of the strongest out-of-the-box algorithms in classical
ML. This section covers the intuition, why it works, how it differs from plain bagging, its
hyperparameters, tuning, OOB evaluation, and feature importance.

---

## 1. Introduction to Random Forest — Intuition

A **Random Forest** is a bagging ensemble specifically made of **decision trees**, with an extra
randomization step baked in: at **each split**, instead of considering all features, the tree
considers only a **random subset of features** and picks the best split among *those*.

So there are two independent sources of randomness per tree:

1. **Row sampling (bootstrap)** — each tree trains on a random sample of rows drawn *with
   replacement* from the training set (same as bagging, Section 18).
2. **Feature sampling** — at every individual split inside every tree, only a random subset of
   columns is even eligible to be split on (this is new — plain bagging doesn't do this).

Predictions are combined the same way as any bagging ensemble: **majority vote** for
classification, **average** for regression.

Why go to the trouble of also randomizing features? Bootstrapping rows alone still produces trees
that look fairly similar to each other, because if one feature is a very strong predictor (say,
"income" for a loan-default model), *almost every* bootstrapped tree will pick it at or near the
root, and the trees end up highly correlated. Random Forest breaks that correlation by sometimes
hiding the dominant feature from a split entirely, forcing the tree to discover other useful
features. The result is a forest of trees that each look at the problem from a different angle.

---

## 2. Why Random Forest Performs So Well — Bias–Variance Trade-off

Recall the bias–variance framing from Section 11. A single decision tree grown deep is:

- **Low bias** — it can fit almost any training pattern (it's a flexible, high-capacity model).
- **High variance** — small changes in the training data produce very different trees
  (unstable, hence "overfits").

Averaging (or voting over) many models reduces variance **only if the models' errors are not
perfectly correlated**. This is the core statistical fact behind every ensemble: for `n` models
each with variance `σ²` and average pairwise correlation `ρ`, the variance of the average is

```
Var(average) = ρσ² + (1 − ρ)σ²/n
```

As `n → ∞`, the second term vanishes but the first term, `ρσ²`, does **not** — it's a floor set
entirely by how correlated the trees are. This is the mathematical reason correlation matters more
than count once you have "enough" trees.

- **Plain bagging** reduces variance by lowering the `1/n` term (more trees, bootstrapped rows
  give some decorrelation) but `ρ` stays relatively high, because strong features dominate every
  tree's splits regardless of which rows were sampled.
- **Random Forest's feature subsampling directly attacks `ρ`.** By forcing different trees to
  split on different features, it decorrelates the trees far more than row-bootstrapping alone
  can. A lower `ρ` pulls the variance floor down, so the ensemble's overall variance drops further
  than bagging's does.

Meanwhile, bias barely moves — each individual tree is still grown deep and low-bias; you're not
constraining what any single tree can express, only what it's *allowed to look at* at each split.

**Net effect:** Random Forest keeps the low bias of deep trees while cutting variance more
aggressively than vanilla bagging, because de-correlation (not just averaging count) is the actual
lever on variance. This is the whole story of "why random forest works so well."

---

## 3. Bagging vs Random Forest — the Precise Difference

These are often described loosely as "the same thing," but there is one exact technical
difference:

| | Bagging (of trees) | Random Forest |
|---|---|---|
| Row sampling | Bootstrap (with replacement) | Bootstrap (with replacement) |
| Feature sampling at each split | **None — all features considered** | **Random subset considered** |
| Base estimator | Any model (trees are just common) | Always decision trees |
| Correlation between trees | Higher | Lower |

**Bagging** (`BaggingClassifier`/`BaggingRegressor` with a decision-tree base estimator) only
randomizes *which rows* each tree sees; every split in every tree still searches *every* feature
for the best split. **Random Forest** additionally randomizes *which features are even eligible*
at each split.

So: **Random Forest is bagging, specialized to trees, plus a second de-correlation mechanism.**
Every random forest is (structurally) a bagging ensemble; not every bagging ensemble is a random
forest. If you set a random forest's feature-subsampling knob (`max_features`) to "all features,"
it degenerates into ordinary tree bagging — which is a useful way to remember exactly what the
extra parameter buys you.

---

## 4. Random Forest Hyperparameters

All hyperparameters below trade off bias against variance; the two most important knobs are
`n_estimators` and `max_features`.

- **`n_estimators`** — number of trees in the forest. Averaging more decorrelated trees only
  reduces the `1/n` variance term, so more trees are (almost) always better or neutral for
  accuracy — they don't cause overfitting the way deeper trees do. Returns diminish quickly though
  (the error curve flattens), and training/prediction cost scales linearly, so this is really a
  compute-budget knob, not a real bias/variance lever.
- **`max_features`** — how many features are randomly considered at each split. **This is the key
  RF-specific knob.**
  - Smaller `max_features` → more decorrelation between trees → lower variance, but each split is
    chosen from a weaker candidate set → slightly higher bias per tree.
  - Larger `max_features` (up to "all") → trees look more like plain bagged trees → less
    decorrelation, more variance, but each split is locally more optimal.
  - Common defaults: `sqrt(n_features)` for classification, `n_features / 3` for regression —
    both scikit-learn conventions baked in from empirical experience.
- **`max_depth`** — max depth per tree. Shallower trees → higher bias, lower variance per tree
  (usually left deep/unbounded in RF, since the ensemble's averaging is what controls variance —
  unlike a standalone tree where you'd prune aggressively).
- **`min_samples_split` / `min_samples_leaf`** — minimum samples required to split a node / to be
  a leaf. Higher values force simpler, shallower trees — a bias↑/variance↓ move, same direction as
  reducing `max_depth`.
- **`bootstrap`** — whether row sampling is with replacement (`True`, the default) or each tree
  sees the full dataset (`False`). Turning it off removes the row-randomness source of
  decorrelation (and also disables OOB scoring — see Section 6, since without bootstrapping there
  is no "left-out" set).
- **`max_samples`** — if `bootstrap=True`, how many rows (or what fraction) each bootstrap draws.
  Smaller values increase per-tree randomness (more decorrelation, more bias per tree); default
  draws a sample the same size as the training set (with replacement, so ~63% unique rows — see
  Section 6).

Practical rule of thumb: start with defaults, bump `n_estimators` until the OOB/CV curve
flattens, then tune `max_features` and `max_depth`/`min_samples_leaf` together — they're the pair
that actually reshapes the bias–variance trade-off.

---

## 5. Hyperparameter Tuning — GridSearchCV vs RandomizedSearchCV

Both wrap cross-validation (Section 13-adjacent concept, formalized here) around a search over
hyperparameter values, then refit the best combination on the full training data.

- **`GridSearchCV`** — exhaustively tries **every combination** in an explicit grid you specify.
  Guaranteed to check the whole grid, but the number of fits is the *product* of the option counts
  across all parameters, so it explodes combinatorially. Use it when the search space is small
  (few parameters, few values each) and you want a guaranteed exhaustive answer.
- **`RandomizedSearchCV`** — samples a fixed number (`n_iter`) of random combinations from
  distributions (or lists) you specify per parameter. Cheaper for large spaces, and in practice
  finds near-optimal regions almost as well as a full grid because most hyperparameters have
  diminishing/flat returns past some point — a random scattering of trials tends to hit a "good
  enough" region without paying for the full combinatorial cost. Use it as the default for
  anything beyond 2–3 parameters, then optionally follow up with a small `GridSearchCV` around the
  best region found.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import GridSearchCV, RandomizedSearchCV

param_grid = {
    "n_estimators": [100, 200, 400],
    "max_features": ["sqrt", "log2", None],
    "max_depth": [None, 10, 20],
    "min_samples_leaf": [1, 2, 5],
}

grid = GridSearchCV(
    RandomForestClassifier(random_state=42),
    param_grid,
    cv=5,
    scoring="accuracy",
    n_jobs=-1,
)
grid.fit(X_train, y_train)
print(grid.best_params_, grid.best_score_)

# For a larger space, sample instead of enumerating:
random_search = RandomizedSearchCV(
    RandomForestClassifier(random_state=42),
    param_distributions=param_grid,
    n_iter=20,
    cv=5,
    scoring="accuracy",
    n_jobs=-1,
    random_state=42,
)
random_search.fit(X_train, y_train)
```

Both use `cv`-fold cross-validation internally to score each candidate, so the reported
`best_score_` is already a cross-validated estimate, not a single train/test split.

---

## 6. OOB Score — Out-of-Bag Evaluation

Bootstrapping (sampling `N` rows with replacement from `N` rows) has a well-known property: each
tree's bootstrap sample contains, on average, only about **63.2%** of the unique rows
(`1 − 1/e ≈ 0.632` as `N → ∞`). The remaining **~36.8%** of rows are never seen by that particular
tree — they are its **out-of-bag (OOB)** rows.

This gives every tree a free, built-in validation set drawn from data it never trained on:

- For each row in the training set, collect predictions only from the trees that did **not** use
  that row in their bootstrap sample.
- Aggregate those out-of-bag predictions (vote/average) and compare against the true label to get
  the **OOB score/error**.

The result is an estimate of generalization performance **without holding out a separate
validation set and without running cross-validation** — you get a CV-like estimate essentially for
free, using data you were already going to train on. It's not identical to k-fold CV (each row is
evaluated by a different, smaller sub-ensemble of trees — roughly 37% of the forest, not all of
it), but in practice it tracks CV/test error closely and is much cheaper to compute since it comes
as a byproduct of training.

```python
rf = RandomForestClassifier(n_estimators=300, oob_score=True, bootstrap=True, random_state=42)
rf.fit(X_train, y_train)
print(rf.oob_score_)   # accuracy estimated purely from each tree's held-out rows
```

`oob_score=True` requires `bootstrap=True` (otherwise there's no held-out set to speak of). It's
most useful as a quick sanity check / model-selection signal when you don't want to burn data on a
separate validation split, but for final reporting or careful tuning, cross-validation is still the
more standard and more scrutinized choice.

---

## 7. Feature Importance using Random Forest and Decision Trees

Both single decision trees and random forests give you a **feature importance** score for free, as
a side effect of how they're built.

**How it's computed (Gini importance / Mean Decrease in Impurity, MDI):**

1. Every split in every tree reduces impurity (Gini/entropy for classification, variance/MSE for
   regression) by some amount, weighted by the number of samples reaching that node.
2. For a given feature, sum up the impurity decrease over **every split that used that feature**,
   across **every tree** in the forest (for a single tree, just across that tree's splits).
3. Normalize so all feature importances sum to 1.

A feature that's picked often, and produces large impurity drops when picked, ends up with high
importance. This is exactly `feature_importances_` in scikit-learn for both `DecisionTreeClassifier`
and `RandomForestClassifier`.

```python
import pandas as pd

rf = RandomForestClassifier(n_estimators=300, random_state=42)
rf.fit(X_train, y_train)

importances = pd.Series(rf.feature_importances_, index=X_train.columns)
importances.sort_values(ascending=False)
```

**Caveat — MDI is biased toward high-cardinality / continuous features.** Because a feature with
many possible split points (e.g. a continuous variable, or a categorical with many levels) simply
gets *more chances* to be tried at each node, it tends to accumulate importance even if it's not
genuinely more predictive than a coarser feature. Low-cardinality binary features can look
artificially unimportant purely because they offer fewer distinct splits, not because they carry
less signal.

**More reliable alternative — permutation importance.** Instead of reading impurity decrease off
the tree structure, shuffle (permute) one feature's column in the validation/OOB data and measure
how much the model's score *drops*. A feature the model actually relies on will hurt performance
when scrambled; an unused feature won't move the score at all. This measures the feature's actual
contribution to predictive performance rather than an artifact of how many split candidates it
offered, and is far less biased by cardinality — at the cost of being more expensive to compute
(requires re-scoring the model once per feature).

---

## 8. Pros and Cons

**Pros:**
- Strong out-of-the-box accuracy with minimal tuning — a reasonable default before reaching for
  boosting.
- Robust to outliers and to irrelevant features (each tree is somewhat insulated by row/feature
  subsampling).
- Handles non-linear relationships and feature interactions natively (tree splits), no scaling
  required.
- Gives feature importances for free.
- Trees train independently — trivially **parallelizable** (`n_jobs=-1`).
- Free generalization estimate via OOB score, no extra validation split needed.

**Cons:**
- Loses the single-tree's clean interpretability — a forest of hundreds of trees isn't something
  you can read like a flowchart.
- Larger memory footprint and slower prediction than a single tree (querying every tree in the
  forest and aggregating).
- Can still overfit on very noisy data or if trees are grown unconstrained and the noise itself
  is somewhat consistent across bootstrap samples — averaging reduces variance but is not a magic
  shield against a truly bad signal-to-noise ratio.
- Weaker than specialized approaches on very high-dimensional sparse data (e.g. text
  bag-of-words) — random feature subsampling struggles when almost all features are individually
  uninformative and only sparse combinations matter, a regime linear models or boosting variants
  often handle better.

---

## Key takeaways

- Random Forest = bagging of decision trees + random feature subsets at every split. The second
  ingredient is what makes it distinct from — and generally better than — plain tree bagging.
- The bias–variance win comes from **de-correlating** the trees (lowering `ρ` in the ensemble
  variance formula), not just from averaging more of them; feature subsampling is the lever that
  attacks correlation directly.
- Bagging vs RF, precisely: bagging considers all features per split (only rows are randomized);
  RF randomizes rows **and** restricts the feature candidates per split.
- `n_estimators` is a compute/diminishing-returns knob; `max_features` is the real bias–variance
  lever; `max_depth`/`min_samples_leaf` control per-tree complexity.
- Use `RandomizedSearchCV` for large hyperparameter spaces, `GridSearchCV` for small/exhaustive
  ones — both are cross-validated searches.
- OOB score exploits the ~63%/~37% bootstrap split to get a free, CV-like validation estimate with
  no extra held-out data (`oob_score=True`).
- Feature importance from trees/forests is impurity decrease (MDI) — fast but biased toward
  high-cardinality features; permutation importance is the more trustworthy (if pricier)
  alternative.

**Next:** [20 — AdaBoost](20-adaboost.md) — moving from bagging (parallel, variance-reduction) to
boosting (sequential, bias-reduction).
