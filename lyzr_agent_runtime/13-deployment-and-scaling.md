# 13 — Deployment and Scaling

Prerequisite: [08 — Runtime Infrastructure](./08-runtime-infrastructure.md) defines the process topology and data stores. [12 — API Surface](./12-api-surface.md) defines the contract we must keep stable under load. This document is about how those pieces are packaged, deployed, operated, scaled, isolated per tenant, and recovered from failure.

Deployment is not the glamorous part of an agent platform. It is the part that determines whether it works at 3 AM.

---

## 1. Packaging

### 1.1 Container images

Three images, one source tree:

| Image | Entrypoint | Responsibilities |
|---|---|---|
| `lyzr/api:<version>` | `uvicorn lyzr.api.app:app` | REST, SSE, WebSocket handlers. Stateless. |
| `lyzr/worker:<version>` | `python -m lyzr.worker` | Claims runs from queue, executes steps. Stateless (state is in stores). |
| `lyzr/cron:<version>` | `python -m lyzr.cron` | Scheduled jobs: memory consolidation, dataset eval runs, retention, GC. Singleton per tenant partition. |

- Base: Python 3.12 slim. Built as multi-stage Dockerfile with deterministic pip install (`--require-hashes` against a locked file).
- Non-root user (`uid=10001`), read-only root filesystem, writable `/tmp` and `/var/run` only.
- No shell in final image. No `curl`, no debugging tools. If you need to inspect, use a debug sidecar or `kubectl debug`.
- Images signed with cosign, SBOM published alongside (CycloneDX), provenance attestation (SLSA level 3).
- Size target: < 250 MB compressed. Larger images slow autoscale.

### 1.2 Image tagging

```
lyzr/api:1.14.2          # exact semver
lyzr/api:1.14            # latest patch of 1.14
lyzr/api:1               # latest minor of 1.x
lyzr/api:sha-abc123      # git sha
lyzr/api:canary          # current canary build
```

Production deploys pin to exact semver. Never deploy `:latest` to production. Never.

---

## 2. Kubernetes deployment

Kubernetes is the reference deployment target. Other targets (Nomad, ECS, raw VMs) are possible but not documented here — the abstractions below translate.

### 2.1 Resource layout per region

```
Namespace: lyzr-system           # Platform components (control plane, operators)
Namespace: lyzr-shared           # Shared tenants (default pool)
Namespace: lyzr-tenant-<id>      # Dedicated-tier tenants (one per)
Namespace: lyzr-data             # Postgres, Redis (if self-managed; usually external)
```

Workloads per tenant namespace (dedicated tier) or in shared:

- `Deployment/api` — 3+ replicas, PodDisruptionBudget `minAvailable: 2`
- `Deployment/worker` — N replicas based on queue depth (HPA)
- `Deployment/cron` — `replicas: 1`, leader election via a Kubernetes Lease for safety
- `Service/api` — ClusterIP; exposed externally via Ingress + WAF
- `HorizontalPodAutoscaler/worker`
- `PodDisruptionBudget` for each deployment
- `NetworkPolicy` — deny ingress from other tenant namespaces, allow egress only to data plane + approved egress proxy
- `ResourceQuota` and `LimitRange` — hard caps per namespace

### 2.2 Pod specs (essentials)

**API pod:**
```yaml
resources:
  requests: { cpu: "500m", memory: "1Gi" }
  limits:   { cpu: "2000m", memory: "2Gi" }
readinessProbe:
  httpGet: { path: /healthz/ready, port: 8000 }
  periodSeconds: 5
livenessProbe:
  httpGet: { path: /healthz/live, port: 8000 }
  periodSeconds: 10
  failureThreshold: 3
terminationGracePeriodSeconds: 45
# SSE connections; give the app time to drain (emit final heartbeat,
# close streams politely) before SIGKILL.
```

**Worker pod:**
```yaml
resources:
  requests: { cpu: "1000m", memory: "2Gi" }
  limits:   { cpu: "4000m", memory: "4Gi" }
# No HTTP readiness; worker readiness = queue connection established.
readinessProbe:
  exec: { command: ["python", "-m", "lyzr.worker.health"] }
  periodSeconds: 10
terminationGracePeriodSeconds: 120
# On SIGTERM: stop claiming new runs, let in-flight step finish,
# checkpoint, re-enqueue, then exit.
```

**Cron pod:**
```yaml
resources:
  requests: { cpu: "250m", memory: "512Mi" }
  limits:   { cpu: "1000m", memory: "1Gi" }
```

