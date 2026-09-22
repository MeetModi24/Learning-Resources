# 18 — Ensemble Learning: Voting & Bagging

**Playlist videos 101–107.** Ensembles combine multiple models to get a predictor that's better
than any single one in the group. This section covers the *why* of ensembling in general, then the
two simplest families: **voting** (different models, same data) and **bagging** (same model,
different data).

---

## 1. Introduction to Ensemble Learning

**Video 101.** The core idea is "wisdom of the crowd": a group of individually-imperfect models,
combined the right way, tends to outperform any single member of the group. This isn't magic — it
depends on two conditions holding on the base models:

1. **Each base model must be reasonably good.** Combining models that are worse than random
   guessing doesn't help — garbage in, garbage out still applies at the ensemble level.
2. **The base models must be diverse** — they need to make **uncorrelated errors**. If every model
   gets the same examples wrong, combining them changes nothing. If their mistakes are scattered
   across different examples, the errors tend to cancel out when combined, while the correct,
   agreed-upon predictions reinforce each other.

Diversity is *the* lever ensembles pull, and different families create diversity in different ways:

- **Voting** — diversity from using **different algorithms** on the same data (different model
  *types* make different kinds of mistakes).
- **Bagging** — diversity from training the **same algorithm** on different random samples of the
  data (different training sets → different fitted models, even though the algorithm is identical).
- **Boosting** (Section 20) — diversity from training models **sequentially**, each one focused on
  the previous one's mistakes.
- **Stacking** (Section 22) — diversity from different algorithms again, but instead of a fixed
  combination rule (vote/average), a **meta-model** learns how to best combine the base models'
  outputs.

This section covers the first two; boosting and stacking get their own sections later because the
mechanics differ enough to deserve full treatment.

---

## 2. Voting Ensemble — Core Idea

**Video 102.** A voting ensemble trains **several different algorithms** (e.g. Logistic Regression,
KNN, an SVM, a Decision Tree) on the **exact same training data**, then combines their predictions
at inference time. Nothing is trained sequentially and no model sees a different dataset — the only
source of diversity is that the algorithms themselves have different inductive biases, so they tend
to get different examples wrong.

The combination rule differs for classification and regression (Sections 3 and 4), but the
mechanical setup — same data, different algorithms, combine outputs — is common to both.

Why this works ties directly back to Section 1's diversity requirement: if a linear model, a
distance-based model, and a tree-based model all make *different* mistakes on the same dataset (which
is likely, since they carve up the feature space in fundamentally different ways), a majority-vote or
average smooths those mistakes out.

---

## 3. Voting Ensemble for Classification — Hard vs Soft Voting

**Video 103.** For classification there are two ways to combine the base classifiers' outputs.

### Hard voting
Each base model outputs a **predicted class label**. The ensemble's prediction is whichever label
gets the **majority of votes**. With 3 classifiers predicting `[1, 1, 0]`, the ensemble outputs `1`.
This only uses the final label — it throws away how *confident* each model was.

### Soft voting
Each base model outputs a **predicted probability** for each class (it must support
`predict_proba`). The ensemble **averages the probabilities** across models for each class, then
takes the `argmax` of the averaged probabilities as the final prediction.

**Why soft voting is usually better:** hard voting treats a model that's 51% confident the same as
one that's 99% confident — both just cast one vote for their top class. Soft voting lets a
confident, correct model outweigh several barely-confident, wrong ones, because it works with the
actual probability mass, not just the final decision. The catch is that this only helps if the base
models are reasonably **well-calibrated** (their probabilities actually reflect real likelihoods) —
if a model's probabilities are badly miscalibrated, averaging them can be misleading. In practice,
soft voting is the default choice whenever every base model can produce probabilities.

```python
from sklearn.ensemble import VotingClassifier
from sklearn.linear_model import LogisticRegression
from sklearn.neighbors import KNeighborsClassifier
from sklearn.tree import DecisionTreeClassifier

voting_clf = VotingClassifier(
    estimators=[
        ("lr", LogisticRegression()),
        ("knn", KNeighborsClassifier()),
        ("dt", DecisionTreeClassifier()),
    ],
    voting="soft",  # or "hard"
)
voting_clf.fit(X_train, y_train)
preds = voting_clf.predict(X_test)
```

---

## 4. Voting Ensemble for Regression

**Video 104.** For regression there's no notion of a "vote" over labels — each base model outputs a
continuous number, so the ensemble simply **averages the predictions** (optionally a weighted
average, giving stronger models more influence):

```python
from sklearn.ensemble import VotingRegressor
from sklearn.linear_model import LinearRegression
from sklearn.tree import DecisionTreeRegressor
from sklearn.svm import SVR

voting_reg = VotingRegressor(
    estimators=[
        ("lr", LinearRegression()),
        ("dt", DecisionTreeRegressor()),
        ("svr", SVR()),
    ]
)
voting_reg.fit(X_train, y_train)
preds = voting_reg.predict(X_test)
```

