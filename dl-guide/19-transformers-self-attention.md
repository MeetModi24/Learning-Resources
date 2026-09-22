# 19 — Transformers & Self-Attention

**Playlist videos 71–78.** Section 18 ended on a cliffhanger: attention alone, without any
recurrence at all, turned out to be enough to relate every position in a sequence to every other
position. This section builds that idea properly — **self-attention**, the mechanism at the core of
every Transformer layer — from the query/key/value machinery, through the scaled dot-product formula
and its geometric meaning, to **multi-head attention** and **positional encoding**, the two
refinements that turn plain self-attention into what the 2017 paper actually shipped. Section 20 picks
up from here and assembles these pieces into the full encoder-decoder Transformer.

---

## 1. Why Transformers — the one-sentence recap

Section 18 already made the case in detail; the short version: RNNs/LSTMs process a sequence **one
timestep at a time**, so step *t* can't start until step *t-1* finishes — slow to train, and still
imperfect at long-range dependencies even with gating. Attention showed that a decoder could relate to
*any* input position directly, with no recurrence needed to carry information forward. The Transformer
(*"Attention Is All You Need"*, 2017) takes that one step further: drop the RNN entirely and build a
model **purely out of attention layers**, applied not just decoder-to-encoder (cross-attention, Section
20) but **within a single sequence, token to token** — that's self-attention. No recurrence means no
sequential bottleneck: an entire sequence's self-attention can be computed **in parallel**, which is
the main reason Transformers scale to the sizes behind BERT, GPT, and everything since.

---

## 2. Self-Attention — the core idea

**Self-attention lets every token in a sequence look at every other token in the *same* sequence and
decide how much to "attend to" each one, in order to build a better, context-aware representation of
itself.**

Contrast this with Section 18's attention: there, the **query** came from the decoder and the
**keys/values** came from the encoder — two different sequences. In self-attention, query, key, *and*
value all come from **the same sequence**. A sentence attends to itself.

### Why this matters: word meaning depends on context

Take the word "bank" in:
- "I sat by the river **bank**."
- "I deposited money in the **bank**."