### 2.3 Ingress / edge

- TLS termination at the ingress (cert-manager for Let's Encrypt or BYO certs).
- HTTP/2 required (SSE over HTTP/2 multiplexes well).
- Idle timeout ≥ 120 s (SSE needs this).
- WAF in front: OWASP CRS + rules for injection-heavy patterns relevant to agent workloads (see doc 10).
- DDoS mitigation via upstream provider (Cloudflare, AWS Shield, etc.). Origin not directly exposed.
- Connection limits: 10 k concurrent per pod, 100 k per region (tune via real load; these are starting points).

---

## 3. Autoscaling

### 3.1 API layer

HPA on CPU + request rate:
```yaml
minReplicas: 3
maxReplicas: 50
metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 60 } }
  - type: Pods
    pods: { metric: { name: http_requests_per_second }, target: { type: AverageValue, averageValue: 200 } }
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies: [{ type: Percent, value: 10, periodSeconds: 60 }]
  scaleUp:
    stabilizationWindowSeconds: 0
    policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
```

Scale up fast, scale down slow. SSE connections are stateful from the client's point of view; aggressive scale-down causes preventable disconnects.

### 3.2 Worker layer — queue-depth-driven

HPA on Redis queue depth, measured per consumer group:
```yaml
minReplicas: 2
maxReplicas: 200
metrics:
  - type: External
    external:
      metric: { name: lyzr_queue_pending_runs, selector: { matchLabels: { group: "workers-default" } } }
      target: { type: AverageValue, averageValue: "10" }
  - type: External
    external:
      metric: { name: lyzr_queue_oldest_pending_seconds, selector: { matchLabels: { group: "workers-default" } } }
      target: { type: AverageValue, averageValue: "5" }
```

Two signals: depth (10 pending per worker → scale up) and age (oldest pending > 5 s → scale up). Age catches the case where a few very expensive runs block the queue even when depth is moderate.

**KEDA** (Kubernetes Event-Driven Autoscaler) is the recommended controller — native Redis Streams trigger, predictable behavior. Native HPA with external metrics works but requires a custom-metrics adapter; KEDA is simpler.

### 3.3 Cron — not autoscaled

Cron runs at fixed replica 1 with leader election. Scaling cron horizontally means duplicate job fires; we don't do that. If cron work is too heavy for a single pod, split the jobs across multiple cron deployments (`cron-memory`, `cron-eval`, `cron-retention`), each with its own lease.

### 3.4 Scale floors per tier

| Tier | API floor | Worker floor | Notes |
|---|---|---|---|
| Shared (default) | 3 | 2 | Per region. Baseline hot capacity. |
| Dedicated standard | 2 | 2 | Per tenant namespace. |
| Dedicated enterprise | 3 | 4 | Per tenant namespace, cross-AZ. |

Floors exist so that cold starts and first-request latency are bounded. A tenant whose traffic spikes from 0 to 50 rps at 9:00 AM Monday should not see worker-scale-up latency on the first dozen runs.

---

## 4. Multi-tenancy

### 4.1 Tenancy models

| Model | Compute | Storage | Use when |
|---|---|---|---|
| Shared pool | Shared API + worker pods across tenants | Shared Postgres, schema-per-tenant or row-level isolation | Default for most tenants. Cheapest. |
| Dedicated namespace | Per-tenant worker pool; shared API | Dedicated Postgres schema; shared cluster | Noisy-neighbor concerns, compliance signals. |
| Dedicated cluster | Per-tenant everything | Dedicated Postgres instance, dedicated Redis | Enterprise, regulated industries, sovereignty. |
| Single-tenant region | Tenant is the only tenant in a region | Dedicated regional data plane | Government, healthcare with data residency requirements. |

The runtime code path is **identical** across models — tenancy is a deployment concern, not a code concern. Same container image, same schema, same queue semantics, different physical placement.

### 4.2 Isolation mechanisms

1. **Logical isolation always.** Every table has `tenant_id`. Every query filters by tenant. Every cache key is tenant-prefixed. Every log line, every span, every event.
2. **Network isolation for dedicated tiers.** NetworkPolicies deny cross-tenant pod traffic. Egress through a tenant-scoped proxy that logs every outbound request.
3. **Resource quotas per tenant namespace** (dedicated tiers) — CPU, memory, pod count, PVC size.
4. **Compute weight tagging** — every Redis Stream message carries `tenant_id` and `tenant_tier`. Worker consumer groups can be configured to prioritize or dedicate to tenants.

### 4.3 Noisy neighbor defenses

Shared tier is where this matters. Defenses:

- **Per-tenant queue lanes.** Runs are sharded into lanes (`lyzr:runs:shard:{tenant_id % N}`) so one tenant can saturate at most one lane's worth of workers.
- **Per-tenant rate limits.** Token bucket on API + admission control on worker claim.
- **Fairness scheduling** at the worker claim step: if tenant X has 100 pending and tenant Y has 3, worker picks tenant Y's job every Mth claim (weighted round-robin, not FIFO). Prevents starvation.
- **LLM provider concurrency fences** — each provider key has a concurrency cap per tenant. One tenant cannot eat all of a provider's headroom.

---

## 5. Data plane

### 5.1 Postgres

- Managed service preferred (AWS Aurora, GCP AlloyDB, Azure Postgres Flexible, or equivalent). Self-hosted only for sovereign-cloud deployments.
- **Primary + 2 replicas**, synchronous replication to one, async to the other.
- PITR (point-in-time recovery) enabled; backup retention 30 days default, 90 days for enterprise.
- Partitioned tables (`runs`, `steps`, `events`, `memory_records`) by `created_at` monthly. Partitions older than retention are dropped, not deleted row-by-row.
- Connection pooling via PgBouncer in transaction-pooling mode. API and worker connect through PgBouncer, never directly.

### 5.2 Redis

- Managed service preferred (ElastiCache, MemoryStore, Azure Cache, Upstash). Sentinel or cluster mode for HA.
- Stream retention: max 100 k entries per stream or 24 h, whichever first. Claimed messages ACKed and removed; XPENDING monitored.
- Keyspace separated by purpose: `q:*` (queues), `sse:*` (pub/sub channels), `rate:*` (rate limits), `cache:*` (warm caches), `lock:*` (leases).
- Not used as source of truth. Redis loss = reprocess in-flight steps from Postgres checkpoints. It degrades, it does not destroy.

### 5.3 Vector store

- Default: `pgvector` in the same Postgres. One less moving piece. HNSW index, dimension 1536 or 3072 depending on embedding model.
- Upgrade path: tenant can opt into external vector DB (Pinecone, Qdrant, Weaviate) when their memory set exceeds ~10 M vectors or their latency SLO requires it. Dual-write during migration, then cut over.

### 5.4 Object storage

- S3 / GCS / Azure Blob for large artifacts (document uploads, tool payload overflow, exported runs).
- Server-side encryption (KMS-backed). Bucket policy: deny public read. Signed URLs with 15-min TTL for client downloads.
- Lifecycle rules: move to infrequent-access tier after 30 days, glacier after 180 days, delete after retention period (tenant-configurable).

### 5.5 Secrets

- Runtime config via Kubernetes Secrets backed by an external secret manager (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault). External Secrets Operator syncs.
- Per-tenant secrets (tenant-supplied tool credentials, tenant-owned LLM keys) encrypted at rest with tenant-specific KMS keys (envelope encryption). Access logs mandatory.

---

## 6. Release engineering

### 6.1 Promotion pipeline

```
dev → staging → canary → prod-us-east → prod-eu-west → prod-in-south → prod-* (rest)
```

- **dev:** every merge to main. Integration tests run. Eval baseline runs on a smoke dataset.
- **staging:** nightly. Full eval dataset. Synthetic load tests. No real traffic.
- **canary:** 1% of prod-us-east traffic. Monitor for 24 h.
- **prod-*:** region-by-region rollout over 72 h. Any region can be paused or rolled back independently.

### 6.2 Deployment strategy

- **Rolling** for worker and cron. `maxSurge: 25%`, `maxUnavailable: 10%`.
- **Blue/green** for API if the change touches handshake semantics (new SSE event types, WebSocket protocol changes). Otherwise rolling.
- **Database migrations** — expand/contract pattern. Two releases minimum to introduce a breaking schema change:
  1. Release N: add new column/table, write to both old and new, read from old.
  2. Release N+1: backfill, switch reads to new.
  3. Release N+2: stop writing to old, drop old.
- **Feature flags** (LaunchDarkly, Unleash, or home-grown) for risky changes. Default off. Canary-only until signal is good.

### 6.3 Rollback

- **Code:** redeploy previous image tag. < 5 minutes.
- **Schema:** only forward. Expand/contract makes rollback safe without schema reversal; if a migration must be undone, we accept the window of dual-write waste and roll forward with a correction. Never `DROP TABLE` as part of rollback.
- **Feature flag:** instant (flag flip).

### 6.4 Upgrade compatibility

- API: v1 breaking changes are never "upgrades." They are v2 (see doc 12).
- Worker: can process runs started on any runtime version within the same minor series. Major version bump drains old runs first (pause the queue for that tenant, let it flush, then switch).
- Agent state schema: versioned (doc 08 §4.3). Migrations are applied on load, not in bulk.

---

## 7. Capacity planning

### 7.1 Sizing inputs

| Input | How to estimate |
|---|---|
| Runs per second | Historical API telemetry; forecast trend + seasonality. |
| Avg step duration | Median ~2 s, P95 ~8 s for standard chat. Tool-heavy workflows run 10–30 s. |
| Avg steps per run | 3–5 for chat, 6–15 for agentic workflows. |
| Concurrent runs | RPS × avg_duration × avg_steps. |
| Tokens per run | Provider-dependent. Baseline 25 k tokens for a typical agent with 3 tool calls. |
| Memory writes per run | ~2 per run (consolidation gated). |

### 7.2 Worker capacity

Rule of thumb per worker pod at reference sizing (2 vCPU, 4 GB):

- 4–8 concurrent runs (limited by LLM latency, I/O, and memory).
- ~1 run/sec throughput for short runs, ~0.1 run/sec for long tool-heavy runs.
- Saturation is usually LLM latency, not CPU. Measure before scaling up hardware.

### 7.3 Postgres sizing

- Baseline small-tenant: `db.t4g.medium` equivalent. Storage scales with run retention.
- Medium tenant (1 M runs/month, 90 d retention): ~400 GB storage, `db.r6g.xlarge`.
- Large tenant (10 M runs/month): dedicated Aurora cluster, 4 TB storage with auto-scaling, `db.r6g.4xlarge` primary + 2 readers.

### 7.4 Redis sizing

- Queue keyspace is small (typically < 100 MB even at high throughput, because messages are claimed quickly).
- SSE pub/sub is ephemeral.
- Cache and rate-limit keys grow with tenant count, not traffic.
- `cache.r6g.large` handles most deployments. Scale vertically before horizontally (clustering adds operational complexity).

---

## 8. Cost model

### 8.1 Unit economics

| Component | Cost driver | Per-run cost at reference scale |
|---|---|---|
| LLM tokens | Provider API spend | **$0.02–$0.50** (dominates) |
| Compute (API + worker) | vCPU × seconds | $0.0003–$0.003 |
| Postgres | Storage + IOPS, per tenant share | $0.0001–$0.001 |
| Redis | Queue/pubsub ops | $0.00005 |
| Egress | Webhooks + SSE bytes | $0.00005–$0.0005 |
| Vector ops | Embedding calls + index queries | $0.0005–$0.005 |
| Object storage | Artifact size × retention | $0.00001 |

Rule: **LLM spend is 80–95% of variable cost.** Infrastructure optimization that doesn't reduce token spend is low-leverage.

### 8.2 Margin levers

1. **Prompt caching** (doc 07) — 50–90% cost cut on warm prefixes.
2. **Router downgrades** — route to smaller models where grader evals show equivalent quality.
3. **Budgets** — enforce per-run caps; reject budget overruns before they happen, not after.
4. **Batching** — for async eval and memory consolidation workloads, use provider batch APIs (50% discount typical).
5. **Self-hosted inference** — for high-volume fixed-latency workloads with stable prompts, self-host an open model. Only economic above ~1 M runs/day on that specific flow.

### 8.3 Cost attribution

Every run has `cost.dollars_detail = {tokens_in, tokens_out, cache_read, cache_write, tool_calls, storage}`. The runtime writes this to `steps.cost_json` on each step commit and rolls up to `runs.cost_total`. Tenants see per-agent, per-tool, per-day cost via `/v1/reports/cost` (reference endpoint; schema in doc 09 §12).

---

## 9. Multi-region

### 9.1 Topology

- **Region-active, tenant-homed.** Each tenant has a home region where its data lives. Writes only in the home region. Reads optionally replicated to peer regions.
- **Not a global active-active runtime.** State-machine runs with side effects and cost accounting do not survive active-active cleanly. We don't pretend otherwise.
- Global control plane (tenant routing, billing, admin) is eventually consistent across regions.
- Failover: if the home region is down, tenant reads degrade (stale from peer); writes fail. Tenant data is not migrated across regions automatically. Manual failover is a documented playbook; it takes minutes, not seconds, and it's the right tradeoff.

### 9.2 Replication

- Postgres: logical replication to peer regions for read replicas (not failover targets — just for geo-distributed reads).
- Redis: not replicated. Ephemeral per region.
- Object storage: cross-region replication for audit-tier artifacts (policy-tier decision).

### 9.3 Data residency

Tenants in regulated regions (EU, India, Middle East) are pinned to their region — their tenant record is flagged `residency_locked: true`, and the control plane refuses to route traffic elsewhere. Cross-region replication disabled. Peer-region readers disabled.

---

## 10. Disaster recovery

### 10.1 Recovery objectives

| Event | RPO (data loss tolerance) | RTO (recovery time) |
|---|---|---|
| Single pod failure | 0 | < 30 s (HPA / restart) |
| AZ failure | 0 | < 5 min (k8s reschedule, multi-AZ DB) |
| Region failure | < 5 min (async replica lag) | Minutes to hours depending on residency constraints |
| Postgres corruption | Last successful backup (15 min in PITR) | < 1 h restore |
| Redis total loss | In-flight steps only | < 5 min (reinitialize, workers reclaim from Postgres) |
| Object storage bucket loss | 0 (versioned, cross-region replicated) | Hours |

### 10.2 Backups

- Postgres: continuous WAL backup + daily full snapshot. Tested restore drills monthly. Retention 30 d (enterprise: 90 d).
- Redis: not backed up (ephemeral).
- Object storage: versioned + cross-region replication.
- Secrets: exported + encrypted to a separate KMS domain monthly (cold backup).
- Config (Kubernetes manifests): GitOps repository is the source of truth; cluster loss recoverable from Git alone.

### 10.3 Drills

Quarterly game days:

- Random pod termination (Chaos Mesh).
- Redis primary failover.
- Postgres failover to replica.
- Simulated region loss — can we serve degraded reads? Can we re-home a tenant?

Every drill produces a writeup. Remediation items become P1 bugs.

---

## 11. Observability for operators

Not the in-app observability (doc 09 is about that). This is what the oncall engineer looks at.

### 11.1 Dashboards

- **Fleet overview** — RPS, error rate, P50/P95/P99 latency per endpoint, per region.
- **Queue health** — depth per consumer group, oldest pending age, claim rate, dead-letter count.
- **Worker health** — active workers, CPU, memory, step commit rate, failed claim rate.
- **Cost meter** — $/min realized, projected monthly burn vs budget, per-tenant top 20.
- **Provider health** — per-provider success rate, P95 latency, rate-limit headroom.

### 11.2 Core alerts (operator-facing)

| Alert | Threshold | Severity | Response |
|---|---|---|---|
| API error rate > 1% (5 min) | P1 | Page | Check deploys, provider status, WAF false positives. |
| Queue oldest-pending > 60 s | P1 | Page | Workers stuck; check logs, scale up, dump XPENDING. |
| Worker fleet CPU > 90% (10 min) | P2 | Ticket | Scale up; investigate hot run. |
| Postgres replica lag > 30 s | P2 | Ticket | Check primary load; may need read-query isolation. |
| Redis memory > 85% | P1 | Page | Scale up; investigate unbounded key. |
| Cost burn > 2× forecast (hourly) | P2 | Ticket | Find the tenant, check if expected, possibly throttle. |
| Tenant quota near exhaustion | P3 | Email tenant | Automated notification at 80%, 95%, 100%. |

---

## 12. Security posture

Deployment-level security. Policy-level is doc 10; API-level is doc 12.

- **Zero-trust networking.** Pod-to-pod traffic requires mTLS (Istio / Linkerd service mesh) in enterprise deployments. Shared tier: NetworkPolicy only.
- **Privileged operations audited.** Any `admin`-scope action writes to a separate audit log (WORM, separate tenant's read access).
- **Secrets rotation.** Quarterly rotation for LLM provider keys, tool credentials, signing keys. Automated via external secret manager.
- **Image vulnerability scanning.** Trivy / Grype on CI and on a daily schedule against running images. CVE with fix available → patch within the SLA (P0: 24 h; P1: 7 d; P2: 30 d).
- **Pod security.** PodSecurityStandards: `restricted`. No privileged, no hostPath, no hostNetwork, no capabilities beyond defaults.
- **Egress proxy.** Tenant traffic to the internet goes through an auditable proxy. Allowlist-based for enterprise; logged for all.

---

## 13. What we explicitly don't do

- **Run customer code in the worker process.** Custom guards and tools that execute untrusted code run in a separate sandbox (gVisor / Firecracker), never in the worker pod.
- **Accept SSH into production pods.** Debug sidecars + `kubectl debug` only. No bastion with creds floating around.
- **Edit production data by hand.** All production mutations go through audited API endpoints or approved runbooks.
- **Deploy to multiple regions simultaneously.** Rollouts are serial. The extra hour of rollout time is cheaper than a multi-region incident.
- **Mix tenant workloads in a single container.** The worker process handles many tenants' runs, but each run is isolated — any CPU/memory escalation by one tenant is bounded by the step's budget and the pod's resource limit. A tenant wanting stronger isolation buys dedicated.

---

## 14. Failure modes

| Failure | Blast radius | Detection | Mitigation |
|---|---|---|---|
| Worker pod OOM | One step → re-enqueued | k8s event + metric | Increase memory limit; hunt leak with pprof/py-spy. |
| Postgres primary fails | Writes pause 20–60 s during failover | Cloud provider alert | Replica promoted; runs continue from checkpoint. |
| Redis queue partition | In-flight runs delayed until partition heals | XPENDING age metric | Reclaim via XCLAIM; workers re-poll. |
| Provider API 5xx storm | Runs fail or route to fallback | Provider health metric | Router fallback (doc 07); downgrade to cheaper model. |
| Bad deploy (crash loop) | Rolling deploy halts at first replica fail | k8s readiness + alert | Automatic rollback on readiness failure quorum. |
| Secret rotation failure | New traffic fails auth to a provider | 401 rate alert | Roll back secret version; investigate. |
| Noisy tenant eats workers | Queue age rises for others | Per-tenant queue age metric | Lane sharding kicks in; throttle tenant; invoice reviews. |
| WAF false positive | Legitimate tenant calls blocked | 403 rate on a single tenant | Exempt pattern; file WAF rule fix. |

---

## 15. Deployment checklist

Before a production-grade deployment is "done":

- [ ] Images pinned to exact semver.
- [ ] Readiness and liveness probes configured.
- [ ] `terminationGracePeriodSeconds` ≥ worker step duration.
- [ ] PDBs in place for every Deployment.
- [ ] HPA configured with queue metric for workers.
- [ ] ResourceQuota and LimitRange per tenant namespace.
- [ ] NetworkPolicy denying cross-tenant ingress.
- [ ] Ingress TLS, HTTP/2, idle timeout ≥ 120 s.
- [ ] Postgres PITR enabled, backup verified by restore drill.
- [ ] Redis persistence configured (AOF or RDB depending on tier).
- [ ] Vector index HNSW parameters tuned for dataset size.
- [ ] Object storage bucket lifecycle + encryption + versioning.
- [ ] Secrets synced from external manager; no plaintext in manifests.
- [ ] Alerts configured with PagerDuty/Opsgenie routing.
- [ ] Dashboards imported and bookmarked.
- [ ] Runbooks written and linked from alerts.
- [ ] Chaos drill passed (pod kill, AZ loss simulation).
- [ ] Cost attribution verified against billing provider.
- [ ] Security scan clean; SBOM published.

If you can't tick all of these, you're not done; you just think you are.

---

## 16. Summary

The deployment story is boring on purpose. Kubernetes, managed Postgres, managed Redis, KEDA for queue-driven scaling, region-pinned tenants, expand/contract migrations, rolling canaries, PITR, versioned images, signed artifacts. Nothing here is novel — that's the point. Novelty belongs in the planning loop, not the deployment pipeline.

The only opinionated parts:

- Tenants are region-homed, not globally replicated. We don't pretend to solve active-active state machines.
- Worker scaling is **queue-depth-and-age-driven**, not CPU-driven. Agent workloads are I/O-bound; CPU is a lagging indicator.
- LLM spend dominates unit economics by 10–50×. Optimize there first.
- Rollouts are serial across regions and strict about expand/contract on schema. Boring failures are cheaper than exciting recoveries.

Next: [14 — Roadmap and Extension Points](./14-roadmap-and-extension-points.md). We've defined what to build and how to run it; finally, we define what comes next and where the runtime is designed to evolve.
