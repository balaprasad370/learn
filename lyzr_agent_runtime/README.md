# Agent Runtime — Architecture Design

A Lyzr-style agent runtime designed from first principles. This directory is a spec, not an implementation — every document below describes what must exist, why, and how the pieces fit.

Audience: (a) you, when walking a Lyzr interviewer through "how would you build this"; (b) a future implementer who starts coding from these docs.

---

## Reading order

Read top-to-bottom in a single sitting. Each doc assumes the one before.

1. [01-high-level-architecture.md](./01-high-level-architecture.md) — the ten-thousand-foot view and component boundaries
2. [02-data-model-and-contracts.md](./02-data-model-and-contracts.md) — the core types every component exchanges
3. [03-planning-and-execution.md](./03-planning-and-execution.md) — the beating heart: how an agent decides and acts
4. [04-memory-systems.md](./04-memory-systems.md) — short-term, working, episodic, semantic, procedural
5. [05-tools-and-function-calling.md](./05-tools-and-function-calling.md) — how the agent reaches into the world
6. [06-multi-agent-coordination.md](./06-multi-agent-coordination.md) — more than one agent, safely
7. [07-llm-provider-layer.md](./07-llm-provider-layer.md) — abstraction over OpenAI/Anthropic/others
8. [08-runtime-infrastructure.md](./08-runtime-infrastructure.md) — event loop, concurrency, persistence
9. [09-observability-and-tracing.md](./09-observability-and-tracing.md) — you cannot debug what you cannot see
10. [10-safety-and-governance.md](./10-safety-and-governance.md) — guardrails, PII, cost, policy
11. [11-evaluation-and-testing.md](./11-evaluation-and-testing.md) — measurement over vibes
12. [12-api-surface.md](./12-api-surface.md) — what clients call
13. [13-deployment-and-scaling.md](./13-deployment-and-scaling.md) — production reality
14. [14-roadmap-and-extension-points.md](./14-roadmap-and-extension-points.md) — what is v1 vs v2 vs later

---

## Vision

A production-grade runtime that makes an **agent** a first-class primitive: a stateful, observable, tool-using, memory-carrying unit of LLM work that can be composed with other agents, resumed after a crash, replayed deterministically from logs, and governed by policy.

## Non-goals

- **Not a training framework.** No fine-tuning, no gradient descent, no weight storage.
- **Not a low-code UI.** That lives above the runtime. The runtime exposes APIs.
- **Not a general workflow engine.** Workflows without LLM reasoning steps should use Temporal or Airflow. The runtime is agent-first.
- **Not a vector DB.** Uses one; does not ship one.
- **Not an LLM.** Consumes them through a provider abstraction.

## Design principles (the nine commandments)

These principles are non-negotiable. Every architectural decision in the following docs can be traced back to one of these.

### 1. Determinism where possible, reproducibility always
Every agent run must be replayable step-by-step from its trace. If LLM output is non-deterministic, at minimum the **inputs** to every LLM call and the **decisions** the runtime made from each output must be logged. Given the same trace, we can reconstruct the exact state at any point in time.

### 2. State is a first-class artifact
An agent is not a function call; it is a conversation between the planner, the tools, and memory, evolving over time. State lives outside the process — in a durable store — so agents survive crashes, restarts, and scale-outs.

### 3. Explicit over implicit, always
No magic auto-tool-discovery that breaks silently. No implicit context injection. Every piece of context the LLM sees, every tool the agent can call, every memory it can read is explicitly declared in configuration or code. Surprises in production come from implicit behavior.

### 4. Streaming-first
Token-level streaming from day one. Intermediate events (plan revisions, tool calls, memory writes) stream to clients too. Batch-only APIs are a fallback, not the default. This is what makes an agent feel responsive.

### 5. Provider independence
No LLM provider is special. The core runtime speaks a single internal Chat + Tool + Embedding contract and adapts providers at the edge. Switching from OpenAI to Anthropic to a local model is a config change, not a rewrite.

### 6. Composability over inheritance
Agents compose with tools, with memories, with other agents — by configuration, not by subclassing. A "research agent" is a config: planner + tools + memory slots + sub-agents. There is no `class ResearchAgent(Agent)`.

### 7. Budgets are inputs, not hopes
Every run has explicit budgets: token budget, tool-call budget, wall-clock budget, dollar budget. The runtime stops when any is exceeded. "It just ran for 40 minutes and cost $80" is a runtime bug, not user error.

### 8. Observability is not optional
Tracing, cost accounting, step replay, and eval hooks are part of the core contract. You cannot ship an agent runtime without them. Debugging an agent without a trace is like debugging a program without a stack.

### 9. Fail loud, fail safe
LLM outputs are untrusted input. Tool inputs are validated. Tool outputs are validated. Policy violations halt the run, do not silently continue. Unrecoverable errors surface with full context, not as "agent said sorry."

---

## The thing in one paragraph

An agent runtime is a **state machine driven by an LLM planner**, where transitions are tool calls, memory reads, memory writes, and sub-agent invocations, all governed by a policy layer and observed by a tracing layer. The LLM proposes the next action; the runtime decides whether to execute it, executes it, records the result, updates memory, and loops — until a stopping criterion fires or a budget is exhausted. Everything else in this directory is detail on one of those nouns or verbs.

## What "done" looks like for v1

A v1 runtime must:

- Run a single agent to completion over a ReAct loop, streaming every step.
- Call tools with validated JSON schemas and return validated results.
- Persist state after every step so a crashed run resumes.
- Enforce a token / tool-call / wall-clock / dollar budget.
- Emit a complete trace that replays the run.
- Support two LLM providers (Anthropic + OpenAI) behind one interface.
- Expose a REST + SSE API to start, observe, pause, resume, and cancel a run.
- Scale horizontally behind a queue — any worker can pick up any paused run.

v2 adds multi-agent coordination, structured planning, persistent long-term memory, and an evaluation harness. See [14-roadmap-and-extension-points.md](./14-roadmap-and-extension-points.md).

## Glossary (read this before anything else)

| Term | Meaning in this doc |
|------|---------------------|
| **Agent** | A configured unit of LLM work: planner + toolset + memory + policy + identity. |
| **Run** | A single invocation of an agent from a user input to a terminal state. Has a unique ID. |
| **Step** | One iteration of the planning/execution loop: (observe → think → act). |
| **Planner** | The component that decides what to do next; typically an LLM call with a specific prompt and response schema. |
| **Executor** | The component that performs a decided action (tool call, memory write, handoff). |
| **Tool** | An externally callable function with a typed schema, exposed to the LLM. |
| **Memory** | Any durable state the agent can read from or write to during a run. See [04](./04-memory-systems.md) for the taxonomy. |
| **Trace** | The ordered log of every event in a run. Enables replay, debugging, and eval. |
| **Blackboard** | Shared read/write state between multiple agents in a team. |
| **Handoff** | A structured transfer of control (and context) from one agent to another. |
| **Budget** | A numeric limit (tokens, $, calls, seconds). Exceeding any budget terminates the run. |
| **Guardrail** | A policy check on input, output, or an intermediate step that can halt the run or rewrite the content. |

Proceed to [01-high-level-architecture.md](./01-high-level-architecture.md).
