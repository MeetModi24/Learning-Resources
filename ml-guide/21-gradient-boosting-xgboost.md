# 21 — Gradient Boosting & XGBoost

**Playlist videos 120–126.** AdaBoost (Section 20) fixes mistakes by *re-weighting* misclassified
samples so the next weak learner pays more attention to them. Gradient boosting fixes mistakes a
different way: each new tree is trained to predict the **residual errors** of the current ensemble.
That single shift — errors as a target instead of sample weights — turns out to be gradient descent
performed in function space, and it's the idea underneath the most dominant tabular-data algorithm
of the last decade, XGBoost.

---

## 1. Gradient Boosting — Core Idea (video 120)

**Setup (regression, easiest case):** you have data `(x_i, y_i)` and want an ensemble of trees that
predicts `y`.

1. **Start with a dumb baseline prediction.** For squared-error loss, the optimal constant
   prediction is the **mean of `y`**. Call this `F_0(x) = mean(y)`.
2. **Compute residuals.** `r_i = y_i - F_0(x_i)` — how wrong the current model is, per sample.
3. **Fit a tree to the residuals**, not to `y`. This tree, `h_1(x)`, learns to predict the *error*
   the baseline made.
4. **Update the ensemble additively, shrunk by a learning rate `η`:**
   `F_1(x) = F_0(x) + η · h_1(x)`.
5. **Repeat:** compute new residuals against `F_1`, fit `h_2` to those, update to `F_2`, and so on
   for `n_estimators` rounds.

Final prediction: `F_M(x) = F_0(x) + η·h_1(x) + η·h_2(x) + ... + η·h_M(x)`.

This is an **additive model built sequentially** — same family as AdaBoost — but the mechanism of
"pay attention to what's wrong" is completely different:

| | AdaBoost | Gradient Boosting |
|---|---|---|
| What changes each round | Sample **weights** (misclassified points get heavier) | The **target** each new tree fits (residuals of current ensemble) |
| Weak learner combination weight | `α` derived from each learner's error rate | Fixed shrinkage `η` (learning rate) applied to every tree |
| Natural home | Mostly classification | Regression first, generalizes to classification via log-odds |

Each tree is typically a **shallow** decision tree (a handful of leaves) — a weak learner, same
philosophy as AdaBoost's stumps, just slightly deeper since it needs to model residual structure,
not just a coarse split.

---

## 2. The Maths — Why "Gradient" (video 121)

Fitting a tree to residuals looks like a hack. It isn't — it's a special case of a general
procedure: **gradient descent in function space**.

In ordinary gradient descent (Section 10) you have parameters `θ` and update
`θ ← θ - η · ∇L(θ)` — you step the parameters in the direction that reduces the loss fastest.

