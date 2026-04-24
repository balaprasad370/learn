# 03 — Planning and Execution

This is the beating heart of the runtime. Everything else exists to serve this loop.

Reading prerequisite: [02 — Data Model](./02-data-model-and-contracts.md). Types like `Plan`, `Action`, `Step`, `AgentState` are assumed.

---

## The universal loop

Regardless of planning strategy, every agent runs the same outer loop:

```
state ← load_state(run_id)
while not terminated(state):
    pre  ← assemble_prompt_context(state)
    plan ← planner.next(pre)                # LLM call
    policy.check_plan(plan, state)
    for action in plan.actions_to_run(state):
        policy.check_action(action, state)
        result ← executor.run(action, state)
        state  ← state.apply(action, result)
        budget.deduct(result)
        checkpointer.save(state)            # <-- survivability
        if terminated(state): break
    if reflect_now(state):
        plan' ← reflector.critique(state)
        if plan' != plan:
            state ← state.with_plan(plan')
finalize(state)
```

Four phases per iteration: **observe → plan → act → checkpoint**. Optional: **reflect**.

---

## Planning strategies

The runtime supports three built-in planner kinds. Which one to use is a property of the Agent config, not of the runtime. All three share the same input (`AgentState`) and output (`Plan`).

### Strategy A — ReAct (single-step, reactive)

**The simplest planner.** Each iteration, the LLM returns one "thought" plus one action. No look-ahead. Default choice for tool-use-heavy tasks where the path depends on tool results.

```
Prompt layout:
  [system persona]
  [tool schemas in compact form]
  [short-term summary]
  [retrieved long-term memories]
  [recent messages]
  [current user intent]

Response schema:
  {
    "thought": string,
    "action": {
      "kind": "tool_call" | "finish" | "handoff",
      ...
    }
  }
```

**Pros:**
- Simple. Easy to debug. Robust to tool failures (replans every step).
- Token-efficient per step.

**Cons:**
- No global view — can loop on side-tracks. Mitigated with a loop-detector (see below).
- Every step is a fresh LLM call; N steps = N calls.

**When to use:** open-ended research, retrieval-heavy, unpredictable tool behavior.

### Strategy B — Plan-Execute (multi-step, upfront plan)

**Two phases per plan:** a **planner LLM call** produces an N-step plan; an **executor** runs steps sequentially (or in parallel by `depends_on`); when done, the runtime either finishes or re-plans.

```
Response schema (planner phase):
  {
    "rationale": string,
    "steps": [
      { "intent": string, "action": Action, "depends_on": [int, ...] },
      ...
    ]
  }
```

**Pros:**
- Parallelism: independent steps fire concurrently.
- Fewer LLM calls for linear work.
- Human-readable plan for audit.

**Cons:**
- Brittle to surprise. If step 2 fails or returns something unexpected, the rest of the plan may be obsolete. Triggers re-plan.
- Harder to debug when plans revise.

**When to use:** well-structured tasks with known tools (e.g., "draft a report with sections X, Y, Z").

### Strategy C — Reflexion (ReAct + critique)

ReAct in the inner loop, plus every K steps or before termination, a **reflector** LLM call reviews the trace and either green-lights or asks for a new plan. The reflector sees the trace-so-far in summarized form.

**Reflector prompt:**
```
System: You are reviewing a reasoning trace. Identify errors, redundant steps,
        or missed information. If the trace is on track, respond "continue".
        Otherwise, respond with a revised plan or a pointed critique.

Input:  [trace summary], [current state], [original goal]

Response: { "verdict": "continue" | "revise", "revision": Plan | null, "note": string }
```

**Pros:**
- Catches loops and dead-ends ReAct misses.
- Improves quality on hard tasks.

**Cons:**
- Adds cost (one extra LLM call per reflection).
- Can get stuck in "critique loops" if reflector never accepts.

**When to use:** high-stakes outputs, long horizons.

### Strategy D — (future) Tree search / LATS

