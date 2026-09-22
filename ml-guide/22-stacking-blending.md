# 22 — Stacking and Blending

**Playlist video 127.** The fourth and last ensemble family, after voting (18), bagging (18),
and boosting (20–21). Voting and bagging combine models with a *fixed* rule (majority vote or
average). Boosting combines them sequentially, each one correcting the last. Stacking takes a
different approach: it **learns** how to combine the base models, by training a model on top of
them.

---

## 1. The Core Idea

Train several diverse **base models** (called **level-0** models) on the same training data —
say a KNN, a decision tree, and an SVM. Each one makes predictions. Instead of averaging those
predictions or voting on them, feed them as **input features** to a new model, the **meta-model**
(or **blender**, **level-1** model), which learns the optimal way to combine them.

```
        ┌──────────┐
        │  X_train │
        └────┬─────┘
     ┌────────┼────────┐
     ▼        ▼         ▼
  Model 1  Model 2   Model 3      (level-0 / base models)
     │        │         │
     ▼        ▼         ▼
   pred1    pred2     pred3
     └────────┼────────┘
              ▼
     [pred1, pred2, pred3]  →  Meta-model  →  final prediction
                                (level-1 / blender)
```

Why this can beat voting: a fixed rule like averaging implicitly assumes every base model deserves
equal weight and that a simple linear combination is enough. The meta-model instead **learns
weights**, and if it's expressive enough, non-linear combination rules — e.g. "trust Model 1 when
Model 2 and Model 3 disagree, otherwise trust Model 2." It can discover that some base model is
only good on a subset of the input space and lean on it selectively. Voting can never do that; it
has no way to condition on the situation.

The trade-off is exactly what you'd expect from adding a learned layer: more complexity, more
moving parts, and a real risk of **overfitting** if the meta-model gets to see predictions that
are already contaminated by leakage (Section 3 explains why).

---

## 2. Why You Can't Just Reuse the Training Predictions

The naive version: train each base model on `X_train`, get their predictions *on `X_train`
itself*, and feed those predictions as features to the meta-model. This is a data leakage bug.

A model that predicts on the same data it was trained on gives predictions that are artificially
close to the true labels — it has already "seen the answer." A model that overfits badly (e.g. an
unpruned decision tree that memorizes the training set) would look like a perfect predictor on
its own training data, even though it generalizes terribly. The meta-model, taking these
predictions at face value, would learn to over-trust exactly the base models that overfit the
most — the opposite of what you want. At test/inference time, when base models predict on genuinely
unseen data, their outputs won't have this artificial reliability, and the whole ensemble
collapses.

The fix is to make sure the meta-model only ever trains on predictions that are honest, held-out
predictions.

---

## 3. Stacking: Out-of-Fold Predictions via K-Fold CV

The standard fix is **K-fold cross-validation** to generate the meta-model's training features:

1. Split `X_train` into `K` folds.
2. For each fold `i`: train each base model on the other `K−1` folds, predict on fold `i`.
3. After looping over all `K` folds, every training row has a prediction from each base model —
   but crucially, that prediction always came from a model that **never saw that row during
   training**. These are the **out-of-fold (OOF) predictions**.
4. Stack the OOF predictions from all base models into a new feature matrix; train the meta-model
   on this matrix against the original labels `y_train`.
5. For the final base models used at inference time, retrain each one on the *full* `X_train`
   (not just K−1 folds) — you want the strongest possible base models in production. Only the
   meta-model's *training features* needed the fold discipline.
6. At prediction time on new data: run it through all (fully-trained) base models to get
   predictions, feed those into the trained meta-model, get the final output.

This is exactly the same discipline as cross-validation for model evaluation (Section 13) — the
model must never be scored on data it trained on. Here it's not scoring, it's *feature generation
for another model*, but the leakage mechanism is identical.

---

## 4. Blending: A Simpler Variant

**Blending** sidesteps K-fold CV entirely by using a plain train/holdout split:

1. Split the training data into a **train** set and a **holdout** (validation) set.
2. Train each base model on the train set only.
3. Get each base model's predictions **on the holdout set** — these predictions are honest because
   the holdout set was never used to fit the base models.
4. Train the meta-model on the holdout predictions vs. the holdout's true labels.
5. At inference, base models (trained on the train set) predict on new data, and those predictions
   feed the meta-model, same as stacking.

**Blending vs. stacking:**

