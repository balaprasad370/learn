# 04 — Context Windows and Position

Goal: understand why context length is hard, why position encoding matters, how RoPE and ALiBi work, and what techniques allow modern models to reach 200 k, 1 M, and 10 M tokens.

---

## 1. What "context window" means

The **context window** is the maximum number of tokens the model can attend to in one forward pass. Everything — system prompt, all prior turns, the current user input, and the generated output so far — must fit in that window.

Historical progression:
```
GPT-2 (2019):           1 024 tokens
GPT-3 (2020):           2 048 → 4 096
GPT-3.5 Turbo:          4 096 → 16 384
GPT-4 (2023):           8 192 → 32 768 → 128 k
GPT-4o / Claude 3:      200 000
Claude 3.5 / 4:         200 000 (some variants 1 M)
Gemini 1.5 Pro:       1 000 000 → 2 000 000
Gemini 2 / some:     10 000 000 (announced)
Llama 3.1:            128 000
```

A 1-million-token window is ~750 k words — roughly 10 novels. And yet, every API call ships that many tokens through O(n²) attention. Why is that even feasible? That's what this doc answers.

---

## 2. Why attention is O(n²)

Recall doc 01 §5: attention computes a `[seq_len, seq_len]` score matrix. For 10 000 tokens, that's 100 M entries per head, per layer. For 80 layers, 64 heads: 100 M × 80 × 64 = **512 billion scores per forward pass**.

Two consequences:

1. **Compute scales as n².** Doubling context = 4× FLOPs on the attention matrices.
2. **Memory scales as n² too** — holding those scores in memory before applying softmax.

Point 2 was the killer. For 10 k tokens in FP16, `10 000 × 10 000 × 2 bytes × 64 heads × 80 layers ≈ 1 TB` of attention matrix memory per sequence. That's absurd. The rest of this doc is about making this manageable.

---

## 3. Position: how does the model know "order"?

A plain attention mechanism is **permutation-invariant** — shuffle the tokens and the attention output is the same (just shuffled). That's a disaster for language: "dog bites man" and "man bites dog" would look identical.

So we must inject **position information** somehow. Three families have emerged.

---

## 4. Sinusoidal position embeddings (original 2017)

The first transformer paper's approach: add a fixed, hand-crafted sinusoidal signal to each token embedding based on its position.

```python
PE[pos, 2i]   = sin(pos / 10000^(2i / d_model))
PE[pos, 2i+1] = cos(pos / 10000^(2i / d_model))
```

- Deterministic, not learned.
- Different positions get different signals.
- "Extrapolates": positions beyond training length have valid signals, even if the model hasn't seen them.

In practice, extrapolation didn't work well — models trained on 512 tokens didn't generalize to 2000. But the idea seeded everything that followed.

---

## 5. Learned absolute position embeddings (BERT, GPT-2, GPT-3)

Instead of hand-crafting, **learn** a position embedding table of shape `[max_seq_len, hidden_dim]`. Add `pos_embedding[i]` to token `i`'s embedding.

Pros: simple, model learns what it needs.
Cons: strict maximum sequence length (the size of the table). Can't go past it. This is why GPT-3 was hard-capped at 2048 — the learned table was 2048 × 12288. Extending meant re-training.

---

## 6. RoPE — Rotary Position Embedding (current dominant)

Used by: Llama 2/3, Mistral, Gemma, most open models, many closed ones.

### 6.1 The idea

Instead of adding position to the embedding, **rotate** the query and key vectors as a function of their position. The rotation is applied inside the attention block, per-head, pre-dot-product.

Conceptually: split each 128-dim Q/K vector into 64 pairs of 2-dim sub-vectors. Rotate each pair by an angle that depends on position. The rotation angle for pair `i` at position `m` is:

$$
\theta_{m,i} = m \cdot 10000^{-2i/d}
$$

High-frequency pairs rotate fast; low-frequency pairs rotate slow.

### 6.2 Why rotation?

After rotating Q at position `m` and K at position `n`, their dot product depends only on their **relative position** `m - n`, not on their absolute positions. This is a huge win:

- Attention between tokens 5 and 7 behaves like attention between tokens 5 000 and 5 002. The model learns "how do nearby tokens interact" once, and it applies everywhere.
- No position embedding is added to the input — it's applied inside attention. Easier to inject at inference time.

### 6.3 RoPE extension: the reason 1 M contexts are possible

RoPE is parameterized by a base frequency (the `10000` above). Increase the base, and the rotation angles slow down — the model can distinguish positions further apart without running out of "unique angles."

