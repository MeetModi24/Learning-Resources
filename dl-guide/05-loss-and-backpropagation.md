# 05 — Loss Functions & Backpropagation

**Playlist videos 14–17, 19.** This is the heart of the guide: how a neural network actually
*learns*. Section 03 covered forward propagation (input → prediction) and Section 04 covered
building networks in Keras, but neither explained how the weights get *good*. This section closes
that gap: define a loss, compute its gradient with respect to every weight via backpropagation, and
(next section, then Section 07) use that gradient to update the weights. Video 18 (vanishing
gradients — a *problem* with backprop) is deliberately deferred to Section 06 so this section can
stay focused on the mechanism itself.

---

## 1. Loss Functions

### Loss vs. Cost

- **Loss** = how wrong the network's prediction is **for one training example**.
- **Cost** = the **average loss over the whole dataset (or a batch)** — this is the quantity
  gradient descent actually minimizes.

The words are often used interchangeably in casual conversation, but keep the distinction in mind:
backprop computes gradients of the loss for one example (or a batch, in practice), and training
minimizes the cost (the aggregate) over many iterations.

Training, end to end, is: **pick a loss function that matches the problem type → compute the cost
on the current predictions → find the gradient of that cost with respect to every weight
(backpropagation) → nudge the weights against the gradient (gradient descent) → repeat.** Choosing
the *right* loss function isn't cosmetic — the wrong choice gives you either a poorly-shaped
optimization surface or a metric that doesn't reflect what you actually care about.

### Regression losses

| Loss | Formula (per example) | Behavior | Use when |
|---|---|---|---|
| **MSE** (Mean Squared Error) | $(y - \hat{y})^2$ | Squares the error → penalizes large errors heavily, small errors gently. Smooth, differentiable everywhere — nice gradients. | Default choice for regression. Sensitive to outliers (a few huge errors dominate the loss). |
| **MAE** (Mean Absolute Error) | $\lvert y - \hat{y} \rvert$ | Penalizes all errors linearly. Robust to outliers — one huge error doesn't dominate. | Data has outliers you don't want to overweight. Downside: gradient has constant magnitude (±1) even near the minimum, so convergence can be less smooth. |
| **Huber** | Quadratic for small errors, linear for large errors (a threshold $\delta$ switches between the two) | Best of both: smooth near zero like MSE, robust to outliers like MAE. | You want MSE's smooth gradient behavior *and* outlier robustness — costs you one extra hyperparameter ($\delta$). |

### Binary classification: Binary Cross-Entropy (log loss)

Pairs with a **sigmoid** output neuron (Section 04's output-layer table: sigmoid activation + BCE
loss for binary classification). For a true label $y \in \{0, 1\}$ and predicted probability
$\hat{y}$:

$$L = -\big[y \log(\hat{y}) + (1-y)\log(1-\hat{y})\big]$$

Intuition: if $y=1$, the loss is $-\log(\hat{y})$ — it's near zero when $\hat{y}$ is close to 1
(confident and correct) and blows up toward infinity as $\hat{y} \to 0$ (confident and *wrong* gets
punished hard). Symmetric logic for $y=0$. This asymmetric, unbounded penalty for confident wrong
answers is exactly what you want from a probability-estimation loss — and it's what MSE doesn't
give you (more below).

### Multiclass classification: Categorical Cross-Entropy

Pairs with a **softmax** output layer (one probability per class, summing to 1). For $K$ classes:

$$L = -\sum_{k=1}^{K} y_k \log(\hat{y}_k)$$

Because $y$ is one-hot (1 for the true class, 0 elsewhere), every term except the true class's
vanishes — the loss reduces to $-\log(\hat{y}_{\text{true class}})$: how much probability mass the
model put on the right answer.

**One-hot vs. sparse categorical cross-entropy** — same math, different label encoding:
- `categorical_crossentropy` expects one-hot labels: `[0, 0, 1, 0]`.
- `sparse_categorical_crossentropy` expects integer labels: `2`.

Use sparse when you have many classes — no need to materialize a giant one-hot matrix just to throw
most of it away in the loss computation. Keras implements both; pick based on how your labels are
already encoded.

### Why cross-entropy, not MSE, for classification

This is a real design decision, not a convention to memorize. Two independent reasons:

1. **Loss surface shape.** Cross-entropy paired with sigmoid/softmax gives a loss surface that is
   (for logistic regression, and empirically well-behaved for networks) smooth and closer to
   convex, so gradient descent converges reliably. MSE on probabilities creates a bumpier,
   harder-to-optimize surface with more spurious local minima.
2. **Gradient magnitude — the real killer.** Recall the sigmoid's derivative is
   $\sigma(z)(1-\sigma(z))$, which is tiny when $\sigma(z)$ is near 0 or 1 (the sigmoid saturates at
   its extremes — see Section 06). If you use **MSE**, the gradient of the loss with respect to the
   pre-activation $z$ contains that sigmoid-derivative factor directly, so a *confidently wrong*
   prediction (output near 0 when it should be 1) produces a **near-zero gradient** — the network
   barely learns from the mistake that matters most. **Cross-entropy's gradient with sigmoid/softmax
   cancels that derivative term algebraically**, leaving a gradient proportional simply to
   $(\hat{y} - y)$ — large when the prediction is confidently wrong, small when it's right. That's
   the gradient shape you want: big correction signal exactly when you need it.

