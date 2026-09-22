# 06 — Missing Data

**Playlist videos 35–40.** Real datasets have holes. Before reaching for any imputation
technique, the videos insist on one thing first: figure out *why* the data is missing. The
mechanism determines which techniques are even valid — get this wrong and you can quietly bias
the model while believing you "cleaned" the data.

---

## 1. Missingness mechanisms: MCAR, MAR, MNAR

This taxonomy (Rubin's classification) describes the *relationship* between the fact that a value
is missing and the data itself.

- **MCAR — Missing Completely At Random.** The probability of a value being missing has nothing to
  do with any observed or unobserved data. It's pure chance — a sensor glitch, a lab sample lost in
  transit, a survey page that got torn. There is no pattern to exploit or bias to correct for.
  Example: a lab dropped a random subset of blood-test tubes due to a shipping mishap.
- **MAR — Missing At Random.** The probability of missingness depends on *other observed*
  columns, but not on the missing value itself. Example: men are less likely to report their weight
  than women — missingness in "weight" depends on the observed "gender" column, not on the actual
  (unseen) weight value. Once you condition on gender, the remaining missingness looks random.
- **MNAR — Missing Not At Random.** The probability of missingness depends on the *value that is
  missing itself* (or on something unobserved). Example: people with very high income tend to skip
  the "income" field on a survey. The missingness is directly tied to the value you didn't get to
  see. This is the hardest case — no purely statistical trick can fully fix it, because the
  mechanism is entangled with the unobserved answer.

**Why this matters for technique choice:** most classical imputation methods (mean/median/mode,
random sampling, KNN, MICE) implicitly assume MCAR or MAR — they reconstruct values consistent with
the *observed* pattern of data. If the true mechanism is MNAR, any of these methods can introduce
systematic bias, because the reason a value is missing is correlated with the value itself, and no
amount of clever averaging recovers that. In practice you rarely *prove* MCAR/MAR/MNAR with
certainty — you reason about it from domain knowledge (why would this field be missing?) and treat
it as a working assumption, not a fact you've verified statistically.

---

## 2. Univariate vs multivariate imputation

- **Univariate imputation** fills missing values in a column using *only that column's own
  observed values* — mean, median, mode, or a random sample from the column itself. Fast, simple,
  ignores relationships between columns.
- **Multivariate imputation** fills missing values in a column using *information from other
  columns* — e.g., "predict this person's missing age from their other features." KNN Imputer and
  MICE (Sections 5–6 below) are multivariate; they're more accurate but more expensive and require
  the other columns to actually carry signal about the missing one.

Complete Case Analysis and Simple/most-frequent Imputer are univariate (or a deletion strategy);
Missing Indicator and Random Sample Imputation are univariate too. KNN Imputer and MICE are
multivariate.

---

## 3. Complete Case Analysis (CCA) / listwise deletion

The simplest approach: **drop every row that has a missing value in any column you care about.**
No imputation happens at all — you just throw away the incomplete rows and train on what's left.

**When it's valid:**
- The missingness mechanism is (plausibly) **MCAR** — if missingness is systematically tied to
  some subgroup or to the value itself, dropping those rows biases the remaining sample toward
  whoever *doesn't* have missing data, which may not represent the whole population.
- The fraction of rows affected is **small** — a common rule of thumb is **under ~5%** missing.
  Beyond that you start throwing away too much signal.

**Pros:**
- Trivial to implement (`df.dropna()`).
- No distortion introduced by invented values — every remaining value is real.

**Cons:**
- **Loses data** — every dropped row also throws away whatever *other* good columns that row had.
- **Can bias the model** if missingness isn't MCAR (e.g., dropping rows disproportionately removes
  a subgroup).
- Not usable at all when many columns each have a few missing values scattered across many rows —
  the intersection of "no missing anywhere" can shrink the dataset drastically.

```python
import pandas as pd

df = pd.read_csv("data.csv")
missing_fraction = df.isnull().mean()          # fraction missing per column
cca_candidates = missing_fraction[missing_fraction < 0.05].index

df_cca = df.dropna(subset=cca_candidates)        # drop rows missing in low-missingness cols
```

---

## 4. Simple Imputer for numerical data: mean vs median

For a numerical column, the univariate fix is to replace every missing value with a single summary
statistic computed from the column's observed values.

- **Mean imputation** — replace missing values with the column's arithmetic mean. Preserves the
  mean of the column exactly, but only makes sense when the distribution is roughly symmetric and
  free of extreme outliers.
- **Median imputation** — replace missing values with the column's median. Preferred when the
  distribution is **skewed** or has **outliers**, because the median is robust to both — a few
  extreme values don't drag it around the way they drag the mean.

**Both distort the data** in the same fundamental way: piling many identical values (the mean or
median) at one point **artificially shrinks variance** and can distort the shape of the
distribution (a spike appears at the imputed value in a histogram). This is a real cost, not a free
lunch — it's a trade-off for simplicity, not a "correct reconstruction" of the missing values.

