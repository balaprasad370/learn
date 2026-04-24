# 11 — Evaluation and Testing

Vibes-based iteration is how agent systems regress silently. This doc defines the measurement apparatus: unit tests, integration tests, replay-based regression, golden evaluations, online scoring, and the feedback loop from each back into the codebase.

Reading prerequisite: [09 — Observability and Tracing](./09-observability-and-tracing.md) (we build eval on top of traces).

---

## The five eval tiers

```
Tier 1 — Unit        — pure-function tests (prompt compiler, parsers, schema validators)
Tier 2 — Component   — isolated components with fakes (planner w/ recorded LLM, memory w/ fixed corpus)
Tier 3 — Integration — full runtime, mocked providers + tools
Tier 4 — Offline eval — real agents, real providers (or replay), golden datasets
Tier 5 — Online eval  — live traffic, sampled grading, user feedback
```

Tier 1–3 run in CI. Tier 4 runs pre-release. Tier 5 runs continuously in production.

---

## Tier 1 — Unit tests

Target: **every pure function.** The stronger this tier, the faster you move.

Must-haves:

- **Prompt compiler** — given `(PromptContext, ProviderAdapter)`, assert `ProviderRequest` matches golden output byte-for-byte for a fixed set of contexts. Break on unintended drift.
- **Output parsers** — for each planner strategy, feed a corpus of real LLM responses (including malformed ones) and assert the parse is correct or the repair path triggers.
- **Schema validators** — for each of our core schemas, positive and negative test cases. Reject additionalProperties, nested null handling, enum mismatches, etc.
- **Token estimators** — character heuristic and per-model tokenizer match provider reports within tolerance on a corpus.
- **Redactors** — every PII pattern we claim to redact has a test. New PII patterns start with the failing test.
- **Budget math** — exhaustive tests on budget arithmetic under handoffs, refunds, failures.
- **Loop detector** — craft state sequences with and without loops; assert detection.

Coverage target: 90%+ on pure modules. Coverage is a floor, not a ceiling — read tests for meaningfulness.

---

## Tier 2 — Component tests

Target: **one component at a time, with its collaborators stubbed.**

### Planner tests

Freeze the LLM (recorded response). Feed the planner a constructed `AgentState`. Assert:
- It produces a plan of the expected shape.
- Tool-call args respect the input schema.
- Repair flow fires on bad LLM output and succeeds on the second try.
- Revision flow produces a new plan with `revised_from` set.

### Executor tests

Stub tool implementations. Feed `Action`s. Assert:
- Correct dispatch by `action.kind`.
- Tool invoker retries on `retryable` errors; surfaces non-retryable.
- Parallel calls respect the concurrency cap.
- Idempotency cache hits short-circuit on second call.

### Memory manager tests

Load a fixed corpus. Run queries:
- Assert hybrid retrieval fuses as expected.
- Assert filters (namespace, subject, deprecated_at) are honored.
- Assert dedup on near-duplicate writes.
- Assert supersede chain on contradictions.

### Provider adapter tests

Replay recorded HTTP sessions (using a tool like `vcrpy` or raw fixture files). Assert:
- Request translation produces expected wire format.
- Response translation produces canonical `ChatResponse`.
- Stream normalization yields correct `ChatStreamChunk` sequence.
- Retries fire on configured error classes.
- Error mapping from provider codes to canonical is exact.

### Policy engine tests

Per guard, a table of `(input, expected_decision)`. No LLM calls — use a stub for LLM-based guards.

---

## Tier 3 — Integration tests

Target: **the whole runtime, provider-stubbed, tool-stubbed.**

A test spins up the runtime in-process:
```python
async def test_happy_path_research():
    rt = TestRuntime(
        providers={"anthropic": RecordedProvider("fixtures/happy_research.json")},
        tools={"web_search": StubTool(lambda args: [...])},
    )
    run = await rt.start(agent="research-basic", input="What is Bengaluru population?")
    await run.wait(timeout=30)
    assert run.status == "success"
    assert "13.6" in run.output.final
    assert run.budget_used.tokens > 0
    assert len(run.events) == expected_event_count
```

What these tests catch:
- Wiring bugs between components.
- Config-loading regressions.
- Checkpoint transaction failures (run against a real Postgres in the test harness).
- Crash-recovery semantics (kill the worker mid-step, assert resume from checkpoint produces same final state).

**Crash-recovery test is mandatory.** A harness spawns a worker, issues a `SIGKILL` during step 2, re-spawns, and asserts the run completes with the same output.

---

## Tier 4 — Offline eval (golden datasets)

Target: **the agent, end-to-end, on a curated dataset of inputs with graded expected outputs.**

### Dataset structure

