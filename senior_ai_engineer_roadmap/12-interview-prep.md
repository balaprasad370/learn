# Interview Prep Playbook

> System design framework, behavioral STAR drills, AI-screen tactics, and comp negotiation specifically for senior IC roles at AI startups in the ₹50 LPA → ₹1 Cr band.

---

## 1. Loop structure at AI startups

Typical 4-7 round loop (varies by stage):
1. **Recruiter screen** — 30 min — fit + comp range + visa / location.
2. **Hiring manager screen** — 45-60 min — story / scope / depth probe.
3. **Coding** — 1-2 rounds — usually practical (build a small RAG, debug an agent loop) more than LeetCode at senior level.
4. **System design** — 1-2 rounds — AI-flavored (one of the 7 canonical prompts in `02-system-design-for-ai.md`).
5. **AI / domain depth** — RAG, agents, voice, serving, fine-tuning depending on the team.
6. **Behavioral / leadership** — STAR; conflict, ownership, mentoring, scope-down.
7. **Founder / VP chat** — fit + selling + comp.

Smaller startups compress this to 3-4 rounds; larger ones (Razorpay, CRED) extend with bar-raisers / cross-team panels.

### Where senior gets evaluated
- **Trade-off articulation** — every choice has costs; can you name them?
- **Failure-mode imagination** — what breaks at 10×, 100× scale?
- **Scope discipline** — can you cut 50% of the design without breaking the use case?
- **Production wisdom** — eval, observability, rollout, on-call.
- **Communication** — clear, structured, succinct under pressure.

---

## 2. System design — the 7-step framework