**Critical leakage rule: fit on TRAIN only.** The mean/median must be computed from the *training*
set alone, then that same fixed value is applied to fill missing values in the test set (and in
production). Computing the statistic from the full dataset (train + test combined) leaks
information about the test set's distribution into training — a subtle but real form of data
leakage that inflates validation performance unrealistically.

```python
from sklearn.impute import SimpleImputer
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

imputer = SimpleImputer(strategy="median")   # or strategy="mean"
imputer.fit(X_train)                          # statistic computed from TRAIN only

X_train_imputed = imputer.transform(X_train)
X_test_imputed = imputer.transform(X_test)    # same fitted statistic applied to test
```

---

## 5. Categorical imputation: most-frequent (mode) and "Missing" category

For categorical columns, mean/median don't apply. Two strategies:

- **Most-frequent (mode) imputation** — replace missing values with the single most common
  category in the column. Reasonable when missingness is small and roughly random relative to the
  categories — it's the categorical analogue of mean imputation, and shares the same weakness: it
  artificially inflates the frequency of the mode category and can mask a real signal in "why was
  this missing."
- **"Missing" as its own category** — instead of guessing a value, add a new category (e.g.
  `"Missing"`) and assign it to every missing entry. This is appropriate when the *fact of being
  missing* might itself be informative (closer to the MNAR case, or simply when you don't trust
  mode imputation to be a fair guess) — it lets the model learn a separate effect for "this field
  wasn't filled in" rather than silently conflating it with the most common real category.

```python
from sklearn.impute import SimpleImputer

mode_imputer = SimpleImputer(strategy="most_frequent")
missing_cat_imputer = SimpleImputer(strategy="constant", fill_value="Missing")
```

Choose most-frequent when you believe the missing entries would plausibly have been the mode
anyway (and the fraction missing is small); choose the "Missing" category when the missingness
itself may carry signal, or when you'd rather not fabricate a guess.

---

## 6. Missing Indicator

Any of the imputation strategies above throws away one piece of information: *whether a value was
originally missing.* If missingness itself is correlated with the target (e.g., people who skip
the "income" question tend to have different loan-default rates), that signal is worth preserving
explicitly.

**Missing Indicator** adds a new binary column per imputed feature: `1` if the original value was
missing, `0` if it was observed. You then still impute the original column (with any strategy
above) *and* keep the new indicator column alongside it — giving the model both a plausible filled
value and a flag saying "this one was guessed."

```python
from sklearn.impute import SimpleImputer, MissingIndicator

imputer = SimpleImputer(strategy="median", add_indicator=True)
X_train_imputed = imputer.fit_transform(X_train)   # extra indicator column(s) appended automatically

# equivalently, standalone:
indicator = MissingIndicator()
missing_mask = indicator.fit_transform(X_train)
```

This is most useful precisely when the mechanism looks like MAR or MNAR — where missingness
correlates with something (observed or not) that matters for prediction — rather than pure MCAR,
where the indicator would just be noise.

---

## 7. Random Sample Imputation

Instead of filling every missing value with the *same* single number (mean/median/mode), **Random
Sample Imputation** fills each missing entry by drawing a random value from the column's own
*observed* values (with replacement).

**Why this is better for distribution shape:** mean/median imputation collapses variance by piling
identical values at one point. Random sample imputation instead reproduces the actual observed
distribution — the imputed values have the same spread and shape as the real data, because they
*are* real observed values, just reused. Variance and distribution shape are far better preserved
than with mean/median imputation.

**Trade-offs:** it introduces randomness (results change with the random seed / aren't
deterministic unless seeded), and — like all univariate methods — it still ignores any
relationship between this column and other columns, and it still assumes the missingness doesn't
depend on the value itself (MCAR/MAR).

```python
import numpy as np

def random_sample_impute(series, random_state=42):
    rng = np.random.RandomState(random_state)
    observed = series.dropna()
    missing_idx = series[series.isnull()].index
    series = series.copy()
    series.loc[missing_idx] = rng.choice(observed, size=len(missing_idx))
    return series
```

(scikit-learn has no first-party `RandomSampleImputer`; it's typically hand-rolled or pulled from
`feature-engine`, as in the video.)

---

## 8. KNN Imputer — multivariate imputation

**KNN Imputer** moves from univariate to **multivariate**: instead of using only the column's own
values, it uses the *other* features to find similar rows.

**How it works:** for a row with a missing value in some column, find the **k nearest rows**
(measured by distance over the *other, non-missing* features) and fill the missing value with the
(weighted) average of those neighbors' values in that column — mean for numerical, majority vote
for categorical use-cases.

**Pros:**
- Generally **more accurate** than univariate methods when features are correlated, because it
  exploits the relationships between columns rather than ignoring them.

**Cons:**
- **Slower** — computing pairwise distances over the whole dataset scales poorly (O(n²)-ish
  behavior), which matters at real dataset sizes.
