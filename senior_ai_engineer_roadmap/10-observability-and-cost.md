# Observability and Cost

> Tracing, metrics, evaluation feedback, and FinOps for LLM systems. Senior signal: you instrument like a distributed-systems engineer, not a notebook hacker.

---

## 1. The three signals (plus one)

Classic observability has three pillars: **metrics**, **logs**, **traces**. LLM systems add a fourth: **evaluations** (offline + online quality signals tied to traces).

| Signal | Question it answers | LLM-specific examples |
|---|---|---|
| **Metrics** | Is the system healthy now? | RPS, latency P50/P95/P99, error rate, token rate, cost rate |
| **Logs** | What happened in detail? | Full prompts + responses, tool calls, decisions, retrieval results |
| **Traces** | How did this request flow through? | LLM call → tool → retrieval → tool → LLM, with spans |
| **Evals** | Was the result *good*? | Faithfulness, helpfulness, refusal correctness, task success |

You need all four. Most teams ship metrics + logs and discover too late that without traces + evals they can't debug agent failures.

---

## 2. OpenTelemetry for AI (the foundation)

**OpenTelemetry (OTel)** is the open standard for traces / metrics / logs. The GenAI semantic conventions standardize span names and attributes for LLM workloads:

- `gen_ai.system` — provider (openai, anthropic, ...).
- `gen_ai.request.model`, `gen_ai.response.model`.
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`.
- `gen_ai.request.temperature`, `top_p`, `max_tokens`.
- `gen_ai.response.finish_reasons`.
- Operation types: `chat`, `text_completion`, `embeddings`, `tool` (`gen_ai.tool.name`).

**OpenLLMetry** (Traceloop) and **OpenInference** (Arize) are SDKs that auto-instrument popular libraries (OpenAI, Anthropic, LangChain, LlamaIndex, vLLM, etc.) into OTel spans. Pair with any OTel backend: Jaeger, Tempo, Honeycomb, Datadog, Phoenix, Langfuse, LangSmith.

Senior posture: **emit OTel-compliant traces from day one**. Don't lock into a vendor's proprietary trace format that you can't move off.

---

## 3. LLM observability platforms

| Platform | Strengths | Notes |
|---|---|---|
| **LangSmith** | Tight LangChain/LangGraph integration; datasets + evaluators; production tracing | Hosted; commercial; the default for LangChain shops |
| **Langfuse** | Open-source; self-hostable; OTel-compatible; prompt mgmt + evals | Strong choice when self-host or open-source is a constraint |
| **Phoenix (Arize)** | Open-source; OpenInference + OTel; RAG + agent evals built-in | Easy local dev; good for research / iteration |
| **Helicone** | Proxy-based; minimal code change (point base URL); cost dashboards | Lightweight onboarding; less rich on agent traces |
| **Traceloop** | OTel-native via OpenLLMetry; ships to any OTel backend | Best when you already have OTel infra |
| **Arize / Galileo / Patronus** | Enterprise; ML + LLM unified; eval suites | More than observability; eval + governance |
| **Datadog / Honeycomb / New Relic** | Generic APM with LLM extensions | Best when you already pay for them and want one pane |
| **PromptLayer** | Prompt registry + tracing | Strong on prompt iteration UX |

### Picking
- **Tight LangChain stack, hosted OK** → LangSmith.
- **Self-host required / open-source preference** → Langfuse or Phoenix.
- **Zero-code-change drop-in** → Helicone proxy.
- **One pane with existing APM** → Datadog / Honeycomb LLM extensions.
- **Eval-first culture** → Phoenix + LangSmith combo is common.

A pragmatic stack many teams converge on: **OpenLLMetry / OpenInference SDK → OTel collector → Langfuse (or LangSmith) for LLM-specific UX + a generic APM for infra spans**.

---

## 4. What to log on every LLM call

Mandatory:
- Trace ID + parent span ID.
- Model + provider + version (`claude-sonnet-4-6-2026-XX`, `gpt-5-2026-XX`).
- Prompt + response (with PII redaction policy).
- Token counts (input, output, cached) + cost.
- Latency (TTFT, total).
- Tool calls and results within the span tree.
- User / tenant / session / use-case tags.
- Outcome: success, error, refused, fallback.

Highly recommended:
- Prompt version / hash.
- Eval scores (when computed online).
- Retrieval results (chunk IDs, scores) for RAG.
- Cache hit / miss + cache key.
- Feature-flag values that affected the route.
- Cost estimates pre-call (for budgeting decisions).

Redaction policy is policy-driven, not ad-hoc. Some teams keep full prompts in a short-retention bucket (7 days) for debugging and emit a redacted version to long-term storage.

---

## 5. Trace structure for agents

A good agent trace is a tree:

```
session_id: s_abc
└── request: req_001 (root span)
    ├── input.preprocess
    ├── policy.check (input classifier)
    ├── plan.llm_call (model: haiku, 200/100 tokens, 240ms)
    ├── execute.step_1
    │   ├── tool: search (180ms)
    │   ├── retrieval (vector: 80ms, rerank: 60ms)
    │   └── llm_call (sonnet, 1.2k/400 tokens, 900ms)
    ├── execute.step_2
    │   └── tool: send_email (HITL approval pending → resumed)
    ├── critic.llm_call (sonnet, 800/200 tokens, 700ms)
    └── output.postprocess
