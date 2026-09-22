# 24 — Imbalanced Data in Machine Learning

**Playlist video 133.** Real-world classification problems are rarely 50/50. Fraud, disease
diagnosis, churn, defect detection — in all of these the class you actually care about is rare.
This section covers what class imbalance breaks, the data-level fixes (undersampling,
oversampling, SMOTE), and the algorithm-level fixes (cost-sensitive learning, threshold tuning).

---

## 1. What Class Imbalance Is

**Class imbalance** means one class vastly outnumbers the other(s) in the training data. Classic
examples:

- **Fraud detection:** 99.9% legitimate transactions, 0.1% fraud.
- **Disease screening:** the vast majority of patients don't have the rare disease.
- **Churn prediction:** most customers don't churn in a given month.

Nothing about the *learning algorithm* breaks mechanically — it will happily fit. The problem is
what it learns to optimize.

### Why it's a problem: the accuracy paradox

Section 13 already flagged this (the **accuracy paradox**): if 99.9% of transactions are
legitimate, a model that predicts "legitimate" for *everything*, with zero intelligence, scores
99.9% accuracy. It has learned nothing about fraud — it has just learned that the majority class is
a safe bet, because minimizing overall error is exactly what the training objective rewards.

Most ML algorithms minimize a loss (or maximize accuracy) that treats every misclassified sample
as equally bad, and gradient/rule updates are driven by whichever class dominates the loss surface
by sheer volume. With imbalance, the "easy win" is to bias toward the majority class, so the model
does exactly that — decision boundaries drift toward always favoring the frequent class, and the
rare class (usually the one you actually care about — the fraud, the disease) gets ignored.

**The fix starts with measurement, not modeling.** If you're still looking at accuracy on an
imbalanced problem, you're solving the wrong problem. Use **precision, recall, F1, and PR-AUC**
(Section 13) — accuracy is close to useless here because it doesn't distinguish "caught the rare
class" from "ignored it." A model with 99.9% accuracy and 0% recall on fraud is worthless; a model
with 91% accuracy and 80% recall on fraud is doing its job. Precision-Recall curves (rather than
ROC-AUC) are the standard choice under heavy imbalance, because ROC-AUC can look deceptively good
even when precision on the rare class is poor.

Everything below — resampling, cost-sensitive learning, threshold tuning — is in service of getting
a model that actually learns the minority class's pattern. None of it matters if you then evaluate
with accuracy and declare victory.

---

## 2. Data-Level Solutions: Resampling

These techniques change the *training data's class balance* before (or during) fitting, without
touching the algorithm. **Critical rule: resample only the training set, and only after the
train/test split.** If you resample before splitting, synthetic or duplicated points can end up
split across train and test — the model is then evaluated on near-copies of what it trained on,
which is data leakage and gives an inflated, meaningless score. The test set must stay untouched
and reflect the real-world class distribution you'll actually see in production.

### Undersampling the majority class

Randomly drop rows from the majority class until the classes are closer to balanced.

- **Pro:** fast, and the resulting dataset is smaller — cheaper to train on.
- **Con:** you're throwing away real data. If the majority class has useful diversity, discarding
  most of it can lose signal and hurt the model's ability to distinguish borderline cases.

Undersampling is most defensible when the majority class is *very* large and there's plenty of it
to spare even after cutting — you're not losing meaningfully diverse examples, just redundant ones.

**Informed (non-random) undersampling** tries to be smarter about *which* majority points to drop,
instead of dropping uniformly at random:

- **Tomek links:** pairs of opposite-class points that are each other's nearest neighbor (i.e. sit
  right on the boundary). Removing the majority-class half of each pair cleans up the decision
  boundary rather than blindly thinning the interior.
- **NearMiss:** selects majority samples specifically *near* the minority class (the ambiguous,
  informative ones) rather than removing at random.
- **Edited Nearest Neighbours (ENN):** removes majority points whose neighborhood disagrees with
  their own label — i.e. noisy or borderline points.

