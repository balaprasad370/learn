# 09 — Observability and Tracing

You cannot debug what you cannot see. An agent runtime that ships without a debugger is unfixable in production.

This document defines what is observed, how it is recorded, how it is queried, and how an operator goes from "this run behaved oddly" to "here is the exact line of reasoning that caused it."

Reading prerequisite: [02 — Data Model](./02-data-model-and-contracts.md) (especially the `Event` type and event catalog) and [08 — Runtime Infrastructure](./08-runtime-infrastructure.md).

---

## Three pillars, one spine

Observability typically means metrics, logs, and traces. For an agent runtime, **the trace is the spine**; metrics and logs exist but always in service of "explain what this run did."

- **Trace** — ordered, structured log of a run. Replayable. Primary tool.
- **Metrics** — aggregated counters, histograms, gauges for dashboards and alerts.
- **Logs** — unstructured/semi-structured lines for operational debugging (worker startup, DB pool, etc.). Secondary.

If a feature fits naturally as "an event in the trace," make it an event — don't invent a sidecar log.

---

## What we emit (and in what shape)

The canonical unit is the `Event` defined in [02](./02-data-model-and-contracts.md):

```
Event(id, trace_id, span_id, parent_span_id, run_id, step_id, type, payload, at, tenant_id)
```

Every event is:
- **Typed** — drawn from a closed enum (see the catalog in [02](./02-data-model-and-contracts.md)).
- **Structured** — payload has a versioned JSON Schema per event type.
- **Addressable** — has an id and participates in a span tree.
- **Durable** — written to Postgres `events` table atomically with the step that emitted it.
- **Streamable** — published to the event bus so SSE clients see it live.

---

## Span model

We use OpenTelemetry conventions for spans. A span represents "a unit of work with start and end." Spans nest to form a tree.

```
Span(
  trace_id,
  span_id,
  parent_span_id,
  name,                          # e.g., "planner.llm_call"
  kind,                          # server | client | internal
  started_at, ended_at,
  status,                        # ok | error
  attributes: dict[str, Any],
  events: list[Event],           # embedded sub-events
  links: list[SpanLink],         # cross-trace references (e.g., to sub-agent run)
)
```

### The canonical span tree of a run

```
run (root span)
├── step.1
│   ├── planner.prompt_compile
│   ├── planner.llm_call
│   │   └── provider.anthropic.chat
│   ├── policy.check_plan
│   ├── executor.dispatch
│   └── checkpoint.commit
├── step.2
│   ├── planner.prompt_compile
│   ├── memory.query
│   │   ├── embed.call
│   │   └── vector.search
│   ├── planner.llm_call
│   ├── policy.check_plan
│   ├── executor.dispatch
│   │   └── tool.invoke:web_search
│   │       ├── policy.check_action
│   │       ├── ratelimit.check
│   │       ├── http.call
│   │       └── schema.validate_output
│   └── checkpoint.commit
└── step.3
    └── ...
```

Reading this tree top-to-bottom tells you exactly what the agent did, in what order, and how long each piece took. That is the job.

### Span attributes — the naming convention

Follow OTel semantic conventions where they apply; invent only where we must.

Standard keys we always set:
- `tenant_id`, `agent_id`, `agent_version`, `run_id`, `step_ordinal`, `principal_id`
- `runtime.version`, `runtime.region`
- `model.provider`, `model.id` (for LLM spans)
- `tool.id`, `tool.version`, `tool.side_effects` (for tool spans)
- `memory.namespace`, `memory.kind` (for memory spans)
- `cost.tokens.prompt`, `cost.tokens.completion`, `cost.usd`
- `error.code`, `error.retryable`, `error.message` (on status=error)

Keys we DO NOT set at the span level:
- Raw prompts (stored in step payload with redaction; link by step id).
- Raw tool outputs (same).
- PII of any kind. If it must be in observability, it must be redacted first.

---

## Event payload schemas (representative samples)

Every event type has a JSON Schema; here are the important ones.

### `planner.completed`
```json
{
  "type": "planner.completed",
  "run_id": "run_...",
  "step_id": "stp_...",
  "payload": {
    "strategy": "react",
    "model": "claude-opus-4-7",
    "tokens": {"prompt": 2812, "completion": 147},
    "latency_ms": 1834,
    "action_kind": "tool_call",
    "tool_id": "web_search@1.2.0",
    "thought_hash": "sha256:...",
    "plan_size": 1
  }
}
```
(Full thought text lives in the step payload, hashed here for cross-reference.)

### `tool.invoked`
```json
{
  "type": "tool.invoked",
  "payload": {
    "tool_id": "web_search@1.2.0",
    "tool_call_id": "tlc_...",
    "transport": "http",
    "args_hash": "sha256:...",
    "args_size_bytes": 142,
    "side_effects": "read",
    "idempotency_key": "sha256:..."
  }
}
```

