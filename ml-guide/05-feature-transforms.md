# 05 — Feature Transforms

**Playlist videos 30–34.** Section 04 got features onto a common scale and turned categories into
numbers. This section changes the *shape of the distribution itself* — reshaping skewed numeric
features toward something more Gaussian, converting continuous values into bins, and untangling two
messy real-world column types: mixed variables and datetime variables.

---

## 1. Why Reshape a Distribution at All?

Scaling (Section 04) changes the *range* of a feature; it does not change its *shape*. A feature
can be perfectly scaled to `[0, 1]` and still be badly right-skewed — most values bunched near zero
with a long tail of large values.

Why does shape matter?

- **Linear Regression** assumes a linear relationship between features and target, and its error
  analysis (confidence intervals, hypothesis tests) leans on the assumption that residuals are
  normally distributed. Skewed inputs tend to produce skewed, heteroscedastic residuals.
- **Logistic Regression** likewise tends to fit better and converge more reliably when features are
  roughly symmetric — a few extreme values in a long tail can dominate the decision boundary.
- Tree-based models (decision trees, random forest, gradient boosting) are largely **immune** to
  this — splits are based on ordering, not magnitude, so skew barely matters. Transformations in
  this section mainly pay off for linear/distance-based models.

### Checking for normality

Before transforming, actually look:

- **Histogram** — the quickest visual check for skew (a long tail on one side).
- **KDE plot** (`sns.kdeplot`) — a smoothed version of the histogram, easier to judge symmetry from.
- **Q-Q plot** (`scipy.stats.probplot`) — plots the data's quantiles against the quantiles of a
  theoretical normal distribution. If the points fall on the diagonal line, the data is close to
  normal; systematic curvature away from the line shows skew or heavy tails.
- **Skewness value** (`scipy.stats.skew` or `df[col].skew()`) — a single number: `0` is symmetric,
  positive means a right (long right tail), negative means a left tail. As a rule of thumb,
  magnitudes beyond roughly `±0.5` to `±1` are worth investigating.

```python
import scipy.stats as stats
stats.probplot(df["col"], dist="norm", plot=plt)
df["col"].skew()
```

The workflow in this section is always the same loop: **plot → transform → plot again** and check
whether skew decreased and the Q-Q plot straightened.

---

## 2. Function Transformer

`FunctionTransformer` applies an arbitrary user-chosen function to a column — the simplest and
least "smart" of the transforms here. There's no fitting involved; you pick the function up front
based on what the histogram/skew tells you.

```python
from sklearn.preprocessing import FunctionTransformer
import numpy as np

log_transformer = FunctionTransformer(func=np.log1p)
X_transformed = log_transformer.fit_transform(X)
```

### Log transform

Best known and most used. Compresses large values much more than small ones, which **pulls in a
long right tail** — turning right-skewed data (the most common real-world skew: income, price,
counts) closer to symmetric.

- Cannot handle **zero or negative** values (`log(0)` is undefined, `log` of a negative is complex).
- The standard fix is `log1p` (`log(1 + x)`), which handles zeros cleanly (`log1p(0) = 0`) — this is
  why `np.log1p` / `np.expm1` (its inverse) show up far more often in practice than raw `np.log`.
- Still doesn't handle negative values; those need a different family (see Power Transformer /
  Yeo-Johnson below).

### Reciprocal transform

`f(x) = 1/x`. Inverts the ordering of magnitude — large values become small and vice versa. Used
much more rarely than log; mainly relevant for very specific right-skewed ratios/rates where log
under- or over-corrects. Undefined at `x = 0` and flips sign for negative `x`.

### Square root transform

`f(x) = √x`. A gentler compression than log — pulls in a right tail, but less aggressively. Useful
when log over-corrects (turns the skew negative) or when the data includes zeros (unlike plain log,
`√0 = 0` is fine) but not negative values.

None of these three are "fit" to the data the way the next section's transform is — you choose
log/reciprocal/sqrt based on inspecting the distribution, apply it, then re-check the histogram/Q-Q
plot/skew to see if it actually helped.

---

## 3. Power Transformer

Function Transformer requires *you* to guess the right function. Power Transformer instead
**searches for the transformation that best normalizes the data**, within a parametric family
indexed by a single parameter, conventionally called **lambda (λ)**.

