# 02 — System Design for AI

> The actual interview format at AI startups. You will be asked to design one of these. Walk through the framework, then practice the prompts.

---

## 1. The framework

A 45–60 minute system-design round at an AI company has the same arc as a regular SDI, but with AI-specific bottlenecks dominating the trade-off discussion.

```
1. Clarify scope         (3–5 min)
2. Estimate scale        (3 min)
3. Sketch APIs           (5 min)
4. Sketch data model     (5 min)
5. Core flow end-to-end  (10 min)
6. Bottlenecks + scaling (10 min)
7. AI-specific concerns  (10 min)
8. Trade-offs / what'd you change at 100×  (5 min)
```

**Senior signal:** you spend more than a third of the time on **bottlenecks + AI-specific concerns + trade-offs**. Junior engineers spend it all on boxes-and-arrows.

---

## 2. The clarifying-questions cheat sheet

Always ask, in this order:

1. **Who's the user?** (developer / end-customer / internal team) — shapes API surface.
2. **What's the read/write ratio?** (chat is 1:1; agent runs are write-heavy; RAG is read-heavy.)
3. **What's "good enough" latency?** (interactive: <2s; voice: <800ms; batch: minutes OK.)
4. **Scale?** (RPS, MAU, doc count, query volume.)
5. **Multi-tenant?** (B2B SaaS = yes; B2C = no.)
6. **Quality bar?** (consumer chat = best-effort; medical / legal = high; voice agent = no hallucinations OK.)
7. **Cost ceiling?** ($/query, monthly burn.)
8. **Build vs buy?** (early stage = buy; scale stage = consider self-host.)

Don't ask all eight. Ask 3–4 that shape the design most.

---

## 3. The estimation cheat sheet

You should be able to back-of-envelope these without a calculator:

### 3.1 Tokens
- 1 word ≈ 1.3 tokens (English).
- 1 page of prose ≈ 500 tokens.
- 1 codebase file ≈ 1k–10k tokens.
- A typical chat message ≈ 50 tokens; a typical answer ≈ 300–800 tokens.

### 3.2 Latency
- Embed query: 30–80ms (API), <30ms (self-host).
- Vector search top-100 on 10M chunks (HNSW, in-memory): 5–30ms.
- BM25 top-100: 10–50ms.
- Cross-encoder rerank top-100: 100–400ms.
- LLM TTFT: 200–1500ms (model + context dependent).
- LLM output: 30–100 tokens/sec (fast tier), 5–30 tokens/sec (reasoning tier).

### 3.3 Cost
- Cheap LLM tier (Haiku/Mini): $0.005–0.020 per chat answer.
- Default tier (Sonnet/GPT-5): $0.020–0.050.
- Reasoning tier (Opus thinking): $0.10–0.30+.
- Embedding (1k tokens): ~$0.00002 (small) – $0.00013 (large).
- GPU $/hr (cloud): A10G $0.5–1, L4 $0.7–1, L40S $1–2, H100 $3–5, H200 $4–7.

### 3.4 GPU memory budget
- LLM weights (FP16): ~2 bytes × params. 7B = 14GB. 70B = 140GB.
- Quantized (INT4): ~0.5 bytes × params. 70B = 35GB.
- KV cache (per token): `2 × n_layers × n_heads × d_head × 2 bytes`. ~50–500KB/token.
- For 70B with 8 concurrent users × 4k context: ~10–20GB extra.

### 3.5 Throughput
- vLLM on 1× H100 with Llama 8B: ~5000 tokens/sec aggregate batch.
- Single user: ~70 tokens/sec.
- 70B on 4× H100: ~1500 tokens/sec aggregate.

### 3.6 Storage
- 1M chunks × 1024-dim vectors (fp32): ~4GB. (int8: 1GB.)
- 1M chunks of text (~500 tok each): ~2GB raw.
- 1M page-image embeddings (ColPali): ~50–100GB.

---

## 4. The seven canonical AI design prompts

### Prompt 1: "Design RAG-as-a-service"

**Clarify:**
- Multi-tenant or single? (Multi.)
- BYO docs or BYO storage? (BYO docs uploaded to us.)
- Self-serve or enterprise? (Both — start self-serve.)
- Streaming UX? (Yes.)

