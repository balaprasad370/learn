# 01 — Distributed Systems

> The non-AI baseline for senior interviews. Assumed knowledge — you don't get points for it, but you lose offers without it.

---

## 1. The mental model

A distributed system is multiple machines that fail independently and communicate over an unreliable network. **Three things will always be true and always trip teams up:**

1. **The network is unreliable.** Packets drop, get reordered, get duplicated, arrive late.
2. **Machines fail partially.** A node that responds to pings can have a corrupt disk, full GC, or stuck thread.
3. **Time is hard.** Clocks drift; "happens-before" is not the same as "wall-clock-before."

Every concept below is a tool to handle one or more of those.

---

## 2. CAP, PACELC, FLP

### 2.1 CAP
Under a network partition, you can have **Consistency** OR **Availability**, not both. Pick one.

- **CP system:** rejects writes during partition (etcd, ZooKeeper, MongoDB primary).
- **AP system:** accepts writes on both sides; reconciles later (Cassandra, Dynamo, Riak).
- **CA "system":** doesn't exist under partition. Single-node DBs are CA only because they have no partition possibility.

### 2.2 PACELC
Extends CAP for the no-partition case: **Else, Latency or Consistency.**

- A system can be CP under partition + LC normally (strong consistency, higher latency: Spanner).
- Or AP + LL (eventual, low latency: Cassandra).

PACELC is a more honest framing — most operational time has no partition.

### 2.3 FLP impossibility
In a fully asynchronous system with even one faulty process, **no deterministic consensus algorithm can terminate**. Real systems escape FLP by adding timeouts (which assume partial synchrony).

**Use in interviews:** when asked "why doesn't X system have stronger guarantees?" the answer is often "FLP — it'd block forever under one bad node."

---

## 3. Consistency models

Strict → loose:

| Model | Guarantee | Example |
|---|---|---|
| Linearizable | Single global order; reads see latest write | etcd, ZooKeeper, single Postgres |
| Sequential | Global order, but not real-time | distributed log w/o clocks |
| Causal | If A → B, all see A before B | TAO (Facebook), some CRDTs |
| Read-your-writes | Your reads see your writes | session affinity |
| Monotonic reads | Reads don't go backward in time | sticky session reads |
| Eventual | Replicas converge "eventually" | DNS, S3 (now strong-after-write) |

**Senior signal:** know that "eventual consistency" is an OK answer when paired with "and these per-session guarantees on top: read-your-writes + monotonic reads."

---

## 4. Replication

### 4.1 Topologies
- **Single-leader (primary-replica):** all writes to leader; replicas read-only or read-after-replication. Postgres, MySQL.
- **Multi-leader:** writes accepted by multiple leaders; conflicts resolved via vector clocks / CRDTs / last-write-wins. Often regretted at scale.
- **Leaderless:** any node accepts writes; quorum (R+W>N) for consistency. Cassandra, Dynamo.

### 4.2 Sync vs async
- **Sync:** write returns after N replicas confirm. Strong durability, higher latency.
- **Async:** write returns from leader. Risk of data loss on leader failure.
- **Semi-sync:** at least 1 replica confirms. Common Postgres setup.

### 4.3 Failure recovery
- **Failover:** elect new leader; promote replica.
- **Split brain:** old leader still alive, two leaders accept writes; need fencing tokens or STONITH.
- **Replication lag:** monitor; alert; route reads to leader if lag > threshold.

---

## 5. Consensus (Raft, Paxos)

You won't implement; you must know the shape.

### 5.1 Raft (the readable one)
- Roles: leader, follower, candidate.
- Leader election: heartbeats, election timeout, term numbers.
- Log replication: leader appends; replicates to majority; commits.
- Safety: log never rolls back committed entries.

**Used in:** etcd, Consul, CockroachDB, TiKV, Kafka KRaft, Redis Sentinel.

### 5.2 Paxos (the one papers cite)
- Multi-Paxos powers Spanner, Chubby.
- Less intuitive; rarely implemented from scratch.

### 5.3 In an interview
"For coordination state, I'd use a Raft-backed store like etcd. I would not implement consensus myself."

---

## 6. Partitioning / sharding

### 6.1 Strategies
- **Hash-based:** `hash(key) % N`. Even distribution; resharding nightmare.
- **Consistent hashing:** ring with virtual nodes; minimal data movement on resharding. Dynamo, Cassandra.
- **Range-based:** ordered key ranges; supports range scans; risk of hot ranges (sequential keys).
- **Directory-based:** lookup table maps keys → shards; flexible; lookup adds latency.

### 6.2 Resharding
- Always painful. Plan from day 1: virtual buckets > physical nodes; rebalance moves buckets, not data.
- Online migration: dual-write, backfill, dual-read with compare, cut over, drop old.

