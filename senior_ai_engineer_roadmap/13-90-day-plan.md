# 90-Day Plan

> Sequenced study + practice + applications. Daily / weekly cadence with milestone gates. Designed to start **2026-04-27** and end **2026-07-26**, mapped to current job-search context.

---

## 0. Operating principles

1. **Volume beats perfection.** Ship 3 medium things, not 1 perfect thing.
2. **Output > input.** Every week produces a writeup, a mock, a project, or an application — not just reading.
3. **Measure interview readiness, not topic coverage.** The metric is "could I pass a 45-min round on this today?", not "did I read it?"
4. **Apply continuously.** Don't wait until "ready". Calibration comes from real loops.
5. **Replay your own work.** Record mocks, re-watch, list 3 fixes. The single highest-leverage practice.
6. **Protect deep-work blocks.** 90 min, no Slack, no email, no LLM-distraction. 2-3 per day on study days.

---

## 1. The 90-day arc

| Phase | Days | Focus | Output |
|---|---|---|---|
| **Foundations** | 1-30 | Distributed systems + system design + LLM serving + RAG depth refresh | Master the canonical 7 system-design prompts; 4 mocks done; resume + LinkedIn polished |
| **AI Production** | 31-60 | Agents + MCP + voice + eval + LLMOps + safety + observability | Ship 1 end-to-end project; 8 mocks total; 5+ active loops in progress |
| **Conversion** | 61-90 | Fine-tuning + interview optimization + comp negotiation + closing | 12+ mocks total; 2-3 offers in hand; final pick + start date |

Phase boundaries are soft; don't gate progress on perfection.

---

## 2. Phase 1 — Foundations (Days 1-30)

### Goals (gate to phase 2)
- Can run the 7-step system-design framework cold for any of the 7 canonical AI prompts.
- Can answer 60-90 sec on every "AI depth" question in `12-interview-prep.md` §4.
- 4 system-design mocks completed + reviewed.
- Resume + LinkedIn updated; 10 active applications in pipeline.
- 1 portfolio project in flight (RAG-as-a-service or agent platform — pick the one that overlaps your target companies).

### Week-by-week

**Week 1 (Apr 27 – May 3)**
- Read `00-curriculum-map.md` end-to-end. Self-assess every leaf 🟢/🟡/🔴.
- Read `01-distributed-systems.md`. Cover gaps with 1 deep-dive paper or talk.
- Resume rewrite focused on AI-relevant scope; quantified outcomes; senior signals.
- LinkedIn rewrite: headline, About, role descriptions, skills.
- Identify top 12 target companies; queue applications.
- **Mock 1:** RAG-as-a-service system design (45 min, recorded).

**Week 2 (May 4 – May 10)**
- Read `02-system-design-for-ai.md`. Memorize the 7 canonical prompts.
- Build a tiny RAG end-to-end for a personal corpus (your notes / docs). Ingest, embed, retrieve, rerank, answer with citations. Use whatever stack — but ship it in 6-8h max.
- Apply to top 5 of the 12 (Lyzr, Sarvam, Krutrim, Composio, Giga).
- **Mock 2:** Voice agent at scale (Giga-style).
- Write 1 STAR story (0→1 launch). Practice out loud.

**Week 3 (May 11 – May 17)**
- Read `03-llm-serving-infra.md`. Memorize: vLLM internals, PagedAttention, continuous batching, KV-cache math, GPU price table.
- Stand up vLLM locally with a small open model (Qwen 2.5 7B or Llama 3.1 8B). Benchmark prefill / decode / throughput. Take notes.
- Apply to next 5 (Together, Modal, Replicate, Razorpay AI team, CRED AI team).
- **Mock 3:** LLM gateway with multi-provider fallback.
- Write 2 more STAR stories (incident, mentoring).

**Week 4 (May 18 – May 24)**
- RAG depth refresh — re-read your `rag_systems/05-retrieval-strategies.md`, `08-evaluation.md`, `10-production-playbook.md`. Write a 1-pager from memory on each.
- Add reranker + eval to your week-2 RAG project. Score recall + faithfulness on a hand-built 30-case set.
- Outreach to 5 founders / hiring managers via LinkedIn (Giga's Esha + Varun, Lyzr's founders, Sarvam, Composio).
- **Mock 4:** Agent platform (Lyzr-style).
- Phase 1 review: which gaps remain 🔴? Schedule them into Phase 2.

### Phase 1 deliverables
- Updated resume + LinkedIn.
- 10 applications submitted, 5 founder outreaches sent.
- Working RAG project with eval (in a public or private repo).
- 4 mocks recorded, self-reviewed, fixes documented.
- 3 STAR stories written + rehearsed.

