# 17 — Decision Trees

**Playlist videos 97–100.** Decision trees are the last standalone algorithm before the guide
turns to **ensembles** (Sections 18–22) — random forests, AdaBoost, and gradient boosting are all
"many trees combined," so understanding a single tree's mechanics and, critically, *why a single
tree is unstable*, is the whole motivation for everything that follows.

---

## 1. Intuition: a Tree of If/Else Questions

A decision tree predicts by asking a sequence of simple yes/no questions about the features, one
at a time, until it reaches a leaf that holds the answer.

```
Is age > 30?
├── No  → Is income > 50k?
│         ├── No  → Class: Reject
│         └── Yes → Class: Approve
└── Yes → Class: Approve
```

Each internal **node** tests one feature against a threshold; each **branch** is the outcome of
that test; each **leaf** is a final prediction (a class label for classification, a number for
regression). Building the tree means deciding, at every node, *which feature and threshold to
split on* — that's the entire learning problem.

This is a **white-box model**: you can read off the exact reasoning for any prediction by walking
the path from root to leaf. Contrast with SVM or a neural net, where the decision boundary exists
but isn't expressible as a short list of human-readable rules.

### Geometric view

Every split is a straight cut perpendicular to one feature's axis (an **axis-aligned** split) —
"age > 30" is a vertical line if age is the x-axis. Stacking splits recursively carves the feature
space into **axis-aligned rectangular regions**, and every point inside one region gets the same
prediction (the majority class, or the mean value, of the training points that landed there).
This geometric picture explains both the tree's strength (it can approximate *any* shape given
enough splits — it makes no assumption about linear separability, unlike logistic regression or a
linear SVM) and its weakness (real boundaries that are diagonal need many small rectangular steps
to approximate, which is inefficient and jagged).

---

## 2. How Splits Are Chosen (Classification): Impurity

At each node the tree must pick the one split, among all features and all thresholds, that makes
the resulting children as **pure** as possible — i.e., each child should be dominated by a single
class. "Impurity" is a number that measures how mixed a node's classes are; 0 means the node is
perfectly pure (single class), and it's higher the more evenly the classes are mixed.

### Entropy

Borrowed from information theory. For a node with classes having proportions `p₁, p₂, ..., pₖ`:

```
Entropy = −Σ pᵢ · log₂(pᵢ)
```

- **Range:** `0` (pure node, one class has `pᵢ = 1`) to `log₂(k)` for `k` classes — maximized when
  classes are perfectly balanced (e.g. for a binary node, max entropy is `1`, at a 50/50 split).
- **Interpretation:** entropy is the average number of bits of "surprise"/uncertainty in the
  node's class label. A pure node has zero uncertainty — you already know the answer, so there's
  nothing left to learn from a split there.

### Gini Impurity

A cheaper alternative used as sklearn's default:

```
Gini = 1 − Σ pᵢ²
```

- **Range:** `0` (pure) to `1 − 1/k` (balanced, e.g. `0.5` for binary at 50/50).
- **Why it exists:** no logarithm — just squares and a subtraction, so it's faster to compute over
  millions of candidate splits. In practice entropy and Gini pick nearly identical splits almost
  all the time; the difference in resulting trees is minor. Gini's default status in most libraries
  is a speed choice, not an accuracy one.

### Information Gain

The actual criterion used to pick a split: how much does impurity drop, weighted by how the data
divides between children?

```
Information Gain = Impurity(parent) − [ (n_left/n) · Impurity(left) + (n_right/n) · Impurity(right) ]
```

The tree greedily evaluates every feature and every candidate threshold, computes the information
gain (or equivalently, with Gini, the "Gini reduction") each would produce, and picks the split
that maximizes it. This repeats recursively at every child node — the tree is built **greedily**:
it always takes the best split *right now*, with no lookahead to whether a locally-worse split
might enable a globally-better tree later.

### Worked example

Node: 10 samples, 6 "Yes" / 4 "No".

