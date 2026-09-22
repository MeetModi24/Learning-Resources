# 03 — Exploratory Data Analysis (EDA)

**Playlist videos 19–22.** Once data is gathered (Section 02), the next step in the MLDLC is to
actually *look at it* before touching a model. This section covers understanding your data,
analyzing one variable at a time (univariate), analyzing relationships between variables
(bivariate/multivariate), and automating the whole first pass with pandas profiling.

---

## 1. What is EDA, and why does it matter?

**Exploratory Data Analysis** is the process of summarizing, visualizing, and questioning a
dataset before modeling — with no hypothesis fixed in advance. The goal is to build intuition:
what does each column represent, what shape does it have, what's broken, what's related to what.

Why it matters:

- **It drives every downstream decision.** You can't choose a scaling strategy (Section 04) without
  knowing a feature's distribution, can't choose an imputation strategy (Section 06) without
  knowing how much and *how* data is missing, and can't pick a model without knowing whether
  relationships look linear or not.
- **It catches problems early.** Wrong dtypes, duplicate rows, impossible values (negative ages),
  leaked columns, severe class imbalance — all cheaper to catch now than after you've trained a
  model on garbage.
- **It's cheap relative to modeling.** A few `.describe()` calls and plots can save hours of
  debugging a model that's "underperforming" only because a column was never cleaned.

EDA is not a fixed checklist — it's a mindset of "understand data → ask a question → answer it
with a stat or plot → repeat." The three sub-skills below (understanding data, univariate,
bivariate/multivariate) are the recurring building blocks.

---

## 2. Understanding Your Data

Before any analysis, get an inventory of what you're working with.

### Shape and structure

```python
df.shape        # (rows, columns)
df.columns      # column names
df.head()       # first few rows — sanity-check values, not just types
```

### `df.info()` — dtypes, non-null counts, memory

```python
df.info()
```

This is usually the first command run on any new dataset. In one call it gives:

- **Dtype per column** — is a numeric-looking column actually stored as `object` (string)? That
  usually means stray characters (commas, currency symbols, mixed text) snuck in.
- **Non-null count per column** — an immediate, per-column view of missing values.
- **Memory usage** — relevant for large datasets; `object` columns are far more memory-hungry than
  numeric ones, and `category` dtype can shrink low-cardinality string columns substantially.

### `df.describe()` — descriptive statistics

```python
df.describe()                    # numeric columns by default
df.describe(include='object')    # categorical columns: count, unique, top, freq
```

For numeric columns this reports **count, mean, std, min, 25%/50%/75% percentiles, max** — the
five-number summary plus mean/std. Reading it is itself a mini-EDA step:

- `mean` vs `50%` (median) far apart → skewed distribution.
- `min`/`max` far outside the 25–75% range → possible outliers.
- `std` relative to `mean` → how spread out the values are.

### Missing values

```python
df.isnull().sum()                    # count per column
df.isnull().mean() * 100             # percentage per column
```

Knowing *how much* is missing per column, not just whether any is missing, decides whether you
drop the column, drop the rows, or impute (Section 06 covers imputation strategies in depth — this
step is just detection).

### Duplicates

```python
df.duplicated().sum()   # count of fully duplicated rows
df.drop_duplicates()
```

Duplicate rows silently inflate the importance of whatever pattern they repeat and can leak
between train/test splits if not caught early.

### Cardinality of categorical columns

```python
df['col'].nunique()
df['col'].value_counts()
```

**Cardinality** = number of distinct values in a column. This matters for two later decisions:

- **Encoding strategy** (Section 04) — one-hot encoding a categorical column with hundreds of
  unique values explodes the feature space; high-cardinality columns need a different approach.
- **Whether a column is even useful** — a column that's unique per row (like an ID) carries no
  learnable pattern; a column with only one distinct value carries no information either.

### Numerical vs categorical, and nominal vs ordinal

Every column falls into one of these buckets, and getting this classification right upfront
affects every plot and every encoding choice later:

- **Numerical** — values are numbers where arithmetic is meaningful (age, price, temperature).
  - *Continuous* — can take any value in a range (height, salary).
  - *Discrete* — countable, integer-valued (number of children, number of orders).
- **Categorical** — values are labels from a fixed set of categories.
  - *Nominal* — categories have **no inherent order** (city, color, gender).
  - *Ordinal* — categories have a **natural order** but the gaps between them aren't necessarily
    equal or numeric (education level: high school < bachelor's < master's; rating: low < medium
    < high).

