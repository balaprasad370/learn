# LLMOps and Lifecycle

> Versioning, deploying, monitoring, and migrating LLM-powered systems. The "ops" half of the role: prompts, models, RAG indexes, fine-tunes, and agents all evolve weekly — your job is to keep that change safe.

---

## 1. What's different from MLOps

Classic MLOps: train → eval → register → deploy → monitor a single model.

LLMOps adds:
- **Prompts as code** — text artifacts that ship like binaries, but mutate weekly.
- **Multiple model providers** — OpenAI, Anthropic, Google, plus open-weight; you'll swap.
- **External knobs you don't control** — provider model updates, rate limits, deprecations.
- **RAG indexes** are versioned artifacts too (embedding model + chunking + corpus snapshot).
- **Agents** combine all the above into longer chains; one regression cascades.
- **Cost** is a first-class deploy metric, not a finance afterthought.

Senior signal: you treat prompts, models, and indexes as **versioned, evaluated, gated, and reversible**.

---

## 2. The artifact catalog

Every LLM system has these artifacts. Each needs a version, an owner, and a deploy story:

| Artifact | Versioning | Storage |
|---|---|---|
| **Prompt** | Semver or hash; immutable once tagged | Git, prompt registry (PromptLayer, LangSmith, Langfuse, Helicone) |
| **Model selection** | Provider + model ID + config (temp, top_p, max_tokens) | Config / feature flag |
| **Tool schemas** | Versioned per tool; backward-compat rules | Git |
| **RAG index** | (corpus_snapshot, embedding_model, chunker, params) | Vector DB + manifest |
| **Fine-tuned model** | Base + dataset + recipe + checkpoint | Model registry (HF, MLflow, WandB) |
| **Eval dataset** | Versioned; pinned per release | Git or eval platform |
| **Guardrail config** | Threshold versions per model version | Git |

If any of these mutate without version control, you've lost the ability to bisect regressions.

---

## 3. Prompt versioning and registries

### Discipline
- **Prompts in Git** is fine for small teams; combine with hash-based deploy keys.
- **Prompt registry** (PromptLayer, Langfuse Prompts, LangSmith Prompt Hub, Helicone Prompts, Pezzo) once non-engineers (PM, content) need to edit.
- **Pin by version, not "latest"**, in production. `prompt://booking_intent@v17`. "Latest" auto-deploy is a footgun.
- **Diff visibility** — prompt PR shows side-by-side eval results before merge.

### What a prompt version contains
- The text (with named variables).
- The intended model (versions degrade across models — Sonnet 3.5 prompt may not work on Sonnet 4).
- Tool schemas referenced.
- Eval pass-rate at time of tag.
- Owner + change reason.

### Anti-patterns
- f-string templating spread across the codebase — no central registry → no audit.
- Editing prompts in a database via admin UI with no diff history.
- Hot-swapping prompts in prod without an eval gate.

---

## 4. Model rollouts — shadow, canary, A/B

When changing model (provider, version, fine-tune) or prompt:

### Shadow
- New version runs **in parallel** with prod for every request.
- Output **discarded** (or scored offline); user sees old version.
- **Use when**: you want full distribution coverage without user risk. Cost = 2× LLM bill during the window.

### Canary
- Small % (1-5%) of traffic routed to new version.
- Real users see it; metrics watched in real time.
- **Auto-rollback** on guardrail breach (error rate, latency, cost, user feedback).
- **Use when**: you're past shadow and ready for real signal.

### A/B
- Larger split (10-50%); randomized per user or session.
- Powered comparison on primary KPI (task success, CSAT) with guardrails.
- Run until **statistically significant** sample.
- **Use when**: deciding between two viable options; need quantified win.

### Gradual ramp
- 1% → 5% → 25% → 50% → 100%, each step holding 24-72h.
- Enables catching slow-burn regressions (cost drift, weekly seasonality).

### Holdout
- Permanent 1-5% slice always on baseline; lets you measure compounding gains over time.

---

## 5. Feature flags / config-driven model selection

Don't hard-code model IDs. Use a config layer:

```python
model = config.get("intent_router.model")  # "claude-haiku-4-5" today
```

Backed by **LaunchDarkly / Unleash / Statsig / GrowthBook / Flipt** or a homegrown KV. Lets you:
- Roll back without redeploy.
- Per-tenant overrides (enterprise customer pinned to known-good model).
- Per-region overrides (GDPR, sovereignty).
- Kill switches for safety incidents.

Pair with **per-model adapter** in code so swapping providers is a one-line change. Don't leak provider-specific calls everywhere.

---

## 6. RAG index lifecycle

Re-indexing is the hardest deploy in RAG. Plan it.

### Triggers
- Embedding model upgrade.
- Chunker change (size, overlap, structure).
- Corpus refresh (full or incremental).
- Schema migration (new metadata).

### Patterns
- **Blue-green index** — build new index in shadow, switch alias atomically. Zero downtime.
- **Dual-write + cutover** — both indexes accept writes during build, reads cut over once new is warm.
- **Incremental** — for corpus updates: per-document hash → re-embed only changed.
- **Backfill window** — large corpora may take hours-days; rate-limit embedding calls; checkpoint progress.

### Eval gate
- Pre-cutover: retrieval metrics (Recall@K, MRR) + downstream RAG eval on regression suite.
- Post-cutover: monitor Recall@K + thumbs delta; rollback alias if regression.

### Storage
- Keep the **N most recent index versions** for emergency rollback (week-long window typical).
- Vector DB supports namespaces / collections / aliases — use them.

Detail in `rag_systems/10-production-playbook.md`.

---

## 7. Fine-tuned model lifecycle

If you ship fine-tunes:
- **Recipe in Git** — base model, dataset version, hparams, training script.
- **Reproducible builds** — pinned seeds, deterministic data ordering, container with frozen deps.
- **Model registry** — HF private repo, MLflow, WandB; tag with semver + parent base.
- **Eval gate** — domain eval + general capability eval (regression on base capabilities, e.g. MMLU drop).
- **Deploy** — same canary / shadow / A/B path as commercial models.
- **Re-train cadence** — define triggers (data drift, X% new examples, performance drop).
- **Sunset plan** — deprecation when base model is upgraded; how do you migrate adapters?

Detail in `11-fine-tuning-handson.md`.

---

## 8. Drift detection

Three drifts that bite LLM systems:

### Input drift
User queries change distribution (new product launch, news event, season).
- Track query topic distribution (cluster + label) week over week.
- Track length / language / channel mix.
- Trigger: > X% deviation → re-evaluate prompts on new distribution.

### Output drift
Same prompt, same model, different outputs over time (provider changed model under the hood, data shift).
- Run **canary prompt set** daily — fixed 50 inputs, log outputs, embed, compare embedding distance to baseline.
- Spike → investigate (was it an Anthropic update? OpenAI rolling deploy?).

### Performance drift
Real metrics degrade silently — thumbs, escalation, task success.
- Time-series with control limits (CUSUM, EWMA).
- Page on sustained degradation, not single-day blip.

### Provider deprecations
Calendar reminders for known sunset dates; integration tests against `gpt-4o-mini-2024-07-18`-style pinned versions; auto-PRs when the deprecation API surfaces a new replacement.

---

## 9. Cost lifecycle

Cost is a deploy gate, a metric, and a regression target.

- **Cost budget per request** — set per use case; alert on breach.
- **Cost regression** in CI — a prompt change that doubles tokens fails the eval gate even if quality is up, unless ROI justifies.
- **Per-tenant attribution** — multi-tenant systems need per-customer cost; tag every LLM call with `tenant_id`, aggregate.
- **Plan-level rate limits** — free tier capped; enterprise gets dedicated capacity.
- **Anomaly detection** — sudden 10× cost on a customer = bug, abuse, or success; either way investigate.
- **Negotiate volume tiers** — at $50k+/mo you can negotiate with Anthropic / OpenAI / Google; bake into FinOps cycle.

Detail in `10-observability-and-cost.md`.

---

## 10. Incident response for LLM systems