Not in v1. Covered in [14 — Roadmap](./14-roadmap-and-extension-points.md).

---

## The planner contract

A planner implements:

```python
class Planner(Protocol):
    strategy: PlanStrategy

    async def next(self, state: AgentState, ctx: PromptContext) -> Plan: ...
    async def revise(self, state: AgentState, reason: str) -> Plan: ...
    def expected_tokens(self, state: AgentState) -> int: ...   # for budget pre-check
```

The runtime — not the planner — owns the loop. The planner only decides *what* to do next, never drives the loop itself.

---

## Prompt context assembly

Before every planner call, the runtime assembles a `PromptContext` with a strict layering. The order is fixed; individual layers may be empty.

```
1. Identity layer           — persona, operating rules, output format
2. Capability layer         — tool schemas, sub-agent schemas, output schema
3. Long-term memory layer   — retrieved semantic + procedural records
4. Short-term summary       — compressed older messages
5. Recent messages          — last N raw messages within token budget
6. Current turn             — the user input + any pending tool results
7. Planner directives       — strategy-specific instructions + response schema
```

Why a fixed layering:
- Determinism. The same inputs produce the same prompt.
- Caching. Providers that cache by prefix (Anthropic prompt caching, OpenAI system caching) hit more often with stable prefixes.
- Evaluability. Swapping a layer is an A/B dimension; without a fixed order, there is no "layer 3" to swap.

### Prompt compiler

The compiler is a pure function `(PromptContext, ProviderAdapter) -> ProviderRequest`. It does NOT touch state. Tested in isolation with golden snapshots.

```python
def compile_prompt(ctx: PromptContext, adapter: ProviderAdapter) -> ProviderRequest:
    parts = [
        render_identity(ctx.identity),
        render_capabilities(ctx.tools, ctx.subagents, ctx.output_schema),
        render_memories(ctx.retrieved_memories),
        render_short_term_summary(ctx.short_term_summary),
        render_recent_messages(ctx.recent_messages, ctx.token_budget),
        render_current_turn(ctx.current_turn),
        render_directives(ctx.strategy, ctx.response_schema),
    ]
    return adapter.assemble(parts, ctx.model, ctx.sampling)
```

Token budgeting is layer-aware: the compiler fits within the model's context by shrinking layers in order `5 → 3 → 4`, never 1, 2, 6, or 7. (Identity and current turn are sacred.)

---

## The executor contract

```python
class Executor(Protocol):
    async def run(self, action: Action, state: AgentState) -> ActionResult: ...
```

The executor is a dispatcher:

```
action.kind == tool_call     -> ToolInvoker.invoke(action.tool_call)
action.kind == memory_write  -> MemoryManager.write(action.memory_op)
action.kind == memory_query  -> MemoryManager.query(action.memory_op)
action.kind == handoff       -> SubAgentInvoker.spawn_and_wait(action.handoff)
action.kind == finish        -> finalize(action.finish)
```

The executor does not decide *whether* to run the action — that is policy's job. By the time the executor sees an action, it has been pre-validated and pre-authorized.

---

## Parallel tool calls

When the planner returns multiple tool calls in a single plan (common in modern models), the executor fans out:

```python
async def run_parallel(calls: list[ToolCall]) -> list[ActionResult]:
    # Respect per-tool rate limits and per-run concurrency cap.
    sem = asyncio.Semaphore(state.agent.max_parallel_tools)
    async def one(c): 
        async with sem: return await invoke(c)
    return await asyncio.gather(*(one(c) for c in calls), return_exceptions=True)
```

**Rules:**
- Two tool calls with side-effect class `write` that target the same resource run sequentially; the scheduler orders them by plan index.
- A failed parallel call does not cancel siblings. The planner sees all results next step.
- Token cost of failed calls still counts.

---

## Termination — when does the loop stop?

A run terminates when **any** of the following fires. Each maps to a `RunStatus` transition.

