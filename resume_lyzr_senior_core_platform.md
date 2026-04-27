# [YOUR NAME]

**Senior Software Engineer — AI Platforms, Agent Runtimes, Cloud Infrastructure**

Bengaluru, India · info@meduniverse.app · [+91-XXXXXXXXXX] · [linkedin.com/in/yourname] · [github.com/yourhandle] · [meduniverse.app]

---

## Summary

Senior engineer with **5 years** building production systems end-to-end — from 0→1 launch to 1→10 scale. Deep stack in **Python/FastAPI backend, React/Next.js frontend, AWS, Docker**, with hands-on experience designing **LLM agent runtimes, RAG pipelines, and multi-agent orchestration**. Shipped `meduniverse.app` as founding engineer. Comfortable owning the full surface: architecture, APIs, storage, observability, CI/CD, and on-call. Looking for a senior IC role on a core platform team pushing agentic AI into enterprise production.

---

## Core Skills

**Languages:** Python (expert), TypeScript/JavaScript, SQL, Bash
**Backend:** FastAPI, Django, async Python, REST, GraphQL, WebSockets, SSE, queue-driven microservices
**AI / LLM:** Agent design (ReAct, Plan-Execute, multi-agent handoff), RAG pipelines (chunking, embedding, hybrid retrieval, reranking), tool/function calling, prompt engineering, prompt caching, structured outputs, evaluation harnesses, OpenAI / Anthropic / Gemini / local model serving, Llama / vLLM
**Data & Storage:** PostgreSQL, MongoDB, Redis (Streams, Pub/Sub, caching), pgvector, Pinecone/Qdrant, S3
**Frontend:** React, Next.js, TypeScript, Tailwind, streaming UIs
**Cloud & DevOps:** AWS (EC2, ECS, Lambda, RDS, SQS, S3, CloudWatch, IAM), Docker, Kubernetes, GitHub Actions, Terraform, Linux, Nginx
**Observability:** OpenTelemetry, Prometheus, Grafana, structured logging, trace-first debugging
**Practices:** System design, API contracts, event-driven architecture, idempotency, multi-tenancy, security by default, budget/cost controls

---

## Experience

### Senior Software Engineer — [Current Company]
*[Month YYYY] – Present · Bengaluru, India*

- Led backend design and delivery for **[product/platform]** serving **[X] users / [Y] requests/day**, built on **FastAPI + Postgres + Redis on AWS** with Docker deploys.
- Designed and shipped an **agent orchestration layer** integrating **OpenAI / Anthropic** with tool calling, streaming responses (SSE), and per-run budget enforcement — cut user-facing latency by **[N]%** and LLM spend by **[N]%** via prompt caching and model routing.
- Built a **RAG pipeline** (chunking → embeddings via [model] → pgvector → hybrid retrieval with BM25 + vector + RRF fusion → reranker) serving **[X]** daily queries at **p95 < [N]ms**.
- Owned **multi-tenant isolation** (tenant-scoped data, rate limits, per-tenant budget caps) and **observability** (OpenTelemetry traces for every agent step, cost attribution per request).
- Introduced **durable queue-based run execution** (Redis Streams, consumer groups, XPENDING reclaim) so long agent workflows survive pod restarts and resume from checkpoints.
- Partnered with frontend on **streaming UIs** (SSE + React) for token-level chat and live tool-call visualization.
- Set up **CI/CD** with GitHub Actions, built **Docker** images signed with cosign, shipped to AWS ECS with blue/green deploys; rolled out expand/contract schema migrations on Postgres without downtime.

### Software Engineer — [Prior Company]
*[Month YYYY] – [Month YYYY] · Bengaluru / Remote*

- Shipped **[N]** production features across backend (FastAPI / Django) and frontend (React / Next.js), owning design → code → deploy → on-call.
- Designed an **event-driven ingestion pipeline** (SQS + Lambda + S3) handling **[volume]** daily records with idempotent processing and dead-letter recovery.
- Built **REST and WebSocket APIs** for real-time features; introduced **Redis**-backed caching layer that dropped p95 response time by **[N]%**.
- Migrated a monolithic service to **containerized microservices on ECS**, reducing deploy time from **[N]** min to **[N]** min and enabling independent team scaling.
- Mentored **[N]** junior engineers through design reviews, pair programming, and code review; authored internal docs that became team onboarding standard.

### Software Engineer — [Earlier Role / First Job]
*[Month YYYY] – [Month YYYY]*