**Scale assumption:** 10k tenants, avg 100k chunks each = 1B chunks total; 100 QPS sustained.

**API:**
```
POST /v1/datasets/{id}/documents     (upload)
POST /v1/datasets/{id}/query         (retrieve + answer, SSE stream)
GET  /v1/datasets/{id}/eval-runs     (eval results)
```

**Data model (Postgres + vector DB):**
- `tenants`, `datasets`, `documents` (Postgres).
- `chunks` (Qdrant collection per tenant tier or shared with payload index).
- `query_logs` (Postgres for cheap analytics; ClickHouse if heavy).

**Core flow:**
1. Ingest: upload → S3 → SQS → ingest worker (parse, chunk, contextualize w/ Anthropic, embed) → Qdrant + Postgres.
2. Query: API → conversational rewrite (cheap LLM) → BM25 + vector retrieval (top-50 each) → fuse (RRF) → cross-encoder rerank → top-10 → Sonnet generation with citations → SSE stream.

**Bottlenecks:**
- **Ingest:** parse + embed throughput. Bound by embedding API. Solve: batch + prefetch.
- **Query latency:** rerank + LLM TTFT dominate. Solve: lighter reranker for fast tier, prompt cache, tier routing.
- **Cost:** LLM dominates. Solve: cheap tier for 70%+ traffic via classifier; cache responses; prompt cache.

**Multi-tenancy:**
- Pool model with tenant_id payload index in Qdrant for small/medium tenants.
- Dedicated collection for top-100 enterprise tenants.
- Per-tenant rate limit + token budget cap.

**AI-specific:**
- Per-tenant eval set; nightly RAGAS run; alert on faithfulness drop.
- Re-embedding migration plan (versioned `embed_model_version` on every chunk).
- Provider failover (Anthropic primary, OpenAI fallback).
- Prompt injection mitigation in retrieved context.

**At 100×:** sharded vector store, dedicated GPU pool for self-hosted reranker + embedder, regional deployment with data residency.

---

### Prompt 2: "Design a multi-tenant agent platform" (Lyzr-style)

This is your home turf — you've already documented it in [`lyzr_agent_runtime/`](../lyzr_agent_runtime/). Walk through:

1. **API surface:** REST + SSE + WebSocket; runs as durable resources; idempotency keys.
2. **State:** Postgres for run metadata, Redis Streams for execution, S3 for artifacts.
3. **Execution:** state-machine-driven runs, atomic step checkpointing, durable queues with reclaim.
4. **Agent loop:** planner → executor → tools, with budget caps.
5. **Memory:** short-term (in-run), long-term (vector + structured KV), per-tenant.
6. **Tool calling:** schema + sandbox + idempotency.
7. **Multi-agent:** handoff via tool, supervisor pattern.
8. **Safety:** 7-hook policy engine (input filter, output filter, tool allow-list, budget, rate, audit, jailbreak detect).
9. **Observability:** trace per run, span per step, cost attribution per tenant per run.
10. **Eval:** trajectory eval, capability tests, regression suite.
11. **Deploy:** k8s + KEDA on queue depth, multi-region, GPU pools for self-hosted models.
12. **Extension:** named extension points for planner, executor, memory, tool transport, LLM provider, etc.

**The senior move:** don't read out the architecture — reason from first principles. "Why durable runs? Because agent steps can take seconds to minutes; a pod restart shouldn't lose work."

---

### Prompt 3: "Design a voice agent at scale" (Giga-style)

**Clarify:**
- Inbound or outbound? (Both.)
- PSTN or web? (Both — Twilio + WebRTC.)
- Latency target? (<800ms turn-to-turn.)
- Concurrent calls? (Start 1k, scale to 100k.)

**Stack:**
```
Caller ↔ Twilio (PSTN) / WebRTC (web)
       ↔ Media server (LiveKit / Pipecat session)
            ↔ STT streaming (Deepgram / Whisper)
            ↔ Agent loop (Sonnet + tools + RAG)
            ↔ TTS streaming (Cartesia / ElevenLabs)
       ↔ Caller speakers
```

