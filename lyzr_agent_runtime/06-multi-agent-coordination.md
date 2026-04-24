# 06 — Multi-Agent Coordination

Two or more agents cooperating on a task.

This is where most agent frameworks get hand-wavy. "Just have them talk." No. Multiple agents without a coordination protocol is a recipe for infinite loops, conflicting writes, and nondeterministic failure. This document defines the protocols.

Reading prerequisite: [03 — Planning and Execution](./03-planning-and-execution.md) and [05 — Tools](./05-tools-and-function-calling.md).

---

## When to use multiple agents at all

Before picking a coordination pattern, ask: do you need multiple agents, or do you need one agent with more tools?

**Use multiple agents when:**
- Different stages have *different operating constraints*: persona, toolset, memory namespace, safety policy, or model. E.g., a "researcher" that reads widely vs. a "writer" that cites and structures.
- One stage's output is the input to another with a sharp handoff and clear contract.
- The sub-task should run with its own budget and be killable without killing the parent.
- Different model sizes fit different stages (small model for triage, big model for synthesis).

**Do NOT use multiple agents when:**
- You just want "sections" in an output. Use a structured-output schema.
- You want the LLM to think step-by-step. That is a planner strategy, not a second agent.
- You want tool variety. Add tools.

Rule of thumb: each agent should be explainable in one sentence. If you cannot, you are painting over a modeling mistake with extra agents.

---

## Coordination patterns — the catalog

Five patterns. v1 ships with (1) and (2); v2 adds (3); (4) and (5) are advanced and require more care.

### Pattern 1 — Supervisor / worker (hierarchical)

A parent agent decides *which* child agent to invoke and *what* to pass to it. Children execute, return, and the parent continues.

```
User → Supervisor agent
         │
         ├─→ Researcher ─→ sources, findings
         ├─→ Writer     ─→ draft
         └─→ Reviewer   ─→ approved or revision notes
       (back to supervisor after each)
```

- **Control flow:** strictly hierarchical. Children cannot spawn siblings directly.
- **Budget:** child agents are spawned with a `budget_fraction` of the parent's *remaining* budget.
- **Result contract:** each child returns a structured output validated against a schema the parent declared.

This is the safest pattern and the one you should default to.

### Pattern 2 — Sequential pipeline

A fixed, ordered chain: A → B → C. Each stage transforms the work; the next stage receives the prior output verbatim.

```
Input → Extractor → Classifier → Summarizer → Output
```

- No LLM decides the next stage — the sequence is static.
- Implemented as a Plan-Execute plan with handoffs as the actions.
- Useful when you want the benefit of agent separation without the uncertainty of a supervisor's routing.

### Pattern 3 — Blackboard (shared workspace)

Multiple agents read/write a shared state ("blackboard"). A coordinator decides whose turn it is.

```
                ┌────────────┐
          ┌────▶│ blackboard │◀────┐
          │     └────────────┘     │
      Agent A                  Agent B
       (reads, writes)         (reads, writes)
          ▲                       ▲
          └────── Coordinator ────┘
              (turn selection)
```

- Good when agents have overlapping competence and the ordering is dynamic.
- Hard to reason about and to debug. Coordinator MUST enforce a "turn token" — exactly one agent is writing at a time.

### Pattern 4 — Group chat (conversation)

Agents exchange messages in a multi-agent conversation; a moderator enforces turn-taking, topic, and termination.

```
      ┌───────────────┐
      │   Moderator   │  (selects next speaker, ends chat)
      └──────┬────────┘
             │
   [Agent1] [Agent2] [Agent3]
         message stream
```

- Powerful for brainstorming / debate patterns.
- Notoriously hard to terminate — add strict budgets and a "max turns" cap.
- Reserve for v2+, only after (1) and (2) are production-stable.

### Pattern 5 — Peer swarm

Agents spawn peers, consume partial results, and collectively converge. Best-effort scatter/gather.

- Useful for highly parallel evaluations (e.g., "score this from 10 critics").
- Implement as a supervisor with a fan-out over children of the same type, then a reducer.

---

## The handoff primitive

All coordination patterns ultimately sit on one primitive: the **handoff**.

A handoff is the structured, auditable transfer of (a) control and (b) context from one agent to another. It produces a child `Run` with its own trace.

```python
class Handoff(BaseModel):
    target_agent_id: AgentId           # which child to invoke
    input: RunInput                    # what to send it
    budget_fraction: float             # (0, 1], of parent's remaining budget
    memory_bridge: MemoryBridgeSpec    # what memory the child inherits
    context_policy: ContextPolicy      # what from parent's context is visible
    return_shape: JsonSchema           # what the parent expects back
    label: str                         # for logs, e.g. "research_step"
```

### `memory_bridge`

```python
class MemoryBridgeSpec(BaseModel):
    share_namespaces: list[str]        # child reads these parent namespaces
    copy_working_set: list[str]        # keys copied from parent's working set
    inherit_short_term: bool           # does child see parent's conversation?
    write_back_policy: WriteBackPolicy # how child's memory writes surface to parent
```

