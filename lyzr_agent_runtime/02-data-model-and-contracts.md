# 02 — Data Model and Contracts

This document pins down the types everything else exchanges. If two components disagree on any type here, the system is broken. Treat this file as the constitution.

Every type is described as: **fields → invariants → lifecycle → where it lives**. Signatures use Pydantic-flavored Python; storage is agnostic (Postgres JSONB, protobuf, etc.).

---

## Guiding rules

1. **IDs are opaque ULIDs.** ULID sorts lexically by time and is safe in URLs. Never expose auto-increment integers externally.
2. **Every record has `created_at`, `updated_at`, and a `tenant_id`.** Multi-tenancy is not bolt-on.
3. **Timestamps are UTC, ISO-8601, millisecond precision.** No local times anywhere.
4. **Enums are closed sets, encoded as strings.** No integer enums leaking to clients.
5. **Fields are snake_case.** SDKs may translate.
6. **Every mutable field belongs to exactly one writer.** No "sometimes the API writes it, sometimes the worker does" — that is how we ship corruption.

---

## Core ID types

```
AgentId       = "agt_" + ULID
RunId         = "run_" + ULID
StepId        = "stp_" + ULID
MessageId     = "msg_" + ULID
ToolCallId    = "tlc_" + ULID
MemoryId      = "mem_" + ULID
TraceId       = "trc_" + ULID
SpanId        = "spn_" + ULID
TenantId      = "ten_" + ULID
PrincipalId   = "prn_" + ULID   # user, service, or API key
```

Prefixes are not decorative — they are validated everywhere. An API that accepts `AgentId` rejects any ID whose prefix is not `agt_`.

---

## `Agent` — the static configuration

An Agent is a **template**. It holds zero state about any particular conversation. State lives in `Run`.

```python
class Agent(BaseModel):
    id: AgentId
    tenant_id: TenantId
    name: str                          # human-readable, not unique
    version: int                       # monotonic per (tenant, name)
    persona: str                       # system prompt body
    planner: PlannerConfig
    tools: list[ToolRef]               # ordered — affects prompt
    memory: MemoryConfig
    policy: PolicyConfig
    budget: BudgetConfig
    output_schema_ref: SchemaRef | None
    metadata: dict[str, str]           # arbitrary labels
    created_at: datetime
    updated_at: datetime
    created_by: PrincipalId
```

**Invariants:**
- `version` monotonically increases within (`tenant_id`, `name`). Published versions are immutable; editing creates a new version.
- `tools` is a list of references, not inlined definitions. Tool source of truth is the Tool Registry.
- `budget` must be finite on every field. There is no "unlimited."

**Storage:** `agents` table in Postgres. Full row is JSON-serializable to disk and to network.

---

## `Run` — a single invocation

```python
class Run(BaseModel):
    id: RunId
    tenant_id: TenantId
    agent_id: AgentId
    agent_version: int                 # pinned at run start
    status: RunStatus
    input: RunInput
    output: RunOutput | None
    error: RunError | None
    budget_used: BudgetUsage
    budget_remaining: BudgetRemaining
    parent_run_id: RunId | None        # set if spawned by another agent
    root_run_id: RunId                 # top of the tree; equals id if root
    initiator: PrincipalId
    started_at: datetime | None
    ended_at: datetime | None
    created_at: datetime
    updated_at: datetime
    checkpoint_version: int            # increments every step
    trace_id: TraceId
```

`RunStatus` is a state machine — transitions are strict:

```
          ┌────────┐
          │created │
          └───┬────┘
              │
              ▼
          ┌────────┐
          │queued  │
          └───┬────┘
              │  worker picks up
              ▼
          ┌────────┐ ── paused ──┐
          │running │             │
          └───┬────┘ ◀── resumed ┘
              │
    ┌─────────┼──────────────┐
    ▼         ▼              ▼
┌────────┐┌─────────┐ ┌───────────┐
│success ││ failed  │ │ cancelled │
└────────┘└─────────┘ └───────────┘
```

**Invariants:**
- Status may only transition along an arrow above.
- `checkpoint_version` increases by exactly 1 per step.
- `budget_used + budget_remaining = agent.budget` at all times.
- If `status == success`, `output` is set and `error` is null; if `failed`, vice versa.
- A run whose `agent_version` points at a deleted (hard-deleted) agent config is **not** allowed — agents are soft-deleted to preserve replayability.

**Storage:** `runs` table + JSONB for nested fields. Partitioned by `created_at` month for hot/cold separation.

---

## `Step` — one iteration of the loop

```python
class Step(BaseModel):
    id: StepId
    run_id: RunId
    ordinal: int                       # 1, 2, 3, ... within run
    phase: StepPhase                   # plan | act | observe | reflect | terminate
    started_at: datetime
    ended_at: datetime | None
    planner_input: PlannerInput
    planner_output: PlannerOutput
    action: Action | None              # what the planner decided
    action_result: ActionResult | None # what happened when we did it
    memory_writes: list[MemoryWrite]   # writes made during this step
    tokens_used: int
    dollars_used: Decimal
    duration_ms: int
    trace_span_id: SpanId
```