**Latency budget (per turn):**
| Stage | Target |
|---|---|
| VAD detect speech end | 200ms |
| STT finalize | 100–200ms |
| LLM TTFT | 200–400ms |
| TTS first audio chunk | 100–200ms |
| Network jitter | 50–100ms |
| **Total turn time** | **650–1100ms** |

**Critical engineering:**
- **Barge-in:** monitor caller VAD continuously; cancel TTS + LLM mid-stream when speech detected.
- **Streaming everything:** STT streams partial transcripts; LLM streams tokens; TTS streams audio chunks. Don't wait for completion at any stage.
- **Connection pooling:** persistent gRPC to STT/TTS; warm LLM connections (provider gateway).
- **Pipeline overlap:** start TTS on first sentence of LLM output, not full output.

**Failure handling:**
- STT silence too long → "Are you still there?" prompt.
- LLM provider 5xx → retry once, then fall back to OpenAI.
- TTS failure → fall back to slower TTS provider.
- Tool call slow → confirmational filler ("Let me check that for you...").

**Multi-tenancy:**
- One LiveKit room per call.
- Tenant config (voice, prompt, tools) loaded from cache.
- Per-tenant concurrent call cap.

**Observability:**
- Per-call trace: VAD events, STT partials/finals, LLM tokens, TTS chunks, latency per stage.
- WER eval against transcript ground truth.
- Conversational quality eval (LLM-as-judge on transcripts).

**At 100k concurrent calls:**
- Sharded media servers (LiveKit horizontal scale).
- GPU pool for self-hosted Whisper if scale economics demand.
- Regional deployment for latency.

→ See [`06-voice-agents.md`](./06-voice-agents.md) for full depth.

---

### Prompt 4: "Design an LLM gateway / router"

**Goal:** single API in front of N LLM providers; routing, fallback, retries, observability.

**Features needed:**
- Provider abstraction (OpenAI, Anthropic, Gemini, Cohere, self-host).
- Routing policies (model alias → provider+model based on tier/cost/latency).
- Failover (provider 5xx → fallback chain).
- Retries with backoff + jitter.
- Rate limit aggregation (you can call provider via N keys; manage budget across them).
- Caching (response, prompt cache).
- Cost tracking per tenant per request.
- Streaming pass-through (SSE all the way).
- Audit log + tracing.

**Reference projects:** LiteLLM, Portkey, Helicone (proxy-based), OpenRouter (managed).

**Data model:**
- `requests` log (Postgres or ClickHouse if high volume).
- `policies` (Redis for hot lookup).
- `keys` (Vault).

**Critical engineering:**
- **Streaming:** must not buffer; chunk-forward SSE bytes. Cost tracking happens after stream completes.
- **Connection pool per provider** with separate breakers.
- **Header preservation:** OpenAI tools, Anthropic prefill, etc — don't lose provider-specific knobs.
- **Observability injection:** add `gen_ai.*` OTel spans without breaking client streams.

**Trade-off:** every gateway adds latency and a failure mode. Worth it once you hit 3+ providers or 5+ teams calling LLMs from many places.

---

### Prompt 5: "Design a prompt-versioning + eval pipeline"

**Use case:** product team iterates on prompts daily; engineering owns deploy gates.

**Workflow:**
```
PR with prompt change
    → CI runs eval set (~100 queries)
    → Compare to baseline; surface delta
    → Reviewer approves / rejects
    → Merge → prompt registered → canary rollout
    → Online monitoring for drift
```

**Components:**
- **Prompt registry:** versioned prompts in repo (or DB) with semantic IDs (`agent.support_assistant.v17`).
- **Eval runner:** loads dataset, runs candidate prompt + model, scores via RAGAS / LLM-judge / domain metrics.
- **Comparison view:** diff between baseline and candidate per query; aggregate stats; slice analysis.
- **Approval gate:** human or auto-merge if delta > threshold.
- **Registry → runtime:** push to Redis/feature flag system for hot-swap.
- **Rollback:** flip flag back; 30s recovery.

**Reference tools:** LangSmith, Braintrust, Promptlayer, Phoenix, custom (often).

