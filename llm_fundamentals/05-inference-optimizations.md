# 05 — Inference Optimizations

Goal: understand how serving a 70B-parameter model at 100 tokens/sec per user, across thousands of users, for a price that makes economic sense, is actually done. KV caching, quantization, batching, speculative decoding, paged attention, and a few more. This is where most of the real engineering happens.

---

## 1. The problem being solved

A frontier LLM has 100B–1T+ parameters. On disk, in FP16, a 70B model is 140 GB. On GPU in FP16, it still needs 140 GB of HBM memory (plus working memory and KV cache). One H100 has 80 GB HBM; you need multiple GPUs just to load the model.

Per generated token, the model must:
1. Read all of its weights through memory at least once.
2. Read and extend the KV cache.
3. Do the forward pass math.

Memory bandwidth ends up being the bottleneck. An H100 has ~3 TB/s of HBM bandwidth. For a 140 GB model in FP16, naive decoding is bounded at 3000 / 140 ≈ **21 tokens/sec per user** — and that's assuming zero overhead. Every optimization in this doc is either:

- Reducing bytes moved per token (quantization, MoE).
- Reusing work across tokens or users (KV cache, batching, caching).
- Generating multiple tokens per forward pass (speculative decoding).
- Fitting more users on the same GPU (paged attention).

---

## 2. KV cache — the single most important optimization

### 2.1 Why a cache exists

During decode, each new token does a forward pass. Naively, that forward pass would recompute attention over the *entire* sequence again, including all past tokens. Wasteful: for tokens that haven't changed, their K and V vectors haven't changed either.

**Solution:** after computing each layer's K and V for a token, store them. On the next step, the new token computes *its own* Q, K, V, then attends to the cached K, V of all previous tokens plus its own.

```
Step t=4 (decode step for 5th token):
  New token's Q:    compute from scratch
  New token's K, V: compute from scratch, append to cache
  Attention = softmax(Q @ stacked_K.T) @ stacked_V
            where stacked_K, stacked_V include all tokens 0..4
```

Per decode step, the math done is proportional to the number of *new* tokens (1), not the total sequence length. Huge.

### 2.2 How big is the KV cache?

Per token, per layer, per head, we store 2 vectors (K and V) of size `d_head`.

```
cache_bytes = 2 (K, V) × n_layers × n_kv_heads × d_head × 2 bytes (FP16) × seq_len
```

For Llama 3 70B (80 layers, 8 KV heads with GQA, d_head=128):
```
per token = 2 × 80 × 8 × 128 × 2 = 327 680 bytes ≈ 320 KB
```

At 128 k context: 128 000 × 320 KB ≈ **41 GB of KV cache — per sequence.** Fits on an H100, but barely, and only one at a time.

Without GQA (if Llama 70B used 64 KV heads instead of 8), the cache would be 8× larger — 328 GB per sequence. Impossible without aggressive tricks.

### 2.3 Why GQA / MQA exist

Grouped Query Attention (doc 01 §6) exists *primarily* to shrink the KV cache. Fewer K, V heads → fewer bytes per token → more concurrent users per GPU. The quality hit is small when done well; the infra win is enormous.

### 2.4 KV cache across requests — prompt caching

The KV cache is per-sequence. But if two sequences share a prefix (e.g., the same system prompt), their prefix KV is identical. Providers exploit this:

- **Compute prefix KV once, reuse across requests.**
- Store in GPU memory for a short window (5 min on Anthropic; configurable on OpenAI).
- Next request with the same prefix: skip prefill for the cached portion.

Reads from cache are ~0.1× the cost of fresh prefill, and latency drops from seconds to tens of ms. This is "prompt caching" as a product feature (doc 08 §4).

---

## 3. Quantization — fewer bits per parameter

Default training uses FP32 (4 bytes/param) or BF16 (2 bytes/param). But for *inference*, full precision is overkill — you can shrink parameters to 8, 4, or even 2 bits with small quality loss.

### 3.1 Bit widths

| Format | Bytes/param | 70B model size | Quality impact |
|---|---|---|---|
| FP32 | 4 | 280 GB | Baseline (training). |
| BF16 / FP16 | 2 | 140 GB | Standard inference. Negligible loss. |
| FP8 (e4m3, e5m2) | 1 | 70 GB | ~0.1–0.5% benchmark drop. H100+ native support. |
| INT8 | 1 | 70 GB | Usually requires calibration; similar to FP8. |
| INT4 (GPTQ, AWQ) | 0.5 | 35 GB | 1–3% drop on hard benchmarks; often acceptable. |
| INT3 / INT2 | < 0.5 | 17 GB | Significant degradation; research territory. |

### 3.2 Quantization techniques

