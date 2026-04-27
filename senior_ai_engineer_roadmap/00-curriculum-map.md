# 00 — Master Curriculum Map

> Every concept a senior IC at an AI startup is expected to know. Format: **Main Area → Sub-Area → Detailed Topic → Relations** (other topics it depends on, enables, or trades off against).
>
> Read once end-to-end to see the whole shape. Then mark each leaf 🟢/🟡/🔴 and prioritize.

---

## Reading the Relations notation

- `→ X.Y` — depends on or pairs with topic X.Y in this map.
- `→ folder/file` — covered in another folder you already have (or another doc in this one).
- `vs Y` — explicit trade-off / alternative to Y.
- `feeds Z` — this concept enables or unlocks Z.

---

# AREA A — SOFTWARE FOUNDATIONS

> The non-AI baseline. Senior bar is "you obviously know this," not "you're learning."

## A.1 Distributed systems

### A.1.1 Consistency models
- Strong, linearizable, sequential, causal, eventual.
- Read-your-writes, monotonic reads, monotonic writes.
- **Relations:** `→ A.4` (storage choices encode consistency), `→ C.2.4` (agent state durability), `vs` AP/CP trade-offs in CAP.

### A.1.2 CAP, PACELC, FLP
- CAP: pick 2 of {Consistency, Availability, Partition tolerance} under partition.
- PACELC adds latency-vs-consistency under no-partition.
- FLP impossibility (no deterministic consensus in async + 1 failure).
- **Relations:** `→ A.1.1` consistency, `→ A.1.4` consensus algorithms.

### A.1.3 Replication
- Leader/follower, multi-leader, leaderless (Dynamo).
- Sync vs async replication; quorum reads/writes (R + W > N).
- Replication lag, failover, split-brain.
- **Relations:** `→ A.4` storage, `→ A.1.5` partitioning often paired.

