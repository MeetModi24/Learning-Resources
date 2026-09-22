# 12 — Hyperparameter Tuning in Deep Learning

**Playlist video 39.** This is the capstone of the ANN block (Sections 02–12): once you know what
architecture to build (Section 03–04), how it learns (05–07), and the tricks that make training
stable (08–11 — activations, init, batch norm, optimizers), the last question is *which settings of
all those knobs* actually work best for your problem. This section covers what a hyperparameter is,
why searching for good ones is expensive and non-trivial, and how to automate the search with
**Keras Tuner**.

---

## 1. Hyperparameters vs. Parameters

A neural network has two very different kinds of numbers attached to it:

- **Parameters (weights and biases)** — learned *by* the network, *during* training, via
  backpropagation + gradient descent (Section 05). You never set these by hand.
- **Hyperparameters** — settings you (or a search algorithm) choose **before training starts**.
  Gradient descent cannot learn these — they define the training process and the model's capacity,
  not a differentiable output of it.

Hyperparameters you've already met in isolation across Sections 02–11, collected here as the full
list you'd actually tune:

| Hyperparameter | What it controls | Covered in |
|---|---|---|
| Number of hidden layers | Model depth / capacity | Section 03 |
| Number of neurons per layer | Model width / capacity | Section 03 |
| Activation function | Sigmoid, tanh, ReLU and variants | Section 08 |
| Weight initialization | Xavier/Glorot, He | Section 09 |
| Learning rate | Step size of gradient descent | Section 07, 11 |
| Optimizer choice | SGD, Momentum, RMSProp, Adam, ... | Section 11 |
| Batch size | How many samples per gradient step | Section 07 |
| Number of epochs | How long to train (often tuned jointly with early stopping) | Section 07 |
| Dropout rate | Fraction of units dropped for regularization | Section 07 |
| Regularization strength (L1/L2) | Penalty on weight magnitude | Section 07 |

The classical-ML version of this problem — grid search, random search, cross-validation — is
covered in [`../ml-guide/25-hyperparameter-tuning.md`](../ml-guide/25-hyperparameter-tuning.md). The
concepts transfer directly; what's different in DL is *what's* being tuned (architecture choices, not
just a `C` or `max_depth`) and *how expensive* each trial is.

---

## 2. Why This Is Hard

Two things make hyperparameter tuning in deep learning much harder than in classical ML:

1. **Combinatorial explosion.** Even a modest search — 3 choices of layer count × 3 neuron counts ×
   3 activations × 3 learning rates × 3 batch sizes — is already 3⁵ = 243 combinations. Add
   dropout and optimizer choice and it explodes further. This is exactly the curse that makes
   exhaustive grid search infeasible.
2. **Each trial is a full training run.** Unlike tuning a `k` in k-NN, evaluating one hyperparameter
   combination in DL means *training an entire neural network from scratch* — potentially hours on a
   GPU. You cannot afford to evaluate all 243+ combinations; the search strategy itself has to be
   efficient.
3. **Hyperparameters interact.** A learning rate that's good for a small network can be wrong for a
   deep one; a good dropout rate depends on the number of neurons it's dropping from. You can't
   always tune one at a time and expect the optimum to hold when you move to the next.

---

## 3. Search Strategies

### Grid Search

Exhaustive: define a fixed set of values for each hyperparameter, try every combination. Simple and
guaranteed to cover the whole specified grid, but the number of trials grows **exponentially** with
the number of hyperparameters — quickly impossible when each trial is a full training run.

### Random Search

Instead of trying every combination, sample combinations at random from the search space for a fixed
budget of trials. Counterintuitively, this is often *more* efficient than grid search. The classic
justification (Bergstra & Bengio, 2012): in most real problems, only a **few hyperparameters
actually matter** for a given dataset — the rest barely affect the result. Grid search wastes most of
its trials varying the unimportant dimensions in lockstep with the important one, effectively testing
only a few distinct values of the dimension that matters. Random search, for the same trial budget,
samples many more distinct values along every dimension — so it explores the important dimension far
more thoroughly by chance. Same idea applies in `../ml-guide`'s `RandomizedSearchCV` vs `GridSearchCV`.

### Bayesian Optimization / Hyperband

Both are "smarter" than blind sampling — they use the results of past trials to decide what to try
next, instead of sampling independently.

