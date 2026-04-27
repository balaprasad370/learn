# 03 — LLM Serving Infrastructure

> The "how to actually run LLMs in production" layer. Asked at every infra-tier role (Sarvam, Krutrim, Giga, Together, Modal). Differentiator at app-tier roles (Lyzr, Composio).

---

## 1. The problem you're solving

A trained LLM is just weights. Serving it means:

- Loading 14–700 GB of weights onto GPUs.
- Running prefill (compute-bound, parallel-friendly) and decode (memory-bound, hard to batch).
- Managing KV cache that grows per request and dwarfs the weights at high concurrency.
- Achieving acceptable latency (TTFT, tokens/sec) and cost ($/M tokens).
- Doing all of this with 100s of concurrent users.

Every concept below is a tool to make one of those tractable.

---

## 2. Inference math (memorize this)

### 2.1 Memory for weights

```
weights_bytes = num_params × bytes_per_param
```

| Precision | Bytes/param | 7B | 13B | 70B | 405B |
|---|---|---|---|---|---|
| FP32 | 4 | 28 GB | 52 GB | 280 GB | 1.6 TB |
| FP16/BF16 | 2 | 14 GB | 26 GB | 140 GB | 810 GB |
| FP8 | 1 | 7 GB | 13 GB | 70 GB | 405 GB |
| INT4 | 0.5 | 3.5 GB | 6.5 GB | 35 GB | 200 GB |

### 2.2 Memory for KV cache

```
kv_per_token = 2 × n_layers × n_kv_heads × head_dim × bytes_per_param
```

The `2` is for K and V. With **GQA** (Grouped-Query Attention), `n_kv_heads < n_heads`, slashing this.

| Model | KV per token (FP16) | 4k context, 8 concurrent |
|---|---|---|
| Llama 3.1 8B | 128 KB | 4 GB |
| Llama 3.1 70B | 320 KB | 10 GB |
| Llama 3.1 405B | 700 KB | 22 GB |

KV cache is often the **dominant memory consumer at high concurrency** — bigger than weights.

### 2.3 Prefill vs decode

- **Prefill:** processes the prompt all at once. Compute-bound. Highly parallelizable.
- **Decode:** generates one token at a time. Each token depends on the previous. **Memory-bandwidth-bound** (model weights streamed from HBM per token).

This split shapes everything: batching matters most for decode; long prompts stress prefill compute.

### 2.4 Throughput math

For a single request, decode throughput on a memory-bound model:

```
tokens_per_sec ≈ HBM_bandwidth / weights_size
```

A 70B model in FP16 (140 GB) on H100 (3.35 TB/s HBM) bound: ~24 tok/s single-stream. Reality: 30–50 tok/s with KV cache hits and arithmetic intensity gains.

Aggregate throughput scales with batching — the entire point of vLLM/TGI/SGLang.

---

## 3. The serving engines

### 3.1 vLLM
- **Origin:** UC Berkeley (PagedAttention paper).
- **Killer feature:** PagedAttention — virtualizes the KV cache like an OS pages memory. No fragmentation; supports beam search, parallel sampling.
- **Continuous batching:** iteration-level scheduling — new requests can join mid-batch as old ones finish tokens.
- **Default choice** for self-hosted serving in 2026.

### 3.2 SGLang
- **Killer feature:** RadixAttention — automatic prefix sharing across requests. Huge win for chat with shared system prompts, agents reusing tool descriptions.
- **Frontend:** structured generation language (control flow, branches).
- Often beats vLLM on prefix-heavy workloads (multi-turn chat, agents, batch eval).

### 3.3 TensorRT-LLM
- **Origin:** NVIDIA.
- **Strength:** peak performance on NVIDIA hardware. INT8/FP8/INT4 kernels, fused attention, in-flight batching.
- **Cost:** lock-in to NVIDIA. Build process is heavier (compile per model + GPU).
- Used in production at Triton-served setups.