### `tool.completed`
```json
{
  "type": "tool.completed",
  "payload": {
    "tool_call_id": "tlc_...",
    "ok": true,
    "duration_ms": 417,
    "retries": 0,
    "output_size_bytes": 4812,
    "output_preview": "[{\"title\":\"...\""   // truncated, redacted
  }
}
```

### `llm.response`
```json
{
  "type": "llm.response",
  "payload": {
    "provider": "anthropic",
    "model": "claude-opus-4-7",
    "finish_reason": "tool_use",
    "tokens": {"prompt": 2812, "completion": 147, "cache_read": 2600},
    "cost_usd": "0.0247",
    "latency_ms": 1834,
    "safety_signals": []
  }
}
```

### `memory.retrieved`
```json
{
  "type": "memory.retrieved",
  "payload": {
    "namespace": "research",
    "query_hash": "sha256:...",
    "hits": [
      {"id": "mem_...", "score": 0.81, "kind": "semantic"},
      {"id": "mem_...", "score": 0.74, "kind": "episodic"}
    ],
    "retrieval_latency_ms": 32,
    "fused": true
  }
}
```

### `policy.triggered`
```json
{
  "type": "policy.triggered",
  "payload": {
    "guard": "pii_redact",
    "stage": "input",
    "action": "redacted",
    "count": 2,
    "severity": "low"
  }
}
```

### `run.budget_exceeded`
```json
{
  "type": "run.budget_exceeded",
  "payload": {
    "dimension": "max_tokens",
    "limit": 200000,
    "used": 201432
  }
}
```

All event payloads are additive — new optional fields are fine; renames are not. Past runs must remain decodable.

---

## Where events are written

Three sinks, all fed from the same emission:

1. **`events` table in Postgres.** Primary, durable. Partitioned by month.
2. **Event bus** (Redis Pub/Sub or Streams) for live consumers (SSE, eval pipelines, alerting).
3. **Metrics pipeline** (derived). A small set of events produces counters/histograms.

Emission happens inside the worker after the step commit, transactionally for (1) and fire-and-forget for (2). (Mis-delivery to (2) doesn't lose the event — it's still in Postgres.)

---

## Sampling

Traces for normal runs are 100% sampled. Sampling kicks in only for:
- Very high-volume tenants where cost of storage matters.
- Specific event types that are inherently noisy (e.g., per-token `llm.stream` deltas — sampled at 1% by default; users can enable full).
- Tenant-configured (debug mode: all; normal: keep everything but drop `llm.stream` deltas after 7 days).

We do NOT head-sample (decide at trace start whether to keep). We tail-sample (keep all initially; age out by tenant retention). An agent bug discovered a week late still has its trace.

---

## Redaction

Everything we store must survive a tenant's data-residency and PII rules.

At emission time, a redaction pipeline runs on any field that MAY contain user data:
- Event payload `output_preview`, raw prompts, tool args.
- Span attributes marked `sensitive=true`.

Redaction strategies:
- **Tokenize** — replace with a stable hash (e.g., "user email X" → `email:sha256:...`), reversible only by a separate key that ops hold. Great for joining across events without exposing the value.
- **Mask** — replace with `[REDACTED:email]`. Lossy but safe.
- **Drop** — omit the field entirely.

Config per tenant; default is "mask." Runtime emits `policy.triggered` when redaction fires so operators see it happened.

---

## Debugger UI — the most important thing we ship

A trace viewer is the product-facing face of observability. Minimum features:

1. **Run timeline** — horizontal bar chart; each step is a bar; hover shows duration, cost, outcome.
2. **Span tree** — collapsible OTel-style tree, click to expand, pin a span to the side panel.
3. **Step detail panel** — shows the full, redacted prompt; the planner's raw response; parsed action; execution result.
4. **Memory diff view** — before/after memory operations; which records were retrieved, which were written, which were superseded.
5. **Tool invocation view** — args, output, retries, duration.
6. **Cost breakdown** — tokens and dollars per step and per tool.
7. **Run comparison** — diff two runs side-by-side (same agent, different inputs; or same input, different agent versions).
8. **Replay** — step forward/back; re-run from any step with a frozen model (eval mode).
9. **Multi-agent tree** — for handoffs, the child runs are collapsible children under the parent step.
10. **Export** — download the trace as NDJSON for offline analysis.

Without these, operators will file "weird run" bugs without a link, and you will have to crawl Postgres to help. Build the UI once, save every future debugging session.

---

## Metrics

Metrics are derived from events. We emit counters, histograms, and gauges via Prometheus.

