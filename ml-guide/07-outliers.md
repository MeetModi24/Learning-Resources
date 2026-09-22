# 07 — Outliers

**Playlist videos 41–44.** An outlier is a data point that deviates so much from the rest of the
data that it looks like it doesn't belong. This section covers what outliers are and why they
matter, then three concrete detection/treatment methods: z-score, IQR, and percentile/winsorization.

---

## 1. What Are Outliers?

An **outlier** is an observation that lies an abnormal distance from the other values in a dataset
— e.g. most house prices cluster between ₹50L–₂Cr, but one row shows ₹500Cr.

### Where outliers come from

- **Genuine extreme values.** The data point is real and correctly recorded — a legitimately huge
  transaction, a celebrity's income in a salary dataset, a genuine fraud event. These are *signal*,
  not noise.
- **Data-entry / measurement errors.** A typo (age = 300), a sensor glitch, a unit mismatch (kg
  entered as g), a broken form field. These are genuinely garbage and usually should be fixed or
  removed.

The distinction matters because it determines what you do next — you can't decide "remove
outliers" as a blanket rule without asking *why* the point is extreme.

### Why outliers matter

Outliers matter because many algorithms are built on statistics (mean, variance, distances) that
are themselves **not robust** — a handful of extreme points can drag them far from where most of
the data actually sits.

**Sensitive to outliers:**
- **Linear regression / logistic regression** — fit by minimizing squared error (or a similar
  loss); a single far-away point can pull the fitted line/coefficients substantially.
- **KMeans** — clusters are defined by means (centroids); one extreme point can drag a centroid
  away from its natural cluster, or even form its own singleton "cluster."
- **PCA** — principal components are computed from variance/covariance, which outliers inflate,
  skewing the directions PCA identifies as "most important."
- **Distance-based methods generally** (KNN, hierarchical clustering, etc.) — any method that
  relies on Euclidean/other distances is thrown off because outliers sit far from everything,
  dominating distance calculations.

**Relatively robust to outliers:**
- **Tree-based models** (decision trees, random forest, gradient boosting) — splits are based on
  thresholds/ordering of values, not on means or distances, so a single extreme value mostly just
  ends up isolated in its own leaf/branch rather than distorting the whole model.

### Don't blindly remove outliers

The instinct to "just delete outliers" is often wrong. If the outlier is a genuine extreme value,
removing it can throw away the exact information you care about:

- **Fraud detection** — the fraudulent transactions *are* the outliers; removing them removes the
  problem you're trying to model.
- **Anomaly detection** (equipment failure, network intrusion) — same story: the outlier is the
  target, not noise to be cleaned.

So the first question before touching outliers is always: *is this an error, or is this the
signal?* Only data-entry-style errors and clearly non-representative noise should be routinely
removed; genuine extremes may need to be kept, modeled explicitly, or handled with a robust
algorithm instead of being deleted.

---

## 2. Z-Score Method

The z-score method assumes the feature is **roughly normally (Gaussian) distributed**. Under that
assumption, values naturally cluster around the mean, and how far a point sits from the mean (in
units of standard deviation) tells you how unusual it is.

**Z-score** for a value `x`:

```
z = (x - mean) / std
```

By the empirical rule for a normal distribution, ~99.7% of values fall within `mean ± 3·std`. So a
common rule: flag any point with `|z| > 3` as an outlier.

```python
mean = df['col'].mean()
std = df['col'].std()

upper_limit = mean + 3 * std
lower_limit = mean - 3 * std

outliers = df[(df['col'] > upper_limit) | (df['col'] < lower_limit)]

# trimming (remove)
df_trimmed = df[(df['col'] <= upper_limit) & (df['col'] >= lower_limit)]
```

**When it applies:** the column should be (approximately) normally distributed — check with a
histogram/KDE plot or a skewness value close to 0 before trusting this method.

**Weakness:** it's somewhat circular — the mean and std used to *define* the bounds are themselves
computed from the data that contains the outliers, so extreme values pull the mean and inflate the
std, which can mask the very outliers you're trying to detect. On skewed data the method is
unreliable because "mean ± 3·std" doesn't correspond to any meaningful tail cutoff.

---

## 3. IQR Method

The IQR method makes no normality assumption, which makes it the right choice for **skewed /
non-normal** distributions.

- **Q1** — the 25th percentile.
- **Q3** — the 75th percentile.
- **IQR** = `Q3 - Q1` (the spread of the middle 50% of the data).

**Bounds:**

```
lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR
```

Any point outside `[lower_bound, upper_bound]` is flagged as an outlier.