### 3.4 TGI (Text Generation Inference)
- **Origin:** Hugging Face.
- Production-ready serving with continuous batching, quantization, streaming.
- Solid alternative to vLLM; HF-ecosystem aligned.

### 3.5 llama.cpp
- **CPU/edge focused;** ggml format; great for laptops, single-GPU experiments, on-device.
- Powers Ollama, LM Studio.
- Don't use for production high-throughput serving.

### 3.6 Other notable
- **MLC-LLM:** cross-platform (web, mobile, edge).
- **DeepSpeed-MII / FasterTransformer:** Microsoft / NVIDIA libraries; often subsumed by above.
- **Ray Serve + vLLM:** for orchestrating multi-model deployments.

### 3.7 Picking
```
Highest throughput on NVIDIA, willing to compile per model → TensorRT-LLM
Prefix-heavy workload (agents, chat with long system prompts) → SGLang
General serving, broad compatibility → vLLM
HF ecosystem, easy ops → TGI
Edge / laptop / on-device → llama.cpp
```

For a senior IC interview default: **vLLM**. Mention SGLang for prefix-sharing wins, TRT-LLM for peak NVIDIA.

---

## 4. Continuous batching (the central idea)

### 4.1 Static batching (legacy)
- Group N requests of similar length; run together; wait for slowest.
- Wastes GPU on length variance; can't accept new requests mid-batch.

### 4.2 Continuous batching (modern)
- Schedule per **iteration** (one token step), not per request.
- A request of 100 output tokens occupies a slot for 100 iterations; another finishes at 50, frees its slot, new request joins.
- GPU stays at high utilization.

### 4.3 Knobs
- **`max_num_seqs`** — max concurrent requests (slots).
- **`max_num_batched_tokens`** — max total tokens prefilled per iter.
- **`gpu_memory_utilization`** — fraction of HBM for KV cache vs reserve.
- **`block_size`** — KV page block (16 or 32 typically).

### 4.4 Latency vs throughput trade-off
- Higher concurrency = higher throughput, higher per-request latency.
- For interactive chat: lower concurrency, lower TTFT.
- For batch eval / async: max concurrency.

You serve different SLAs from different deployments, often with the same model.

---

## 5. PagedAttention / KV-cache management

### 5.1 The problem
Naive KV cache: pre-allocate `max_seq_len × max_batch` per request. Wastes memory when requests are short. Fragments when reqs come and go.

### 5.2 Solution
Allocate KV in fixed-size **blocks** (16 or 32 tokens). Maintain a per-request **block table** mapping logical positions to physical blocks. New blocks allocated on demand.

### 5.3 Wins
- **No fragmentation** — blocks are uniform.
- **Beam search / parallel sampling** — share prefix blocks across beams (copy-on-write).
- **Prefix caching** — reuse blocks across requests with same prompt prefix.

This is why vLLM was a step-function improvement when it came out.

---

## 6. Prefix sharing (SGLang's edge)

When many requests share a prefix (system prompt, few-shot examples, retrieved context cached for a session), recompute is wasteful.

**RadixAttention:** maintain a radix tree of prefix → KV blocks. New request's prefill checks the tree, reuses matched prefix's KV, only computes the suffix.

**Wins on:**
- Multi-turn chat (system prompt + earlier turns reused).
- Agent loops (tool descriptions + scratchpad reused across iterations).
- Eval batch (same system prompt, different questions).

vLLM has prefix caching too (since v0.4); SGLang's is more aggressive.

---

## 7. Quantization (in serving)

Reduces precision of weights and/or activations to shrink memory and speed up math.

### 7.1 Weight-only quantization
- Weights stored low-precision; activations and matmul still high-precision.
- **GPTQ, AWQ:** post-training, calibration-based, INT4. Quality drop usually <1%.
- **bitsandbytes (NF4):** widely used for QLoRA training and inference.

