# 01 — Transformer Architecture

Goal: leave this doc able to draw the transformer from memory, explain what each box does, and describe what happens to a token vector as it flows through the stack.

---

## 1. The one-sentence summary

A transformer is a stack of layers where each layer updates every token's vector by letting that token **attend** to every other token, then passes the updated vector through a small feed-forward network. That's it. Everything else is engineering around those two operations.

---

## 2. Why transformers replaced RNNs

Before 2017, sequence models were **recurrent**: process token 1, pass a hidden state to process token 2, and so on. Two problems:

1. **Sequential**. You can't process token 5 until you've finished tokens 1–4. No parallelism.
2. **Long-range forgetting**. Information from token 1 had to survive 500 matrix multiplies to influence token 500. Gradients vanished; models forgot.

The transformer ("Attention Is All You Need", 2017) solved both:

1. **Parallel**. Every token in a sequence is processed at the same time during training.
2. **Direct long-range connections**. Token 500 attending to token 1 is **one operation**, not 500 hops.

The cost: attention is O(n²) in sequence length. More on that in doc 04.

---

## 3. The big picture (decoder-only)

Modern LLMs (GPT, Claude, Gemini, Llama) are **decoder-only** transformers. Ignore the "encoder" half of the original paper for now — we'll mention it in §11.

```
  Input tokens: [The, cat, sat, on, the]
                     │
                     ▼
              ┌──────────────┐
              │  Embedding   │  tokens → vectors (e.g., 4096-dim)
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │   Layer 1    │ ──┐
              ├──────────────┤   │
              │   Layer 2    │   │
              ├──────────────┤   │   N layers (32, 80, 120...)
              │     ...      │   │   Each layer is identical
              ├──────────────┤   │   in structure but has its
              │   Layer N    │   │   own weights.
              └──────┬───────┘ ──┘
                     ▼
              ┌──────────────┐
              │  Final norm  │
              └──────┬───────┘
                     ▼
              ┌──────────────┐
              │   Un-embed   │  vectors → token probabilities
              └──────┬───────┘
                     ▼
        Logits over the vocabulary (e.g., 100k values)
                     │
                     ▼
              Pick next token (sampling)
```

Each layer has the same two sub-components:

```
                  ┌─────────────────────────────────┐
                  │          Layer k                │
                  │                                 │
  input vectors ──┼──► [Attention] ─► [Add+Norm] ──┤──► output vectors
                  │          ▲              │       │
                  │          │              ▼       │
                  │          │       [Feed-Forward] │
                  │          │              │       │
                  │          │              ▼       │
                  │          └──────── [Add+Norm] ──┤
                  │                                 │
                  └─────────────────────────────────┘
```

So: **attention → feed-forward → attention → feed-forward → ...** N times. With residual connections (the "Add") and normalization at each step. That's the entire model shape. Everything interesting is *inside* those two blocks.

---

## 4. Token embedding

Every token in the vocabulary has a learned vector. If the vocab size is 100 000 and the model's hidden dimension is 4096, the embedding table is a 100 000 × 4096 matrix of learned parameters.

```python
# Conceptual: token_ids is a list of integers, each < vocab_size
embedding_table: Tensor[vocab_size, hidden_dim]  # learned parameters

# Lookup
token_vectors = embedding_table[token_ids]  # shape: [seq_len, hidden_dim]
```

At the start, the embedding table is random. During training, semantically similar tokens' vectors drift closer (the classic `king - man + woman ≈ queen` is a *consequence* of this, not how it's trained).

**Tie weights:** the embedding table and the final un-embedding layer often share weights (GPT-2 did this; many modern models still do). Saves parameters; also ties the output space to the input space.

---

## 5. The heart: attention

The single most important operation in a transformer. Everything else can be understood later; **attention has to be understood now**.

### 5.1 The intuition

For each token, attention answers: "which other tokens should I look at, and how much, to update my own representation?"

Example: in "The cat sat on the mat because **it** was warm", when processing **it**, attention should strongly weight **mat** (that's what's warm) and ignore most other tokens.

The model learns this — it's not hardcoded. And the "should look at" pattern changes by layer and by task.

