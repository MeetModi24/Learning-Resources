# 13 — CNN Foundations

**Playlist videos 40–46.** This is the first section on a *specialized architecture* rather than a
generic trick for training MLPs better. CNNs (Convolutional Neural Networks) are the architecture of
choice for images and other grid-structured data. We cover why they exist, their biological
inspiration, the convolution operation itself, padding/strides, pooling, the classic LeNet-5
architecture, and a head-to-head comparison with plain ANNs.

---

## 1. Why CNNs? The problem with using an ANN on images

Section 04 built an MNIST classifier with a plain ANN: each 28×28 image was **flattened** into a
784-length vector and fed to a `Dense` layer. That works — MNIST is small and simple — but it hides
two serious problems that blow up on real images:

**Problem 1 — flattening destroys spatial structure.** An image isn't just a bag of independent
numbers. Nearby pixels are correlated: an edge, an eye, a wheel is a *local pattern* spread across a
small neighborhood of pixels. Flattening a 2D grid into a 1D vector throws away the information
about which pixels are next to which — the network has to *rediscover* spatial relationships from
scratch, using weights that have no notion of "neighboring."

**Problem 2 — a fully-connected layer on a real image needs an intractable number of parameters.**
MNIST images are tiny (28×28 = 784 pixels). A real photo might be 1000×1000×3 (RGB) = 3,000,000
input values. A single `Dense` neuron connected to every pixel needs 3 million weights; a hidden
layer of just 1,000 such neurons needs **3 billion weights** — before you've added a second layer.
This is:
- **Computationally intractable** to train.
- **Prone to severe overfitting** — far more parameters than any realistic dataset has examples to
  constrain them (the bias–variance trade-off from `../ml-guide`, in its most extreme form).

**The CNN fix — exploit structure instead of ignoring it.** Images have two properties an ANN never
uses:
1. **Spatial locality** — a pixel is most related to its immediate neighbors, so a feature detector
   only needs to look at a small local window (a *receptive field*), not the whole image at once.
2. **Translation invariance** — a cat's eye looks like a cat's eye whether it's in the top-left or
   bottom-right of the photo. A pattern detector learned in one location should be reusable
   everywhere.

CNNs bake both properties directly into the architecture via **local connectivity** (each unit looks
at a small patch, not the whole image) and **parameter sharing** (the same small set of weights — a
*filter* — is reused at every location). The parameter count for a layer no longer depends on image
size, and it's orders of magnitude smaller than the fully-connected equivalent — which is exactly the
concrete comparison in Section 6.

---

## 2. Biological inspiration: Hubel & Wiesel's cat experiment

The convolution idea isn't just a mathematical convenience — it mirrors how biological vision
actually works, and this connection is part of why CNNs were proposed in the first place.

In 1959, **David Hubel and Torsten Wiesel** ran experiments on a cat's visual cortex, recording
individual neurons' activity while showing the cat patterns of light. Their key findings:

- Neurons in the visual cortex don't respond to the whole visual field — each neuron has a small
  **receptive field**, responding only to stimuli in a specific, localized region.
- Different neurons respond to different simple patterns — e.g., edges at particular orientations.
- The visual system is organized **hierarchically**: "simple cells" detect basic features like edges;
  their outputs feed "complex cells" that combine simple features into more complex patterns
  (corners, motion, shapes); further downstream, cells respond to entire objects.

This is precisely the structure a CNN implements: early layers with small receptive fields learn
simple features (edges, blobs), later layers combine those into increasingly complex, more global
patterns. Hubel and Wiesel won the 1981 Nobel Prize in Physiology or Medicine for this work.

**A brief history of CNNs**, extending the timeline from Section 01:

- **1980 — Fukushima's Neocognitron:** the first neural network directly inspired by Hubel &
  Wiesel's hierarchy, introducing the idea of layered, locally-connected feature detectors.
- **1989/1998 — LeCun's LeNet:** added backpropagation training to the convolutional idea, producing
  the first practical CNN, used commercially to read handwritten digits and checks (Section 5 below).
