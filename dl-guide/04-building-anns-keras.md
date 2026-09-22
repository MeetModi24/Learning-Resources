# 04 — Building ANNs in Keras

**Playlist videos 11–13.** Three hands-on projects that turn Sections 02–03's theory (perceptron,
MLP, forward propagation) into a repeatable Keras recipe: **customer churn** (binary classification),
**MNIST digits** (multiclass classification), and **graduate admission** (regression). The interesting
part isn't the code — it's that all three projects share one workflow and differ in exactly three
places: the **output layer size**, the **output activation**, and the **loss function**. Nailing that
mapping is the whole point of this section, and it sets up Section 05 (loss functions in depth).

---

## 1. The shared Keras workflow

Every supervised ANN project — regardless of task type — follows the same seven steps:

1. **Load & preprocess data.** Handle missing values, encode categoricals, and — critically —
   **scale the features** (`StandardScaler`/`MinMaxScaler`). Unscaled inputs (e.g. a salary column in
   the thousands next to an age column in the tens) make gradient descent slow and unstable; the full
   "why" is in Section 07's feature-scaling discussion. Scaling is not optional polish — an unscaled
   churn dataset can fail to converge at all.
2. **Split** into train and test sets (`train_test_split`), same as classical ML.
3. **Build the model**: a `Sequential` stack of `Dense` layers. Hidden layers use a nonlinear
   activation (ReLU is the default choice, covered properly in Section 08); the **output layer** is
   where the three projects diverge.
4. **Compile**: pick an **optimizer** (`adam` is the safe default — Section 11 covers why), a **loss
   function** (task-dependent — see the table below), and **metrics** to monitor (`accuracy` for
   classification, `mae`/`mse` for regression).
5. **Fit**: `model.fit(X_train, y_train, epochs=..., batch_size=..., validation_split=...)`. Keras
   trains for the given number of epochs, running mini-batches of `batch_size` per step, and — if
   `validation_split` is set — holds out a slice of the training data each epoch to report validation
   loss/metrics without touching the test set.
6. **Evaluate**: `model.evaluate(X_test, y_test)` — the honest, held-out number.
7. **Predict**: `model.predict(X_new)` — raw outputs (probabilities for classification, a continuous
   value for regression), which the caller thresholds/interprets as needed.

Everything else in this section is just: what changes in steps 3–4 per project.

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense

model = Sequential([
    Dense(32, activation='relu', input_dim=n_features),
    Dense(16, activation='relu'),
    Dense(OUTPUT_UNITS, activation=OUTPUT_ACTIVATION),   # <-- varies by task
])

model.compile(optimizer='adam', loss=LOSS_FN, metrics=METRICS)  # <-- varies by task

history = model.fit(X_train, y_train, epochs=100, batch_size=32, validation_split=0.2)
model.evaluate(X_test, y_test)
model.predict(X_new)
```

---

## 2. Project 1 — Customer Churn Prediction (binary classification)

**Task:** given tabular customer attributes (tenure, contract type, monthly charges, etc.), predict
whether the customer will churn (0/1).

- **Input:** a plain tabular feature vector — one `Dense` layer's worth of numbers per customer, after
  encoding categoricals and scaling numerics.
- **Output layer:** **1 neuron, sigmoid activation.** Sigmoid squashes the single output logit into
  `(0, 1)`, which is exactly the reading of "probability the customer churns" you want. This is the
  same sigmoid from the perceptron/logistic regression discussion in Section 02 — a single ANN output
  neuron with a sigmoid is literally logistic regression's output layer; the difference is everything
  feeding into it was learned by hidden layers instead of hand-crafted.
- **Loss:** `binary_crossentropy` (BCE). BCE and sigmoid are a matched pair for the same reason they
  were in logistic regression: BCE is derived from the likelihood of a Bernoulli outcome, and its
  gradient with respect to the sigmoid's input has a clean, well-behaved form (no vanishing-gradient
  trap that other loss/activation pairings can produce). Using MSE here instead would work in
  principle but trains worse — get used to "loss must match the output's probabilistic assumption,"
  it's the theme of Section 05.
- **Metric:** `accuracy` (with class imbalance, this alone can be misleading — same caveat as in
  classical ML).

```python
model.add(Dense(1, activation='sigmoid'))
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])
```

---

## 3. Project 2 — Handwritten Digit Classification on MNIST (multiclass classification)

**Task:** given a 28×28 grayscale image of a handwritten digit, predict which of the 10 digits (0–9)
it is.

- **Input:** MNIST images are `28×28` pixel grids. A plain ANN has no notion of 2D spatial structure —
  its `Dense` layers only accept flat vectors — so the **naive ANN approach is to flatten each image
  into a 784-length vector** (`Flatten()` layer, or reshape before feeding in). This throws away all
  spatial locality (which pixels are neighbors), which is precisely why CNNs (Section 13) beat plain
  ANNs on images: convolution preserves and exploits that 2D structure instead of discarding it. Treat
  this project as "the ANN baseline you'll later see CNNs crush," not the final word on image models.
- **Preprocessing:** pixel values are integers `0–255`; **normalize by dividing by 255** so inputs land
  in `[0, 1]`. This is the same scaling principle as Project 1 — gradient descent trains better on
  small, comparable-magnitude inputs.
- **Output layer:** **10 neurons (one per digit class), softmax activation.** Softmax turns the 10 raw
  output logits into a probability distribution that sums to 1 across all classes — each output
  neuron's value is "probability of being this digit, given it must be exactly one of the 10." Sigmoid
  wouldn't work here because sigmoid treats each output independently (they wouldn't sum to 1, and
  nothing enforces "exactly one class wins"); softmax couples the outputs together, which is what
  mutually-exclusive multiclass labels require.
- **Loss:** `categorical_crossentropy` if labels are one-hot encoded, or `sparse_categorical_crossentropy`
  if labels are plain integers (0–9) — same math, different label encoding, Keras just wants to know
  which format it's getting.
- **Metric:** `accuracy`.

```python
from tensorflow.keras.layers import Flatten