### 5.2 Query, Key, Value

Each token's vector is projected into **three** different vectors through three learned weight matrices:

- **Query** (Q): "What am I looking for?"
- **Key** (K): "What do I contain?"
- **Value** (V): "What do I contribute if I'm attended to?"

```python
# For each token in the sequence, compute Q, K, V:
Q = X @ W_q   # [seq_len, d_head]
K = X @ W_k   # [seq_len, d_head]
V = X @ W_v   # [seq_len, d_head]
```

where `X` is the input to the attention block (shape `[seq_len, hidden_dim]`) and `W_q, W_k, W_v` are learned matrices.

### 5.3 The attention formula

$$
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{Q K^T}{\sqrt{d_k}}\right) V
$$

In English:

1. **`Q K^T`** — dot-product every query with every key. Result is a `[seq_len, seq_len]` matrix of raw similarity scores. Entry `(i, j)` = "how much should token `i` attend to token `j`?"
2. **`/ √d_k`** — scale so the softmax doesn't saturate at high dimensions. Numerical stability fix.
3. **`softmax`** — turn each row into a probability distribution summing to 1 (the attention weights).
4. **`@ V`** — weighted sum of value vectors. Each token's output is a mixture of all tokens' value vectors, weighted by its attention distribution.

Result shape: `[seq_len, d_head]` — same as `V`.

### 5.4 Causal mask (the "can only see the past" constraint)

In decoder-only LLMs, token 5 must not attend to token 6 — that would be cheating during training (predicting the next token while already seeing it).

Solution: before the softmax, set all entries above the diagonal to `-∞`:

```
        t1   t2   t3   t4   t5
  t1 [  ■   -∞   -∞   -∞   -∞ ]
  t2 [  ■    ■   -∞   -∞   -∞ ]
  t3 [  ■    ■    ■   -∞   -∞ ]
  t4 [  ■    ■    ■    ■   -∞ ]
  t5 [  ■    ■    ■    ■    ■ ]
```

After softmax, those `-∞` entries become 0 — so token `i` attends only to tokens `1..i`. This **causal mask** is what makes the model autoregressive.

### 5.5 Worked example

Let's attend three tokens with 2-dim vectors for legibility.

```
X = [ [1, 0],      # token 1
      [0, 1],      # token 2
      [1, 1] ]     # token 3

W_q = W_k = W_v = I (identity, for clarity)

Q = K = V = X

Q @ K.T =
  [ [1, 0, 1],
    [0, 1, 1],
    [1, 1, 2] ]

Apply causal mask and scale (d_k=2, √2 ≈ 1.41):
  [ [ 0.71, -∞,   -∞  ],
    [ 0,     0.71, -∞  ],
    [ 0.71,  0.71, 1.41] ]

Softmax each row:
  [ [1.0, 0.0, 0.0],
    [0.33, 0.67, 0.0],    # (approximate)
    [0.21, 0.21, 0.58] ]

(Attention weights) @ V →
  Token 3's new vector = 0.21*[1,0] + 0.21*[0,1] + 0.58*[1,1]
                       = [0.79, 0.79]
```

Token 3 ended up with a representation that's a blend of all three tokens' values, weighted by similarity. That's all attention is doing — a million times, in parallel, across layers.

---

## 6. Multi-head attention

One attention computation captures *one* pattern of "what to look at." But tokens have many simultaneous relationships: syntactic (subject-verb), semantic (pronoun-antecedent), positional (nearby tokens), etc. So we run attention **many times in parallel**, each with its own Q/K/V projection matrices, each learning a different pattern.

```
Hidden dim = 4096
Num heads = 32
Dim per head = 128   (4096 / 32)

Each "head":
  Q_h = X @ W_q_h  # shape [seq_len, 128]
  K_h = X @ W_k_h
  V_h = X @ W_v_h
  head_h = Attention(Q_h, K_h, V_h)  # [seq_len, 128]

Concatenate 32 heads:  [seq_len, 4096]
Project through W_o:   [seq_len, 4096]  ← final output of the attention block
```

