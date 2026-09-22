# 15 — CNNs in Practice

**Playlist videos 49–54.** Sections 13–14 built the theory of convolution, pooling, and how gradients
flow through them. This section is the applied payoff: build a real image classifier end to end, hit
the overfitting wall every small image dataset hits, and then learn the three standard fixes — data
augmentation, pretrained models, and transfer learning — plus how to peek inside a trained CNN and
how to build non-linear model graphs with the Keras Functional API.

---

## 1. Cat vs Dog Image Classification Project (CNN)

This is the anchoring example for the whole section: a binary image classifier built the way you'd
actually build one, not a toy.

### Pipeline

1. **Organize the data.** Images on disk, arranged into `train/cats`, `train/dogs`,
   `validation/cats`, `validation/dogs` directories. The directory structure *is* the label — no
   separate label file needed.
2. **Load with a generator.** Keras' `ImageDataGenerator` (or `image_dataset_from_directory`) reads
   images off disk in batches, resizes them to a fixed input shape, and yields `(batch_of_images,
   batch_of_labels)` tuples — this is required because a full image dataset rarely fits in memory at
   once.
3. **Build the model**: a stack of `Conv2D → MaxPooling2D` blocks (Section 13) that progressively
   extract and downsample features, then `Flatten` into a `Dense` head, ending in a single sigmoid
   unit for binary classification.
4. **Train** with `binary_crossentropy` loss and `Adam` (Section 11), monitoring both training and
   validation accuracy/loss per epoch.

```python
from tensorflow.keras.preprocessing.image import ImageDataGenerator
from tensorflow.keras import layers, models

train_gen = ImageDataGenerator(rescale=1./255).flow_from_directory(
    "data/train", target_size=(150, 150), batch_size=32, class_mode="binary")
val_gen = ImageDataGenerator(rescale=1./255).flow_from_directory(
    "data/validation", target_size=(150, 150), batch_size=32, class_mode="binary")

model = models.Sequential([
    layers.Conv2D(32, (3, 3), activation="relu", input_shape=(150, 150, 3)),
    layers.MaxPooling2D(2, 2),
    layers.Conv2D(64, (3, 3), activation="relu"),
    layers.MaxPooling2D(2, 2),
    layers.Conv2D(128, (3, 3), activation="relu"),
    layers.MaxPooling2D(2, 2),
    layers.Flatten(),
    layers.Dense(128, activation="relu"),
    layers.Dense(1, activation="sigmoid"),
])
model.compile(optimizer="adam", loss="binary_crossentropy", metrics=["accuracy"])
model.fit(train_gen, epochs=20, validation_data=val_gen)
```

### The overfitting problem

Train this on a small dataset (a few thousand images per class, the classic Kaggle Cats vs Dogs
setup) and a very recognizable pattern shows up: **training accuracy climbs toward ~99% while
validation accuracy stalls and validation loss starts *rising*** after a handful of epochs. This is
the bias–variance trade-off from `../ml-guide`, made visible in an image setting: the network has
enough capacity (millions of parameters) to memorize the specific training images — their exact
poses, backgrounds, lighting — rather than learning the general concept "cat" or "dog."

Why this is *especially* bad for images: a CNN with a few thousand training examples has vastly more
parameters than data points, and images have huge natural variability (angle, lighting, scale,
background) that a small dataset can't represent. This overfitting problem is precisely what motivates
the next three techniques — augmentation attacks it by manufacturing more effective data,
pretrained/transfer learning attacks it by not needing to learn low-level features from scratch at
all.

---

## 2. Data Augmentation

**Data augmentation artificially expands the training set** by applying random, **label-preserving**
transformations to each training image on the fly — a rotated cat is still a cat.

### Common transformations

| Transform | What it does |
|---|---|
| Rotation | Rotate the image by a random small angle |
| Horizontal/vertical flip | Mirror the image |
| Zoom | Randomly zoom in/out |
| Width/height shift | Translate the image within the frame |
| Shear | Skew the image along an axis |
| Brightness | Randomly darken/brighten |

### Why it works

- **Teaches invariance.** The whole point of a good image classifier is to recognize the object
  regardless of small changes in viewpoint, position, or lighting. Augmentation forces the network to
  learn exactly that invariance, instead of overfitting to one exact framing.
- **Reduces overfitting.** The network never sees the *exact* same image twice, so it can't
  memorize pixel-level specifics — it's pushed toward learning general features instead.
- **Effectively more data, for free.** No new images or labels are collected; each epoch generates
  fresh variations from the same underlying images.

### Applied only to training data, on the fly

Augmentation is applied **only to the training set** — never to validation/test data, since you want
validation to measure performance on realistic, untouched images. It also happens **on the fly**,
generating a different random transform each time an image is drawn during training rather than
precomputing and storing an inflated dataset.

```python
train_gen = ImageDataGenerator(
    rescale=1./255,
    rotation_range=20,
    width_shift_range=0.1,
    height_shift_range=0.1,
    shear_range=0.1,
    zoom_range=0.2,
    horizontal_flip=True,
    brightness_range=[0.8, 1.2],
).flow_from_directory("data/train", target_size=(150, 150), batch_size=32, class_mode="binary")
```