These keep more of the *useful* majority signal than random undersampling, at the cost of extra
computation.

### Oversampling the minority class (random)

Randomly duplicate minority-class rows until the classes balance out.

- **Con:** the model sees the exact same minority points multiple times. It doesn't learn a
  broader concept of "what minority examples look like" — it just memorizes those specific points
  harder, which is a recipe for **overfitting** to the duplicated samples. On unseen minority
  examples that don't closely resemble the duplicated ones, performance doesn't actually improve
  much.

This is where SMOTE improves on the naive approach.

### SMOTE (Synthetic Minority Over-sampling Technique)

SMOTE oversamples the minority class, but instead of duplicating existing points, it **generates
new, synthetic minority points** by interpolating between real ones. Mechanism, for a chosen
minority point:

1. Find its **k nearest neighbors** within the minority class (in feature space).
2. Pick one of those neighbors at random.
3. Create a new synthetic point somewhere on the line segment between the original point and that
   neighbor: `x_new = x_i + λ · (x_neighbor − x_i)`, where `λ` is a random number between 0 and 1.
4. Repeat until the desired amount of oversampling is reached.

Why this beats naive duplication: the synthetic points are *new* locations in feature space that
plausibly belong to the minority class (because they sit between two confirmed minority points),
rather than exact repeats. This gives the model more *variety* to learn from — it sees the minority
class occupying a region, not a handful of exact coordinates — which reduces the overfitting risk
that comes from duplicate-heavy oversampling. The model generalizes better to minority examples it
hasn't seen exactly before.

SMOTE has known limits: it interpolates blindly between neighbors regardless of where they sit
relative to the decision boundary, which can generate synthetic points inside majority-class
territory if the minority class is noisy or its neighbors span across the boundary. Two variants
address this:

- **Borderline-SMOTE:** only generates synthetic samples from minority points that are near the
  decision boundary (the ones actually at risk of being misclassified), rather than from all
  minority points uniformly — focusing effort where it matters most.
- **ADASYN (Adaptive Synthetic Sampling):** generates *more* synthetic samples for minority points
  that are harder to learn (surrounded by more majority neighbors), adaptively weighting effort
  toward the hardest regions instead of spreading synthesis evenly.

### The imbalanced-learn (`imblearn`) library

All of the above — random/informed undersampling, random oversampling, SMOTE and its variants — are
implemented in **`imbalanced-learn`** (`pip install imbalanced-learn`, imported as `imblearn`). It's
built to slot into scikit-learn pipelines (its samplers implement `fit_resample`, and it ships a
`Pipeline` variant that applies resampling as a pipeline step only during `fit`, not during
`predict` — which is exactly what keeps you from accidentally resampling the test set).

```python
from sklearn.model_selection import train_test_split
from imblearn.over_sampling import SMOTE

# Split FIRST — resampling only ever touches the training half.
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

smote = SMOTE(random_state=42)
X_train_res, y_train_res = smote.fit_resample(X_train, y_train)  # X_test/y_test untouched

model.fit(X_train_res, y_train_res)
preds = model.predict(X_test)   # evaluate on the real, untouched distribution
```

---

## 3. Algorithm-Level Solutions

Instead of (or in addition to) touching the data, you can make the *algorithm* care more about the
minority class.

### Cost-sensitive learning / `class_weight='balanced'`

Most scikit-learn classifiers (`LogisticRegression`, `SVC`, `RandomForestClassifier`, etc.) accept a
`class_weight` parameter. Setting it to `'balanced'` automatically reweights the loss function so
that misclassifying a minority-class sample is penalized more heavily than misclassifying a
majority-class sample, in inverse proportion to how frequent each class is. The model is trained on
the *same* data (no synthetic points, no dropped rows) but the optimization is pushed to pay
attention to the rare class because getting it wrong now costs more.

```python
from sklearn.linear_model import LogisticRegression

model = LogisticRegression(class_weight="balanced")
model.fit(X_train, y_train)   # no resampling needed — the loss itself is reweighted
```