Each case:
```yaml
id: case_001
agent: research-v3
input: "What is the population of Bengaluru and how does it compare to Mumbai?"
expected:
  contains: ["13.6", "21", "Mumbai", "larger"]
  not_contains: ["unknown", "sorry", "unclear"]
  schema: research_result_v1
  grader: numeric_accuracy_v1
budget_max:
  tokens: 50000
  dollars: 0.50
tags: [research, comparison, demographic]
```

### Graders

Graders are functions `(expected, actual) -> Score`. Types:

1. **Exact match** — string or JSON equality.
2. **Substring** — must contain / must not contain lists.
3. **Schema match** — output validates against a JSON Schema.
4. **Numeric accuracy** — extract numbers, compare with tolerance.
5. **LLM-as-judge** — an independent LLM grades quality against a rubric. Used for open-ended tasks; expensive; caveat: judge-bias and distribution-shift.
6. **Code execution** — run generated code against test cases.
7. **Human review** — flag for human rating on ambiguous outputs.

Every grader reports `{score: float, passed: bool, rationale: str}`. The rationale is crucial for understanding regressions.

### Running an eval suite

```bash
lyzr eval run --agent research-v3 --suite research_core --providers anthropic
```

Output:
- Per-case pass/fail + score.
- Aggregate pass rate, average score, latency and cost distribution.
- Comparison to the previous run (diff).
- HTML / Markdown report artifact.

### Replay-based eval

Every production run's trace is eligible for replay. A nightly job:
- Samples yesterday's runs by agent.
- Replays in deterministic mode against the current code.
- Compares: did the result match? Did cost change? Did steps change?

This is how we catch regressions from prompt tweaks and library bumps before they hit users.

### Frozen-model consideration

LLM outputs shift as providers update models. We pin an eval model with an immutable version tag (`claude-opus-4-7-20260101`) for the golden suite. When a new model ships, we re-baseline the suite deliberately — graders that still pass, we move to the new pinned version; those that fail, we investigate (has the agent truly regressed? Or is the grader brittle?).

### Dataset curation

- Start small: 20 hand-written cases covering the agent's primary use cases.
- Grow organically: every production regression, every customer-reported bad output, every edge case discovered in red-team becomes a case.
- Tag aggressively: tags enable subset runs (`--tag demographic` to test only that slice after a related change).
- Version datasets: `research_core_v2` is a separate dataset from `v1`; agents that change surface may need a migrated eval suite.

---

## Tier 5 — Online eval

Target: **live traffic, continuous quality signal, user feedback loop.**

### Signals

- **Explicit user feedback** — thumbs up/down on final output. Aggregated per agent version.
- **Implicit feedback** — did the user continue the conversation? Did they rephrase the same question (sign of failure)? Did they abandon?
- **LLM-as-judge sampling** — randomly sample 1% (or 5% for lower-volume agents) of live runs, have a judge LLM rate against a rubric, track the distribution.
- **Cost outliers** — runs in top 1% cost for an agent; auto-flag for review.
- **Latency outliers** — top 1% duration; auto-flag.
- **Failure analysis** — all failed runs categorized by failure reason; trends by agent version.

### Feedback loop

Bad online runs → tagged as candidates → human curator reviews → promoted to the golden suite (or as a "blocker" regression test).

This closes the loop: production pain becomes a test that breaks the next bad PR.

### A/B testing

When releasing a new agent version, route a portion of traffic to the new version and compare metrics. Shape:

```yaml
experiment:
  id: research-v4-rollout
  agent_a: research-v3     # control
  agent_b: research-v4     # treatment
  traffic_split: 0.1       # 10% to v4
  metrics:
    - user_thumbs_up_rate
    - cost_per_run_usd
    - avg_latency_ms
  stopping_rules:
    - metric: failure_rate
      condition: "rate_v4 > rate_v3 * 1.2"
      action: rollback
```

Stopping rules are enforced automatically: if v4's failure rate significantly exceeds v3's, traffic reverts.

Statistical rigor: bake in sequential testing (mSPRT or similar) so we can stop early on strong signal without inflating false-positive rate.

---

## The eval harness architecture

```
┌───────────────────────────────────────────────────┐
│          Eval CLI                                 │
│  commands: run, compare, baseline, promote        │
└──────────────────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────────────────┐
│         Orchestrator                              │
│  loads suite, spawns runs, collects outputs       │
│  runs graders, aggregates, diffs                  │
└───────────────────────────────────────────────────┘
        │                │                │
        ▼                ▼                ▼
┌──────────────┐  ┌─────────────┐  ┌─────────────┐
│   Runtime    │  │   Graders   │  │  Artifacts  │
│  (real/test) │  │  (pluggable)│  │  (reports)  │
└──────────────┘  └─────────────┘  └─────────────┘
```