(Newer Keras APIs offer the same idea as composable augmentation *layers* — e.g.
`layers.RandomRotation`, `layers.RandomZoom` — placed directly inside the model so augmentation is
part of the graph and automatically skipped at inference time.)

Augmentation alone typically closes much, but not all, of the train/validation accuracy gap seen in
Section 1 — it makes better use of a small dataset, but it doesn't give the network the general
visual knowledge that a much larger dataset would.

---

## 3. Pretrained Models | ImageNet | ILSVRC | Keras Applications

### ImageNet and ILSVRC

**ImageNet** is a massive labeled image dataset — over 14 million images across roughly 20,000+
categories. **ILSVRC** (the ImageNet Large Scale Visual Recognition Challenge) was the annual
competition (2010–2017) built on a ~1.2 million image, 1,000-class subset of ImageNet, and it directly
drove the architectures that shaped modern computer vision:

- **AlexNet (2012)** — the deep CNN that dramatically beat every prior (non-deep) approach on
  ILSVRC, kicking off the deep learning boom referenced in Section 01.
- **VGG (2014)** — very deep, uniform stacks of small 3×3 convolutions.
- **GoogLeNet / Inception (2014)** — parallel branches of different filter sizes within a block
  (this is exactly the kind of non-linear, branching graph that motivates the Functional API below).
- **ResNet (2015)** — introduced skip/residual connections, enabling networks hundreds of layers
  deep (also a Functional-API-only architecture — see below).

### Keras Applications

**Keras Applications** ships these exact architectures **with weights already trained on
ImageNet** — someone else already spent enormous compute and data learning general-purpose visual
features (edges, textures, shapes, object parts). You don't have to redo that work.

```python
from tensorflow.keras.applications import VGG16

base_model = VGG16(weights="imagenet", include_top=False, input_shape=(150, 150, 3))
```

`include_top=False` drops the original 1000-class ImageNet classification head, leaving just the
convolutional feature-extracting base — exactly what's needed for transfer learning onto a new task.

---

## 4. Transfer Learning | Fine Tuning vs Feature Extraction

**Transfer learning** reuses a model pretrained on a large dataset (ImageNet) for a *different*,
often much smaller, target task (cats vs dogs). This is the direct fix for the overfitting problem in
Section 1: instead of learning low-level visual features from a few thousand images, you start from
features already learned on millions of images and only adapt what's necessary.

There are two modes.

### Feature extraction (freeze the base)

**Freeze the pretrained convolutional base entirely** (`base_model.trainable = False`) and use it as
a fixed feature extractor. Only a new classifier head — a few `Dense` layers added on top — is
trained on your data.

```python
base_model.trainable = False

model = models.Sequential([
    base_model,
    layers.Flatten(),
    layers.Dense(128, activation="relu"),
    layers.Dense(1, activation="sigmoid"),
])
model.compile(optimizer="adam", loss="binary_crossentropy", metrics=["accuracy"])
model.fit(train_gen, epochs=10, validation_data=val_gen)
```

Best when your dataset is **small** and **similar** to ImageNet (natural photos of everyday objects,
animals, etc.) — there's little reason to touch the base's already-general features, and freezing
them means far fewer trainable parameters, which is exactly what a small dataset needs to avoid
overfitting again.

### Fine-tuning (unfreeze and adapt)

**Unfreeze some of the top (later) layers** of the pretrained base and train them — at a **low
learning rate** — together with the new head, adapting the higher-level features to your specific
task.

```python
base_model.trainable = True
for layer in base_model.layers[:-4]:   # keep all but the last few conv layers frozen
    layer.trainable = False

model.compile(optimizer=tf.keras.optimizers.Adam(1e-5),  # low LR
              loss="binary_crossentropy", metrics=["accuracy"])
model.fit(train_gen, epochs=10, validation_data=val_gen)
```

Best when you have **more data** to fine-tune with.

**Why freeze the early layers but adapt the later ones?** This connects straight back to Section 13's
CNN-feature-hierarchy intuition and Section 5 below: early conv layers learn generic, low-level
features (edges, colors, simple textures) that are useful for *any* natural image task — no reason to
retrain those. Later conv layers learn increasingly task-specific, higher-level features (object
parts, whole shapes) that benefit from being nudged toward your particular classes.