```python
Q1 = df['col'].quantile(0.25)
Q3 = df['col'].quantile(0.75)
IQR = Q3 - Q1

lower_bound = Q1 - 1.5 * IQR
upper_bound = Q3 + 1.5 * IQR

outliers = df[(df['col'] < lower_bound) | (df['col'] > upper_bound)]
df_trimmed = df[(df['col'] >= lower_bound) & (df['col'] <= upper_bound)]
```

This is exactly the rule a **boxplot** draws: the box spans Q1–Q3, the whiskers extend to
`Q1 - 1.5·IQR` / `Q3 + 1.5·IQR` (or the nearest data point within that range), and points beyond
the whiskers are plotted individually as outliers.

Because Q1, Q3, and IQR are based on ranks/percentiles rather than mean and variance, the IQR
method is **robust to distribution shape** — it works whether the data is skewed, has heavy tails,
or isn't Gaussian at all, which is why it's the more generally reached-for method versus z-score.

---

## 4. Percentile Method and Winsorization (Capping)

The percentile method sidesteps distributional assumptions entirely: define the outlier boundary
directly as a **fixed percentile** of the data, most commonly the **1st and 99th percentiles**
(anything below the 1st or above the 99th is an outlier). This is a direct, tunable way to trim a
chosen fraction of the extremes off each tail.

```python
lower_limit = df['col'].quantile(0.01)
upper_limit = df['col'].quantile(0.99)
```

From here there are two different treatments:

### Trimming (removal)

Drop rows outside the limits entirely:

```python
df_trimmed = df[(df['col'] >= lower_limit) & (df['col'] <= upper_limit)]
```

- **Trade-off:** simple, but you lose rows — and every other column's value in that row is
  discarded too, which can matter if rows are otherwise informative.

### Capping / Winsorization

Instead of deleting the row, **clip** the outlying value to the boundary value itself — values
below the 1st percentile are set *to* the 1st-percentile value, values above the 99th are set to
the 99th-percentile value:

```python
df['col'] = df['col'].clip(lower=lower_limit, upper=upper_limit)
```

- **Trade-off:** keeps every row (no data loss in other columns), but it distorts the true value
  of the capped points — they're no longer what was actually observed, just pinned to a boundary.
  Capping is generally preferred over trimming when you can't afford to lose rows (small datasets,
  or rows where other features are valuable).

---

## 5. Train/Test Discipline and Choosing a Treatment

**Compute bounds (mean/std, Q1/Q3/IQR, or percentiles) on the training set only, then apply those
same fixed bounds to the test set.** Recomputing bounds on test data — or on train+test combined —
leaks test-set distribution information into the "cleaning" step, the same leakage problem as
fitting a scaler or imputer on the full dataset (Sections 04 and 06). In a pipeline, the outlier
boundary is a *parameter learned from training data*, not a property recomputed at inference time.

**Removal vs. capping vs. transform — how to decide:**
- If the outlier is a **data-entry error**, fix it if you can reconstruct the true value, otherwise
  treat as missing (Section 06) or remove the row.
- If the outlier is a **genuine extreme value** and you don't want to lose the row, prefer
  **capping/winsorization** — it bounds the influence of the extreme value while keeping the row's
  other information intact.
- If the outlier is a genuine extreme value that is itself the thing you're trying to detect
  (fraud, anomalies), **don't remove or cap it at all** — keep it, and consider a model/algorithm
  that is robust to or specifically targets such points (e.g. tree-based models, or an anomaly
  detection approach) instead of preprocessing it away.
- If the underlying issue is that the feature is **skewed** rather than truly having bad points, a
  power/log transform (Section 05) can pull extreme values inward at the source instead of
  detecting and treating them after the fact.

---

## Key takeaways

- An outlier is a point far from the rest of the data — either a genuine extreme value (signal) or
  a data-entry/measurement error (noise). Decide which before treating it.
- Mean/variance/distance-based models (linear/logistic regression, KMeans, PCA, KNN) are sensitive
  to outliers; tree-based models are comparatively robust.
- Don't blindly remove outliers — in fraud/anomaly detection the outliers are the target.
- **Z-score** (`|z| > 3`, using mean ± 3·std): fast, but assumes roughly-normal data and its own
  mean/std are distorted by the outliers it's trying to catch.
- **IQR** (`Q1 - 1.5·IQR`, `Q3 + 1.5·IQR`): the boxplot rule, robust to skew and distribution shape
  — the safer default for non-normal data.
- **Percentile method**: define outliers as beyond a fixed percentile (e.g. 1st/99th); treat via
  **trimming** (delete the row) or **capping/winsorization** (clip to the boundary, keep the row).
- Always compute bounds on train data and apply the same bounds to test data — recomputing on test
  data is leakage.

**Next:** [08 — Dimensionality & PCA](08-dimensionality-pca.md) — feature construction, the curse
of dimensionality, and principal component analysis.
