# 03 — MLP Notation & Forward Propagation

**Playlist videos 8–10.** Section 02 left us with a single perceptron — one linear boundary, unable
to solve XOR or anything else that isn't linearly separable. This section fixes that by stacking
perceptrons into **layers** (the Multi-Layer Perceptron), gives that structure precise notation, and
walks through **forward propagation** — the mechanical process by which a network turns an input into
a prediction.

---

## 1. From One Perceptron to a Network

A single perceptron draws one straight line (or hyperplane) through the feature space. That's its
ceiling — Section 02's XOR problem is exactly a dataset no single straight line can separate.

The fix isn't a smarter perceptron — it's *more of them, arranged in layers*:

- **Input layer** — not really a computing layer, just the raw features handed to the network.
- **Hidden layer(s)** — one or more layers of neurons sitting between input and output. "Hidden"
  because their outputs aren't the thing you observe — they're intermediate computations.
- **Output layer** — produces the final prediction.

This is the **Multi-Layer Perceptron (MLP)**, and it's the same architecture the guide calls **ANN**.

### Why stacking layers solves non-linearity

Each individual neuron still only draws a simple linear boundary (a line/plane), *then* applies a
non-linear activation. The trick is **composition**:

- A hidden neuron can learn a boundary like "is this point to the left of line A?"
- Another hidden neuron learns "is this point below line B?"
- The output neuron combines those two intermediate answers into a decision region that is
  **curved / non-linear** in the original input space — e.g. the XOR pattern, which needs two lines,
  not one, to separate correctly.

Stack more neurons and more layers, and you can approximate arbitrarily complex decision boundaries.
This composability is *why* depth matters — it's not just "more capacity," it's capacity structured so
simple pieces combine into complex shapes. Without a non-linear activation at each layer, though, this
whole argument collapses: stacking purely linear layers just gives you another linear function (matrix
multiplication is composable), so **non-linear activations are what make depth actually pay off**.

### Why hidden layers = automatic feature learning

This is the connection back to Section 01's biggest DL-vs-ML claim: deep learning learns its own
features instead of you hand-engineering them. Concretely, each hidden layer's output — its vector of
activations — is a **learned re-representation of the input**, not the raw input itself. Early hidden
layers tend to learn simple, low-level combinations of the input; later hidden layers combine *those*
into higher-level, more abstract combinations. By the time the signal reaches the output layer, the
network has effectively engineered a set of features for itself — well-suited to producing a good
prediction — with no human choosing them. That's the mechanism behind "automatic feature engineering."

---

## 2. MLP Notation

This notation is used through the rest of the guide (backpropagation especially), so it's worth
locking down precisely now.

- **Layers are indexed by a superscript `[l]`**, where `l = 0` is the input layer, `l = 1` is the
  first hidden layer, ..., `l = L` is the output layer.
- **`a^[l]`** — the activation (output) vector of layer `l`. By convention `a^[0] = x`, the raw input
  vector — the input layer has no computation, it just *is* the data.
- **`W^[l]`** — the weight matrix connecting layer `l-1` to layer `l`.
- **`b^[l]`** — the bias vector for layer `l`.
- **`z^[l]`** — the pre-activation (linear combination, before the non-linearity) for layer `l`:

  ```
  z^[l] = W^[l] · a^[l-1] + b^[l]
  ```

- **`g^[l]`** — the activation function used at layer `l` (sigmoid, tanh, ReLU, etc. — Section 08).
  The activation itself:

  ```
  a^[l] = g^[l](z^[l])
  ```

- **Subscripts index individual neurons** within a layer. So `a^[l]_j` is the activation of the
  `j`-th neuron in layer `l`, and `W^[l]_{jk}` is the weight connecting the `k`-th neuron of layer
  `l-1` to the `j`-th neuron of layer `l`. **Superscript = which layer, subscript = which neuron
  (and, for weights, which connection).**

### Shapes — the part people get wrong

If layer `l-1` has `n^[l-1]` neurons and layer `l` has `n^[l]` neurons:

| Quantity | Shape | Why |
|---|---|---|
| `a^[l-1]` | `(n^[l-1], 1)` | one activation per neuron in the previous layer |
| `W^[l]` | `(n^[l], n^[l-1])` | **rows = neurons in the current layer, columns = neurons in the previous layer** |
| `b^[l]` | `(n^[l], 1)` | one bias per neuron in the current layer |
| `z^[l]`, `a^[l]` | `(n^[l], 1)` | one value per neuron in the current layer |

The row/column rule for `W^[l]` is the one thing to memorize: it's what makes
`W^[l] · a^[l-1]` — a `(n^[l], n^[l-1])` matrix times a `(n^[l-1], 1)` vector — produce a
`(n^[l], 1)` result, i.e. exactly one pre-activation per neuron in the current layer.

### Counting parameters

Each layer `l` contributes:

```
params(l) = (n^[l] × n^[l-1])   [weights]   +   n^[l]   [biases]
```

Total network parameters = sum of `params(l)` over all layers `l = 1 … L`. This is worth computing by
hand once — it's exactly what `model.summary()` in Keras reports per layer, and it's the number that
explains why deep networks need so much data and compute (Section 01).

---

## 3. Forward Propagation

