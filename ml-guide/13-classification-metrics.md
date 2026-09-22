# 13 — Classification Metrics

**Playlist videos 76–78.** Logistic regression (Section 12) gives you a classifier, but "is it
good?" isn't answered by accuracy alone. This section builds the full evaluation toolkit for
classification: why accuracy lies to you on imbalanced data, the confusion matrix and its four
cells, precision/recall/F1 and the trade-off between them, and the ROC curve / AUC for evaluating
a classifier across all thresholds at once.

---

## 1. Accuracy — and why it's misleading

**Accuracy** is the simplest metric:

```
Accuracy = (correct predictions) / (total predictions) = (TP + TN) / (TP + TN + FP + FN)
```

It looks like the obvious metric, and for **balanced** classes it's a fine first check. The problem
is **imbalanced data** — one class vastly outnumbers the other. This is the **accuracy paradox**:

> Suppose 1% of a dataset is positive (e.g. fraud, rare disease). A model that predicts **"negative"
> for every single row**, doing zero learning, scores **99% accuracy**. It is simultaneously
> completely useless — it catches 0% of the actual positives, which is usually the entire point of
> building the model.

This is not a hypothetical edge case — it's the normal situation for fraud detection, disease
screening, churn, spam, and manufacturing-defect detection: the "interesting" class is rare by
definition. A high accuracy number on such data tells you almost nothing about whether the model
does the job it was built for. This is exactly why Section 24 (imbalanced data) and this section
exist: **accuracy alone is not a safe metric whenever classes are imbalanced or errors have unequal
cost** — which is most real-world classification.

---

## 2. The confusion matrix

The confusion matrix breaks predictions down by what was predicted **vs.** what was actually true.
For binary classification (positive class = the thing you care about, e.g. "has disease", "is
spam"), there are four outcomes:

| | **Predicted Positive** | **Predicted Negative** |
|---|---|---|
| **Actual Positive** | True Positive (TP) | False Negative (FN) |
| **Actual Negative** | False Positive (FP) | True Negative (TN) |

- **True Positive (TP):** predicted positive, actually positive. Correct catch.
- **True Negative (TN):** predicted negative, actually negative. Correct pass.
- **False Positive (FP):** predicted positive, actually negative. A false alarm — **Type 1 error**.
- **False Negative (FN):** predicted negative, actually positive. A miss — **Type 2 error**.

```
Accuracy  = (TP + TN) / (TP + TN + FP + FN)
```

### Type 1 vs Type 2 error — and which one is worse

- **Type 1 error = False Positive** — you raised an alarm that shouldn't have fired. Cost: wasted
  effort, unnecessary action, annoyance.
- **Type 2 error = False Negative** — you missed something real. Cost: the bad thing happens
  undetected.

Which is worse is entirely **domain-dependent** — there's no universal answer, which is exactly why
a single metric like accuracy can't capture it:

- **Medical diagnosis (e.g. cancer screening):** a False Negative (Type 2) — telling a sick patient
  they're healthy — is usually far worse than a False Positive (Type 1), which just triggers a
  follow-up test. You bias the model to minimize FN, i.e. optimize **recall**.
- **Spam filtering:** a False Positive (Type 1) — a real, important email (job offer, client
  message) marked as spam and never seen — is usually worse than a False Negative, which just means
  one spam email lands in the inbox as a minor annoyance. You bias toward minimizing FP, i.e.
  optimize **precision**.

The confusion matrix is what lets you *see* this trade-off instead of collapsing it into one number.

---

## 3. Precision, Recall, Specificity

### Precision

```
Precision = TP / (TP + FP)
```

"Of everything I predicted positive, how many were actually positive?" Precision is about the cost
of **False Positives** — high precision means when the model says "positive," you can trust it.
Low precision means lots of false alarms. Optimize precision when FP is the expensive error (spam
filtering above, or e.g. flagging a transaction as fraud and blocking a legitimate customer).

### Recall (Sensitivity, True Positive Rate / TPR)

```
Recall = TP / (TP + FN)
```

"Of everything that was actually positive, how many did I catch?" Recall is about the cost of
**False Negatives** — high recall means you're not missing real positives. Low recall means you're
letting a lot of positives slip through undetected. Optimize recall when FN is the expensive error
(disease screening, fraud detection where missing fraud is costlier than a false alarm).

### The precision–recall trade-off

Precision and recall pull in opposite directions, and a classifier that outputs a *probability*
(like logistic regression) exposes the knob that trades one for the other: the **decision
threshold** (default 0.5 — predict positive if `P(y=1) >= threshold`).

- **Raise the threshold** (e.g. to 0.8): the model only predicts positive when very confident →
  fewer FPs → **precision goes up**, but it now misses borderline true positives → **recall goes
  down**.
- **Lower the threshold** (e.g. to 0.2): the model predicts positive more liberally → catches more
  true positives → **recall goes up**, but also flags more actual negatives → **precision goes
  down**.

There is no threshold that maximizes both simultaneously (except in a perfect classifier); you pick
the threshold that matches the business cost of FP vs. FN from Section 2.

### Specificity (True Negative Rate)

```
Specificity = TN / (TN + FP)
```

"Of everything that was actually negative, how many did I correctly call negative?" It's the mirror
image of recall, but for the negative class. It also equals `1 - FPR` (False Positive Rate),
which matters directly for the ROC curve below.

---

## 4. F1 score

Precision and recall alone force you to look at two numbers and reason about a trade-off. The
**F1 score** collapses them into one number when you want a balanced summary:

```
F1 = 2 · Precision · Recall / (Precision + Recall)
```

F1 is the **harmonic mean** of precision and recall, not the arithmetic mean. This matters:

- Arithmetic mean of 1.0 and 0.0 is **0.5** — it hides the fact that one of the two is zero.
- Harmonic mean of 1.0 and 0.0 is **0** — it punishes imbalance between the two heavily.

The harmonic mean is always closer to the *smaller* of the two numbers. That's exactly the property
you want from a single summary metric: a classifier with precision = 0.95 and recall = 0.05 (great
at avoiding false alarms, terrible at actually catching positives) is not a good classifier, and F1
reflects that (F1 ≈ 0.095), while a naive average (0.5) would flatter it.

**When to use F1:** when you want one number and don't have a strong domain reason to favor
precision or recall individually — or when both FP and FN carry meaningful cost and you want a
metric that penalizes neglecting either one. If the domain clearly favors one error type (medical →
recall, spam → precision), optimize that metric directly instead of F1.

### Multiclass averaging: macro / micro / weighted

Precision, recall, and F1 are inherently binary (positive vs. negative) definitions. For multiclass
problems, scikit-learn computes a per-class score (treating each class as "positive" vs. all others
in turn) and then aggregates:

- **Macro average:** unweighted mean of the per-class scores. Every class counts equally regardless
  of how many samples it has — good for seeing if the model is neglecting a rare class.
- **Micro average:** aggregate all TP/FP/FN across classes first, *then* compute one global
  precision/recall/F1. Equivalent to accuracy in the single-label multiclass case.
- **Weighted average:** like macro, but each class's score is weighted by its support (number of
  true instances) — large classes dominate, so it's less sensitive to how a rare class performs.