**Why a low learning rate?** The base's weights already encode good features. A normal-sized learning
rate would apply large gradient updates and wreck those pretrained weights ("catastrophic
forgetting") before the new head has even learned anything useful. A low LR makes small, careful
adjustments that adapt rather than overwrite.

### Which strategy to use

| Your dataset is... | Similar to ImageNet | Different from ImageNet |
|---|---|---|
| **Small** | Feature extraction (freeze base, train head only) | Feature extraction + train head on more epochs (limited headroom either way — few options are great) |
| **Large** | Fine-tuning (unfreeze top layers, low LR) | Fine-tuning more layers, or train from scratch if very different and very large |

---

## 5. What does a CNN see? | Visualizing Filters and Feature Maps

This is the interpretability payoff that confirms the feature-hierarchy intuition from Section 13
directly, by looking inside a trained network.

- **Filter visualization**: each convolutional filter is itself a small grid of learned weights.
  Visualizing filters from the *first* conv layer typically shows simple, recognizable patterns —
  edge detectors, color blobs, oriented gradients. Filters in *deeper* layers are harder to interpret
  directly as images (they operate on already-abstracted feature maps), which is why feature-map
  visualization is more informative for later layers.
- **Feature map (activation) visualization**: pick a trained model, feed it a real image, and plot
  the output of each conv layer's feature maps. This shows *what the network responded to* for that
  specific image, layer by layer.

The consistent finding across both techniques, confirming Section 13's "spatial hierarchy" intuition:

- **Early layers** activate on **edges, colors, and simple textures** — low-level, generic, image-agnostic
  features.
- **Middle layers** activate on **combinations of edges/textures** — patterns like fur, eyes, or
  wheels.
- **Deep layers** activate on **whole object parts or entire objects** — highly abstract,
  task-specific features.

This is exactly why transfer learning's freeze/fine-tune split (Section 4) makes sense: early layers'
features are generic enough to reuse as-is, while deep layers' features are specific enough that
adapting them to a new task is worthwhile.

---

## 6. Keras Functional API / Functional Model | Non-linear Neural Networks

Every model built so far in this guide has used `Sequential` — a straight-line stack of layers, each
feeding only into the next. That's a strong constraint: it cannot express a layer with **multiple
inputs**, **multiple outputs**, a **shared layer used in two places**, or a **skip/residual
connection** that adds an earlier layer's output to a later one.

The **Functional API** removes that constraint. Layers become callable objects applied directly to
tensors, and any resulting graph — not just a chain — can be assembled into a model.

```python
from tensorflow.keras import Input, layers, Model

inputs = Input(shape=(150, 150, 3))
x = layers.Conv2D(32, (3, 3), activation="relu")(inputs)
x = layers.MaxPooling2D(2, 2)(x)
x = layers.Flatten()(x)
outputs = layers.Dense(1, activation="sigmoid")(x)

model = Model(inputs=inputs, outputs=outputs)
```

The pattern: `x = SomeLayer(...)(previous_tensor)` — each layer is called *on* a tensor, producing a
new tensor, and the graph of calls is wired up explicitly. `Model(inputs, outputs)` then bundles that
graph into a trainable model, wherever the wiring is.

### Why this matters here

Every non-linear architecture referenced earlier in this section needs it:

- **ResNet's skip connections** — `layers.Add()([x, shortcut])` — a branch that bypasses several
  layers and is added back in later. Impossible to express as a linear stack.
- **Inception's parallel branches** — several different conv paths applied to the *same* input
  tensor, then concatenated — again, not a straight line.
- **Assembling a transfer-learning model with multiple named layers/branches**, or any model with
  auxiliary outputs, multiple inputs (e.g. an image plus metadata), or a shared embedding layer used
  in two places.

`Sequential` remains the right, simpler tool for straight stacks (like the Cat vs Dog model in
Section 1). The Functional API is what you reach for the moment the architecture branches, merges, or
shares.

---

## Key takeaways

- The **Cat vs Dog project** is the template applied CNN workflow: organize data into
  train/validation directories, load with a generator, stack `Conv2D`/`MaxPooling2D` into a `Dense`
  sigmoid head, train — and on small datasets, watch training accuracy pull away from validation
  accuracy. That gap **is** overfitting, and it's the motivation for everything else in this section.
- **Data augmentation** (rotation, flip, zoom, shift, shear, brightness) manufactures more effective
  training data by applying random label-preserving transforms **to the training set only, on the
  fly** — it teaches invariance and reduces overfitting without collecting new images.
- **ImageNet/ILSVRC** produced the landmark architectures (AlexNet, VGG, Inception, ResNet); **Keras
  Applications** ships them with **ImageNet-pretrained weights** so you don't relearn general visual
  features from scratch.
- **Transfer learning** has two modes: **feature extraction** (freeze the base, train only a new
  head — best for small, ImageNet-similar data) and **fine-tuning** (unfreeze top layers, train at a
  **low learning rate** alongside the head — best with more data). Freeze early/generic layers,
  adapt later/task-specific ones.
- **Visualizing filters and feature maps** confirms the CNN feature hierarchy directly: early layers
  = edges/colors/textures, deeper layers = object parts and whole objects.
- The **Functional API** builds non-linear model graphs (multiple inputs/outputs, shared layers,
  skip/branch connections) that `Sequential` cannot express — required for ResNet-style skip
  connections, Inception-style branches, and flexible transfer-learning assembly.

**Next:** [16 — Recurrent Neural Networks](16-recurrent-neural-networks.md) — moving from grid-like
image data to sequential data, and why a CNN/MLP can't handle it.