```

Sessions group related requests; requests are root spans; each LLM call, tool call, retrieval is a child span. With this tree you can debug "why did the agent loop?" by replaying spans.

LangSmith / Langfuse / Phoenix all visualize this tree. Make sure your code emits structured parent/child relationships, not flat logs.

---

## 6. Metrics that matter

### Health
- Request rate, error rate, latency P50/P95/P99 per route, per model.
- Tool error rate per tool.
- Provider error rate (OpenAI 5xx, Anthropic 5xx, ...).
- Queue depth (if you have async workers).

### Quality
- Online eval score (sampled live traces).
- Thumbs up/down rate, escalation rate, abandonment rate.
- Refusal rate (split benign vs adversarial when possible).
- Tool-call accuracy (where ground truth available).

### Cost
- $ / hour, $ / request, $ / session, $ / tenant, $ / use-case.
- Tokens / hour input + output.
- Cache hit rate.
- % cost from each model tier.

### Capacity
- Tokens / second per region / per provider.
- Rate-limit headroom (calls remaining / minute).
- GPU utilization (if self-hosting).

### Agent / RAG-specific
- Steps per task, loop rate, max-hop hits.
- Retrieval Recall@K (online proxy via citation correctness).
- Reranker drop rate.

Set **SLOs** on the user-visible metrics (latency P95, error rate, task success) and burn-rate alerts. Internal metrics (token rate, cache hit) inform capacity planning, not paging.

---

## 7. Cost attribution + FinOps

### Per-tenant attribution
Every LLM call must carry a `tenant_id` tag. Aggregations:
- Daily $ per tenant; alert on top-N spike.
- $ per active user per tenant (catch single-user abuse).
- Margin per plan tier (free vs pro vs enterprise).

### Per-feature attribution
Tag calls with `use_case` (`onboarding_summary`, `support_classify`, `voice_intent`). Lets product / finance see which features are profitable vs subsidized.

### Budgets and quotas
- Per-tenant TPM / RPM caps in code (token bucket).
- Per-user daily $ caps for free tier.
- Soft warning at 80%, hard cutoff at 100%, custom upsell flow.

### Anomaly detection
- Day-over-day per-tenant variance > 3σ → alert.
- Per-user runaway loop detector (calls per session > threshold).
- Unexpected model use (free tier hitting Opus → bug or abuse).

### Negotiation rhythm
- Quarterly review with each provider (Anthropic, OpenAI, Google).
- Volume tier discounts; committed-use discounts; enterprise agreements.
- Multi-provider leverage — concrete fallback wired in lets you negotiate.

### Levers to cut cost (in order of impact)
1. **Right-size the model** — Haiku where Sonnet was unnecessary.
2. **Prompt caching** (Anthropic, OpenAI) — 50-90% off on cached input tokens.
3. **Shorter prompts** — system prompt length reviewed quarterly.
4. **Batch / async** for non-interactive paths (Anthropic Batch, OpenAI Batch — 50% off, 24h SLA).
5. **Compression** — RAG context smaller, summarized history.
6. **Output capping** — `max_tokens` set per use case; agents don't need 4000-token replies.
7. **Self-host** — for high-volume narrow tasks (embeddings, classification, simple QA).
8. **Provider fine-tunes** are usually NOT a cost win unless volume is large; commercial pricing eats the savings.

Detail in `03-llm-serving-infra.md` §13 for self-host break-even.

---

## 8. Logging hygiene

### Volume control
- Sample full prompt+response (e.g., 10%); log all metadata always.
- Tier retention: hot (7-30d full traces), warm (90d aggregates), cold (year+ aggregated metrics).
- Storage cost adds up — a chatty agent can produce GB/day per heavy tenant.

### Privacy
- PII redaction before write (regex + classifier hybrid).
- Tenant-scoped indexing — staff queries can't cross tenants without explicit grant.
- Right-to-deletion — pipelines to purge a user's traces on request.
- Encryption at rest + in transit, SSO + RBAC on the observability platform.

### Search + replay
- Every trace addressable by trace ID linkable to a session, user, tenant.
- Replay button in your internal tool that re-runs an exact trace against a candidate prompt / model — invaluable for debugging.

---

## 9. Online eval pipelines

Production traces are eval gold:
- **Sample N% per day** of production requests.
- **Run an eval suite** asynchronously (LLM-as-judge or programmatic checks).
- **Publish scores** as time-series; alert on drops.
- **Curate** — promote interesting failures into the offline eval set.

This closes the loop: production → eval → improvements → production.

Tools: LangSmith online evaluators, Langfuse evals, Phoenix evals, custom Python pipelines wired to the trace store.

---

## 10. Alerting

### What to page on (waking someone up)
- Provider total outage (>X% errors for Y minutes).
- Latency P95 breach beyond SLO with sustained burn-rate.
- Cost spike — per-tenant or aggregate.
- Safety incident detector (jailbreak classifier flagging > N/min).
- Catastrophic quality drop (online eval falls a lot in short window).

### What to ticket (next business day)
- Slow drift in metrics.
- Specific tenants degrading.
- Eval set drift > threshold.
- Per-tool error rate climbing.

### Alert hygiene
- Every alert has a runbook link.
- No alert without owner.
- Burn-rate alerts (multi-window) over single-threshold to reduce noise.
- Synthetic checks every minute for primary endpoints.

---

## 11. Dashboards (the senior ones)

A senior IC builds dashboards that answer **decisions**, not vanity:

1. **System health** — RPS, latency P95, error rate per route. Burn-rate annotation.
2. **Cost** — $ today vs MTD vs forecast, by use-case + by tenant + by model.
3. **Quality** — online eval scores by use-case + by slice; thumbs / escalation.
4. **Per-tenant** — top-20 by traffic, by cost, by error rate.
5. **Provider** — per-provider error rate + latency + cost; failover events.
6. **Agent** — steps per task, loop rate, HITL rate, max-hop hits.
7. **Change correlation** — deploy markers + flag flips overlaid on metrics so triage starts at "what changed".

Avoid: 50-panel dashboards no one looks at. Prefer: 5 dashboards that get used daily.

---

## 12. Capacity planning + forecasting

- Token rate per use-case → forecast monthly cost; flag plan-vs-actual variance.
- Per-provider headroom — what % of rate limit you're using; when do you negotiate up?
- GPU forecasting (if self-host) — utilization, queue depth, autoscaling behavior.
- Growth scenarios — at 3× traffic, do quotas hold? At 10×?

Use this in the quarterly biz review. Senior IC owns this.

---

## 13. Senior interview talk track

"How do you observe an LLM agent in production?"
1. **OTel from day one** — vendor-neutral; GenAI semantic conventions; OpenInference / OpenLLMetry SDK.
2. **Trace tree** — session → request → spans for plan / tool / retrieval / critic. Trace IDs propagate across services.
3. **What's in every span** — model, tokens, latency, cost, version, tags (tenant / use-case / user).
4. **Metrics + SLOs** on user-visible signals; burn-rate alerts; runbooks linked.
5. **Online eval** — sampled traces auto-scored; drift alarms; failures graduated into offline eval set.
6. **Cost FinOps** — per-tenant + per-feature attribution; quotas + anomaly detection; quarterly negotiation rhythm.
7. **Privacy** — PII redaction, tenant scoping, retention tiers, deletion plumbing.
8. **Tool + platform pick** — name a stack (Langfuse + Datadog + Phoenix evals), justify against constraints.

If they ask "how would you find a 10% latency regression": deploy markers, P95 by route + version, span breakdown, identify the slowest stage, candidate causes (provider drift, prompt growth, tool slowness, cold start), evidence collected from traces.

---

## 14. Self-check
- Can you name OTel GenAI semantic-convention attributes from memory?
- Can you sketch a trace tree for an agent with plan / tool / critic?
- Do you know your per-tenant cost attribution plan?
- Can you name 5 levers to cut LLM cost in priority order?
- Do you have alerts configured for the right things vs the noisy things?
- Can you compare LangSmith / Langfuse / Phoenix and pick one for a constraint?

---

**Next:** `11-fine-tuning-handson.md` — when to fine-tune vs RAG, SFT / LoRA / QLoRA / DPO, data prep, deployment.