In gradient boosting, the "parameter" being optimized is the **entire function** `F(x)` (the
ensemble's prediction), and at each round you add a small step in the direction that reduces the
loss fastest — approximated by a tree, because you can only step in directions a tree can express.

For squared-error loss `L(y, F) = ½(y - F)²`, the negative gradient with respect to `F` is:

```
-∂L/∂F = -(F - y) = y - F
```

That is exactly the **residual**. So "fit a tree to the residuals" *is* "fit a tree to the negative
gradient of the loss" — for squared error these are numerically identical, which is why the
regression case looks like plain residual-fitting with no mention of calculus.

The general algorithm, for **any differentiable loss** `L`:

1. Initialize `F_0(x)` = the constant that minimizes `L` (mean for squared error, log-odds for
   log-loss — see below).
2. For each round `m = 1..M`:
   - Compute the **pseudo-residuals**: `r_i = -∂L(y_i, F)/∂F` evaluated at the current ensemble.
   - Fit a tree `h_m(x)` to the pseudo-residuals.
   - Update: `F_m(x) = F_{m-1}(x) + η · h_m(x)`.

This generalization is the entire point of the name: swap in a different loss (log-loss, Huber,
quantile loss) and the same machinery works — only the pseudo-residual formula changes. Squared
error is just the case where "negative gradient" and "residual" happen to coincide exactly.

---

## 3. Gradient Boosting for Classification (video 122)

For binary classification you can't average `0`/`1` labels the way you average continuous `y`. The
model instead works in **log-odds space**, exactly like logistic regression (Section 12):

1. **Initial prediction** `F_0` = log-odds of the base rate: `log(p / (1-p))` where `p` is the
   fraction of positive-class samples.
2. Use **log-loss** as the objective. Its negative gradient (the pseudo-residual) turns out to be:

   ```
   r_i = y_i - p_i
   ```

   where `p_i = sigmoid(F(x_i))` is the current predicted probability. This is "residual" in
   *probability* space even though the model itself is additive in *log-odds* space — the actual
   value being fit is the gap between the true label (0 or 1) and the current predicted probability.
3. Fit a tree to those pseudo-residuals, add it (shrunk by `η`) to the log-odds ensemble.
4. Repeat. At the end, pass the accumulated log-odds `F_M(x)` through the **sigmoid** to get a
   probability (softmax for multiclass, one set of trees per class per round).

**Geometric intuition:** each tree nudges the decision boundary a little, in the direction that
most reduces confident wrong predictions (samples with `y_i - p_i` far from zero — either a
confidently-wrong or a barely-right prediction) — the same "focus on what's currently wrong" spirit
as AdaBoost, but expressed as a continuous gradient signal rather than discrete reweighting.

---

## 4. Key Hyperparameters

- **`n_estimators`** — number of boosting rounds (trees). More rounds fit the training data more
  closely.
- **`learning_rate` (shrinkage, `η`)** — how much of each new tree's prediction gets added. Smaller
  `η` means each tree contributes less, so you need *more* trees (`n_estimators`) to reach the same
  fit — but the resulting ensemble generalizes better. This is the central trade-off: **low
  learning_rate + high n_estimators** is the standard recipe for good generalization; a high
  learning rate converges fast but overfits and is jumpy.
- **`max_depth`** — depth of each individual tree. Kept shallow (2–5 is typical) — these are meant
  to be weak learners; deep trees per round overfit fast in an additive ensemble.
- **`subsample`** — fraction of training rows sampled (without replacement) for each tree, i.e.
  **stochastic gradient boosting**. `subsample < 1.0` adds randomness (reduces variance and
  correlation between trees, similar in spirit to bagging) and speeds up training, at the cost of
  a noisier gradient estimate per round.

---

## 5. Introduction to XGBoost (video 123)

**XGBoost (Extreme Gradient Boosting)** is not a new algorithm family — it's a heavily optimized,
regularized, engineered implementation of the gradient boosting idea above. What sets it apart:

**1. Regularization built into the objective.** The objective XGBoost minimizes each round is
   `loss + Ω(tree)`, where `Ω` penalizes tree complexity directly — L2 (and optionally L1)
   penalties on the **leaf weights** (the `λ` and `α` hyperparameters), plus a penalty per leaf
   (`γ`, used for pruning — see below). Plain gradient boosting has no such term; it relies purely
   on shrinkage + shallow trees to avoid overfitting. XGBoost adds a second, explicit lever.

**2. Second-order optimization.** Plain gradient boosting only uses the **gradient** (first
   derivative) of the loss to decide what each tree should fit. XGBoost also uses the **Hessian**
   (second derivative), via a second-order Taylor expansion of the loss around the current
   prediction. This gives a more informed, curvature-aware step and a closed-form way to score
   candidate splits and leaf values:

   ```
   similarity score (a leaf) = (Σ gradients)² / (Σ hessians + λ)
   gain (a split)            = similarity(left) + similarity(right) - similarity(root)
   ```

   Intuitively: a leaf's score rewards a *large, consistent* sum of gradients (the leaf's samples
   are all wrong in the same direction — a signal worth acting on) and penalizes a small or noisy
   sum, damped by `λ` (regularization — the leaf weight shrinks toward zero unless there's enough
   Hessian "evidence" behind it). The tree is grown split by split, greedily picking whichever split
   maximizes `gain`; a split with `gain < γ` (the pruning threshold) isn't worth the added
   complexity and is pruned back.