| | Stacking (K-fold) | Blending (holdout) |
|---|---|---|
| Leakage protection | Out-of-fold predictions across K folds | A single held-out split |
| Data efficiency | High — every training row gets used to generate a meta-feature | Lower — the holdout slice is "spent" and not used to train base models |
| Meta-model training set size | Full training set size | Only the holdout size (a fraction of training data) |
| Implementation complexity | Higher — needs a CV loop, more bookkeeping | Simpler — one split, no folds |
| Risk | More code paths, easier to introduce a subtle leakage bug | Less signal for the meta-model since it trains on fewer examples |

Blending trades data efficiency for simplicity: it's less code, less room for a CV-implementation
bug, but the meta-model only ever sees the (small) holdout set, whereas stacking's meta-model
effectively gets to learn from the entire training set's worth of OOF predictions.

---

## 5. Choice of Meta-Learner

The meta-model sits on top of predictions that are already fairly informative (each base model
already tries to solve the task). Because of this, the meta-model doesn't need to be complex —
in fact, a complex meta-model is the fastest way to overfit the ensemble, since it's fitting on a
relatively small, high-signal feature set (one feature per base model, potentially fewer rows).

A **simple linear model** — logistic regression for classification, linear regression for
regression — is the common default. It effectively learns a weighted combination of the base
models' outputs, which is expressive enough to beat naive averaging (the weights aren't forced
to be equal or non-negative) while staying too simple to badly overfit the meta-feature space.

---

## 6. Multi-Level Stacking

Nothing stops you from stacking stacks: the meta-model's output (or a second layer of diverse
meta-models) can itself feed into a further meta-model, giving level-0 → level-1 → level-2, and so
on. In practice this is rare beyond two levels — each added layer multiplies training cost and
the leakage-avoidance bookkeeping, for diminishing accuracy gains, and it becomes much harder to
reason about why the ensemble makes a given prediction. It's worth knowing the concept exists
(you'll see it in Kaggle write-ups) more than reaching for it by default.

---

## 7. Minimal sklearn Code

```python
from sklearn.linear_model import LogisticRegression
from sklearn.svm import SVC
from sklearn.tree import DecisionTreeClassifier
from sklearn.neighbors import KNeighborsClassifier
from sklearn.ensemble import StackingClassifier

base_models = [
    ("knn", KNeighborsClassifier()),
    ("tree", DecisionTreeClassifier()),
    ("svc", SVC(probability=True)),
]

stack = StackingClassifier(
    estimators=base_models,          # level-0 models
    final_estimator=LogisticRegression(),  # level-1 / meta-model
    cv=5,                             # K-fold CV to generate OOF predictions
)

stack.fit(X_train, y_train)
preds = stack.predict(X_test)
```

`StackingClassifier`'s `cv` parameter is what implements the out-of-fold mechanism from Section 3
internally — it handles the fold loop, the OOF prediction generation, and the final refit of base
models on the full training set, so you don't have to hand-roll it. `cv=5` (or an explicit
`KFold`/`StratifiedKFold` object) is the usual choice; there's no built-in blending mode in
scikit-learn — blending is typically hand-rolled with a manual `train_test_split`.

---

## Key takeaways

- Stacking is the 4th ensemble family: instead of a fixed combination rule (vote/average), it
  trains a **meta-model** on the base models' predictions to *learn* how to combine them.
- The critical implementation detail is avoiding leakage: the meta-model must train on
  predictions the base models generated on data they **never saw during training** —
  out-of-fold predictions via K-fold CV for stacking, or holdout predictions for blending.
- Blending is simpler and less leakage-code to get wrong, but "spends" a chunk of data as a
  holdout set that base models never train on and the meta-model has less data to learn from;
  stacking's OOF approach uses the full training set more efficiently at the cost of more
  implementation complexity.
- A simple meta-learner (logistic/linear regression) is usually preferred — it's expressive
  enough to learn better-than-equal weights over base models without overfitting the small,
  high-signal meta-feature space.
- Stacking can outperform voting because it learns per-situation, potentially non-linear,
  weighted combinations rather than assuming every base model deserves equal say — but it's more
  complex and easier to get wrong (leakage, overfitting) than voting or bagging.
- Multi-level stacking (stacking meta-models on meta-models) exists but is rarely worth the added
  complexity beyond two levels.

**Next:** [23 — Clustering](23-clustering.md) — K-Means, hierarchical clustering, DBSCAN.