**Why averaging helps:** each model's prediction can be thought of as the true value plus some
error. If the errors from different models are uncorrelated (Section 1's diversity condition),
averaging them cancels out a lot of that error noise while the shared "true value" component stays
— this is the same variance-reduction logic that underlies bagging (Section 5), just applied across
different algorithms instead of different data samples.

---

## 5. Bagging — Introduction

**Video 105.** **Bagging** (**B**ootstrap **AGG**regat**ING**) takes the opposite approach to
voting: instead of varying the *algorithm* and keeping the *data* fixed, it keeps the algorithm
fixed and varies the *data*.

### Bootstrap sampling
A **bootstrap sample** is a random sample drawn from the training set **with replacement**, the
same size as the original dataset. "With replacement" means a row can be picked more than once —
so each bootstrap sample typically contains some duplicated rows and leaves out roughly a third of
the original rows entirely. Drawing many different bootstrap samples from the same dataset gives you
many datasets that overlap heavily but aren't identical.

### The bagging procedure
1. Draw `n` bootstrap samples from the training data.
2. Train the **same algorithm** independently on each sample, producing `n` fitted models.
3. **Aggregate** their predictions — majority vote for classification, average for regression
   (mirroring Sections 3–4, but now across resamples of one algorithm instead of across different
   algorithms).

### Why it works: variance reduction
Bagging's entire value proposition is reducing **variance**, not bias (bias/variance is formalized
in Section 11 and revisited for trees in Section 19). A model with low bias but high variance —
meaning it fits the training data well but changes a lot if you perturb the training set slightly —
benefits the most. Training many versions of that model on different bootstrap resamples and
averaging smooths out the sample-to-sample variance, the same way averaging noisy regression
predictions did in Section 4. This is why **deep, unpruned decision trees** (Section 17) are the
textbook bagging base learner: individually they overfit (low bias, high variance), and bagging is
precisely the antidote to high variance. Bagging a low-variance, high-bias model (like a linear
model) buys you much less, because there's little variance left to reduce.

### Row sampling vs feature sampling
Bagging is usually described in terms of resampling **rows**, but the same idea extends to
**features**:

- **Bagging (row sampling only)** — the classic case: bootstrap the rows, keep all features.
- **Random Subspaces** — sample a random *subset of features* for each base model, keep all rows.
- **Random Patches** — sample both rows *and* features randomly for each base model — the most
  general form, combining both sources of diversity.

Sampling features as well as rows adds another layer of decorrelation between the base models,
which is exactly the diversity condition from Section 1 — and this generalized idea is what Random
Forest (Section 19) builds on for decision trees specifically.

Bagging is also naturally **parallelizable**: since each base model trains independently on its own
bootstrap sample, all of them can be fit at the same time (`n_jobs=-1` in scikit-learn) — unlike
boosting, where each model depends on the previous one.

---

## 6. Bagging Classifier

**Video 106.** `BaggingClassifier` implements the classification case: bootstrap samples, one base
classifier per sample, combine by **majority vote**.

```python
from sklearn.ensemble import BaggingClassifier
from sklearn.tree import DecisionTreeClassifier

bag_clf = BaggingClassifier(
    estimator=DecisionTreeClassifier(),
    n_estimators=100,      # number of bootstrap samples / base models
    max_samples=0.8,       # fraction of rows per bootstrap sample
    max_features=0.8,      # fraction of features per base model (random patches)
    bootstrap=True,        # sample rows with replacement
    n_jobs=-1,
)
bag_clf.fit(X_train, y_train)
preds = bag_clf.predict(X_test)
```

Key knobs: `n_estimators` (more base models generally reduces variance further, at a compute cost),
`max_samples`/`max_features` (control how aggressive the resampling is — smaller fractions mean more
diversity between base models but each one sees less data), and `bootstrap`/`bootstrap_features`
(toggle row/feature sampling with or without replacement independently).

---

## 7. Bagging Regressor

**Video 107.** `BaggingRegressor` is the same procedure for regression: bootstrap samples, one base
regressor per sample, combine by **averaging** — exactly the aggregation used in the voting
regressor (Section 4), just over resamples of one algorithm rather than over different algorithms.

```python
from sklearn.ensemble import BaggingRegressor
from sklearn.tree import DecisionTreeRegressor

bag_reg = BaggingRegressor(
    estimator=DecisionTreeRegressor(),
    n_estimators=100,
    max_samples=0.8,
    bootstrap=True,
    n_jobs=-1,
)
bag_reg.fit(X_train, y_train)
preds = bag_reg.predict(X_test)
```

The same variance-reduction logic from Section 5 applies unchanged: bagging a high-variance
regressor (like a deep, unpruned regression tree) smooths out its instability across resamples; the
gain shrinks for a base learner that's already low-variance.

---

## Voting vs Bagging — the contrast

| | Voting | Bagging |
|---|---|---|
| Base models | **Different** algorithms | **Same** algorithm |
| Training data | **Same** for every model | **Different** bootstrap sample per model |
| Source of diversity | Different inductive biases | Different resampled data |
| Best combined with | Models that are individually strong and behave differently | A single high-variance base learner (e.g. deep trees) |
| Parallelizable | Yes | Yes |

---

## Key takeaways

- Ensembles beat single models only when the base models are individually decent *and* make
  diverse/uncorrelated errors — combining correlated mistakes changes nothing.
- **Voting**: different algorithms, same data. Classification uses **hard voting** (majority label)
  or **soft voting** (average probabilities, then argmax) — soft usually wins if models are
  reasonably well-calibrated, since it uses confidence, not just the top label. Regression averages
  the predictions directly.
- **Bagging**: same algorithm, many **bootstrap samples** (random sampling with replacement),
  aggregated by vote (classifier) or average (regressor). Its job is **reducing variance**, so it
  pays off most with high-variance, low-bias base learners like deep decision trees.
- Bagging generalizes beyond row sampling to **feature sampling** (random subspaces) and **both**
  (random patches) — more decorrelation between base models, at the cost of each seeing less
  information. Training is embarrassingly parallel.

**Next:** [19 — Random Forest](19-random-forest.md) — bagging specialized (and extended) for
decision trees, plus OOB scoring and feature importance.