Each head can specialize. Research has found heads that do things like "copy the previous token," "track the subject of the sentence," "attend to the first token," "track the ends of quoted strings." Interpretability work maps these out on small models; on frontier models, heads are harder to cleanly characterize.

**GQA (Grouped Query Attention) / MQA (Multi-Query Attention):** modern models often share the K and V projections across multiple Q heads. Llama uses GQA with 8 K/V heads serving 32 Q heads; saves memory on the KV cache (doc 05) with minimal quality loss.

---

## 7. Feed-forward network (FFN / MLP block)

After attention, each token's vector goes through a per-token feed-forward network. It's just a 2-layer MLP, but it does a lot of the "knowledge storage" work of the model.

```python
# Per token, independently:
h1 = activation(x @ W_1 + b_1)   # [hidden_dim] → [ffn_dim], often 4x expansion
out = h1 @ W_2 + b_2              # [ffn_dim]    → [hidden_dim]
```

- Input: `hidden_dim` (e.g., 4096)
- Hidden: `ffn_dim`, usually ~4× hidden (e.g., 16384, or 11008 for Llama-style)
- Output: back to `hidden_dim`

**Activation functions.** Original transformers used ReLU. Modern models use **SwiGLU** or **GeGLU** — a gated variant that empirically works better:

```python
# SwiGLU (Llama, Claude, many recent)
out = (Swish(x @ W_gate) * (x @ W_up)) @ W_down
# Swish(z) = z * sigmoid(z)
```

The FFN is where a huge fraction of the parameters live. In a typical LLM:
- Attention: ~1/3 of parameters
- FFN: ~2/3 of parameters

So when people say "GPT-4 has 1.8T parameters," most of those parameters are in FFN weight matrices.

**Mixture of Experts (MoE).** Some models (GPT-4, Mixtral, DeepSeek, Claude 3.5+) replace the FFN with a *bank* of FFN "experts," and a router chooses **which 2** experts to run per token. If there are 8 experts, the model has 8× the FFN parameters but only activates 2 per token — same compute per token, more capacity. We'll revisit in doc 07.

---

## 8. Normalization and residual connections

### 8.1 Residual connections

Every sub-layer is wrapped in a residual connection: the input is added back to the output.

```python
x = x + attention(norm(x))
x = x + ffn(norm(x))
```

Why: without residuals, deep networks are nearly impossible to train. Residuals let gradients flow directly from layer N back to layer 1. They're also why you can think of each layer as a small *edit* to the running representation — not a replacement.

### 8.2 Layer normalization

Before each sub-layer, the input is normalized. The original transformer used **LayerNorm** (post-norm: after the sub-layer). Modern models use **RMSNorm** applied **pre-norm** (before the sub-layer):

```python
def rms_norm(x, weight, eps=1e-6):
    # Normalize by root-mean-square of the vector, then scale by learned weight.
    rms = (x.pow(2).mean(-1, keepdim=True) + eps).sqrt()
    return (x / rms) * weight
```

RMSNorm drops the mean-centering of LayerNorm — turns out that wasn't necessary, and it's cheaper. Pre-norm (normalizing *before* the sub-layer) is more stable to train than post-norm at high depth.

---

## 9. Putting one layer together

Pseudocode for one decoder-only transformer layer, modern style:

```python
def transformer_layer(x, weights):
    # Pre-norm + attention + residual
    x_norm = rms_norm(x, weights.attn_norm)
    attn_out = multi_head_attention(
        x_norm,
        W_q=weights.W_q, W_k=weights.W_k, W_v=weights.W_v, W_o=weights.W_o,
        causal_mask=True,
    )
    x = x + attn_out

    # Pre-norm + FFN + residual
    x_norm = rms_norm(x, weights.ffn_norm)
    ffn_out = swiglu_ffn(x_norm, weights.W_gate, weights.W_up, weights.W_down)
    x = x + ffn_out

    return x
```

Stack N of these. N is "number of layers" or "depth." For reference:

| Model | Layers | Hidden | Heads | FFN dim | Params |
|---|---|---|---|---|---|
| GPT-2 small | 12 | 768 | 12 | 3072 | 124 M |
| GPT-3 175B | 96 | 12288 | 96 | 49152 | 175 B |
| Llama 3 8B | 32 | 4096 | 32 (8 KV) | 14336 | 8 B |
| Llama 3 70B | 80 | 8192 | 64 (8 KV) | 28672 | 70 B |

Frontier closed models (GPT-4, Claude, Gemini) don't publish exact numbers but are in the 100s of layers, 10–100k hidden, MoE routing. Same architecture — just scaled.

---

## 10. The final step: logits

After the last layer, the output vectors (shape `[seq_len, hidden_dim]`) are projected to vocabulary size:

```python
logits = final_norm(x) @ W_unembed   # shape [seq_len, vocab_size]
```

`logits[i, v]` = "unnormalized score for token `v` being at position `i+1`."

To get a probability distribution:
```python
probs = softmax(logits, dim=-1)
```

To generate, we look at the **last position** (the position that predicts the next-to-come token) and sample from it:

```python
next_token_logits = logits[-1, :]  # shape [vocab_size]
next_token = sample(next_token_logits, temperature=1.0, top_p=0.9)
```

Sampling strategies (temperature, top-p, top-k) are doc 03 §5.

---

## 11. Aside: encoder-decoder vs decoder-only

The 2017 paper's original transformer had an **encoder** (bidirectional attention — every token sees every other) and a **decoder** (causal attention + cross-attention to the encoder). Used for translation: encoder reads the source, decoder generates the target.

Modern LLMs are **decoder-only**. Why?

- Training is simpler (one loss: next-token prediction on raw text).
- The same model handles "understand" and "generate" — input and output are the same sequence, separated implicitly.
- Scales better empirically.

**Encoders alone** (BERT, RoBERTa, modern embeddings models) are still used for embeddings, classification, retrieval. They're good at producing a fixed vector for a sequence. They don't generate text.

**Encoder-decoder** (T5, BART) is largely historical for LLMs, though still used in specific domains (speech, some translation pipelines).

When someone says "transformer" in a 2026 context, 99% of the time they mean decoder-only.

---

## 12. What the transformer learns (high level)

Different layers tend to specialize, on average:

- **Early layers**: surface features — POS-like patterns, token identity, local context.
- **Middle layers**: syntactic and semantic patterns — subject-verb agreement, coreference, high-level semantics.
- **Late layers**: task-specific reasoning, formatting, outputting the final prediction.

Attention heads in middle layers are where most "induction head" behavior lives — heads that implement in-context learning ("I saw `A → B`, so when I see `A` again, output `B`"). Induction heads are the mechanistic story behind why LLMs can learn from examples in the prompt. (See Anthropic's "A Mathematical Framework for Transformer Circuits" for the deep version.)

---

## 13. A few things the architecture does NOT do

- **It does not have a long-term memory across calls.** Each inference is stateless. "Memory" in products like ChatGPT is implemented by stuffing prior conversation back into the context window.
- **It does not "think" between tokens.** Each token's generation is a single forward pass. Reasoning models (doc 07 §8) get around this by generating hidden "thinking" tokens before the visible answer.
- **It does not have reliable internal beliefs.** Attention weights are not "attention" in the cognitive sense — they're learned routing.
- **It does not deterministically produce the same output twice** unless you pin temperature=0, sampling seed, and use the same batch composition (batch-level non-determinism exists due to how GPUs schedule).

These are doc-03 and doc-07 topics, but flag them now so the architecture doesn't feel more magical than it is.

---

## 14. Summary

A transformer is a stack of identical layers. Each layer has two moves:

1. **Attention** — update every token's vector using information from every other (previous) token.
2. **Feed-forward** — run every token's vector through a small MLP.

Wrap each with normalization and residual connections. Stack N times. Embed tokens at the bottom, un-embed to logits at the top. Sample.

That's the whole thing. The difference between GPT-2 and Claude Opus 4.7 is ~1000× scale, much better training data, post-training alignment, and a bag of inference tricks — not a different architecture.

In the next doc, we look at **tokenization** — how text becomes the token IDs that feed the embedding table in the first place.

---

Next: [02 — Tokenization](./02-tokenization.md).