- **2012 — AlexNet:** scaled the same convolutional idea up with GPUs and much more data, winning
  ImageNet by a huge margin and triggering the modern deep learning boom (Section 01's "why now").

---

## 3. The convolution operation

### The core idea

A **convolution** slides a small matrix of learnable weights — the **filter** (or **kernel**) —
across the input image, and at every position computes a single number: the dot product (element-wise
multiply-then-sum) between the filter and the patch of the image it currently covers. The collection
of all these output numbers forms a **feature map**: a 2D grid where high values mark locations where
the pattern the filter has learned to detect is present.

Three ideas make this powerful:

1. **The filter's weights are learned**, exactly like any other network weights, via backpropagation
   (the mechanics of backprop through a conv layer are covered in Section 14). The network decides
   for itself what pattern each filter should detect — no one hand-designs "an edge detector."
2. **Parameter sharing.** The *same* filter (same weights) is applied at every position of the image.
   This directly implements translation invariance — a filter that has learned "vertical edge" fires
   on a vertical edge anywhere in the image — and keeps the parameter count independent of image
   size: a 3×3 filter has 9 weights (+1 bias) no matter how big the input is.
3. **Hierarchy of features.** A single conv layer only detects simple local patterns. Stacking conv
   layers lets later layers combine the feature maps of earlier layers into progressively more
   complex, larger-scale patterns — mirroring Hubel & Wiesel's simple-cell → complex-cell hierarchy:
   early layers learn edges and colors, middle layers learn textures and shapes, deep layers learn
   object parts.

**Multiple filters, multiple feature maps.** A conv layer normally applies *many* filters in parallel
(e.g., 32 filters), each free to learn a different pattern. The output is a stack of feature maps —
this stack depth is called the number of **channels** (analogous to how an input color image already
has 3 channels: R, G, B). Each filter in a conv layer looks across *all* input channels at once, so a
filter applied to an RGB image is really a small 3D block (e.g., 3×3×3), not a flat 2D patch.

### A small numeric example

Take a 4×4 grayscale input and a 3×3 filter that has learned to detect vertical edges:

```
Input (4x4):                Filter (3x3):
1  1  0  0                   1  0 -1
1  1  0  0                   1  0 -1
1  1  0  0                   1  0 -1
1  1  0  0
```

Sliding the filter over the top-left 3×3 patch of the input:

```
Patch:            Element-wise product:      Sum
1 1 0             1  0  0
1 1 0      x       1  0  0     =    1+1+1+1+1+1 = 6
1 1 0               1  0  0
```

That single number, 6, becomes one entry of the output feature map, at the position corresponding to
that patch. Sliding the filter one step to the right gives the next patch `[1 0 0 / 1 0 0 / 1 0 0]`,
whose dot product with the filter is `1+0+0+1+0+0+1+0+0 = 3` — a smaller value, because that patch
straddles the edge less cleanly. Repeating this across every valid position produces the full
feature map — high values cluster right where the vertical edge in the input actually is (the
boundary between the left 1s and right 0s), which is exactly the point: the filter "lights up" where
its learned pattern occurs.

---

## 4. Padding and strides

Sliding a filter over an image the naive way has two side effects worth controlling deliberately.

### Padding

Without any adjustment, convolution **shrinks the output** (a 3×3 filter on a 4×4 input gives only a
2×2 output above) and **under-uses border pixels** — a pixel in the corner is covered by far fewer
filter positions than a pixel in the center, so edge information gets diluted across layers.

**Padding** adds a border of (usually zero-valued) pixels around the input before convolving:

- **Valid padding** — no padding at all. The output shrinks with every conv layer, and border pixels
  are underrepresented. This is the default in the numeric example above.
- **Same padding** — add just enough border pixels so the output has the **same spatial size** as
  the input. This preserves size (important for building deep stacks of conv layers without the
  feature map vanishing to nothing) and gives border pixels fairer representation.

### Strides

**Stride** is the step size the filter moves by at each slide (default is 1, moving one pixel at a
time). A **larger stride** skips positions, producing a **smaller output** — it's a cheap, simple way
to **downsample** the spatial resolution as a side effect of convolving (as opposed to pooling,
Section 5, which downsamples explicitly).

### The output-size formula

For a square input of size `n`, filter size `f`, padding `p`, and stride `s`, the output size is:

```
output_size = floor((n + 2p - f) / s) + 1
```

Check it against the numeric example: `n=4, f=3, p=0, s=1` → `(4 + 0 - 3)/1 + 1 = 2`, matching the
2×2 feature map produced above. With "same" padding for a 3×3 filter and stride 1, `p=1` restores the
output to 4×4: `(4 + 2 - 3)/1 + 1 = 4`. ✓

---

## 5. Pooling: Max Pooling

A **pooling layer** downsamples a feature map by summarizing small windows (typically 2×2) into a
single value, sliding across the feature map the same way a filter does — but with **no learnable
weights**. It is a fixed, deterministic operation.

**Max pooling** (by far the most common) takes the **maximum** value in each window:

```
Feature map (4x4):        2x2 max pooling, stride 2:
1  3  2  4
2  6  1  0                6  4
0  1  8  5        ->      8  5
2  3  4  1
```

(Top-left 2×2 window `[1,3/2,6]` → max 6; top-right `[2,4/1,0]` → max 4; and so on.)

**Average pooling** is the alternative — takes the mean of each window instead of the max — but is
used less often because it dilutes strong activations.

### Why pooling, and why max specifically

- **Reduces spatial size → less computation and fewer parameters** in every subsequent layer,
  which also reduces overfitting risk (same bias–variance logic as any parameter reduction).
- **Adds a degree of translation invariance.** If the strongest edge-detector activation shifts by a
  pixel or two (because the object in the image moved slightly), max pooling over a window still
  picks up that same strong activation as long as it falls anywhere within the window — so small
  shifts in the input don't change the pooled output.
- **Max, specifically, keeps the strongest signal.** A feature map's high values mark "the pattern
  this filter detects is strongly present here." Taking the max of a local neighborhood preserves
  exactly that strongest evidence and discards the rest — appropriate when what matters is *whether*
  a feature was detected in a region, not the average activation across it. This is why max pooling
  dominates in practice over average pooling for feature-detection tasks.

---

## 6. CNN architecture: the general pattern, and LeNet-5

### The general pattern

A CNN is built by stacking a **feature-extraction** front end and a **classification** back end:

```
[Conv → ReLU → Pool] x N   (feature extraction — repeated, each block sees more abstract features)
        ↓
     Flatten                (collapse the final stack of feature maps into a 1D vector)
        ↓
 Dense → Dense → Output     (classification — an ordinary MLP, exactly like Section 03/04)
```

Each `[Conv → ReLU → Pool]` block extracts features at one level of abstraction and shrinks the
spatial size (via pooling and/or strides) while typically *increasing* the number of channels
(more filters per layer deeper in) — trading spatial resolution for a richer set of learned
features. By the time the stack reaches `Flatten`, the representation is compact enough that a plain
`Dense` classifier (everything from Sections 02–04) can do the final job cheaply.

### LeNet-5 (LeCun, 1998)

LeNet-5 is the classic, historically foundational instance of exactly this pattern, originally built
to read handwritten digits (the same task MNIST — Section 04 — represents today). Conceptually, its
layer sequence is:

```
Input (32x32 grayscale image)
  → Conv (6 filters)   → Pool
  → Conv (16 filters)  → Pool
  → Flatten
  → Dense → Dense
  → Output (10 classes)
```

The pattern to internalize from LeNet-5: **convolution + pooling blocks progressively extract
features and shrink spatial size while growing channel depth; flattening bridges to a standard
dense classifier at the end.** Every modern CNN — from AlexNet through today's vision backbones — is
a variation on this same skeleton, just deeper, wider, and with more sophisticated building blocks
(covered in Section 15).

---

## 7. Comparing CNN vs ANN

| Dimension | ANN (fully-connected) | CNN |
|---|---|---|
| **Parameters** | Huge — every input pixel connects to every neuron. A 1000×1000×3 image needs millions of weights per neuron. | Small and fixed by filter size, not image size. A 3×3 filter has 9 weights (+1 bias) regardless of image resolution. |
| **Spatial structure** | Destroyed — image is flattened into a 1D vector before the network ever sees it. | Preserved — convolution operates on the 2D (or 3D with channels) grid directly. |
| **Translation invariance** | None — a pattern must be relearned at every position it can appear in. | Built in — parameter sharing means one learned filter detects its pattern anywhere in the image; pooling adds further robustness to small shifts. |
| **Suitability for images** | Poor beyond tiny images (like MNIST) — intractable parameter counts and severe overfitting on realistic images. | Purpose-built for images and other grid data; scales to large, high-resolution inputs. |
| **Feature learning** | Learns features, but only as flat, position-specific combinations of pixels — no notion of local pattern reuse. | Learns a spatial *hierarchy* of features — simple local patterns (edges) in early layers composed into complex, more global patterns in later layers, mirroring the visual cortex. |

**Concrete parameter comparison:** for a 1000×1000×3 image, a single `Dense` layer with 1,000 units
needs `3,000,000 × 1,000 = 3,000,000,000` weights. A conv layer with 1,000 filters of size 3×3×3
needs `(3×3×3 + 1) × 1,000 = 28,000` weights — over **five orders of magnitude** fewer, independent
of how large the image is, because the filter is shared across every spatial position instead of
having a unique weight for every pixel.

---

## 8. Minimal Keras code

```python
from tensorflow import keras
from tensorflow.keras import layers

model = keras.Sequential([
    layers.Conv2D(32, kernel_size=(3, 3), activation='relu',
                   padding='same', input_shape=(28, 28, 1)),
    layers.MaxPooling2D(pool_size=(2, 2)),

    layers.Conv2D(64, kernel_size=(3, 3), activation='relu', padding='same'),
    layers.MaxPooling2D(pool_size=(2, 2)),

    layers.Flatten(),
    layers.Dense(128, activation='relu'),
    layers.Dense(10, activation='softmax'),
])

model.compile(optimizer='adam',
              loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])
```

This is the exact `[Conv → ReLU → Pool] x2 → Flatten → Dense → Output` skeleton from Section 6,
applied to an MNIST-sized (28×28×1) input — the same dataset Section 04 solved with a plain ANN,
now solved with far fewer parameters and much better use of spatial structure.

---

## Key takeaways

- Flattening an image for an ANN destroys spatial structure and requires an intractable number of
  parameters; CNNs fix both by exploiting **spatial locality** and **translation invariance**.
- CNNs are biologically inspired by **Hubel & Wiesel's cat experiment**: local receptive fields and a
  simple-to-complex feature hierarchy in the visual cortex, mirrored by conv layers and their stacking.
- **Convolution** slides a learned filter over the input, computing a dot product at each position to
  produce a **feature map**; the same filter is reused everywhere (**parameter sharing**), giving
  translation invariance and a parameter count independent of image size. Multiple filters produce
  multiple feature maps (**channels**); depth of layers builds a simple → complex feature hierarchy.
- **Padding** ("same" vs "valid") controls whether spatial size is preserved; **stride** controls the
  step size and downsamples the output. Output size: `(n + 2p - f)/s + 1`.
- **Max pooling** downsamples feature maps with no learnable parameters, reduces computation and
  overfitting, and adds translation invariance by keeping only the strongest activation in each
  window.
- The canonical CNN skeleton — feature extraction (`[Conv → ReLU → Pool]` stacked) then
  classification (`Flatten → Dense → Output`) — dates back to **LeNet-5** and still underlies modern
  architectures.
- Against a plain ANN, a CNN wins on parameters (orders of magnitude fewer), preserves spatial
  structure, has built-in translation invariance, and learns a genuine feature hierarchy — which is
  why CNNs, not ANNs, are the default for image data.

**Next:** [14 — CNN Backpropagation](14-cnn-backpropagation.md) — how gradients flow backward through
convolution, pooling, and flatten layers.
