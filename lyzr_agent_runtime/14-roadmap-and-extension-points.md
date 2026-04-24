# 14 — Roadmap and Extension Points

Prerequisite: all prior docs ([00](./README.md)–[13](./13-deployment-and-scaling.md)) define the runtime as it should exist at v1. This document is about **time**: what lands in v1 vs v2 vs later, where the runtime is designed to be extended without forking, what we explicitly will **not** build, and how we communicate evolution to builders on top of the platform.

A roadmap is not a promise. It is a statement of intent, and a boundary around what not to do this quarter.

---

## 1. Versioning philosophy

Three layers, versioned independently:

| Layer | Versioning | Stability guarantee |
|---|---|---|
| Public API (HTTP/SSE/WS contracts) | Semver with major = URL prefix (`/v1`, `/v2`) | **Strict.** No breaking changes within a major. 12-month parallel window on major change. |
| SDK (Python, TypeScript) | Semver | Strict within major. Tracks API major. |
| Internal APIs (planner/executor/memory/policy/tool/provider plugin interfaces) | Semver | Documented contracts; breaking changes in minor releases allowed but telegraphed 1 release ahead. |
| Data model (DB schema, event payloads at rest) | Schema version field | Forward-migrating. Never broken in place — expand/contract only (doc 13 §6.2). |

The public API is the load-bearing promise. Everything else can move faster.

---

## 2. What v1 ships

Minimum viable runtime — what's required to call this a platform, not a demo:

### 2.1 Planning and execution (doc 03)
- [x] Universal loop with ReAct and Plan-Execute strategies.
- [x] Step state machine with strict transitions.
- [x] Loop detector (repeated action + semantic similarity).
- [x] Retry at 3 layers with clear ownership.
- [ ] Reflexion strategy ships as opt-in preview.

### 2.2 Memory (doc 04)
- [x] STM with windowing + automatic summarization.
- [x] Episodic, semantic, procedural LTM.
- [x] Hybrid retrieval (vector + BM25 + RRF fusion + rerank + recency).
- [x] Namespace scope chain.
- [x] Forgetting (expiration, deprecation, importance decay).
- [ ] Cross-agent semantic memory sharing ships v1.1.

### 2.3 Tools (doc 05)
- [x] In-process, HTTP, and queue transports.
- [x] MCP transport bridge (read-only + write with explicit scope).
- [x] Idempotency key expressions (JMESPath).
- [x] Structured error codes with retryability.
- [x] Parallel tool invocation where model supports it.

### 2.4 Multi-agent (doc 06)
- [x] Handoff primitive (sync + async) with memory bridge and context policy.
- [x] Supervisor/worker, Sequential pipeline, Blackboard, Group chat.
- [x] Depth and cycle protection.
- [x] Budget fractions with fair splitting.
- [ ] Peer swarm pattern ships v1.1.

### 2.5 LLM provider layer (doc 07)
- [x] Anthropic, OpenAI, Gemini, Bedrock adapters.
- [x] Router with health, fallback, and cost preference.
- [x] Prompt caching with cache_control blocks.
- [x] Deterministic mode (seed + temperature=0 pinning).
- [ ] Local / self-hosted adapter (vLLM, TGI) ships v1.1.
- [ ] Provider-agnostic fine-tuning integration — post-v1.

### 2.6 Infrastructure (doc 08)
- [x] API / Worker / Cron split.
- [x] Redis Streams queue with consumer groups.
- [x] Postgres as source of truth with partitioned tables.
- [x] Atomic step commit transaction.
- [x] Pause / resume / cancel / inject.
- [x] Event bus with SSE fan-out.

### 2.7 Observability (doc 09)
- [x] OTel-compatible spans.
- [x] Canonical event taxonomy.
- [x] Per-step cost accounting.
- [x] Debugger UI with 10 core features.
- [ ] Trace diffing across runs — v1.1.

### 2.8 Safety and governance (doc 10)
- [x] Policy engine with 7 hook points.
- [x] Built-in guards (pii_redact, injection_detect, content_filter, tool_allowlist, depth_check, budget_precheck, secret_exfil_detector, size_cap, schema_enforce).
- [x] Human-in-the-loop primitive.
- [x] Red-team harness.
- [ ] Third-party-certified content filter integration (e.g., Lakera) — v1.1.

### 2.9 Evaluation (doc 11)
- [x] 5-tier evaluation structure.
- [x] Datasets as first-class resource.
- [x] 7 grader types.
- [x] Replay-based eval.
- [x] PR gates.
- [ ] Shadow-mode at scale with auto-sampling — v1.1.