---

## 3. Phase 2 — AI Production (Days 31-60)

### Goals (gate to phase 3)
- Can speak fluently to multi-agent, MCP, voice, eval, LLMOps, safety, observability at 60-90 sec each.
- Ship 1 end-to-end portfolio project that demonstrates production thinking (eval + observability + cost notes + readme).
- 8 total mocks; 5 active interview loops in progress.
- 5 STAR stories rehearsed.

### Week-by-week

**Week 5 (May 25 – May 31)**
- Read `04-multi-agent-orchestration.md` + `05-mcp-and-tool-protocols.md`.
- Add a tool to your RAG project — e.g., a "search the web" tool. Wire as an agent loop with explicit handoff conditions.
- **Mock 5:** Coding agent with sandboxed execution.
- 2 active loops should be in screen / first-round stage by now. Update tracker.

**Week 6 (Jun 1 – Jun 7)**
- Read `06-voice-agents.md`. If targeting Giga / Sarvam / Vapi: stand up a Pipecat or LiveKit Agents demo locally (cascaded STT → LLM → TTS).
- Measure end-to-end turn latency. Write up the breakdown per stage.
- **Mock 6:** Voice agent with telephony specifics (Indian DIDs, barge-in, cost model).
- Apply to 5 more (Inngest, Hugging Face, LangChain, Modal, Anyscale, or Indian: Neysa, Simplismart, Ema, Gupshup, Haptik).

**Week 7 (Jun 8 – Jun 14)**
- Read `07-evaluation-engineering.md` + `08-llmops-and-lifecycle.md`.
- Add CI eval to your portfolio project: smoke (50 cases) blocks PRs; nightly regression suite. Use Promptfoo or DeepEval.
- Write a deploy plan (canary / shadow / ramp) in the README.
- **Mock 7:** Prompt + model deploy pipeline (PromptOps end-to-end).
- 2 STAR stories: scope-down decision, conflict resolution.

**Week 8 (Jun 15 – Jun 21)**
- Read `09-safety-and-guardrails.md` + `10-observability-and-cost.md`.
- Add OTel tracing (OpenLLMetry or OpenInference SDK) to your project. Trace tree visible in Phoenix or Langfuse.
- Add a Llama Prompt Guard input classifier + a HITL approval node for any destructive tool call.
- Write a 1-page postmortem template.
- **Mock 8:** Observability + cost platform design.
- Phase 2 review: how many 🔴 remain? Re-prioritize into Phase 3.

### Phase 2 deliverables
- Portfolio project at v2: agent + RAG + tools + CI eval + tracing + safety. Public README.
- 8 mocks total, recorded.
- 5+ active loops at various stages; 2-3 reaching final rounds.
- 5 STAR stories rehearsed.
- Outreach: 10 more LinkedIn DMs to founders / hiring managers.

---

## 4. Phase 3 — Conversion (Days 61-90)

### Goals (gate to ship)
- 12+ total mocks.
- 2-3 offers in hand by day 90.
- Negotiated comp at top of asking band.
- Picked role; start date set.

### Week-by-week

**Week 9 (Jun 22 – Jun 28)**
- Read `11-fine-tuning-handson.md`. Optionally fine-tune a tiny LoRA (Qwen 2.5 3B SFT on a 1k-example task). Write up; don't go deep — it's a differentiator only if it comes up.
- Re-read `12-interview-prep.md` cover-to-cover.
- Prep loops in flight: write per-company tactical notes; rehearse company-tailored STAR stories.
- **Mock 9:** Hardest unmet topic from your 🔴 list.

**Week 10 (Jun 29 – Jul 5)**
- Loops should be entering final / onsite rounds.
- 2 mocks per week now; full-loop simulation if possible (1 day, 4 rounds back to back, with a peer).
- Refresh comp research; list your asking band, walk-away, and best-case for each company.
- **Mock 10 + 11.**

**Week 11 (Jul 6 – Jul 12)**
- Continue loops; expect first offers.
- Negotiation drills: rehearse the 3 hardest moments out loud (anchoring, equity question, competing offer reveal).
- Tighten tracker: what's verbal vs written, what's the deadline, what's the gap to your target.
- **Mock 12.**

**Week 12 (Jul 13 – Jul 19)**
- Multi-offer phase. Use written competing offers as leverage (politely, factually).
- Final references; ask each company for 30-min "team coffee" with future teammates if not already done.
- Decide. Sign. Notice period plan.

