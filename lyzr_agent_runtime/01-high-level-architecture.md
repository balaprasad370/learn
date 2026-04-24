# 01 — High-Level Architecture

This document is the map. Every component named here is expanded in a later doc. Read this to understand how the pieces fit; read the others to understand what they are.

---

## Layered view

```
┌──────────────────────────────────────────────────────────────────┐
│                       API Layer (12)                             │
│  REST • SSE • WebSocket • SDKs (Python, TS) • CLI                │
└──────────────────────────────────────────────────────────────────┘
          │ start / observe / pause / resume / cancel
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                   Orchestration Layer (08)                       │
│  Run Manager • Queue • Scheduler • Checkpointer • Event Bus       │
└──────────────────────────────────────────────────────────────────┘
          │ dispatch run → worker
          ▼
┌──────────────────────────────────────────────────────────────────┐
│                        Agent Layer                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  Planner    │→ │  Executor   │→ │  Reflector  │  (03)         │
│  │ (03)        │  │  (03)       │  │  (03, opt)  │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│         ▲                  │                                      │
│         │                  ▼                                      │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Memory (04)   │  │ Tools (05)   │  │ Sub-Agents   │           │
│  │ STM/LTM/Epi.  │  │ Registry+    │  │ (06)         │           │
│  └───────────────┘  │ Invoker       │  └──────────────┘           │
│                     └──────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
          │            │               │
          ▼            ▼               ▼
┌─────────────────┐ ┌──────────┐ ┌────────────────────────────────┐
│ LLM Providers   │ │  Policy  │ │  Observability (09)            │
│ (07) OpenAI,    │ │ (10)     │ │  Tracer • Metrics • Cost • Eval│
│ Anthropic, ...  │ │ Guards   │ │                                │
└─────────────────┘ └──────────┘ └────────────────────────────────┘
          │            │               │
          ▼            ▼               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Persistence Layer (08)                        │
│  Postgres (state, runs, messages) •  Redis (queue, cache,        │
│  pub/sub) • Vector DB (long-term memory) • Object store (blobs)  │
└──────────────────────────────────────────────────────────────────┘
```

Each parenthetical number points to the document that expands that component.

---

## Component inventory

Every component below exists. If a responsibility does not map to one of these, either the responsibility is wrong or the list is incomplete.

### API Layer
- **HTTP server** — REST endpoints for `POST /runs`, `GET /runs/{id}`, `POST /runs/{id}:cancel`, etc.
- **SSE endpoint** — streams trace events as they happen.
- **WebSocket endpoint** — bidirectional for human-in-the-loop: client can inject messages mid-run.
- **SDKs** — thin clients that wrap the HTTP + SSE protocol.
- **Auth & multi-tenancy** — every request carries a tenant + principal; isolation enforced downstream.

### Orchestration Layer
- **Run Manager** — owns the lifecycle of a run (created → queued → running → paused → completed/failed/cancelled).
- **Queue** — durable queue (Redis Streams, SQS, or Postgres-as-queue) that holds pending/resumable runs.
- **Scheduler** — decides which worker picks up which run. Enforces per-tenant concurrency limits.
- **Checkpointer** — after every step, persists the full agent state so a crashed worker's run can be resumed by another.
- **Event Bus** — pub/sub for trace events. API layer subscribes for SSE; observability writes to sinks.

### Agent Layer
- **Planner** — generates the next intended action given state. Typically an LLM call with a tool-choice constraint or a structured-output constraint. (See [03](./03-planning-and-execution.md).)
- **Executor** — validates and performs the planner's chosen action. Dispatches to tool invocation, memory I/O, or sub-agent invocation.
- **Reflector** *(optional)* — after N steps or before termination, asks the LLM to critique progress and optionally revise the plan. (Reflexion-style.)
- **Memory Manager** — routes reads/writes across short-term, working, episodic, semantic, procedural memory stores. (See [04](./04-memory-systems.md).)
- **Tool Registry + Invoker** — looks up the tool, validates args, runs it (with retries and timeouts), validates result. (See [05](./05-tools-and-function-calling.md).)
- **Sub-Agent Invoker** — treats another agent as a tool call, spawning a child run with its own budget. (See [06](./06-multi-agent-coordination.md).)
- **Policy Engine** — applies guardrails on every input, output, and tool call. Can halt or rewrite. (See [10](./10-safety-and-governance.md).)

### LLM Provider Layer
- **Provider interface** — minimal contract: `chat()`, `stream_chat()`, `embed()`, `count_tokens()`.
- **Adapters** — one per provider (Anthropic, OpenAI, Gemini, Bedrock, local). Translate to/from internal format.
- **Router** — picks the provider+model for a call based on the agent config, fallback rules, and provider health. (See [07](./07-llm-provider-layer.md).)
- **Cache** — deterministic prompt → response cache for eval and cost control.

### Persistence Layer
- **Primary OLTP** (Postgres) — runs, steps, messages, agent configs, tool definitions, tenants.
- **Cache + queue** (Redis) — short-term memory, session buffers, pub/sub, work queue.
- **Vector DB** (pgvector / Pinecone / Qdrant / Weaviate) — long-term semantic memory.
- **Object store** (S3 / GCS) — large blobs: tool inputs/outputs over N KB, file attachments, full traces.

### Observability
- **Tracer** — every event emits a span. OpenTelemetry-compatible format. Spans nest (planner span > tool-call span > LLM-call span).
- **Metrics** — counters (runs by status), histograms (step duration, token count), gauges (queue depth).
- **Cost accounting** — per-run ledger of LLM tokens, tool invocations, vector queries. Rolls up to tenant for billing / caps.
- **Eval hooks** — every run can be tagged for offline eval. (See [11](./11-evaluation-and-testing.md).)