- Delivered full-stack features on **Python + React** across **[N]** sprints; consistently shipped ahead of schedule with clean code review feedback.
- Built **[specific subsystem]** that [specific measurable outcome].
- Picked up **cloud (AWS) and DevOps (Docker, CI/CD)** on the job; volunteered for infra work that nobody else wanted.

---

## Selected Projects

### meduniverse.app — Founding Engineer
*[YYYY] – Present*

0→1 healthcare platform I architected and launched single-handedly.
- **Stack:** Python/FastAPI backend, Next.js/React frontend, Postgres, Redis, AWS (ECS, RDS, S3, CloudFront), Docker, GitHub Actions.
- **AI integration:** RAG pipeline over medical reference content; agent-based triage assistant using tool calling (lookup, scheduling, escalation); prompt caching keeping per-session cost under **$[N]**.
- **Scale:** **[N]** users, **[N]** daily sessions, **p95 < [N]ms** on core endpoints.
- **Ownership:** infra, security (HIPAA-aligned patterns, encrypted-at-rest, least-privilege IAM), billing, observability (OpenTelemetry, Grafana dashboards), on-call rotation.
- Repo / case study: **[meduniverse.app]** · Architecture writeup: **[link]**

### Agent Runtime Reference Architecture — Personal Project
*[YYYY]*

15-document deep-dive design of a production agent runtime (planning/execution loop, memory systems, tool calling, multi-agent handoff, observability, safety, evaluation, deployment). Drew on production experience to spec: state-machine-driven runs, atomic step checkpointing, durable queues, budget-as-input safety controls, 7-hook policy engine, 5-tier evaluation structure.
- **Link:** [github.com/yourhandle/agent-runtime-spec]

### [Optional third project — OSS contribution, hackathon win, or side build]

---

## Open Source & Writing

- **[Repo 1]** — [brief description, e.g., "Python library for X, N+ stars"]
- **[Repo 2]** — [brief description]
- Blog: **[blog url]** — writing on LLM internals, agent architecture, Python infra
- LLM Fundamentals notes: open-source learning resource covering transformers, tokenization, inference optimization, model evolution — **[link]**

---

## Education

**[Degree]**, [College/University] · [YYYY] – [YYYY]
[Optional: GPA if strong, relevant coursework, honors]

---

## Additional

- **Languages:** English (fluent), [Hindi / Telugu / Tamil / Kannada / other].
- **Availability:** Open to Bengaluru on-site; immediate / **[N]** weeks notice.
- **What I'm looking for:** Senior IC role on a platform team building agentic AI for production. Interested in: runtime internals, LLM serving, multi-tenancy, observability, developer experience.

---

## How to read this resume against Lyzr's Senior Core Platform Engineer JD

*(Delete this section before sending — it's a self-check, not part of the resume.)*

JD hits:
- **Python + FastAPI** → Experience §1, §2; meduniverse.app
- **MongoDB** → add to Data & Storage if you've used it; otherwise note Postgres + "comfortable picking up MongoDB"
- **AI Agent + RAG** → Experience §1, meduniverse.app, Agent Runtime spec project
- **AWS + Docker** → Experience §1, §2, meduniverse.app
- **5–8 years** → 5 years, right at the entry threshold — every bullet must reinforce "senior"
- **Core platform ownership** → framed in §1 as "owned multi-tenancy, observability, durable runs" — exactly core-platform language

Things to add before submitting:
1. Fill every **[bracketed]** placeholder.
2. Pin real metrics (users, RPS, latency, $ saved, % improved). If you don't have exact numbers, use honest order-of-magnitude estimates — "~10k daily sessions" beats "[N] sessions."
3. If you've touched MongoDB in any project, surface it.
4. Tie one bullet specifically to an agent runtime or LLM platform problem you debugged in production — recruiters at agent companies screen for this.
5. Keep it to **one page** if possible, two at most. Cut the weakest bullet in §2/§3 if you run long.
6. Export to **PDF** for application; keep this `.md` as source.

---

## Cover-letter kernel (optional, for the application textbox)

> I'm a 5-YoE senior engineer based in Bengaluru, currently building meduniverse.app end-to-end on FastAPI + AWS with an LLM agent layer for triage. Lyzr's position on agentic AI for enterprise is where I want my next chapter to be — I've spent the last year going deep on agent runtime internals (planning loops, memory, tool calling, multi-agent handoff, evaluation) and want to ship that at Lyzr's scale and stage. Attaching my resume for the Senior Core Platform Engineer role. Happy to walk through a system-design of an agent runtime on a first call.
>
> — [Your Name]