### Business metrics (what product cares about)
- `runs_total{status, agent_id}` — counter
- `run_duration_seconds{agent_id}` — histogram
- `run_cost_usd{agent_id, tenant_id}` — counter
- `steps_per_run{agent_id}` — histogram
- `tools_invoked_total{tool_id, ok}` — counter
- `memory_hits_per_query{namespace}` — histogram
- `policy_triggered_total{guard, stage, action}` — counter

### Operational metrics (what ops cares about)
- All of [08](./08-runtime-infrastructure.md)'s ops metrics.
- `events_emitted_total{type}` — counter; should grow at a steady rate.
- `events_sink_lag_seconds` — gauge; time between emit and sink write.
- `trace_query_latency_ms` — histogram.

Golden signals per tenant: request rate, error rate, latency, saturation. Dashboards per agent_id surface the first three.

---

## Alerts

Alerts fire on derived conditions, not on single events.

Examples:
- `rate(runs_total{status="failed"}[5m]) / rate(runs_total[5m]) > 0.05` for 10m — "elevated failure rate."
- `histogram_quantile(0.95, run_duration_seconds) > target_p95` — "latency regression."
- `rate(run_cost_usd[1h]) > tenant_hourly_cap` — "cost spike."
- `rate(policy_triggered_total{action="halt"}[5m]) > 10` — "sudden policy halts — investigate."

Every alert has a runbook link. No runbook, no alert.

---

## Cost accounting

Cost is a first-class observability concern. Every LLM call and every paid tool invocation contributes to:

- Per-run ledger: `runs.budget_used.dollars` (authoritative).
- Per-tenant rollup (materialized view, refreshed hourly).
- Per-agent rollup (for "which of my agents is expensive?").

The ledger answers "where did the money go?" at any granularity — step, run, agent, tenant, day.

### Pricing provenance
Every cost entry stores:
- `pricing_table_version`
- `model_id`
- `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_write_tokens`
- Computed `cost_usd` (Decimal, never float)

Historical re-computation uses the pricing valid at the time — never current prices.

---

## Trace replay

Given a trace id, we can re-construct the run step by step.

```
1. Load all events for trace_id, ordered by (at, id).
2. Reconstruct initial state from `run.created` payload.
3. For each step's events:
    - Build the exact planner prompt from `planner.prompt_compile` payload.
    - Feed to LLM in deterministic mode (temp=0, seed from run_seed).
    - Compare actual output to recorded output.
    - If diverged: produce a diff view — "the model would have said X now, but said Y then."
    - If matched (or replaced with recording): apply action, repeat.
4. Optional: replay with a changed component (different model, different tool, different prompt) to see what would have happened.
```

Replay is the foundation of both debugging ("why did this run fail?") and eval ("does the new agent beat the old on the same 500 cases?"). See [11 — Evaluation and Testing](./11-evaluation-and-testing.md).

---

## Data retention

| Tier | Location | Default retention | Configurable |
|------|----------|-------------------|--------------|
| Hot traces | Postgres `events` partitions | 14 days | per tenant, up to 90 days |
| Warm traces | Object store NDJSON gz | 90 days | per tenant, up to 2 years |
| Cold archive | Object store (Glacier class) | per tenant plan | yes |
| Metrics | Prometheus / long-term store | 30 days @ 10s; 1 year @ 1m | ops-configurable |
| Logs | Log store | 30 days | ops-configurable |

Deletion is GDPR-compliant: a per-subject purge script walks hot+warm+cold and removes records tied to the subject, preserving structural aggregate metrics.

---

## Correlating across the stack

A request lands in API server with a `trace_id` (either the client's or a newly-generated one); that id propagates everywhere — HTTP headers (`traceparent`), event bus messages, DB rows, worker logs. One id, one story.

A support ticket that includes a `run_id` gives us everything: the trace, the state blob, the events, the metrics at that time, the worker logs. No "can you reproduce?" — we already have the reproduction data.

---

## Making observability cheap

Every feature here costs bytes. Controls:

- **Redaction before write** avoids oversized PII fields.
- **Compress** on-the-wire to sinks (gzip on event bus and storage).
- **Summarize** `llm.stream` deltas after 24h into a single `llm.stream_summary` per span.
- **Cold-tier** old partitions after 14 days — queryable but slow; cheap.
- **Batch writes** of events to the `events` table in small groups per step commit.

Target: under 1 KB of stored observability per step after compression, excluding raw prompts/outputs.

---

## The one-paragraph rule

If you cannot write a paragraph from the trace explaining what a run did and why, observability is incomplete. Every time the team hits a run they can't explain, file a "missing observability" issue and close it by adding the event or attribute that would have made it explainable. This is how observability gets good.

---

## Next: [10 — Safety and Governance](./10-safety-and-governance.md). Observability tells you what happened; governance decides what is *allowed* to happen in the first place.
