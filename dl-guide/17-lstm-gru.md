# 17 — LSTM, GRU, Stacked and Bidirectional RNNs

**Playlist videos 61–66.** Section 16 set up the plain RNN and ended with its fatal flaw: it can't
remember far back into a sequence, because backprop-through-time is just Section 06's vanishing
gradient problem playing out along the time axis instead of the layer axis. This section is the
fix — **LSTM**, its lighter cousin **GRU**, and two structural extensions (**stacking** and
**bidirectionality**) that get more out of either one.

---

## 1. LSTM Part 1 — The What (the long-term dependency problem)

### Why plain RNNs forget

A plain RNN has one hidden state `h_t` that gets **overwritten** at every time step:

```
h_t = tanh(W_hh · h_{t-1} + W_xh · x_t + b)
```

Every step multiplies the previous state by (roughly) the same weight matrix and squashes it through
`tanh` again. Unroll this across many time steps and it's exactly the deep-network chain-rule product
from Section 06, except now "depth" = sequence length. If the recurrent weight's effective
contribution per step is less than 1 (it usually is, especially with `tanh` saturating), the gradient
w.r.t. an early time step shrinks toward zero over dozens or hundreds of steps. The network literally
cannot learn a dependency that spans that many steps, because there's no gradient signal left to
learn it with.

### The classic example

> "I grew up in France... [200 words of unrelated plot] ...so I speak fluent **French**."

