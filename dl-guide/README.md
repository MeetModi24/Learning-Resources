# Deep Learning — Theory Guide

A theory-first companion to CampusX's **"100 Days of Deep Learning"** playlist (84 videos).
Like its sibling [`../ml-guide`](../ml-guide/README.md), the goal here is *understanding* — what each
idea means, the intuition behind it, why it works, when it breaks, and how the pieces connect. Math
appears only where it genuinely clarifies the concept (the same places the videos slow down and
derive it). Short code snippets (mostly Keras/TensorFlow) appear only when they make an idea concrete.

> This guide assumes the classical-ML foundations from `../ml-guide` — especially gradient descent,
> the bias–variance trade-off, regularization, and evaluation metrics. Deep learning reuses all of
> them; we build on that vocabulary rather than repeat it.

## How to read this

- **If you're new to DL:** read top to bottom. The order mirrors how a neural network is actually
  built up: a single neuron → a network → how it learns → how to make it learn *well* → specialized
  architectures (CNNs for images, RNNs/LSTMs for sequences, Transformers for everything).
- **Notation:** weights **w**, biases **b**, activations **a**, pre-activations **z**. Vectors and
  matrices bold where it matters. Loss functions, gradients, and probabilities are spelled out in
  words first, symbols second.

## The through-line of this guide

Deep learning is, at its core, three ideas stacked together:
1. **A flexible function** built from layers of simple units (neurons) — Sections 01–04.
2. **A way to train it** — define a loss, compute gradients by backpropagation, step with gradient
   descent — Sections 05–07.
3. **A large bag of tricks to make training actually work** — activations, initialization,
   normalization, optimizers, regularization — Sections 07–12.

Everything after that (CNNs, RNNs, Transformers) is a specialized *architecture* — a smarter way to
wire the neurons for a particular kind of data.

## Table of contents

| # | File | Videos | Topics |
|---|------|--------|--------|
| 01 | [dl-foundations.md](01-dl-foundations.md) | 1–3 | What deep learning is, DL vs ML, types of neural networks, history, applications |
| 02 | [perceptron.md](02-perceptron.md) | 4–7 | The perceptron, geometric intuition, the perceptron trick, loss function, its limits |
| 03 | [mlp-forward-propagation.md](03-mlp-forward-propagation.md) | 8–10 | MLP notation, multi-layer perceptron intuition, forward propagation |
| 04 | [building-anns-keras.md](04-building-anns-keras.md) | 11–13 | Building ANNs in Keras: classification, MNIST, regression |
| 05 | [loss-and-backpropagation.md](05-loss-and-backpropagation.md) | 14–17, 19 | Loss functions, backpropagation (what/how/why), memoization |
| 06 | [vanishing-exploding-gradients.md](06-vanishing-exploding-gradients.md) | 18 | The vanishing and exploding gradient problems |
| 07 | [improving-neural-networks.md](07-improving-neural-networks.md) | 20–26 | GD variants, improving performance, early stopping, scaling, dropout, L1/L2 |
| 08 | [activation-functions.md](08-activation-functions.md) | 27–28 | Sigmoid, tanh, ReLU and its variants |
| 09 | [weight-initialization.md](09-weight-initialization.md) | 29–30 | Why init matters, Xavier/Glorot and He initialization |
| 10 | [batch-normalization.md](10-batch-normalization.md) | 31 | Batch normalization |
| 11 | [optimizers.md](11-optimizers.md) | 32–38 | EWMA, momentum, NAG, AdaGrad, RMSProp, Adam |
| 12 | [hyperparameter-tuning.md](12-hyperparameter-tuning.md) | 39 | Keras Tuner |
| 13 | [cnn-foundations.md](13-cnn-foundations.md) | 40–46 | Convolution, padding/strides, pooling, LeNet, CNN vs ANN |
| 14 | [cnn-backpropagation.md](14-cnn-backpropagation.md) | 47–48 | Backpropagation through conv, pooling, and flatten layers |
| 15 | [cnns-in-practice.md](15-cnns-in-practice.md) | 49–54 | Projects, data augmentation, pretrained models, transfer learning, visualization, functional API |
| 16 | [recurrent-neural-networks.md](16-recurrent-neural-networks.md) | 55–60 | Why RNNs, architecture, BPTT, types of RNN, problems with RNNs |
| 17 | [lstm-gru.md](17-lstm-gru.md) | 61–66 | LSTM, GRU, stacked and bidirectional RNNs |
| 18 | [seq2seq-attention.md](18-seq2seq-attention.md) | 67–70 | Encoder–decoder, seq2seq, attention, Bahdanau vs Luong |
| 19 | [transformers-self-attention.md](19-transformers-self-attention.md) | 71–78 | Transformers intro, self-attention, scaled dot-product, geometric intuition, multi-head, positional encoding |
| 20 | [transformers-architecture.md](20-transformers-architecture.md) | 79–84 | Layer norm, encoder, masked attention, cross attention, decoder, inference |

## Status

This guide is being written section by section. Completed sections are linked above; if a file
isn't present yet, it's still being written.
