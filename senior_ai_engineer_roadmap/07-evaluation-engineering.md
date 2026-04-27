# Evaluation Engineering

> Eval is the only thing that turns LLM work from vibes into engineering. Senior signal: you treat evaluation as a first-class system — datasets, harnesses, metrics, regression gates — not an afterthought.

---

## 1. Why eval is the senior bar

Anyone can build a demo. Senior engineers ship systems that **don't regress**, can be **A/B tested**, and let the team **change models confidently**. None of that exists without eval. Interviewers probe eval to separate "I shipped a chatbot" from "I shipped a chatbot we can improve every week without breaking customers."

Three claims that must hold:
1. **You measure what users care about**, not what's easy to measure.
2. **Eval runs on every change** (prompt, model, RAG, tool) before it ships.
3. **You distinguish noise from regression** — sample sizes, confidence intervals, paired comparisons.

---

## 2. The eval taxonomy

| Layer | What it tests | Example |
|---|---|---|
| **Unit / component** | Single LLM call or sub-step | "Extractor returns valid JSON for 100 inputs" |
| **Trajectory / agent** | Sequence of steps the agent took | "Did the agent call search before answer? Did it loop?" |
| **End-to-end / task** | Did the user's goal get completed | "Booking confirmed, correct date, no hallucinated price" |
| **Behavioral** | Cross-cutting properties | Refusals, safety, tone, verbosity, language |
| **Online** | Production telemetry | Thumbs up/down, task completion, escalation rate, latency, cost |

You need all five. Most teams only do (1) and (5) and wonder why production breaks.

---

## 3. Building the eval dataset

### Sources
- **Real production traces** — sample, redact, label. Highest signal.
- **Hand-written golden cases** — for edge cases real traffic doesn't cover yet.
- **Adversarial cases** — known failure patterns (jailbreaks, ambiguous inputs, off-topic, multi-language).
- **Synthetic (with caution)** — generate variants from seeds; useful for breadth, weak for distribution match.

### Sizing
- **Smoke / CI**: 20-50 cases, runs in <2 min, blocks PRs.
- **Regression suite**: 200-1000 cases, runs nightly, blocks releases.
- **Deep eval**: 5k+ cases, runs weekly, paired against production baseline.

### Composition
Stratify by:
- **Use case / intent** (each major flow proportional to traffic).
- **Difficulty** (easy / medium / hard tier so you see where regressions land).
- **Slice** (language, persona, tenant tier, channel).
- **Adversarial bucket** kept separate so it doesn't dilute headline metric.