**Key trade-offs:**
- Prompt-in-code (typed, versioned with code) vs prompt-in-DB (hot-update without deploy).
- LLM-judge cost: $1–10 per eval run × N PRs/day = real money.
- Eval set growth: must add failing prod queries continually (`rag_systems/10` Section 13).

---

### Prompt 6: "Design a fine-tuning pipeline"

**Use case:** weekly fine-tune from production logs to keep a small model competitive with a big one for a specific task.

**Workflow:**
```
Production logs
    → privacy filter (PII scrub) + sample
    → labeling (human / strong-LLM-as-teacher)
    → train/val split + hard-negative mining
    → SFT job (LoRA on Llama 8B, 1–4 hrs on 1× H100)
    → eval against held-out + domain capability tests
    → if pass: register adapter version → deploy via vLLM multi-LoRA
    → A/B against current adapter on 1% traffic
    → ramp if quality + cost OK
```

**Components:**
- **Data lake:** S3 partitioned by tenant + date.
- **Privacy gate:** Presidio or custom NER; secrets scanner.
- **Labeling:** Argilla / Lilac / custom UI; or distillation from teacher LLM.
- **Training:** Modal / Together / Anyscale / custom; Axolotl / TRL / unsloth.
- **Adapter store:** S3 + metadata DB; versioned.
- **Serving:** vLLM with `--enable-lora`; hot-swap adapters.
- **Eval:** capability suite + regression set.

**Critical:** **never** train on data you can't legally use. Per-tenant opt-in; strict PII scrub; tenant-scoped models for B2B.

→ Full depth in [`11-fine-tuning-handson.md`](./11-fine-tuning-handson.md).

---

### Prompt 7: "Design observability for a large LLM application"

**What's specific to LLM observability:**
- High cardinality (every conversation is unique).
- Costs vary 100× per request (Haiku vs Opus thinking).
- Quality is fuzzy and lagging (no immediate ground truth).
- Latency budget is tight; can't add 100ms of telemetry.

**Stack:**
- **Tracing:** OpenTelemetry with `gen_ai.*` semantic conventions; bridge to Tempo / Honeycomb / Datadog.
- **Per-LLM-call spans:** input tokens, output tokens, model, $cost, latency, cache hit/miss.
- **Per-conversation spans:** session ID, tenant, total $.
- **Logs:** structured per call (query, retrieved chunks, prompt rendered, raw output, citations validated, refusal yes/no).
- **Metrics:** TTFT histogram, total latency, $/req, refusal rate, cache hit rate, eval-set score (sampled, async).
- **Sampling:** 100% of errors, 1–5% of success (tunable per route).

**Dashboards:**
- Per-tenant: requests, $, latency, refusal rate, eval delta.
- Per-feature: same metrics, sliced.
- Per-model: latency, throughput, error rate per provider.
- Cost: $/req trend, cache hit rate, top-10 expensive sessions.

**Alerts:**
- Cost +20% week-over-week.
- Refusal rate ±10pts swing.
- Faithfulness <0.85 (sampled).
- p95 TTFT +30%.
- Provider error rate >2%.

→ Full depth in [`10-observability-and-cost.md`](./10-observability-and-cost.md).

---

## 5. AI-specific scaling levers (memorize)

When the interviewer pushes "now 10× the load":

| Lever | How it scales | Cost |
|---|---|---|
| Tier routing | More traffic to cheap model | -60–80% LLM $ |
| Prompt cache | Reuse static prefix | -50–90% input $ on hit |
| Response cache | Reuse exact answer | -100% on hit |
| Semantic cache | Cluster similar queries | -30–60% on similar queries; risk of staleness |
| Continuous batching | Higher GPU throughput | Same GPU = more QPS |
| Multi-LoRA serving | One engine, many adapters | -N× GPUs for N models |
| Quantization | Smaller model in same GPU | -2–4× GPU $; small quality dip |
| Reranker quantization / lighter model | Rerank more candidates per dollar | Marginal quality dip |
| Stream + cancel | Don't pay for unused tokens | -10–30% on long replies users abandon |
| Async eval | Don't block requests | Latency restored |
| Regional deployment | Lower latency, data residency | More infra $ |
| Provider routing | Cheap-region preference | -10–30% $ |

