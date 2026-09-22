# 04 — Feature Engineering, Scaling & Encoding

**Playlist videos 23–29.** With EDA done (Section 03) you understand your data; this section is
about *reshaping* it into a form models can actually use — scaling numeric ranges, encoding
categories as numbers, and chaining all of that into reusable, leak-safe pipelines. Nothing here
changes the information content of the data; it changes its *representation*.

---

## 1. What is Feature Engineering?

**Feature engineering** is the process of using domain knowledge and transformation techniques to
create, modify, or select the input variables (features) a model sees, so that the model can learn
the underlying pattern more easily. It's the step referenced in Section 01's MLDLC as consuming a
huge share of real project effort — the ceiling on model performance is usually set by feature
quality, not algorithm choice.

It's commonly split into four parts:

- **Feature Transformation** — changing the *form* of existing features without creating new ones
  or discarding old ones: scaling (this section), encoding categorical variables (this section),
  and mathematical transforms to change a variable's distribution (Section 05).
- **Feature Construction** — building *new* features from existing ones (e.g. combining `height`
  and `weight` into `BMI`, or splitting a `datetime` into `day`/`month`/`hour`). Covered further in
  Section 08.
- **Feature Selection** — choosing the *subset* of existing features that are actually useful,
  dropping redundant or irrelevant ones.
- **Feature Extraction** — deriving a smaller set of new features that summarize the information in
  many original ones (e.g. PCA — Section 08).

This section covers the transformation half of that list: scaling and encoding.

---

## 2. Feature Scaling — Standardization

**Standardization** (a.k.a. Z-score normalization) rescales a feature so it has **mean 0** and
**standard deviation 1**:

```
z = (x - mean) / std
```

Every value is shifted by how far it is from the mean, measured in units of standard deviation.
The result has no fixed range (unlike Min-Max scaling below) — it can be any real number, though
most values land roughly in `[-3, 3]` for approximately normal data.

### Why it's needed

Many algorithms are sensitive to the **scale** of input features:

- **Distance-based models** — KNN, K-Means, SVM, PCA — compute distances (usually Euclidean)
  between points. A feature ranging in the thousands (e.g. `income`) will dominate the distance
  calculation over a feature ranging 0–1 (e.g. a normalized score), even if the second feature is
  more predictive. Scaling puts every feature on comparable footing.
- **Gradient-descent-based models** — linear regression, logistic regression, and neural networks
  trained with gradient descent — converge much faster and more reliably on standardized data.
  Wildly different feature scales distort the loss surface into a narrow, elongated bowl, which
  makes gradient descent zig-zag; standardized features make the bowl closer to circular.

### What standardization does *not* do

It does **not** change the **shape** of the distribution. A skewed feature stays skewed after
standardization — only its center and spread change. If the shape itself is the problem (e.g. you
need something closer to Gaussian for a model that assumes normality), that's a job for the
function/power transforms in Section 05, not scaling.

### When it's *not* needed

**Tree-based models** (decision trees, random forest, gradient boosting) split on threshold
comparisons per feature independently — the split point simply adapts to whatever scale the
feature is in. Scaling changes nothing about the splits, so it's unnecessary overhead for these
models.

### Fit on train only — the leakage trap

The scaler's `mean` and `std` are statistics **learned from data**, exactly like a model's
parameters. If you fit the scaler on the full dataset (train + test combined) before splitting, or
fit it on the test set at all, information about the test set's distribution leaks into training —
**data leakage**. The correct procedure:

```python
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)   # learn mean/std from train only
X_test_scaled = scaler.transform(X_test)          # apply those same stats to test
```

`fit` (or `fit_transform`) is called only on `X_train`. `X_test` only ever gets `transform` —
reusing the training set's mean and std. This mirrors how the model itself will see live/unseen
data in production: you never get to "peek" at the target distribution.

---

## 3. Feature Scaling — Normalization

**Normalization** here refers to a family of scalers that rescale a feature to a fixed, bounded
range, rather than centering it around a mean.

### MinMaxScaler

Rescales every value into `[0, 1]`:

```
x' = (x - min) / (max - min)
```

```python
from sklearn.preprocessing import MinMaxScaler
scaler = MinMaxScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled = scaler.transform(X_test)
```

Same fit-on-train-only rule applies — `min`/`max` are learned from `X_train` and reused on
`X_test`. **Sensitive to outliers**: a single extreme value stretches `min` or `max`, compressing
the rest of the data into a tiny sliver of the `[0, 1]` range.

### MaxAbsScaler

Divides every value by the feature's **maximum absolute value**:

```
x' = x / max(|x|)
```

Result range is `[-1, 1]`. Its distinguishing property is that it does **not shift/center** the
data — zero stays at zero. That makes it the right choice for **sparse data** (e.g. one-hot
encoded or TF-IDF matrices), where most entries are exactly 0 and you don't want a scaler to
disturb that sparsity by shifting every zero into a nonzero value.