**Post-Training Quantization (PTQ).** Take a trained model, compute quantized weights in one shot. Cheap. Works surprisingly well down to INT8.

- **GPTQ** (Frantar et al., 2022): layer-by-layer optimization using a small calibration dataset. De facto standard for INT4 open models.
- **AWQ** (Lin et al., 2023): observes that some weight channels matter much more; quantizes carefully around those. Slightly better than GPTQ in practice.
- **GGUF / llama.cpp quants** (q4_0, q5_K_M, q8_0, etc.): practical family for CPU/consumer-GPU inference.

**Quantization-Aware Training (QAT).** Simulate quantization during training. Better quality at low bit widths but expensive. Rare for frontier models.

**FP8 hardware support.** H100, H200, MI300 have native FP8 matmul units — FP8 inference runs ~2× faster than FP16, with near-zero quality loss. Every major provider has moved or is moving to FP8 for frontier serving.

### 3.3 What quantization does not shrink

- **Activations** — the intermediate vectors during forward pass. Usually kept in higher precision.
- **KV cache** — sometimes quantized (FP8, INT8), trading a small quality hit for 2× more concurrent sequences.
- **Attention softmax accumulation** — needs FP32 internally for stability.

### 3.4 Quality vs cost tradeoff

For most production workloads:
- **BF16 / FP8**: frontier quality, frontier cost.
- **INT8**: ~95% of the quality at ~50% of the memory and latency.
- **INT4**: ~90% on easy tasks, 70–85% on hard reasoning tasks. Good for local/on-device.

Rule of thumb for self-hosted: FP8 if your hardware supports it, INT4 (GPTQ/AWQ) if you're squeezing onto a consumer GPU and the task isn't reasoning-critical.

---

## 4. Batching — the other half of GPU utilization

A single-user decode uses ~1/10th of GPU compute — the GPU is idle most of the cycle, waiting on memory. If 10 users are decoding at the same time, you can batch their forward passes: load weights once, use them for 10 separate sequences.

### 4.1 Static batching (simple and bad for production)

Collect a batch of N requests, run them together until the longest one finishes. Problem: the fast ones wait for the slow ones. GPU is idle after short responses complete.

### 4.2 Continuous batching (in-flight batching)

The production solution (vLLM, TGI, TensorRT-LLM, all implement this).

- After every decode step, check for completed sequences. Swap them out.
- Insert new requests into the batch's empty slots immediately.
- The batch has dynamic composition — different sequences at different positions in their generation.

Result: near-100% GPU utilization under load. The key enabler of economical LLM serving.

### 4.3 The batching-vs-latency trade-off

Batching increases throughput (tokens/sec across all users) but doesn't improve per-user latency. In fact, larger batches can slightly slow each user's decode (more competition for memory bandwidth).

- Low-traffic serving: smaller batches, faster per-user tokens.
- High-traffic: larger batches, better total throughput, slightly slower per-user.

APIs balance this automatically. Self-hosting: tune `max_num_seqs` and `max_num_batched_tokens` in vLLM.

---

## 5. Paged attention — the KV cache revolution

A problem with continuous batching: every sequence's KV cache is a different size. Naively allocating a contiguous buffer for each one wastes memory — some sequences are short, some long, and you have to reserve max-length buffers just in case.

**PagedAttention** (Kwon et al., 2023, the vLLM paper) borrows virtual-memory page-table ideas from OS kernels:

- KV cache is stored in fixed-size **pages** (e.g., 16 tokens each).
- A logical sequence is a list of pages; the physical pages can be scattered anywhere.
- New pages are allocated as needed, freed when the sequence ends.

Benefits:
- **Near-zero fragmentation.** Memory is allocated in page units; no wasted pre-allocated buffer.
- **Prefix sharing.** Multiple sequences sharing a prefix can share the same physical pages. Massive savings for few-shot prompts, system prompts, parallel sampling.
- **Scheduler freedom.** Can preempt and swap sequences in/out because their KV pages aren't tied to a contiguous slab.

Result: **2–4× more concurrent sequences on the same GPU** vs naive allocation. This is what makes large-scale self-serving practical.

Virtually every serious inference server uses this or a close analog (vLLM's PagedAttention, TensorRT-LLM's KV cache manager, etc.).

---

## 6. Speculative decoding — generating N tokens per forward pass

### 6.1 The idea

Decode is memory-bound — the GPU spends most of its time reading weights. What if we could *verify* multiple candidate next-tokens in parallel, using the same weight read?

Speculative decoding:

1. A small **draft model** (e.g., 1B params) quickly generates K candidate next-tokens.
2. The large **target model** (e.g., 70B) runs a *single* forward pass that scores all K candidates at once.
3. Accept the longest prefix of candidates that the target model agrees with.
4. If K=4 draft tokens were proposed and the target agrees with 3, generate 3 tokens in one forward pass — 3× effective speedup.

