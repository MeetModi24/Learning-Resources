# 14 — Backpropagation in CNNs

**Playlist videos 47–48.** Section 13 covered the forward pass of a CNN: convolution, padding/strides,
pooling, and how these stack into something like LeNet. This section closes the loop the same way
Section 05 did for plain ANNs — **how does a CNN learn?** The headline answer is: it doesn't need a
new learning algorithm. Forward pass → loss → gradients via the chain rule → gradient descent update
(Section 05, Section 07) is exactly the same machinery. The *only* new work is figuring out how a
gradient flows *backward* through three layer types that don't exist in a plain ANN: **flatten**,
**max pooling**, and **convolution**. Once a gradient reaches the filter weights, they get updated by
the optimizer (Section 11) exactly like any dense-layer weight.

The three layers are covered in the order that makes backprop easiest to build intuition for —
easiest first: flatten (no computation at all), then max pooling (routing, no parameters), then
convolution (the real computation, with parameters to update).

---

## 1. The big picture: nothing new, just new plumbing

A typical CNN is `[conv → pool] × N → flatten → dense layers → output`. Forward propagation pushes
data through every layer left to right; backpropagation pushes the **gradient of the loss**
backward through the exact same layers, right to left, one layer at a time — precisely the chain-rule
relay from Section 05 ("each layer receives an error signal from the layer after it... and passes a
new error signal further back").

The dense layers at the end of the network already know how to do this — that's ordinary ANN
backprop. What's new here is: **when the incoming gradient reaches the flatten layer, then a pooling
layer, then a convolution layer, what does that layer do with it, and does it have anything to
update?**

| Layer | Learnable parameters? | What backward pass does |
|---|---|---|
| Flatten | None | Reshape the gradient back to the feature-map shape it came from. |
| Max Pooling | None | Route the gradient to only the position that was the max; zero everywhere else. |
| Convolution | Filter weights (+ bias) | Compute gradient w.r.t. the filters (to update them) *and* gradient w.r.t. the input (to keep propagating backward) — both are themselves convolution operations. |

Two of the three rows have **no parameters to learn** — pooling and flatten only *route* gradient,
they never generate a weight update. Only the convolution layer has learnable weights, so it's the
only one of the three that the optimizer actually touches.

---

## 2. Flatten layer — pure reshape

**Forward:** takes the final stack of feature maps (shape `height × width × channels`) and reshapes
it into a single 1D vector, so it can feed into the dense layers.

**Backward:** the dense layers hand back a gradient vector of that same 1D length (via ordinary ANN
backprop). Flatten's job is to reshape that vector back into the `height × width × channels` shape
it originally came from — nothing more.

There is no math here beyond bookkeeping: flatten has no weights, performs no arithmetic on the
values, and its "backward pass" is just the inverse of its forward reshape. It exists purely so
gradient computed in vector-shaped dense-layer land lands back in the right spatial position to
continue flowing into the pooling/conv layers behind it.

---

## 3. Max Pooling layer — route the gradient to the winner

**Forward (recap from Section 13):** slide a window (e.g. 2×2) over the feature map and keep only the
maximum value in each window, discarding the rest.

**Backward:** max pooling has no weights, so there's nothing to update — the only question is how to
pass the incoming gradient further back to the layer before it. The rule:

> **The gradient for a pooling window goes entirely to the single input position that was the max
> during the forward pass ("the argmax"). Every other position in that window gets a gradient of
> zero.**

### Why this is correct, not just convenient

Only the max value actually flowed forward into the output — the other values in the window were
looked at and discarded. If you nudge any of the *non-max* values up or down slightly, the pooling
output doesn't change at all (as long as it's still not the biggest), so its effect on the loss is
zero — that's exactly what a zero gradient means. If you nudge the max value, the output moves in
lockstep, so all of the downstream gradient belongs to it.

Concretely, for a 2×2 window `[[1, 5], [2, 3]]` that maxed out at `5`, an incoming gradient of `0.8`
backpropagates as:

```
[[0, 0.8],
 [0,   0]]
```

The `5`'s position gets the full `0.8`; the `1`, `2`, and `3` positions get `0` — they had no
influence on the output, so they get no blame (or credit) for the loss.

**Average pooling, by contrast** (mentioned for comparison; not the focus of the playlist videos)
does contribute every position to its output equally, so its backward pass **distributes the
incoming gradient equally across every position in the window** instead of concentrating it on one.
That difference — winner-takes-all vs. equal split — is a direct mirror of the forward computation
each pooling type performs.

Max pooling backprop, then, is pure **routing**: remember which index won each window during the
forward pass (this is exactly the kind of intermediate the memoization idea from Section 05 says to
cache), and send the gradient there. No multiplication, no new computation.

---

## 4. Convolution layer — the layer that actually learns

This is the one layer of the three with learnable parameters (the filter/kernel weights, plus a
bias), so it's the one the optimizer needs a real gradient for. Two separate gradients are needed
here, for two different purposes:

1. **Gradient w.r.t. the filter weights** — needed to *update* the filter (this is the whole point:
   the filter is a parameter, like any dense-layer weight, and gradient descent needs
   $\partial L/\partial(\text{filter})$ to nudge it).