A common trap: a column stored as an integer isn't automatically numerical in the meaningful
sense — a "star rating 1–5" or a "zip code" is numeric-looking but categorical/ordinal in nature.
Deciding this correctly upfront is what tells you whether to compute a mean (numerical) or a
mode/`value_counts()` (categorical) later.

---

## 3. Univariate Analysis

**Univariate** = analyzing **one variable at a time**, in isolation, with no reference to any
other column. The goal is to understand that single variable's distribution, center, spread, and
any anomalies.

### Numerical columns

**Histogram** — bins the range of values and counts how many observations fall in each bin. Shows
the overall shape of the distribution.

```python
df['col'].hist(bins=30)
```

**Distplot / KDE (Kernel Density Estimate)** — a smoothed, continuous version of the histogram's
shape, useful for comparing distribution shape without bin-width artifacts.

```python
import seaborn as sns
sns.kdeplot(df['col'])
sns.histplot(df['col'], kde=True)   # histogram + KDE overlay
```

**Boxplot** — a compact five-number-summary picture: box = 25th–75th percentile (the IQR),
line inside = median, whiskers extend to the typical range, and points beyond the whiskers are
flagged as potential outliers.

```python
sns.boxplot(x=df['col'])
```

**Measures of central tendency and spread** — the numeric complement to the plots:

- Central tendency: **mean** (sensitive to outliers), **median** (robust to outliers), **mode**
  (most frequent value; useful even for numerical data that clusters).
- Spread: **range**, **variance**, **standard deviation**, **IQR** (interquartile range,
  Q3 − Q1 — robust to outliers, unlike range/std).

**Skewness** — measures asymmetry of the distribution.

```python
df['col'].skew()
```

- `skew ≈ 0` → roughly symmetric.
- `skew > 0` (right/positive skew) → a long tail toward high values; mean > median.
- `skew < 0` (left/negative skew) → a long tail toward low values; mean < median.

What you're looking for across all of the above: is the distribution roughly symmetric or skewed,
is it unimodal or multimodal, are there visible outliers, and does the spread look reasonable for
what the column represents.

### Categorical columns

**`value_counts()`** — the categorical equivalent of a histogram: frequency of each category.

```python
df['col'].value_counts()
df['col'].value_counts(normalize=True)   # as proportions
```

**Countplot** — bar chart of category frequencies.

```python
sns.countplot(x=df['col'])
```

**Pie chart / bar chart** — visualize the proportion each category holds, especially useful for
spotting **class imbalance** (one category dominating the rest), which matters a lot later for
classification (Section 24 covers this in depth).

```python
df['col'].value_counts().plot(kind='pie', autopct='%1.1f%%')
df['col'].value_counts().plot(kind='bar')
```

What you're looking for: is any single category overwhelmingly dominant (imbalance), are there
rare categories that might need grouping, and does the number of categories match what you'd
expect (a "yes/no" column with five distinct values means dirty data).

---

## 4. Bivariate and Multivariate Analysis

**Bivariate** = relationship between **two** variables. **Multivariate** = relationship among
**three or more**. This is where you start asking "does X relate to Y," which is the whole point
of supervised learning down the line — the target variable is very often one side of these plots.

### Numerical – Numerical

**Scatterplot** — plots each observation as a point on two numeric axes; the go-to way to *see* a
relationship (linear, curved, none, clustered).

```python
sns.scatterplot(x='col1', y='col2', data=df)
```

**Correlation** — a single number summarizing the strength and direction of a *linear*
relationship, ranging from −1 (perfect negative) to +1 (perfect positive), with 0 meaning no
linear relationship.

```python
df[['col1', 'col2']].corr()
```

**Heatmap** — visualizes a full correlation matrix across many numeric columns at once, using
color intensity for correlation strength.

```python
sns.heatmap(df.corr(numeric_only=True), annot=True, cmap='coolwarm')
```

### Numerical – Categorical

**Boxplot / violin plot per category** — draw a separate numeric distribution (box or violin) for
each category, side by side, to compare the numeric variable's distribution *across* groups.

```python
sns.boxplot(x='category_col', y='numeric_col', data=df)
sns.violinplot(x='category_col', y='numeric_col', data=df)
```

A violin plot is a boxplot with the KDE shape drawn around it — it shows the same summary stats
plus the actual distribution shape per category.

**Barplot** — plots an aggregate (typically the mean, with a confidence interval) of the numeric
variable per category.

