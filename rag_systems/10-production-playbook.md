# 10 — Production Playbook

> The 0 → 100% accuracy roadmap. The operations checklist. The failure-mode atlas. What turns a working RAG into one that survives real users at scale.

By doc 09 you have every technique. This doc is **what order to apply them in, how to know when to stop, and what to monitor once you're live.**

---

## 1. The 0 → 100% accuracy roadmap

Build in stages. Measure between every stage. Don't stack three changes at once — you won't know which helped.

### Stage 0 — Naive RAG (Day 1, ~50–65% Hit@10)

```
load → fixed 1000-char chunks → OpenAI text-embedding-3-small
     → pgvector → top-5 → stuff into prompt → GPT-mini answer
```

Build the eval set first (30–50 queries with expected chunks). Measure:
- Hit@10, Recall@10
- Faithfulness, answer relevance
- Latency p50/p95
- Cost per query

This is your baseline trend. Every change goes against this.

### Stage 1 — Better chunking (~+5–10 pts)

- Switch fixed → recursive character splitter, 512 tokens, 50 overlap.
- Preserve heading hierarchy in metadata.
- Use a real parser for PDFs (Unstructured / LlamaParse / Marker — pick by document type).
- Strip boilerplate, keep tables as markdown.

Re-run eval. If Hit@10 drops on a slice, your chunks went too small or too big for that slice — use slice analysis to decide.

### Stage 2 — Hybrid retrieval (~+5–15 pts)

- Add BM25 (Postgres `tsvector`, OpenSearch, or Elasticsearch).
- Run both, fuse with RRF (k=60).
- Over-fetch top-50 from each.

This is one of the highest-ROI changes in the entire stack. Skipping hybrid is the most common reason production RAG plateaus.

### Stage 3 — Reranker (~+10–25 pts)

- `bge-reranker-v2-m3` self-hosted, or Cohere Rerank 3.5 API.
- Rerank top-100 candidates → top-10.
- Calibrate score threshold for refusal path.

Now your top-10 is genuinely good. From here, gains are in single-digit points and need targeted work.

### Stage 4 — Grounded prompt + structured output (~+5 pts on faithfulness, big drop in hallucinations)

- "Use ONLY context" rule.
- Citations per claim, structured output enforced via tool/JSON schema.
- Refusal rule when threshold not met.
- Self-check on high-stakes queries.

This rarely moves Hit@10 — it moves *answer-side* metrics (faithfulness, citation accuracy, refusal correctness).

### Stage 5 — Contextual Retrieval (~+5 pts retrieval)

- LLM-generated 1–2 sentence prefix per chunk at ingest.
- Re-index. (One-time backfill cost.)
- Anthropic measured 35–67% reduction in retrieval failures.

Single highest-leverage advanced technique. Add it before you reach for graph/agentic RAG.

### Stage 6 — Query rewriting + decomposition (~+3–10 pts on hard slices)

- Conversational rewrite for multi-turn.
- Decomposition for compound queries.
- Self-querying for metadata extraction.
- Optional HyDE for underspecified queries.

Targets specific failing slices identified in eval. Don't enable globally — gate per query type.

### Stage 7 — Tier routing (cost win, neutral quality)

- Cheap classifier picks model tier (Haiku / Sonnet / Opus-thinking).
- 60–80% of traffic on the cheap tier without quality hit.
- 5–10× cost reduction.

### Stage 8 — Advanced patterns where eval points (the long tail)

Pick the pattern from doc 09's failure-mode lookup based on which slice still fails. Common picks:
- Multi-hop slice → agentic RAG.
- Layout-heavy docs → ColPali.
- Relationship-heavy domain → GraphRAG.
- Coverage gaps → CRAG with web fallback.
- Domain jargon → fine-tune embedder.

You may never need stage 8. Most production systems live happily at stage 5–7.

### Indicative gains, cumulative

| Stage | Hit@10 (typical) |
|---|---|
| 0 — Naive | 0.50–0.65 |
| 1 — Chunking | 0.55–0.72 |
| 2 — Hybrid | 0.65–0.82 |
| 3 — Reranker | 0.78–0.92 |
| 4 — Grounded prompt | (faithfulness +5–15) |
| 5 — Contextual | 0.85–0.96 |
| 6 — Query ops | 0.88–0.97 |
| 7 — Tier routing | (cost −60–80%) |
| 8 — Advanced | 0.92–0.98+ |