To predict the last word, the model needs to recall "France," which appeared 200 tokens earlier. A
plain RNN's single overwritten hidden state has long since been overwritten by everything in between
— this is the **long-term dependency problem**, and it's the reason vanilla RNNs are barely usable
beyond short sequences (short-term dependencies, like predicting "sky" after "the clouds are in the
___", are fine — those only need a handful of steps back).

### The fix, at the intuition level

LSTM (Long Short-Term Memory) solves this by splitting memory into two things instead of one:

- A **hidden state `h_t`** — same role as before, the short-term, per-step working memory.
- A **cell state `C_t`** — a *separate* long-term memory that flows across time steps largely
  **unchanged**, like a conveyor belt running the length of the sequence. Information can be dropped
  onto it or read off it at each stop, but by default it just keeps moving forward.

Instead of overwriting memory every step, LSTM learns **gates** — small neural networks (a sigmoid
layer) that output a number between 0 and 1 per memory dimension, deciding how much of something to
let through:

- `0` → completely block (forget this / write nothing / output nothing).
- `1` → completely pass (keep this / write it in full / output it in full).
- values in between → partial pass.

Because the cell state is updated **additively** (add and subtract, rather than being reprojected and
squashed through `tanh` every step like the plain RNN's hidden state), a piece of information the
gates decide to keep can survive essentially unchanged for hundreds of steps — that conveyor belt is
precisely what fixes the vanishing-gradient problem for sequences.

---

## 2. LSTM Architecture Part 2 — The How (the gates)

Each LSTM cell, at time step `t`, looks at two inputs: the current input `x_t` and the previous hidden
state `h_{t-1}` (concatenated as `[h_{t-1}, x_t]`), and produces three gates plus a candidate, all as
sigmoid or tanh layers over that same concatenated input:

### Forget gate — what to erase from the cell state

```
f_t = σ(W_f · [h_{t-1}, x_t] + b_f)
```

A sigmoid over the same standard input, producing a vector of values in `[0, 1]`, one per cell-state
dimension. `f_t ≈ 0` for a dimension means "throw this piece of long-term memory away"; `f_t ≈ 1`
means "keep it." This is what lets the model drop "France" once the sentence has moved on to a new
subject, or *keep* it if it's still relevant 200 words later — the gate is learned, not hardcoded.

### Input gate + candidate — what new information to write

Two pieces work together here:

```
i_t = σ(W_i · [h_{t-1}, x_t] + b_i)        # how much of the new info to let in (0–1)
C̃_t = tanh(W_c · [h_{t-1}, x_t] + b_c)     # the new info itself (candidate values)
```

`C̃_t` is a candidate update — "here's what could be added to memory right now" (e.g., "the subject
just changed to a country"). `i_t` is the gate that decides how much of that candidate actually gets
written in. Separating "what could be written" (`tanh`, can be negative or positive) from "how much
to write" (sigmoid, `0`–`1`) is what lets the network both add and selectively suppress information.

### Cell state update — combining forget and input

```
C_t = f_t * C_{t-1} + i_t * C̃_t
```

Read this literally: **keep the fraction of old memory the forget gate allows, plus write in the
fraction of new information the input gate allows.** This is a simple element-wise multiply-then-add
— no matrix reprojection, no repeated `tanh` squashing of the *entire* accumulated state. That's the
whole trick.

### Why the additive update fixes vanishing gradients

Take the gradient of `C_t` with respect to `C_{t-1}`: it's just `f_t` (plus a term from the other
branch). Compare that to a plain RNN, where `∂h_t/∂h_{t-1}` involves multiplying by the recurrent
weight matrix *and* the `tanh` derivative every single step — a product of many sub-1 factors that
shrinks exponentially (Section 06). Here, if the forget gate learns to stay close to `1` for a
dimension worth remembering, the gradient for that dimension flows backward through many time steps
**largely unchanged** — no repeated squashing, no repeated reprojection. The gate doesn't just control
what's remembered going forward; it controls how far the gradient can travel going backward. That's
the mechanism, in one sentence: **additive, gated memory updates give the gradient a near-direct path
back through time, instead of a long product of shrinking factors.**

### Output gate — what to read out as the hidden state

```
o_t = σ(W_o · [h_{t-1}, x_t] + b_o)
h_t = o_t * tanh(C_t)
```

The cell state `C_t` holds everything currently in long-term memory, but not all of it is relevant to
predict *this* time step's output. The output gate decides which parts of the (squashed) cell state
to expose as the hidden state `h_t` — the thing actually used for this step's prediction and passed
to the next step as `h_{t-1}`.

### Putting the three gates together

| Gate | Formula | Question it answers |
|---|---|---|
| Forget | `f_t = σ(W_f·[h_{t-1}, x_t] + b_f)` | What do I erase from long-term memory? |
| Input | `i_t = σ(W_i·[h_{t-1}, x_t] + b_i)`, `C̃_t = tanh(...)` | What new information do I write in, and how much? |
| Output | `o_t = σ(W_o·[h_{t-1}, x_t] + b_o)` | What do I expose right now from memory? |

All three are the same shape of computation (a sigmoid, or tanh for the candidate, over the
concatenated `[h_{t-1}, x_t]`) with their own learned weights — the gating mechanism is uniform, only
its *role* in the cell differs.

---

## 3. LSTM Part 3 — Next Word Predictor (code walkthrough)

A minimal but complete pipeline for the classic "predict the next word" task, exactly the shape used
to demo LSTMs on text:

1. **Corpus → tokenize.** Split text into words/subwords, build a vocabulary, map each token to an
   integer id (`Tokenizer` in Keras).
2. **Build input/target pairs.** For a sentence like "deep learning is fun," generate progressively
   longer prefixes as inputs with the next word as the target: `["deep"] → "learning"`,
   `["deep","learning"] → "is"`, etc. This turns "predict the next word" into ordinary supervised
   multi-class classification (one class per vocabulary word).
3. **Pad sequences** to a fixed length so they can be batched.
4. **Embedding layer** turns each integer token id into a dense learned vector — this is *why* text
   needs an embedding layer before an RNN: raw token ids have no meaningful numeric relationship, but
   the embedding space learns one.
5. **LSTM layer** processes the embedded sequence, carrying the cell/hidden state forward.
6. **Dense + softmax** over the vocabulary produces a probability distribution over "what word comes
   next"; train with categorical cross-entropy.

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Embedding, LSTM, Dense

vocab_size = 5000
max_len = 20

model = Sequential([
    Embedding(input_dim=vocab_size, output_dim=100, input_length=max_len),
    LSTM(150),
    Dense(vocab_size, activation="softmax"),
])
model.compile(loss="categorical_crossentropy", optimizer="adam", metrics=["accuracy"])
```

At inference time, generation is autoregressive: feed a seed phrase, take the highest-probability (or
sampled) next word, append it to the input, and repeat — the same pattern used for any RNN-based text
generator.

---

## 4. GRU — Gated Recurrent Unit

GRU (Cho et al., 2014) is a simplification of LSTM: **it merges the cell state and hidden state into
one, and uses only two gates instead of three.**

- **Reset gate `r_t`** — decides how much of the previous hidden state to "forget" when computing the
  new candidate — plays a role similar to LSTM's forget gate, but applied to a single unified state.
- **Update gate `z_t`** — decides the *balance* between keeping the old hidden state and taking in the
  new candidate — it does the job of LSTM's forget gate and input gate *combined*, since there's no
  separate long-term cell state to manage independently.

```
r_t = σ(W_r · [h_{t-1}, x_t] + b_r)
z_t = σ(W_z · [h_{t-1}, x_t] + b_z)
h̃_t = tanh(W_h · [r_t * h_{t-1}, x_t] + b_h)
h_t = (1 - z_t) * h_{t-1} + z_t * h̃_t
```

Same idea as LSTM — sigmoid gates controlling how information flows, an additive final update so
gradients aren't crushed by repeated multiplication — just fewer moving parts.

### LSTM vs GRU

| | LSTM | GRU |
|---|---|---|
| States | Two: cell state `C_t` + hidden state `h_t` | One: hidden state `h_t` only |
| Gates | Three: forget, input, output | Two: reset, update |
| Parameters | More (roughly 4 weight matrices per layer) | Fewer (roughly 3 weight matrices per layer) → smaller model, less memory |
| Training/inference speed | Slower | Faster, typically ~25–30% fewer computations |
| Empirical accuracy | Often slightly better on very long sequences / large datasets | Often comparable, sometimes better on smaller datasets |
| When to prefer | Long sequences, ample data/compute, when squeezing out the last bit of accuracy matters | Faster iteration, resource-constrained settings, smaller datasets, as a strong default first try |

Neither is a strictly superior — this is genuinely a "try both, GRU is cheaper to try first" situation
in practice, not a settled theoretical winner.

---

## 5. Deep RNNs — stacking LSTM/GRU layers

A single recurrent layer learns one level of temporal abstraction. Just like stacking Dense layers in
an MLP lets each layer build on more abstract features from the last, **stacking recurrent layers**
lets each layer learn temporal patterns built on the patterns the layer below it already extracted —
e.g., a first LSTM layer might pick up local word-level patterns, a second might pick up phrase- or
clause-level structure built from the first layer's output sequence.

The mechanical requirement: a recurrent layer that feeds *another* recurrent layer must output its
**full sequence of hidden states** (one per time step), not just the final one — set
`return_sequences=True` on every LSTM/GRU layer except the last one in the stack (the last layer
returns just the final state, unless something downstream also needs the full sequence, e.g. an
attention layer in Section 18).

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import LSTM, Dense

model = Sequential([
    LSTM(128, return_sequences=True, input_shape=(max_len, num_features)),  # feeds next LSTM
    LSTM(64),                                                                # last layer: final state only
    Dense(1, activation="sigmoid"),
])
```

Forgetting `return_sequences=True` on an intermediate layer is a very common bug: the next LSTM layer
would receive a single vector instead of a sequence and either error out or silently treat it as a
sequence of length 1, losing all the temporal information the stack was supposed to build on.

---

## 6. Bidirectional RNN / BiLSTM

A standard (unidirectional) RNN only ever sees the past: at time step `t`, `h_t` was built from
`x_1 ... x_t`, never from `x_{t+1}` onward. For many tasks that's a real handicap — the correct
interpretation of a word often depends on what comes *after* it too:

> "The bank raised its interest rates" vs. "The bank of the river flooded."

Both sentences are identical up to "bank" — you need the rest of the sentence to disambiguate it, and
a forward-only RNN reading left to right hasn't seen that yet at the point it processes "bank."

### The idea

A **Bidirectional RNN** runs **two independent recurrent layers** over the same input sequence:

- A **forward** layer, reading `x_1 → x_2 → ... → x_T` as usual.
- A **backward** layer, reading the same sequence in reverse: `x_T → x_{T-1} → ... → x_1`.

At each time step `t`, the two layers' hidden states are combined (typically concatenated) into one
output: `h_t = [h_t^forward, h_t^backward]`. Every position in the output sequence now encodes
information from **both directions** — everything before it and everything after it.

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Bidirectional, LSTM, Dense

model = Sequential([
    Bidirectional(LSTM(64), input_shape=(max_len, num_features)),
    Dense(1, activation="sigmoid"),
])
```

`Bidirectional(...)` is a wrapper — it internally instantiates two copies of the layer you pass it
(one run forward, one run on the reversed sequence) and handles the concatenation.

### When to use it — and when you can't

Bidirectional RNNs are strictly more informative *when the entire sequence is available up front* —
tasks like named entity recognition, sentiment analysis, and part-of-speech tagging, where the model
gets the full sentence/document before it has to answer, benefit clearly from seeing future context.

But that "entire sequence available up front" condition is also the hard constraint: **a
bidirectional RNN cannot be used for real-time/streaming prediction or autoregressive generation**,
because the backward pass requires knowing the *future* tokens of the sequence — which don't exist yet
when you're generating text one token at a time, or processing a live audio/sensor stream as it
arrives. The next-word predictor in Section 3 above, and any decoder in Section 18's seq2seq models,
must stay unidirectional for exactly this reason.

| | Unidirectional (LSTM/GRU) | Bidirectional (BiLSTM/BiGRU) |
|---|---|---|
| Sees | Only past context | Past **and** future context |
| Use when | Streaming, real-time, autoregressive generation (next-word/token prediction, decoders) | Full sequence available upfront: NER, classification, tagging, encoders |
| Cost | 1x compute/parameters | ~2x compute/parameters (two passes) |

---

## Key takeaways

- Plain RNNs fail on long sequences because BPTT is the vanishing-gradient problem (Section 06)
  playing out over time steps instead of layers — the **long-term dependency problem**.
- **LSTM** fixes this with a separate **cell state** (long-term memory, updated additively) plus three
  sigmoid **gates** — forget, input, output — that learn what to erase, write, and read. The additive
  update `C_t = f_t·C_{t-1} + i_t·C̃_t` gives gradients a near-direct path backward through time.
- **GRU** simplifies LSTM to one state and two gates (reset, update) — fewer parameters, faster,
  often comparable accuracy; a good first thing to try before reaching for LSTM.
- **Stacking** recurrent layers (`return_sequences=True` on all but the last) lets the network learn
  hierarchical temporal features, the same way stacking Dense layers builds hierarchical features in
  an MLP.
- **Bidirectional RNNs** run forward and backward passes and concatenate them, giving every position
  both past and future context — a clear win for NER/classification/tagging, but unusable for
  real-time streaming or autoregressive generation, which structurally cannot see the future.

**Next:** [18 — Seq2Seq and Attention](18-seq2seq-attention.md) — encoder–decoder architectures,
sequence-to-sequence learning, and the attention mechanism that lets a decoder look back at the whole
input instead of relying on one compressed final state.