| Condition | Status |
|-----------|--------|
| Planner emits `action.kind=finish` | `success` |
| Budget exceeded on any dimension | `failed` (reason: `budget_exceeded`) |
| Max steps reached | `failed` (reason: `max_steps`) |
| Policy halt (guard rejected output) | `failed` (reason: `policy_halt`) |
| Unrecoverable error (e.g., provider outage after retries) | `failed` (reason: error code) |
| User cancellation | `cancelled` |
| Loop detector fires | `failed` (reason: `stuck`) |

### Loop detector

Runs are not allowed to circle forever. After each step, the detector computes a fingerprint over the last K actions and observations (K defaults to 3). If the fingerprint has been seen twice in the same run, the detector signals "stuck." The runtime then:

1. Fires a reflection (if reflection is enabled and not already mid-reflection).
2. If reflection does not change the plan, terminates with `stuck`.

Fingerprint = hash of `(action_kind, tool_id, canonicalized_args, hash_of_observation)`. Exact observation hash is overkill; we use first-N tokens of the rendered text.

---

## Step state machine (detail)

A single step traverses these phases. Errors skip forward to `terminate(step)`.

```
┌────────────┐      ┌────────┐       ┌─────────────┐
│  start     │─────▶│ plan   │──────▶│ policy_plan │───┐
└────────────┘      └────────┘       └─────────────┘   │
                                            │         fails
                                            ▼           │
                                       ┌──────────┐     │
                                       │   act    │◀────┘
                                       └────┬─────┘
                                            │
                                            ▼
                                     ┌─────────────┐
                                     │policy_result│
                                     └──────┬──────┘
                                            │
                                            ▼
                                     ┌───────────────┐
                                     │ memory_write  │
                                     └──────┬────────┘
                                            │
                                            ▼
                                     ┌───────────────┐
                                     │  checkpoint   │
                                     └──────┬────────┘
                                            │
                                            ▼
                                     ┌───────────────┐
                                     │  emit events  │
                                     └──────┬────────┘
                                            ▼
                                     ┌───────────────┐
                                     │  step_end     │
                                     └───────────────┘
```

**Checkpoint is the atomic commit point.** State, step row, events, memory writes, and budget deduction commit in a single Postgres transaction. Anything before that can be re-tried after a crash; anything after is durable.

---

## Retry semantics

Retries live at **three** distinct layers. Do not mix them.

| Layer | What it retries | How it retries |
|-------|------------------|----------------|
| Provider adapter | HTTP errors, 429s, transient 5xx from the LLM | Exponential backoff, capped at 3 attempts |
| Tool invoker | `retryable=true` action errors | Tool's own `max_retries`, usually 1–2 |
| Planner | Parsing failures of LLM output (bad JSON, missing fields) | 1 retry with a "repair prompt" appending the parse error |

**Retries consume budget.** A 3-attempt LLM call counts as 3× the tokens. No free retries.

**Retries do not re-order.** A retry of step N is step N with retry counter; it is not a new step.

---

## Structured output and schema conformance

When the agent has an `output_schema_ref`, the final `Finish` action's `final_output` is validated against the schema. The planner is instructed to emit matching structure. Validation failure:

- First failure: repair prompt with the schema violation message. Retry once.
- Second failure: run status = `failed`, reason = `output_schema_violation`. Do not return partial output to client; return error.

The runtime uses **JSON Schema 2020-12** with optional Pydantic conversion. Agents can also declare output as a reference to a named schema in the registry — this enables versioning output contracts separately from agent versions.

---

## Interruption & resumability

A run can be **paused** mid-step. The checkpoint always reflects a consistent step boundary, so the resumer sees either state-before-step-N or state-after-step-N, never mid-step.

**Cancellation:**
- A cancel request sets a flag in Redis checked between sub-phases.
- If cancellation arrives mid-step, the current step completes (to keep state consistent) and the loop exits at the next check.
- Hard-cancel (force) is available for ops but marks the run with `cancelled_dirty=true`; state may be inconsistent.