**3. Engineering that plain gradient boosting implementations don't have:**
   - **Parallelized tree construction** — the sequential *rounds* still happen one after another
     (boosting is inherently sequential), but finding the best split *within* a tree is
     parallelized across features/blocks.
   - **Cache-aware access patterns** and **out-of-core computation** for data larger than memory.
   - **Sparsity-aware split finding** — missing values are handled natively; the algorithm learns a
     default direction (left/right) for missing entries at each split instead of requiring
     imputation.
   - **Approximate split finding** (histogram/quantile-sketch based) for speed on large datasets,
     vs. exhaustively scanning every possible split point.
   - **Column subsampling** (`colsample_bytree`, analogous to Random Forest's feature subsampling)
     alongside row `subsample`, adding another decorrelation lever.

None of this changes the *core* idea from Section 1–3 — additive trees fit to gradients of a loss.
XGBoost is that idea made fast, regularized, and missing-value-robust enough to run on real,
messy, large tabular data.

---

## 6. XGBoost for Regression & Classification (videos 124–125)

The mechanics are the same additive-boosting loop as Sections 1–3, with the leaf values now chosen
by the similarity-score formula instead of a plain average of residuals:

- **Regression:** initial prediction = mean of `y` (or a configurable base score). Gradients and
  Hessians are computed from squared-error loss (gradient = residual, Hessian = constant = 1 for
  squared error, which is why plain gradient boosting regression trees end up looking like a
  simpler special case of this). Each leaf's output value is
  `leaf_output = -Σ gradients / (Σ hessians + λ)` — the regularized, Hessian-weighted average of the
  residuals falling into that leaf, not a plain mean.
- **Classification:** same log-odds framing as Section 3 — gradients come from log-loss
  (`p_i - y_i`, sign convention aside) and Hessians from `p_i(1 - p_i)`. Leaf outputs are again the
  regularized ratio above, then the accumulated log-odds pass through sigmoid/softmax for the final
  probability.

In both cases, tree growth proceeds split-by-split, picking the split with the highest `gain`
(Section 5) at each node, subject to `max_depth`, `min_child_weight` (a floor on `Σ hessians` in a
leaf — controls how small a leaf is allowed to get), and `gamma` (the pruning threshold).

---

## 7. Why XGBoost Dominates Tabular Competitions

For structured/tabular data (rows and columns, mixed numeric/categorical features, no strong
spatial/sequential structure), XGBoost has been the most consistently winning algorithm on Kaggle
and similar competitions for years, because it combines: strong accuracy out of the box, built-in
regularization that resists overfitting better than vanilla gradient boosting, native handling of
missing values, and speed that makes extensive hyperparameter search (Section 25) practical.

**LightGBM** and **CatBoost** are siblings in the same "gradient boosting done fast and
regularized" family — LightGBM grows trees leaf-wise with histogram-based splitting for extra
speed on large datasets, CatBoost specializes in native categorical-feature handling — but the
core mechanism (additive trees fitting gradients/Hessians of a loss, with regularization) is the
same idea covered above.

---

## 8. Pros and Cons

**Pros:**
- Typically the strongest out-of-the-box accuracy among classical ML algorithms on tabular data.
- Built-in L1/L2 regularization (via `λ`, `α`) and pruning (`γ`) resist overfitting better than
  plain gradient boosting.
- Handles missing values natively — no imputation step required.
- Fast relative to its accuracy, thanks to the engineering optimizations in Section 5.
- Works for regression, classification, and ranking with the same core engine.

**Cons:**
- Many hyperparameters (`n_estimators`, `learning_rate`, `max_depth`, `subsample`,
  `colsample_bytree`, `gamma`, `lambda`, `alpha`, `min_child_weight`, ...) — tuning is nontrivial
  and often needs a systematic search (Section 25).
- Still boosting: **sequential** by nature (each tree depends on the previous ensemble's residuals),
  so training doesn't parallelize across rounds the way bagging/Random Forest does.
- Can still overfit if `n_estimators` is too high relative to `learning_rate` and regularization,
  especially on small or noisy datasets — early stopping on a validation set is standard practice.
- Less interpretable than a single decision tree — an ensemble of hundreds of trees has no simple
  "read the splits" explanation, though feature importance and SHAP values are common workarounds.

---

## 9. Minimal Code

```python
from sklearn.ensemble import GradientBoostingClassifier
from xgboost import XGBClassifier

# scikit-learn's built-in gradient boosting (first-order only, no Hessian, no leaf regularization)
gbc = GradientBoostingClassifier(
    n_estimators=200,
    learning_rate=0.05,
    max_depth=3,
    subsample=0.8,
)
gbc.fit(X_train, y_train)
preds = gbc.predict(X_test)

# XGBoost — regularized, second-order, engineered implementation of the same idea
xgb = XGBClassifier(
    n_estimators=200,
    learning_rate=0.05,
    max_depth=3,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_lambda=1.0,     # L2 on leaf weights
    reg_alpha=0.0,      # L1 on leaf weights
    gamma=0.0,          # min gain to allow a split (pruning threshold)
    eval_metric="logloss",
)
xgb.fit(X_train, y_train)
preds = xgb.predict(X_test)
```

---

## Key takeaways

- Gradient boosting builds an additive ensemble sequentially, like AdaBoost, but each new tree fits
  the **residuals** (errors) of the current ensemble instead of reweighting samples.
- For squared-error loss, residuals *are* the negative gradient of the loss — so this is gradient
  descent performed in function space, and it generalizes to any differentiable loss via
  **pseudo-residuals**.
- Classification works in **log-odds space**: pseudo-residuals are `y - p` from log-loss, and final
  predictions pass through sigmoid/softmax.
- `learning_rate` and `n_estimators` trade off against each other: smaller learning rate needs more
  trees but generalizes better. `max_depth` keeps individual trees weak; `subsample` adds
  stochasticity.
- **XGBoost** = gradient boosting + explicit regularization (L1/L2 on leaf weights, pruning via
  `gamma`) + second-order (gradient + Hessian) split/leaf scoring (`similarity score`, `gain`) +
  heavy systems engineering (parallel split-finding, sparsity/missing-value awareness, approximate
  histograms). The core sequential-residual-fitting idea is unchanged; what changes is how well each
  step is chosen and how fast it runs.
- LightGBM and CatBoost are the same family with different engineering trade-offs (leaf-wise growth,
  native categorical handling).

**Next:** [22 — Stacking & Blending](22-stacking-blending.md).