```python
from sklearn.preprocessing import MaxAbsScaler
scaler = MaxAbsScaler()
X_train_scaled = scaler.fit_transform(X_train)
```

### RobustScaler

Uses the **median** and the **interquartile range (IQR = Q3 − Q1)** instead of mean/std or
min/max:

```
x' = (x - median) / IQR
```

Because the median and IQR aren't pulled around by extreme values the way the mean, std, min, and
max are, RobustScaler is **robust to outliers** — a good default when a feature has heavy outliers
you don't want to (or can't yet) remove.

```python
from sklearn.preprocessing import RobustScaler
scaler = RobustScaler()
X_train_scaled = scaler.fit_transform(X_train)
```

### Standardization vs. Normalization — which to use

| | Standardization (`StandardScaler`) | Normalization (Min-Max / MaxAbs / Robust) |
|---|---|---|
| Output range | Unbounded, centered at 0 | Fixed range (`[0,1]`, `[-1,1]`) |
| Outlier sensitivity | Somewhat sensitive (mean/std) | MinMax: very sensitive · Robust: not sensitive · MaxAbs: sensitive |
| Good default for | Distance/gradient-based models generally; when the feature is roughly Gaussian | When you need a bounded range (e.g. neural net inputs, image pixels); MinMax when there are no significant outliers; Robust when there are |
| Preserves sparsity | No | MaxAbs: yes |

In practice: reach for `StandardScaler` as the general default for linear models, SVM, KNN, PCA,
and neural nets. Switch to `RobustScaler` if outliers are a known problem you haven't otherwise
handled. Use `MinMaxScaler` when a bounded `[0, 1]` range is specifically required. Use
`MaxAbsScaler` for sparse matrices.

---

## 4. Encoding Categorical Data

Models need numbers, not strings — categorical columns must be converted. The right technique
depends entirely on whether the categories have an inherent **order**.

### Ordinal Encoding — for ordinal *input* features

Use when a categorical **feature** has a natural ranking, e.g. `education = ['High School',
"Bachelor's", "Master's", 'PhD']` or `review = ['Poor', 'Average', 'Good']`. `OrdinalEncoder` maps
each category to an integer that reflects that order:

```python
from sklearn.preprocessing import OrdinalEncoder

encoder = OrdinalEncoder(categories=[['Poor', 'Average', 'Good']])
X_train_enc = encoder.fit_transform(X_train[['review']])
X_test_enc = encoder.transform(X_test[['review']])
```

Passing `categories` explicitly matters — otherwise sklearn assigns integers by alphabetical
order, which won't match the real-world ranking.

### Label Encoding — for the *target*

`LabelEncoder` also maps categories to integers (`0, 1, 2, ...`), but it's meant for encoding the
**target/label column** `y`, not input features — e.g. turning class names `['cat', 'dog',
'rabbit']` into `[0, 1, 2]` for a classifier.

```python
from sklearn.preprocessing import LabelEncoder

le = LabelEncoder()
y_train_enc = le.fit_transform(y_train)
y_test_enc = le.transform(y_test)
```

### Why the distinction matters

Mechanically, `OrdinalEncoder` and `LabelEncoder` do the same arithmetic. The difference is
*intent and consequence*. If you use integer/label-style encoding on a **nominal** input feature
(categories with no real order — `color = ['Red', 'Green', 'Blue']`), you silently tell the model
`Blue (2) > Green (1) > Red (0)` and that the gap between Red and Green equals the gap between
Green and Blue. A model that uses magnitude or distance (linear regression, KNN, SVM) will pick up
on this fabricated ordering and fabricated spacing — a false signal that isn't in the real data.
That's exactly the failure mode One Hot Encoding exists to avoid.

---

## 5. One Hot Encoding

For **nominal** categorical features (no inherent order), One Hot Encoding creates one new binary
(0/1) column per category, instead of a single integer column. Only one of the new columns is `1`
("hot") for a given row; the rest are `0`. This removes any implied ordering or false distance
between categories entirely — every category is equidistant from every other.

```python
from sklearn.preprocessing import OneHotEncoder

ohe = OneHotEncoder(drop='first', sparse_output=False)
X_train_enc = ohe.fit_transform(X_train[['color']])
X_test_enc = ohe.transform(X_test[['color']])
```

### The dummy variable trap

If a feature has `k` categories, one-hot encoding naively produces `k` columns — but the last
column is entirely redundant: if a row is `0` in every other column, it *must* be the `k`-th
category. Keeping all `k` columns creates **perfect multicollinearity** (each column is a linear
combination of the others), which breaks the assumptions of models like linear/logistic regression
(unstable, non-unique coefficient estimates). This is the **dummy variable trap**, and the fix is
`drop='first'`, which drops one column per feature and keeps only `k - 1` columns — no information
is lost, since the dropped category is implied when all remaining columns are `0`.

### Many categories

A feature with high cardinality (many distinct categories — e.g. `zip code`, `merchant name`)
blows up into an enormous number of one-hot columns, most of which are sparse and rarely populated.
A common practical fix is to one-hot encode only the **top-N most frequent categories** and bucket
everything else into a single `"other"` category, keeping the column count manageable:

```python
top_categories = X_train['city'].value_counts().nlargest(10).index
X_train['city'] = X_train['city'].where(X_train['city'].isin(top_categories), 'other')
```

### Sparse output

Because most entries in a one-hot matrix are `0`, `OneHotEncoder` by default returns a **sparse
matrix** (only nonzero entries stored) rather than a dense NumPy array — far more memory-efficient
for high-cardinality features. `sparse_output=False` (as used above) forces a dense array, which is
convenient for small feature counts but wasteful at scale.

---

## 6. Column Transformer in scikit-learn

A real dataset almost never needs the *same* transformation applied to every column: numeric
columns need scaling, ordinal columns need `OrdinalEncoder`, nominal columns need `OneHotEncoder`,
and some columns may need nothing at all. Doing this manually means slicing the DataFrame into
pieces, transforming each piece separately, then concatenating the results back together — tedious
and easy to get wrong (mismatched row order, forgetting a column).

`ColumnTransformer` applies a list of `(name, transformer, columns)` triplets in one call and
assembles the results for you:

```python
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OrdinalEncoder, OneHotEncoder

transformer = ColumnTransformer(transformers=[
    ('num', StandardScaler(), ['age', 'income']),
    ('ord', OrdinalEncoder(categories=[['Poor', 'Average', 'Good']]), ['review']),
    ('ohe', OneHotEncoder(drop='first', sparse_output=False), ['color'])
], remainder='passthrough')  # columns not listed are kept as-is (or 'drop')

X_train_trans = transformer.fit_transform(X_train)
X_test_trans = transformer.transform(X_test)
```

The same fit-on-train / transform-on-test discipline from Section 2 applies to the whole
`ColumnTransformer` as a unit — `fit_transform` on train, `transform` only on test.

---

## 7. Machine Learning Pipelines

A **Pipeline** chains a sequence of transformers and a final estimator into a single object with
one `fit` and one `predict`/`transform` call — e.g. `ColumnTransformer → scaler → model` becomes
one object instead of several manual steps.

```python
from sklearn.pipeline import Pipeline
from sklearn.linear_model import LogisticRegression

pipe = Pipeline(steps=[
    ('preprocessing', transformer),      # the ColumnTransformer from above
    ('model', LogisticRegression())
])

pipe.fit(X_train, y_train)
preds = pipe.predict(X_test)
```

### Why pipelines matter (beyond convenience)

- **Prevents data leakage during cross-validation.** Without a pipeline, it's tempting to scale/
  encode the *entire* dataset once, then cross-validate — but that means every fold's scaler/
  encoder was fit partly on data that ends up in its own validation fold. Wrapping preprocessing
  and the model in one `Pipeline` and passing the *whole pipeline* to `cross_val_score` /
  `GridSearchCV` ensures every fit-on-train step is redone correctly *inside* each fold, using only
  that fold's training data.
- **Cleaner code.** One `fit`/`predict` call instead of manually threading data through N
  transformation steps in the right order, twice (train and test).
- **Reproducible deployment.** The trained pipeline — preprocessing steps and all — can be
  serialized (e.g. with `pickle` or `joblib`) as a single artifact. In production you load one
  object and call `pipe.predict(new_data)`; there's no risk of applying preprocessing steps in the
  wrong order, with the wrong fitted parameters, or forgetting a step entirely.

---

## Key takeaways

- Feature engineering has four parts: **transformation** (scaling/encoding — this section),
  **construction**, **selection**, and **extraction**.
- **Standardization** (`z = (x - mean)/std`) centers data to mean 0, std 1. Needed for
  distance-based (KNN, SVM, PCA) and gradient-based (linear/logistic regression, neural nets)
  models; unnecessary for tree-based models; does not change distribution shape.
- **Normalization** scales to a bounded range: `MinMaxScaler` (`[0,1]`, outlier-sensitive),
  `MaxAbsScaler` (`[-1,1]`, preserves sparsity), `RobustScaler` (median/IQR-based, outlier-robust).
- Always **fit scalers/encoders on the training set only**, then `transform` train and test —
  fitting on test data (or on the combined set before splitting) is data leakage.
- **Ordinal encoding** is for input features with a real order; **label encoding** is for the
  target. Applying integer-style encoding to a nominal input feature fabricates a false
  order/distance the model will pick up on.
- **One Hot Encoding** is for nominal input features; use `drop='first'` to avoid the dummy
  variable trap; bucket rare categories into `"other"` when cardinality is high.
- **`ColumnTransformer`** applies different transformers to different columns in a single call
  instead of manual slicing and concatenation.
- **`Pipeline`** chains preprocessing + model into one object — primarily to prevent leakage during
  cross-validation, but also for cleaner code and safer, reproducible deployment.

**Next:** [05 — Feature Transforms](05-feature-transforms.md) — function & power transforms,
binning, and mixed/datetime variables.