### Hygiene
- **Versioning** — datasets are code; pin versions in eval runs.
- **Provenance** — for each case: source (prod / hand / synth), labeler, label date.
- **Drift audit** — re-sample real traffic monthly; deprecate cases that no longer reflect users.
- **No leakage** — never train on cases used in eval (true even for prompt iteration if you're rigorous).
- **PII redaction + access control** — eval data is production data.

---

## 4. Metrics — what to actually compute

### Deterministic metrics (cheap, reliable)
- **Schema validity** — JSON parses, required fields present.
- **String / regex match** — for extraction tasks with canonical answers.
- **Numerical / set match** — exact, fuzzy (token-set ratio), or semantic similarity above threshold.
- **Tool-call correctness** — did the agent call the right tool with right params?
- **Latency / cost / token** distributions per request.

### Reference-based (need ground truth)
- **Recall@K, Precision@K, MRR, nDCG** — retrieval (covered in `rag_systems/08-evaluation.md`).
- **BLEU / ROUGE / METEOR** — translation, summarization. Mostly legacy; weak for generative quality.
- **Embedding similarity** — generation vs gold answer. Cheap but noisy.

### Reference-free (no ground truth)
- **Faithfulness** — answer grounded in provided context (LLM-as-judge or NLI model).
- **Answer relevance** — answer addresses the question.
- **Context precision / recall** — when ground truth contexts available (RAGAS).
- **Toxicity / safety** — classifier (Llama Guard, Perspective, Detoxify).
- **Format compliance** — adheres to system instructions (length, language, refusal pattern).

### Trajectory metrics (agents)
- **Task success** — did the goal complete?
- **Steps to success** — efficiency.
- **Tool-call accuracy** — wrong tool chosen, wrong args.
- **Loop rate** — same (state, action) repeated > N times.
- **Recovery rate** — given a planted tool error, did the agent recover?

### Online (production) metrics
- **Thumbs / explicit feedback rate**.
- **Implicit signals** — user retried, edited, abandoned.
- **Resolution / escalation** for support agents.
- **Latency P50/P95/P99**, **error rate**, **cost per session**.

---

## 5. LLM-as-judge — done right

LLM-as-judge is unavoidable for open-ended outputs but easy to do badly.

### Make it reliable
- **Pairwise > pointwise.** "Is A or B better?" is more reliable than "rate A 1-10".
- **Rubric, not vibes.** Provide explicit criteria; ask for reasoning before label.
- **Constrained output.** `{score: 1-5, reasoning: "..."}` schema-validated.
- **Stronger judge than generator.** A Sonnet generating answers + Sonnet judging is cozy; use Opus or a different model family as judge for correlation.
- **Anchor calibration.** Periodically run judge on hand-labeled cases; track agreement (Cohen's kappa, Spearman); recalibrate if it drifts.
- **Multiple judges + majority.** When stakes high; expensive.

### Known biases
- **Position bias** — first option scored higher. Mitigate by random swap + averaging.
- **Length bias** — longer answers preferred. Mitigate by penalty term or length normalization.
- **Self-preference** — model prefers its own outputs. Use cross-family judges.
- **Verbosity / sycophancy** — judge picks confident-sounding wrong answer. Catch via spot-check audits.
- **Cultural / language drift** — judges trained on English may misjudge Hindi / Tamil. Use native speakers for sample audits.

### When to skip LLM-judge
- When you have a deterministic check (schema, regex, exact match) — use it.
- When the task is high-stakes and you can't afford judge errors — use humans.
- When you're optimizing the judge model itself — circular reference.

---

## 6. Eval harnesses

### Pure-OSS
- **OpenAI Evals** — registry-based, supports model graded + custom; mature.
- **Inspect** (UK AISI) — Python-first, batteries-included, growing fast in agent eval.
- **Promptfoo** — config-driven, great DX for prompt iteration; assertion library.
- **DeepEval** — RAGAS-style metrics + pytest integration.
- **RAGAS** — RAG-specific (faithfulness, context precision/recall, answer relevance).
- **TruLens** — feedback functions + dashboard; older but solid.
- **Helicone / Patronus / Galileo** — managed; trace-driven eval.

### Tracing-integrated
- **LangSmith** — datasets, evaluators, online eval, regression dashboards.
- **Langfuse** — open-source LangSmith-equivalent; self-hostable.
- **Phoenix (Arize)** — open-source, OpenTelemetry-native, RAG-centric.
- **OpenLLMetry** — OTel semantic conventions for GenAI; pairs with any OTel backend.

### Picking
- **Tight CI loop** → Promptfoo or DeepEval.
- **Agent / trajectory** → Inspect or LangSmith.
- **RAG-heavy** → RAGAS + Phoenix.
- **Multi-team platform** → LangSmith / Langfuse with shared datasets and custom evaluators.

Senior teams typically combine: dev-loop tool (Promptfoo) + platform tool (LangSmith / Langfuse) + custom Python harness for bespoke metrics.

---

## 7. Eval as part of CI/CD

Make eval **non-skippable** for PRs touching prompts / models / RAG / tools.

```
PR → smoke eval (50 cases, ~2 min) → block on regression
PR merged → regression suite (1000 cases, ~30 min) → alert on regression
Nightly → full eval + slice analysis → dashboard
Pre-release → human eval sample of 100 cases on candidate
```

Regression detection:
- **Headline pass-rate** — drops > X percentage points trigger alert.
- **Per-slice deltas** — regression hidden in subgroup (specific tenant, language).
- **Paired comparison** — same case, baseline vs candidate; report % wins / ties / losses with CIs.
- **Bootstrapped confidence intervals** — 1000 resamples; report 95% CI on metric. Don't compare two means without it.
- **Cost / latency budget** — regression on these is also a regression.

Common pitfall: a flaky LLM-judge causes false regressions. Mitigate with seed pinning, judge model pin, and tolerance bands tuned to noise floor.

---

## 8. Slice analysis (where regressions hide)

Headline metric ticks up, one slice tanks. Always slice by:
- Use case / intent
- Difficulty
- Language / locale
- Tenant tier (free vs paid)
- Channel (web / mobile / phone / API)
- Length bucket
- Newness (cases added in last 30 days)
- Has-tool-call vs no-tool-call

Set per-slice **floors**: any slice dropping below absolute or relative threshold blocks the release even if average is fine.

---

## 9. Online eval / production feedback

Offline eval is necessary but lies. Online eval keeps you honest.

- **Thumbs up/down** with optional reason — keep friction low.
- **Implicit signals** — copy / retry / edit / abandon rate.
- **Auto-eval on production traces** — sample N% per day, run LLM-judge against your rubric, track drift.
- **Escalation / fallback rate** — how often the agent hands off to human.
- **Outcome signals** — for transactional flows, did the user complete the booking / purchase / resolution?
- **A/B framework** — randomized assignment, guardrail metrics (latency, cost, escalation), minimum sample size before reading.
- **Shadow mode** — new prompt/model runs on every request, output not shown, compared offline. No user impact, full distribution coverage.

---

## 10. Specific eval recipes

### RAG eval
- Retrieval: Recall@K, MRR, nDCG against gold-context labels.
- Faithfulness: NLI / LLM-judge — does the answer claim only what's in the cited chunks?
- Citation correctness: do the cited chunk IDs actually support the claim?
- Answer relevance: LLM-judge on question / answer pair.
- Context precision: fraction of retrieved chunks that are actually relevant.
- (See `rag_systems/08-evaluation.md` for depth.)

### Agent / trajectory eval
- **Scenario fixtures** — scripted tool-mocks with planted edge cases (timeouts, missing data, ambiguous results).
- **Replay** — run the agent against the fixture, assert (a) final answer correctness, (b) tool sequence within tolerance, (c) no infinite loop, (d) HITL approvals fired correctly.
- **Differential testing** — same scenario, model A vs model B; track win rate.

### Tool-call eval
- **Schema-valid call rate** per tool.
- **Right-tool-chosen** rate when ground truth tool known.
- **Argument correctness** via JSON-diff or LLM-judge.

### Voice eval
- See `06-voice-agents.md` §10. WER, endpointing precision/recall, turn latency, sampled human ratings on naturalness.

### Safety eval
- Adversarial prompt suite (prompt injection, jailbreak attempts, PII probing).
- Refusal-rate calibration on benign-but-edge cases (false-positive refusals are also bad).
- Toxicity / bias classifiers on outputs.

---

## 11. Cost & ops

Eval is **expensive at scale** — running 5k cases against Opus three times nightly will cost real money. Manage:
- **Tiered eval** — cheap model first, escalate failures to expensive model.
- **Cache** — pin (model, prompt, input) → result; deterministic settings; reuse across runs that don't touch upstream.
- **Sampling** — full suite weekly, smoke daily, slice deep-dives on demand.
- **Parallelism** — eval is embarrassingly parallel; rate-limit aware concurrency.
- **Storage** — keep eval traces; useful for failure analysis weeks later.

---

## 12. Senior interview talk track

If asked "how do you eval an LLM system":
1. **Taxonomy** — name the 5 layers (unit, trajectory, end-to-end, behavioral, online) and where each fits.
2. **Dataset** — sources, sizing, stratification, hygiene; treat as production data.
3. **Metrics** — deterministic-first, then reference-based, then judge-based; understand the trade-offs.
4. **LLM-as-judge** done right — pairwise, rubric, calibration, bias mitigation, anchoring.
5. **Harness** — tooling pick + CI integration (smoke / regression / nightly).
6. **Regression detection** — paired comparison with CIs, slice floors, cost/latency as first-class metrics.
7. **Online eval** — thumbs, implicit, auto-eval on traces, A/B + shadow.
8. **Failure stories** — concrete time you caught (or missed) a regression and what you changed.

If they ask "how would you A/B a new model": coverage of guardrails, sample sizing, slice analysis, shadow first, gradual ramp.

---

## 13. Self-check
- Can you sketch a 5-layer eval taxonomy and place real metrics in each?
- Do you know how to mitigate the 4 main LLM-judge biases?
- Can you describe a CI eval pipeline (smoke / regression / nightly) with concrete sizing?
- Can you explain why bootstrapped CIs matter when comparing two prompt versions?
- Can you list 5 slices to watch for hidden regressions?
- Do you have an answer for "your eval looks good but production is degrading — what's wrong?"

---

**Next:** `08-llmops-and-lifecycle.md` — versioning, A/B, shadow, canary, drift detection, prompt registries, model migration playbooks.