This is why the output-layer pairing table from Section 04 (sigmoid→BCE, softmax→CCE, linear→MSE)
isn't arbitrary — each pairing produces well-behaved gradients precisely because of this
cancellation.

---

## 2. Backpropagation Part 1 — The What

**Backpropagation is the algorithm that computes the gradient of the loss with respect to every
weight and bias in the network**, so that gradient descent (Section 07) has something to step
against. That's it — backprop doesn't update weights itself; it just computes
$\partial L / \partial w$ for every $w$ (and $\partial L / \partial b$ for every $b$) efficiently.

This slots into the full training loop:

1. **Forward pass** — feed the input through the network (Section 03) to get a prediction $\hat{y}$.
2. **Compute loss** — compare $\hat{y}$ to the true label $y$ using the appropriate loss function
   (Section 1 above).
3. **Backward pass (backpropagation)** — propagate the error *backward* from the output layer to
   the input layer, computing $\partial L/\partial w$ and $\partial L/\partial b$ for every
   parameter along the way.
4. **Update weights** — gradient descent nudges every weight in the direction that reduces the
   loss: $w \leftarrow w - \eta \cdot \partial L/\partial w$.
5. **Repeat** for the next example/batch, for many epochs.

Without backprop, step 3 has no efficient way to happen — and without step 3, step 4 has nothing to
act on. Backprop is the piece that makes a *multi-layer* network trainable at all (this is exactly
why 1986's popularization of backprop — Section 01's history — revived the field: perceptrons could
be trained by inspection, but nobody had an efficient way to train deep, multi-layer networks until
backprop).

---

## 3. Backpropagation Part 2 — The How

The mechanism is the **chain rule of calculus**, applied layer by layer, moving *backward* from the
output.

### The core idea

Every weight in the network affects the final loss only *indirectly* — through a chain of
intermediate computations (the weight affects a pre-activation, which affects an activation, which
feeds the next layer, ..., which eventually determines the output and hence the loss). The chain
rule says: to get the derivative of the loss with respect to a weight buried deep in the network,
**multiply the local derivatives along the path from that weight to the loss.**

Concretely, for a weight $w$ feeding pre-activation $z$, activation $a = f(z)$, contributing to loss
$L$:

$$\frac{\partial L}{\partial w} = \frac{\partial L}{\partial a} \cdot \frac{\partial a}{\partial z} \cdot \frac{\partial z}{\partial w}$$

Each factor is a **local, easy-to-compute derivative**:
- $\partial z/\partial w$ — just the input feeding that weight (from the forward pass).
- $\partial a/\partial z$ — the derivative of the activation function at that point (e.g. the
  sigmoid derivative $a(1-a)$).
- $\partial L/\partial a$ — how much the loss changes as this neuron's output changes; for the
  *output* layer this comes straight from the loss formula (and, as shown above, often has a clean
  closed form like $\hat{y}-y$); for a *hidden* layer it has to be **received from the layer
  after it**, because a hidden neuron's effect on the loss only exists via the layers downstream of
  it.

### Walking through it conceptually

Think of a small network: input → hidden layer → output layer → loss.

1. **At the output layer**, the gradient is direct: how far off is the prediction, and how does
   the output activation respond to changes in its pre-activation? This gives
   $\partial L/\partial z_{\text{output}}$ directly — call this "the error signal at the output."
2. **Propagate that error signal backward one layer.** Each hidden neuron contributed to *every*
   output neuron it's connected to (through its outgoing weights), so its share of the blame is a
   **weighted sum** of the output layer's error signal, weighted by the connecting weights — then
   passed through that hidden neuron's own activation derivative.
3. **Once a layer has its error signal**, the gradient for *any weight feeding into that layer* is
   just: (error signal at this layer) × (the activation value flowing into that weight, from the
   forward pass). This is why the forward pass has to happen — and be remembered — before backprop
   can run.
4. Repeat, moving one layer backward at a time, until every layer (all the way to the input) has
   had its weights' gradients computed.

The name "backpropagation" is literally this: the **error signal propagates backward** through the
network, layer by layer, each layer computing its own local gradients and passing a summarized
error signal to the layer before it.

**Intuition to hold onto:** the gradient at a weight tells you *how much that weight contributed to
the final error*, and in *which direction* increasing it would make the error worse or better.
Gradient descent then moves every weight a small step in the direction that reduces the loss —
"downhill" on the loss surface.

---

## 4. Backpropagation Part 3 — The Why

Why compute gradients this specific way — backward, layer by layer — instead of some other way?
**Efficiency.** This is the entire justification, and it's a bigger deal than it sounds.