---

## 6. Senior trade-off articulation

Every design decision = a trade-off. Speak in this format:

> "I'd choose **X** because **Y**. The cost is **Z**, which is acceptable here because of **scope/scale assumption**. If **assumption flipped**, I'd switch to **alternative**."

Examples:

> "I'd start with pgvector. It scales to 10M chunks per node, which is plenty for the v1 scale you described, and it gives me transactions and joins for free. Cost is operational simplicity. If we cross 100M chunks or need filter-aware ANN at scale, I'd migrate to Qdrant or Milvus."

> "I'd self-host the cross-encoder reranker on a single GPU. Cohere Rerank API would be simpler but at 100 QPS we'd burn $200/day on it. Self-host pays for itself in two weeks. Trade-off: one more thing to operate."

> "I'd default to single-agent with tools, not multi-agent. Multi-agent doubles latency and complicates eval. The cases where it's worth it — separate domains, separate models, very long horizons — should be the exception, not the rule."

---

## 7. Senior failure modes in the interview

Common ways smart engineers blow this round:

1. **Boxes and arrows for 40 minutes; never gets to trade-offs.** Solution: spend max 15 min on the diagram.
2. **No estimation.** "How many GPUs?" → "Some" is wrong. Always do the math out loud.
3. **Over-engineering for v1 scale.** "Even at 1k QPS we need Kafka" — no, you don't.
4. **Ignoring multi-tenancy.** B2B AI = always multi-tenant. Build it in.
5. **Treating LLM as a black box.** Senior should reason about TTFT, batching, KV cache, cost.
6. **No observability story.** "We'll add monitoring later" is a red flag.
7. **No eval story.** "How do you know it's working?" → tracing dashboard + sampled LLM-judge.
8. **No rollback plan.** Every change needs a one-command rollback.
9. **No cost number.** "$/query" is the senior currency. Have one.
10. **Pronoun "we."** It's an interview about you. Use "I'd."

---

## 8. The structured ending

Last 5 minutes, always do:

1. **Recap** the design in 3 sentences.
2. **List the top 3 risks** and how you'd mitigate.
3. **List 2 things you'd do differently at 10× / 100× scale.**
4. **List 2 things you'd build in v2** (defer to ship faster).
5. Ask the interviewer: "Where would you push back hardest on this design?"

That last question is a senior signal — you welcome critique and want to learn.

---

## 9. Practice cadence

- **Weekly:** one full 60-min mock against the prompts in section 4. Record yourself. Review.
- **Bi-weekly:** mock with a peer or paid mock service.
- **Read post-mortems:** Anthropic, OpenAI, Cloudflare, GitHub. They show the actual failure modes you'll be asked about.
- **Build mini-versions:** sketch the architecture in a markdown file, then build the smallest possible version on a weekend. The fastest way to internalize.

---

## 10. Resources

- *System Design Interview Vol 1 + 2* — Alex Xu (canonical building blocks).
- ByteByteGo (Alex Xu's newsletter) — real architectures of well-known systems.
- Anthropic engineering blog, OpenAI cookbook — read both.
- *Designing Machine Learning Systems* — Chip Huyen (ML system design depth).
- HackerNews "Show HN" + YC Demo Day — see how 0→1 systems are pitched.

---

## 11. Self-check

Without looking back, can you:

- [ ] Walk through "design RAG-as-a-service" in 30 minutes with no notes.
- [ ] Estimate GPU memory for a 70B model serving 8 concurrent users.
- [ ] Justify pgvector vs Qdrant for a 5M-chunk multi-tenant system.
- [ ] Articulate the latency budget for a voice agent turn.
- [ ] Explain when single-agent + tools beats multi-agent.
- [ ] Sketch a prompt-version + eval CI pipeline.
- [ ] List 5 cost levers and quantify their impact.
- [ ] Give a structured ending with risks + v2 backlog.

If you can do all of these without crutches, you're ready for the round.

---

Next: [`03-llm-serving-infra.md`](./03-llm-serving-infra.md)
