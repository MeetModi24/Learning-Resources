# 18 — Seq2Seq and Attention

**Playlist videos 67–70.** This section is the bridge between the recurrent world (Sections 16–17)
and the Transformer world (Sections 19–20). It covers the encoder-decoder (seq2seq) architecture
that made variable-length translation possible with RNNs/LSTMs, the fixed-size-context bottleneck
that limited it, and the attention mechanism invented to fix that bottleneck — which then turned out
to be powerful enough to replace recurrence altogether.

---

## 1. A Little History: From LSTMs to Transformers (Video 67)

Section 01 sketched the big-picture DL timeline ending at "2017 — Attention Is All You Need." Here's
the more detailed arc that leads there, since every step in it directly motivates the next:

- **RNNs (Section 16)** could process sequences, but suffered vanishing/exploding gradients over long
  sequences.
- **LSTMs and GRUs (Section 17)** fixed the long-range memory problem with gates — but they're still
  fundamentally sequential: step *t* can't be computed until step *t-1* finishes. That's slow to
  train and still struggles with very long sequences.
- **Seq2Seq (2014, this section)** wired two RNNs/LSTMs together — an encoder and a decoder — to
  handle inputs and outputs of *different* lengths (e.g., a 10-word English sentence → an 8-word
  French sentence). This unlocked machine translation as a mainstream deep learning task.
- **Attention (2014–2015, this section)** fixed seq2seq's single-context-vector bottleneck by letting
  the decoder look back at *all* encoder states, weighted by relevance. This was the single biggest
  quality jump in neural machine translation at the time.
- **Transformers (2017, Section 19–20)** asked: if attention is doing all the real work, why keep the
  slow, sequential recurrence around at all? They dropped RNNs entirely and built a model purely out
  of attention layers — fully parallelizable, and dramatically better at long-range dependencies.
- **BERT, GPT, ChatGPT (2018–present)** are Transformer variants (encoder-only, decoder-only, and
  scaled-up decoder-only, respectively) trained on massive text corpora. Every one of them inherits
  the attention mechanism this section introduces.

The one-sentence version: **seq2seq introduced the encoder/decoder split; attention removed its
bottleneck; the Transformer then removed the recurrence that attention no longer needed.** The rest
of this section builds that chain step by step.

---

## 2. Encoder–Decoder (Seq2Seq) Architecture (Video 68)

### The problem it solves

Plain RNN/LSTM classification or tagging assumes input and output are aligned one-to-one, or at least
the same length. Many real tasks don't work that way:

- **Machine translation:** "how are you" (3 words) → "comment vas-tu" (3 words in French, but not
  always — length varies by language and sentence).
- **Summarization:** a 500-word article → a 2-sentence summary.
- **Question answering / chatbots:** variable-length question → variable-length answer.

You need an architecture that decouples input length from output length entirely. That's what
seq2seq does.

### The architecture

Two separate RNNs/LSTMs, chained together:

```
Input:  "how"  "are"  "you"
          |      |      |
        [Encoder LSTM, one cell per timestep]
          |      |      |
         h1  →   h2  →  h3            (h3 = final hidden state = CONTEXT VECTOR)
                          |
                          v
              [Decoder LSTM, initialized with context vector]
                          |
                 <START> → "comment" → "vas" → "tu" → <END>
                (each decoder output is fed back in as the next input)
```

- **Encoder:** an RNN/LSTM that reads the entire input sequence, one token at a time, and produces a
  hidden state at each step. It doesn't produce any output tokens — its only job is to compress the
  whole input into its **final hidden state** (and cell state, for LSTM). That final state is called
  the **context vector**: a single fixed-size vector meant to summarize the entire input sentence's
  meaning.
- **Decoder:** a second RNN/LSTM, **initialized with the encoder's context vector** as its starting
  hidden state. It then generates the output sequence one token at a time, starting from a `<START>`
  token, and feeding its own predicted output back in as the input to the next timestep — this is
  called **autoregressive generation**. It stops when it emits an `<END>` token.

This is why it's called encoder-**decoder**: the encoder *encodes* the input into a compressed
representation, and the decoder *decodes* that representation back out into a (possibly
different-length, possibly different-language) output sequence.

### Teacher forcing