**YaRN, NTK scaling, Position Interpolation:** three methods to stretch a model trained on 4k context to run at 32k, 128k, or 1M. The gist is that you either:
- Multiply position indices by a factor (PI): position 32000 in inference = position 4000 in training, pushed through the same rotations.
- Change the rotation base (NTK / YaRN): tune so that high-frequency rotations shrink their effective range, giving more headroom at long distances.

These are **inference-time or fine-tuning-time tweaks** that don't require pretraining from scratch. Llama 3's 8k → 128k context was done this way, followed by a short context-extension fine-tuning run.

---

## 7. ALiBi — Attention with Linear Biases

Used by: BLOOM, MPT, some research models.

Simpler idea: instead of rotating Q and K, add a **linear bias** to attention scores based on distance:

$$
\text{score}(i, j) = q_i \cdot k_j - \lambda \cdot (i - j)
$$

where `λ` is a per-head constant. The further back a token is, the more negative the bias, the less it's attended to (after softmax).

ALiBi extrapolates naturally — the bias formula works at any distance. But the penalty is fixed per head; the model doesn't learn richer position patterns the way RoPE allows. Outside the original BLOOM line, ALiBi lost to RoPE in practice.

---

## 8. The memory problem: FlashAttention

Even with a fix for position, you still have to *compute* `Q @ K^T` over 200k tokens, and the memory blowup is real.

**FlashAttention** (Dao et al., 2022, v2 2023, v3 2024) is an algorithmic reorganization of how attention is computed on GPUs. Core idea:

1. **Don't materialize the full attention matrix.** Instead, process blocks of Q and K tiled through GPU SRAM (fast, tiny) rather than HBM (slow, big).
2. **Use online softmax.** Compute softmax incrementally so you never need the whole row in memory at once.
3. **Fuse operations.** One kernel for Q @ K, softmax, and @ V — no intermediate read/write.

Result: attention memory goes from **O(n²)** to **O(n)**, and throughput jumps 2–4×. Every modern inference stack uses FlashAttention or its descendants.

FlashAttention doesn't reduce FLOPs — attention is still O(n²) in compute. But it fits in memory, and it uses GPU hardware near its theoretical peak.

---

## 9. Sparse attention patterns

FlashAttention keeps full attention tractable. Sparse attention patterns attack the O(n²) cost directly by skipping most of the matrix.

### 9.1 Sliding window (Longformer, Mistral)

Each token attends only to the previous W tokens. For W=4096, even at 128k context, each token's attention is bounded to W neighbors.

```
Full attention (causal):                  Sliding window (W=3):
  ■                                         ■
  ■ ■                                       ■ ■
  ■ ■ ■                                     ■ ■ ■
  ■ ■ ■ ■                                     ■ ■ ■
  ■ ■ ■ ■ ■                                     ■ ■ ■
```

Token `i` sees only `[i-W, i]`. Compute drops from O(n²) to O(n·W).

Downside: long-range dependencies beyond W are lost. Typically combined with:

### 9.2 Global tokens

Designate a few tokens (e.g., `<system>`, `<summary>`) that every token attends to and that attend to every token. Gives the model a "global blackboard" without dense attention everywhere.

### 9.3 Strided / dilated patterns

Attend to every k-th token in addition to local. Good for coverage, complicates implementation.

### 9.4 Hybrid: Mistral's approach

Mistral 7B uses sliding window attention with W=4096. At 32k context, you're not paying 32k² attention cost — you're paying 32k × 4096. The model is still decent at long-range because attention is **stacked across layers**: token at position 32000 can't directly see token at position 0 in one layer, but through 32 layers of W=4096 windows, information propagates.

---

## 10. Linear attention and state-space models

FlashAttention and sparse patterns still live in the "attention" family. A different thread of research replaces attention entirely:

### 10.1 Linear attention

Replace softmax with a kernel function you can compute in O(n) by reordering multiplications:

$$
\text{softmax}(QK^T)V \approx \phi(Q) (\phi(K)^T V)
$$

where $\phi$ is a feature map. Gets you O(n) time and memory. Quality historically lagged full attention, though newer variants (Performer, Linear Attention with proper kernels) close the gap for some tasks.

### 10.2 State-space models: Mamba

Not attention at all. A continuous-time recurrent model that processes tokens sequentially but in parallel via scan operations.

- O(n) time and memory — no quadratic at all.
- Competitive with transformers at small scale; at frontier scale, hybrid (Mamba + attention) architectures are showing up in research.
- No KV cache — state is a fixed-size vector.

### 10.3 Hybrid models (emerging 2024–2026)

Many production models now mix:
- **Transformer layers** (full attention) for maximum expressivity.
- **Mamba/SSM layers** for efficient long-context information propagation.