`classification_report` below reports all three so you can pick the one matching your concern
(rare-class performance → macro; overall correctness → weighted/micro).

---

## 5. ROC curve and AUC

Precision/recall/F1 are all evaluated **at one fixed threshold**. The **ROC curve** (Receiver
Operating Characteristic) instead shows how the classifier performs **across every possible
threshold**, by plotting two rates against each other as the threshold sweeps from 1 down to 0:

```
TPR (Recall) = TP / (TP + FN)         — y-axis
FPR           = FP / (FP + TN) = 1 - Specificity   — x-axis
```

- At threshold = 1 (predict positive only when maximally confident): both TPR and FPR ≈ 0 → bottom
  left corner.
- At threshold = 0 (predict positive for everything): both TPR and FPR = 1 → top right corner.
- As the threshold decreases from 1 to 0, the curve traces a path from (0,0) to (1,1).

**Reading the curve:** a good classifier's curve hugs the **top-left corner** — high TPR achieved
while FPR is still low, meaning you catch most positives before you start accumulating false
alarms. A useless (random-guessing) classifier traces the **diagonal line** `y = x`, because at any
threshold TPR and FPR rise together in lockstep.

### AUC (Area Under the Curve)

**AUC** condenses the entire ROC curve into one number: the area underneath it.

- **AUC = 1.0** — perfect classifier (curve hugs the top-left corner, hits (0,1) exactly).
- **AUC = 0.5** — no better than random guessing (the diagonal).
- **AUC < 0.5** — worse than random (predictions are systematically backwards).

