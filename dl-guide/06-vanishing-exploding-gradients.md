# 06 — Vanishing and Exploding Gradient Problems

**Playlist video 18: "Vanishing Gradient Problem in ANN | Exploding Gradient Problem | Code
Example."** One video, but a pivotal one — this is the failure mode that motivates almost every
"trick" in Sections 08–10, and it resurfaces in a sequence-specific form when RNNs show up in
Section 16–17. Understand this section well before moving on.

---

## 1. Where the problem comes from

Recall backpropagation from Section 05: to update an early (close-to-input) layer's weights,
gradient descent needs `∂Loss/∂w` for that layer. The chain rule computes this gradient as a
**product of many terms**, one for each layer the error signal has to flow back through — layer
derivatives, activation derivatives, and weights, multiplied together all the way from the output
layer back to that early layer.

```
∂Loss/∂w(layer 1)  =  (a chain of many partial derivatives, one per layer, multiplied together)
```

The deeper the network, the longer this product. And that's the entire problem in one sentence:

> **Backprop propagates gradients backward by multiplying many numbers together. If those numbers
> are consistently less than 1, the product shrinks exponentially toward zero as it travels back.
> If they're consistently greater than 1, the product grows exponentially.**

This isn't a bug in the math — it's an unavoidable consequence of depth combined with multiplication.
It's the same shape as compound interest, just running in the wrong direction for the person who
wants a *non-zero* gradient at layer 1.

### The vanishing case

If each layer's gradient contribution is, say, 0.4, then after 10 layers the accumulated factor is
roughly `0.4^10 ≈ 0.0001`. Whatever gradient arrived at the output layer gets multiplied by that
before it reaches layer 1. The result: **the update to layer 1's weights is essentially zero**, no
matter how large the original error was.

### The exploding case

Symmetrically, if each layer's factor is, say, 2, then after 10 layers the accumulated factor is
`2^10 = 1024`. A perfectly reasonable-sized error signal at the output becomes a massive, unstable
weight update once it reaches the early layers.

Both are a **depth problem**: the deeper the network, the more multiplications happen, the more
extreme (toward 0 or toward ∞) the product becomes. Shallow networks barely notice this; it's why
the problem became a real obstacle only once people started stacking many layers.

---

## 2. Why sigmoid/tanh make it worse

The activation function's derivative is one of the terms in that chain-rule product every single
layer, so it matters enormously which activation you pick (this is exactly the motivation for
Section 08).

- **Sigmoid:** `σ(z) = 1 / (1 + e^-z)`. Its derivative is `σ(z) · (1 − σ(z))`, which has a **maximum
  value of 0.25** (at z = 0) and **saturates toward 0** for large positive or negative z (the
  classic flat tails of the sigmoid curve). So every layer using sigmoid contributes a factor that
  is *at best* 0.25, and often much smaller once neurons saturate.
- **Tanh:** better than sigmoid (max derivative 1.0, zero-centered), but it still saturates to
  derivative ≈ 0 for large |z|, so the same problem shows up, just less severely.

Chain a handful of sigmoid layers together and you're multiplying numbers that are at most 0.25:
`0.25^5 ≈ 0.001`, `0.25^10 ≈ 0.000001`. This is precisely why deep sigmoid networks were notoriously
hard to train in the pre-2010s era, and why the vanishing gradient problem — not lack of compute —
was the real reason "deep" learning didn't take off until activations and initialization were fixed.

Exploding gradients are less inherently tied to a specific activation; they show up more from **large
weight values** or **poor initialization** producing pre-activations that keep growing layer over
layer (more on this in Section 09), and they're especially common in recurrent networks where the
"depth" is effectively the sequence length (Section 16).

---

## 3. Symptoms — how you actually notice this while training

- **Vanishing:**
  - Early (input-side) layers' weights barely move between epochs — you can literally check this by
    logging weight norms or gradient magnitudes layer by layer.
  - Training loss plateaus early and refuses to improve, even though the model clearly has capacity.
  - Deeper networks perform *worse* than shallower ones on the same problem — a red flag, since more
    layers should only help or stay flat, not hurt.