### 6.2 Why it works

The math: the target forward pass produces logits at every position. Running over `[existing_context + draft_token_1 + draft_token_2 + draft_token_3 + draft_token_4]` produces the target's logits at each of those positions. Comparing against the draft, you can mathematically prove that accepting K tokens is equivalent to having sampled them from the target directly — given a proper acceptance criterion.

No quality loss. Just speed.

### 6.3 Variants

- **Speculative sampling** (original, Chen et al. 2023).
- **Medusa** (Cai et al. 2024): multiple linear heads on the target model predict future tokens simultaneously — no separate draft model.
- **EAGLE** (Li et al. 2024): small auxiliary model trained on target's intermediate features; better acceptance rates than generic small drafters.
- **Lookahead decoding** (Fu et al. 2024): no extra model; uses n-gram patterns seen in-context to propose candidates.

Production speedups: 1.5–3× for most workloads. Higher on repetitive/structured outputs (code, JSON) where the draft model guesses well; lower on creative writing where acceptance rate drops.

---

## 7. Tensor parallelism, pipeline parallelism — multi-GPU serving

A 70B model in FP16 doesn't fit on one 80 GB GPU. Solution: split the model across GPUs.

### 7.1 Tensor parallelism

Split each weight matrix across GPUs along a dimension. For matmul:

```
X @ W → split W by column across 4 GPUs
Each GPU computes X @ W[:, i*k:(i+1)*k]
Results concatenated via all-reduce or all-gather
```

- Each forward pass: frequent all-reduce between GPUs → needs NVLink / NVSwitch / InfiniBand for low latency.
- Works best across 4–8 GPUs within one node.

### 7.2 Pipeline parallelism

Split the model by **layer**: GPU 1 handles layers 1–20, GPU 2 handles 21–40, etc. Tokens flow through the pipeline.

- Less communication than tensor parallelism.
- Requires batching to keep all pipeline stages busy ("bubble" problem if only one request is in flight).
- Works across nodes where inter-node bandwidth is lower.

### 7.3 Expert parallelism (for MoE)

For Mixture-of-Experts (doc 07 §3.6), experts can live on different GPUs. Routing a token to 2 experts out of 8 means only 2 GPUs do work for that token; the others are idle for that layer. Tricky to balance.

### 7.4 Practical setups

- **70B dense model**: 4×H100 with tensor parallelism — fits, fast.
- **405B dense**: 8×H100 (nearly full node), TP across 8.
- **1T+ MoE**: multi-node with hybrid TP+PP+EP. DeepSeek-V3 at 671B (37B active) runs on ~1 node with aggressive parallelism.

---

## 8. CPU offload and tiered memory

When GPU memory is tight, some systems offload pieces to CPU RAM or NVMe:

- **CPU-offload of weights**: the CPU holds layer weights, streams them to GPU for each forward pass. Drastically slower (PCIe < HBM) but fits huge models on small GPUs. Used in HF accelerate, DeepSpeed-Inference. Fine for batch/offline work, not real-time.
- **SSD-offload of KV cache**: for huge contexts, cache pages can swap to NVMe. Latency cost is real but beats "can't handle the request at all."

These are last-resort techniques for squeezing models onto insufficient hardware. Not how frontier serving works; how hobby and research serving survives.

---

## 9. Token-level tricks

### 9.1 Early exit

Some architectures allow a token to skip remaining layers if an early layer's prediction is already confident enough. Research-stage; not standard in production yet.

### 9.2 Speculative *sampling* vs draft

As covered in §6 — essentially the same idea applied at the sampling level: sample multiple candidates, keep the ones the full model agrees with.

### 9.3 Dynamic batching across prefill and decode

Modern servers interleave prefill (compute-heavy) and decode (memory-heavy) in the same batch. Prefill fills GPU compute; decode fills memory bandwidth. Together they utilize both better than either alone.

---

## 10. Caching layers (not just KV)

Beyond the KV cache, production systems add higher-level caches:

### 10.1 Prompt cache (provider feature)

Anthropic prompt caching, OpenAI prompt caching. At the API layer, detect shared prefixes across requests and charge ~10% of normal input cost for cached portions. Latency drops by the prefill cost for the cached prefix (seconds → tens of ms).

Rules (vary by provider):
- Minimum cache-eligible prefix length (Anthropic: 1024 tokens for Opus, 2048 for Haiku).
- TTL: 5 minutes (Anthropic), refreshed on hit.
- Explicit `cache_control` blocks in the request to mark cache boundaries.

### 10.2 Semantic cache (application layer)

Cache entire responses for **semantically similar** requests. Embed the incoming query; if an existing cached query has cosine similarity > threshold, return the cached answer.