**Probabilistic interpretation** (the intuitive one worth memorizing): AUC is the probability that
the model ranks a randomly chosen actual positive **higher** than a randomly chosen actual negative.
This is threshold-independent — it measures the model's ability to *separate* the two classes by
its predicted scores, not its performance at any one cutoff. That's what makes AUC useful for
comparing models before you've committed to a threshold, and the curve itself is what you use
*after* to pick that threshold (e.g. the point closest to (0,1), or wherever your cost trade-off
from Section 2 says to cut).

### ROC-AUC vs. PR-AUC

ROC-AUC has a well-known weakness: **TN dominates FPR's denominator** (`FP + TN`), and on heavily
imbalanced data (the 1%-positive case from Section 1) TN is huge, so FPR stays deceptively low even
when FP count is, in absolute terms, comparable to the tiny number of actual positives. ROC-AUC can
look good on a model that's actually not very useful for the minority class.

The **Precision-Recall curve** (plotting Precision vs. Recall across thresholds, summarized as
**PR-AUC**) doesn't involve TN at all — both axes are built only from TP, FP, FN. On heavily
imbalanced data, **PR-AUC is the more honest metric**: it stays sensitive to how well the model
handles the rare positive class, which is usually the class you actually care about.

**Rule of thumb:** roughly balanced classes → ROC-AUC is fine and standard. Heavy imbalance (fraud,
rare disease, defect detection) → prefer PR-AUC, or at minimum look at both.

---

## 6. Which metric for which problem — practical guidance

| Situation | Prefer |
|---|---|
| Balanced classes, no strong cost asymmetry | Accuracy (as a sanity check), F1 |
| Imbalanced classes | **Not** accuracy — precision/recall, F1, or PR-AUC |
| FN is the costly error (disease, fraud missed) | **Recall** |
| FP is the costly error (spam→inbox loss, false accusation) | **Precision** |
| Need one balanced number, no domain skew | **F1** |
| Comparing models across all thresholds, roughly balanced | **ROC-AUC** |
| Comparing models across all thresholds, heavy imbalance | **PR-AUC** |
| Multiclass, care about rare classes equally | **Macro** F1/precision/recall |
| Multiclass, overall/weighted-by-frequency performance | **Weighted** or **micro** |

The common thread from Section 1 all the way through: **pick the metric that reflects which error
is expensive in your domain**, before you look at any single score.

---

## 7. Minimal sklearn code

```python
from sklearn.metrics import (
    confusion_matrix, classification_report,
    precision_score, recall_score, f1_score,
    roc_curve, roc_auc_score,
)

# y_test: true labels, y_pred: hard predictions (0/1), y_prob: predicted P(y=1)

# Confusion matrix -> [[TN, FP], [FN, TP]] with labels=[0, 1]
cm = confusion_matrix(y_test, y_pred, labels=[0, 1])

# Full precision/recall/F1 per class + macro/weighted averages
print(classification_report(y_test, y_pred))

precision_score(y_test, y_pred)   # single-number precision (positive class)
recall_score(y_test, y_pred)      # single-number recall
f1_score(y_test, y_pred)          # single-number F1

# ROC-AUC needs predicted probabilities, not hard labels
auc = roc_auc_score(y_test, y_prob)
fpr, tpr, thresholds = roc_curve(y_test, y_prob)   # for plotting the curve
```

---

## Key takeaways

- Accuracy = `(TP+TN)/total`, but it's meaningless on imbalanced data — the accuracy paradox: a
  model that always predicts the majority class scores high while catching zero minority cases.
- Confusion matrix: TP, TN, FP, FN. FP = **Type 1 error** (false alarm); FN = **Type 2 error**
  (miss). Which is worse is domain-specific (medical → minimize FN; spam → minimize FP).
- **Precision** = `TP/(TP+FP)` — trustworthiness of positive predictions, cost of FP.
  **Recall** = `TP/(TP+FN)` — coverage of actual positives, cost of FN. They trade off against each
  other as you move the classification threshold.
- **F1** = harmonic mean of precision and recall — punishes imbalance between the two, unlike a
  plain average. Use macro/micro/weighted averaging to extend to multiclass.
- **ROC curve** plots TPR vs. FPR across all thresholds; **AUC** is the probability a random
  positive is ranked above a random negative (0.5 = random, 1.0 = perfect). Prefer **PR-AUC** over
  ROC-AUC under heavy class imbalance.
- Choose the metric based on which error costs more in your domain — decide this *before* training,
  same as framing the problem in Section 01.

**Next:** [14 — Naive Bayes](14-naive-bayes.md) — probability foundations, Bayes' theorem, and the
Naive Bayes classifier family.