Conceptually: for a range of candidate λ values, the transform computes how close the resulting
distribution is to Gaussian (via maximum likelihood), and picks the λ that makes it *most* normal.
You don't need to derive or memorize the formula — the practical takeaway is "λ is a knob the
algorithm tunes automatically; different λ values reduce to familiar cases" (e.g. λ near 0 behaves
like a log transform, λ near 0.5 behaves like a square-root transform). This is why Power
Transformer often outperforms hand-picking a Function Transformer: it's the log/sqrt/identity/etc.
family, auto-tuned per column.

scikit-learn's `PowerTransformer` supports two methods:

### Box-Cox

The classical version. **Requires strictly positive data** (`x > 0`) — it is undefined for zero or
negative values, same limitation as plain `log`.

```python
from sklearn.preprocessing import PowerTransformer
pt = PowerTransformer(method="box-cox")
X_bc = pt.fit_transform(X)   # X must be > 0
```

### Yeo-Johnson

A generalization of Box-Cox that **handles zero and negative values** as well as positive ones, by
using a piecewise definition split at zero. This is scikit-learn's default `method` and the
practical choice whenever a column isn't guaranteed strictly positive.

```python
pt = PowerTransformer(method="yeo-johnson")  # default; handles 0 and negatives
X_yj = pt.fit_transform(X)
```

Like scaling, Power Transformer **must be fit on training data only** and applied (`transform`, not
`fit_transform`) to test data, to avoid leakage — same discipline as every other `fit`/`transform`
preprocessor in Section 04.

---

## 4. Binning and Binarization (Discretization)

**Discretization** converts a continuous numeric feature into a small number of discrete
**bins/categories**. It throws away fine-grained precision on purpose, in exchange for:

- **Robustness to outliers** — an extreme value just falls into the top (or a dedicated) bin instead
  of dominating a distance calculation or a coefficient.