- **Exploding:**
  - Loss suddenly spikes to `NaN` or `inf`.
  - Loss oscillates wildly instead of decreasing smoothly.
  - Weight values (or gradient norms, if you log them) grow to enormous magnitudes over a few steps.

A simple diagnostic: print or plot the average gradient magnitude per layer after a backward pass.
A healthy network has roughly comparable magnitudes across layers; a vanishing-gradient network
shows magnitudes collapsing toward the input layers, and an exploding one shows them blowing up.

```python
# Sketch of the diagnostic, not a full training loop
for layer in model.layers:
    grads = layer.trainable_weights[0]  # weight gradients for this layer
    print(layer.name, tf.reduce_mean(tf.abs(grads)).numpy())
```

---

## 4. Solutions (each gets its own full section later)

There is no single fix — the field's answer is a *combination* of techniques, each attacking a
different part of the chain-rule product:

- **Better activation functions — Section 08.** Swapping sigmoid/tanh for **ReLU** (and variants
  like Leaky ReLU) removes the saturation problem: ReLU's derivative is exactly 1 for any positive
  input, so it doesn't shrink the gradient at all in that region. This is the single biggest lever.
- **Proper weight initialization — Section 09.** **Xavier/Glorot** and **He initialization** choose
  the initial weight scale specifically so that the variance of activations (and gradients) stays
  roughly constant as they pass through layers, instead of shrinking or growing layer by layer.
  Combined with the activation choice, this addresses the *root cause* rather than a symptom.
- **Batch normalization — Section 10.** Normalizes each layer's inputs to have stable mean/variance
  during training, which keeps pre-activations in a well-behaved range and prevents them from
  drifting into saturation regions or blowing up — indirectly stabilizing gradients throughout the
  network.
- **Gradient clipping.** A direct countermeasure for *exploding* gradients specifically: after
  computing gradients, if their norm exceeds some threshold, rescale them down before the weight
  update. Doesn't fix vanishing gradients at all — it's a targeted fix for the other side.
- **Residual/skip connections, and gated recurrent units — Section 17.** Skip connections give the
  gradient a shorter path back to early layers (an "highway" that bypasses some of the
  multiplication). For sequence models specifically, **LSTM** and **GRU** replace the RNN's plain
  recurrent multiplication with gated cell state updates designed explicitly to let gradients flow
  back across many time steps without vanishing — the RNN version of this exact same problem,
  covered when Section 16 sets it up and Section 17 solves it.

None of these techniques are exclusive — modern deep networks routinely combine ReLU-family
activations, He initialization, and batch norm by default, precisely because each one chips away at
a different multiplicative culprit in the chain rule.

---

## Key takeaways

- Backprop computes early-layer gradients as a **product of many per-layer terms** (chain rule).
  Consistently sub-1 terms → **exponential shrinkage** (vanishing gradients); consistently >1 terms
  → **exponential growth** (exploding gradients). Both are consequences of *depth*, not bugs.
- **Sigmoid/tanh are especially prone to vanishing** because their derivatives are small (sigmoid
  maxes out at 0.25) and saturate toward 0 for large |input| — multiplying several of these together
  crushes the gradient long before it reaches early layers.
- **Symptoms:** vanishing → early layers don't learn, loss plateaus; exploding → loss becomes
  `NaN`/oscillates wildly. Diagnose by inspecting per-layer gradient magnitudes.
- **Fixes, each covered in depth ahead:** better activations like ReLU (Section 08), Xavier/He
  weight initialization (Section 09), batch normalization (Section 10), gradient clipping
  (exploding only), and skip connections / LSTM-GRU gating for sequence models (Section 17).

**Next:** [07 — Improving Neural Networks](07-improving-neural-networks.md) — gradient descent
variants and the broader toolkit (early stopping, feature scaling, dropout, L1/L2) for making
training actually work well.