### A.1.4 Consensus
- Raft, Paxos (you don't implement; you know shape and use cases).
- Used in: etcd, Consul, ZooKeeper, Kafka KRaft.
- **Relations:** `→ A.1.3` replication, `→ C.2.4` agent run leader election.

### A.1.5 Partitioning / sharding
- Hash, range, directory; consistent hashing; resharding pain.
- **Relations:** `→ A.4` storage, `→ B.4.5` vector store sharding (`rag_systems/04`).

### A.1.6 Idempotency, exactly-once, at-least-once
- Idempotency keys, dedupe windows, transactional outbox.
- **Relations:** `→ A.5.2` queues, `→ C.2.5` agent step retries (`lyzr_agent_runtime/`).

### A.1.7 Failure modes
- Network partitions, GC pauses, slow nodes, gray failures, cascading failures.
- Backpressure, circuit breakers, bulkheads, timeouts, retries with jitter.
- **Relations:** `→ A.7` reliability, `→ E.3` cost (cascades blow budgets).

## A.2 Networking & protocols

### A.2.1 HTTP/1.1, HTTP/2, HTTP/3
- Multiplexing, head-of-line blocking, server push.
- Keep-alive, connection pooling.
- **Relations:** `→ A.2.2` SSE, `→ C.5.4` voice transports.

### A.2.2 SSE, WebSocket, gRPC streaming
- SSE: server→client only, HTTP-native, simple.
- WebSocket: bidi, low overhead.
- gRPC streaming: typed, multiplexed over HTTP/2.
- **Relations:** `→ C.4.6` agent streaming, `→ C.5.3` voice realtime.

### A.2.3 WebRTC
- DataChannel, audio/video tracks, SDP, ICE/STUN/TURN.
- Used by: LiveKit, Pipecat for voice agents.
- **Relations:** `→ C.5` voice agents.

### A.2.4 TLS, mTLS, JWT, OAuth2, OIDC
- Token lifetimes, refresh, key rotation, JWKS.
- **Relations:** `→ A.7.2` security, `→ C.6.3` MCP authentication.

## A.3 OS & runtime

### A.3.1 Concurrency models
- Threads, processes, async (Python asyncio, Node event loop, Go goroutines).
- Race conditions, locks, deadlocks; CSP vs shared-memory.
- **Relations:** `→ A.5.2` queues, `→ B.3.4` LLM serving (continuous batching needs deep async).

### A.3.2 Memory & GC
- Heap vs stack, GC pauses, off-heap (mmap, Arrow), zero-copy.
- **Relations:** `→ B.3.3` KV cache memory, `→ B.4.3` vector store RAM.

### A.3.3 Linux fundamentals
- File descriptors, sockets, /proc, cgroups, ulimits.
- top, htop, iostat, vmstat, strace, perf, lsof.
- **Relations:** `→ A.7.4` debugging incidents.

## A.4 Storage

### A.4.1 Relational (Postgres)
- Indexes (B-tree, GIN, GiST), explain analyze, query planning.
- Transactions, isolation levels, MVCC.
- Extensions: pgvector, pg_trgm, pg_partman.
- **Relations:** `→ B.4` vector stores (`rag_systems/04`), `→ A.4.5` migrations.

### A.4.2 NoSQL
- KV (Redis, DynamoDB), document (MongoDB), wide-column (Cassandra), graph (Neo4j).
- When each wins; consistency models per system.
- **Relations:** `→ A.1.1`, `→ B.4` vector DBs (some are NoSQL-native).

### A.4.3 Search engines
- Elasticsearch, OpenSearch, Vespa.
- BM25 + dense fusion (`rag_systems/05`).
- **Relations:** `→ B.4`, `→ B.5` retrieval.

### A.4.4 Object storage
- S3 / GCS / R2. Strong-after-write consistency, lifecycle, signed URLs.
- **Relations:** `→ B.2` ingestion (`rag_systems/02`), `→ E.3` cost.

### A.4.5 Schema migrations
- Expand-contract, online schema change, gh-ost / pg_repack.
- **Relations:** `→ A.7.5` zero-downtime deploys.

## A.5 Caching, queues, events

### A.5.1 Caching
- Cache-aside, read-through, write-through, write-behind.
- Eviction (LRU, LFU, TTL); consistency hazards (cache stampede, dog-piling).
- Levels: CDN → app → DB; Redis as a layer.
- **Relations:** `→ B.6.5` LLM response cache (`rag_systems/06`), `→ E.3` cost.

### A.5.2 Queues / streams
- SQS, RabbitMQ (queue); Kafka, Redis Streams, NATS JetStream (log/stream).
- Consumer groups, offsets, replay, DLQ.
- **Relations:** `→ A.1.6` idempotency, `→ C.2.4` durable agent runs.

### A.5.3 Event-driven architecture
- Event sourcing, CQRS, change data capture (CDC).
- Outbox pattern for transactional → event reliability.
- **Relations:** `→ A.5.2`, `→ C.4.5` agent event sinks.

## A.6 APIs & contracts

### A.6.1 REST design
- Resource modeling, idempotent verbs, pagination, error catalogs (RFC 7807).
- **Relations:** `→ A.6.4` versioning.

### A.6.2 gRPC / Protobuf
- Schema-first, codegen, streaming RPCs.
- **Relations:** `→ A.2.2`.

### A.6.3 GraphQL
- N+1, dataloader, persisted queries.

### A.6.4 Versioning
- URI versioning, header, content negotiation; deprecation policies.
- **Relations:** `→ C.6.4` MCP versioning, `→ D.2` model A/B.

## A.7 Reliability, security, ops

### A.7.1 SLO / SLI / error budgets
- Define availability, latency, freshness; budget-based release decisions.
- **Relations:** `→ D.4` LLM observability.

### A.7.2 AppSec basics
- OWASP top 10, secrets management, least-privilege IAM, audit logs.
- Rate limiting (token bucket, leaky bucket), DDoS basics.
- **Relations:** `→ C.7` AI safety/guardrails.

### A.7.3 Multi-tenancy
- Row-level isolation, namespace, dedicated cluster; noisy neighbor.
- Per-tenant quotas, budgets, cost attribution.
- **Relations:** `→ D.5` cost engineering, `→ B.4.6` vector store tenancy.

### A.7.4 Incident response
- Runbooks, postmortems, blameless culture.
- Pager hygiene, on-call rotation.
- **Relations:** `→ A.7.1` SLOs.

### A.7.5 Deploy strategies
- Blue/green, canary, feature flags, dark launch.
- **Relations:** `→ D.2` model rollouts.

## A.8 Containers & orchestration

### A.8.1 Docker
- Multi-stage builds, layer caching, distroless, image signing (cosign).
- **Relations:** `→ A.8.2` k8s.

### A.8.2 Kubernetes
- Pods, Deployments, StatefulSets, Services, Ingress.
- HPA, KEDA, custom metrics.
- **Relations:** `→ B.3.5` LLM autoscaling (`lyzr_agent_runtime/13`).

### A.8.3 IaC
- Terraform, Pulumi, CDK; state management; drift detection.
- **Relations:** `→ A.7.5`.

### A.8.4 Cloud (AWS / GCP)
- IAM, VPC, managed DBs, S3/GCS, Lambda/Cloud Run, GPU instances.
- **Relations:** `→ B.3.6` GPU economics.

---

# AREA B — AI INTERNALS (already covered, listed for completeness)

## B.1 Transformer architecture
- Attention, multi-head, FFN, MoE, RMSNorm, SwiGLU.
- → [`llm_fundamentals/01`](../llm_fundamentals/01-transformer-architecture.md)

## B.2 Tokenization & generation
- BPE, byte-level, SentencePiece; prefill/decode; sampling.
- → [`llm_fundamentals/02`](../llm_fundamentals/02-tokenization.md), [`/03`](../llm_fundamentals/03-tokens-to-output.md)

## B.3 LLM serving (deepens in this folder)

### B.3.1 Inference math
- KV cache size = `2 · n_layers · n_heads · d_head · seq_len · batch · bytes_per_param`.
- Memory-bound decode vs compute-bound prefill.
- **Relations:** `→ B.3.2`–`B.3.6`.

### B.3.2 Continuous batching
- Iteration-level scheduling; vLLM, TGI, SGLang.
- **Relations:** `→ A.3.1` async, `→ 03-llm-serving-infra.md`.

### B.3.3 PagedAttention / KV-cache management
- Block-table virtualization of KV cache (vLLM core).
- **Relations:** `→ B.3.1` math, `→ A.3.2` memory.

### B.3.4 Quantization
- FP16, BF16, FP8, INT8, INT4 (AWQ, GPTQ); accuracy trade-offs.
- **Relations:** `→ B.4.5` vector quantization (`rag_systems/04`).

### B.3.5 LoRA serving
- Multi-LoRA in one engine; hot-swap adapters.
- **Relations:** `→ E.1` fine-tuning.

### B.3.6 GPU economics
- A10G / L4 / L40S / H100 / H200 / B200 — compute, HBM, $/hr.
- Throughput per $ for various model sizes.
- **Relations:** `→ D.5` cost.

### B.3.7 Speculative decoding, prefix sharing
- Draft model + verify; cross-request prefix sharing.
- **Relations:** `→ B.6` prompt caching.

## B.4 RAG

> Full coverage in [`rag_systems/`](../rag_systems/) (11 docs).

### B.4.1 Ingestion + chunking → `rag_systems/02`
### B.4.2 Embeddings → `rag_systems/03`
### B.4.3 Vector stores (HNSW, IVF, PQ, DiskANN) → `rag_systems/04`
### B.4.4 Retrieval (BM25, hybrid, RRF, HyDE) → `rag_systems/05`
### B.4.5 Reranking → `rag_systems/06`
### B.4.6 Multi-tenancy in vector stores → `rag_systems/04`, `A.7.3`
### B.4.7 Evaluation (RAGAS, slice analysis) → `rag_systems/08`
### B.4.8 Advanced (Contextual, GraphRAG, RAPTOR, ColPali) → `rag_systems/09`

## B.5 Agent runtime

> Full coverage in [`lyzr_agent_runtime/`](../lyzr_agent_runtime/) (15 docs).

### B.5.1 Planner / executor / loop control
### B.5.2 Memory (short-term, long-term, episodic)
### B.5.3 Tool calling
### B.5.4 Multi-agent handoff
### B.5.5 Safety hooks (7-hook policy engine)
### B.5.6 Evaluation (5-tier)
### B.5.7 Deployment / scaling / extension points

## B.6 Token economics & prompt caching
- Anthropic / OpenAI / Gemini cache mechanics.
- → [`llm_fundamentals/08`](../llm_fundamentals/08-token-economics.md)

---

# AREA C — AI PRODUCTION SURFACE

## C.1 LLM serving infrastructure (deep)

### C.1.1 Engines
- vLLM (PagedAttention reference), SGLang (RadixAttention prefix sharing), TGI, TensorRT-LLM, llama.cpp.
- When each wins (vLLM = throughput; SGLang = prefix-heavy; TRT-LLM = NVIDIA-only peak; llama.cpp = CPU/edge).
- **Relations:** `→ B.3`, `→ 03-llm-serving-infra.md`.

### C.1.2 Serving topology
- Single-tenant pod vs shared engine pool.
- Tensor parallelism (split layers), pipeline parallelism (split stages), data parallelism (replicate).
- **Relations:** `→ A.8.2` k8s, `→ A.1.5` partitioning.

### C.1.3 Autoscaling
- KEDA on queue depth or in-flight requests.
- Cold start: 30s–5min for big models — predictive vs reactive.
- **Relations:** `→ A.8.2`, `→ D.5` cost.

### C.1.4 Routing & fallback
- LLM gateway: model router, failover, retry policies, rate limiting per provider.
- LiteLLM, Portkey, OpenRouter, custom.
- **Relations:** `→ D.5` cost routing, `→ A.7.1` SLOs.

### C.1.5 Hosted inference platforms
- Together, Fireworks, Anyscale, Modal, Replicate.
- Pricing models: per-token, per-second, dedicated endpoint.
- **Relations:** `→ D.5`, `→ E.1.5` fine-tune deploy.

## C.2 Agent platforms & orchestration

### C.2.1 LangGraph
- DAG/cyclic graph of nodes (LLMs, tools, conditions, checkpoints).
- State persistence, time-travel, human-in-the-loop.
- **Relations:** `→ B.5` agent runtime concepts.

### C.2.2 CrewAI / AutoGen / OpenAI Swarm
- Role-based agents, conversation patterns.
- AutoGen (group chat), Swarm (handoff-as-tool), CrewAI (sequential/hierarchical).
- **Relations:** `→ C.2.4` handoff, `→ B.5.4`.

### C.2.3 Patterns
- Supervisor / worker, hierarchical, debate, reflection, planner-executor, ReAct.
- **Relations:** `→ B.5.1`.

### C.2.4 Handoff mechanics
- State carried across hands; tool-as-handoff; message routing.
- **Relations:** `→ B.5.4`.

### C.2.5 When NOT to use multi-agent
- Single-agent + tools beats multi-agent in most cases.
- Multi-agent overhead: latency, debugging, eval explosion.
- **Relations:** `→ D.1` evaluation.

## C.3 Tool protocols

### C.3.1 Function calling (OpenAI / Anthropic / Gemini)
- JSON schema input, parallel calls, streaming.
- **Relations:** `→ A.6.1`, `→ B.5.3`.

### C.3.2 OpenAPI tools
- Auto-generate tools from OpenAPI specs (LangChain, Composio).
- **Relations:** `→ A.6.1`.

### C.3.3 Model Context Protocol (MCP)
- Anthropic-led standard. Server (tool provider) ↔ client (LLM host).
- Resources, prompts, tools, sampling.
- Auth, transport (stdio, SSE, HTTP).
- **Relations:** `→ A.2.4`, `→ 05-mcp-and-tool-protocols.md`.

### C.3.4 Tool design patterns
- Idempotency, error contracts, dry-run, side-effect typing.
- Pagination, batch, partial-failure semantics.
- **Relations:** `→ A.1.6`, `→ A.6.1`.

### C.3.5 Sandbox & execution
- E2B, Modal, Daytona for code-exec tools.
- Filesystem isolation, network egress control, time limits.
- **Relations:** `→ A.7.2`, `→ C.7` safety.

## C.4 Streaming & realtime

### C.4.1 Token streaming (text)
- SSE → React; chunked rendering; backpressure on the client.
- **Relations:** `→ A.2.2`.

### C.4.2 Tool-call streaming
- Stream tool args as they're generated; cancel mid-flight.
- **Relations:** `→ C.3.1`.

### C.4.3 Reasoning streams
- Surfacing chain-of-thought (Claude extended thinking, GPT-thinking).
- **Relations:** `→ B.5.1`.

### C.4.4 Bidirectional streams
- WebSocket vs WebRTC for chat; trade-offs.
- **Relations:** `→ A.2.2`, `→ A.2.3`.

### C.4.5 Event sinks for agents
- Structured events: `step_started`, `tool_called`, `step_completed`, etc.
- For traces, UIs, audit.
- **Relations:** `→ D.4`.

## C.5 Voice agents

> Deep coverage in `06-voice-agents.md`.

### C.5.1 STT (Speech-to-Text)
- Whisper (open), Deepgram, AssemblyAI, ElevenLabs Scribe.
- Streaming vs batch; word-level timestamps; diarization.
- **Relations:** `→ C.5.6` latency.

### C.5.2 TTS (Text-to-Speech)
- ElevenLabs, Cartesia, Sesame, OpenAI TTS, Google.
- Streaming TTS; voice cloning; emotional control.
- **Relations:** `→ C.5.6`.

### C.5.3 Real-time pipelines
- LiveKit Agents, Pipecat, Vapi, Retell.
- Pipeline: mic → VAD → STT → LLM → TTS → speaker.
- **Relations:** `→ A.2.3`.

### C.5.4 Telephony integration
- Twilio, Vonage; SIP, WebRTC bridges.
- **Relations:** `→ A.2.3`.

### C.5.5 Barge-in / interruption
- Detect user speech mid-TTS; cancel, requeue, resume.
- **Relations:** `→ C.5.6`.

### C.5.6 Latency budgets
- Total target ≤800ms for natural turn-taking; STT ≤200, LLM TTFT ≤300, TTS ≤200.
- **Relations:** `→ B.3` serving, `→ E.3` cost.

### C.5.7 Voice eval
- WER (word error rate), conversational quality, task success.
- **Relations:** `→ D.1`.

### C.5.8 Multi-modal models
- GPT-4o realtime, Gemini Live (voice-in / voice-out in one model).
- vs pipeline approach: lower latency, less control.
- **Relations:** `→ C.5.3`.

## C.6 LLM application frameworks

### C.6.1 LangChain / LangGraph
- Pros: ecosystem, integrations. Cons: API churn, abstraction tax.

### C.6.2 LlamaIndex
- RAG-first; data connectors; query engines.

### C.6.3 Pydantic AI / Instructor / DSPy
- Type-first, schema-first, prompt optimization.

### C.6.4 Custom / minimal stacks
- Often the senior choice — direct provider SDKs + your own state machine.
- **Relations:** `→ B.5` agent runtime.

## C.7 Safety & guardrails

### C.7.1 Prompt injection
- Direct: user pastes "ignore previous instructions."
- Indirect: malicious instructions inside RAG corpus, tool results.
- Mitigations: instruction hierarchy, content delimiters, refusal training.
- **Relations:** `→ B.4` RAG (`rag_systems/07`), `→ C.3.5` sandbox.

### C.7.2 Jailbreaking patterns
- DAN, role-play, encoding (base64, leetspeak), many-shot, gradient.
- **Relations:** `→ C.7.5` red-teaming.

### C.7.3 Output safety
- PII detection, toxic content, secret leakage.
- Tools: Llama Guard, Guardrails AI, NeMo Guardrails, Presidio.
- **Relations:** `→ A.7.2`.

### C.7.4 Structured output enforcement
- JSON schema, grammars (outlines, xgrammar, llguidance).
- **Relations:** `→ C.3.1`.

### C.7.5 Red-teaming
- Adversarial test suite; manual + automated.
- HarmBench, Garak, PAIR.
- **Relations:** `→ D.1` evaluation.

### C.7.6 Constitutional / policy-based filtering
- Rule LLM screens prompts/outputs against policy.
- **Relations:** `→ B.5.5` agent safety hooks.

---

# AREA D — AI OPERATIONS

## D.1 Evaluation engineering

### D.1.1 Eval taxonomy
- Unit (single prompt), behavior (capabilities), regression (don't drop), eval-as-code, online (live).
- **Relations:** `→ rag_systems/08`, `→ B.5.6`.

### D.1.2 Golden sets
- Human-curated; versioned; sliced by difficulty/category/tenant.
- **Relations:** `→ rag_systems/08`.

### D.1.3 Agent trajectory eval
- Did the agent reach the goal? In how many steps? With what tool calls?
- AgentBench, SWE-Bench, τ-bench, WebArena.
- **Relations:** `→ C.2`.

### D.1.4 LLM-as-judge
- Pairwise > absolute; pin judge model; calibrate against humans.
- Position bias, verbosity bias, self-preference.
- **Relations:** `→ rag_systems/08`.

### D.1.5 Eval harnesses
- OpenAI Evals, Inspect (UK AISI), Promptfoo, DeepEval, Braintrust, Phoenix.

### D.1.6 Online vs offline
- Offline: eval set in CI. Online: sampled live, async LLM-judged.
- **Relations:** `→ A.7.1` SLOs.

### D.1.7 A/B and shadow
- Shadow: run candidate in parallel, log diffs, no user impact.
- A/B: split traffic; metric: faithfulness, deflection, CSAT.
- **Relations:** `→ D.2`.

## D.2 LLMOps & lifecycle

### D.2.1 Prompt versioning
- Source-of-truth in repo; semver; templates with variable validation.
- Promptlayer, Helicone, LangSmith, Braintrust.
- **Relations:** `→ A.6.4`.

### D.2.2 Model versioning
- Pin provider+version; track rollouts; rollback playbook.
- **Relations:** `→ A.7.5`.

### D.2.3 Drift detection
- Input drift (query distribution), output drift (style/length), quality drift (eval score).
- **Relations:** `→ D.1.6`.

### D.2.4 Canary / staged rollout
- 1% → 10% → 50% → 100%; gates on KPIs; auto-rollback.
- **Relations:** `→ A.7.5`, `→ D.4`.

### D.2.5 Feature flags for AI
- Per-tenant, per-cohort model/prompt overrides.
- LaunchDarkly, GrowthBook, custom.

### D.2.6 Data flywheel
- Prod logs → eval set → fine-tune data → improved model.
- Privacy controls; opt-out; PII scrubbing.
- **Relations:** `→ E.1` fine-tuning, `→ C.7.3`.

## D.3 Observability

### D.3.1 Tracing for LLM apps
- Span per LLM call, per tool call, per retrieval; nested.
- Tools: LangSmith, Langfuse, Phoenix (Arize), OpenLLMetry.
- **Relations:** `→ A.7.4` incident response.

### D.3.2 OpenTelemetry for AI
- Semantic conventions for LLM (gen_ai.* attributes).
- Bridges to existing infra (Honeycomb, Datadog, Grafana).
- **Relations:** `→ D.3.1`.

### D.3.3 Metrics (golden signals for LLM apps)
- TTFT, total latency, tokens/s, $/request, refusal rate, faithfulness (sampled).
- **Relations:** `→ D.5`.

### D.3.4 Structured logging
- Per-request: query, retrieved chunks, prompt, output, citations, scores, costs.
- **Relations:** `→ D.2.6` data flywheel.

## D.4 Reliability for AI systems

### D.4.1 Provider failover
- Anthropic → OpenAI fallback; degrade quality, preserve uptime.
- **Relations:** `→ C.1.4` LLM gateway.

### D.4.2 Caching layers
- Response cache, prompt cache, embedding cache.
- **Relations:** `→ A.5.1`, `→ rag_systems/06`.

### D.4.3 Backpressure & quotas
- Per-tenant rate limits; per-model quotas; queue with timeouts.
- **Relations:** `→ A.7.3`.

### D.4.4 Graceful degradation
- Smaller model → no retrieval → cached → "system unavailable, try later."

## D.5 Cost engineering

### D.5.1 Unit economics
- $/query by stage; $/MAU; gross margin per tier.
- **Relations:** `→ rag_systems/10`.

### D.5.2 Tier routing
- Cheap classifier picks Haiku/Mini/Sonnet/Opus per query; 5–10× cost reduction.
- **Relations:** `→ C.1.4`.

### D.5.3 Prompt caching
- Anthropic 90% off cache hits; OpenAI similar.
- Cache breakpoints in long system prompts/tools.
- **Relations:** `→ rag_systems/06`.

### D.5.4 Self-host vs API
- Break-even ~50–200M tokens/day depending on model.
- **Relations:** `→ B.3.6`, `→ C.1.5`.

### D.5.5 FinOps for AI
- Per-tenant cost attribution; budget alerts; spend-per-feature dashboards.
- **Relations:** `→ A.7.3`.

---

# AREA E — DOMAIN ADAPTATION

## E.1 Fine-tuning

### E.1.1 When fine-tune vs prompt vs RAG
- Decision tree: knowledge → RAG; behavior/style → SFT/DPO; format → prompt; latency → distillation.
- **Relations:** `→ B.4`, `→ E.2`.

### E.1.2 SFT (supervised fine-tuning)
- Data prep, format (chat/instruction), epochs, learning rate.
- **Relations:** `→ E.1.4` LoRA.

### E.1.3 DPO / RLHF / KTO
- Preference learning without explicit reward model.
- **Relations:** `→ llm_fundamentals/06`.

### E.1.4 LoRA / QLoRA
- Adapter ranks, quantization for QLoRA, multi-LoRA serving.
- **Relations:** `→ B.3.5`.

### E.1.5 Fine-tune deployment
- Together, Fireworks, OpenAI fine-tune, self-host on vLLM.
- **Relations:** `→ C.1.5`.

### E.1.6 Eval for fine-tunes
- Held-out test set; capability regression; do no harm on out-of-domain.
- **Relations:** `→ D.1`.

## E.2 Prompt engineering (advanced)

### E.2.1 Patterns
- Zero/few-shot, CoT, ToT (Tree-of-Thoughts), Reflexion, Self-Consistency.
- **Relations:** `→ B.5.1`.

### E.2.2 Structured outputs
- JSON schema, tool-use as structured output, grammar constraints.
- **Relations:** `→ C.7.4`.

### E.2.3 Compression
- LLMLingua, summarization for context shrink.
- **Relations:** `→ rag_systems/06`.

### E.2.4 Meta-prompting / DSPy
- Programmatic prompt optimization.

### E.2.5 Long-context strategies
- Lost in the middle, position-aware ordering.
- **Relations:** `→ rag_systems/06`.

## E.3 Domain knowledge encoding
- Knowledge graphs, ontologies, taxonomies; when each helps.
- **Relations:** `→ rag_systems/09` GraphRAG.

---

# AREA F — SENIOR ENGINEER SIGNALS

## F.1 System design interviews (AI-flavored)

### F.1.1 Frameworks
- Clarify scope → estimate scale → APIs → data model → core flow → bottlenecks → trade-offs.
- **Relations:** `→ 02-system-design-for-ai.md`.

### F.1.2 Common AI prompts
- "Design a RAG-as-a-service" / "an agent platform" / "voice agent at scale" / "an LLM gateway" / "prompt-versioning system" / "eval pipeline."
- **Relations:** all of Area C.

### F.1.3 Trade-off articulation
- Latency vs cost vs quality; build vs buy; mono-tenant vs multi-tenant.
- **Relations:** `→ rag_systems/10` Section 4 (four ops dimensions).

### F.1.4 Estimation fluency
- Tokens/sec at batch X; GPU memory budget; QPS to GPU mapping; cost back-of-envelope.

## F.2 Behavioral interviews

### F.2.1 STAR stories — the senior set
- 0→1 launch; 1→10 scale; cross-team conflict; mentoring; saying no; wrong technical call + recovery; on-call incident.

### F.2.2 Senior signals interviewers look for
- Ownership (you saw it through), scope-down (you cut what didn't matter), clarity (you can explain trade-offs), influence (others followed your design), failure post-mortem (you learn).

### F.2.3 Failure modes
- Pronouns (we vs I); story too long; no metric; no trade-off.

## F.3 Tech leadership

### F.3.1 Design docs
- Problem, goals/non-goals, options, decision, trade-offs, rollout, eval criteria.

### F.3.2 Code review at scale
- Bias-toward-merge; suggest don't dictate; review intent before line-by-line.

### F.3.3 Mentoring junior engineers
- Pair > review > docs; pull don't push; protect their time.

### F.3.4 On-call ownership
- Runbooks; pager hygiene; postmortems; reduce future load.

## F.4 Hiring & team-building (sometimes asked at startup senior)
- Bar-raiser mindset; how you'd interview your replacement; how you'd onboard a new engineer.

---

# AREA G — STARTUP-SPECIFIC

## G.1 0→1 product engineering
- Ship-then-polish; instrument from day 1; resist over-architecting; pick boring stack.
- **Relations:** `→ A.7.5`, `→ D.3`.

## G.2 1→10 scaling
- Find the bottleneck (CPU, memory, IO, lock, queue); fix root cause; add observability.
- **Relations:** `→ A.7`.

## G.3 Working with founders
- Async + concise; surface trade-offs not just answers; ship weekly.

## G.4 Indian AI startup landscape
- Lyzr, Sarvam, Krutrim, Composio, Neysa, Simplismart, Ema, Haptik, Gupshup, Observe.AI, Mad Street Den.
- US-with-India: Giga, Together, Modal, Replicate, Hugging Face, LangChain, Inngest.
- See [`../OPPORTUNITIES.md`](../OPPORTUNITIES.md).

## G.5 Comp negotiation
- Anchor on band research; cash > equity at this stage; ask for sign-on; competing offer is leverage; never accept on the call.

## G.6 Open-source presence
- One real repo > many one-shots; thoughtful README; demos > screenshots.

## G.7 Personal brand
- Blog or thread on agent runtime / RAG depth; LinkedIn architecture writeups; conference talks.

---

# CROSS-CUTTING CONCERN: WHAT TIES IT ALL TOGETHER

```
┌─────────────────────────────────────────────────────────────┐
│  AREA F — Senior engineering judgment, system design, comm. │
│  (this is the SCORE you're graded on)                       │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ demonstrated through
                          │
┌─────────────────────────────────────────────────────────────┐
│  AREA C + D — Production AI surface + Operations             │
│  (this is the WORK at an AI startup)                        │
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ built on
                          │
┌─────────────────────────────────────────────────────────────┐
│  AREA A + B — SWE foundations + AI internals                 │
│  (this is the BASELINE — assumed, not credited)              │
└─────────────────────────────────────────────────────────────┘
```

**Senior bar:** Area A + B are assumed. You earn the title in Area C + D + F.

**Comp bar (₹50 LPA → ₹1 Cr+):** Area C must be deep + Area F must be sharp. Area E is the differentiator.

---

# SELF-ASSESSMENT TEMPLATE

Copy this checklist and mark each leaf 🟢 / 🟡 / 🔴.

```
A.1 Distributed systems       [🟢🟡🔴 each sub]
A.2 Networking & protocols    [...]
A.3 OS & runtime              [...]
A.4 Storage                   [...]
A.5 Caching, queues, events   [...]
A.6 APIs & contracts          [...]
A.7 Reliability/security/ops  [...]
A.8 Containers & orchestration[...]

B.1 Transformers              → llm_fundamentals/01
B.2 Tokenization & generation → llm_fundamentals/02-03
B.3 LLM serving               [deep dive: 03-llm-serving-infra.md]
B.4 RAG                       → rag_systems/
B.5 Agent runtime             → lyzr_agent_runtime/
B.6 Token economics           → llm_fundamentals/08

C.1 LLM serving infrastructure[deep: 03]
C.2 Agent platforms           [deep: 04]
C.3 Tool protocols / MCP      [deep: 05]
C.4 Streaming & realtime      [...]
C.5 Voice agents              [deep: 06]
C.6 Application frameworks    [...]
C.7 Safety & guardrails       [deep: 09]

D.1 Evaluation engineering    [deep: 07]
D.2 LLMOps & lifecycle        [deep: 08]
D.3 Observability             [deep: 10]
D.4 Reliability for AI        [...]
D.5 Cost engineering          [deep: 10]

E.1 Fine-tuning               [deep: 11]
E.2 Prompt engineering        [...]
E.3 Domain knowledge          [...]

F.1 System design interview   [deep: 12]
F.2 Behavioral interview      [deep: 12]
F.3 Tech leadership           [...]
F.4 Hiring & team-building    [...]

G.* Startup-specific          [deep: 12 + 13]
```

---

# WHAT TO DO NEXT

1. **Spend an hour** going through every leaf above. Self-assess honestly.
2. **List your 🔴s.** Those are your study queue.
3. **Open `13-90-day-plan.md`** for the schedule.
4. **For every 🔴**, jump into the corresponding deep-dive doc in this folder.
5. **Re-assess monthly.** Track 🔴 → 🟡 → 🟢 movement.

---

**Folder index:** see [`README.md`](./README.md).