### 7.2 Weight + activation quantization
- Both quantized — bigger speedup, more risk.
- **FP8** (e8m4, e5m2): supported on H100/B200. Near-FP16 quality, ~2× throughput.
- **INT8 (SmoothQuant, LLM.int8()):** older, still useful.

### 7.3 KV cache quantization
- Quantize the cache itself (FP8 or INT8 KV).
- Halves the dominant memory consumer at high concurrency. ~Negligible quality hit.

### 7.4 Picking precision
```
Tight quality needs, plenty of GPUs → BF16 weights + BF16 KV
Cost-sensitive, modest quality loss OK → FP8 weights + FP8 KV (H100)
Edge / cheap GPU → INT4 weights (AWQ/GPTQ) + INT8 KV
```

Always benchmark on **your eval set** before committing. Public benchmarks can hide domain regressions.

---

## 8. LoRA serving (multi-adapter)

If you fine-tune one base model into N adapters (per tenant, per task), don't run N engines.

### 8.1 vLLM `--enable-lora`
- One base model in GPU; adapters loaded into a small LoRA pool.
- Per-request `lora_request` selects adapter; switching is microseconds.
- Adapter weights ~50–500MB each; GPU holds ~4–16 simultaneously, others on CPU.

### 8.2 Use cases
- Per-tenant fine-tunes in B2B SaaS.
- Per-task adapters (summarize, classify, extract) on one base.
- Smaller fine-tunes serving many users at one base-model cost.

### 8.3 Limits
- All adapters share the base; can't mix base models in one engine.
- Quality of LoRA depends on training; not free.

---

## 9. Speculative decoding

A small draft model proposes K tokens; the big model verifies in one forward pass; accept the matching prefix.

- **Wins:** 1.5–4× decode speedup on memory-bound workloads (decode is bandwidth-bound, so verifying K tokens at once is nearly free).
- **Cost:** draft model takes GPU memory + a bit of compute; engineering complexity.

Variants:
- **Medusa:** multiple draft heads on the same model.
- **Lookahead decoding:** generate from N-grams of own history.
- **EAGLE:** stronger drafts via auxiliary model.
- **Online speculative tuning:** match draft to real distribution.

Provider implementations: vLLM has it; TRT-LLM has it; SGLang has it.

When to enable: workloads with high decode-to-prefill ratio (long answers, agents).

---

## 10. Parallelism strategies

For models that don't fit on one GPU:

### 10.1 Tensor parallelism (TP)
- Split each layer's weights across GPUs.
- Forward pass requires all-reduce across GPUs per layer.
- Best within a node (NVLink); falls off across nodes.
- Common: TP=2, 4, 8.

### 10.2 Pipeline parallelism (PP)
- Different layers on different GPUs.
- Activations passed between stages.
- Better across nodes; worse single-request latency (sequential).
- Combined with TP: TP within node, PP across nodes.

### 10.3 Data parallelism (DP)
- Replicate the model on N GPUs; each handles different requests.
- Linear scale-out for throughput; no help for one big model on small GPUs.

### 10.4 Expert parallelism (EP)
- For MoE models. Different experts on different GPUs; route tokens.
- Mixture of all-to-all + DP/TP. Complex.

### 10.5 The decision tree
```
Model fits on 1 GPU? → DP across GPUs for throughput.
Doesn't fit on 1 GPU, fits on 1 node? → TP across GPUs in node.
Doesn't fit on 1 node? → TP within node + PP across nodes.
MoE? → add EP layer.
```

Modern serving engines (vLLM, TRT-LLM) handle TP+PP for you; you set flags.

---

## 11. GPU economics (cloud, 2026)

