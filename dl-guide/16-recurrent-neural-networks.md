# 16 — Recurrent Neural Networks

**Playlist videos 55–60.** ANNs and CNNs both assume a **fixed-size** input with no meaningful
temporal order between features. Text, speech, and time-series data break both assumptions: they're
**sequences** of variable length where **order carries the meaning**. This section introduces the
RNN — a network with a loop, so it can carry a "memory" of what it has seen so far — covers its
forward pass, a sentiment-analysis walkthrough, the four input/output shapes it can be wired into,
how it's trained (BPTT), and the two structural problems that motivate the next section (LSTM/GRU).

---

## 1. Why RNNs are needed

Recall from Section 04: to feed an image into a plain ANN you **flatten** it into a fixed-length
vector, which throws away 2D spatial structure — that's exactly why CNNs exist. Sequential data has
the analogous problem, but worse:

- **Variable length.** A sentence can be 3 words or 300. A plain `Dense` layer needs a fixed number
  of input features, so there's no single vector shape that fits every sentence.
- **Order matters.** "dog bites man" and "man bites dog" would flatten (or bag-of-words encode) to
  the *same* set of tokens, but they mean opposite things. An ANN fed a flattened/bag-of-words
  representation has no notion of *which token came before which* — it sees an unordered pile of
  features.
- **No memory across positions.** Even if you pad every sequence to the same fixed length and feed
  it to an ANN, each input position gets its own independent set of weights. The network never
  learns "what I saw two steps ago should influence how I interpret what I'm seeing now" — every
  position is processed as if the others didn't exist.

An **RNN** fixes all three: it processes a sequence **one element at a time**, reusing the **same**
small network at every step, and passes forward a **hidden state** — a running summary vector — that
carries context from every earlier element into the current one. This is what "recurrent" means:
the network feeds its own previous output (the hidden state) back into itself.

---

## 2. RNN architecture and forward propagation

At each timestep `t`, the RNN cell takes two inputs: the current element of the sequence, `x_t`, and
the hidden state carried over from the previous step, `h_{t-1}`. It combines them into a new hidden
state:

```
h_t = tanh( W_xh · x_t + W_hh · h_{t-1} + b_h )
y_t = g( W_hy · h_t + b_y )        # g = softmax/sigmoid/identity depending on the task
```