Built on top of the real runtime; uses deterministic mode for replay, real mode for "fresh" evaluations.

---

## Shadow-mode execution

To test a new agent version safely before cutover:

```
For each live run:
  primary = run on current agent
  shadow  = run on new agent (recorded, not returned to user)
  compare(primary, shadow) → diff report
```

Shadow runs cost real money. Policy: shadow sample rate tunable per tenant; default 0 (off), enabled for select agents during rollout.

Output: per-run diff with a summary ("92% of shadow runs produced outputs within 0.9 semantic similarity; 8% diverged; 2% failed"). The diverged ones feed curator review.

---

## Reproducibility rules

- Every eval run is identified by `eval_run_id` and produces a complete NDJSON of traces + grades.
- Artifacts are stored in object storage; a report in Postgres links to them.
- Graders are versioned; `grader_version` recorded on every score.
- Datasets are versioned; `dataset_version` recorded.
- Runtime is versioned; `runtime_commit_sha` recorded.

Anyone reading a historical eval report can reproduce it if (a) the pinned LLM is still available and (b) the dataset version is still accessible.

---

## Pre-submit checks (PR gates)

Before a PR merges, CI runs:

1. All Tier 1 tests.
2. All Tier 2 tests.
3. A subset of Tier 3 tests (`--fast`), full set on main.
4. A "smoke" Tier 4 eval — 10 golden cases per modified agent.
5. Lint + type + security scans.

Full Tier 4 runs nightly, against trunk. Tier 5 runs always in prod.

---

## Evaluating memory

Memory retrieval has its own eval surface. For each namespace of interest:

- A fixed corpus (committed to the repo or to a versioned store).
- A query set with expected top-K membership.
- Graders: recall@k, precision@k, MRR.

Memory regressions (embedding model change, chunker change, fusion-weight change) are caught here before they hit the planner.

---

## Evaluating policy

Policy has its own suite:

- **Adversarial corpus** — known prompt injections, PII bombs, secret-exfil attempts. Expected: halt or rewrite.
- **False-positive corpus** — legitimate inputs that look scary (security engineer asking about injection patterns; doctor discussing clinical self-harm cases). Expected: allow.
- **Budget tests** — craft runs that race toward budgets; assert halts.

Policy evals run nightly. Each new guard ships with a corpus.

---

## Evaluating multi-agent

Multi-agent evaluations are harder: you're grading both routing and sub-agent quality.

Tactics:

- **Route-correctness test** — given an input, assert the supervisor routed to the right child (labeled ground truth).
- **Sub-agent in isolation** — grade each child agent as its own unit (Tier 4).
- **End-to-end** — grade the final output of the full tree.
- **Decomposition analysis** — given a complex task, does the supervisor's decomposition look sane? LLM-as-judge on the decomposition itself.

---

## Cost and latency SLOs

Eval tracks cost and latency just as aggressively as accuracy.

SLO examples (per agent):
- p50 latency < 6s, p95 < 20s.
- Average cost < $0.05/run.
- Failure rate < 2%.

A change that improves quality but blows a latency SLO is not shippable without product approval. Eval reports surface these trade-offs explicitly — every report header has a "SLO status" traffic light.

---

## Continuous improvement: the weekly rhythm

Monday — pull last week's online-eval findings; curator reviews flagged runs.
Tuesday — promote issues to golden suite; file bugs for code changes.
Wednesday/Thursday — fix, with Tier 4 gating merges.
Friday — rollout fixes to prod; start A/B where relevant.

This cadence compounds: each week's pain becomes next week's regression test. Agents get better without getting surprisingly worse.

---

## Common failure modes in agent evaluation

- **Graders that degrade silently.** A string-match grader passes because the agent's output shape changed, but the substring still appears — even though the answer is wrong. Mitigation: combine graders; use LLM-as-judge alongside string match.
- **Overfitting to the golden set.** Agent prompts tuned to pass specific cases become brittle. Mitigation: hold out 20% of cases; track held-out performance separately.
- **Brittle provider-tied tests.** Model updates break tests that had specific-token expectations. Mitigation: prefer semantic graders over literal; pin eval model versions.
- **Biased LLM judges.** Judge prefers its own style or the same provider's style. Mitigation: use multiple judges when possible; rotate; spot-check against humans.
- **Cherry-picked scenarios.** Eval set is too "easy." Mitigation: grow the set from production pain — by design, they're the cases where we actually fail.

---

## The evaluation posture, summarized

Build the eval harness before the agent you want to ship is complete. Agents without evals decay. Agents with evals compound. This is not a "nice to have" — it is the reason a v1 becomes a v3 that is actually better.

---

## Next: [12 — API Surface](./12-api-surface.md). We've defined everything the runtime does; now we define what clients see.