model = Sequential([
    Flatten(input_shape=(28, 28)),
    Dense(128, activation='relu'),
    Dense(32, activation='relu'),
    Dense(10, activation='softmax'),
])
model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
```

---

## 4. Project 3 — Graduate Admission Prediction (regression)

**Task:** given applicant features (GRE score, TOEFL score, university rating, GPA, research
experience, etc.), predict a continuous **chance of admit** (e.g. `0.0`–`1.0`, but treated as a
real-valued target, not a probability from a classifier).

- **Input:** tabular, scaled — same as Project 1.
- **Output layer:** **1 neuron, no activation (linear/identity).** Any bounded activation (sigmoid,
  tanh) would clamp the output into a fixed range and distort the raw regression target; "no
  activation" just means `output = z` (the raw weighted sum), letting the network predict any real
  number the target actually needs. This is the direct regression analogue of "why sigmoid for
  binary" — the output layer's activation has to match what kind of number you're trying to produce.
- **Loss:** `mean_squared_error` (MSE) — penalizes squared distance from the true value, the same MSE
  from linear regression in the ML guide.
- **Metric:** `mae` and/or `mse` reported alongside the loss so you can read the error in the target's
  own units (MAE) as well as the squared-error training signal (MSE).

```python
model.add(Dense(1))  # no activation = linear
model.compile(optimizer='adam', loss='mean_squared_error', metrics=['mae'])
```

---

## 5. The table to memorize

This is the single fact from this whole section worth committing to memory — it's what Section 05
builds on when it goes deeper into *why* each loss function has the shape it does.

| Problem type | Output neurons | Output activation | Loss function |
|---|---|---|---|
| **Binary classification** | 1 | sigmoid | binary_crossentropy |
| **Multiclass classification** | *n* (number of classes) | softmax | categorical_crossentropy (or sparse variant) |
| **Regression** | 1 | linear (none) | mean_squared_error |

The pattern underneath the table: **output activation encodes what kind of number the task needs**
(a probability, a distribution over classes, or an unrestricted real number), and **the loss function
is the one whose gradient behaves well against that specific activation** — not an arbitrary pairing.

---

## 6. Overfitting shows up immediately

Run any of these three projects for enough epochs and plot `history.history['loss']` against
`history.history['val_loss']` (Keras records both automatically when `validation_split` is set): early
on both drop together, then at some point **training loss keeps falling while validation loss flattens
or rises again**. That gap is overfitting — the network is memorizing the training set's specifics
rather than the pattern that generalizes, exactly the bias–variance story from the ML guide, now with
a live training curve to watch it happen in real time.

This is a preview, not a fix — the actual toolbox (early stopping, dropout, L1/L2 weight regularization,
reducing model capacity) is Section 07's job. For now, the takeaway is purely diagnostic: **always plot
train vs. validation loss before trusting a model**, on all three project types.

---

## Key takeaways

- All three projects share one Keras skeleton: preprocess (scale!) → split → `Sequential` of `Dense`
  layers → compile (optimizer + loss + metrics) → fit (epochs, batch_size, validation_split) →
  evaluate → predict. Only the output layer and loss change.
- **Binary classification** (churn): 1 output neuron, sigmoid, `binary_crossentropy` — the ANN
  generalization of logistic regression's output.
- **Multiclass classification** (MNIST): flatten the 28×28 image to 784 features (the naive-ANN
  approach; CNNs in Section 13 do better by not discarding spatial structure), normalize pixels
  `/255`, 10 output neurons, softmax, `categorical_crossentropy`/`sparse_categorical_crossentropy`.
- **Regression** (admission): 1 output neuron with **no** activation, `mean_squared_error` — the
  output activation has to leave the raw real-valued prediction untouched.
- Memorize the neurons/activation/loss table in Section 5 — it's the direct on-ramp to Section 05's
  deeper treatment of loss functions.
- Plot train vs. validation loss on every project — the gap between them is overfitting, and it's the
  motivating problem for Section 07's regularization toolbox.

**Next:** [05 — Loss Functions & Backpropagation](05-loss-and-backpropagation.md) — why these specific
loss functions, and how the network actually uses them to update every weight.
