# Multi-Agent Orchestration

> Patterns, frameworks, and trade-offs for systems where multiple LLM-driven agents collaborate. Senior signal: knowing **when not to** use multi-agent is as important as knowing how.

---

## 1. What "agent" means here

An **agent** is an LLM call wrapped in a loop that can:
1. Reason over state (scratchpad / memory).
2. Choose and call **tools** (functions, APIs, retrievers).
3. Receive tool results and decide the next step.
4. Stop when a goal is met or budget is exhausted.

A **multi-agent system** has ≥2 such loops that exchange messages, hand off control, or share state. Agents may have:
- **Different models** (cheap router → expensive solver).
- **Different tools / permissions** (researcher reads, writer writes).
- **Different prompts / personas** (planner, critic, executor).

> Mental model: multi-agent = **modular cognition**. You're trading single-prompt simplicity for the ability to specialize, parallelize, and gate.

---

## 2. When to use multi-agent (and when not to)

### Use multi-agent when
- **Distinct skills / tool sets** that don't compose well in one prompt (e.g., SQL agent + code-interpreter + web-search).
- **Parallelizable subtasks** (research 5 competitors at once).
- **Quality gating** — you want a separate critic / verifier that can reject the actor's output.
- **Permission boundaries** — read-only researcher must not be able to call write APIs.
- **Different latency / cost tiers** — Haiku router, Sonnet workers, Opus only on hard branches.

### Avoid multi-agent when
- A single well-prompted agent + good tools works. Multi-agent adds latency, tokens, failure modes, and debugging cost.
- The task is mostly linear (parse → transform → respond) — that's a **pipeline**, not multi-agent.
- You can't define **clear handoff conditions** — implicit handoffs cause loops.
- You haven't built **eval** for the single-agent baseline yet. You'll never know if multi-agent is actually better.

> Senior heuristic: start single-agent + good tools. Add a second agent only when you can name the **specific failure mode** that single-agent can't fix.

---

## 3. Canonical orchestration patterns

### 3.1 Supervisor / Router
One **supervisor** agent decides which **worker** to call next. Workers don't talk to each other; they return to supervisor.

```
        ┌──────────┐
        │Supervisor│
        └────┬─────┘
   ┌─────────┼─────────┐
   ▼         ▼         ▼
[Search] [SQL Agent] [Writer]
```

- **Pros:** simple control, easy to log, easy to add workers.
- **Cons:** supervisor becomes the bottleneck and the smartest model (cost).
- **Use:** customer-support routing, "do research then summarize" flows, most production systems.

### 3.2 Hierarchical (supervisor of supervisors)
Supervisors can themselves be agents that own sub-teams. Useful when you have **domains** (legal team, finance team, ops team) each with their own routing.

- **Pros:** scales org-style; encapsulation.
- **Cons:** hops add latency. Plan for **handoff metadata propagation** (auth, tenant, trace ID).

### 3.3 Sequential pipeline (chain of agents)
A → B → C, each consuming the prior output. Often better expressed as a **DAG** rather than free-form agents.

- **Pros:** deterministic, cheap, easy to eval.
- **Cons:** no adaptation. If you don't need adaptation, **don't call this multi-agent** — call it a pipeline.

### 3.4 Parallel fan-out / map-reduce
Supervisor splits work, runs N agents concurrently, then a **reducer** merges. Classic for research ("compare 5 vendors") and batch analysis.

- **Pros:** wall-clock latency drops linearly.
- **Cons:** token cost is N×; merging is non-trivial (dedup, conflict resolution).
- **Tip:** cap N (often 3–7). Use **structured output** so the reducer can merge programmatically.

### 3.5 Debate / multi-perspective
Two or more agents argue from different angles; a judge picks. Used in research labs more than production. Production form: **actor + critic** loop (next).

### 3.6 Actor–Critic (a.k.a. reflexion)
Actor proposes, critic finds flaws, actor revises. Cap iterations (typically 1–3).

- **Pros:** measurable quality lift on reasoning-heavy tasks.
- **Cons:** 2-3× tokens. Critic must be **calibrated** — over-critical critics cause infinite revision.
- **Stop conditions:** critic returns "no issues", max-iter reached, or revision diff below threshold.

### 3.7 Handoff / Swarm
Agents transfer control to each other directly via a `handoff(target_agent)` tool call. Pioneered by **OpenAI Swarm**. Resembles a state machine with LLM-decided transitions.

- **Pros:** less central coordination; each agent only knows its own scope and which neighbors it can hand off to.
- **Cons:** hard to debug; easy to loop; needs explicit handoff graph + max-hop budget.

### 3.8 Plan-and-execute
Planner emits a multi-step plan once; executor runs each step; replanner triggered only on failure. Reduces decision count vs ReAct-on-every-step.

- **Pros:** cheaper, more steerable, easier eval.
- **Cons:** brittle if plan was wrong and replan is rare.