- `W_xh` — weights from the input to the hidden state.
- `W_hh` — weights from the previous hidden state to the current hidden state (this is the "memory"
  connection — it's what makes the network recurrent).
- `W_hy` — weights from the hidden state to the output at this timestep (only needed if you read an
  output at every step; some architectures only read the output at the last step — see §4).
- `h_0` is usually initialized to zeros.

**The critical detail: `W_xh`, `W_hh`, `W_hy`, and the biases are the *same* matrices at every
timestep.** The network doesn't learn a different set of weights for position 1 vs. position 50 — it
learns *one* cell and applies it repeatedly. This is **weight sharing across time**, directly
analogous to the parameter sharing a CNN kernel does across space (Section 13): a CNN slides one
small filter over every image patch; an RNN slides one small cell over every timestep. This is also
what solves the variable-length problem — the same fixed-size cell can be applied 3 times or 300
times, so sequence length no longer determines parameter count.

### Unrolling through time

It's easiest to *think* about backprop and architecture by drawing the RNN "unrolled" — one copy of
the cell per timestep, laid out left to right, with the hidden state as an arrow flowing from each
copy to the next:

```
x_1 → [cell] → h_1 → [cell] → h_2 → [cell] → h_3 → ...
        ↑ h_0           ↑ x_2          ↑ x_3
        ↓ y_1           ↓ y_2          ↓ y_3
```

This is a visualization trick, not a different model — it's still one cell with one set of weights,
just drawn once per timestep so you can see the data flow and (in §5) the gradient flow.

---

## 3. Sentiment analysis walkthrough (Integer Encoding → Embedding → SimpleRNN)

A concrete pipeline for classifying a review as positive/negative:

1. **Integer encoding.** Build a vocabulary of the most common words in the training corpus and map
   each word to an integer index (e.g. `"good"` → `42`). A sentence becomes a sequence of integers,
   padded/truncated to a fixed max length so it can batch efficiently. Integers alone are a bad
   direct input to a network — `42` and `43` being adjacent implies a numeric relationship between
   two words that doesn't exist.

2. **Embedding layer.** Instead of feeding raw integers (or one-hot vectors) into the RNN, an
   `Embedding` layer maps each integer index to a **learned dense vector** (e.g. 32 or 100
   dimensions). Why embeddings beat one-hot encoding:
   - **One-hot is huge and sparse** — a 20,000-word vocabulary means every token is a 20,000-length
     vector that's all zeros except one 1. That's wasteful and gives the network nothing to learn
     from structurally.
   - **Embeddings are dense and learned** — the network trains these vectors jointly with the rest of
     the model, so words used in similar contexts end up with similar vectors ("good" and "great"
     land near each other in the embedding space) purely as a side effect of optimizing the loss.
   - **Embeddings are lower-dimensional and reusable** — dense vectors are cheap to compute with, and
     pretrained embeddings (Word2Vec, GloVe) can be dropped in instead of training from scratch.

3. **SimpleRNN layer.** The sequence of embedding vectors (one per word) is fed into a `SimpleRNN`
   layer one timestep at a time, exactly as in §2 — each word updates the hidden state, so by the
   last word the hidden state is a summary of the whole sentence.

4. **Dense + sigmoid.** The final hidden state (a fixed-size vector regardless of sentence length) is
   passed to a `Dense(1, activation='sigmoid')` layer to produce a positive/negative probability —
   this is the **many-to-one** shape from §4.

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Embedding, SimpleRNN, Dense

model = Sequential([
    Embedding(input_dim=vocab_size, output_dim=32, input_length=max_len),
    SimpleRNN(32),
    Dense(1, activation='sigmoid')
])
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])
```

---

## 4. Types of RNN by input/output shape

The same recurrent cell can be wired up differently depending on whether you read an input / produce
an output at every timestep, or only once:

| Type | Shape | Example |
|---|---|---|
| **One-to-one** | single input → single output | Vanilla feedforward network (not really recurrent — included for contrast) |
| **One-to-many** | single input → sequence of outputs | Image captioning: one image in, a sequence of words out |
| **Many-to-one** | sequence of inputs → single output | Sentiment classification (§3): a whole sentence in, one label out |
| **Many-to-many (equal length)** | sequence in → sequence out, same length, aligned per step | Named Entity Recognition, Part-of-Speech tagging: one tag per input word |
| **Many-to-many (unequal length)** | sequence in → sequence out, different length | Machine translation: an **encoder-decoder** architecture (two RNNs — one reads the input sequence, the other generates the output sequence) — covered in Section 18 |

The equal-length many-to-many case reads an output `y_t` at every timestep directly off `h_t`. The
unequal-length case can't do that — the output length doesn't match the input length — so it needs
the encoder-decoder split, which is a forward reference to Section 18.

---

## 5. Backpropagation Through Time (BPTT)

Training an RNN is backpropagation applied to the **unrolled** network from §2. Conceptually:

1. Run the forward pass across all timesteps, computing `h_1, h_2, ..., h_T` and the output(s).
2. Compute the loss (summed/averaged across timesteps if there's an output at every step, as in
   many-to-many; or just at the last step, as in many-to-one).
3. Backpropagate the loss gradient **backward through the unrolled graph, from the last timestep to
   the first** — hence "through time." At each timestep the gradient with respect to `h_t` depends
   on the gradient with respect to `h_{t+1}` (chain rule across the `W_hh` connection), so gradients
   flow step by step all the way back to `t=1`.

Because `W_xh`, `W_hh`, and `W_hy` are the **same matrices reused at every timestep** (§2), each of
them receives a separate gradient contribution *from every timestep it participated in*, and — just
like the shared-weight gradient accumulation you saw for a CNN kernel reused at every spatial
position (Section 14) — those per-timestep gradients are **summed** into one total gradient per
weight matrix before the update step:

```
∂L/∂W_hh = Σ_t  ∂L_t/∂W_hh
```

This is what makes BPTT expensive: for a sequence of length `T`, you're effectively backpropagating
through a `T`-layer-deep network (one "layer" per timestep) every single training step.

---

## 6. Problems with RNN

Two structural problems fall directly out of "backprop through a `T`-layer-deep unrolled network":

1. **Vanishing / exploding gradients over long sequences.** The chain rule across `T` timesteps
   multiplies the same `W_hh` (and the same activation derivative) together `T` times — this is
   exactly the repeated-multiplication mechanism from Section 06, just playing out across time
   instead of across depth. With `tanh`/sigmoid activations the gradient tends to shrink toward zero
   (**vanishing**) as it's pushed back through many steps, or with certain weight magnitudes it can
   grow unboundedly (**exploding**). The practical consequence is the **long-term dependency
   problem**: an RNN struggles to learn a relationship between elements that are far apart in the
   sequence, because the gradient signal connecting them has vanished by the time it reaches the
   earlier timestep. (E.g. it can't reliably connect a pronoun late in a paragraph back to a noun
   introduced many sentences earlier.)

2. **Sequential computation can't be parallelized.** `h_t` depends on `h_{t-1}`, which depends on
   `h_{t-2}`, and so on — there's no way to compute timestep 50 before timestep 49 is done. Unlike a
   CNN or a plain feedforward layer, where every output element can be computed independently and in
   parallel, an RNN is fundamentally a step-by-step loop. This makes both training and inference slow
   on modern hardware that's built for massive parallelism.

These two problems are the entire motivation for what comes next: **LSTM and GRU** (Section 17) are
built specifically to fix the long-term dependency problem within the recurrent framework, and later
the **Transformer** (Sections 18–20) abandons recurrence altogether — replacing it with attention —
to fix both problems at once (no vanishing gradient over long chains, and fully parallelizable
computation).

---

## Key takeaways

- RNNs exist because sequential data is **variable-length** and **order-dependent** — a plain ANN
  needs a fixed input size and has no notion of position, so it can't represent either property
  (compare to flattening images for CNNs, Section 04).
- The RNN cell reuses **the same weights** (`W_xh`, `W_hh`, `W_hy`) at every timestep, carrying a
  **hidden state** `h_t = tanh(W_xh·x_t + W_hh·h_{t-1} + b)` forward as memory — weight sharing across
  time, analogous to a CNN kernel's weight sharing across space.
- A typical NLP pipeline is integer encoding → **Embedding** (learned dense word vectors, far better
  than sparse one-hot) → **SimpleRNN** → task-specific output layer.
- RNNs come in four input/output shapes: one-to-one, one-to-many (captioning), many-to-one
  (sentiment), and many-to-many (equal-length tagging, or unequal-length translation via
  encoder-decoder — Section 18).
- **BPTT** is ordinary backprop on the unrolled network; because weights are shared across timesteps,
  each weight's gradient is the **sum** of its per-timestep gradients (same principle as Section 14's
  shared-kernel gradients).
- RNNs have two structural problems: **vanishing/exploding gradients** over long sequences (repeated
  multiplication across timesteps, as in Section 06, causing the long-term dependency problem), and
  **no parallelism** (each step depends on the last). These motivate LSTM/GRU next, and Transformers
  later.

**Next:** [17 — LSTM, GRU](17-lstm-gru.md) — fixing the RNN's memory problem.