```
p(Yes) = 0.6, p(No) = 0.4
Entropy(parent) = −(0.6·log₂0.6 + 0.4·log₂0.4) ≈ −(0.6·(−0.737) + 0.4·(−1.322)) ≈ 0.971
Gini(parent)    = 1 − (0.6² + 0.4²) = 1 − (0.36 + 0.16) = 0.48
```

Suppose a candidate split ("income > 50k") sends 5 samples left (all "Yes", pure) and 5 right
(1 "Yes" / 4 "No"):

```
Entropy(left)  = 0        (pure)
Entropy(right) = −(0.2·log₂0.2 + 0.8·log₂0.8) ≈ 0.722
Weighted child entropy = (5/10)·0 + (5/10)·0.722 = 0.361
Information Gain = 0.971 − 0.361 = 0.61
```

That gain (0.61) is compared against the gain from every other candidate split (other features,
other thresholds); whichever scores highest wins the node. A perfectly pure resulting child (like
the left one here) is exactly what drives gain up — the algorithm is explicitly hunting for splits
that isolate a single class as cleanly as possible.

---

## 3. Regression Trees

Same recursive splitting idea, different target and different criterion, since there's no "class
proportion" for a continuous target.

- **Split criterion:** minimize the **variance** (equivalently, MSE) of the target within the
  resulting children, weighted by child size — the regression analogue of information gain. A
  split is good if it separates high-target-value samples from low-target-value samples, shrinking
  the spread inside each child.
- **Leaf prediction:** the **mean** of the target values among the leaf's training samples (not a
  fitted line — just the average).

