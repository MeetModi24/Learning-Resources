# 20 — AdaBoost

**Playlist videos 115–119.** Section 18–19 covered bagging (parallel, variance-reducing
ensembles). This section covers the other ensemble family: **boosting**, starting with the
original boosting algorithm, AdaBoost, then closing with a direct bagging-vs-boosting comparison.

---

## 1. Boosting: the big idea

**Boosting builds models sequentially**, not in parallel. Each new model is trained specifically
to fix the mistakes of the ensemble built so far. The models are combined into a single strong
learner, but they are not independent the way bagged trees are — each one exists *because of* the
errors of the ones before it.

Boosting's building blocks are deliberately **weak learners**: models only slightly better than
random guessing. The classic weak learner is a **decision stump** — a decision tree of depth 1
(one split, two leaves). A single stump is a bad classifier on its own. Boosting's claim is that
a long, carefully-sequenced chain of weak learners, each correcting the last, can combine into a
strong classifier.

This is the opposite failure mode from bagging:

- **Bagging** starts with a low-bias, high-variance base learner (a fully grown tree) and averages
  many of them in parallel to **reduce variance**.
- **Boosting** starts with a high-bias, low-variance base learner (a stump) and chains many of
  them sequentially, each one targeting what's still wrong, to **reduce bias**.

Both end up trading a bad property of a single weak model for a good ensemble — they just start
from opposite ends of the bias–variance trade-off (Section 11, Section 19 §2).

---

## 2. AdaBoost — step by step

**AdaBoost (Adaptive Boosting)** is the original boosting algorithm. "Adaptive" refers to how it
adapts sample weights round by round to force each new stump to focus on whatever the ensemble is
still getting wrong.

Given `N` training points, the algorithm runs for a fixed number of rounds. Each round:

1. **Sample weights.** Every training point has a weight — how much it "counts" for the next
   stump. Initially all `N` points get equal weight `1/N`.

2. **Train a weak learner.** Fit a decision stump on the (weighted) training data. The stump
   picks whichever single feature/split best separates the classes given the *current* weights —
   so as weights shift in later rounds, different stumps get picked because different points now
   matter more.

3. **Compute the stump's error.** The **weighted error rate** — sum the weights of the
   misclassified points (not just a raw count), since a misclassified heavily-weighted point hurts
   more than a misclassified lightly-weighted one.

4. **Compute the stump's "say" (alpha).** Each stump gets a vote weight, α, based on its error:
   lower error → higher α (more say in the final vote). A stump with error near 0.5 (no better
   than a coin flip) gets α near zero — it contributes almost nothing to the final vote. A stump
   with very low error gets a large positive α. (The exact form is
   `α = ½ · ln((1 − error) / error)`; what matters is the shape — α grows without bound as error
   → 0, is 0 at error = 0.5, and goes negative if error > 0.5, i.e. a consistently-wrong stump is
   flipped and still used.)

5. **Re-weight the samples.** This is the "adaptive" step:
   - Points the stump **got wrong** have their weight **increased**.
   - Points the stump **got right** have their weight **decreased**.
   - Weights are then **normalized** back to sum to 1.

   The intuition: whatever this stump missed becomes *more prominent* in the data the next stump
   sees, so the next stump is effectively forced to specialize on exactly the cases the ensemble
   is currently failing on. Round by round, the ensemble's attention shifts toward the hardest
   points.

6. **Repeat** for the configured number of rounds (or until the error is low enough), each time
   training a fresh stump on the newly re-weighted data.

**Final prediction.** AdaBoost doesn't take a plain majority vote — it takes a **weighted vote**
using each stump's α: sum up `α_i × prediction_i` across all stumps and take the sign (binary
classification) or the weighted majority class (multiclass). Stumps that turned out more accurate
get more influence on the final call than stumps that barely beat chance.

---

## 3. AdaBoost from scratch — what the code actually does

You won't need to hand-roll AdaBoost in practice (scikit-learn ships it), but the shape of a
from-scratch implementation is worth having in your head because it makes the weight/alpha
mechanics concrete:

```python
import numpy as np
from sklearn.tree import DecisionTreeClassifier

def adaboost_fit(X, y, n_estimators=50):
    # y assumed in {-1, +1}
    n = X.shape[0]
    sample_weights = np.full(n, 1 / n)     # step 1: equal weights
    stumps, alphas = [], []

    for _ in range(n_estimators):
        stump = DecisionTreeClassifier(max_depth=1)          # step 2: weak learner
        stump.fit(X, y, sample_weight=sample_weights)
        pred = stump.predict(X)

        incorrect = (pred != y)
        error = np.sum(sample_weights[incorrect])            # step 3: weighted error
        error = np.clip(error, 1e-10, 1 - 1e-10)              # avoid log(0)

        alpha = 0.5 * np.log((1 - error) / error)             # step 4: stump's "say"

        sample_weights *= np.exp(-alpha * y * pred)           # step 5: reweight
        sample_weights /= sample_weights.sum()                # normalize

        stumps.append(stump)
        alphas.append(alpha)

    return stumps, alphas

def adaboost_predict(X, stumps, alphas):
    votes = sum(a * s.predict(X) for s, a in zip(stumps, alphas))  # step 6: weighted vote
    return np.sign(votes)
```