- **Bayesian optimization** builds a probabilistic model of "hyperparameters → validation score" from
  the trials run so far, then picks the next combination predicted to do well (balancing exploiting
  what looks promising against exploring what's uncertain).
- **Hyperband** attacks the cost problem differently: instead of training every candidate to
  completion, it trains *many* candidates for a *small* budget (a few epochs), throws away the worst
  performers, and gives the survivors a bigger budget — repeating this "successive halving" until
  only a few well-performing configurations remain and are trained fully. This means compute is spent
  where it matters, and clearly bad configurations are killed early instead of run to completion.
  It's usually the best default when training is expensive, which in DL it always is.

---

## 4. Keras Tuner

**Keras Tuner** is a library that automates this search for Keras models — you describe the search
*space*, pick a search *strategy*, give it a training *budget*, and it runs the trials for you.

### The workflow

**1. Write a model-building function that takes `hp`.** Instead of hardcoding hyperparameter values,
you query an `hp` object for each one — this is what defines the search space:

```python
import keras_tuner as kt
from tensorflow import keras

def build_model(hp):
    model = keras.Sequential()
    model.add(keras.layers.Flatten(input_shape=(28, 28)))

    # number of hidden layers + neurons per layer are both searched
    for i in range(hp.Int("num_layers", min_value=1, max_value=3)):
        model.add(keras.layers.Dense(
            units=hp.Int(f"units_{i}", min_value=32, max_value=256, step=32),
            activation=hp.Choice("activation", values=["relu", "tanh"]),
        ))

    model.add(keras.layers.Dense(10, activation="softmax"))

    learning_rate = hp.Float("learning_rate", min_value=1e-4, max_value=1e-2, sampling="log")
    model.compile(
        optimizer=keras.optimizers.Adam(learning_rate=learning_rate),
        loss="sparse_categorical_crossentropy",
        metrics=["accuracy"],
    )
    return model
```

`hp.Int` / `hp.Float` / `hp.Choice` are the building blocks — they define a range or a discrete set
for one hyperparameter, and the tuner samples from them according to its strategy.

**2. Pick a tuner with an objective and a budget.**

```python
tuner = kt.RandomSearch(
    build_model,
    objective="val_accuracy",
    max_trials=20,            # total number of configurations to try
    directory="tuner_logs",
    project_name="mnist_tuning",
)

# or, for a smarter budget-aware search:
tuner = kt.Hyperband(
    build_model,
    objective="val_accuracy",
    max_epochs=20,
    directory="tuner_logs",
    project_name="mnist_hyperband",
)

# or a model-based search:
tuner = kt.BayesianOptimization(
    build_model,
    objective="val_accuracy",
    max_trials=20,
)
```

**3. Search — same shape as `model.fit`.**

```python
tuner.search(X_train, y_train, epochs=10, validation_data=(X_val, y_val))
```

**4. Pull out the winner.**

```python
best_hp = tuner.get_best_hyperparameters(num_trials=1)[0]
best_model = tuner.get_best_models(num_models=1)[0]
```

`get_best_hyperparameters` gives you the winning `hp` values (useful for inspection or rebuilding);
`get_best_models` gives you the trained model directly. A common final step is to take `best_hp`,
rebuild with `build_model(best_hp)`, and train it for longer on the full training set — the tuner's
own runs are usually kept short to make the search itself affordable.

---

## 5. Practical Guidance

- **Learning rate matters most.** Get it in the right order of magnitude before anything else — a bad
  learning rate makes every other hyperparameter's effect impossible to read. Search it on a **log**
  scale (`sampling="log"` above), not linearly.
- **Architecture size (layers/neurons) matters next.** Tune coarsely first (few discrete choices),
  then refine around what worked.
- **Default to Random Search or Hyperband over Grid Search.** Grid search's exhaustiveness is rarely
  worth its cost once you have more than two or three hyperparameters in play — the same lesson as in
  `../ml-guide`.
- **Keep tuning-phase training short.** The whole point of Hyperband's early-kill strategy is not to
  waste full training budgets on bad configurations — don't fight that by forcing every trial to run
  to convergence.

---

## Key takeaways

- **Hyperparameters** (layers, neurons, activation, learning rate, batch size, epochs, optimizer,
  dropout, regularization strength, init) are set *before* training and chosen by search, not learned
  by gradient descent — unlike weights and biases.
- Tuning is hard because the search space is combinatorial, each trial requires a full (expensive)
  training run, and hyperparameters interact with each other.
- **Random search beats grid search** in most realistic settings, because usually only a few
  hyperparameters matter and random sampling covers those dimensions far better for the same budget.
- **Hyperband** (successive halving) and **Bayesian optimization** (model-based sampling) are smarter
  than blind search and are the practical default for expensive DL training.
- **Keras Tuner** operationalizes this: write `build_model(hp)` using `hp.Int`/`hp.Float`/`hp.Choice`
  to define the search space, choose a tuner (`RandomSearch`, `Hyperband`, `BayesianOptimization`)
  with an objective and budget, call `tuner.search(...)`, then retrieve the winner with
  `get_best_hyperparameters` / `get_best_models`.

**Next:** [13 — CNN Foundations](13-cnn-foundations.md) — leaving the fully-connected ANN behind for
an architecture built for images.