Numbers vary by domain. The slope matters more than the absolute.

---

## 2. The failure-mode atlas

When something breaks, walk this table from symptom to suspect.

### 2.1 Retrieval problems

| Symptom | Suspect | Diagnose | Fix |
|---|---|---|---|
| Right chunk in DB, never retrieved | Wrong distance metric, missing query/doc prefix, wrong embedder for domain | Check `input_type`/prefix; verify cosine vs L2; eval against BM25 baseline | Apply prefix; switch metric; switch model |
| Recall craters after deploy | New embedder, old vectors mixed in | Check `embed_version` distribution in payload | Reindex; tag every vector with version |
| Multi-tenant leaks | Filter only in app code | Check DB query — is filter pushed down? | Use filter-aware index (pgvector WHERE, Qdrant payload) |
| Top-K returns <K | Selective filter, post-filtered | Log retrieved count vs requested | Over-fetch + filter; switch to filter-aware ANN |
| Latency spike during writes | HNSW deletes piling up | Check tombstone count; rebuild metrics | Trigger compaction; rebuild during low-traffic window |
| Same query, different ranks across calls | ANN non-determinism on tied scores | Fingerprint result set; check query+seed | Usually OK; pin seed if reproducibility critical |

### 2.2 Reranker problems

| Symptom | Suspect | Fix |
|---|---|---|
| Reranker returns top-1 with low score | Threshold not enforced | Add cutoff → refuse if score < τ |
| Reranker very slow | Running on CPU; long chunks | GPU; chunk smaller; batch larger |
| Reranker swaps positions every release | Model version unpinned | Pin reranker version; track in eval |
| Quality dropped after enabling rerank | Too few candidates fed in | Over-fetch more (top-100 → top-200) |

### 2.3 Generation problems

| Symptom | Suspect | Fix |
|---|---|---|
| Confident hallucination | "ONLY context" not in prompt; threshold not enforced | Tighten system prompt; refuse path |
| Refuses despite info present | Threshold too high; context labels confusing | Lower threshold; add few-shot positive |
| Wrong citation index | Indices changed between rerank and prompt | Stable IDs end-to-end |
| Cites chunks that don't exist | Model invented numbers | Force structured output; post-validate |
| Verbose / repetitive | No max_tokens | Cap output; lower temp |
| Trusts pretraining over context | Model behavior | "Context overrides any prior knowledge" rule |
| Drift mid-conversation | History accreting; instructions diluted | Re-anchor system prompt; summarize history |
| JSON schema breaks | Asked nicely, didn't enforce | Use structured output mode (OpenAI / Anthropic tool use) |

### 2.4 Pipeline-wide

| Symptom | Suspect | Fix |
|---|---|---|
| Quality drops without code changes | Drift: corpus grew, models updated upstream, queries shifted | Continuous eval; alert on metric deltas |
| Costs balloon overnight | Re-embed migration with no guardrails; tier routing broken | Daily cost dashboard; per-stage cost split |
| One tenant complains | Tenant-specific drift | Per-tenant eval slice; check filter selectivity |
| Latency p99 huge | Reranker batching off; long chunks; cold cache | Profile per-stage; warmup; smaller chunks |
| Cache hit rate drops | Query distribution shift; TTL too short | Inspect dropped keys; tune TTL |

---

## 3. The production checklist

Before you launch:

### 3.1 Ingest
- [ ] Pinned versions: parser, chunker, embedder. Stored on every chunk.
- [ ] Idempotent ingest — same source produces same chunk IDs.
- [ ] Content hash on each chunk for incremental re-ingest.
- [ ] Retry + dead-letter for failed parses.
- [ ] Per-tenant queues (no noisy-neighbor stalls).

### 3.2 Storage / Index
- [ ] Vector store backups (snapshot daily, weekly cold copy).
- [ ] Source corpus stored separately and authoritatively.
- [ ] Index parameters documented (HNSW M, ef_construction, distance metric).
- [ ] Filter-aware index for tenant_id + any high-selectivity field.
- [ ] Quantization decision documented (full / int8 / binary).

