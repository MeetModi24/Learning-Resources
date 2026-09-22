# 14 — Naive Bayes

**Playlist videos 82–90.** This section builds probability from the ground up — conditional
probability, independence, mutual exclusivity — then derives Bayes' theorem, and finally uses it
to construct a full classifier: Naive Bayes. The "naive" assumption is the whole story here, so
the goal is to understand exactly what it assumes, why that's usually false, and why the classifier
still works anyway.

---

## 1. Conditional Probability

**Conditional probability** is the probability of event A happening *given that* event B has
already happened, written `P(A|B)`:

```
P(A|B) = P(A ∩ B) / P(B)        (requires P(B) > 0)
```

`P(A ∩ B)` is the probability both A and B happen. Dividing by `P(B)` **shrinks the sample space**
down to just the world where B is true, then asks what fraction of that world also has A. This is
the core move behind everything that follows: conditioning on new information narrows the universe
of possibilities.

Example: draw one card from a deck. `P(King) = 4/52`. But `P(King | Face card) = 4/12` — knowing
the card is a face card (J, Q, K) shrinks the sample space from 52 cards to 12, and Kings are now a
bigger fraction of it.

Rearranging the definition gives the **multiplication rule**, used constantly below:

```
P(A ∩ B) = P(A|B) · P(B) = P(B|A) · P(A)
```

---

## 2. Independent Events

Two events A and B are **independent** if knowing one happened tells you *nothing* about the
other:

```
P(A|B) = P(A)   and   P(B|A) = P(B)
```

Plugging into the multiplication rule gives the defining equation used in practice:

```
P(A ∩ B) = P(A) · P(B)          (independent events only)
```

Example: two separate fair coin flips. The first flip's outcome has zero bearing on the second.
`P(both heads) = 0.5 × 0.5 = 0.25`.

Contrast with **dependent** events: drawing two cards *without replacement*. `P(2nd is King | 1st
was King) = 3/51 ≠ P(2nd is King) = 4/52` — the first draw changed the deck, so it changed the
probability of the second.

---

## 3. Mutually Exclusive Events

Two events are **mutually exclusive** if they **cannot both happen** in the same trial:

```
P(A ∩ B) = 0
```

Example: rolling a die, `A = "roll is 2"` and `B = "roll is 5"`. You can't get both on one roll.
For mutually exclusive events, the addition rule simplifies: `P(A ∪ B) = P(A) + P(B)` (no overlap
to subtract).

### Independent vs. mutually exclusive — don't conflate these

This is a genuinely common confusion, and the two ideas are almost opposites:

| | Independent | Mutually exclusive |
|---|---|---|
| Meaning | One event's occurrence doesn't affect the other's probability | The two events can never occur together |
| Formula | `P(A ∩ B) = P(A) · P(B)` | `P(A ∩ B) = 0` |
| Can both happen at once? | Yes, and at the "expected" joint rate | Never |

If two events are mutually exclusive *and* both have nonzero probability, they **cannot** be
independent — `P(A)·P(B) > 0` but `P(A ∩ B) = 0`, so the independence equation is violated. Rolling
a 2 and rolling a 5 are mutually exclusive, but they are also *dependent*: knowing the die rolled a
2 makes `P(roll is 5)` drop to zero, which is about as far from "no effect on the other" as
possible. Independence is about *information*; mutual exclusivity is about *co-occurrence*. Two
unrelated coin flips are independent but not mutually exclusive (both can be heads).

---

## 4. Bayes' Theorem

Start from the multiplication rule, which gives two expressions for the same joint probability:

```
P(A|B) · P(B) = P(B|A) · P(A)
```

Divide both sides by `P(B)`:

```
P(A|B) = P(B|A) · P(A) / P(B)
```

That's **Bayes' theorem**. It lets you flip a conditional probability around — go from `P(B|A)`
(which is often easy to measure) to `P(A|B)` (which is often what you actually want to know).

Each term has a name, and the vocabulary matters because it's used everywhere in the rest of this
section:

| Term | Name | Meaning |
|---|---|---|
| `P(A)` | **Prior** | What you believed about A before seeing evidence B |
| `P(B\|A)` | **Likelihood** | How probable the evidence B is, if A is true |
| `P(B)` | **Evidence** | Overall probability of observing B (normalizing constant) |
| `P(A\|B)` | **Posterior** | Updated belief about A after seeing evidence B |

In words: **posterior ∝ likelihood × prior**. Bayes' theorem is a formal rule for updating a belief
in light of new evidence.

### Classic worked example: disease testing

A disease affects 1% of a population (`P(Disease) = 0.01`). A test is 99% accurate for both sick
and healthy people (`P(Positive|Disease) = 0.99`, `P(Positive|No Disease) = 0.01`, i.e. a 1% false
positive rate). You test positive. What's `P(Disease|Positive)`?

First expand the evidence term `P(Positive)` using the **law of total probability** — sum over all
the ways to test positive:

```
P(Positive) = P(Positive|Disease)·P(Disease) + P(Positive|No Disease)·P(No Disease)
            = 0.99 × 0.01 + 0.01 × 0.99
            = 0.0099 + 0.0099 = 0.0198
```

Now Bayes' theorem:

```
P(Disease|Positive) = P(Positive|Disease) · P(Disease) / P(Positive)
                     = (0.99 × 0.01) / 0.0198
                     = 0.5   (50%)
```

Despite a "99% accurate" test, a positive result only means a **50% chance** of actually having the
disease — because the disease is rare (a low prior), the false positives from the huge healthy
population swamp the true positives from the tiny sick population. This is the single most
important intuition Bayes' theorem teaches: **the prior matters as much as the test's accuracy.**
Ignoring the prior (a mistake called the *base rate fallacy*) is exactly the error this theorem
corrects.

---

## 5. Naive Bayes: Intuition

**Naive Bayes** is a classifier: given a set of features, it uses Bayes' theorem to compute
`P(class | features)` for every possible class and predicts whichever class has the highest
posterior.

The problem is that with several features, `P(features | class)` is a *joint* probability over all
of them together, and estimating a joint distribution over many features requires an enormous
amount of data (the joint blows up combinatorially — this is the curse of dimensionality from
Section 08 again).

The **naive assumption** cuts through this: assume the features are **conditionally independent of
each other, given the class**. That means, once you know the class, learning the value of one
feature tells you nothing extra about another feature's value. This lets the joint likelihood
factor into a simple product of per-feature likelihoods (Section 6).

Why "naive"? Because this assumption is almost never exactly true in the real world. In spam
detection, the words "free" and "money" are *not* independent even given the class "spam" — they
tend to co-occur. The model pretends they are anyway.

**Why it still works well despite the false assumption:**

- What matters for classification isn't getting the *exact* posterior probability right — it's
  getting the *ranking* of classes right (which class wins). Correlated features get double-counted
  by roughly the same factor across classes, so the ranking often survives even when the
  probability magnitudes are off.
- With many weakly-informative features (exactly the text-classification setting: thousands of
  words, each mildly informative), the errors from ignoring correlations tend to wash out.
- It needs comparatively little training data because it only ever has to estimate simple
  per-feature, per-class distributions — never a joint distribution.

This is why Naive Bayes became the classic first algorithm for text classification (spam
filtering, sentiment, document/topic classification): high-dimensional, sparse, correlated features
— exactly where the "naive" shortcut pays for itself in speed and data-efficiency without paying
much accuracy cost.

---

## 6. The Mathematics Behind Naive Bayes

For a class `Cₖ` and features `x₁, x₂, ..., xₙ`, Bayes' theorem gives:

```
P(Cₖ | x₁,...,xₙ) = P(x₁,...,xₙ | Cₖ) · P(Cₖ) / P(x₁,...,xₙ)
```

Applying the **conditional independence assumption** to the likelihood turns the joint into a
product:

```
P(x₁,...,xₙ | Cₖ) = P(x₁|Cₖ) · P(x₂|Cₖ) · ... · P(xₙ|Cₖ) = ∏ᵢ P(xᵢ|Cₖ)
```

