# Senior AI Engineer — Eligibility Roadmap

> Everything a senior IC at an AI-premium startup (Lyzr, Sarvam, Krutrim, Composio, Giga, Zep, Together, etc.) is expected to know — organized as **Main → Sub → Detailed → Relations** so you can see the whole shape and trace dependencies.

Goal: be hireable at **₹50 LPA – ₹1 Cr+** for a **senior IC role** at a top-tier AI startup. That means clearing:
- **System design** (AI-flavored: agents, RAG, voice, multi-tenant LLM platform).
- **AI internals** (transformers, serving, evaluation, agent runtimes).
- **Production engineering** (distributed systems, observability, safety, cost).
- **Senior signals** (ownership, mentoring, design docs, on-call, 0→1 + 1→10).

---

## What you already have (don't re-read; just reference)

| Folder | What it covers |
|---|---|
| [`../llm_fundamentals/`](../llm_fundamentals/) | Transformers, tokenization, inference, training, model evolution, token economics |
| [`../rag_systems/`](../rag_systems/) | 11-doc RAG curriculum (foundations → production playbook) |
| [`../lyzr_agent_runtime/`](../lyzr_agent_runtime/) | 15-doc agent runtime architecture (planner, executor, memory, tools, safety, eval, deployment) |

These cover the **AI internals** and **agent runtime** dimensions. This folder fills the rest.

---

## What this folder adds

| Doc | Topic | Why it matters at senior level |
|---|---|---|
| [`00-curriculum-map.md`](./00-curriculum-map.md) | **Master map** — Main → Sub → Detailed → Relations for every topic | The shape of the whole job. Read first. |
| `01-distributed-systems.md` | Distributed systems refresher | Foundation for every system-design round |
| `02-system-design-for-ai.md` | AI-flavored system design (agent platform, RAG-as-a-service, voice at scale) | The actual interview format at AI startups |
| `03-llm-serving-infra.md` | vLLM, SGLang, TGI, batching, KV-cache, GPU economics | Asked at any infra-tier role (Sarvam, Krutrim, Giga) |
| `04-multi-agent-orchestration.md` | LangGraph, CrewAI, AutoGen, Swarm, handoff patterns | Lyzr/Composio surface area |
| `05-mcp-and-tool-protocols.md` | Model Context Protocol, function calling, tool design | Becoming the standard; competitive edge |
| `06-voice-agents.md` | STT, TTS, real-time, telephony, barge-in, latency | Giga + Sarvam Voice direct hit |
| `07-evaluation-engineering.md` | Beyond RAG eval — agent trajectory, golden sets, eval harnesses | Quality-obsessed startups grill on this |
| `08-llmops-and-lifecycle.md` | Versioning, A/B, shadow, canary, drift detection | Differentiates senior from mid |
| `09-safety-and-guardrails.md` | Prompt injection, jailbreaking, NeMo, Llama Guard | Required for enterprise-facing AI |
| `10-observability-and-cost.md` | LangSmith, Langfuse, Phoenix, OpenLLMetry, FinOps | Senior IC owns these |
| `11-fine-tuning-handson.md` | SFT, LoRA, DPO, when to fine-tune vs RAG | Differentiator if asked |
| `12-interview-prep.md` | System design + behavioral + comp negotiation playbook | Convert prep into offers |
| `13-90-day-plan.md` | Sequenced study + practice plan | Execution layer |

---

## How to use

1. **Read `00-curriculum-map.md` end-to-end first.** It's the index of every concept and the map of how they relate. ~30 min read.
2. **Self-assess** — for each leaf topic in the map, mark yourself: 🟢 confident, 🟡 vague, 🔴 don't know.
3. **Prioritize 🔴 → 🟡** — those become study sessions. The other docs in this folder cover most 🔴 topics in depth.
4. **Run `13-90-day-plan.md`** as your operating cadence: weekly reading + weekly system-design practice + weekly mock + bi-weekly resume update.
5. **Apply pressure-tests**: every Friday, do one mock system-design out loud (record it), one behavioral STAR drill, and review one company JD against the map to find gaps.

---

## Target outcomes after 90 days

- Pass **system-design interview** at AI startups (you can design: multi-tenant agent platform, voice agent at scale, RAG-as-a-service, LLM gateway).
- Talk fluently about **LLM serving** internals (vLLM internals, KV cache, continuous batching, LoRA serving).
- Ship **multi-agent orchestration** end-to-end (LangGraph or equivalent) with eval + observability.
- Demonstrate **production AI engineering**: prompt versioning, drift detection, cost attribution, safety layer.
- Strong **behavioral signal** — STAR stories ready for: 0→1 launch, hard incident, mentoring, scope-down decision, conflict.
- Comp negotiation playbook ready for **₹50 LPA – ₹1 Cr** band.

---

## Companies this roadmap targets

(See [`../OPPORTUNITIES.md`](../OPPORTUNITIES.md) for the live list.)

- **Indian AI-premium:** Lyzr, Sarvam, Krutrim, Composio, Neysa, Simplismart, Ema, Gupshup, Haptik, Observe.AI.
- **US-with-India presence:** Giga (1 Cr+ band confirmed), Together, Modal, Replicate, Hugging Face, LangChain, Inngest.
- **Cross-domain Indian unicorns hiring senior + AI:** Razorpay, CRED, Postman, Zepto, Groww.