### 3.3 Retrieval
- [ ] Hybrid (BM25 + dense + RRF) wired.
- [ ] Reranker pinned to version, batch size set.
- [ ] Score threshold per (model, version) recorded.
- [ ] Over-fetch budget for filtered queries.
- [ ] Query rewrite for multi-turn (if applicable).

### 3.4 Generation
- [ ] Single source-of-truth system prompt, version-tracked.
- [ ] Structured output enforced (schema-bound).
- [ ] Citation post-validation.
- [ ] Refusal path tested with adversarial queries.
- [ ] Output safety filter (PII, toxicity, prompt injection).

### 3.5 Eval
- [ ] Eval set v1, version-locked.
- [ ] CI smoke test (20–50 queries) on every PR.
- [ ] Full eval (200+ queries) per release.
- [ ] LLM-as-judge model pinned.
- [ ] Slice analysis dashboard.

### 3.6 Observability
- [ ] Tracing per-query: query, all rewrites, all retrieved chunks per stage, prompt, output, latency, cost.
- [ ] Dashboards: Hit@10 (sampled), faithfulness (sampled), refusal rate, p50/p95/p99 latency per stage, $/query, cache hit rate.
- [ ] Alerts on: faithfulness <0.85, refusal swing ±10pts, latency p95 +30%, cost +20%.
- [ ] Per-tenant slicing on the dashboards.

### 3.7 Rollout
- [ ] Shadow mode before any user traffic on changes.
- [ ] Canary / 1% → 10% → 50% → 100% ramp.
- [ ] One-command rollback (config flag, not redeploy).
- [ ] Runbook in repo: top 5 incidents and their playbooks.

