# 25 — Hyperparameter Tuning using Optuna (Bayesian Optimization)

**Playlist video 134.** The final section. Every model in this guide had knobs you set before
training started — `max_depth`, `C`, `learning_rate`, `k`, `n_estimators`. Choosing those knobs
well is its own problem, distinct from fitting the model. This section covers what that problem
is, why naive search strategies (grid/random, seen already in Sections 16, 17, 19, 21) don't scale,
and how Bayesian optimization — as implemented by **Optuna** — fixes that.

---

## 1. Parameters vs Hyperparameters

This distinction is the whole premise of the section:

- **Parameters** are **learned from data during training**. Linear regression's coefficients, a
  decision tree's actual split points and leaf values, a neural net's weights. You never set these
  by hand; the optimization algorithm (gradient descent, the CART splitting procedure, etc.) finds
  them.
- **Hyperparameters** are **set before training starts** and control *how* the learning happens.
  They are not learned from the training data by the model itself — you (or a search procedure)
  choose them. Examples seen throughout this guide: `learning_rate` and `n_estimators` in gradient
  boosting (Section 21), `max_depth`/`min_samples_split` in decision trees (Section 17), `C` and
  the kernel choice in SVM (Section 16), `k` in KNN (Section 15), `alpha` in Ridge/Lasso
  (Section 11).

**Tuning hyperparameters is model selection**: for a fixed algorithm, you're searching over the
space of *configurations* of that algorithm to find the one that generalizes best. It sits on top
of ordinary training — for every hyperparameter combination you try, you still fully train (and
cross-validate) the model to see how good that combination is.

---

## 2. Baseline Strategies: Grid Search and Random Search

Both of these already appeared in earlier sections (e.g. tuning Random Forest and XGBoost in
Sections 19/21) as `GridSearchCV` / `RandomizedSearchCV`. Recapping them here because Bayesian
optimization is explicitly the answer to their shared weakness.

### Grid Search
Define a discrete grid of values for each hyperparameter and **exhaustively** try every
combination (via cross-validation for each). Thorough within the grid you specified, but the
number of combinations grows **exponentially** with the number of hyperparameters and grid
resolution — 5 hyperparameters × 5 values each is `5^5 = 3125` full training-and-CV runs. Expensive
models (deep trees, boosting with many trees, neural nets) make this infeasible quickly.

### Random Search
Instead of trying every combination, **sample** a fixed number of random combinations from the
hyperparameter space (distributions, not just a fixed grid) and evaluate those. Bergstra & Bengio's
finding (the reason this is standard practice, not just a shortcut): in **high-dimensional**
spaces, most hyperparameters don't matter much for a given problem — only a few dimensions actually
drive performance. Random search explores the full range of every dimension instead of wasting
budget on a dense grid over dimensions that barely matter, so it's typically more efficient per
evaluation than grid search at the same budget.

### The shared weakness
Both are **memoryless**. Every trial — grid point or random draw — is chosen independently of
every previous trial's result. Grid search doesn't notice that a region of the space is clearly
bad and keep grinding through it anyway; random search doesn't notice that a region looks
promising and sample more densely there. Neither *learns* from the trials it has already run. That
is exactly the gap Bayesian optimization closes.

---

## 3. Bayesian Optimization: Searching with Memory

Bayesian optimization treats hyperparameter tuning as an optimization problem over an **unknown,
expensive-to-evaluate function**: `f(hyperparameters) → validation score`. It's unknown because you
have no formula for it — you only find out `f`'s value at a point by actually training and
validating a model there. It's expensive because each evaluation is a full train+CV cycle. The
strategy has two parts working in a loop:

### Surrogate model
Because evaluating the real objective is expensive, Bayesian optimization builds a cheap
**probabilistic model** — the surrogate — of the objective function, fit on the (hyperparameter,
score) pairs observed so far. Classically this is a **Gaussian Process**, which gives not just a
predicted score at any untried point but also an estimate of *uncertainty* at that point (wide
where few trials have been nearby, narrow where many have). Optuna's default sampler (TPE, below)
builds a related but different kind of probabilistic model, but the role is the same: a fast stand-
in for the real objective that gets more accurate as more real trials come in.

### Acquisition function
Given the surrogate's prediction + uncertainty everywhere in the space, the **acquisition function**
scores every candidate point by how *worthwhile* it would be to try next, and the next trial is
the point that maximizes it. This is where **exploration vs exploitation** gets balanced explicitly:

- **Exploitation** — try points near the best-so-far result, because the surrogate predicts they're
  good.
- **Exploration** — try points where the surrogate is *uncertain*, because they might hide an even
  better region that hasn't been sampled yet.

A good acquisition function (e.g. Expected Improvement) picks points that are attractive on
*either* count — likely to be good, or likely to teach you something new about the space.

### Why this beats grid/random
Each new trial's location is chosen **using everything learned from all previous trials** via the
surrogate. The search actively steers toward promising regions and away from regions already shown
to be bad, instead of blindly covering the space. In practice this means Bayesian optimization
reaches a good hyperparameter configuration in **far fewer evaluations** than grid or random search
need — the point of the whole exercise when each evaluation is a full training run.

---

## 4. Optuna

Optuna is a hyperparameter optimization framework built around this Bayesian-optimization idea (plus
some practical engineering on top). Two core objects:

### The objective function
You write a Python function that takes a single argument, `trial`, an object representing one run
of the search. Inside it you:

1. Ask `trial` to **suggest** a value for each hyperparameter, using a method matching the
   parameter's type/range:
   - `trial.suggest_float("lr", 1e-4, 1e-1, log=True)` — continuous range.
   - `trial.suggest_int("max_depth", 2, 32)` — integer range.
   - `trial.suggest_categorical("kernel", ["linear", "rbf", "poly"])` — a fixed set of choices.
2. Train/evaluate a model using those suggested values (cross-validation, as always).
3. **Return** the score being optimized (e.g. mean CV accuracy, or negative RMSE).

### The study
A `study` runs the objective repeatedly for a target number of trials, choosing whether to
`maximize` or `minimize` the returned score, and internally drives the sampler that decides what
values to suggest on each trial:

```python
import optuna
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score

def objective(trial):
    n_estimators = trial.suggest_int("n_estimators", 50, 300)
    max_depth = trial.suggest_int("max_depth", 2, 32)
    min_samples_split = trial.suggest_int("min_samples_split", 2, 20)

    model = RandomForestClassifier(
        n_estimators=n_estimators,
        max_depth=max_depth,
        min_samples_split=min_samples_split,
        random_state=42,
    )
    score = cross_val_score(model, X_train, y_train, cv=5, scoring="accuracy").mean()
    return score

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=100)

print(study.best_params)   # best hyperparameter combination found
print(study.best_value)    # its CV score
```

Every call to `objective` inside `study.optimize` is one trial: Optuna's sampler picks the
suggested values (informed by all prior trials' scores, per the surrogate + acquisition loop in
Section 3), the function trains and cross-validates, and the returned score feeds back into the
sampler for the next trial.

### Key features

- **TPE (Tree-structured Parzen Estimator)** — Optuna's default sampler, and a Bayesian approach
  in the same family described above: it models the distribution of hyperparameters that led to
  *good* trials versus *bad* trials, and samples the next trial from the region that looks more
  like "good." This is Optuna's concrete answer to "how do I build a surrogate + pick the next
  point," without requiring a Gaussian Process.
- **Pruning** — Optuna can **early-stop unpromising trials** partway through training (e.g. a
  trial's intermediate validation score after a few boosting rounds is already clearly worse than
  other trials at the same point), instead of wasting the full training budget on a configuration
  that's obviously not going to win. This is a major source of Optuna's practical speedup on top of
  the sampling strategy itself.
- **Define-by-run search space** — the search space isn't declared upfront as a static config;
  it's defined dynamically *inside* the objective function as it runs (e.g. you can suggest a
  different set of hyperparameters depending on which `kernel` was suggested earlier in the same
  trial). This makes conditional/nested search spaces natural to express.
- **Visualization** — built-in plots (`optuna.visualization`) for optimization history, parameter
  importance, and slice plots, useful for understanding *which* hyperparameters actually mattered
  for a given problem — echoing the Bergstra & Bengio point from Section 2.

---

## 5. Practical Guidance

- **Always tune using cross-validation on the training set**, never the test set. The score
  `study.optimize` is chasing must be a CV estimate; the test set stays untouched until the very
  end, after the best configuration is locked in. This is the same discipline as every other
  section in this guide (Sections 09, 13) — tuning is still "fitting" in a broad sense (fitting the
  *hyperparameters* to data), so it's just as capable of overfitting.
- **Watch for overfitting the validation set.** Run enough trials to find a good region, but if you
  run so many trials against the same validation/CV split that you start cherry-picking noise, the
  reported CV score stops being a trustworthy estimate of real generalization. A held-out test set
  evaluated once at the end is the check against this.
- **Which method to use when:**
  - **Small, low-dimensional search space, cheap model** → grid search is fine; exhaustive is
    affordable and guarantees you've covered what you specified.
  - **Larger space, more hyperparameters, or an expensive model to train** → random search as a
    quick baseline, Bayesian optimization (Optuna) when you want the best result for a limited
    evaluation budget — which is most real-world tuning problems, since each trial is a full
    training run.

---

## Key takeaways

- **Parameters** are learned from data during training; **hyperparameters** are set before
  training and control the learning process. Tuning hyperparameters is a model-selection problem
  layered on top of ordinary training.
- **Grid search** is exhaustive but exponential in cost; **random search** samples randomly and is
  usually more efficient in high dimensions (Bergstra & Bengio) because not every hyperparameter
  matters equally. Both are memoryless — no trial learns from earlier trials.
- **Bayesian optimization** fixes that: fit a cheap **surrogate model** of the objective from
  trials seen so far, use an **acquisition function** to pick the next point by balancing
  **exploration** (uncertain regions) and **exploitation** (regions predicted to be good), repeat.
  This uses information from past trials to find good hyperparameters in far fewer evaluations.
- **Optuna** operationalizes this: define an **objective(trial)** function that calls
  `trial.suggest_float/int/categorical` to propose hyperparameters, trains/evaluates, and returns a
  score; a **study** runs `optimize` for N trials, driven by default by the **TPE** sampler
  (a Bayesian approach). **Pruning** early-stops unpromising trials; the search space can be
  defined dynamically (define-by-run); built-in visualization shows optimization history and
  parameter importance.
- Tune on cross-validated training scores only, keep a held-out test set untouched until the end,
  and match the method to the budget: grid for small/cheap spaces, Bayesian/Optuna for
  large/expensive ones.

---

This is the last section of the classical-ML guide. Together, Sections 01–25 track the CampusX
"100 Days of Machine Learning" playlist end to end: framing a problem, gathering and cleaning data,
feature engineering, the core supervised/unsupervised algorithms, evaluation, ensembling, and
finally the model-selection layer — hyperparameter tuning — that sits on top of all of them. See
the [README](README.md) for the full table of contents.