The reweighting line is the whole algorithm in one expression: `exp(-alpha * y * pred)` is `> 1`
(weight goes up) when `y * pred < 0`, i.e. misclassified, and `< 1` (weight goes down) when
correctly classified — scaled by how confident/accurate that stump was (`alpha`).

---

## 4. Hyperparameters

```python
from sklearn.ensemble import AdaBoostClassifier
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import GridSearchCV

base = DecisionTreeClassifier(max_depth=1)   # default base estimator is already a stump

model = AdaBoostClassifier(estimator=base, n_estimators=50, learning_rate=1.0)

param_grid = {
    "n_estimators": [50, 100, 200],
    "learning_rate": [0.01, 0.1, 0.5, 1.0],
}
grid = GridSearchCV(AdaBoostClassifier(estimator=base), param_grid, cv=5, scoring="accuracy")
grid.fit(X_train, y_train)
```

- **`estimator`** (weak learner to boost) — defaults to a depth-1 decision stump. You *can* swap
  in a deeper tree or another classifier, but the whole point of AdaBoost is chaining weak
  learners; using a strong base learner defeats the design and tends to overfit fast.
- **`n_estimators`** — how many rounds/stumps to add. More rounds keeps reducing bias, but past a
  point returns diminish and, on noisy data, overfitting risk climbs.
- **`learning_rate`** — shrinks each stump's contribution (its α) before it's added to the vote.
  Lower learning rate = each round contributes less = you need *more* `n_estimators` to reach the
  same fit. This is the same shrinkage-vs-rounds trade-off you'll see again with gradient boosting
  in Section 21: a smaller learning rate with more estimators is usually the more robust
  combination, at the cost of training time.
- Tune both together with **`GridSearchCV`** (or `RandomizedSearchCV` for a larger grid) — they
  interact, so tuning one at a time can miss the best combination.

---

## 5. Bagging vs Boosting

| | Bagging | Boosting |
|---|---|---|
| Model training | **Parallel** — each model trained independently | **Sequential** — each model depends on the previous ones |
| Data per model | Random bootstrap sample, **equal** weight on all rows | **Full data, re-weighted** each round to emphasize prior mistakes |
| Model weighting in final vote | Equal vote per model (or averaged) | **Weighted vote** — better models (lower error → higher α) count more |
| What it targets | Reduces **variance** | Reduces **bias** |
| Base learner | Strong, low-bias, high-variance (e.g. full-depth trees) | Weak, high-bias, low-variance (e.g. stumps) |
| Overfitting behavior | Resistant — averaging independent models smooths out noise | Can overfit, especially with noisy labels/outliers |
| Sensitivity to outliers/mislabeled data | Low — one bad bootstrap sample doesn't dominate | High — boosting *chases* misclassified points, so it keeps upweighting outliers/mislabeled rows round after round |
| Example algorithm | Random Forest | AdaBoost, Gradient Boosting, XGBoost |

The overfitting/outlier row is the practical takeaway: boosting's core mechanism — increase the
weight of whatever you got wrong — is exactly what makes it vulnerable to noisy or mislabeled data.
A genuinely mislabeled point *looks* hard, so boosting keeps raising its weight round after round
and effectively lets one bad row distort many stumps. Bagging has no such mechanism; a bad point
just sits in whichever bootstrap samples happen to include it.

---

## 6. Pros and cons

**Pros**
- Often reaches high accuracy despite using only very simple base learners (stumps).
- Conceptually simple: the whole algorithm is "reweight and repeat."
- Fewer moving parts to tune than some ensembles — mainly `n_estimators` and `learning_rate`.

**Cons**
- **Sensitive to noise and outliers** — see §5; mislabeled data can badly distort the ensemble.
- **Sequential training is slower** than bagging's parallel training — you can't fit stump `k+1`
  until stump `k`'s errors are known.
- Can still overfit if run for too many rounds on noisy data, despite each individual learner
  being weak.

---

## Key takeaways

- Boosting builds weak learners **sequentially**, each one correcting the errors of the ensemble
  so far, and combines them into a strong learner. It primarily reduces **bias**, unlike bagging's
  variance reduction.
- AdaBoost's loop: equal weights → train a stump → compute its weighted error → convert error to
  a vote weight α (lower error → higher α) → **increase weights on misclassified points**,
  decrease on correct ones, normalize → repeat. Final prediction is a weighted vote by α.
- Key hyperparameters: `n_estimators` (rounds), `learning_rate` (shrinks each round's
  contribution, trades off against `n_estimators`), and `estimator` (base learner, default depth-1
  stump). Tune with `GridSearchCV`.
- Bagging is parallel/variance-reducing/outlier-resistant; boosting is sequential/bias-reducing
  and outlier-sensitive because it keeps upweighting whatever it gets wrong — including noise.

**Next:** [21 — Gradient Boosting & XGBoost](21-gradient-boosting-xgboost.md) — boosting
generalized to arbitrary loss functions via gradients, and the engineering behind XGBoost.