### 6.3 Hot keys
- One celebrity row gets all traffic. Mitigate: cache in front, replicate that key, write-through with sub-keys.

---

## 7. Idempotency, exactly-once, at-least-once

### 7.1 The semantics
- **At-most-once:** fire and forget. Loses data on failure.
- **At-least-once:** retry until ACK. Duplicates possible. **The common default.**
- **Exactly-once:** duplicates suppressed at consumer. The hard one.

### 7.2 Idempotency in practice
- **Idempotency key:** client-generated UUID per logical operation; server stores `(key, result)` for a TTL; replays return cached result.
- **Required for:** payment APIs, agent step execution, any retry-able write.

### 7.3 Exactly-once via outbox
- Same DB transaction writes business state + outbox row.
- Background job ships outbox rows to message bus.
- Consumer is idempotent (dedupes by event ID).
- Result: end-to-end exactly-once illusion built from at-least-once primitives.

### 7.4 Kafka EOS
- Kafka transactions + idempotent producer + read-committed consumer.
- Limited to within Kafka; bridging to external systems still needs idempotent consumer.

---

## 8. Networking essentials

### 8.1 Timeouts, retries, jitter
- Always set timeouts on every RPC. "No timeout" is a future incident.
- Retries with **exponential backoff + jitter** to avoid thundering herd.
- Cap retries; classify errors: retry only on transient (5xx, network) not permanent (4xx).

### 8.2 Backpressure
- Don't accept what you can't serve. Prefer rejection (429) over queue collapse.
- Bounded queues. Adaptive concurrency limits (Netflix concurrency-limits).

### 8.3 Circuit breakers
- After N failures, stop calling the dependency for T seconds. Half-open probe.
- Stops cascading failure; lets dependencies recover.

### 8.4 Bulkheads
- Isolate resource pools per dependency. One slow downstream can't drain all your threads.

### 8.5 Hedged requests
- Send to two replicas; take the first response. Trades cost for tail latency.

---

## 9. Time

### 9.1 Clocks lie
- NTP keeps within ~10ms; can jump or skew.
- **Never** order events across machines using wall clocks alone.

### 9.2 Logical clocks
- **Lamport timestamps:** total order; loses concurrency info.
- **Vector clocks:** preserves concurrency; size = #nodes.

### 9.3 Hybrid logical clocks (HLC)
- Combines wall clock + counter; bounded skew.
- Used in CockroachDB, MongoDB (since 3.6).

### 9.4 TrueTime (Spanner)
- Hardware-bounded clock skew (atomic + GPS).
- Enables external consistency without coordination.
- Few have it. Most use HLC.

---

## 10. Failure modes

| Failure | Detection | Mitigation |
|---|---|---|
| Crash | Health check fail | Failover, restart |
| Slow node ("gray failure") | p99 latency spike | Hedge, bypass, capacity-based routing |
| GC pause | Heap metrics, RTT spikes | Tune GC, smaller heap, ZGC/Shenandoah |
| Network partition | Quorum loss, partial connectivity | Quorum-based decisions, fencing tokens |
| Cascading failure | Multi-service alerts firing together | Circuit breakers, bulkheads, backpressure |
| Thundering herd | Spike on cache expiry / restart | Jittered retries, request coalescing |
| Memory leak | Slow OOM | Process restart; root-cause heap dump |
| Disk full | Write failures | Disk monitoring; log rotation; oldest-first eviction |
| Clock skew | Logical clock anomalies | NTP monitoring; HLC; reject events with future timestamps |
| Poison pill / hot row | Per-row latency spike | Per-row metrics; circuit-break that row |

---

## 11. Storage choices (decision tree)

```
Need ACID transactions across multiple keys?
  Yes → Postgres / CockroachDB / Spanner.
  No → continue.

Need millisecond latency, simple key-based access?
  Yes → Redis / DynamoDB / Memcached.

Need full-text or vector search?
  Yes → Elasticsearch / OpenSearch / pgvector / Qdrant / Milvus.

Need analytics over append-only events?
  Yes → ClickHouse / BigQuery / Snowflake / Druid / Pinot.

Need flexible schema, document model?
  Yes → MongoDB / DocumentDB.

Need graph traversal?
  Yes → Neo4j / Neptune / DGraph.

Need huge throughput, ordered log?
  Yes → Kafka / Pulsar / Kinesis.

Need cheap blob storage?
  Yes → S3 / GCS / R2.

Default: Postgres. It's good enough for >90% of workloads.
```

---

## 12. Caching

### 12.1 Strategies
- **Cache-aside:** app checks cache → DB → fills cache. Simplest.
- **Read-through:** cache calls DB on miss. Transparent.
- **Write-through:** writes go cache + DB synchronously. Strong consistency, slower.
- **Write-behind:** writes to cache, DB async. Risk of loss; high throughput.

