# 20 — Transformer Architecture

**Playlist videos 79–84.** This is the capstone. Section 19 built self-attention, scaled dot-product
attention, multi-head attention, and positional encoding — the pieces. This section assembles them into
the actual **Transformer**: layer normalization (the normalization scheme Transformers actually use),
the full **encoder** stack, **masked self-attention** and **cross-attention** (the two attention
variants unique to the **decoder**), the full **decoder** stack, and finally how a *trained*
Transformer generates output at **inference** time. By the end you'll have the complete architecture
from *"Attention Is All You Need"* end to end — and the guide's through-line closes here.

---

## 1. Layer Normalization (Video 79)

Before assembling the encoder/decoder, one more building block: Transformers normalize activations
inside every layer, but with **Layer Normalization (LayerNorm)**, not the **Batch Normalization**
from Section 10.

### What LayerNorm normalizes

Batch Norm (Section 10) computes mean and variance **across the batch dimension, for each feature
independently** — i.e., for one feature, it looks at that feature's values across all *examples* in
the mini-batch. LayerNorm instead computes mean and variance **across the feature dimension, for each
example independently** — for one token's embedding vector, it looks across that vector's own
dimensions, ignoring every other example (or token) in the batch entirely.

```
LayerNorm:  normalize x̂ = (x − μ) / sqrt(σ² + ε),  then y = γ·x̂ + β
            — μ, σ² computed over the FEATURES of ONE example, independent of the batch.

BatchNorm:  μ, σ² computed over the BATCH, for ONE feature, independent of the other features.
```

The formula (Section 10's Steps 3–4) is identical — normalize, then apply learnable scale `γ` and
shift `β`. What changes is **which axis you average over**.

### Why Transformers need LayerNorm, not BatchNorm

Section 10 already flagged the batch-size dependency that motivates this. Three concrete reasons it
bites specifically in Transformers:

- **Variable sequence lengths.** A batch of sentences has different lengths, usually handled with
  padding. Batch Norm's statistics, computed per-position across the batch, get distorted by padding
  tokens and by the fact that "position 47" might be real content in one sentence and padding in
  another. LayerNorm normalizes within a single token's own feature vector, so sequence length and
  padding are irrelevant to it.
- **Independent of batch size.** Batch Norm's statistics get noisier as batch size shrinks (Section
  10, §3). LayerNorm's statistics never depend on the batch at all — computed per example, they're
  identical whether the batch size is 1 or 1000.
- **Works at inference on a single sequence.** Batch Norm needs running statistics accumulated during
  training to handle single-example inference correctly (Section 10, §4) — an extra training/inference
  mode switch to get right. LayerNorm needs no such switch: it computes the same per-example statistics
  at inference as it did at training, because it never depended on the batch to begin with. This matters
  enormously for autoregressive generation (§6 below), where the decoder genuinely does process one
  growing sequence at a time.

| | **Batch Normalization** (Section 10) | **Layer Normalization** |
|---|---|---|
| Normalizes across | The **batch**, per feature | The **features**, per example |
| Depends on batch size | Yes — noisy at small batch sizes | No — computed per example |
| Needs train/inference statistics switch | Yes (running mean/var) | No — same computation always |
| Handles variable-length sequences | Poorly (padding distorts per-position batch stats) | Naturally — each token normalized independently |
| Typical home | CNNs, MLPs (fixed-size batches of images/vectors) | Transformers, RNNs (sequences, variable length) |

### Pre-norm vs. post-norm

The original Transformer paper applies LayerNorm **after** each sub-layer's residual addition —
`LayerNorm(x + Sublayer(x))`, called **post-norm**. Many later Transformer implementations instead
apply LayerNorm **before** each sub-layer — `x + Sublayer(LayerNorm(x))`, called **pre-norm** — because
it produces more stable gradients early in training for very deep stacks, at a small cost in final
performance in some settings. Both are seen in practice; the diagrams in this section use the
original post-norm placement, but treat the choice as an implementation detail, not a change to any of
the mechanisms themselves.

---

## 2. Transformer Encoder Architecture (Video 80)

The encoder's job, same as Section 18's encoder: turn an input sequence into a rich representation. The
Transformer encoder does this with **self-attention instead of recurrence**.

### Input

Every input token is converted to an embedding, then Section 19's **positional encoding** is added
directly to it — `input = token_embedding + positional_encoding` — before anything else happens. This
is the *only* place positional information enters the model; every layer after this operates on
embeddings that already carry position information mixed in.