### The naive alternative is intractable

If you tried to compute $\partial L/\partial w$ independently for every single weight — treating
each as its own from-scratch calculus problem — you would recompute the same intermediate
quantities over and over. A weight deep in the network shares almost its entire dependency chain
(every downstream activation, every downstream weight) with every *other* weight in the same or
earlier layers. Naively recomputing each gradient from scratch means recomputing those shared
sub-expressions exponentially many times as the network gets deeper — training would be far too
slow to be practical for any real network.

### Backprop is dynamic programming

Backpropagation avoids this by recognizing that the chain-rule factors **overlap massively** across
weights, and computing each shared piece **exactly once**, then reusing it everywhere it's needed.
This is precisely the dynamic-programming idea: solve overlapping subproblems once, cache the
result, reuse it. The "error signal" passed backward from one layer to the previous one (Part 2,
step 2) *is* that cached, reusable quantity — every weight feeding into a layer reuses the same
error signal computed for that layer, instead of each weight re-deriving it independently.

This is why backpropagation runs in time roughly proportional to the **size of the network**
(one forward pass + one backward pass, each touching every weight once) rather than blowing up
combinatorially with depth. That efficiency is *the* reason multi-layer networks are trainable at
all — it's what made the 1986 backprop paper (Section 01) a turning point rather than a curiosity.

---

## 5. MLP Memoization

"Memoization" is the concrete mechanism behind the efficiency argument above, made precise for a
multi-layer perceptron:

- **During the forward pass**, every pre-activation $z$ and activation $a$ at every layer gets
  **computed once and stored** (not thrown away).
- **During the backward pass**, computing the gradient at any layer needs: (a) the error signal
  handed down from the layer after it, and (b) the activation values flowing into that layer's
  weights — **both of which were already computed and cached**, either during the forward pass
  (the activations) or earlier in this same backward pass (the error signal from the next layer
  out). Nothing has to be recomputed from scratch.
- Each weight's gradient is then a cheap local computation: cached error signal × cached
  activation. Multiply once, done.

Without this caching, computing the gradient for a weight in layer 2 of a 5-layer network would
require re-running large chunks of the forward pass just to reconstruct values that were already
available. With it, each stored value is computed once and read as many times as needed. This is
exactly the "why" from Part 3, turned into an implementation detail: **store forward-pass
intermediates, reuse them (and the backward-pass error signals) instead of recomputing**, and the
whole gradient computation for a network with $n$ weights costs roughly $O(n)$ work — one forward
pass, one backward pass — instead of blowing up with network depth.

---

## Putting it together: the full training loop

Sections 03, 05, and 07 now form one coherent story:

1. **Forward propagation** (Section 03): input → predictions, layer by layer.
2. **Loss function** (this section): quantify how wrong the prediction was, choosing the loss that
   matches the problem type (MSE/MAE/Huber for regression, BCE for binary, CCE for multiclass).
3. **Backpropagation** (this section): use the chain rule, backward through the network, reusing
   cached forward-pass values and backward error signals (memoization), to get
   $\partial L/\partial w$ for every weight efficiently.
4. **Gradient descent** (Section 07): update every weight against its gradient, and repeat the loop
   for many examples and epochs.

This four-step loop — forward, loss, backward, update — is what "training a neural network" *means*
mechanically. Everything else in this guide (better activations, initialization, optimizers,
regularization) is about making this same loop converge faster, more stably, or to a better
solution — not replacing it.

---

## Key takeaways

- **Loss** is per-example error; **cost** is the average loss over the dataset/batch — gradient
  descent minimizes the cost.
- Match the loss to the problem: **MSE/MAE/Huber** for regression (MAE for outlier robustness,
  Huber for a middle ground), **binary cross-entropy** with sigmoid for binary classification,
  **categorical cross-entropy** with softmax for multiclass (sparse vs. one-hot is just a label-
  encoding choice).
- **Cross-entropy beats MSE for classification** because its gradient with sigmoid/softmax cancels
  the saturating activation derivative, giving a large correction signal for confidently-wrong
  predictions instead of a near-zero one.
- **Backpropagation** computes $\partial L/\partial w$ for every weight/bias in the network — it's
  the "how the network learns" step between the forward pass and the weight update.
- Mechanically, it's the **chain rule applied backward**, layer by layer: each layer receives an
  error signal from the layer after it, combines it with its own local derivatives, and passes a
  new error signal further back.
- Backprop is efficient because it's **dynamic programming** — shared sub-expressions (forward-pass
  activations, backward-pass error signals) are computed once and **memoized/reused**, giving
  $O(\text{network size})$ cost instead of exponential blowup.
- The full training loop: **forward pass → loss → backward pass (backprop) → weight update
  (gradient descent) → repeat.**

**Next:** [06 — Vanishing & Exploding Gradients](06-vanishing-exploding-gradients.md) — what goes
wrong with this backward chain-rule product in deep networks, and why it's the direct downside of
the mechanism just described.