Examples: Jamba (AI21), StripedHyena (Together), early Gemini variants. These aren't dominant yet but represent where architecture is heading: pure transformer stacks are not the final answer for long context.

---

## 11. What "long context" actually buys you

Having 1 M tokens doesn't mean you can stuff 1 M tokens of random text and get a useful answer. Two phenomena:

### 11.1 Needle-in-a-haystack

Place a specific fact at some position in a long irrelevant context, then ask a question about it. Does the model find it?

Results on modern frontier models:
- **≤ 32 k**: near-perfect retrieval across positions.
- **32 k – 128 k**: good, but dips in the "middle" positions (around the 40–60% mark) — the so-called "lost in the middle" effect.
- **128 k – 1 M**: increasingly degraded. Claims of "perfect retrieval at 1 M" have not survived careful evaluation.
- **> 1 M**: currently more marketing than production capability.

### 11.2 Reasoning at long context

Even worse than retrieval. Asking the model to **reason** over multiple facts spread across a long context — multi-hop inference, cross-document comparison — degrades sharply past 32 k. Models can find single facts at 200 k; they struggle to connect 10 facts at 50 k.

### 11.3 Practical implications

- If you have 200 k tokens of context and ask a simple lookup question: expect good results.
- If you have 200 k tokens of context and ask "compare the trends across these three documents": expect inconsistency. Consider summarizing or splitting.
- RAG (retrieval + short context) often outperforms long-context dumps for knowledge-heavy tasks — because the model reasons better on 10 k of well-chosen content than 500 k of kitchen sink.

Long context is a tool, not a substitute for careful prompt engineering.

---

## 12. How context length translates to cost and latency

### 12.1 Cost

Input tokens are billed — 500 k input tokens on Claude Opus 4.7 at $15/M = **$7.50 per call**. A chat conversation that replays the same 500 k context on every turn burns $7.50 × N turns, unless prompt caching applies (doc 08).

### 12.2 Prefill latency

Prefill is O(n²) in the *attention component* and O(n) in FFN. At 100 k tokens on a frontier model, prefill can be 5–20 seconds. Some models prefill faster than others thanks to FlashAttention v3 and architectural tricks. It's still the slowest part of a long-context call.

### 12.3 Decode latency

Unaffected by prompt length *at the per-token level* — each new token just does one forward pass against the cached KV (doc 05 §2). Longer prompts → larger KV cache → slightly slower decode due to memory bandwidth, but sub-linearly.

---

## 13. When long context breaks

Several failure modes as context grows:

1. **Attention dilution.** With 100 k tokens, each token's attention is spread across a huge space. Useful signal can get washed out.
2. **Format drift.** Long contexts often contain their own formatting (markdown, quoting, pseudo-code). The model's output format can drift toward what's in context.
3. **Instruction following degrades.** Long context often pushes the system prompt "far away" in the sequence. Models may progressively ignore system instructions as context grows. Strong system prompts need repetition or positioning at the end.
4. **Information order matters.** The same facts in a different order can produce different conclusions. Put the most important content last (closest to the cursor) when possible.

---

## 14. Practical checklist for long-context prompts

1. **Summarize before scaling.** If you can summarize 100 k tokens into 10 k, do it once and cache the summary.
2. **Use prompt caching** for any context you'll reuse across calls (doc 08).
3. **Re-state instructions near the end.** Don't trust a system prompt at position 0 when the user input is at position 200 000.
4. **Prefer retrieval over stuffing.** RAG on 10 k of relevant content beats 200 k of dump for most tasks.
5. **Test the specific length you'll use.** Don't assume a model rated "200 k context" works equally well at 10 k and 200 k. Benchmark on your actual task.
6. **Expect higher variance.** Temperature=0 at 200 k still produces more output variance than at 2 k, due to tiny numerical differences propagating through more layers of attention.

---

## 15. Summary

Context length is constrained by (a) how the model represents position and (b) how much compute and memory attention costs. Early models used learned absolute embeddings with hard caps. Modern models use RoPE, which is extensible via PI / YaRN / NTK. FlashAttention fixes the memory cost of computing attention. Sliding windows, sparse patterns, and state-space hybrids attack the compute cost. Long context works at retrieval, struggles at reasoning, and costs real money per call. Treat "context window size" as an upper bound on inputs — not a recommendation for how much to use.

Next we look at **inference optimizations** — KV cache, quantization, batching, speculative decoding, and the rest of the bag of tricks that makes serving a 70 B-parameter model at 100 tokens/sec economically viable.

---

Next: [05 — Inference Optimizations](./05-inference-optimizations.md).