---

## 4. Frameworks — what to know

### 4.1 LangGraph (LangChain)
Graph-based agent runtime. Nodes are functions (often LLM calls); edges are transitions; state is a typed dict reduced across nodes.

- **Strengths:** explicit state, **checkpointing** (persist state per thread → resume / time-travel / human-in-the-loop), conditional edges, subgraphs, streaming, LangSmith integration.
- **When to pick:** production agents where you need durability, HITL approval steps, or replay for debugging.
- **Gotchas:** verbose; reducer functions must be associative; vendor-y if you fight the abstractions.

### 4.2 CrewAI
Role-based abstraction: define `Agent`s (with role + goal + backstory) and `Task`s; `Crew` runs them sequentially or hierarchically.

- **Strengths:** fastest from-zero for "team of personas" use cases; nice DX for prototypes and demos.
- **When to pick:** content workflows, research crews, internal tools.
- **Gotchas:** abstractions hide latency / cost; harder to do anything outside the role/task mental model. Limited fine-grained control.

### 4.3 Microsoft AutoGen
Conversational multi-agent: agents exchange messages in **GroupChat** with a manager that picks next speaker. Strong for code-gen / executor patterns.

- **Strengths:** mature; tool use + code execution baked in; supports nested chats; AutoGen Studio UI.
- **When to pick:** code-generation pipelines, research agents that execute generated code, simulation.
- **Gotchas:** chat-as-protocol can balloon tokens; turn-selection logic is central to behavior.

### 4.4 OpenAI Swarm (and the newer Agents SDK)
Lightweight handoff library — agents are functions returning either a message or `handoff(target)`. The successor **OpenAI Agents SDK** generalizes this with built-in tracing, guardrails, and provider-agnostic models.

- **Strengths:** minimal API, very readable code, explicit handoff graph.
- **When to pick:** stateless routing systems, support triage, when you want to own the loop.
- **Gotchas:** Swarm itself was a reference impl; production should use the Agents SDK or LangGraph-equivalent.

### 4.5 Others worth naming
- **Lyzr Agent Framework** — the runtime you've documented; positions as multi-agent + memory + safety + eval out of the box.
- **LlamaIndex Agents / Workflows** — event-driven workflow runtime, tighter integration with their RAG stack.
- **Inngest / Trigger.dev / Temporal** — durable workflow engines you can layer LLM calls onto. Not "agent frameworks" per se but increasingly used as the backbone (see §6).
- **Anthropic Claude Agents (claude.ai/code)** and **OpenAI Assistants API** — hosted runtimes with built-in tools (code interpreter, file search).

### 4.6 Picking a framework
- **Throwaway demo / hackathon** → CrewAI or Swarm.
- **Production agent with HITL or long-running state** → LangGraph + a durable workflow engine.
- **Code-gen heavy** → AutoGen.
- **You already own the loop and want minimal magic** → write it yourself with the model SDK + a tracing tool. Honest answer for many senior teams.

---

## 5. Designing the handoff (the actual hard part)

Handoffs are where multi-agent systems silently fail. Specify them explicitly.