**Week 13 (Jul 20 – Jul 26)**
- Buffer week. If timeline slipped, this is recovery; if on time, used for transition prep.
- Write a 30/60/90 plan for the new role (impresses hiring managers if presented during final).

### Phase 3 deliverables
- 12+ mocks total.
- 2-3 written offers; 1 signed.
- Notice period at current employer initiated.
- Updated `OPPORTUNITIES.md` with outcomes; archive closed loops.

---

## 5. Daily cadence (study days)

A typical study day (3-4 productive hours; you have a job, this is realistic):

- **Morning block (90 min)** — reading or coding on portfolio project.
- **Afternoon block (45 min)** — STAR drill or AI-depth Q&A drill, out loud.
- **Evening block (60-90 min)** — apply / outreach / interview / mock.

A typical interview day:
- Block the 2h before for prep + warmup; nothing else on calendar.
- Block 30 min after for notes.
- No more than 2 rounds back-to-back; quality drops fast.

---

## 6. Weekly cadence (recurring rituals)

- **Monday** — week plan: 3 outcomes for the week. Update tracker.
- **Tuesday-Thursday** — execution.
- **Friday** — 1 mock (recorded), 1 STAR drill, 1 JD-vs-curriculum-map gap analysis.
- **Saturday** — portfolio project work block.
- **Sunday** — week review (what shipped, what slipped, what to change). Bi-weekly: resume + LinkedIn pass.

---

## 7. Tracker structure

A single `OPPORTUNITIES.md` (already exists) plus a per-company micro-doc when active:

```
- Company: <name>
- Role: <title>
- Compensation signal: <band>
- Source: <inbound / outbound / referral>
- Stage: <screening / first / system design / final / offer>
- Last touch: <date>
- Next action: <what / when>
- People: <names + roles>
- Notes: <key signals; what they care about; recurring asks>
```

Review every Sunday. Stale > 14 days = chase or close.

---

## 8. Time budgets — what to cut if behind

If you fall behind (you will), cut in this order:
1. **Project polish** — ship at v1 quality. README + working demo > shiny code.
2. **Reading depth** — skim chapters; do the self-check; move on.
3. **Fine-tuning hands-on** (Phase 3) — optional differentiator; cut if loop calendar is full.
4. **Mock quantity** — drop one per week to make room for real loops.

Do not cut:
- Real interview loops (highest signal).
- Mock self-review (the loop's payoff).
- STAR rehearsal out loud (interview-day muscle).
- Outreach (top of funnel).

---

## 9. Success metrics — what to track

Weekly:
- Applications submitted.
- Outreaches sent.
- Loops active by stage.
- Mocks completed.
- Hours in deep-work blocks (target: 12+/week).

Monthly:
- 🔴 → 🟡 → 🟢 transitions on the curriculum map self-assessment.
- Conversion: applications → screens → first round → final → offer.
- Comp signals collected vs target band.

Day 90 outcome:
- 2-3 offers, 1 signed at ≥ ₹50 LPA fixed (or equivalent USD remote).
- For Giga / 1 Cr+ band: confirmed pipeline, even if not closed by day 90 — the plan continues.

---

## 10. Risk register

| Risk | Mitigation |
|---|---|
| Burnout | Hard stop on weekend evenings; one full off-day per week |
| Loop drought | Continuous outbound — 5 founders / week; reactivate dormant warm intros |
| Single-company over-investment | Always ≥3 active loops; never negotiate solo |
| Notice period blocking start | Negotiate joining date + buyout up front |
| Tech surface evolution | Read 1 long-form post / week (Anthropic, OpenAI, Latent Space, Simon Willison, swyx, Eugene Yan) |
| Family / health blocker | Buffer week (week 13) baked in; absorb 1-2 lost weeks without panic |

---

## 11. Self-check
- Do you have a written week plan for the next week?
- Do you know your top 12 companies and stage in each?
- Did you record + review at least 1 mock this week?
- Did you rehearse 3 STAR stories out loud this week?
- Do you have 1 portfolio artifact you'd link in your next message?
- Do you know your asking band cold?

---

## 12. End state — what "ready" looks like at day 90

- A senior IC offer at an AI-premium startup at ≥ ₹50 LPA fixed (target) or top-of-band at chosen tier.
- A portfolio project that demonstrates production thinking, public-readable.
- A network of ~30 founders / hiring managers who know your name + work.
- A negotiation playbook you've used; comfortable refusing first offers.
- A repeatable cadence so the next career inflection is months, not years, of work.

---

That's the plan. Execute it. Update it weekly. Don't perfect it.