### 11.1 Common SKUs
| GPU | HBM | TFLOPs FP16 | $/hr (cloud) | Use |
|---|---|---|---|---|
| T4 | 16GB | 65 | $0.35–0.50 | Tiny models, light traffic |
| A10G | 24GB | 125 | $0.50–1.00 | 7B–13B, embedders, rerankers |
| L4 | 24GB | 121 | $0.70–1.00 | 7B–13B, Hopper-gen efficiency |
| L40S | 48GB | 362 | $1.00–2.00 | 13B–34B FP16, 70B INT4 |
| A100 80G | 80GB | 312 | $2.00–4.00 | 70B BF16 (with TP=2), still common |
| H100 SXM | 80GB | 989 | $3.00–5.00 | 70B BF16, 405B INT4 |
| H200 | 141GB | 989 | $4.00–7.00 | 70B at higher concurrency, 405B FP8 |
| B200 | 192GB | ~2200 | $6.00–10.00 | Newest; emerging in 2026 |

### 11.2 $/M tokens self-host vs API
Rule of thumb on H100, vLLM, BF16:

| Model | Self-host $/M tokens | API equivalent |
|---|---|---|
| Llama 3.1 8B | $0.05–0.15 | Together $0.20, OpenAI Mini $0.20 input |
| Llama 3.1 70B | $0.30–0.80 | Together $0.90, Sonnet-tier APIs $3+ |

**Break-even:** for 8B-class models, self-host wins beyond ~50M tokens/day. For 70B, ~200M/day.

Below break-even: APIs win on TCO (no GPU idle time, no ops). Above: dedicated GPU pays.

### 11.3 Spot vs on-demand
- Spot: 50–80% discount; preemption. Use for batch eval, fine-tuning jobs.
- On-demand: production interactive serving.
- Reserved/committed: 1-year commit can hit 30–40% off on-demand.

---

## 12. Autoscaling

### 12.1 The challenge
LLM serving is stateful (KV cache per active request) and slow to start (model load = 30s to 5min for 70B).

### 12.2 KEDA on queue depth
Standard pattern:

```
gateway → request queue → vLLM pods
                          ↑
                          KEDA scales pods on queue depth
```

When queue grows past threshold, scale up. When idle, scale down.

### 12.3 Cold start mitigation
- **Warm pool:** keep N pods always alive even at zero traffic.
- **Predictive scaling:** scale on time-of-day curve (B2B traffic patterns).
- **Pre-pulled images:** weights cached on a high-speed local volume.
- **Tiered: hot/warm/cold.** Few hot pods serve baseline; warm pre-pulled; cold spin up on demand.

### 12.4 Serverless inference
- Modal, Replicate, Banana, Beam — auto-spin per request.
- Cold start ~10–30s for 7B; ~1–3min for 70B.
- Good for low-traffic, irregular workloads. Bad for sub-second SLAs.

### 12.5 Provider-side
- Together, Fireworks, Anyscale dedicated endpoints: you pay for reserved GPUs; they handle scaling within your reserved capacity.

---

## 13. Latency targets

For an interactive chat with retrieval:

| Stage | Budget |
|---|---|
| Auth + setup | 10ms |
| Embed query | 30–80ms |
| Retrieve | 5–30ms |
| Rerank | 100–400ms |
| Build prompt | 5ms |
| **LLM TTFT** | **200–1500ms** |
| Stream rest | 30–100 tok/s |

For LLM TTFT specifically:
- Short prompt + small model: <300ms.
- Long prompt + 70B model: 1–2s.
- Long prompt + reasoning model: 5–60s+.

**Levers if TTFT is too high:**
- Smaller model (or tier-route).
- Prompt cache (cache hit slashes prefill).
- Shorter context.
- Speculative decoding (helps decode, not TTFT directly).
- Self-host on faster GPU.
- Pin to closer region.

---

## 14. Observability for LLM serving

### 14.1 Metrics
- **Per-request:** TTFT, total latency, input tokens, output tokens, cache hit, $cost.
- **Per-engine:** GPU utilization, HBM utilization, KV cache utilization, batch size, queue depth.
- **Per-model:** RPS, error rate, p50/p95/p99 latency.