### 12.2 Hazards
- **Stampede / dog-pile:** N requests miss simultaneously, all hit DB. Mitigate: per-key locking, request coalescing, early refresh.
- **Stale-while-revalidate:** serve stale, refresh in background. Lower p99.
- **Negative caching:** cache misses too; bound TTL.
- **Hot key:** one key gets all hits — replicate that key; client-side cache.

### 12.3 Eviction
- LRU, LFU, FIFO, TinyLFU (Caffeine, modern best).
- TTL alone is not eviction; you also need size cap.

### 12.4 In LLM systems
- Embedding cache (query → vector).
- Response cache (query+context fingerprint → answer).
- Prompt cache (provider-side; Anthropic 90% off hits).
- Semantic cache (cluster similar queries; serve same answer).

---

## 13. Queues, streams, events

### 13.1 Queue vs log
- **Queue (SQS, RabbitMQ):** message consumed once, then gone. Random access not supported.
- **Log (Kafka, Pulsar, Kinesis, Redis Streams):** append-only; multiple consumers; replay; time-bounded retention.

### 13.2 Consumer groups
- Partitions split across consumers; one partition per consumer in a group.
- Adding consumers > partitions = idle consumers.

### 13.3 Offsets, replay, DLQ
- Track position per consumer group.
- Replay from offset for backfill/recovery.
- Dead-letter queue for poison messages; alert and inspect.

### 13.4 Ordering guarantees
- Per-partition order in Kafka; key-based partitioning to keep related events together.
- No global order across partitions — design around it.

### 13.5 Event sourcing
- State = fold(events). Replay rebuilds state.
- Strong audit trail; complex to evolve; CQRS often paired.

### 13.6 Change data capture (CDC)
- Stream DB changes (Postgres logical replication, Debezium) → Kafka → downstream.
- Keep search index, cache, analytics in sync without dual-write hazards.

---

## 14. APIs at scale

### 14.1 REST
- Resources, idempotent verbs (GET, PUT, DELETE), POST for creates.
- Pagination: cursor > offset (offset O(N) on deep pages).
- Errors: RFC 7807 Problem Details JSON.
- Versioning: URI (`/v1/`) is most common; header-based for advanced cases.

### 14.2 gRPC
- Schema-first (Protobuf). Code generation.
- Streaming: client, server, bidi.
- Better than REST for internal service-to-service.

### 14.3 GraphQL
- Single endpoint; client picks fields.
- N+1 problem: solve with DataLoader.
- Federation across services (Apollo Federation, Relay).
- Persisted queries for cache + security.

### 14.4 Rate limiting
- Token bucket: refill rate + burst.
- Leaky bucket: smooths bursts.
- Sliding window: precise but stateful.
- Per user, per tenant, per IP, per endpoint. Multi-dimension.

### 14.5 Idempotency at API level
- `Idempotency-Key` header → cached response for TTL.
- Required for: payments, agent runs, any retry-able write.

---

## 15. Observability

### 15.1 The three pillars
- **Metrics:** Prometheus, time-series, aggregates. Cheap, low-cardinality.
- **Logs:** structured JSON; ship to Loki/ELK/Datadog. Searchable. Expensive at volume.
- **Traces:** OpenTelemetry; one trace per request; spans for sub-operations. Causality across services.

### 15.2 Golden signals (Google SRE)
- **Latency** (p50, p95, p99, p99.9). Especially long-tail.
- **Traffic** (RPS, throughput).
- **Errors** (rate, by type).
- **Saturation** (CPU, memory, queue depth).

### 15.3 SLIs / SLOs / error budgets
- **SLI:** the metric (p99 < 500ms).
- **SLO:** the target (99.9% of requests).
- **Error budget:** 0.1% × period = downtime allowance. Spend on releases vs reliability work.

### 15.4 Distributed tracing
- One trace ID per request, propagated through every service via headers (`traceparent` W3C standard).
- Span per operation; parent-child relationship.
- Tools: Jaeger, Tempo, Honeycomb, Datadog APM.

---

## 16. Security baseline

### 16.1 OWASP top 10
- SQL injection, XSS, broken auth, broken access control, IDOR, SSRF, deserialization, log4shell-class.
- Know each; know the fix.

### 16.2 Secrets management
- Never in env vars in plaintext at rest. Use AWS Secrets Manager, GCP Secret Manager, Vault.
- Rotate. Audit. Least-privilege IAM per service.

### 16.3 Authn / authz
- AuthN: who you are. JWT, OAuth2/OIDC, SAML.
- AuthZ: what you can do. RBAC, ABAC, OPA/Cedar.
- Multi-tenant: tenant_id in every query, enforced at DB or middleware.