(Same one in `02-system-design-for-ai.md`; here's the interview-tactical version.)

### Time allocation (45-min round)
1. **Clarify** — 5 min. Don't skip; this is graded.
2. **Estimate** — 3 min. Back-of-envelope numbers anchor the design.
3. **High-level diagram** — 5 min. APIs, components, data stores. Talk while drawing.
4. **Deep dive on 1-2 components** — 15 min. Where the interviewer's attention goes. Volunteer the deep-dive areas.
5. **AI-specific concerns** — 8 min. Eval, safety, drift, cost, observability — many candidates skip this; it's the senior signal.
6. **Trade-offs + alternatives** — 5 min. Name what you'd cut, what you'd add at 10×.
7. **Wrap** — 2 min. Recap key decisions + risks.

### Clarifying questions (always ask)
- Who's the user, what's the JTBD?
- Scale — RPS, concurrent users, data volume, growth.
- Latency target (P50, P95).
- Quality bar — accuracy / faithfulness / refusal.
- Multi-tenant? Multi-region? Compliance?
- Build vs buy bias? Existing infra constraints?
- What's out of scope?

Write answers on the board / shared doc. Reference them when justifying decisions.

### Mistakes that fail seniors
- Diving into solution before clarifying.
- Vague hand-waving ("we'd use vector DB"); always pick a specific one and justify.
- Forgetting eval / safety / cost / observability.
- No trade-off articulation; everything looks rosy.
- Running out of time on the deep-dive because you spent 25 min on the diagram.
- Defending a choice when the interviewer pushes back; engage with the critique, don't dig in.

### Phrases that land senior
- "The 80% case is X; for the 20% I'd add Y but only after eval shows it's worth it."
- "Cheapest design that meets the SLO is X; here's what I'd add at 10×."
- "I'd ship behind a feature flag with shadow eval first."
- "The thing I'd watch for is [specific failure mode]; signal would be [specific metric]."
- "Trade-off here is latency vs cost vs quality — I'd pick [X] because [user constraint]."

---

## 3. Practice prompts (drill these)

Run each as a 45-min mock:
1. **RAG-as-a-service** for enterprises (multi-tenant, EU data residency).
2. **Agent platform** — Lyzr-style (build, deploy, monitor agents).
3. **Voice agent at scale** — Giga / Sarvam (10k concurrent calls, <800ms turn latency).
4. **LLM gateway** — multi-provider, rate-limit, cost-attribution, caching, fallback.
5. **Prompt versioning + eval pipeline** — PromptOps end-to-end.
6. **Fine-tuning pipeline** — multi-tenant LoRA serving on shared GPU pool.
7. **Observability platform** — LangSmith / Langfuse competitor.
8. **Coding agent** — Cursor / Cline-style with sandboxed execution.
9. **Document analysis pipeline** — 100M PDFs, ingestion + retrieval + extraction.
10. **Realtime translation agent** — speech-to-speech, telephony, low latency.

For each: time yourself, record audio, replay. The replay is unpleasant — and the highest-leverage prep activity.

---

## 4. AI-depth interview prep

Common probes (have a clean 60-90 second answer for each):

### RAG depth
- "Walk me through your RAG pipeline end-to-end."
- "How do you eval RAG?" → recall + faithfulness + answer relevance + slice analysis.
- "Hybrid search — why and how?"
- "Cross-encoder vs ColBERT vs LLM rerank."
- "How would you handle a 1M-document corpus with daily updates?"

### Agents
- "When would you use multi-agent vs single-agent?"
- "How do you prevent agent loops?"
- "Tool design principles."
- "Prompt injection — direct vs indirect; mitigations."
- "HITL design for an approval-required tool."

### LLM serving
- "PagedAttention — what problem does it solve?"
- "Continuous batching vs static batching."
- "When self-host vs API; back-of-envelope break-even."
- "Multi-LoRA serving with vLLM."
- "Quantization trade-offs (FP16 vs FP8 vs INT4)."

### Voice
- "Latency budget for a sub-800ms voice agent."
- "Barge-in implementation details."
- "Cascaded vs speech-to-speech trade-offs."
- "Telephony specifics (Indian DIDs, STIR/SHAKEN, codecs)."

### Eval / Ops
- "How do you ship a new prompt without regressing prod?"
- "LLM-as-judge biases and mitigations."
- "Drift detection in production."
- "Cost attribution per tenant."

### Fine-tuning
- "When does fine-tuning beat prompt + RAG?"
- "LoRA vs QLoRA vs full FT."
- "DPO vs PPO."

Don't memorize scripts; internalize the structure so you can ad-lib confidently.

---

## 5. Behavioral / STAR drills

STAR = Situation / Task / Action / Result. Senior round signals: **scope, judgment, mentoring, conflict, recovery**.

### Stories you should have ready (3-5 each)
- **0→1 launch** — built something from scratch; what would you do differently.
- **1→10 scale** — took something working and made it production-grade.
- **Hard incident** — outage, debugged under pressure, postmortem.
- **Conflict / disagreement** — peer / manager / stakeholder; how resolved.
- **Mentoring** — leveled up a junior; specific outcome.
- **Scope-down decision** — said no to a feature; why; what happened.
- **Cross-functional** — worked with PM / design / sales / ops on something complex.
- **Failure / mistake** — owned, fixed, learned. Required for senior.
- **Tech leadership** — drove a decision across people who didn't report to you.
- **Hiring / interviewing** — bonus if you can speak to it.

### Format
- Situation: 2 sentences max.
- Task: what was specifically yours.
- Action: 5-7 specific things you did. **Use "I" not "we"** when describing your actions.
- Result: quantified. ("Cut latency P95 from 2.4s to 700ms; thumbs +18%; on-call pages -60%.")

### Common pitfalls
- Vague stories without numbers.
- "We" the whole way; can't tell what *you* did.
- No conflict / failure stories; senior needs both.
- Bringing a story without a learning.
- Talking 5 minutes without pause; offer to expand if interesting.

### Drill cadence
- Pick 8 stories. Write 1-page each. Practice out loud weekly. Record + replay.
- Tailor 2-3 to each company; mention their domain in the story (voice for Giga, agents for Lyzr, RAG for Krutrim).

---

## 6. AI startup-specific signals

### What founders / hiring managers actually want
- **Bias for action** — built things, shipped things; specific.
- **Customer obsession** — name customers, conversations, failures.
- **Fast learning** — picked up new framework / paradigm; specific.
- **Frugality** — managed cost; back-of-envelope math; not over-engineered.
- **Ownership** — on-call, postmortems, customer comms.
- **AI-current** — read the space; have opinions on recent papers / launches.
- **Output bias** — care about user outcome > internal process.
- **Scrappiness** — comfortable with ambiguity, can do work outside your job description.

### What turns them off
- "I built this elaborate framework but didn't ship to users."
- "I'd want a PRD before I start." (At a startup, you write the PRD.)
- "We had a process for that." (They don't have process; you make process.)
- Salary negotiation as the first topic of the first call.
- Vague AI literacy ("LangChain handled that for us").

### Recent-trends signals worth showing fluency in
- MCP and tool standardization (mid-2025 onwards).
- Speech-to-speech models (GPT-4o Realtime, Gemini Live, open Moshi/Sesame).
- DeepSeek-R1 / GRPO and the reasoning-model paradigm.
- Anthropic Contextual Retrieval / late-interaction.
- Agentic search (Perplexity, ChatGPT Search) and reasoning-search hybrid.
- Multi-LoRA serving and per-tenant fine-tunes.
- Inference cost compression (Sonnet < $1/M input typical for tier-1 frontier as of 2026).
- Sovereign AI in India (Sarvam, Krutrim, BharatGPT) and the policy backdrop.

Drop these naturally; don't lecture.

---

## 7. Coding round at senior level

Less LeetCode, more **practical / messy / open-ended**.

### Common formats
- **Build a small RAG**: ingest a folder of markdown, embed, retrieve, answer with citations. 60-90 min.
- **Fix this agent**: a buggy LangGraph / LangChain script that loops or returns wrong tool calls. Debug live.
- **Design + implement a tool**: function-calling tool with schema + handler + tests.
- **Eval harness**: write a small eval that scores responses against gold; talk about metrics.
- **Streaming**: implement SSE / WebSocket streaming of LLM tokens to a client.
- **Old-school DSA**: still happens, especially at larger startups; usually one round.

### Tactics
- **Talk while you code.** Name the data structures, the trade-offs, the assumptions.
- **Write small, working pieces first.** Then iterate. Don't try to write the whole thing at once.
- **Tests, even sketchy ones.** Show you think about correctness.
- **Acknowledge what you'd improve** at the end ("I'd add retries, paginate the retrieval, etc.").
- **Don't bluff syntax.** Pseudocode is fine; ask if a specific lib is OK.

### Prep
- One LangChain / LangGraph end-to-end project from scratch in 90 min.
- Time yourself building a RAG with retrieval + reranker + grounded answer + citations.
- One agent debug session — write a buggy agent, then debug as if it's not yours.
- Refresh basics: async Python, pydantic, FastAPI, SSE/WebSockets, simple SQL.

---

## 8. Compensation negotiation (₹50 LPA → ₹1 Cr)

### Indian senior IC bands at AI startups (2026)
- **Mid-tier (Series A/B Indian)**: ₹35-60 LPA fixed + 0.05-0.2% ESOP. Total ~₹50 LPA.
- **Top-tier (well-funded Indian, US-with-IN presence)**: ₹60 LPA – ₹1.2 Cr fixed + 0.1-0.4% ESOP. Total ~₹70 LPA – ₹1.5 Cr.
- **US HQ with IN remote** (Giga, Together, Modal): often $80k-$160k cash equivalent (~₹70 LPA – ₹1.4 Cr) + meaningful equity.
- **Frontier labs / OpenAI India / Google DeepMind India**: ₹1 Cr+ trivially for senior IC.

Bonuses (joining, retention) common at top tier; vary widely.

### Levers
- **Base** — hardest to move; usually band-anchored.
- **ESOP / RSU** — biggest upside / variance; understand vest cliff, exercise window, fair-market price.
- **Joining bonus** — flexible if base is band-capped.
- **Notice-period buyout** — non-trivial; ₹5-15L for senior at current employer.
- **Leveling** — sometimes "Senior" → "Staff" yields more than negotiating within a level.
- **Remote / location** — Bangalore band may differ from full remote band.
- **Working hours / on-call** — for US-HQ companies, push back on hard US-hours expectation.

### Process
1. **Don't anchor first.** "I'm exploring opportunities at the senior level; what's the band for this role?" If pushed: "I'd want to align on scope before anchoring; my current TC is ₹X."
2. **Get 2-3 competing offers** before signing. Single-offer negotiation has 30% the leverage.
3. **Negotiate in writing**, after verbal alignment. Use email; recap the agreed numbers; ask for the official letter.
4. **Specifics on equity**: # of shares, total shares outstanding (so you compute %), strike price, vesting schedule, cliff, post-termination exercise window, last 409A / fair-market valuation (US) or recent funding round (IN), liquidation preferences (1×, multiple, participating?).
5. **Don't accept ESOP without strike + last valuation**. Without those numbers, the equity is a story, not a number.
6. **Joining date flexibility** is leverage; faster start = some bonus room.
7. **Reference offer** — "Company X offered ₹Y; if you can match base + close gap on ESOP, I'm in." Honest, specific, time-boxed.

### Red flags
- Company can't show recent valuation / cap table summary.
- ESOP grants in absolute number with no total share count (so you can't compute %).
- Refusal to put offer in writing.
- "We don't negotiate" — sometimes true at series-D+, often a tactic earlier.
- Rapid pressure to sign in <24h with no time to think.

---

## 9. Companies-specific tactical notes

### Lyzr
- Agent platform; depth on agent runtime, multi-tenant, eval, MCP.
- Read their docs; they ship publicly. Reference specifics.
- Indian + US presence; ESOP in USD-denominated entity.

### Sarvam
- India-first multilingual LLMs + voice; depth on Indic NLP, voice pipeline, on-device.
- Sovereign AI angle; policy fluency helps.
- Currently well-funded; senior IC band competitive.

### Krutrim (Ola)
- Compute + foundation models. Depth on training infra, GPU economics.
- Bigger company structure; more bar-raiser style rounds.

### Composio
- Tool catalog / MCP-adjacent. Depth on MCP, OAuth flows, function-calling reliability.
- Smaller, ICs ship a lot.

### Giga
- Voice agent for ops (DoorDash etc.). Depth on voice latency, telephony, live agent design.
- US HQ, India hires; expect comp at top of band; English fluency + late-evening overlap mentioned.

### Together / Modal / Replicate / Hugging Face
- Infra-tier. Depth on serving, GPU economics, distributed systems, performance tuning.
- US HQ, IN remote; dollar-denominated; high bar but high pay.

### Razorpay / CRED / Postman / Zepto / Groww
- Cross-domain unicorns hiring AI seniors into existing product orgs.
- Depth on integration with existing systems, scale, compliance (RBI for fintech, PCI), pragmatism.
- Bigger interview loops; better cash but lower equity upside vs early-stage.

---

## 10. Pre-interview checklist (for each loop)

24h before:
- Re-read the JD; underline the 3 things they care most about; prepare a 90-sec "why I fit" hook.
- Check team via LinkedIn — who'll interview you, what they've built, recent posts.
- Check funding + recent product launches; have 1 question per round about something specific.
- Pick 2 STAR stories per round that you'll lead with.
- Mock the system-design prompt most likely for them (RAG / agent / voice).

Day of:
- 5-min pre-call: water, tea, bathroom, no Slack, no email open.
- Notebook / whiteboard tool open and ready.
- Audio + camera tested.
- Calendar invite has the right link; backup phone number ready.

After each round:
- Write down what they asked, what you said, what you wish you'd said.
- Send a thank-you note within 24h with one substantive add ("Thinking more about your question on barge-in — the cancellation token approach I mentioned would also let us...").
- Update interview tracker with status, comp signal, next steps.

---

## 11. Mock interview cadence (90-day plan)

Weekly:
- 1 system-design mock out loud (recorded, self-reviewed, or with peer).
- 2 STAR drills out loud.
- 1 AI-depth Q&A drill.

Bi-weekly:
- Coding mock (real-time, 60-90 min).
- Resume + LinkedIn pass.

Monthly:
- 1 full loop simulation with a friend playing all rounds.
- Compensation research refresh (Glassdoor, LinkedIn, Levels.fyi for US HQ, founder-friend signals for IN startups).

This is volume work — the 10th mock is dramatically better than the 1st. Senior IC offers come from rep + clarity, not from memorizing more facts.

---

## 12. Self-check
- Can you run the 7-step system design framework in 45 min cold?
- Do you have 8 STAR stories with quantified results, written and rehearsed?
- Can you name your asking band (base + equity + bonus) without flinching?
- Do you have 3 competing-offer scenarios mapped out (best, base, walk-away)?
- Can you name 5 recent AI trends + your take on each?
- Do you have a tracker of every conversation, status, and next step?

---

**Next:** `13-90-day-plan.md` — sequenced study + practice plan, weekly cadence, milestone gates.