### 14.2 Prometheus exporters
vLLM and TGI ship `/metrics` endpoints with the above. Scrape and dashboard.

### 14.3 Tracing (OpenTelemetry)
- Span per LLM call with `gen_ai.*` attributes (request.model, response.tokens, etc.).
- Bridge to LangSmith / Langfuse / Phoenix for AI-flavored views, or Tempo / Honeycomb for general infra.

### 14.4 Alerts
- TTFT p95 > target.
- KV cache utilization > 90% (about to OOM).
- Queue depth growing > scaling rate.
- Error rate > 1%.
- $/req trending up week-over-week.

---

## 15. Failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| OOM mid-traffic | KV cache exceeded budget | Lower `gpu_memory_utilization`; cap context length; smaller `max_num_seqs` |
| Latency spikes after a few hours | Memory fragmentation (rare with PagedAttention; watch CPU/RAM) | Restart pod; monitor; raise alert before breach |
| All-reduce slow on multi-GPU | NCCL on suboptimal interconnect | Pin to NVLink-equipped nodes; check NCCL env |
| Queue grows unboundedly | No backpressure to clients | Add queue cap with 429; alert |
| Cold start too slow | 70B model loading | Warm pool; pre-pulled volume |
| Quality drop after deploy | New quant scheme; new model version | Revert; eval before rollout |
| One-region overload | No regional routing | Add second region; route by latency or tenancy |
| Sudden cost spike | Long-output requests; runaway loop | Per-request `max_tokens` cap; per-tenant token budget |
| Slow tokens/sec | Compute-bound on prefill, decode-bound on decode | Look at prefill vs decode latency separately; right-size GPU |

---

## 16. Hosted inference platforms (the buy alternative)

### 16.1 General serving
- **Together AI:** open-model serving; per-token pricing; dedicated endpoints option.
- **Fireworks:** similar; strong perf focus.
- **Anyscale Endpoints:** Ray-based; mixed open-model + custom.
- **Replicate:** API for arbitrary models; cold-start tradeoff.
- **Modal:** serverless functions with GPU; great for custom code + custom models.
- **Hugging Face Inference Endpoints:** managed deploy of any HF model.

### 16.2 Provider hosted (proprietary)
- OpenAI, Anthropic, Google, Cohere — black-box but easiest.

### 16.3 When to buy vs build
- Buy when: <100M tokens/day, no special model needs, need to ship fast.
- Build when: scale economics flip, need custom models, need data residency, need predictable latency.

---

## 17. The senior interview answer

If asked "how would you serve this model in production":

1. **Pick engine** based on workload: vLLM default; SGLang for prefix-heavy; TRT-LLM for peak NVIDIA.
2. **Size GPU** with the math (weights + KV at concurrency target).
3. **Pick precision** based on quality bar (BF16 default; FP8/INT4 if cost-sensitive).
4. **Set autoscale** on queue depth + warm pool for baseline.
5. **Multi-LoRA** if N adapters; one engine instead of N.
6. **Provider failover** even when self-hosting (route to API on degradation).
7. **Observability:** Prometheus from vLLM + OTel traces.
8. **$/M tokens** estimate; compare to managed alternative; defend the choice.

Senior signal: you reason from first principles (memory math, decode-vs-prefill, batching) not just buzzwords.

---

## 18. Self-check

- [ ] Compute KV cache memory for 70B model, 4k context, 8 concurrent users.
- [ ] Explain PagedAttention in 30 seconds.
- [ ] Articulate when SGLang beats vLLM.
- [ ] Walk through self-host vs API break-even math for an 8B model.
- [ ] Explain TP vs PP vs DP and when each applies.
- [ ] Name 3 ways to reduce LLM TTFT.
- [ ] Describe a multi-LoRA serving setup.
- [ ] List 3 alerts you'd set on a vLLM deployment.

---

Next: [`04-multi-agent-orchestration.md`](./04-multi-agent-orchestration.md)