### One encoder layer

The encoder is a **stack of N identical layers** (N = 6 in the original paper), each with exactly two
sub-layers:

```
   input
     |
     v
[ Multi-Head Self-Attention ]     (Section 19 — every token attends to every other token in this sequence)
     |
     v
[ Add & Norm ]                    (residual connection + LayerNorm)
     |
     v
[ Position-wise Feed-Forward Network ]
     |
     v
[ Add & Norm ]
     |
     v
   output  → fed as input to the NEXT encoder layer (or to the decoder's cross-attention, once N layers deep)
```

- **Multi-Head Self-Attention** — Section 19's mechanism, run over the input sequence to itself. This
  is what lets every position build a context-aware representation using every other position in the
  same input.
- **Add & Norm.** "Add" is a **residual/skip connection**: the sub-layer's output is added to its own
  input (`x + Sublayer(x)`), not used to replace it. This is the same idea Section 06 introduced as one
  of the fixes for vanishing gradients — a residual connection gives the gradient a direct, unobstructed
  path backward through the "+x" term, regardless of how deep the stack gets, so a 6- or 12-layer
  Transformer stack doesn't inherit the vanishing-gradient problem that plain deep stacks would. "Norm"
  is LayerNorm (§1), applied after the addition, to keep the resulting activations well-scaled before
  they enter the next sub-layer.
- **Position-wise Feed-Forward Network (FFN).** A small two-layer MLP — `Linear → non-linearity (ReLU
  or similar) → Linear` — applied **independently to every token position**, with the *same* weights
  reused at every position (it's "position-wise" because it doesn't mix information across positions at
  all — that's self-attention's job; the FFN's job is to further transform each position's own vector).
  It typically expands to a larger hidden dimension internally (e.g., 512 → 2048 → 512) before
  projecting back down, giving the layer extra representational capacity beyond what attention alone
  provides.

### What the encoder produces

After N layers of self-attention + FFN, each with residual connections and LayerNorm, the encoder
outputs one vector per input token — a **contextualized representation of the entire input sequence**,
where every token's vector has been informed by every other token, refined N times over. This output is
what the decoder's cross-attention (§4) will read from, at every decoder layer.

---

## 3. Masked Self-Attention (Video 81)

The decoder also uses self-attention over its own (output) sequence — but with one critical
difference from the encoder's self-attention: it must not be allowed to see the future.

### The problem: peeking during training

The decoder generates output tokens one position at a time: token 1, then token 2 using token 1, then
token 3 using tokens 1–2, and so on — exactly the autoregressive generation from Section 18. But during
**training**, Section 18's **teacher forcing** feeds the entire target sequence into the decoder at
once, for efficiency (so the whole sequence can be trained in parallel instead of one token at a time).

If the decoder's self-attention were left unrestricted, when computing the representation for output
position *t* it would be able to attend to positions *t+1, t+2, …* — tokens that, at real generation
time, **don't exist yet**. The model would learn to "cheat" by copying from the future, which it will
never have access to during actual inference. This would make training deeply misleading: the model
would appear to solve the task by peeking, then fail completely once inference forces it to generate
genuinely left-to-right.

### The fix: a causal (look-ahead) mask

**Masked self-attention adds a mask to the raw attention scores — before softmax — that sets every
score for a "future" position to `−∞`.**

```
scores = Q · Kᵗ / sqrt(d_k)
scores[t, j] = −∞   for every j > t     (mask out all future positions)
weights = softmax(scores)               (−∞ → exactly 0 weight after softmax)
```

Because `softmax(−∞) = 0`, masked positions contribute **exactly zero** to the weighted sum over
values — position *t*'s output is a weighted combination only of positions `1 … t`, never `t+1`
onward. This is called a **causal mask** or **look-ahead mask**: causal because output position *t*
can only depend on causes (inputs) at or before *t*, matching how generation will actually happen at
inference.

### Why this specific trick, and not something else

The mask is applied to the *scores*, before softmax, rather than zeroing out the *weights* after
softmax — this matters, because if you zeroed weights after softmax, the remaining weights wouldn't
sum back to 1 without extra renormalization. Masking with `−∞` before softmax makes the exclusion of
future positions and the renormalization of the remaining weights happen in the same softmax step,
automatically.

### Why this enables parallel training