During **training**, if you let the decoder feed its own (early, mostly-wrong) predictions back into
itself, one bad prediction early in the sentence poisons every prediction after it, and gradients
become noisy and slow to converge. **Teacher forcing** avoids this: at each decoder timestep during
training, you feed in the *actual* ground-truth previous token (from the target sentence) instead of
the model's own prediction, regardless of what the model actually predicted. The model still trains to
predict the next token correctly; it just isn't punished by its own earlier mistakes compounding.

At **inference** time there's no ground truth to feed in, so the decoder genuinely is autoregressive —
each predicted token becomes the next input, exactly as in the diagram above.

---

## 3. The Bottleneck Problem

Plain seq2seq has one glaring weakness: **the entire input sequence — no matter how long — is
squeezed into one fixed-size context vector** (the encoder's final hidden state).

Think about what that means for a 3-word sentence versus a 50-word paragraph: both get compressed
into the *same-size* vector. For short sequences this is barely a problem — a 3-word sentence's
meaning fits fine into a few hundred numbers. But as the input gets longer, you're asking that same
fixed-size vector to hold proportionally more information, and it can't. Early tokens in a long
sequence get diluted or overwritten by the time the encoder finishes processing later tokens.

**The empirical result:** translation quality (measured by BLEU score in the original papers) holds
up fine for short sentences and **degrades sharply as sentence length increases**. The model has
nothing wrong with its language modeling — it simply never received enough information about the
early part of a long input, because that information never survived being crammed into one vector.

This is exactly the motivation for attention: instead of forcing the decoder to work off a single
compressed summary, **give it access to every encoder hidden state, and let it decide — dynamically,
per output word — which of those states matter right now.**

---

## 4. Attention Mechanism (Video 69)

### The core idea

Instead of the decoder using one fixed context vector for the whole output sequence, **attention
computes a fresh, custom context vector at every single decoder timestep** — a weighted combination
of *all* the encoder's hidden states, not just the last one.

Concretely, at decoder timestep *t*:

1. **Score** every encoder hidden state against the decoder's current state, producing one raw
   "alignment score" per input position — how relevant is input word *i* to generating output word
   *t* right now?
2. **Softmax** those scores into **attention weights** that sum to 1 across all input positions.
3. **Weighted sum:** multiply each encoder hidden state by its attention weight and add them up. This
   weighted sum *is* the context vector for this specific decoder step.
4. Feed that context vector (along with the decoder's own state) into the step that predicts the next
   output token.

Then repeat all four steps at the *next* decoder timestep — with a different set of weights, because
a different output word usually depends on different input words.

### The translation intuition

When translating "how are you" → "comment vas-tu", a human translator's attention naturally shifts
word by word: focus on "you" when producing "tu", focus on "are" when producing "vas". Attention
gives the model exactly that same ability — mechanically, via softmax weights — instead of forcing it
to remember everything at once and hope the right piece survives to the end.

This directly fixes the bottleneck: every encoder hidden state remains individually accessible for
the entire decoding process. Nothing gets overwritten or diluted by later tokens, and there's no
single-vector ceiling on how much input information the decoder can draw on.

### Query, key, value — a preview

You can describe the same computation with a small vocabulary that Section 19 will build on directly:

- The decoder's current hidden state acts as the **query** — "what am I looking for right now?"
- Each encoder hidden state acts as both a **key** (what it's scored against) and a **value** (what
  actually gets summed into the context vector once weighted).
- The alignment/scoring step is "compare query to every key"; softmax turns those comparisons into
  weights; the weighted sum over values is the output.

In this section keys and values happen to be the *same* vectors (the encoder states). Self-attention
in Section 19 generalizes this by learning separate query, key, and value projections for every token
in a single sequence, rather than only between a decoder and an encoder — but the query/key/value
mechanics you'll see there are exactly this idea, one abstraction layer higher.

### Attention weights are interpretable

Because the weights at each decoder step sum to 1 and are indexed by input position, you can plot them
as a heatmap — decoder output words on one axis, encoder input words on the other, weight as color
intensity. This produces an **alignment map**: a visual, human-checkable picture of which input words
the model actually used to generate each output word. This was one of attention's most compelling
selling points early on — unlike the single opaque context vector, you could *see* what the model was
doing and sanity-check it against how a human translator would align the same sentence pair.

---

## 5. Bahdanau vs Luong Attention (Video 70)

Attention wasn't introduced once — it was introduced as one idea with two influential variants that
differ in *when* the context vector is computed relative to the decoder step, and *how* the alignment
scores are computed.

| | **Bahdanau Attention** (2014) | **Luong Attention** (2015) |
|---|---|---|
| Also known as | Additive attention | Multiplicative attention |
| Decoder state used for scoring | **Previous** decoder hidden state (*h*ₜ₋₁) | **Current** decoder hidden state (*h*ₜ) |
| Scoring function | A small **feed-forward network** (concatenate query + key, pass through a learned layer) — "additive" | **Dot product** (or a simple learned bilinear form) between query and key — "multiplicative" |
| When context is computed | **Before** the decoder produces its output for this step (context feeds into the decoder cell) | **After** the decoder has already updated its hidden state (context is combined with it to produce the final prediction) |
| Compute cost | Higher — a full feed-forward network per score | Lower — just a dot product, much faster |
| Variants | One general form | Offers **global** (attend to all encoder states, like Bahdanau) and **local** (attend to only a small window of encoder states, cheaper for very long inputs) |
| Historical note | The original attention paper — first to solve the seq2seq bottleneck | Simplified and popularized the idea; the dot-product scoring is closer to what Transformers eventually use |

**Practical takeaway:** the two are more alike than different — same core weighted-sum-over-encoder-
states idea — and the choice mostly comes down to a compute/simplicity trade-off. Bahdanau's additive
scoring is more expressive but slower; Luong's dot-product scoring is cheaper and, not coincidentally,
is the direct ancestor of the **scaled dot-product attention** used inside every Transformer layer in
Section 19.

---

## 6. The Handoff to Transformers

Attention solved the seq2seq bottleneck so effectively that it raised an obvious question: if the
attention mechanism is doing the heavy lifting of relating input and output tokens, **how much is the
underlying recurrence (the RNN/LSTM doing the encoding and decoding) still contributing?**

The 2017 paper *"Attention Is All You Need"* answered that question by removing the recurrence
entirely and building a model made **purely of attention layers** (applied to a sequence's own tokens
against each other — self-attention — plus the same encoder-decoder cross-attention seen here). With
no recurrence, there's no more step-by-step sequential dependency, so the whole sequence can be
processed **in parallel** rather than one timestep at a time — a massive speed and scalability win
that is a large part of why Transformers could be scaled up to the sizes behind BERT, GPT, and
ChatGPT. That architecture is Section 19's subject.

---

## Key takeaways

- **Seq2seq (encoder-decoder)** handles variable-length input → variable-length output tasks: an
  encoder RNN/LSTM compresses the whole input into a final hidden state (**context vector**); a
  decoder RNN/LSTM, initialized with that vector, generates output tokens one at a time,
  autoregressively feeding its own previous output back in. **Teacher forcing** feeds ground-truth
  tokens during training instead, so early mistakes don't compound.
- **The bottleneck problem:** cramming an entire input sequence into one fixed-size context vector
  loses information — especially for long sequences — and translation quality degrades sharply as
  input length grows. This is *the* reason attention was invented.
- **Attention** replaces the single context vector with a **per-decoder-step weighted combination of
  all encoder hidden states**, where softmax-normalized attention weights say how relevant each input
  position is to the output word being generated right now. This removes the bottleneck and enables
  long-range alignment; the weights are interpretable as alignment heatmaps.
- The query/key/value framing (decoder state = query, encoder states = keys/values) previews
  **self-attention** in Section 19.
- **Bahdanau (additive, uses previous decoder state, context before the decoder step)** vs **Luong
  (multiplicative/dot-product, uses current decoder state, context after the decoder step, global/
  local variants)** — different scoring mechanics, same core idea. Luong's dot-product scoring is the
  direct ancestor of Transformer attention.
- Attention proved powerful enough that the next step was to **drop recurrence entirely** and build a
  model purely from attention — the Transformer.

**Next:** [19 — Transformers & Self-Attention](19-transformers-self-attention.md) — dropping
recurrence and building a model purely out of attention.