### 16.4 Data at rest / in transit
- TLS everywhere. mTLS for service-to-service in zero-trust.
- Disk encryption (KMS, AWS EBS encryption).
- Field-level encryption for PII.

### 16.5 Audit trail
- Who did what, when, on what resource. Immutable log.
- Required for SOC2, HIPAA, regulated industries.

---

## 17. Multi-tenancy

### 17.1 Isolation tiers
- **Pool:** shared everything; tenant_id in every row. Cheapest, leak-prone.
- **Silo (namespace):** logical separation; shared physical resources.
- **Bridge:** dedicated DB / cluster per tenant. Most expensive, strongest isolation.

### 17.2 Noisy neighbor
- One tenant's burst kills another's latency.
- Mitigate: per-tenant rate limits, queue priority, dedicated tier for big customers.

### 17.3 Cost attribution
- Tag every operation with tenant_id; aggregate cost per tenant.
- Shows margin per customer, informs pricing.

### 17.4 Data residency
- EU customer data must stay in EU. Multi-region deployment with tenant→region mapping.

---

## 18. Deploy strategies

### 18.1 Blue/green
- Two identical environments; switch traffic; rollback by switching back.
- Resource-heavy; clean.

### 18.2 Canary
- 1% → 10% → 50% → 100%; gates on metrics.
- Standard for risky changes.

### 18.3 Feature flags
- Code is deployed; behavior toggled per user/tenant.
- Decouples deploy from release.
- Tools: LaunchDarkly, GrowthBook, Unleash.

### 18.4 Dark launches
- Run new code in shadow; log only; compare to old.
- Validates correctness before exposing to users.

### 18.5 Database migrations
- **Expand-contract:** add new schema, dual-write, backfill, cut over reads, drop old.
- **Online schema change:** gh-ost (MySQL), pg_repack (Postgres) for ALTERs without locks.
- Never block deploys on long migrations; run them out-of-band.

---

## 19. Common system design building blocks

You should be able to design any of these from memory:

- **Rate limiter** (Redis + sliding window or token bucket).
- **URL shortener** (base62 + DB + cache + 301 redirect).
- **Distributed counter** (Redis HINCRBY or sharded counter).
- **Fan-out feed** (push vs pull vs hybrid; Twitter-style).
- **Notification system** (queue + worker + idempotent send + DLQ).
- **Job scheduler** (cron + DB + leader election + idempotent jobs).
- **Webhook delivery** (queue + retries + DLQ + signature verification).
- **Distributed lock** (Redis Redlock — controversial; etcd lease — better).
- **Pagination** (cursor-based with stable sort key).
- **Search autocomplete** (trie or prefix index + ranked candidates).

For interviews, practice each in 5–8 minutes.

---

## 20. AI-system intersections (where this matters most)

When the interview is for an AI startup:

- **Agent runs as durable jobs** → queue + idempotent worker + checkpoint state (`lyzr_agent_runtime/`).
- **Vector store sharding** → consistent hashing, namespace per tenant (`rag_systems/04`).
- **LLM serving autoscale** → KEDA on queue depth + warm pool for cold-start (`03-llm-serving-infra.md`).
- **Streaming responses** → SSE + backpressure + cancel-on-disconnect.
- **Multi-tenant RAG** → row-level filter + per-tenant cost cap + per-tenant eval slice.
- **Provider failover** → circuit breaker on Anthropic→ OpenAI fallback.
- **Cost attribution** → tenant_id propagated through every span; spans tagged with model + tokens.

---

## 21. Books / references (depth on demand)

- Designing Data-Intensive Applications — Kleppmann (the bible; read it once).
- The Site Reliability Engineering book (Google) — chapters on SLOs, error budgets.
- Database Internals — Petrov (storage engines depth).
- System Design Interview — Alex Xu (vol 1 & 2; covers the building blocks list).

---

## 22. Self-check

Without looking back, can you:

- [ ] Explain CAP and PACELC with one example each.
- [ ] Sketch Raft leader election and log replication.
- [ ] Design a rate limiter on Redis (token bucket, key schema, edge cases).
- [ ] Articulate when to choose AP over CP and vice versa.
- [ ] Explain the outbox pattern and why it solves dual-write.
- [ ] List the three queue ordering pitfalls in Kafka.
- [ ] Describe at-least-once + idempotent consumer = effectively-once.
- [ ] Define SLI / SLO / error budget for a real service.
- [ ] Walk through expand-contract migration for a NOT NULL column.
- [ ] Explain why circuit breakers exist and where they go.

If any are 🔴: that's your study session this week.

---

Next: [`02-system-design-for-ai.md`](./02-system-design-for-ai.md)