This is the payoff: because the mask is just a static pattern (a triangular matrix of `−∞`s applied to
every training example, computable once per batch), the decoder's self-attention for **all** output
positions can be computed **simultaneously** in one matrix operation — each position's attention just
happens to be blind to the future by construction. Training therefore keeps the parallelism benefit of
teacher forcing (Section 18) *and* respects the causal generation order the model will actually be
constrained to at inference. Without masking, teacher-forcing-in-parallel would train a model that
cheats; without teacher forcing, training would have to run token by token like RNN training and lose
the whole point of dropping recurrence.

---

## 4. Cross-Attention (Video 82)

The decoder needs one more piece: a way to actually look at the **input** sequence, not just its own
partial output. That's **cross-attention** — the decoder's second attention block.

### Q from the decoder, K/V from the encoder

Self-attention (Section 19, and §3 above) always has Q, K, and V come from **the same sequence**.
Cross-attention breaks that symmetry deliberately:

```
Query  ← from the DECODER's own (masked self-attention) output
Key    ← from the ENCODER's final output
Value  ← from the ENCODER's final output
```

Every decoder position asks "given what I've generated so far, which parts of the *input* sequence are
relevant to what I should generate next?" — and the answer is computed by scoring the decoder's query
against every encoder position's key, exactly the scaled dot-product mechanics of Section 19, just with
Q coming from one sequence and K/V from another.

### This is Section 18's attention, generalized