So:

```
P(Cₖ | x₁,...,xₙ) = P(Cₖ) · ∏ᵢ P(xᵢ|Cₖ)  /  P(x₁,...,xₙ)
```

### Dropping the denominator

The evidence term `P(x₁,...,xₙ)` doesn't depend on which class `Cₖ` you're evaluating — it's the
same number regardless of the class you plug in. Since we only care about *which class maximizes
the posterior*, not the posterior's exact value, we can drop it and work with a **proportionality**:

```
P(Cₖ | x₁,...,xₙ)  ∝  P(Cₖ) · ∏ᵢ P(xᵢ|Cₖ)
```

The predicted class is the one that maximizes the right-hand side — this decision rule is called
**MAP (Maximum A Posteriori)** estimation:

```
ŷ = argmax_Cₖ  [ P(Cₖ) · ∏ᵢ P(xᵢ|Cₖ) ]
```

In practice this product of many small probabilities underflows numerically, so implementations
sum **log-probabilities** instead (`log P(Cₖ) + Σᵢ log P(xᵢ|Cₖ)`), which is equivalent since log is
monotonic.

### The zero-frequency problem and Laplace smoothing

If a feature value never appeared with a given class in training data (say the word "cryptocurrency"
never appeared in any training "ham" email), then `P("cryptocurrency" | ham) = 0`. Because the
formula is a *product*, a single zero factor forces the **entire product to zero** — no matter how
strongly every other feature points to "ham," one unseen word wipes out the whole posterior. That's
clearly wrong: an unseen combination in a finite training set shouldn't be treated as literally
impossible.

The fix is **Laplace (additive) smoothing**: add a small constant `α` (typically 1) to every count
before normalizing:

```
P(xᵢ|Cₖ) = (count(xᵢ, Cₖ) + α) / (count(Cₖ) + α · |vocabulary|)
```

This guarantees every probability is nonzero without meaningfully distorting the estimates for
features that *did* appear frequently. `α = 1` is called **Laplace smoothing**; smaller `α`
(sometimes called Lidstone smoothing) is a lighter touch. `sklearn`'s Naive Bayes classes all take
this as the `alpha` parameter and default to `alpha=1.0`.

---

## 7. Naive Bayes Variants

The math above is generic in `P(xᵢ|Cₖ)` — how you actually model that per-feature likelihood
depends on the feature's data type. This gives three standard variants:

| Variant | Feature type | Likelihood model | Typical use |
|---|---|---|---|
| **MultinomialNB** | Counts (non-negative integers) | Multinomial distribution over counts | Text classification via word counts / TF |
| **BernoulliNB** | Binary (present/absent) | Bernoulli distribution | Text classification via word presence, not frequency |
| **GaussianNB** | Continuous numeric | Gaussian (normal) distribution per class | Any numeric feature set (Section 9 below) |

Picking the right variant is a modeling decision, not a hyperparameter to tune blindly — it should
match how the feature was actually generated.

---

## 8. Simple Example (Code)

The classic toy case: categorical/count features, `MultinomialNB`.

```python
from sklearn.naive_bayes import MultinomialNB
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = MultinomialNB(alpha=1.0)   # alpha = Laplace smoothing constant
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(accuracy_score(y_test, y_pred))

# Class probabilities (the posteriors from Section 6), not just the hard label:
model.predict_proba(X_test)
```

`model.fit` is exactly the "estimate `P(Cₖ)` and every `P(xᵢ|Cₖ)` from training data" step; `predict`
applies the MAP rule at inference time.

---

## 9. Handling Numerical Data — Gaussian Naive Bayes

The math in Section 6 assumed we can compute `P(xᵢ|Cₖ)` — straightforward for categorical/count
features (count frequencies), but a *continuous* numeric feature (age, salary, a sensor reading)
can take infinitely many values, so a frequency table doesn't work.

**GaussianNB**'s solution: assume that, *within each class*, each numeric feature is normally
(Gaussian) distributed. For class `Cₖ`, estimate the mean `μ` and variance `σ²` of feature `xᵢ`
from the training rows belonging to that class, then use the **Gaussian probability density
function** as the likelihood:

```
P(xᵢ | Cₖ) = 1/√(2πσ²ₖ) · exp( -(xᵢ - μₖ)² / (2σ²ₖ) )
```

So each class effectively gets its own bell curve per feature; a new point's likelihood under class
`Cₖ` is just how tall that class's bell curve is at that point. Everything else — the product over
features, dropping the denominator, MAP — carries over unchanged from Section 6, just with this PDF
in place of a frequency-table lookup.

```python
from sklearn.naive_bayes import GaussianNB

model = GaussianNB()
model.fit(X_train, y_train)     # learns mean & variance per feature, per class
y_pred = model.predict(X_test)
```

The Gaussian assumption is itself an approximation (a second "naive-ish" assumption layered on top
of feature independence) — if a numeric feature is strongly skewed or multimodal within a class,
GaussianNB's estimates will be off. In practice it's still a solid, fast baseline for numeric data,
and feature transforms from Section 05 (e.g. making a skewed feature more normal) can help it.

---

## Pros, Cons, and When to Use Naive Bayes

**Pros:**
- Very fast to train and predict — just counting/mean-variance estimation, no iterative
  optimization.
- Handles high-dimensional feature spaces gracefully (thousands of words) — exactly where models
  needing a joint distribution would struggle.
- Needs comparatively little training data, since it only estimates simple per-feature
  distributions, never a joint one.
- A strong, cheap baseline, especially for text.

**Cons:**
- The conditional independence assumption is almost always violated to some degree — accuracy
  suffers when features are strongly correlated.
- **Poor probability calibration**: the predicted class ranking is often right, but the actual
  `predict_proba` values tend to be overconfident (pushed toward 0 or 1) because correlated
  features get "double-counted" in the product. Don't trust the raw probabilities for decisions
  that need calibrated risk estimates — recalibrate (e.g. Platt scaling) if you need that.
- The zero-frequency problem requires smoothing (Section 6) or it breaks entirely.

**Typical use cases:** spam filtering, sentiment analysis, document/topic classification, and any
setting with many weak, roughly independent-ish signals — the classic "fast baseline before you
reach for something heavier" model.

---

## Key takeaways

- Conditional probability `P(A|B) = P(A∩B)/P(B)` narrows the sample space to the conditioning
  event. Independence (`P(A∩B) = P(A)P(B)`) and mutual exclusivity (`P(A∩B) = 0`) are different,
  often-confused ideas — mutually exclusive events with nonzero probability are always dependent.
- Bayes' theorem, `P(A|B) = P(B|A)P(A)/P(B)`, flips a likelihood into a posterior: **posterior ∝
  likelihood × prior**. The disease-test example shows why ignoring the prior (base rate) leads to
  badly wrong conclusions even with an "accurate" test.
- Naive Bayes applies Bayes' theorem to classification, with the naive assumption that features are
  conditionally independent given the class — turning an intractable joint likelihood into a simple
  product, `P(Cₖ|x) ∝ P(Cₖ)·∏ᵢP(xᵢ|Cₖ)`, decided by MAP (argmax over classes).
- The independence assumption is rarely literally true, but the classifier still works well because
  it usually gets the class *ranking* right even when it misjudges exact probabilities — especially
  with many weak, high-dimensional features (text).
- Zero-frequency counts zero out the whole product; Laplace/additive smoothing (`alpha`) fixes this
  by never letting any per-feature probability be exactly zero.
- Three variants match feature type: **MultinomialNB** (counts), **BernoulliNB** (binary),
  **GaussianNB** (continuous, via the Gaussian PDF as the per-feature likelihood).
- Fast, data-efficient, great for high-dimensional/text problems; weak spot is poorly calibrated
  probabilities and accuracy loss when features are strongly correlated.

**Next:** [15 — K-Nearest Neighbors](15-knn.md) — an instance-based classifier with no training
phase at all, the polar opposite of Naive Bayes' explicit probability model.