Failure modes unique to LLMs:
- **Provider outage** — 4xx / 5xx spikes from OpenAI / Anthropic. Mitigation: multi-provider fallback with per-provider circuit breaker; degraded-mode static responses.
- **Latency spike** — provider slow; switch to fallback or smaller model; queue depth alarm.
- **Quality regression** — caught by online eval or thumbs spike. Roll back via feature flag.
- **Cost spike** — runaway loop, abuse, or new traffic. Per-tenant rate limits; kill switch.
- **Safety incident** — jailbreak in production, harmful output. Pre-built playbook: incident commander, comms template, containment via guardrail tightening, postmortem, customer notification rules.
- **Index corruption** — vector DB returns wrong/empty; rollback alias; trigger reindex.
- **Tool dependency breakage** — third-party API change. Tool-level circuit breaker; fallback responses.

Have **runbooks per failure class** and exercise them in game days.

---

## 11. Multi-provider strategy

Don't single-source. Wire at minimum:
- **Primary** — your default model.
- **Fallback** — different provider, similar capability tier; auto-failover on 5xx / timeout.
- **Cheap fallback** — degraded answer better than no answer.
- **Routing layer** — could be your gateway (LiteLLM, Portkey, Helicone, Kong AI Gateway, Cloudflare AI Gateway) or in-house.

Watch for:
- **Eval per provider** — same eval suite per fallback; you must know what users see when failover triggers.
- **Schema differences** — tool-call shape varies per provider; abstract behind your own adapter.
- **Pricing & latency drift** — provider economics change; recompute routing quarterly.

---

## 12. Rollback and reversibility

Every change must be reversible in <5 minutes.
- Feature flags > deploys for model / prompt swaps.
- Vector DB index aliases for RAG rollback.
- Tool schema versions side-by-side; clients pin.
- Database migrations: never destructive in same release as the consumer change.
- Observability with **change correlation** — every deploy emits a marker on dashboards; outage triage starts at "what changed in the last 60 min".

---

## 13. Governance + compliance

For enterprise / regulated customers:
- **Data residency** — route requests to in-region providers (Azure OpenAI EU, Anthropic via Bedrock in-region, self-hosted in-region open weights for IN/EU/sovereign).
- **PII handling** — redact before logging; log retention SLAs; right-to-deletion plumbing.
- **Audit log** — every prompt, model, tool call, response with user + timestamp; immutable.
- **Approved-model list** — models customers contractually allow; flag rejects unapproved.
- **DPA + sub-processor list** — keep updated; new model providers = new sub-processor disclosure.
- **Model card / system card** — public-facing description of system behavior, limits, safety.

---

## 14. Senior interview talk track

"How do you ship a new prompt / model to production?"
1. **Diff in PR** — old vs new with eval pass-rate side-by-side.
2. **Eval gate** — smoke (block PR), regression (block release), with paired CI on slice floors.
3. **Shadow** — full-distribution sanity check; compare offline.
4. **Canary** — 1-5% traffic, guardrail watch, auto-rollback.
5. **Ramp** — 1 → 5 → 25 → 50 → 100, hold per stage.
6. **Holdout** — permanent baseline slice for long-term tracking.
7. **Observability** — change marker on dashboards; cost / latency / quality / safety panels.
8. **Reversibility** — feature flag, prepared rollback plan.
9. **Communication** — release notes to stakeholders; postmortem if anything paged.

If they ask "how do you handle a provider deprecation": pinned model IDs in config, deprecation calendar, parallel eval on candidate replacements, planned cutover window with shadow + canary.

---

## 15. Self-check
- Can you list every artifact that needs versioning in an LLM system?
- Can you describe shadow vs canary vs A/B and when each is right?
- Do you know how to deploy a re-indexed RAG corpus with zero downtime?
- Can you name 3 drift types and the signal you'd watch for each?
- Do you have a runbook for "Anthropic returns 5xx for 10 minutes"?
- Can you explain why hard-coding model IDs in app code is a senior-level smell?

---

**Next:** `09-safety-and-guardrails.md` — prompt injection, jailbreaking, NeMo Guardrails, Llama Guard, content moderation, the safety stack for enterprise AI.