This is precisely the mechanism Section 18 introduced — decoder state as query, encoder states as
keys/values — described there as the direct ancestor of Transformer attention (and specifically
resembling Luong's dot-product scoring). Cross-attention *is* that same encoder-decoder attention,
computed with the scaled dot-product formula and, in the full Transformer, run as multi-head attention
just like self-attention is.

| | **Self-attention** (Section 19; also decoder §3) | **Cross-attention** |
|---|---|---|
| Query from | Same sequence | **Decoder's** own (masked) representation |
| Key/Value from | Same sequence | **Encoder's** output |
| Purpose | Relate tokens within one sequence to each other | Let the decoder pull relevant information from the *input* sequence |
| Masking | Encoder: none. Decoder: causal mask (§3). | No causal masking needed — the full encoder output is always visible; there's no "future" concept over the input |

---

## 5. Transformer Decoder Architecture (Video 83)

Like the encoder, the decoder is a **stack of N identical layers**, but each layer has **three**
sub-layers instead of two:

```
   decoder input (previous output tokens, embedded + positional encoding)
     |
     v
[ Masked Multi-Head Self-Attention ]      (§3 — attend to decoder's own prior output only, causal mask)
     |
     v
[ Add & Norm ]
     |
     v
[ Cross-Attention  ]                      (§4 — Q from decoder, K/V from ENCODER's output)
     |
     v
[ Add & Norm ]
     |
     v
[ Feed-Forward Network ]                  (same position-wise FFN as the encoder, §2)
     |
     v
[ Add & Norm ]
     |
     v
   output → next decoder layer (or, after N layers, the final Linear + Softmax)
```

Every one of the three sub-layers is wrapped in the same **residual + LayerNorm** pattern from §1–2 —
gradient flow and normalization work identically whether the sub-layer is self-attention,
cross-attention, or the FFN.

### The final output layer

After N decoder layers, one more step turns the decoder's final vector (per output position) into an
actual predicted token: a **Linear layer** projects it to the size of the vocabulary, and a **Softmax**
turns that into a probability distribution over every possible next token. The predicted token is
whichever one that distribution assigns the highest probability to (at least under greedy decoding —
more on decoding strategies in §6).

### How encoder and decoder connect

The **encoder runs once**, over the whole input sequence, producing one final output (a sequence of
vectors, one per input position) after its N layers. That single encoder output is then fed as the
K/V source into **every one of the decoder's N layers' cross-attention sub-layers** — not just the
last decoder layer. Every decoder layer, at every depth, gets to look back at the full encoder output
independently; the decoder's own self-attention (masked) and its cross-attention into the encoder are
two separate, alternating channels of information within each layer.

---

## 6. Transformer Inference (Video 84)

Training and inference use the *same* trained weights, but they run the model very differently — this
distinction (foreshadowed by teacher forcing in Section 18 and the masking discussion in §3) is worth
making completely explicit now that the whole architecture is on the table.

### Training: parallel, with teacher forcing

During training, the entire target sequence is known in advance. The decoder receives the whole
target sequence at once (shifted by one position, with a start token prepended), masked self-attention
(§3) ensures no position can see beyond itself, and the loss compares the predicted-next-token
distribution at **every** position simultaneously against the true next token. One forward pass, one
backward pass, covers the entire sequence's worth of next-token predictions — this is the parallelism
payoff of dropping recurrence and using masking instead.

### Inference: sequential and autoregressive

At inference time, there is no target sequence to look ahead into — the model is generating output
that doesn't exist yet. So generation reverts to the **autoregressive** loop from Section 18, run
concretely as:

1. **Encode once.** Run the full input sequence through the encoder a single time. Its output is
   reused, unchanged, for every decoding step that follows — no need to re-encode the input as
   generation proceeds.
2. **Start.** Feed the decoder a `<START>` token (and the encoder's output, via cross-attention).
3. **Generate one token.** The decoder produces a probability distribution over the vocabulary for
   the next position; pick a token from it (§ below on how).
4. **Append and repeat.** Append that token to the decoder's input sequence so far, feed the *growing*
   sequence back through the decoder (masked self-attention still applies, now over the real tokens
   generated so far), generate the next token, and repeat.
5. **Stop.** When the decoder generates an `<END>` token, or a maximum length is reached, stop.

The key asymmetry: **the encoder runs once; the decoder runs once per output token.** This is why
generation latency for a Transformer scales with output length, even though the whole input was
processed in one parallel pass.

### Decoding strategies — how step 3 actually picks a token

The softmax at each step gives a full probability distribution, not a single answer — several
strategies exist for turning that into an actual token, each trading off quality, diversity, and cost:

- **Greedy decoding.** Always pick the single highest-probability token. Fast and deterministic, but
  myopic — a locally-best token at step *t* can lock the model into a globally worse sequence, since
  there's no way to reconsider once committed.
- **Beam search.** Keep the top-*k* most probable partial sequences ("beams") at every step instead of
  just one, expanding each and pruning back down to the top *k* by total sequence probability at the
  end. More likely to find a higher-quality overall sequence than pure greedy, at *k* times the compute.
- **Sampling / temperature.** Instead of always taking the max, sample the next token from the
  probability distribution itself, optionally sharpened or flattened by a **temperature** parameter
  (low temperature → closer to greedy/deterministic; high temperature → more random/diverse output).
  Useful when some variety in output is desirable (creative generation) rather than a single "best"
  answer.
- **Top-k / top-p (nucleus) sampling.** Restrict sampling to only the *k* most probable tokens
  (top-k), or to the smallest set of tokens whose cumulative probability exceeds threshold *p*
  (top-p/nucleus), before sampling — this avoids the small but nonzero chance of sampling an
  absurdly unlikely token that plain temperature sampling can't rule out.

None of these change the architecture — they only change how the final softmax distribution at each
step gets turned into a chosen token.

---

## 7. Putting It All Together

The full encoder-decoder Transformer, end to end:

```
INPUT SEQUENCE                                    OUTPUT SEQUENCE (generated so far)
     |                                                    |
token embedding + positional encoding          token embedding + positional encoding
     |                                                    |
     v                                                    v
[ Encoder layer 1..N ]                          [ Decoder layer 1..N ]
  Self-Attn → Add&Norm                            Masked Self-Attn → Add&Norm
  FFN       → Add&Norm                            Cross-Attn (K,V from encoder) → Add&Norm
     |                                             FFN → Add&Norm
     v                                                    |
  ENCODER OUTPUT  ------------(feeds every decoder layer's cross-attention)---->
                                                           v
                                                  Linear → Softmax → next-token probabilities
```

- **Encoder:** self-attention (Section 19) + FFN, N layers, run once over the input. Produces
  contextualized input representations.
- **Decoder:** masked self-attention (causal, §3) + cross-attention into the encoder (§4) + FFN, N
  layers, run once per output token at inference (in parallel, teacher-forced, during training).
  Produces next-token probabilities.
- **LayerNorm + residual connections** wrap every sub-layer in both stacks, keeping the whole deep
  stack trainable (§1–2, tying back to Sections 06 and 10).

### Real-world variants

Not every Transformer-based model uses both halves:

- **Encoder-only (e.g., BERT).** Just the encoder stack — no causal masking anywhere, so every token
  can see the whole input, including "future" tokens. Good for tasks that consume a whole input and
  produce one judgment or set of labels for it (classification, tagging, embeddings) rather than
  generating new text.
- **Decoder-only (e.g., GPT).** Just the decoder stack, with masked self-attention and no
  cross-attention sub-layer at all (there's no separate encoder output to cross-attend to) — the model
  is purely autoregressive over one sequence. This is the architecture behind modern large language
  models: given text so far, predict the next token, repeatedly.
- **Encoder-decoder (e.g., T5, and the original 2017 translation Transformer this whole guide has been
  building toward).** Both halves, exactly as described above — suited to tasks that genuinely
  transform one sequence into a different one (translation, summarization).

This maps directly back to Section 01's "types of neural networks" framing and Section 18's history:
the Transformer isn't one fixed architecture so much as a **kit of interchangeable attention-based
parts** (self-attention, masked self-attention, cross-attention, LayerNorm'd residual stacks), and
which parts you keep depends on whether the task is understanding a sequence, generating one
autoregressively, or transforming one into another.

---

## Where to go next

This guide's through-line — a flexible function, a way to train it, tricks to make training actually
work, then architecture after architecture specialized to a data shape — ends here, at the Transformer.
It doesn't cover everything the field has moved on to since 2017, and it shouldn't try to fake depth on
topics this guide hasn't earned:

- **Large language models** — decoder-only Transformers (§7) scaled up by orders of magnitude in
  parameters and training data. The *architecture* is exactly what's in this section; the interesting
  new material is training-corpus scale, distributed training engineering, and emergent behavior at
  scale — a different subject from "how does a Transformer work."
- **Fine-tuning and transfer learning** — taking a pretrained Transformer (BERT, GPT, T5) and adapting
  it to a specific task or domain, echoing Section 15's transfer-learning treatment for CNNs, but for
  language models.
- **RLHF (reinforcement learning from human feedback)** — how models like ChatGPT are aligned to
  follow instructions and match human preference, on top of the base language-modeling objective this
  guide covers. A genuinely different training paradigm, not an architecture change.
- **Diffusion models** — the dominant modern approach for image generation, built on a completely
  different mechanism (iterative denoising) than anything in this guide. Some diffusion architectures
  use Transformer *blocks* internally, but the training objective and generation process are unrelated
  to autoregressive next-token prediction.

Each of those is a real, substantial subject in its own right — worth a guide each, not a paragraph
here.

---

## Key takeaways

- **LayerNorm**, not BatchNorm, normalizes Transformers: across a single example's own features, not
  across the batch — sidesteps batch-size dependency, works with variable-length sequences, and needs
  no train/inference statistics switch. Pre-norm vs. post-norm is a placement detail, not a different
  mechanism.
- **Encoder layer** = Multi-Head Self-Attention → Add & Norm → Position-wise FFN → Add & Norm, stacked
  N times, over input embeddings + positional encoding (Section 19). Residual connections fight
  vanishing gradients (Section 06) across the depth of the stack.
- **Masked self-attention** in the decoder sets future-position scores to `−∞` before softmax (a
  causal/look-ahead mask), so position *t* only ever attends to positions `≤ t` — necessary because
  autoregressive generation can never see the future, and this masking is what lets the decoder still
  be trained in parallel via teacher forcing (Section 18) without cheating.
- **Cross-attention** is the decoder's second attention block: Query from the decoder, Key/Value from
  the encoder's output — the modern, scaled-dot-product form of Section 18's Bahdanau/Luong
  encoder-decoder attention, contrasted with self-attention where Q/K/V all come from the same sequence.
- **Decoder layer** = Masked Multi-Head Self-Attention → Add & Norm → Cross-Attention → Add & Norm →
  FFN → Add & Norm, stacked N times, ending in a Linear + Softmax over the vocabulary. The encoder's
  output feeds every decoder layer's cross-attention, not just the last one.
- **Inference is autoregressive and sequential** (encode once, then decode one token at a time,
  feeding each generated token back in until `<END>` or max length) — contrast with **parallel,
  teacher-forced training**, where the whole target sequence is processed in one masked pass. Decoding
  strategies (greedy, beam search, temperature sampling, top-k/top-p) control how the softmax output at
  each step becomes an actual chosen token.
- Real models mix and match the two halves: **encoder-only** (BERT, understanding tasks),
  **decoder-only** (GPT, autoregressive generation/LLMs), **encoder-decoder** (T5, translation — the
  original 2017 architecture this guide has built up to).

**Next:** back to the [guide index](README.md) — you've reached the end of the through-line.