- **Capturing non-linearity for linear models** — a linear model can only fit a straight-line effect
  of a raw numeric feature; once binned, each bin gets its own effect (via one-hot/dummy encoding of
  the bin label), letting the model express a non-monotonic relationship (e.g. "risk is high for
  very young *and* very old, low in between") that a single linear coefficient cannot.
- **Simplicity / interpretability** — "age 30–40" is easier to reason about and report than a raw
  continuous coefficient.

The cost: loss of precision, and the bin boundaries themselves become a modeling choice that can
introduce artifacts if chosen badly.

### Types of binning

- **Uniform / equal-width binning** — split the observed range into `k` bins of equal width
  (`max - min` divided evenly). Simple, but if the data is skewed, most points crowd into one or two
  bins while others stay nearly empty.
- **Quantile / equal-frequency binning** — split so each bin holds roughly the same *number* of
  points (based on percentiles). More robust to skew than equal-width, since bin boundaries adapt to
  the data's actual density.
- **K-Means binning** — run 1-D K-Means clustering on the feature values and use the resulting
  cluster assignments as bins. Boundaries follow natural density clusters in the data rather than a
  fixed rule, which can capture structure equal-width/equal-frequency binning would miss (e.g.
  distinct modes).

```python
from sklearn.preprocessing import KBinsDiscretizer

kbd_uniform  = KBinsDiscretizer(n_bins=5, encode="ordinal", strategy="uniform")
kbd_quantile = KBinsDiscretizer(n_bins=5, encode="ordinal", strategy="quantile")
kbd_kmeans   = KBinsDiscretizer(n_bins=5, encode="ordinal", strategy="kmeans")

X_binned = kbd_quantile.fit_transform(X)
```

`strategy` maps directly to the three types above (`"uniform"`, `"quantile"`, `"kmeans"`).

### Binarization

A special case of binning with exactly **two** outcomes: threshold a numeric feature into `0`/`1`
based on a cutoff (e.g. "number of past purchases > 0" becomes "has purchased before: yes/no").
Useful when only the presence/absence or a single cutoff matters, not the exact magnitude.

```python
from sklearn.preprocessing import Binarizer
binarizer = Binarizer(threshold=0.0)
X_bin = binarizer.fit_transform(X)
```

---

## 5. Handling Mixed Variables

A **mixed variable** is a single column that contains **both numeric and categorical information
tangled together** — not two separate well-typed columns, one messy one. Common real-world examples:

- An identifier like `"A123"`, `"B456"` — a categorical prefix plus a numeric part.
- A field where most rows are numbers but a few contain text codes (e.g. a `ticket_number` column
  that's usually numeric but occasionally holds a string like `"NUM12"` for a special case).

Models can't use a column like this directly — pandas will store it as `object` dtype and any
numeric meaning in the numeric-looking values is invisible to the model, while the categorical
signal in the text values is diluted by inconsistent formatting.

The fix is to **split the mixed column into separate, clean columns**: one holding the extracted
numeric part (as an actual numeric dtype) and one holding the extracted categorical/text part.

```python
# Example: column values like "A123", "B456", "C12"
df["cat_part"] = df["mixed_col"].str.extract(r"([A-Za-z]+)")   # letters -> categorical
df["num_part"] = df["mixed_col"].str.extract(r"(\d+)").astype(float)  # digits -> numeric
```

Once split, each new column goes through the normal pipeline for its actual type — numeric scaling
for `num_part`, categorical encoding (Section 04) for `cat_part`.

---

## 6. Handling Date and Time Variables

A raw datetime/timestamp column (e.g. `2024-03-15 14:32:00`) is **essentially useless to most ML
models as-is** — it's neither a clean number a linear model can use meaningfully nor a category a
model can split on. The value is in what the timestamp *implies*, so the standard approach is to
**decompose it into multiple derived numeric/categorical features**, then drop the original column.

Typical extracted features (using pandas' `.dt` accessor once a column is parsed as `datetime64`):

```python
df["date_col"] = pd.to_datetime(df["date_col"])

df["year"]         = df["date_col"].dt.year
df["month"]        = df["date_col"].dt.month
df["day"]           = df["date_col"].dt.day
df["day_of_week"]  = df["date_col"].dt.dayofweek       # 0=Monday ... 6=Sunday
df["is_weekend"]   = df["day_of_week"].isin([5, 6]).astype(int)
df["hour"]          = df["date_col"].dt.hour
df["is_month_end"] = df["date_col"].dt.is_month_end.astype(int)
```

Why each matters:

- **Year / month / day** — expose seasonality and trend (sales by month, growth by year) that a raw
  timestamp integer hides.
- **Day of week / is_weekend** — many business/behavioral signals (traffic, transactions) differ
  sharply between weekdays and weekends.
- **Hour** — for finer-grained data, time-of-day patterns (e.g. peak usage hours).
- **is_month_end** — captures calendar-boundary effects (e.g. billing cycles, end-of-month spikes).
- **Time elapsed / age** — often more useful than an absolute date: e.g. `(reference_date -
  date_col).dt.days` gives "days since X," and the same subtraction against "today" gives an *age* in
  years from a date of birth. Elapsed time is usually the actually-predictive quantity, not the
  calendar date itself.

```python
df["days_since"] = (pd.Timestamp.today() - df["date_col"]).dt.days
```

After extraction, the original datetime column is normally dropped — its information now lives in
the interpretable derived columns above.

---

## Key takeaways

- Skewed features hurt linear/logistic regression more than tree-based models; check shape with a
  histogram, KDE, Q-Q plot, and the numeric skewness value before and after transforming.
- **Function Transformer** applies a fixed function you choose: log/`log1p` for right-skewed
  positive data (zeros need `log1p`), reciprocal, or square root (gentler, tolerates zero).
- **Power Transformer** searches for the best power-family transform (parameter λ) to maximize
  normality: **Box-Cox** needs strictly positive data; **Yeo-Johnson** handles zero and negative
  values and is the safer default.
- **Binning/discretization** turns continuous values into categories — uniform (equal-width),
  quantile (equal-frequency), or KMeans-based (density-driven) — trading precision for outlier
  robustness and the ability of linear models to capture non-linear effects. **Binarization** is the
  two-bin special case (threshold → 0/1).
- **Mixed variables** (numeric and categorical values tangled in one column) must be split into
  separate clean numeric and categorical columns before encoding/scaling.
- **Datetime columns** carry no direct signal until decomposed into year/month/day/day-of-week/
  weekend flag/hour/month-end flag and elapsed-time/age features; the raw timestamp itself is
  dropped afterward.

**Next:** [06 — Missing Data](06-missing-data.md) — CCA, imputation strategies, and MICE.