### 2.10 API (doc 12)
- [x] REST + SSE + WebSocket surface.
- [x] Python + TypeScript SDKs.
- [x] Webhook delivery with HMAC signatures + retry.
- [x] OpenAPI 3.1 spec, machine-generated.

### 2.11 Deployment (doc 13)
- [x] Helm chart for Kubernetes.
- [x] KEDA autoscaling on queue.
- [x] Multi-tenancy (shared + dedicated namespace tiers).
- [ ] Dedicated single-tenant cluster mode ships v1.1.
- [ ] Bring-your-own-cloud deployment mode — post-v1.

### 2.12 Studio / UI
- [x] Run debugger (trace inspector, replay, diff).
- [x] Dataset editor.
- [x] Agent config editor with live eval preview.
- [x] Cost dashboard.

The defining characteristic of v1: **you can build and run a production agent without writing runtime code.** Everything else is evolution.

---

## 3. v1.1 — hardening release (target: +90 days post-v1)

Focus: reducing friction discovered in early v1 deployments.

1. Reflexion planner strategy promoted from preview.
2. Peer swarm coordination pattern.
3. Cross-agent semantic memory sharing with explicit namespace grants.
4. Local/self-hosted provider adapter (vLLM, TGI, llama.cpp server).
5. Trace diffing across runs (N-way comparison).
6. Shadow-mode at scale — automated sampling, shadow-to-prod divergence reports.
7. Dedicated single-tenant cluster deployment mode.
8. Third-party content filter integrations (Lakera, Azure AI Content Safety).
9. Tenant-managed LLM provider keys (BYOK).
10. Cost budget alerts with automatic throttling.

These are **hardening**, not **new capabilities**. The surface area stays the same; the depth increases.

---

## 4. v2 horizon (not yet committed; directional)

v2 is a year-plus away and involves breaking changes that justify a new major API. Topics under consideration, **not promises**:

1. **Streaming-first planner.** Today the planner emits a complete output then the executor acts. v2 explores interleaved streaming where tools can be invoked mid-token-emission (requires protocol-level support from providers we don't fully have yet).
2. **Graph-structured plans.** Today plans are lists of steps with implicit DAG between actions. v2 explores explicit DAG plans with native fork/join semantics, mirroring structures like LangGraph / AutoGen graphs but with the runtime's checkpoint/replay guarantees.
3. **Memory as a query plan.** Today retrieval is hybrid-by-default with a fixed pipeline. v2 explores a planner-emitted retrieval strategy per step (query decomposition, sub-retrievals, iterative expansion).
4. **First-class voice/video modalities.** Today multi-modal support is passthrough to providers that offer it. v2 explores modality-specific memory, tools, and policies.
5. **Durable WebSocket sessions.** Today WebSockets are single-socket ephemeral. v2 explores session resumption so mobile clients can drop/resume without losing their place.
6. **Agent-owned code execution.** A first-class sandboxed code-execution tool integrated end-to-end with provenance, reproducibility, and safety guards. Today it's a community pattern via external tool; v2 makes it native.

Explicitly **not** on the v2 horizon:
- A proprietary agent DSL. Agents stay as config-plus-plugins. No new language.
- A built-in LLM. Provider-independent is a permanent value.
- A proprietary vector DB. Pluggable stays pluggable.

---

## 5. Extension points

Extension points are the contract the runtime promises to keep stable so that ecosystem plugins, customer code, and enterprise integrations survive minor version changes.

### 5.1 Plugin interfaces (numbered so they can be referenced)

| # | Interface | Stability | Examples |
|---|---|---|---|
| E1 | `Planner` (doc 03 §5) | Stable | ReAct, Plan-Execute, Reflexion, custom strategies. |
| E2 | `Executor.invoke_tool` (doc 03 §6) | Stable | Rarely replaced; mostly hook points are used instead. |
| E3 | `MemoryStore` (doc 04 §7) | Stable | Postgres+pgvector, external Pinecone/Qdrant/Weaviate, custom. |
| E4 | `MemoryRetriever` (doc 04 §9) | Stable | Hybrid (default), semantic-only, custom ranking. |
| E5 | `ToolTransport` (doc 05 §6) | Stable | in-process, HTTP, MCP, queue, custom (e.g., gRPC). |
| E6 | `LLMProvider` (doc 07 §2) | Stable | Anthropic, OpenAI, Gemini, Bedrock, local. |
| E7 | `Router` (doc 07 §8) | Stable | Round-robin, cost-preferred, latency-preferred, custom. |
| E8 | `PolicyGuard` (doc 10 §3) | Stable | All built-ins + customer-authored guards. |
| E9 | `Grader` (doc 11 §4) | Stable | Built-ins + LLM-as-judge + custom (e.g., code execution). |
| E10 | `EventSink` (doc 09 §5) | Stable | Postgres (default), Kafka, S3, customer-owned webhook. |
| E11 | `TraceExporter` (doc 09 §5) | Stable | OTel OTLP, Honeycomb, Datadog, self-hosted. |
| E12 | `WebhookDeliverer` (doc 12 §8) | Stable | Default HMAC-signed HTTP, custom (e.g., SNS, Kafka, customer SQS). |
| E13 | `TokenCounter` (doc 07 §12) | Stable | Provider-specific, fallback tiktoken-based, custom. |
| E14 | `Sandbox` (doc 13 §12) | Stable | gVisor, Firecracker, Wasmtime, custom. |

Each extension point has:

1. A documented Protocol/interface in the SDK.
2. A stability tier (see §5.2).
3. A contract test suite the plugin must pass.
4. A reference implementation in tree.
5. A changelog tracked separately from the runtime's main changelog.

### 5.2 Stability tiers

| Tier | Meaning | Breaking change window |
|---|---|---|
| **Stable** | Production-ready. Safe to build against. | Never in minor. Major only, with 1 release deprecation window. |
| **Preview** | Feature complete; still tuning the interface. | May break in minor, but telegraphed 1 release ahead. |
| **Experimental** | In active design. Interface may change anytime. | No guarantees. Use at your own risk. |
| **Deprecated** | Still works. Being removed. | Removal date in `Sunset` header; minimum 6 months. |

Every Protocol and configurable in the runtime carries a `@stability("stable" | "preview" | "experimental" | "deprecated")` decorator in code and a corresponding tag in docs.

### 5.3 Plugin packaging

Plugins are Python packages (for in-process components) or independent services (for transport-layer components). Discovery via:

- `entry_points` in `pyproject.toml` — the canonical mechanism. Example:
  ```toml
  [project.entry-points."lyzr.guards"]
  my_custom_pii = "my_package.guards:CustomPIIGuard"

  [project.entry-points."lyzr.planners"]
  my_graph_planner = "my_package.planners:GraphPlanner"
  ```
- Runtime startup lists discovered plugins in `GET /v1/admin/plugins` (admin-only).
- Plugins run in the same process (trust boundary: plugin code is tenant-trusted in dedicated deployments; prohibited entirely in shared-tier unless it's a blessed one).

For out-of-process plugins (tools via HTTP/MCP, LLM providers via network), configuration is manifest-based — no code in the runtime pod.

### 5.4 Plugin lifecycle hooks

Every plugin type has three lifecycle methods:

```python
class Plugin(Protocol):
    def on_load(self, ctx: PluginContext) -> None: ...
    def on_reload(self, ctx: PluginContext) -> None: ...
    def on_unload(self, ctx: PluginContext) -> None: ...
```

`PluginContext` exposes:
- `config: dict` — merged config from agent + tenant + platform.
- `logger: Logger` — namespaced logger.
- `metrics: MetricsEmitter` — for plugin-specific metrics.
- `trace: TraceSpan` — for plugin-specific spans.
- `tenant: TenantHandle` — read-only tenant metadata.

Plugins **do not** get access to: database connections, Redis, object storage. They interact through the runtime's abstractions, not the data plane directly. This is load-bearing: a badly written plugin can't corrupt a tenant's data.

---

## 6. Deprecation policy

### 6.1 How a thing becomes deprecated

1. Announcement in changelog + `Deprecation: true` header on responses using the deprecated feature.
2. `Sunset` header with removal date (minimum 6 months out; 12 months for anything API-surface).
3. Deprecation warnings in SDK at call time.
4. Weekly digest to integrators using the deprecated feature.
5. After sunset, the feature returns `410 Gone` with `error_code: feature_sunsetted` for 30 days, then is removed.

### 6.2 Exceptions

- **Security-driven removals** may skip the warning window if the feature has an exploitable design. Emergency notice + replacement guidance. Rare.
- **Zero-usage removals** may remove faster if telemetry shows < 5 tenants with < 10 requests/month for 3 consecutive months. Still 90 days' notice.

### 6.3 Never-deprecated by policy

- ULID IDs. IDs are forever.
- Run status enum values (`queued`, `running`, `paused`, `succeeded`, `failed`, `canceled`, `awaiting_hitl`). New values may be added; existing values are never removed.
- Core event types (`step.started`, `step.completed`, `run.completed`, `run.failed`). New event types added freely; these five are forever.
- Idempotency semantics.
- `/v1` URL prefix (parallel to any future majors for 12+ months).

---

## 7. Community and plugin ecosystem

### 7.1 Intended dynamic

The runtime is the kernel. Most user-visible capability should be assemblable from plugins and configs. Internal teams ship reference implementations; community extends.

### 7.2 Plugin registry

A future (not v1) feature: an official plugin registry at `registry.lyzr.ai`. v1 relies on PyPI + documented entry_points.

### 7.3 Contribution tiers

| Tier | Who | Support |
|---|---|---|
| Core | Internal team | Guaranteed. |
| Official | Contributed by external devs, adopted in tree | Supported to stable tier. |
| Certified | Third-party, passing contract tests, listed in docs | Compatibility guaranteed; support by maintainer. |
| Community | Everyone else | As-is. |

### 7.4 Contract tests

Each extension point ships with a test suite that any plugin must pass to claim compatibility. Example for `PolicyGuard`:

```python
class GuardContractTests:
    def test_inspect_prompt_returns_allow_or_deny(self): ...
    def test_deny_includes_reason_and_error_code(self): ...
    def test_transform_is_reversible_or_logged(self): ...
    def test_hook_points_advertised_match_implementation(self): ...
    def test_no_network_calls_without_declared_egress(self): ...
    def test_p95_latency_under_50ms(self): ...
```

Plugins that don't pass contract tests don't claim "certified" status, and the runtime warns when loading them.

---

## 8. What we deliberately will not build

Important to say out loud. A platform's constraints are part of its identity.

1. **A proprietary agent DSL.** Agents are config + plugins. Not a new language. Every attempt at proprietary DSLs in this space has ended with users wanting to escape to code.
2. **Our own LLM.** Provider-independent forever. We route; we don't train.
3. **A proprietary vector database.** pgvector default, pluggable out. The vector DB market is fine without us.
4. **Low-code UI for end users** (non-developers composing agents via drag-drop). Studio is for developers. Non-developer tooling is a different product, not an extension of this runtime.
5. **Arbitrary long-running background agents without explicit schedules.** Agents run in response to inputs or cron triggers. "Always-on autonomous agents" are a thin wrapper on top of cron + memory, not a separate primitive.
6. **Auto-retraining loops.** Eval failures don't automatically fine-tune. Humans close the loop with approval.
7. **Mass deletion APIs.** Forgetting is per-namespace. No `DELETE /v1/tenants/{id}/memories`. Customers who need bulk delete use the export+re-import path, which leaves an audit trail.
8. **Cross-tenant intelligence sharing.** Whatever a tenant's agents learn stays in that tenant. We don't build "shared memory" or "global semantic index across all customers."
9. **Crypto primitives beyond what we need.** We use HMAC for webhooks, KMS for secrets, TLS everywhere. We do not roll crypto.

These aren't restrictions imposed by engineering debt. They are product decisions that keep the runtime coherent.

---

## 9. Telemetry-driven roadmap

The roadmap evolves on evidence, not opinion. Signals we act on:

| Signal | Source | Triggers |
|---|---|---|
| Feature requests | Public GitHub issues, customer success conversations | Prioritization review monthly. |
| Adoption of preview features | Usage metrics | Promotion to stable after ≥ 3 external users + 90 d stability. |
| Error rate on specific endpoints | API telemetry | Surface-level redesign if error rate > 1% sustained. |
| Plugin downloads | PyPI / registry | Popular community patterns → candidates for in-tree. |
| P95 latency regressions | Per-endpoint SLO | Engineering capacity allocated by impact. |
| Support-ticket categories | CS team | "Top 5 ticket causes" reviewed quarterly; top cause becomes an engineering initiative. |

Explicit: **customer anecdotes alone do not drive the roadmap**. We need a signal corroborated across multiple inputs. Shiny-object syndrome is the default failure mode of AI platforms; this is our guardrail.

---

## 10. Communication channels

- **Changelog** (`/v1/changelog` endpoint + public docs): every merge.
- **Preview program**: opt-in tenants get access to preview features + direct feedback channel to engineering.
- **Release notes**: per minor version, human-written, published with the release tag.
- **Quarterly roadmap update**: blog post, public. What shipped, what's next, what changed direction and why.
- **Security advisories**: dedicated channel, CVE-numbered where applicable.

---

## 11. Versioning the documentation

These 15 docs are themselves versioned. Each doc's header carries `Runtime version: <x.y>` (or "pre-1.0"). When a runtime release lands, the docs are re-published under a version tag; old docs remain accessible at `/docs/v1.0/`, `/docs/v1.1/`, etc.

Breaking changes to the public surface get a migration guide in the docs with **code** (before/after), not just prose.

---

## 12. Success criteria for v1

v1 is shipped when, measured against 10 design-partner tenants each running ≥ 100 k runs over 30 days:

1. **Availability** ≥ 99.9% on API, ≥ 99.5% on run-completion success (excluding policy-blocked / budget-exhausted, which are correct outcomes).
2. **P95 first-token latency** ≤ 1500 ms on warm prompts.
3. **Determinism**: eval replays produce identical outputs in > 98% of cases (remaining 2% tolerated for provider non-determinism).
4. **Cost attribution accuracy** within 2% of provider invoice (reconciled weekly).
5. **Escape hatches exercised**: every extension point used by at least one design partner in production. (If no one needs E7 `Router`, for example, the abstraction is probably over-designed; we revisit.)
6. **Zero P0 security incidents** attributed to platform (not tenant misconfiguration).
7. **Policy engine catches ≥ 95%** of red-team attacks in the quarterly benchmark.
8. **Operational oncall load ≤ 2 pages/week** across the fleet.

If we hit 5 of 8, v1 ships. If we hit 3 of 8, we don't, and we fix the gaps before calling it.

---

## 13. Post-v1 — the forever list

Things the runtime will be investing in indefinitely, not as features but as ongoing work:

- **Provider adapters.** New providers launch; existing ones change. Adapter maintenance is a permanent team concern.
- **Grader evolution.** Judges drift; benchmarks saturate; new eval methodologies emerge. Eval is never "done."
- **Security posture.** Red-team rotations, threat model updates, dependency CVE triage. Forever.
- **Cost efficiency.** Prompt caching improvements, routing strategy evolution, batch API adoption, self-hosted inference options for stable workloads.
- **Observability depth.** New dashboards, new debugger affordances, new replay modes. The debugging experience compounds with usage data.
- **Developer experience.** SDK ergonomics, documentation quality, onboarding time-to-first-run. Measured, invested in.

None of these are roadmap items. They're background work. The roadmap is for things that end.

---

## 14. Closing statement

This document closes the 15-doc architecture set. The runtime described across these documents is opinionated on a small number of things (**determinism, explicit contracts, state as first-class, observability not optional, budgets as inputs, provider independence**) and deliberately flexible on everything else (planners, memory backends, transports, routers, graders, policies, guards).

The roadmap's north star: **the runtime should get simpler, not more clever, over time**. New primitives require strong evidence. Removed primitives require stronger evidence. Extensions are cheap because extension points are well-defined. v1 is the base camp, not the summit.

What ships in v1 works for the problems that exist today. What's designed into v1 makes sure we don't corner ourselves on the problems that exist tomorrow. That's the whole bet.

---

**End of architecture set.**

Reading order:
- [00 — README / Principles](./README.md)
- [01 — High-Level Architecture](./01-high-level-architecture.md)
- [02 — Data Model and Contracts](./02-data-model-and-contracts.md)
- [03 — Planning and Execution](./03-planning-and-execution.md)
- [04 — Memory Systems](./04-memory-systems.md)
- [05 — Tools and Function Calling](./05-tools-and-function-calling.md)
- [06 — Multi-Agent Coordination](./06-multi-agent-coordination.md)
- [07 — LLM Provider Layer](./07-llm-provider-layer.md)
- [08 — Runtime Infrastructure](./08-runtime-infrastructure.md)
- [09 — Observability and Tracing](./09-observability-and-tracing.md)
- [10 — Safety and Governance](./10-safety-and-governance.md)
- [11 — Evaluation and Testing](./11-evaluation-and-testing.md)
- [12 — API Surface](./12-api-surface.md)
- [13 — Deployment and Scaling](./13-deployment-and-scaling.md)
- [14 — Roadmap and Extension Points](./14-roadmap-and-extension-points.md)  ← you are here