```python
sns.barplot(x='category_col', y='numeric_col', data=df)
```

### Categorical – Categorical

**Crosstab / contingency table** — a table of counts (or normalized proportions) for every
combination of two categorical variables' values.

```python
pd.crosstab(df['col1'], df['col2'])
pd.crosstab(df['col1'], df['col2'], normalize='index')
```

**Clustered/grouped bar chart** — visualizes the same contingency counts as bars grouped by one
category and colored by the other.

```python
pd.crosstab(df['col1'], df['col2']).plot(kind='bar')
```

### Multivariate

**Pairplot** — a grid of scatterplots for every pair of numeric columns (histograms on the
diagonal), giving a quick overview of all pairwise relationships at once.

```python
sns.pairplot(df)
```

**Correlation matrix / heatmap with more columns** — the same tool as above, just used as the
multivariate overview across the whole numeric feature set rather than one pair.

**`hue`** — the standard way to bring a third (often categorical) variable into a bivariate plot:
color points/bars by category to see whether a two-variable relationship differs by group.

```python
sns.scatterplot(x='col1', y='col2', hue='category_col', data=df)
sns.pairplot(df, hue='category_col')
```

### Correlation vs causation

A correlation between two variables — however strong, however visually obvious in a scatterplot
or heatmap — is **not evidence that one causes the other**. Both could be driven by a third,
unmeasured variable (a confounder), or the relationship could be coincidental. EDA can surface
*that* two variables move together; it says nothing about *why*. Treat every correlation you find
as a lead to investigate, not a conclusion to act on.

---

## 5. Pandas Profiling (Automated EDA Reports)

Manually running `.info()`, `.describe()`, histograms, and correlation heatmaps on every new
dataset is repetitive. The `ydata-profiling` library (formerly `pandas-profiling`) automates that
first pass into a single HTML report.

```python
from ydata_profiling import ProfileReport

profile = ProfileReport(df, title="Dataset Profiling Report")
profile.to_file("report.html")
```

What it auto-generates:

- **Overview** — number of rows/columns, dtypes, missing cells, duplicate rows, memory usage.
- **Per-variable statistics** — for each column, the same kind of univariate summary covered in
  Section 3 (distribution, mean/median/std, quantiles, distinct count) generated automatically,
  with a plot per variable.
- **Correlations** — a correlation matrix/heatmap across numeric variables, generated for you.
- **Missing-value maps** — visualizations (matrix/heatmap/bar) of *where* values are missing
  across the dataset, useful for spotting whether missingness is random or clustered in certain
  rows/columns.
- **Warnings** — automatic flags for things like high correlation between two columns, high
  cardinality, skewed distributions, columns that are constant, or a high proportion of missing
  values.

**When it's useful:** as a fast first pass on a new dataset, to get the same information Sections
2–4 describe without writing the code by hand, and to catch obvious warnings (a near-constant
column, a highly correlated pair) before you go looking for them manually.

**Limits:** it can be **slow on large or wide datasets** — computing every pairwise correlation
and per-variable plot doesn't scale gracefully as rows and columns grow. It's also generic: it
won't know that "zip code" is categorical even though it's stored as a number, and it can't
substitute for the judgment calls in Sections 2–4 (choosing what a plot actually implies, or
deciding what to do about a warning). Treat it as a fast starting point, not the whole analysis.

---

## Key takeaways

- EDA means understanding data *before* modeling — it directly determines your feature
  engineering, missing-value strategy, and model choice; skipping it means debugging a model
  later for problems that were visible in a `.describe()` call.
- Start with `.info()` / `.describe()` / `isnull().sum()` / `duplicated().sum()` and get the
  numerical-vs-categorical (and nominal-vs-ordinal) classification of every column right.
- Univariate analysis looks at one variable at a time: histogram/KDE/boxplot + skew/central
  tendency for numerical, `value_counts()`/countplot/pie for categorical.
- Bivariate/multivariate analysis looks at relationships: scatterplot/correlation/heatmap
  (numerical–numerical), boxplot/barplot per group (numerical–categorical), crosstab/clustered bar
  (categorical–categorical), pairplot/hue (multivariate) — and correlation is never causation.
- `ydata-profiling` automates the first pass into one report, but it's slow on big/wide data and
  doesn't replace judgment about what the findings mean.

**Next:** [04 — Feature Engineering, Scaling, Encoding](04-feature-engineering-scaling-encoding.md)
— turning understood data into model-ready features.
