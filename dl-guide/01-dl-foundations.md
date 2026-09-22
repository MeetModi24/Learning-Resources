# 01 — Deep Learning Foundations

**Playlist videos 1–3.** This section frames the whole guide: what deep learning is, how it differs
from the classical ML you already know, the main *families* of neural networks and what each is for,
and why the field suddenly took off. No neurons or math yet — just the map of the territory.

---

## 1. What is Deep Learning?

**Deep learning is a subfield of machine learning that uses artificial neural networks with many
layers** ("deep" = many hidden layers) to learn patterns directly from raw data. It is inspired —
loosely — by the structure of the human brain: simple units (neurons) connected in layers, where
each layer transforms its input and passes it on.

Recall the nesting from the ML guide: **AI ⊃ ML ⊃ DL**. Deep learning is ML done with deep neural
networks. So everything true of ML is still true here — you learn from data, you worry about
overfitting, you split train/test, you use gradient descent. What changes is the *model class* and,
crucially, **who does the feature engineering**.

### Deep Learning vs Machine Learning — the real differences

This is the most important comparison in the whole section. The differences are not cosmetic:

| Dimension | Classical ML | Deep Learning |
|---|---|---|
| **Feature engineering** | *Manual.* A human decides which features matter and hand-crafts them. This is where most of the effort goes. | *Automatic.* The network **learns** useful features/representations layer by layer from raw data. This is the single biggest difference. |
| **Data requirements** | Works well on small/medium datasets. | **Data-hungry** — usually needs large datasets to shine. |
| **Hardware** | Runs fine on a CPU. | Needs **GPUs/TPUs** for the massive parallel matrix math. |
| **Training time** | Fast (seconds to minutes). | Slow (hours to days). |
| **Interpretability** | Often interpretable (a decision tree, linear coefficients). | Mostly a **black box**. |
| **Data type** | Best on structured/tabular data. | Excels on **unstructured** data — images, audio, text, video. |
| **Performance ceiling** | Plateaus — more data helps only up to a point. | Keeps improving as data and model size grow. |

**The one-sentence version:** classical ML makes *you* find the features; deep learning finds them
*itself*, at the cost of needing far more data and compute. This is why DL dominates images, speech,
and language (where good features are almost impossible to hand-design) but is often overkill for a
clean tabular dataset (where gradient-boosted trees frequently still win — see `../ml-guide`).

### Why now? (Why deep learning took off)

The core ideas (neural nets, backpropagation) are decades old. Three things changed to make them
work in practice:

1. **Data.** The internet produced enormous labeled datasets (ImageNet, web text). DL's appetite
   for data could finally be fed.
2. **Hardware.** GPUs — originally for gaming — turned out to be perfect for the parallel matrix
   multiplications neural networks are made of. Training that took months became days.
3. **Algorithms & tooling.** Better activations (ReLU), initialization, optimizers, regularization
   (all covered later in this guide), plus frameworks (TensorFlow, Keras, PyTorch) that made
   building networks accessible.

---

## 2. Types of Neural Networks

You don't invent a network from scratch for each problem — you pick an **architecture** suited to
the *structure of your data*. The major families, each of which gets its own section later:

- **ANN / MLP (Artificial Neural Network / Multi-Layer Perceptron)** — the foundational
  fully-connected network. Good for tabular/structured data and the base case for understanding
  everything else. *(Sections 02–12.)*
- **CNN (Convolutional Neural Network)** — specialized for **grid-like data, especially images**. It
  exploits spatial locality (nearby pixels are related) via convolution, so it needs far fewer
  parameters than a fully-connected net on the same image. *(Sections 13–15.)*
- **RNN (Recurrent Neural Network)** — specialized for **sequential/temporal data** (text,
  time-series, audio). It has a "memory" that carries information from earlier steps forward.
  Variants **LSTM** and **GRU** fix its memory problems. *(Sections 16–17.)*
- **Transformers** — the modern architecture behind large language models. It replaces recurrence
  with **attention**, processing whole sequences in parallel and modeling long-range dependencies
  far better than RNNs. *(Sections 18–20.)*

There are others the field uses (autoencoders, GANs, etc.), but this playlist — and this guide —
focuses on the four above, which is the right backbone to learn first.

### A short history

The field has gone through boom-and-bust cycles, which is worth knowing so the ideas have context:

- **1943 — McCulloch & Pitts:** first mathematical model of a neuron.
- **1958 — Rosenblatt's Perceptron:** the first trainable neural model (Section 02).
- **1969 — Minsky & Papert:** showed a single perceptron can't even learn XOR → funding dried up
  ("first AI winter"). *This is exactly the "problem with the perceptron" you'll meet in Section 02.*
- **1986 — Backpropagation popularized** (Rumelhart, Hinton, Williams): made training *multi-layer*
  networks practical, reviving the field.
- **1998 — LeNet (LeCun):** convolutional nets read handwritten digits (Section 13).
- **2012 — AlexNet:** crushed the ImageNet competition, kicking off the modern deep-learning
  explosion — the "why now" of Section 1 made concrete.
- **2017 — "Attention Is All You Need":** the Transformer, which led to BERT, GPT, and today's LLMs
  (Sections 18–20).

---

## 3. Applications of Deep Learning

Where DL is state-of-the-art, organized by data type (which maps back to the architectures above):

- **Computer vision (images/video → CNNs):** image classification, object detection, face
  recognition, medical imaging, self-driving perception, image generation.
- **Natural language processing (text → RNNs/Transformers):** machine translation, sentiment
  analysis, chatbots and large language models, summarization, question answering.
- **Speech (audio → RNNs/CNNs/Transformers):** speech-to-text, voice assistants, text-to-speech.
- **Recommendation & ranking:** deep models power feeds and recommendations at scale.
- **Generative AI:** image generation, text generation, music, and more.

The pattern to internalize: **match the architecture to the data's structure** — grids → CNN,
sequences → RNN/Transformer, plain tables → often classical ML is enough.

---

## Key takeaways

- Deep learning = machine learning with **deep (many-layered) neural networks**; AI ⊃ ML ⊃ DL still
  holds.
- The defining difference from classical ML is **automatic feature learning** — the network learns
  representations instead of you engineering features — bought at the price of **more data, more
  compute, less interpretability**.
- DL took off because three things aligned: **big data, GPUs, and better algorithms/tooling**.
- Pick the **architecture to fit the data**: MLP (tabular), CNN (images), RNN/LSTM (sequences),
  Transformers (modern sequence/language modeling).
- Use DL where features are hard to hand-craft (vision, language, speech); for clean tabular data,
  classical ML is often the better tool.

**Next:** [02 — The Perceptron](02-perceptron.md) — the single neuron that everything is built from.