**Invariants:**
- `ordinal` is 1-indexed, dense, and unique within `run_id`.
- A step is immutable once `ended_at` is set. Corrections are new steps, not edits.
- `planner_output` is stored raw (the LLM's literal response) **and** parsed into `action`. If parsing fails, the step is in phase `terminate` with failure reason.

**Storage:** `steps` table, partitioned by `run_id`. The raw LLM output goes to object storage if over a threshold (say 64 KB) with a reference here.

---

## `Message` — an entry in the conversation

Messages are the unit that flows to the LLM. Steps are what the runtime produces; messages are what the LLM sees.

```python
class Message(BaseModel):
    id: MessageId
    run_id: RunId
    role: MessageRole                  # system | user | assistant | tool | developer
    content: list[ContentBlock]
    tool_call_id: ToolCallId | None    # for role=tool
    tool_calls: list[ToolCall] | None  # for role=assistant
    created_at: datetime
    token_count: int

class ContentBlock(BaseModel):
    kind: Literal["text", "image", "file", "json"]
    text: str | None
    image_url: str | None
    file_id: str | None
    json_value: Any | None
```

**Why blocks, not strings:** multimodal inputs, provider-agnostic structured content, future-proofing for audio.

**Invariants:**
- `role=tool` messages have exactly one `tool_call_id` referencing the assistant's call.
- `role=assistant` messages with `tool_calls` have `content=[]` or a single text block preceding the calls.

---

## `Plan` — what the planner produces

`Plan` is an internal artifact; it does not always correspond to a single LLM message. For a simple ReAct planner, a plan is a single next action. For Plan-Execute, it is a full ordered list of steps.

```python
class Plan(BaseModel):
    id: str                            # "pln_" + ULID
    run_id: RunId
    strategy: PlanStrategy             # react | plan_execute | reflexion
    rationale: str                     # the "thought"
    steps: list[PlannedStep]           # 1 for ReAct, N for Plan-Execute
    created_at: datetime
    revised_from: str | None           # prior plan id if this is a revision

class PlannedStep(BaseModel):
    intent: str                        # short natural language
    action: Action
    depends_on: list[int]              # indices into plan.steps
```

**Invariants:**
- `depends_on` references are acyclic and point only at earlier indices.
- A plan with `strategy=react` has exactly one step.
- Plans are append-only; revision creates a new Plan linked via `revised_from`.

---

## `Action` — the unit of doing

```python
class Action(BaseModel):
    kind: ActionKind   # tool_call | memory_write | memory_query | handoff | finish
    tool_call: ToolCall | None
    memory_op: MemoryOp | None
    handoff: Handoff | None
    finish: Finish | None

class ToolCall(BaseModel):
    id: ToolCallId
    tool_id: str
    args: dict                          # validated against tool input schema
    parallelizable: bool                # hint from planner

class Handoff(BaseModel):
    target_agent_id: AgentId
    input: RunInput
    budget_fraction: float              # 0 < x <= 1 of remaining budget

class Finish(BaseModel):
    final_output: Any                   # validated against agent.output_schema_ref
```

**Exactly one of the sub-fields is non-null.** Schema validation on write.

---

## `ActionResult` — what happened

```python
class ActionResult(BaseModel):
    ok: bool
    value: Any | None                   # tool return, memory payload, sub-agent output
    error: ActionError | None
    duration_ms: int
    retries: int
    side_effect_ids: list[str]          # e.g. memory record ids created
```

`ActionError` is structured:

```python
class ActionError(BaseModel):
    code: ErrorCode   # enum: tool_timeout | tool_rejected | policy_violation | ...
    message: str
    retryable: bool
    details: dict | None
```

Errors are not strings. Strings are for humans; code flow decisions only use `code` and `retryable`.

---

## `MemoryRecord` — the durable memory atom

```python
class MemoryRecord(BaseModel):
    id: MemoryId
    tenant_id: TenantId
    agent_id: AgentId | None           # null = global to tenant
    subject_id: str | None             # e.g., user id this memory is about
    kind: MemoryKind                   # episodic | semantic | procedural | preference
    namespace: str                     # partitioning within agent
    content: str                       # canonical text form
    embedding: list[float] | None      # for semantic retrieval
    source_run_id: RunId | None
    source_step_id: StepId | None
    importance: float                  # 0..1, affects retention
    access_count: int
    last_accessed_at: datetime | None
    expires_at: datetime | None
    metadata: dict[str, Any]
    created_at: datetime
```

Full memory taxonomy in [04](./04-memory-systems.md). This is just the record.

**Invariants:**
- `embedding` present iff `kind in {semantic, episodic}` (other kinds can be retrieved lexically).
- `importance` defaults to 0.5; memory GC uses it for forgetting.

---

## `Tool` — the registry record

```python
class Tool(BaseModel):
    id: str                            # e.g. "web_search@1.2.0"
    tenant_id: TenantId | None         # null = platform-wide
    name: str
    version: str
    description: str                   # LLM-facing — must be excellent
    input_schema: JsonSchema           # JSON Schema draft 2020-12
    output_schema: JsonSchema
    side_effects: SideEffectClass      # none | read | write | external
    timeout_seconds: int
    max_retries: int
    rate_limit: RateLimit | None
    cost_hint: CostHint | None
    transport: ToolTransport           # inproc | http | mcp | queue
    transport_config: dict
    auth_ref: str | None               # secret reference, not the secret
    created_at: datetime
    deprecated_at: datetime | None
```

Why `description` matters: it is the only thing the LLM uses to decide to call the tool. A bad description = a skipped or misused tool. Full detail in [05](./05-tools-and-function-calling.md).

---

## `Event` — the universal audit unit

Everything interesting emits an Event. Traces are sequences of Events. SSE streams are serialized Events.

```python
class Event(BaseModel):
    id: str                            # "evt_" + ULID
    trace_id: TraceId
    span_id: SpanId
    parent_span_id: SpanId | None
    run_id: RunId | None               # null for system-level
    step_id: StepId | None
    type: EventType                    # see catalog below
    payload: dict                      # schema depends on type
    at: datetime
    tenant_id: TenantId
```

### Event type catalog (closed set — adding requires a schema migration)

```
run.created          run.started         run.completed      run.failed
run.cancelled        run.paused          run.resumed        run.budget_exceeded

step.started         step.completed      step.failed

planner.started      planner.completed   planner.parse_failed

tool.selected        tool.validated      tool.invoked       tool.retried
tool.completed       tool.failed         tool.rejected

memory.queried       memory.retrieved    memory.wrote       memory.evicted

handoff.requested    handoff.accepted    handoff.returned

llm.request          llm.stream          llm.response       llm.error

policy.triggered     policy.rewrote      policy.halted

reflection.started   reflection.completed
```

Every event type has a **payload schema**. Changing a payload schema requires a new event version, not an in-place edit — past runs' event streams must remain parseable forever.

---

## `AgentState` — the checkpointable bundle

Between steps, the worker persists this exact blob. Restoring it onto a different worker reproduces the agent's state.

```python
class AgentState(BaseModel):
    run_id: RunId
    checkpoint_version: int
    messages: list[Message]            # windowed — full history in DB
    working_memory: dict[str, Any]     # transient scratchpad
    short_term_summary: str | None     # running compressed history
    active_plan: Plan | None
    plan_cursor: int                   # which step of the plan we're on
    pending_tool_calls: list[ToolCall]
    budget_used: BudgetUsage
    last_step_id: StepId | None
    subagent_runs: dict[str, RunId]    # logical name -> child run
    custom: dict[str, Any]             # planner-specific, versioned
```

**Invariants:**
- `checkpoint_version` in `AgentState` matches the same field on `Run` after persistence.
- Restoring a checkpoint + the agent config must deterministically reconstruct the in-memory state needed to execute the *next* step.
- Nothing in `AgentState` points to process-local resources — no file handles, no open sockets.

---

## `Budget` — the accounting unit

```python
class BudgetConfig(BaseModel):
    max_tokens: int
    max_dollars: Decimal
    max_wall_seconds: int
    max_tool_calls: int
    max_steps: int
    max_subagent_depth: int            # recursion guard
    max_memory_writes: int

class BudgetUsage(BaseModel):
    tokens: int
    dollars: Decimal
    wall_seconds: int
    tool_calls: int
    steps: int
    subagent_depth_max: int
    memory_writes: int

class BudgetRemaining(BudgetUsage): ...  # same shape, computed
```

The runtime checks **all** budget dimensions before every step. First violation wins; run ends with `budget_exceeded`.

---

## `Trace` — the replayable log

A Trace is not a separate record; it is the ordered set of Events with `trace_id=X`. You query it, you don't "create" it. One Run has one Trace. One sub-agent Run has its own Trace, linked by `parent_run_id`.

**Retention policy:**
- Hot (Postgres): 14 days, full fidelity.
- Warm (object store + index): 90 days.
- Cold (object store, compressed): per tenant plan.

Replay reads events in `at` order (ties broken by ULID), reconstructs state, and produces a step-by-step rerun view.

---

## Schema versioning

Every model above has a `schema_version` field in persistence (not shown in the Python classes for brevity). When a field is added:

- **Backwards-compatible add:** new field has a default; bump minor version.
- **Breaking change:** write a one-time migration that rewrites existing rows; bump major version; old readers refuse to parse new rows until updated.

Events and messages are frozen once written. No exceptions. Replayability dies the moment we rewrite history.

---

## What the next doc does with all this

[03 — Planning and Execution](./03-planning-and-execution.md) takes these types and describes **the loop** — how Planner consumes Messages + Memory to produce a Plan, how Executor turns an Action into an ActionResult, and how Steps thread through it all.