Tools: GPTCache, Redis + vector search, home-grown.

Risks:
- False positives kill correctness. "Transfer $100 to Alice" and "Transfer $1000 to Alice" are semantically close. Don't use semantic cache for non-idempotent or precision-critical calls.
- Works best for FAQ-style repetitive reads.

### 10.3 Deterministic output cache

For temperature=0 requests with pinned model versions, the output is nearly deterministic. Cache by `hash(prompt + model + params)` → response. Skip the LLM entirely on hit.

---

## 11. Distillation — not inference, but produces smaller models

A different optimization axis: instead of making a big model faster, make a small model nearly as good.

**Knowledge distillation**: train a small "student" model to mimic a large "teacher" model's outputs (logits, or just final answers). Examples:

- GPT-4o mini, Claude Haiku — distilled/derived from their larger siblings.
- Gemma 2 — distilled from Gemini.
- Phi-3 — distilled from curated, synthetic data generated by GPT-4.

A well-distilled 7B model can match a stock 70B model on many benchmarks. Deployed at ~10× lower cost and ~3× faster. Limitation: distillation preserves the teacher's strengths *and* weaknesses; it doesn't invent new capability.

---

## 12. What "cheap inference" costs

Rough economics for self-hosting Llama 3.1 70B at FP8 on 4×H100:

- Hardware: ~$160k for 4×H100 SXM, or ~$4/hr per GPU on cloud = $16/hr, $11k/month.
- Throughput: ~10 000 tokens/sec aggregate across concurrent users (with continuous batching + PagedAttention).
- At 90% utilization: ~23B tokens/month.
- Cost per 1M tokens ≈ $0.50 (amortized).

Compare to Llama 3.1 70B hosted (Groq, Together, Fireworks): $0.80–$1.00 per M tokens. The hosted provider is making a small margin; their scale gives them leverage you don't have at 1 node.

**Break-even threshold:** self-host makes sense only above ~5B tokens/month on a single model. Below that, hosted wins on operations cost alone.

---

## 13. Inference stack choices

| Stack | Best for | Notes |
|---|---|---|
| **vLLM** | Production self-serving | Open source, PagedAttention, continuous batching, wide model support. |
| **TensorRT-LLM** | Max throughput on NVIDIA | Closed-ish, NVIDIA-optimized. Complex to set up; fastest available. |
| **SGLang** | Complex multi-turn / structured | RadixAttention for prefix sharing; good at agent workloads. |
| **Text Generation Inference (TGI)** | HF ecosystem | Mature, HF-integrated. vLLM has overtaken it on most metrics. |
| **llama.cpp / GGUF** | CPU + consumer GPU | Aggressive quantization; for local/edge. |
| **Ollama / LM Studio** | Developer laptops | Wraps llama.cpp; for personal use. |
| **MLC-LLM** | Mobile / browser | WebGPU, iOS, Android targets. |
| **Groq / Cerebras (hardware)** | Ultra-low latency | Custom silicon; 500+ tok/s/user on 70B+. Hosted only. |

---

## 14. Putting it all together: the life of a token, optimized

Request arrives, 2000-token prompt, 500-token response:

1. **API tier**: check prompt cache → 1800 tokens already cached, 200 fresh.
2. **Scheduler**: assign to a GPU whose batch has slots.
3. **Prefill**: run 200 fresh tokens through FlashAttention v3 kernels at FP8 on 4 H100s with tensor parallelism. Fuse into existing in-flight batch (continuous batching). Store new KV in paged cache.
4. **Decode**: step by step, drafter (1B model) proposes 4 tokens → target (70B) verifies in 1 pass → accept 3, redo last, generate net 3 new tokens. Repeat.
5. **Streaming**: tokens fly out over SSE as they're accepted.
6. **Complete**: KV pages remain live for 5 min for prompt-cache reuse. Metrics emitted.

Every step above is one of the optimizations in this doc. Together they push a 70B model from "barely usable at $50/M tokens" to "production at $1/M tokens, 100+ tok/sec/user."

---

## 15. Summary

The three big wins that made modern LLM serving economical:

1. **KV cache + paged memory management.** Don't recompute; don't waste memory holding it.
2. **Quantization (FP8, INT8, INT4).** Move fewer bytes per token.
3. **Continuous batching.** Use the GPU to its full width across many users.

The secondary wins — speculative decoding, prompt caching, flash attention, tensor parallelism — each add another 1.5–3× somewhere. Stack them and you get 10–50× over the naive baseline.

When someone says "we made our inference 5× cheaper this year," they're almost always talking about some combination of the techniques above, not a new algorithm.

Next we look at **training** — how these models get their weights in the first place, and why every new generation requires not just scale but careful post-training.

---

Next: [06 — Training Pipeline](./06-training-pipeline.md).
