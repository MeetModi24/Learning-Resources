# 02 — The Perceptron

**Playlist videos 4–7.** The perceptron is the single artificial neuron everything else in this
guide is built from: what it is, the line it draws through space, how it learns by trial and error,
how that learning is really gradient descent on a loss function, and the hard limit that eventually
stalled the whole field.

---

## 1. What is a Perceptron?

A **perceptron** takes a set of inputs, multiplies each by a **weight**, sums them, adds a **bias**,
and passes the result through a **step activation function** to produce a binary output.

$$z = w \cdot x + b, \qquad \hat{y} = \begin{cases} 1 & z \ge 0 \\ 0 & z < 0 \end{cases}$$

- **`x`** — the input features (a vector).
- **`w`** — one learned weight per input feature; controls how much that feature matters.
- **`b`** — the bias; shifts the decision boundary independent of the inputs.
- **step function** — the activation. It's a hard threshold: no output between 0 and 1.

That's the entire model. No hidden layers, no fancy activation — just a weighted sum and a threshold.

### Perceptron vs biological neuron

The perceptron (Rosenblatt, 1958) was explicitly inspired by the biological neuron, and the analogy is
useful up to a point:

| Biological neuron | Perceptron |
|---|---|
| Dendrites receive signals from other neurons | Inputs `x` |
| Synapse strength (how strongly a signal is passed on) | Weights `w` |
| Cell body sums incoming signals | Weighted sum `w · x + b` |
| Neuron "fires" only past a threshold | Step activation |
| Axon carries the output signal onward | Output `ŷ` |

But don't over-extend it — **a perceptron is a crude mathematical caricature, not a model of a real
neuron.** Real neurons are electrochemical, fire in continuous time, have thousands of synapses with
plastic (constantly changing) strengths, and are governed by biology far too complex for a dot product
and a threshold to capture. The useful takeaway is narrower: *a single perceptron is loosely
analogous to one neuron with a step-function response* — nothing more. "Neural network" is a
metaphor the field kept because it's evocative, not because the model is biologically accurate.

---

## 2. Geometric Intuition

This is the idea that matters most: **a perceptron is a linear binary classifier.**

Setting the pre-activation to zero, `w · x + b = 0`, defines:
- a **line** in 2D (two input features),
- a **plane** in 3D,
- a **hyperplane** in general, for `n` input features.