Because every region of the feature space maps to one constant (the region's mean), a regression
tree's prediction surface is a **step function**: flat within each rectangular region, with a
jump at every split boundary. This is fundamentally different from linear regression's smooth
plane — a tree can capture arbitrary non-linear shapes (in the limit of many splits) but can never
produce a smooth prediction; it always looks "staircased," and it cannot extrapolate outside the
range of training targets since a leaf's prediction is just a mean of values it already saw.

---

## 4. Overfitting and Underfitting: Hyperparameters

Left unconstrained, a tree keeps splitting until every leaf is **pure** (classification) or has a
single sample (regression) — it will happily draw a tiny rectangle around one outlier. That's a
tree that has **memorized the training set**: zero training error, but it hasn't learned a
generalizable pattern — it has learned the noise. This is **high variance**: small changes in the
training data produce very different trees, because a fully-grown tree chases every fluctuation in
the data.

At the other extreme, a tree stopped too early (e.g. `max_depth=1`) can't capture real structure —
**underfitting**, high error on both train and test.

**Pre-pruning** hyperparameters stop growth *before* it happens:

| Hyperparameter | What it limits | Effect of increasing it |
|---|---|---|
| `max_depth` | Longest root-to-leaf path | Smaller value → simpler tree, less overfitting; too small → underfits |
| `min_samples_split` | Minimum samples a node needs to be allowed to split at all | Higher → fewer splits happen, coarser tree |
| `min_samples_leaf` | Minimum samples a leaf must end up with | Higher → prevents tiny leaves fit to single outliers, smooths predictions |
| `max_features` | How many features are considered per split | Lower → more randomness/regularization (also the lever ensembles like random forest tune) |
| `max_leaf_nodes` | Total number of leaves allowed | Caps overall tree size directly |
| `min_impurity_decrease` | Minimum information-gain a split must produce to be kept | Higher → rejects marginal splits that only shave off noise |

Each of these trades bias for variance: tightening any of them makes the tree simpler (more bias,
less variance — risk of underfitting if pushed too far); loosening them lets the tree grow more
freely (less bias, more variance — risk of overfitting).

**Post-pruning**: grow the full, unconstrained tree first, then prune back. **Cost-complexity
pruning** (`ccp_alpha` in sklearn) is the standard approach — it repeatedly removes the subtree
whose removal costs the least increase in training impurity per leaf removed, controlled by a
complexity penalty `α`. `α = 0` keeps the full tree; increasing `α` prunes more aggressively.
Post-pruning can find a better bias/variance tradeoff than pre-pruning because it evaluates whole
subtrees *after* seeing how they turned out, rather than guessing stopping points in advance — but
it costs more compute since the full tree must be grown first.

---

## 5. Visualizing Trees

Because a tree is just nested if/else rules, it's directly inspectable — this is the practical
payoff of being a white-box model.

```python
from sklearn.tree import plot_tree
plot_tree(model, feature_names=X.columns, filled=True)
```

`plot_tree` (built into sklearn) draws the split conditions, impurity, and sample counts at every
node. **`dtreeviz`** is a nicer third-party alternative — it renders the actual feature
distributions at each split (not just a box of text), color-codes classes, and shows the decision
path for an individual prediction highlighted through the tree. The value isn't cosmetic: seeing
the tree exposes *which features actually matter* to the model, whether a split makes intuitive
sense given the data, and whether the tree has grown suspiciously deep/lopsided in a way that
signals overfitting — often faster than reading hyperparameter numbers alone.

---

## 6. Pros, Cons, and Why Ensembles Exist

**Pros:**
- Highly interpretable — the white-box property; you can explain any prediction as a rule path.
- No feature scaling needed — splits only compare a feature to a threshold, unaffected by units
  (unlike distance-based methods like KNN or SVM).
- Naturally handles non-linear relationships and mixed feature types (numeric and categorical)
  without preprocessing tricks.

**Cons:**
- **Unstable / high variance.** A small change in the training data — even removing a few rows —
  can flip which split looks best near the root, cascading into a completely different tree.
- **Greedy, not globally optimal.** Picking the locally-best split at each node doesn't guarantee
  the best possible overall tree; finding the true global optimum is computationally intractable,
  so no one tries.
- **Biased toward features with many distinct values/levels** — a high-cardinality feature has
  more candidate thresholds to try, giving it more chances to find a split that looks good on the
  training set (possibly spuriously).
- **Overfits easily** if left unconstrained, as covered above.

The instability is the key motivation for the next section: if a single tree changes a lot with
small data perturbations, then **averaging many trees trained on different subsets of the data**
(bagging) or **combining many weak trees sequentially** (boosting) should cancel out that
variance and produce a far more stable, accurate model. Decision trees are rarely the final model
in practice — they're the building block for random forests and gradient boosting, which dominate
tabular ML.

---

## Minimal sklearn usage

```python
from sklearn.tree import DecisionTreeClassifier, DecisionTreeRegressor

# Classification
clf = DecisionTreeClassifier(
    criterion="gini",       # or "entropy"
    max_depth=5,
    min_samples_split=10,
    min_samples_leaf=5,
)
clf.fit(X_train, y_train)
clf.predict(X_test)

# Regression
reg = DecisionTreeRegressor(
    max_depth=5,
    min_samples_leaf=5,
)
reg.fit(X_train, y_train)
reg.predict(X_test)
```

---

## Key takeaways

- A decision tree recursively asks if/else questions on features, carving the feature space into
  axis-aligned rectangular regions; each leaf predicts the majority class (classification) or mean
  target (regression) of the training points that land there. It's a white-box model.
- Classification splits are chosen by maximizing **information gain** — the drop in impurity
  (**entropy** `−Σpᵢlog₂pᵢ` or **Gini** `1−Σpᵢ²`) from parent to weighted children. Gini is faster
  to compute; the two rarely disagree in practice.
- Regression splits instead minimize variance/MSE within children; leaves predict the mean,
  producing a step-function prediction surface, not a smooth one.
- Unconstrained trees grow until leaves are pure — memorizing training data (high variance,
  overfitting). Pre-pruning (`max_depth`, `min_samples_split`, `min_samples_leaf`, `max_features`,
  `max_leaf_nodes`, `min_impurity_decrease`) constrains growth up front; post-pruning
  (cost-complexity pruning via `ccp_alpha`) grows fully then trims back.
- `plot_tree` / `dtreeviz` make the learned rules and splits directly inspectable — the practical
  payoff of interpretability.
- Trees are interpretable and scale-free but unstable, greedy, and prone to overfitting — exactly
  the weaknesses ensembles (many trees combined) are built to fix.

**Next:** [18 — Ensembles: Voting & Bagging](18-ensembles-voting-bagging.md) — combining multiple
models to cancel out a single tree's instability.