For each handoff edge define:
- **Trigger condition** — what tool call / state predicate fires it.
- **Payload** — exactly what state moves to the next agent (don't dump full history; **summarize**).
- **Permissions delta** — what tools become available / unavailable.
- **Rollback** — can the receiver hand back? Under what condition?
- **Termination guard** — global max hops, per-edge cooldown, repeated-handoff detector.

**Anti-patterns:**
- Free-text handoffs ("call the writer when ready") → unreliable. Use **structured tool calls**.
- Passing the entire message history each hop → token blow-up. **Compress** to facts the next agent needs.
- No max-hop counter → loops cost real money before you notice.

---

## 6. State, memory, and durability

Multi-agent systems amplify the **state problem**. Three layers:

| Layer | Lifetime | Examples |
|---|---|---|
| **Working memory** | Single agent turn | Scratchpad, current message list |
| **Session memory** | Conversation / task | Thread state, partial plan, intermediate results |
| **Long-term memory** | Across sessions | User profile, past tickets, learned preferences (vector + KV) |

For production:
- **Checkpoint state** at each node boundary (LangGraph does this; you can do it yourself with Postgres). Lets you resume after crash, retry one step, or insert HITL approval.
- **Externalize long-running waits** to a workflow engine (Temporal / Inngest / SQS + worker). LLM calls inside an in-process loop don't survive deploys.
- **Trace IDs propagate end-to-end** — one trace covers all sub-agents. Without this, debugging is impossible.

---

## 7. Human-in-the-loop (HITL)

Senior systems treat HITL as a **first-class node**, not an afterthought.

- **Approval gates:** before destructive tool calls (send email, charge card, push code).
- **Clarification:** agent emits a question, system pauses, human answers, state resumes.
- **Override / takeover:** human can edit the agent's plan or message and resume.

Implement with:
- Persisted state + a typed "interrupt" event.
- A queue (or webhook) the human UI consumes.
- A **resume token** the UI returns with the answer.

LangGraph has `interrupt_before` / `interrupt_after` for exactly this; AutoGen has `human_input_mode`.

---

## 8. Evaluation for multi-agent

Single-agent eval (input → output) is insufficient. Add:
- **Trajectory eval** — was the sequence of tool calls reasonable? Did it loop?
- **Per-node eval** — did the planner produce a valid plan? Did the critic catch the planted bug?
- **End-to-end eval** — task success rate on a frozen scenario set.
- **Cost / latency per task** — multi-agent that's 2× better at 10× cost may not ship.

Tools: **LangSmith** datasets + evaluators, **Inspect** (UK AISI), **Promptfoo**, **OpenAI Evals**, **Phoenix** for tracing-driven eval. Covered deeper in `07-evaluation-engineering.md`.

---

## 9. Failure modes and mitigations

| Failure | Symptom | Mitigation |
|---|---|---|
| **Handoff loop** | A → B → A → B forever | Max-hop counter; loop detector on (agent, state-hash) tuples |
| **Token explosion** | Each hop appends history | Compress / summarize at handoff boundary |
| **Critic over-rejects** | Endless revision cycles | Cap iterations; calibrate critic on labeled set; raise threshold |
| **Planner hallucinates tools** | "Call tool X" that doesn't exist | Validate plan against tool registry before execute; constrained decoding for tool names |
| **State drift** | Different agents see inconsistent facts | Single source of truth (workflow state); pass references not copies |
| **Lost trace** | Can't reproduce production bug | Mandatory trace ID propagation; persisted spans for every LLM + tool call |
| **Permission leak** | Read-only agent calls write tool | Per-agent tool whitelist enforced at runtime, not just in prompt |
| **Cascading failure** | One sub-agent's bad output poisons downstream | Schema-validate every agent output; reject + retry on malformed |
| **Stuck on flake** | Tool times out, loop hangs | Timeouts on every tool; circuit breakers; dead-letter for stuck threads |

---

## 10. Cost & latency math

Rough envelope for a 3-agent pipeline (planner → executor → critic), each averaging 800 in / 400 out tokens, Sonnet-tier pricing (~$3/M in, $15/M out):

- Per call: 800 × $3/M + 400 × $15/M ≈ $0.0024 + $0.0060 = **$0.0084**
- Per task (3 agents, 1 critic loop): ~6 calls ≈ **$0.05/task**.
- At 100k tasks/day: **$5k/day** (~$1.5M/yr). Real systems also pay for retries, embeddings, and tool calls.

Latency: each hop adds ~2-5s for non-streamed agent turns. A 4-hop pipeline easily takes 10-20s end-to-end — unacceptable for chat, fine for async workflows. **Stream what you can** (planner + writer can stream; critic usually can't until done).

Cost knobs: cheaper model for router/critic, **prompt caching**, parallel fan-out instead of serial, smaller context via summarization, structured outputs to skip parse retries.

---

## 11. Security considerations

Multi-agent expands the attack surface vs single-agent:
- **Tool-via-tool injection** — a retrieved doc tells the search agent to "now call `transfer_funds`". Validate at every tool boundary, not just user input. (Detail in `09-safety-and-guardrails.md`.)
- **Privilege creep** via handoff — receiving agent must re-validate it's allowed to do the requested op.
- **Cross-tenant leakage** — passing memory between agents must carry tenant ID; the data store enforces.
- **Action confirmation** — destructive tool calls behind HITL or signed user intent.

---

## 12. Senior-interview talk track

When asked "design a multi-agent system for X", structure the answer:
1. **Single-agent baseline first** — would this work with one good prompt + tools? Why isn't it enough?
2. **Pick the pattern** — supervisor / pipeline / actor-critic / fan-out — and **justify** against the failure mode you're solving.
3. **Specify handoffs** — trigger, payload, permission delta, termination guard.
4. **State + durability** — checkpointing strategy, workflow engine choice.
5. **Eval** — trajectory + end-to-end + cost/latency budget.
6. **Failure modes** — name 3 and how you detect/mitigate.
7. **Knobs to scale down complexity** — what you'd remove if metrics didn't justify the cost.

The signal interviewers want: you can articulate **why each agent earns its keep** and what you'd cut first.

---

## 13. Self-check
- Can you name 3 patterns and a real use case where each is the right pick?
- Can you list 5 failure modes specific to multi-agent and the detection signal for each?
- Do you know when LangGraph beats CrewAI beats writing the loop yourself?
- Can you sketch a handoff spec (trigger / payload / permission / termination) for a 2-agent system?
- Can you put a $/task and latency budget on a 3-agent pipeline without a calculator?

---

**Next:** `05-mcp-and-tool-protocols.md` — Model Context Protocol, function-calling design, tool security and sandboxing.