2. **Gradient w.r.t. the layer's input** — needed to keep the chain-rule relay going *further*
   backward, into whatever layer feeds this convolution (an earlier conv layer, or the input image).

The elegant (and somewhat surprising) result from working out the chain rule over a convolution's
sliding-window structure is that **both of these gradients are themselves convolution operations**:

- **Gradient w.r.t. the filter** = convolve the layer's **input** with the **incoming output
  gradient** (using the incoming gradient in place of a filter, slid over the original input).
  Intuitively: a filter weight's gradient asks "how would the loss change if I nudged this weight?",
  and the weight only ever multiplied against the input values it slid over — so this convolution is
  just accumulating that relationship across every position the filter touched.

- **Gradient w.r.t. the input** = a **"full" convolution** of the incoming output gradient with the
  filter **flipped (rotated 180°)**. Intuitively: to push the gradient back to *where it came from* in
  the input, you have to reverse the sliding-window operation the forward pass did — and reversing a
  convolution's window-and-multiply pattern is itself a convolution, just with the filter flipped
  and the boundary handling padded out ("full" convolution) to cover every input position that
  contributed to *some* output.

You don't need to grind through the index algebra to use this — the important shape to keep is:
**convolution forward, convolution backward too**, just with the roles of "input", "filter", and
"gradient" rearranged. This is also exactly why frameworks like Keras/TensorFlow implement conv-layer
backprop as a specialized (and heavily GPU-optimized) convolution kernel rather than a generic
matrix-multiply — it's structurally the same operation as the forward pass.

### Parameter sharing means gradients accumulate

Recall from Section 13 that a convolution's whole efficiency advantage over a dense layer is
**parameter sharing** — the same small filter slides across every spatial position, reusing the same
weights everywhere instead of learning a separate weight per position. That sharing has a direct
consequence for backprop:

> **Each filter weight contributed to the output at every position the filter slid over — so its
> gradient is the *sum* of its contribution to the loss at every one of those positions.**

A dense-layer weight touches exactly one connection, so its gradient is a single term. A
shared conv filter weight touches (say) every one of a 28×28 output map's positions, so its gradient
is a sum over all 28×28 of those local contributions before it's used to update the weight. This is
the same accumulate-and-reuse spirit as the memoized error signals in Section 05 — nothing about the
chain rule changes, there's just more terms landing on the same shared weight because that weight was
*reused* more.

### Then: business as usual

Once $\partial L/\partial(\text{filter weights})$ has been computed this way, there's nothing
CNN-specific left to do — the optimizer (Section 11: SGD, momentum, Adam, whichever is configured)
consumes that gradient and updates the filter exactly as it would update any dense-layer weight:
$w \leftarrow w - \eta \cdot \partial L/\partial w$ (or the optimizer's fancier variant of that
update rule).

---

## Putting it together

A full backward pass through a CNN, right to left:

1. **Dense layers** at the top: ordinary ANN backprop (Section 05), producing a gradient vector
   for the flattened feature vector.
2. **Flatten**: reshape that vector back into the feature-map shape — no computation.
3. **Max pooling**: route the gradient to the position that was the max in each window; zero
   elsewhere. Average pooling would split it equally instead.
4. **Convolution**: compute the gradient w.r.t. the filter (a convolution of input × incoming
   gradient, summed across every shared position the filter touched) to hand to the optimizer, *and*
   the gradient w.r.t. the input (a full convolution of the incoming gradient with the flipped
   filter) to keep propagating backward into the previous layer.
5. Repeat step 3–4 for every `conv → pool` block, until the gradient reaches the input image, where
   backprop stops (the input isn't a learnable parameter).

Same four-step training loop as Section 05 (forward → loss → backward → update) — CNNs just add two
new kinds of "backward" behavior (routing, and convolution-shaped gradients) to the relay.

---

## Key takeaways

- CNN training uses **the same backprop + gradient descent machinery as any ANN** (Section 05); the
  only new work is how the gradient flows backward through conv, pooling, and flatten layers.
- **Flatten** has no parameters and does no computation in reverse — it just reshapes the incoming
  gradient back into the feature-map shape.
- **Max pooling** has no parameters; its backward pass routes the full gradient to the position that
  was the max in the forward pass and zeroes every other position in the window (average pooling
  would split the gradient equally instead) — because only the max value actually influenced the
  output.
- **Convolution** is the only one of the three with learnable parameters (filters + bias), and both
  gradients it needs are themselves convolutions: gradient w.r.t. the filter = input convolved with
  the incoming gradient; gradient w.r.t. the input = incoming gradient convolved with the
  **flipped** filter (a "full" convolution).
- **Parameter sharing cuts both ways**: it's what makes conv layers efficient in the forward pass,
  and it's why a filter weight's gradient is a **sum over every spatial position** the filter was
  applied to during the forward pass.
- Once the filter gradients are computed, the optimizer (Section 11) updates them exactly like any
  other weight — there is no CNN-specific update rule, only a CNN-specific way of *getting* the
  gradient there.

**Next:** [15 — CNNs in Practice](15-cnns-in-practice.md) — projects, data augmentation, pretrained
models, transfer learning, and visualizing what a trained CNN actually learned.