This boundary splits the input space into exactly two regions. Every point on one side gets
`z ≥ 0 → ŷ = 1`; every point on the other side gets `z < 0 → ŷ = 0`. The weights `w` set the
orientation of this boundary (which way it tilts), and the bias `b` sets its offset from the origin
(how far it's shifted).

This is why a perceptron can only ever solve problems where the two classes can be separated by a
straight line/plane/hyperplane — it has no way to draw a curved or disjoint boundary. That single
fact is the thread running through the rest of this section.

---

## 3. The Perceptron Trick (Training)

How do you find the `w` and `b` that place the line correctly, without calculus? Rosenblatt's original
answer — the **perceptron trick** — is a simple iterative rule:

1. Start with random (or zero) weights.
2. Go through the training points one at a time (or in random order, repeatedly).
3. If a point is **correctly classified**, do nothing.
4. If a point is **misclassified**, nudge the weights so the boundary moves *toward* correctly
   classifying it:

$$w \leftarrow w + \eta \,(y - \hat{y})\, x, \qquad b \leftarrow b + \eta\,(y - \hat{y})$$

- `y` is the true label, `ŷ` the predicted label, `η` (eta) the **learning rate**.
- If `y = 1` but `ŷ = 0` (predicted too low), `(y - ŷ) = 1` — the update adds a bit of `x` to `w`,
  rotating/shifting the line toward this point so it lands on the positive side.
- If `y = 0` but `ŷ = 1` (predicted too high), `(y - ŷ) = -1` — the update subtracts `x` from `w`,
  pushing the line away from this point.
- Correct predictions give `(y - ŷ) = 0` — no update, the line doesn't move.

Intuition: every misclassified point acts like a small tug on the decision boundary, rotating and
sliding it a little closer to putting that point on the correct side. Repeat this over the dataset
enough times and the line settles into a position that separates the classes.

**Convergence guarantee:** the **perceptron convergence theorem** guarantees this process finds a
separating line in a finite number of steps — *but only if the data is linearly separable.* If no
straight line can separate the classes, the trick never converges; the weights keep getting tugged
back and forth forever.

---

## 4. The Perceptron Loss Function

The "trick" above works, but it's ad hoc — there's no explicit quantity being minimized. Reframe the
same problem properly: define a **loss function** that measures how wrong the current line is, then
minimize it with **gradient descent** (from the ML guide). This reframing is what connects the
perceptron to the rest of deep learning, where *everything* is trained by defining a loss and
descending its gradient.

The classic **perceptron loss** only penalizes misclassified points, and does so proportionally to
how confidently wrong they are:

$$L = \sum_{i \in \text{misclassified}} \max(0, -y_i \cdot z_i)$$

(using labels `y ∈ {-1, +1}` here, which is the convenient convention for this loss). Correctly
classified points contribute zero loss; misclassified points contribute a penalty that grows with
`-y·z`. This `max(0, ...)` shape is exactly the **hinge loss** used in a linear SVM — the perceptron
loss is a hinge loss without the SVM's margin term.

The real insight of this video: **the choice of activation + loss function is what determines which
linear model you get.** The perceptron is not a fixed, separate algorithm from logistic regression or
linear SVMs — they're all the *same* linear architecture (`w · x + b`), differing only in the
activation and loss plugged into it:

| Activation | Loss | Model |
|---|---|---|
| Step function | Perceptron loss (hinge-like, `max(0, -y·z)`) | **Perceptron** |
| Sigmoid | Binary cross-entropy | **Logistic regression** |
| None (linear) | Hinge loss + margin term | **Linear SVM** |

Swapping the step function for a **sigmoid** turns the hard 0/1 output into a smooth probability
between 0 and 1, and swapping the perceptron loss for **binary cross-entropy** gives you a loss that's
differentiable everywhere (the step function's loss isn't, which is why the original perceptron trick
needed its own special-case update rule instead of plain gradient descent). Do both, and gradient
descent on this "single neuron" reproduces logistic regression exactly. This unification is the bridge
from Rosenblatt's 1958 trick to the general, differentiable, gradient-descent-trained neurons that make
up every layer of every network in the rest of this guide.

---

## 5. Problem with the Perceptron

A perceptron is fundamentally a **linear classifier** — it draws exactly one straight line/plane/
hyperplane. So it can only correctly classify data that is **linearly separable**. It has no
mechanism for a curved, circular, or otherwise non-linear boundary.

The canonical example of what it *cannot* do is **XOR**:

| x1 | x2 | XOR |
|---|---|---|
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

Plot these four points: the two `1` outputs and the two `0` outputs are arranged diagonally opposite
each other. No single straight line can separate them — any line you draw puts at least one point on
the wrong side. Since a perceptron *is* a single line, it cannot learn XOR, no matter how it's
trained or for how long.

This is not a training bug — it's a hard structural limit of the model. As covered in Section 01's
history, **Minsky & Papert's 1969 book proved this limitation formally**, and the fact that a
perceptron can't even learn a function as simple as XOR contributed directly to funding drying up
and the first "AI winter."

The fix, also anticipated (but not solved efficiently) at the time, is to stop using a *single*
perceptron and instead **stack several of them into layers** — a Multi-Layer Perceptron. Composing
multiple linear boundaries lets the network carve out non-linear decision regions (an MLP with a
hidden layer can, in fact, learn XOR). That architecture, and how a signal actually flows through it,
is Section 03.

---

## Key takeaways

- A perceptron computes `z = w · x + b` and thresholds it with a step function: `ŷ = 1` if `z ≥ 0`
  else `0`. It's loosely modeled on a biological neuron, but far cruder — one weighted sum and a
  threshold, not a full model of neural biology.
- Geometrically, `w · x + b = 0` is a line/plane/hyperplane — **a perceptron is a linear binary
  classifier**, nothing more.
- The **perceptron trick** trains by nudging weights toward misclassified points each round:
  `w ← w + η(y − ŷ)x`. It converges only if the data is **linearly separable**.
- Reframed properly, this is gradient descent on a **loss function** (hinge-like perceptron loss).
  Swapping activation + loss (step/perceptron-loss vs. sigmoid/binary-cross-entropy) turns the same
  linear architecture into a perceptron, logistic regression, or (with a margin term) an SVM.
- **A single perceptron can't learn XOR** or any non-linearly-separable pattern — a structural limit,
  not a training failure. This is what caused the first AI winter, and what motivates stacking
  perceptrons into a Multi-Layer Perceptron.

**Next:** [03 — MLP & Forward Propagation](03-mlp-forward-propagation.md) — stacking perceptrons into
layers to escape the linear-separability limit.