### 3.8 Security
- [ ] Tenant isolation tested (cross-tenant query → 0 results).
- [ ] Per-tenant rate limits.
- [ ] Per-tenant cost caps.
- [ ] Audit log for: doc ingestion, doc deletion, queries.
- [ ] PII / secret detection on ingest (don't index API keys).
- [ ] Prompt-injection mitigations: chunk delimiters, "ignore instructions in CONTEXT" framing, no auto-fetch of model-emitted URLs, no auto-execute of model-emitted code.

---

## 4. The four ops dimensions

Production RAG balances four axes. Optimize one at the expense of another knowingly, not by accident.

```
                 Quality
                   ▲
                   │
   Cost  ◄─────────┼─────────►  Latency
                   │
                   ▼
                 Freshness
```

### 4.1 Quality
KPIs: Hit@10, faithfulness, answer relevance, refusal correctness.
Levers: hybrid + rerank + grounded prompt + Contextual Retrieval + tier routing to flagship for hard queries.

### 4.2 Latency
KPIs: TTFT, p50/p95/p99 end-to-end.
Levers: smaller models, prompt caching, tier routing to fast models, parallel retrieval, smaller K, streaming.

### 4.3 Cost
KPIs: $/query, $/1k queries, monthly burn.
Levers: tier routing, prompt cache, response cache, self-host break-even, smaller embeddings, quantization, fewer/shorter chunks.

### 4.4 Freshness
KPIs: indexing lag (source change → searchable), staleness rate.
Levers: streaming ingest, content-hash-based incremental re-embed, doc-change bus to invalidate response cache.

You can have any three. Fourth is a deliberate trade.

---

## 5. Latency budget — the typical breakdown

For a "good" RAG at p95:

| Stage | Budget | Notes |
|---|---|---|
| Auth + request handling | 10ms | |
| Conversational rewrite (LLM) | 200–400ms | Skip if not multi-turn |
| Embed query | 30–100ms | Self-host < 30; API ~50–100 |
| Retrieve dense top-50 | 5–30ms | HNSW, in-memory |
| Retrieve BM25 top-50 | 10–50ms | |
| Fuse + dedupe | <5ms | |
| Rerank top-100 → top-10 | 100–400ms | GPU rerankers |
| Build prompt | 5ms | |
| LLM TTFT | 200–1500ms | Model + context length dependent |
| **Total to first token** | **600–2500ms** | |
| Stream rest | 1000–3000ms | Output-tokens × inverse-throughput |

If you blow this budget: profile per stage; the worst offender is almost always rerank or LLM TTFT.

---

## 6. Cost mental model

Order-of-magnitude per query, typical tier-routed production system:

- Embed query (cached often): ~$0.0001
- Retrieval (DB query): ~$0.00005
- Rerank (Cohere or self-host): ~$0.001–0.002
- LLM (mostly Haiku/mini): ~$0.005–0.020
- LLM (Sonnet for hard queries): ~$0.020–0.050
- LLM (Opus thinking, rare): ~$0.10–0.30
- Tracing / logging: ~$0.0001

Blended typical: **$0.01–0.03/query** with smart routing. **$0.05–0.10/query** without.

For a 1M queries/day system, that's $10k–30k/day vs. $50k–100k/day. Tier routing is the single biggest lever.

Self-host crossover: typically ~50–200M LLM tokens/day. Below that, APIs win on TCO. Above, dedicated GPU starts paying.

---

## 7. Re-embedding migrations

You will do this 2–3 times in any serious project:

1. New index (different name).
2. Backfill: re-embed every chunk into the new index. Track progress; resumable.
3. Dual-write all new chunks to old + new during backfill.
4. When backfill complete: dual-read with shadow comparison. Log differences.
5. Cut over reads. Keep dual-write for a week as a rollback.
6. Drop old index.

For 10M chunks at $0.02/1M tokens × 500 tokens/chunk = ~$100. Trivial. The risk isn't cost — it's downtime and silent quality regression mid-migration. Always shadow-compare before cutover.

---

## 8. Multi-tenant scaling

Three operating modes by tenant size:

**Small tenants (1k–10k chunks):** field-on-row in shared collection. `WHERE tenant_id = X`. Filter-aware index. Cheap, simple.

**Medium tenants (100k–1M chunks):** namespace per tenant in shared collection (Pinecone namespace, Qdrant payload index). Logical isolation, shared physical resources.

**Large tenants (>10M chunks, regulated, or paying enough):** dedicated collection or dedicated cluster. Total isolation, per-tenant index tuning.

Migration paths between modes happen as tenants grow. Pre-design for it.

**Per-tenant guardrails:**
- Rate limit (queries/sec).
- Token budget (per query, per day).
- Concurrency cap.
- Cost cap with notification.
- Eval slice — measure quality per tenant separately.

---

## 9. Incident playbooks (the three you'll have)

### 9.1 "Quality dropped overnight"
1. Check eval drift dashboard. Which metric, which slice?
2. Diff the last 24h of: code, model versions, embedder/reranker versions, corpus updates.
3. If embedder/reranker version bumped → roll back.
4. If corpus update touched many docs → check ingest pipeline; specifically if Contextual Retrieval prefix generation degraded.
5. If query distribution shifted → not a regression, it's drift; eval set may need new queries.

### 9.2 "Costs doubled overnight"
1. Per-stage cost dashboard. Which stage?
2. If LLM: check tier router — is it routing everything to flagship? Misclassified queries?
3. If embedder: did re-ingest run unintentionally? Check ingest queue depth.
4. If rerank: candidate set inflation? Is over-fetch K runaway?
5. If cache hit rate dropped: query distribution shift; consider raising TTL.

### 9.3 "Latency p99 spiked"
1. Per-stage latency dashboard. Which stage?
2. If rerank: GPU saturation? Batch backed up?
3. If LLM: provider issue? Switch to backup provider/region.
4. If retrieval: HNSW index bloat from deletes? Compaction needed.
5. If embed: API region degraded? Failover.

Build these into a runbook before launch. The first time you debug at 2 AM, you'll thank yourself.

---

## 10. The "is RAG working" health score

A composite KPI to track per release on the dashboard:

```
RAG_health = 0.30 · Hit@10
           + 0.25 · faithfulness
           + 0.15 · answer_relevance
           + 0.10 · refusal_correctness
           + 0.10 · (1 - p95_latency_normalized)
           + 0.10 · (1 - cost_per_query_normalized)
```

Weights are illustrative — pick weights that match your business. The point is: **one number tracked over time** that captures multi-dimensional health, plus the underlying components when you need to drill in.

Healthy production RAG sits at 0.80+ on this scale. Anything <0.70 means a slice is broken.

---

## 11. What you don't need

Common over-engineering to skip:

- **Multiple vector DBs unless you have a clear reason.** One pgvector or one Qdrant covers 95% of projects.
- **Custom-trained embedders day one.** Off-the-shelf for the first 6 months. Fine-tune only after you've maxed every other knob.
- **GraphRAG when your corpus isn't relationship-heavy.** Most docs aren't. RAPTOR doesn't help short docs.
- **Agentic RAG for FAQ queries.** 2× the cost, 2× the latency, no quality win.
- **Long-context "RAG-less" for serious corpora.** Doesn't scale economically.
- **Multi-stage rerankers.** One good cross-encoder is enough until eval says otherwise.
- **Chains of multiple LLMs everywhere.** Each link adds latency and a failure mode.

Every added component has to earn its place on the eval dashboard.

---

## 12. The "ship it" rubric

A RAG system is ready for production when:

1. **Eval set ≥ 100 queries**, version-locked, real (not synthetic-only).
2. **Hit@10 ≥ 0.85** on the eval set, with no slice <0.65.
3. **Faithfulness ≥ 0.90** with refusal correctly enabled.
4. **Refusal rate**: 0% on answerable slice, ≥80% on unanswerable slice.
5. **p95 latency ≤ 2s** to first token.
6. **Cost/query** within budget with 3× headroom for traffic growth.
7. **Tracing** every query end-to-end.
8. **One-command rollback.**
9. **Runbook** for the three incident types above.
10. **Tenant isolation** verified by adversarial testing.

Anything missing → not ready. Ship the missing piece first.

---

## 13. After launch — the perpetual loop

```
Monday:   Review last week's drift dashboard. Pick worst slice.
Tuesday:  Sample 10 failures from that slice. Diagnose.
Wed:      Form hypothesis. Add failing queries to eval set.
Thursday: Implement fix. Run eval. Compare deltas.
Friday:   Ship if better, rollback if worse. Repeat.
```

This is the job. Most of RAG quality after launch comes from **a slow accretion of fixed slices**. Each fix permanent because it's gated by an expanding eval set.

After 6 months: hundreds of curated edge cases, a stable system, and a team that knows exactly what would break it.

---

## 14. Keyword cheat-sheet

- **Roadmap stages 0–8** — the canonical build order.
- **Health score** — composite metric for dashboards.
- **Drift** — quality degradation without code changes.
- **Per-stage tracing** — observability of every pipeline component.
- **Tenant isolation / per-tenant guardrails** — multi-tenancy ops.
- **Re-embedding migration** — versioned switch of the embedder.
- **Tier routing** — per-query model size selection.
- **Failure-mode atlas** — symptom → suspect → fix lookup.
- **Production checklist** — pre-launch readiness gate.
- **Incident playbook** — runbook for the three common breakages.
- **Ship-it rubric** — the 10-item gate.

---

## 15. Recap — the whole journey

You started with raw docs and a question. Through 10 documents:

1. **Foundations** — what RAG is and what it solves.
2. **Ingestion + chunking** — turning sources into useful units.
3. **Embeddings** — making meaning numerical.
4. **Vector stores** — finding nearest neighbors fast.
5. **Retrieval** — getting the right candidates back.
6. **Reranking + context** — ordering them so the LLM reads them right.
7. **Generation + citations** — turning context into trustworthy answers.
8. **Evaluation** — proving any of it works.
9. **Advanced patterns** — what to add when baseline ceilings.
10. **Production playbook** — operating it in the real world.

Every stage is a decision tree. Every decision tree has trade-offs. Every trade-off has a metric to measure it by.

That's RAG. The rest is execution and iteration.

Good luck.

---

**Folder index:**
- [README](./README.md)
- [01 — Foundations](./01-foundations.md)
- [02 — Ingestion and Chunking](./02-ingestion-and-chunking.md)
- [03 — Embeddings](./03-embeddings.md)
- [04 — Vector Stores](./04-vector-stores.md)
- [05 — Retrieval Strategies](./05-retrieval-strategies.md)
- [06 — Reranking and Context](./06-reranking-and-context.md)
- [07 — Generation and Citations](./07-generation-and-citations.md)
- [08 — Evaluation](./08-evaluation.md)
- [09 — Advanced Patterns](./09-advanced-patterns.md)
- [10 — Production Playbook](./10-production-playbook.md)