This is often the simplest thing to try first: no data manipulation, no leakage risk, one
parameter. It's a form of **cost-sensitive learning** — explicitly encoding that a false negative
on the minority class (missed fraud, missed disease) is more costly than a false positive, which
usually reflects the real business cost asymmetry anyway.

### Threshold tuning

Classifiers that output a probability (`predict_proba`) default to a 0.5 decision threshold for
"positive." Under imbalance, 0.5 is an arbitrary and usually bad choice — the model's predicted
probabilities skew low for the rare class simply because it saw fewer examples of it. Moving the
threshold down (e.g. flag anything above 0.2 probability as fraud, not just above 0.5) trades
precision for recall, and you pick the threshold based on the precision/recall tradeoff your
business actually needs — using the precision-recall curve from Section 13 to choose it explicitly,
rather than leaving 0.5 as an accident of the library default.

---

## 4. Choosing an Approach — Honest Guidance

There's no universal winner; what matters is being deliberate:

- **Start with the metric, not the resampling.** If you're still eyeballing accuracy, fix that
  before touching SMOTE or class weights — otherwise you can't even tell if a "fix" helped.
- **Try `class_weight='balanced'` first.** It's nearly free, has no leakage surface, and often gets
  you most of the way there for linear models and tree ensembles.
- **SMOTE tends to beat random oversampling** when you go the resampling route — it's cheap to try
  and rarely worse. But it isn't magic: on very high-dimensional or very noisy data, synthetic
  interpolation can create unrealistic points. Borderline-SMOTE/ADASYN help but add complexity —
  reach for them once plain SMOTE is a known baseline, not first.
- **Undersampling is a reasonable choice only when you have data to spare.** If the majority class
  is huge and diverse, cutting it down is cheap and fast. If your total dataset is small overall,
  undersampling can starve the model of majority-class signal too.
- **Resampling and cost-sensitive learning solve the same underlying problem two different ways —
  you rarely need both stacked at once.** Combining them can overcorrect and just flip which class
  gets treated unfairly. Tune one, evaluate, then decide if you need the other.
- **Threshold tuning is nearly always worth doing regardless of what else you did** — it's a
  post-hoc, free lever on an already-trained model.
- Whatever you choose, **the resampling never touches the test set**, and the metric you report
  must be precision/recall/F1/PR-AUC, not accuracy. Get those two things right and the specific
  resampling technique matters much less than people assume.

---

## Key takeaways

- Class imbalance breaks the *implicit assumption* behind accuracy: a model can score very high by
  simply always predicting the majority class, learning nothing about the class you actually care
  about (the accuracy paradox, Section 13).
- Data-level fixes: **undersampling** (drop majority rows — fast but discards information;
  Tomek links/NearMiss/ENN are smarter, informed variants), **random oversampling** (duplicate
  minority rows — risks overfitting to duplicates), and **SMOTE** (generate new synthetic minority
  points by interpolating between a point and its nearest minority neighbors — more variety, less
  overfitting than duplication; Borderline-SMOTE and ADASYN focus synthesis near the boundary or on
  harder points).
- Resample the **training set only, after the split** — resampling before splitting or resampling
  the test set is leakage and invalidates your evaluation.
- Algorithm-level fixes: `class_weight='balanced'` / cost-sensitive learning penalizes minority
  misclassification more heavily without touching the data; threshold tuning moves the decision
  cutoff off the default 0.5 to trade precision for recall deliberately.
- None of this matters without the right evaluation metric — precision, recall, F1, PR-AUC, not
  accuracy. Fix the metric first, then judge whether resampling or cost-sensitivity actually helped.
- `imbalanced-learn` (`imblearn`) implements all the resamplers here and integrates with
  scikit-learn pipelines.

**Next:** [25 — Hyperparameter Tuning](25-hyperparameter-tuning.md) — Optuna and Bayesian
optimization.