### Safety / Governance
- **Input guards** — PII redaction, prompt-injection detection, content filters.
- **Output guards** — same, plus structure validation and toxicity/compliance checks.
- **Tool policy** — allowlist/denylist per agent, per tenant; rate limits per tool.
- **Budget enforcer** — hard stop on tokens, dollars, wall time, tool calls. (See [10](./10-safety-and-governance.md).)

---

## Request lifecycle — a full round trip

**Scenario:** a user POSTs a question; a single-agent run completes in three steps.

```
1. CLIENT → API        POST /v1/runs
                       { agent_id, input, budget, stream=true }
                       Returns run_id, then opens SSE.

2. API → Orchestration Creates Run row (status=queued). Emits run.created.
                       Enqueues {run_id} to work queue.

3. WORKER              Dequeues run_id. Loads agent config + persistent state
                       (empty on first run). Marks status=running.
   (loop starts)

4. STEP 1 — PLAN       Planner builds a prompt from: system + persona,
                       relevant memory (STM + retrieved LTM), recent messages,
                       available tool schemas. Calls LLM.
                       LLM returns: { thought, tool_call: search("...") }.
                       Emits planner.completed, tool.selected events.

5. STEP 1 — POLICY     Policy engine inspects tool_call. Allowed.

6. STEP 1 — EXECUTE    Tool Invoker validates args, runs search("..."),
                       times out at 30s, retries once on transient error.
                       Result validated against tool output schema.
                       Emits tool.invoked, tool.completed events.

7. STEP 1 — MEMORY     Result and thought appended to short-term memory.
                       If result is "notable," embedded and stored in LTM.

8. STEP 1 — CHECKPOINT Worker persists updated state + step record to Postgres
                       atomically. Emits step.completed. Budget deducted.

9. STEP 2              Same shape. Planner sees step 1's observation in prompt.
                       Produces tool_call: fetch_url("..."). Run continues.

10. STEP 3             Planner produces {thought, final_answer: "..."}.
                       Policy inspects output. Output passes.
                       Emits run.completed. Worker marks status=completed.

11. CLIENT             Received streaming events throughout steps 4-10 via SSE.
                       Receives final_answer. Closes connection.
```

If the worker crashes between steps 2 and 3, the Run row is still `running` but has no recent heartbeat. The scheduler re-enqueues it; another worker loads the checkpoint from step 2 and resumes at step 3 with no state loss. **This is the point of checkpointing after every step.**

---

## Boundaries — what crosses what

Clear boundaries are how this system stays maintainable. Violating them is the fastest way to make the runtime unmaintainable.

| Boundary | Crosses over | Does NOT cross over |
|----------|--------------|--------------------|
| API ↔ Orchestration | Run commands (create, cancel, pause, resume), run IDs, event subscriptions | Raw LLM calls, raw tool results, memory internals |
| Orchestration ↔ Agent | A single run's state, events, step-level control | Persistence driver details, queue implementation |
| Agent ↔ Provider | Normalized ChatRequest + ChatResponse + ToolCall | Provider-specific formatting, retries (those live in provider adapter) |
| Agent ↔ Memory | Typed memory operations (read_working_set, write_episode, query_semantic) | Vector DB query syntax, SQL |
| Agent ↔ Tools | ToolInvocation{tool_id, args} → ToolResult{ok, value, error} | Tool implementation code — tools are black boxes to the agent |
| Everything ↔ Observability | Structured events with schema; fire-and-forget | Business logic — observability never shapes control flow |

Rule of thumb: if a component must import from a component two layers below it, that is a design smell. Refactor through an interface at the layer boundary.

---

## Concurrency model (summary — full detail in [08](./08-runtime-infrastructure.md))

- **Within a run:** steps are sequential; tools within a step **may** be called in parallel if the planner returns multiple `tool_call`s.
- **Across runs on one worker:** `asyncio` event loop handles I/O concurrency; CPU-bound work offloaded to a thread pool.
- **Across workers:** shared-nothing. The persistence layer is the only coordination point. Any run can migrate to any worker between steps.

---

## What the user actually configures

An agent is a YAML (or JSON) config that binds:

```yaml
agent_id: research-agent-v3
display_name: "Research Agent"
persona: |
  You are a careful researcher. Cite sources. Prefer primary over secondary.
planner:
  kind: react            # react | plan-execute | reflexion
  model: claude-opus-4-7 # resolved via provider router
  temperature: 0.2
  max_steps: 12
tools:
  - web_search
  - fetch_url
  - summarize_document
memory:
  short_term:
    kind: buffer_with_summary
    buffer_tokens: 4000
  long_term:
    kind: vector
    store: pgvector
    namespace: research
    retrieval:
      top_k: 5
      min_score: 0.72
policy:
  input_guards: [pii_redact, injection_detect]
  output_guards: [structure_check]
  tool_allowlist: [web_search, fetch_url, summarize_document]
budget:
  max_tokens: 200000
  max_dollars: 2.00
  max_wall_seconds: 300
  max_tool_calls: 30
output:
  schema_ref: research_result_v1   # optional structured output contract
  stream: true
```

Every field maps to a component in this doc. If a config field exists with no matching component, that is a bug in the config schema.

---

## What's in the next doc

[02 — Data Model and Contracts](./02-data-model-and-contracts.md) pins down the *types* everything above exchanges: `Run`, `Step`, `Message`, `ToolCall`, `Plan`, `MemoryRecord`, `Event`, and the invariants each one must satisfy.