**Human-in-the-loop:**
- A tool can return `{status: "needs_input", prompt: "..."}`. The runtime pauses, emits `run.paused` with the prompt, and waits.
- A subsequent `POST /runs/{id}:inject` resumes with the new input as a user message.

---

## Cost & token accounting

Every LLM call returns:
- `prompt_tokens`, `completion_tokens`, `total_tokens`
- `model_id`, `provider_id`, `cost_usd` (derived from provider pricing table)

These are recorded on the `Step` and incrementally subtracted from `budget_remaining`. The budget pre-check before an LLM call estimates tokens from the compiled prompt; if the estimate already exceeds remaining budget, the step fails fast with `budget_exceeded` — no wasted provider call.

---

## Failure modes and their handling

| Failure | Where detected | Recovery |
|---------|----------------|----------|
| LLM returns malformed JSON | Planner output parser | Repair-prompt retry; on second failure, terminate |
| LLM hallucinates a nonexistent tool | Pre-execution validator | Inject correction message; next step |
| Tool times out | Tool invoker | Tool-level retry; then mark action failed; planner decides next step |
| Tool returns invalid schema | Output validator | Inject validation error as tool result; planner sees it |
| Memory query returns empty | Memory manager | Not a failure — planner sees empty retrieval and decides |
| Provider 429 | Provider adapter | Backoff + retry; after N failures, try fallback provider |
| Provider outage | Router | Fallback provider (if configured); else run.failed |
| Worker crash mid-step | Scheduler | Re-enqueue; another worker resumes from last checkpoint |
| Checkpoint write fails | Runtime | Retry transaction; if persistent, run.failed — do NOT continue without checkpoint |

**Golden rule:** we never continue past a checkpoint failure. Lose the ability to replay and you lose the product.

---

## A worked example

Input: "Find the population of Bengaluru and compare to Mumbai."

```
Step 1 — plan
  planner.next → { thought: "need two lookups, parallelize",
                    plan: [ {web_search("Bengaluru population 2025")},
                            {web_search("Mumbai population 2025")} ]
                    strategy: plan_execute }
  policy OK → execute (parallel)

Step 1 — act (parallel)
  tool.invoked  web_search("Bengaluru population 2025") → "13.6M"
  tool.invoked  web_search("Mumbai population 2025")    → "21.3M"

Step 1 — checkpoint
  state.messages += 2 tool results
  budget_used.tool_calls += 2; tokens += provider report

Step 2 — plan
  planner.next → { thought: "have both, compute ratio, cite",
                    plan: [ { finish: { final_output:
                      "Bengaluru ~13.6M, Mumbai ~21.3M, Mumbai is ~1.57x larger." }}]}
  policy OK → execute

Step 2 — act
  finish → run.completed
```

Two planner calls, two tool calls in parallel, total wall time dominated by the slower web search. That is Plan-Execute doing its job.

---

## What is NOT in the planner's job

- Executing tools (that is executor).
- Writing memory (planner emits intent; memory manager writes).
- Policy decisions (planner proposes, policy disposes).
- Retrying (provider adapter handles transport; tool invoker handles action-level).
- Deciding to terminate the run (the runtime decides; planner signals with `finish`).

A planner that tries to do any of the above is over-reaching. Keep it a pure function of state → plan.

---

## Testing the planner in isolation

Because `PromptContext → Plan` is pure (given a fixed LLM), it is testable:

- **Golden tests:** capture `(state, plan)` pairs from real runs; re-run with a frozen model to detect drift.
- **Synthetic states:** construct states that induce specific behaviors (missing memory, rate-limited tool, ambiguous input) and assert plan shape.
- **Replay tests:** a recorded trace's inputs should regenerate bit-for-bit equivalent plans if the LLM is frozen.

Detail on the eval harness in [11](./11-evaluation-and-testing.md).

---

## Next: [04 — Memory Systems](./04-memory-systems.md). The planner depends on memory; memory depends on storage primitives. That comes next.