A plain word embedding (Section 16's input representation) gives "bank" the exact same vector in both
sentences — it has no way to know which sense is meant. Self-attention fixes this by letting "bank"
look at the *other words in its own sentence* ("river" vs. "deposited", "money") and pull in
information from them, producing a **contextualized representation** — a vector for "bank" that is
different in the two sentences because the surrounding context is different. This is the single
biggest reason self-attention-based representations beat static word embeddings across nearly every
NLP task.

---

## 3. Query, Key, Value — turning attention into a computation

Section 18 previewed the query/key/value vocabulary for encoder-decoder attention. Self-attention
generalizes it: for **every token** in the sequence, learn three separate projections of its embedding
vector:

- **Query (Q)** — "what am I looking for?"
- **Key (K)** — "what do I contain, for others to match against?"
- **Value (V)** — "what do I actually offer once someone decides to attend to me?"

Concretely, each token's embedding `x` (a vector) is multiplied by three separate learned weight
matrices to produce its query, key, and value vectors:

```
Q = X · W_Q
K = X · W_K
V = X · W_V
```

`X` is the whole sequence's embedding matrix (one row per token); `W_Q`, `W_K`, `W_V` are learned
weight matrices, trained by backpropagation exactly like any other weights in the network. Every token
ends up with its own `q`, `k`, `v` triplet — the *same* token plays the role of query for itself and
key/value for every other token's query.

### Why three separate projections, not just the raw embedding?

If you used the raw embedding directly as query, key, *and* value, the model would have no room to
learn "what's a good thing to search for" separately from "what's a good thing to be found by" and
"what's a good thing to actually hand over once matched." Three independent learned projections give
the model that flexibility — it's the same reasoning as learning `γ`/`β` separately in batch norm
(Section 10): a rigid, unlearned constraint would cost representational power for no good reason.

---

## 4. Scaled Dot-Product Attention

With Q, K, V in hand, self-attention computes, for every query token, a weighted sum over all value
vectors — weighted by how well that query matches each key. This is exactly the four-step recipe from
Section 18's attention, made precise with a formula:

```
Attention(Q, K, V) = softmax( (Q · Kᵗ) / sqrt(d_k) ) · V
```

Step by step:

1. **Score:** `Q · Kᵗ` — dot product every query against every key. This is a similarity measure: two
   vectors pointing in similar directions have a large dot product, unrelated vectors have a dot
   product near zero. The result is a matrix of raw **alignment scores**, one per (query, key) pair —
   i.e., for every pair of tokens in the sequence.
2. **Scale:** divide by `sqrt(d_k)`, where `d_k` is the dimensionality of the key vectors. This is the
   one genuinely new ingredient versus Luong's plain dot-product attention (Section 18) — see below
   for why it's needed.
3. **Softmax:** normalize each query's row of scores into weights that sum to 1 — same softmax role as
   Section 18's attention weights, just computed for every token against every other token at once,
   not just decoder-against-encoder.
4. **Weighted sum:** multiply the softmax weights by `V` to get, for every token, a new vector that's a
   context-aware blend of the whole sequence's value vectors.

### Why divide by `sqrt(d_k)`? The scaling isn't cosmetic

Dot products of two random vectors grow in magnitude with their dimensionality — roughly, the variance
of `Q · Kᵗ` scales with `d_k`. For a large `d_k` (real Transformers use 64 or more per head), the raw
scores can become large in magnitude. Softmax is an exponential function: very large or very negative
inputs push it into its **saturated regions**, where the gradient is tiny — the exact vanishing
gradient mechanics from Section 06, showing up here inside the attention computation itself. Dividing
by `sqrt(d_k)` rescales the scores back down to a range where softmax has healthy, well-behaved
gradients, keeping the whole layer trainable. This is why the mechanism is named *scaled* dot-product
attention, not just dot-product attention.

---

## 5. Geometric Intuition

It helps to picture this spatially rather than purely as matrix algebra:

- Every token's embedding is a point (vector) in a high-dimensional space.
- The **query** vector is "the direction I'm searching in."
- The **key** vector is "the direction I present myself in, so queries can find me."
- A large `Q · K` dot product means the query vector and that token's key vector point in a
  **similar direction** — geometrically close/aligned — so that token gets a large attention weight.
- The **output** for a given token is a point that has been **pulled toward** the value vectors of the
  tokens it attends to most strongly, and pulled away from (near-zero weight on) the ones it doesn't.

So self-attention is, geometrically, each token's representation **moving toward the tokens most
relevant to it** in the embedding space, by an amount controlled by how aligned their query/key
directions are. "Bank" near "river" moves toward the riverbank-sense direction; "bank" near "deposit"
moves toward the financial-institution direction — this is exactly the disambiguation from Section 2,
now visualized as vectors repositioning themselves based on their neighbors.

---

## 6. Multi-Head Attention

A single self-attention computation learns *one* notion of "relevance" — one way of scoring which
tokens matter to which. But language has many simultaneous kinds of relationships: syntactic
(subject–verb agreement), semantic (which noun a pronoun refers to), positional (nearby words often
matter more), and more. Forcing all of that into one attention pattern is a bottleneck.

**Multi-head attention runs several self-attention computations in parallel — "heads" — each with its
own independently learned `W_Q`, `W_K`, `W_V`, then concatenates the results and projects them back
down** to the original dimensionality with one more learned weight matrix:

```
head_i = Attention(X·W_Q_i, X·W_K_i, X·W_V_i)     for i = 1 … h
MultiHead(X) = Concat(head_1, …, head_h) · W_O
```

Each head is free to specialize in a different kind of relationship purely by gradient descent finding
different useful projections for each — nobody hand-assigns "head 3 does syntax." Empirically, when
attention weights are visualized, different heads do learn to attend to noticeably different patterns
(some heads track adjacent words, others track long-range dependencies, others track specific
grammatical relations). The original Transformer uses 8 heads; larger modern models use more.

**Cost:** if the model's total embedding dimension is `d_model` and there are `h` heads, each head
typically works with dimension `d_model / h`, so multi-head attention's total compute is comparable to
one full-size attention computation — you're not paying `h` times the cost, you're splitting the same
budget `h` ways and running them in parallel.

---

## 7. Positional Encoding

There's a structural problem with everything above: self-attention, as defined, is **permutation-
invariant** — it computes a weighted sum over all tokens using only their content (via Q/K/V), with no
notion of *order*. Shuffle the words in a sentence and self-attention alone would produce the same set
of pairwise relationships, just reassigned to different positions. But word order obviously matters:
"dog bites man" and "man bites dog" have the same *tokens* and very different *meaning*. An RNN gets
order "for free" because it processes tokens sequentially; a Transformer, having deliberately removed
recurrence for parallelism, has no such built-in signal and must be given one explicitly.

**Positional encoding adds information about each token's position in the sequence directly into its
input embedding**, before self-attention ever runs:

```
input_to_model = token_embedding + positional_encoding
```

The original Transformer paper uses a fixed (not learned) scheme built from sine and cosine functions
at different frequencies, one pair per embedding dimension:

```
PE(pos, 2i)   = sin(pos / 10000^(2i / d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i / d_model))
```

Two properties make this a good choice, not an arbitrary one:

- **Every position gets a unique encoding vector** (different `pos` → different combination of
  sin/cos values across dimensions), so the model can distinguish position 1 from position 2 from
  position 50.
- **Relative position is recoverable via a fixed linear relationship** — because of trigonometric
  identities, `PE(pos + k)` can be expressed as a linear function of `PE(pos)` for a fixed offset `k`.
  This means the model can learn to attend based on *relative* distance between tokens ("the word 3
  positions back"), not just absolute position, without needing to see every absolute position during
  training.

This encoding is added once, at the input, and then self-attention (and everything downstream) operates
on token+position information jointly — nothing later in the network needs to treat position
specially again. (Some later Transformer variants use *learned* positional embeddings instead of this
fixed sinusoidal scheme, or relative-position schemes baked directly into the attention computation
itself — implementation details that don't change the core requirement: **some** explicit position
signal is mandatory, because self-attention itself has none.)

---

## Key takeaways

- **Self-attention** lets every token attend to every other token in the *same* sequence, producing
  **contextualized representations** — the same word gets a different vector depending on the words
  around it, fixing the "bank" (riverbank vs. financial) ambiguity that static embeddings can't.
- Every token gets a **query, key, value** triplet via three independently learned projections
  (`W_Q`, `W_K`, `W_V`); query = what I'm looking for, key = how I'm matched against, value = what I
  hand over once matched.
- **Scaled dot-product attention:** `softmax(Q·Kᵗ / sqrt(d_k)) · V`. The `sqrt(d_k)` scaling factor
  keeps raw dot-product scores from growing too large with dimensionality and pushing softmax into its
  saturated, near-zero-gradient regions — a vanishing-gradient issue (Section 06) showing up inside the
  attention mechanism itself.
- **Geometrically:** attention pulls each token's representation toward the value vectors of the
  tokens whose key vectors align most closely with its query vector, in embedding space.
- **Multi-head attention** runs several independent self-attention computations in parallel, each free
  to specialize in a different kind of relationship (syntax, coreference, position, etc.), then
  concatenates and projects the results back down — same total compute budget, split `h` ways.
- **Positional encoding** is necessary because self-attention alone is permutation-invariant and has no
  notion of order; a fixed sinusoidal signal (unique per position, and linearly related across relative
  offsets) is added to token embeddings before attention runs, so position information is available
  everywhere downstream.

**Next:** [20 — Transformer Architecture](20-transformers-architecture.md) — layer normalization, the
full encoder and decoder stacks, masked and cross-attention, and how a trained Transformer actually
generates output.