- **Needs scaling first** — distance is only meaningful if features are on comparable scales, so
  numerical features should be standardized/normalized before applying KNN Imputer (the same
  requirement KNN as a *model* has, back in Section 15's territory).
- Sensitive to the choice of `k` and to irrelevant/noisy features, which distort distance.

```python
from sklearn.impute import KNNImputer
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ("scale", StandardScaler()),
    ("impute", KNNImputer(n_neighbors=5, weights="distance")),
])
X_train_imputed = pipe.fit_transform(X_train)
X_test_imputed = pipe.transform(X_test)
```

---

## 9. MICE — Multivariate Imputation by Chained Equations (Iterative Imputer)

**MICE** is the most powerful (and most expensive) technique here. The idea: treat imputation as a
**regression problem, one column at a time, iterated to convergence.**

**The algorithm, roughly:**
1. Start by filling all missing values with a simple placeholder (e.g., column mean).
2. Pick one column with missing values. Treat it as the target; train a regression model on the
   *other* columns (using only rows where this column's original value was observed) to predict it.
3. Use that model to re-predict (re-impute) the missing values in this column.
4. Move to the next column with missing values and repeat step 2–3, now using the *updated* values
   from previously processed columns as predictors.
5. Cycle through all columns repeatedly ("chained equations") — each full pass is one iteration —
   until the imputed values stabilize (converge) or a maximum iteration count is reached.

Because each column's imputation depends on the current guesses for every other column, and those
guesses get refined round after round, MICE captures multivariate structure far more richly than
KNN Imputer's single-pass nearest-neighbor average.

**Cost and assumptions:**
- **Most expensive** technique in this section — it fits a regression model per column per
  iteration, repeated multiple times.
- Assumes the missingness is MCAR/MAR (like the rest of this section) and that the chosen
  regression model per column is a reasonable fit for that column's relationship to the others; a
  poor per-column model choice propagates errors through the chain.
- Convergence isn't guaranteed to be fast or clean on every dataset — it needs to be checked, not
  assumed.

scikit-learn implements this as `IterativeImputer` (experimental API, needs an explicit enable
import):

```python
from sklearn.experimental import enable_iterative_imputer  # noqa: required to unlock IterativeImputer
from sklearn.impute import IterativeImputer
from sklearn.linear_model import BayesianRidge

mice_imputer = IterativeImputer(
    estimator=BayesianRidge(),   # per-column regression model used each round
    max_iter=10,
    random_state=42,
)
X_train_imputed = mice_imputer.fit_transform(X_train)
X_test_imputed = mice_imputer.transform(X_test)
```

---

## 10. Decision guide: which method, when

There's no universal winner — the honest answer depends on how much is missing, whether it's
numerical or categorical, and how much compute/complexity you're willing to spend.

- **< ~5% missing, plausibly MCAR, and you can afford to lose those rows** → Complete Case
  Analysis. Simplest, no invented values.
- **Numerical column, need something fast and simple, distribution roughly symmetric** → mean
  imputation.
- **Numerical column, skewed or has outliers** → median imputation.
- **Categorical column, missingness looks incidental** → most-frequent (mode) imputation.
- **Categorical column, missingness might itself be meaningful** → "Missing" as its own category.
- **Suspect missingness correlates with the target (MAR/MNAR-ish)** → add a Missing Indicator
  alongside whatever imputation you use, so the model can see "this was missing" as a feature.
- **Want to preserve the original variance/distribution shape more faithfully than mean/median
  allow** → Random Sample Imputation.
- **Features are meaningfully correlated with each other, dataset isn't huge, and you can afford
  the compute** → KNN Imputer (remember to scale first).
- **You want the most accurate reconstruction possible and can afford the runtime cost** → MICE /
  IterativeImputer — best fit for multivariate structure, but heaviest to run and to reason about.

In all cases: whichever statistic or model is used to impute, **fit it on the training data only**
and apply the fitted transform to validation/test/production data — this is the single most common
mistake and it silently inflates offline metrics.

---

## Key takeaways

- Missingness has a mechanism — MCAR, MAR, or MNAR — and that mechanism determines whether an
  imputation technique is even valid. Most classical techniques assume MCAR/MAR; MNAR resists
  clean statistical fixes.
- Univariate methods (CCA, mean/median, mode/"Missing" category, random sample) use only the
  column's own values; multivariate methods (KNN Imputer, MICE) exploit relationships across
  columns for better accuracy at higher cost.
- CCA is only safe for small (~<5%), plausibly-MCAR missingness — otherwise it loses data and can
  bias the sample.
- Mean/median/mode imputation are simple but shrink variance and distort distribution shape;
  median and "Missing"-category variants exist specifically to be more robust to skew/outliers and
  to informative missingness, respectively.
- Missing Indicator preserves the signal of "this was missing" that plain imputation discards.
  Random Sample Imputation preserves distribution shape/variance better than mean/median.
- KNN Imputer and MICE are multivariate, more accurate, but progressively more expensive — KNN
  needs scaled features and is slow at scale; MICE iterates a regression per column to convergence
  and is the most powerful but heaviest option here.
- Always fit imputation statistics/models on the training split only, then apply to test/production
  — computing them on the full dataset is leakage.

**Next:** [07 — Outliers](07-outliers.md) — detecting and handling extreme values with z-score,
IQR, and percentile/winsorization methods.