Explicit memory sharing is the key to safe multi-agent. Defaults:
- `share_namespaces = []` (child starts blind to parent's long-term memory unless listed)
- `copy_working_set = []`
- `inherit_short_term = false` (child does NOT see the parent's conversation by default)
- `write_back_policy = "isolated"` (child's writes land in the child's namespace, not parent's)

A parent that wants its child to see its thread sets `inherit_short_term: true`. This is explicit-over-implicit at work: the parent declares what the child can see.

### `context_policy`

Fine-grained controls over what crosses the handoff boundary:

```python
class ContextPolicy(BaseModel):
    include_persona: bool              # parent's persona as the child's system?
    redact_pii_in_input: bool
    allow_tool_allowlist_override: bool # child can add tools beyond its default?
    propagate_tracing: bool            # child's trace links under parent's
```

---

## The handoff lifecycle

```
1. PLAN
   Parent planner emits Action{kind=handoff, handoff=Handoff(...)}.

2. POLICY
   Policy engine validates: target agent exists, parent has permission to invoke it,
   budget_fraction is sane, depth < max_subagent_depth.

3. SPAWN
   Orchestrator creates child Run with:
     - parent_run_id = parent.id
     - root_run_id   = parent.root_run_id  (or parent.id if parent is root)
     - budget        = parent.budget_remaining * budget_fraction
     - input         = handoff.input
     - memory ctx    = resolved from memory_bridge
   Enqueues the child run.

4. PARENT WAITS
   Parent run status = "paused_awaiting_child".
   Worker may be released to pick up other runs (see async handoff below).

5. CHILD EXECUTES
   Child runs its own loop. Its trace nests under parent's via propagated trace_id.

6. CHILD TERMINATES
   On success: child.output validated against handoff.return_shape.
   Result written to parent's inbox (a durable queue of child results).

7. PARENT RESUMES
   Worker picks up parent; the child's output is injected as an ActionResult.
   Parent's working set updated per write_back_policy.
   Parent's next step sees the child's result as context.
```

### Synchronous vs. async handoff

- **Synchronous (default):** parent's worker blocks until the child finishes. Simpler. Wastes worker slots on long children.
- **Async:** parent checkpoints and releases its worker. Orchestrator re-enqueues parent when child's result lands. Higher throughput; strictly required for high concurrency.

For v1, sync handoffs. Move to async when wait times > a few seconds start blocking workers.

---

## Depth, recursion, cycles

A child can itself be a supervisor. Recursion is allowed up to `budget.max_subagent_depth` (default: 5).

**Cycle protection:** an agent cannot spawn itself (direct or transitive) with the same input hash in the same run tree. Detected at spawn:

```
ancestor_fingerprints = { (run.agent_id, hash(run.input)) for run in ancestors }
if (target_agent_id, hash(handoff.input)) in ancestor_fingerprints:
    reject: cycle_detected
```

---

## Budget arithmetic

When a parent spawns a child with `budget_fraction = 0.4`, the parent transfers 40% of each budget dimension:

```
child.budget = parent.budget_remaining * 0.4
parent.budget_remaining *= 0.6
```

On child completion:
```
parent.budget_remaining += child.budget_remaining  # unused budget returns
```

If the child is cancelled or fails, unused budget still returns. If the child exceeded its budget, the parent is not charged for the overage (the platform absorbs it), but a `budget_leak` event fires for observability — something is wrong with estimation.

---

## Blackboard semantics

A blackboard is a typed, shared key-value space scoped to a run tree.

```python
class Blackboard(Protocol):
    async def read(self, key: str) -> Any: ...
    async def write(self, key: str, value: Any, expected_version: int | None) -> int: ...
    async def watch(self, key: str) -> AsyncIterator[BlackboardEvent]: ...
    async def cas(self, key: str, old: Any, new: Any) -> bool: ...
```

### Access rules
- Scoped to `root_run_id`. A run tree has exactly one blackboard.
- Keys are typed at schema registration time. Writing the wrong shape is a policy violation.
- Writes carry an `expected_version` for optimistic concurrency; stale writes fail and the writer sees latest value.
- Reads are consistent within a single step (snapshot isolation).

### Coordinator role
In the blackboard pattern, a coordinator agent (or a deterministic coordinator, no LLM) decides which agent gets the next turn. The coordinator holds a "turn token":

```
while not done:
    next_agent = coordinator.select(blackboard)
    if next_agent is None: break
    spawn handoff to next_agent with inherit_blackboard=true, turn_token=token
    wait for result; possibly revoke token; update blackboard
```

**Invariant:** at most one agent holds the turn token at a time. The coordinator is the only writer of the `turn` key on the blackboard.

---

## Group-chat protocol

If you must do group chat (pattern 4), the skeleton is:

```
Moderator (deterministic or LLM) drives a loop:
  msg = agents[current].speak(conversation_history)
  conversation_history.append(msg)
  if stop_condition(conversation_history): return compile_result(...)
  current = moderator.next_speaker(conversation_history)
```

**Required controls:**
- `max_turns` (absolute).
- `max_turns_per_agent` (fairness).
- `topic_drift_detector` (LLM checks whether the conversation is still on-topic every K turns; if not, moderator pulls it back or ends).
- `stop_condition` (explicit: "all agents agree", "a consensus schema filled", etc.).

---

## Routing — picking a child

In Supervisor / worker, the parent needs to route. Three ways:

### a. LLM routing
The supervisor's planner emits a handoff with `target_agent_id` chosen by the LLM. The LLM sees each candidate child's id, description, and input schema. Flexible; subject to hallucinated targets — mitigated by validating against the allowlist before spawn.

### b. Rule-based routing
Child selection is a deterministic function of input shape: a set of routing rules (regex on keywords, schema inspection, classifier result). Predictable; requires up-front design.

### c. Hybrid
LLM routes; a rule-based fast path short-circuits for known patterns. Best for production.

In all cases, the set of possible targets is pinned in the parent's config — no dynamic discovery of agents at run time.

---

## Error propagation

A child's failure shape:

```
child.status = failed | cancelled
child.error  = { code, message, retryable }
```

The parent sees this as an `ActionResult{ok=false, error=child.error}` and decides. Options:

- **Retry** with adjusted inputs (counts against parent's retry budget).
- **Fallback** to a different child.
- **Propagate** — parent itself fails with a wrapped error code `subagent_failed`.

A child failing does NOT automatically fail the parent. The parent's planner decides.

---

## Tracing across agents

Every span carries `trace_id` shared by the root run and all descendants. Each run has its own `run_id`; spans are nested via `parent_span_id`.

The debug UI shows a tree view: root run at the top, handoffs as expandable nodes, each with its own step-by-step trace. This is what makes multi-agent runs debuggable.

---

## Write conflicts between agents

Two agents writing to shared state at the same time. Policy:

| Shared thing | Isolation level |
|--------------|-----------------|
| Blackboard key | Optimistic concurrency; one winner, loser retries |
| Long-term memory (same namespace, same subject) | Serial per subject; writes queue |
| A tool that mutates external state | Tool's idempotency + side-effect isolation |

If two agents are writing to the same LTM subject in ways that could contradict, that is a design smell — prefer a supervisor that reads children's outputs and writes once.

---

## Sub-agent budget leakage — the classic bug

Symptom: parent budget is 100k tokens; child spawns grandchild; grandchild runs wild; parent thinks everything is fine.

Prevention:
- `max_subagent_depth` (hard limit).
- Child budgets are *fractional* of parent-remaining at spawn time — no absolute budgets inherited.
- Grandchild's overage surfaces to child; if the *child* overruns its budget, *it* fails, and the parent sees `subagent_failed`. The parent is not charged beyond the fraction it gave.
- Pre-spawn static check: `handoff.budget_fraction` must be ≤ 0.8 by default to leave headroom for the parent's own continuation.

---

## Observability — the multi-agent debugger

A multi-agent trace without a good UI is unreadable. The debugger must:

1. Show the run tree (root → children → grandchildren).
2. Let you click any run and see its step-by-step trace.
3. Show inbox messages between runs (child outputs → parent).
4. Show blackboard state transitions (with agents highlighted by color).
5. Show cumulative cost per subtree.
6. Replay: step forward/back at any level.

This is not optional for a multi-agent system. Reserve 20% of multi-agent engineering effort for this UI.

---

## Minimal config example — a supervisor with two children

```yaml
agent_id: research-supervisor
persona: |
  Route research tasks to specialized agents. Decide when the task is complete.
planner:
  kind: react
  model: claude-sonnet-4-6
tools: []       # no direct tools — supervisor only handoffs
subagents:
  - name: searcher
    target_agent_id: web-researcher-v2
    default_budget_fraction: 0.35
    return_shape_ref: research_findings_v1
  - name: writer
    target_agent_id: report-writer-v3
    default_budget_fraction: 0.50
    return_shape_ref: report_v1
budget:
  max_tokens: 300000
  max_dollars: 3.50
  max_wall_seconds: 600
  max_subagent_depth: 3
```

The supervisor sees `searcher` and `writer` in its prompt as "sub-agent tools." Its planner calls them in sequence or parallel, per the task.

---

## Patterns to avoid

- **Agents arguing indefinitely.** If your pattern relies on "they'll converge," add a deterministic stop condition and a max-turn counter.
- **Shared mutable state without a protocol.** The blackboard needs a coordinator; two agents writing freely will corrupt state within one run.
- **Hidden circular dependencies.** Agent A calls B; B calls A for a "side question." Ban this by policy; use a supervisor instead.
- **Identical agents as "parallelism".** Running 5 copies of the same agent on the same task is fake parallelism — use parallel tool calls or structured batched outputs instead.
- **Letting the LLM invent target agent ids.** Always validate against an allowlist.

---

## Next: [07 — LLM Provider Layer](./07-llm-provider-layer.md). All of the above issues tool calls and chat completions. That layer is what makes those calls possible across providers.