**Forward propagation is just the chain rule of computation, run forward:** feed the input in, and at
every layer, apply a linear transform then a non-linearity, layer by layer, until you reach the
output. That output *is* the network's prediction — this whole process is **inference**, not training
(no loss, no gradients yet — that's Section 05).

For a single example `x`, forward propagation is literally the two equations above, repeated for
`l = 1, 2, …, L`:

```
a^[0] = x

for l = 1 to L:
    z^[l] = W^[l] · a^[l-1] + b^[l]
    a^[l] = g^[l](z^[l])

ŷ = a^[L]          # the final prediction
```

Every layer is "(matrix multiply) → (add bias) → (squash with a non-linearity)," and the output of
one layer is simply the input to the next. Nothing more exotic is happening — a 20-layer network is
this same two-line loop executed 20 times.

### Vectorized form for a batch

In practice you never propagate one example at a time — you propagate a whole batch through matrix
operations, because that's what GPUs are fast at (Section 01's "why now"). Stack `m` examples as
columns of a matrix `A^[l-1]` of shape `(n^[l-1], m)`; the bias broadcasts across the batch:

```
Z^[l] = W^[l] · A^[l-1] + b^[l]      # shape (n^[l], m), b^[l] broadcasts over the m columns
A^[l] = g^[l](Z^[l])                 # elementwise activation, same shape
```

Same equations, one extra dimension. This is exactly what a Keras/TensorFlow `Dense` layer computes
internally for you.

---

## 4. A Tiny Concrete Example

Take a network with **2 inputs → 1 hidden layer of 2 neurons → 1 output neuron**:

- `x = [x1, x2]`, so `a^[0]` has shape `(2, 1)`.
- Hidden layer: `n^[1] = 2`, so `W^[1]` is `(2, 2)`, `b^[1]` is `(2, 1)`.
  - `z^[1] = W^[1] · a^[0] + b^[1]` → 2 numbers.
  - `a^[1] = g(z^[1])` → 2 numbers (say, each passed through sigmoid).
- Output layer: `n^[2] = 1`, so `W^[2]` is `(1, 2)`, `b^[2]` is `(1, 1)`.
  - `z^[2] = W^[2] · a^[1] + b^[2]` → 1 number.
  - `a^[2] = g(z^[2])` → the prediction `ŷ`.

Parameter count: layer 1 has `2×2 + 2 = 6`, layer 2 has `1×2 + 1 = 3` → **9 parameters** total, all
currently at whatever values they were initialized to (Section 09 covers *how* to initialize them
well; training, covered from Section 05 on, is what adjusts them from these arbitrary starting values
into ones that actually predict something useful).

Conceptually: each of the 2 hidden neurons draws its own line across the `(x1, x2)` plane and squashes
the result; the output neuron then combines those two squashed values into one final line/squash. Two
simple linear boundaries, combined, can carve out a non-linear region — this tiny network is already
capable of representing XOR, given the right weights (which is precisely what training would find).

---

## 5. Minimal Keras Code

Forward propagation is what happens every time you call `model.predict(x)` — it's the architecture
above, expressed as a `Sequential` stack of `Dense` layers:

```python
from tensorflow import keras
from tensorflow.keras import layers

model = keras.Sequential([
    layers.Dense(2, activation="sigmoid", input_shape=(2,)),  # hidden layer: 2 neurons
    layers.Dense(1, activation="sigmoid"),                     # output layer: 1 neuron
])

model.summary()      # shows Param # per layer — matches the hand-count above (6 + 3 = 9)
```

`model.summary()` will report exactly the 6 params for the hidden `Dense` layer and 3 for the output
`Dense` layer, confirming the shape/parameter-counting rules above. No training has happened yet —
this model, right now, with random initial weights, already does a full forward pass if you call
`model.predict(...)` on it; it just won't predict anything *useful* until Section 05 onward.

---

## Key takeaways

- **MLP = layers of perceptrons** (input → one or more hidden layers → output), each neuron applying
  a linear transform followed by a non-linear activation.
- Stacking layers with non-linear activations lets simple linear boundaries **compose** into complex,
  non-linear decision boundaries — this is what solves XOR and beyond, the exact wall Section 02's
  single perceptron hit.
- Hidden layers are also *why* deep learning does automatic feature learning (Section 01): each
  hidden layer's activations are a learned re-representation of the input, not the raw data.
- **Notation**: `a^[l] = g(z^[l])`, `z^[l] = W^[l]·a^[l-1] + b^[l]`, with `a^[0] = x`. Superscript =
  layer, subscript = neuron. `W^[l]` has shape `(n^[l], n^[l-1])` — rows = current layer, columns =
  previous layer.
- **Forward propagation** = repeatedly applying (linear transform → non-linear activation), layer by
  layer, until the output layer produces the prediction. It's inference, not training — vectorized
  over a batch it's just `Z^[l] = W^[l]·A^[l-1] + b^[l]`, `A^[l] = g(Z^[l])`.
- Parameter count per layer = `n^[l] × n^[l-1]` weights + `n^[l]` biases — this is exactly what
  `model.summary()` reports.

**Next:** [04 — Building ANNs in Keras](04-building-anns-keras.md) — turning this notation into a
real trained classifier, MNIST digit classifier, and regressor.
