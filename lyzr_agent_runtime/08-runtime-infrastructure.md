# 08 — Runtime Infrastructure

Where runs actually live, how they survive, and how they scale.

Reading prerequisite: [02 — Data Model](./02-data-model-and-contracts.md) (especially `Run`, `AgentState`, `Step`) and [03 — Planning and Execution](./03-planning-and-execution.md).

---

## The runtime in one sentence

A pool of stateless workers consumes runs from a durable queue, loads per-run state from Postgres, executes one step at a time, commits state + events atomically after each step, and returns the run to the queue if it's not terminal.

No worker owns a run. Any worker can pick up any step.

---

## Processes and topology

```
┌──────────────┐   ┌──────────────┐     ┌──────────────┐
│  API server  │   │  API server  │ ... │  API server  │   (stateless, N copies)
└──────────────┘   └──────────────┘     └──────────────┘
       │                  │                    │
       ▼                  ▼                    ▼
┌──────────────────────────────────────────────────────┐
│                  Event Bus (Redis pub/sub)           │
└──────────────────────────────────────────────────────┘
       ▲                                         ▲
       │                                         │
┌──────────────┐   ┌──────────────┐     ┌──────────────┐
│  Worker      │   │  Worker      │ ... │  Worker      │   (stateless, M copies)
└──────────────┘   └──────────────┘     └──────────────┘
       │                  │                    │
       ▼                  ▼                    ▼
┌──────────────────────────────────────────────────────┐
│            Queue  (Redis Streams or Postgres)        │
└──────────────────────────────────────────────────────┘
       ▲                                         │
       │                                         │
       └──── enqueue ────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│   Persistence  ( Postgres + Vector DB + Object Store )│
└──────────────────────────────────────────────────────┘
```

Three process kinds:

1. **API server** — HTTP/SSE/WebSocket endpoint. Validates requests, creates/updates Runs, enqueues work, proxies event streams to clients.
2. **Worker** — async event-loop process that pops a run from the queue, loads state, executes one step (or a batch of steps up to a time slice), commits, and either enqueues for next step or marks terminal.
3. **Cron / background** — embeddings backfill, memory GC, trace rollups, eval harness, cost reporting. Distinct from the request path.

None of the three owns state. Everything is in Postgres + Redis. Scale is horizontal for all three.

---

## The queue

A work queue that holds `{run_id, priority, ready_at, attempt}`.

**v1 choice: Redis Streams.** Reasons:
- Native consumer groups give us exactly-once delivery semantics per consumer.
- Durable (AOF + replication).
- Built-in pending list for crash recovery (`XPENDING`, `XCLAIM`).
- Low operational overhead — we already run Redis for the event bus.

**v2 alternative: Postgres as queue** (SKIP LOCKED on a `work_items` table). Attractive when we want transactional enqueue atomic with a Run insert; but operationally heavier under high QPS.

**Not SQS / SNS** — adds cross-cloud coupling; we want to stay self-contained until scale forces it.

### Message shape
```
work_item = {
    run_id: RunId,
    tenant_id: TenantId,
    priority: int,                 # 0..100, higher first; default 50
    ready_at: datetime,            # for delayed retries / debounce
    attempt: int,
    hint: "start" | "resume" | "human_injected" | "subagent_result",
}
```

### Producers
- API layer on `POST /runs` (hint = "start").
- Worker after a non-terminal step (hint = "resume").
- API layer on `POST /runs/{id}:inject` (hint = "human_injected").
- Sub-agent completion handler (hint = "subagent_result") — unblocks parent.

### Consumers
- Workers. Each runs a consumer group `workers`. On popping:
  1. Claim the message (Redis XREADGROUP).
  2. Process the step.
  3. On success: XACK + enqueue next (if needed).
  4. On worker crash: message remains pending; another worker XCLAIMs after idle_timeout.

---

## The worker lifecycle

```
while True:
    work = queue.pop(timeout=1s, consumer_group="workers")
    if not work:
        continue
    run = load_run(work.run_id)
    if run.status not in {queued, paused, running_with_resumable_state}:
        queue.ack(work)
        continue

    state = load_agent_state(run.id)
    agent = load_agent(run.agent_id, run.agent_version)
    run.status = "running"
    save_run(run)

    try:
        result = execute_step(agent, state, ctx)
        commit_step_transaction(result)   # <-- critical atomic boundary
    except TransientError as e:
        backoff_requeue(work, attempt+1, reason=e)
        continue
    except FatalError as e:
        mark_failed(run, e)
        queue.ack(work)
        continue

    if result.terminated:
        mark_terminal(run, result)
        queue.ack(work)
    else:
        queue.ack(work)
        enqueue_next(run.id)
```

Worker is a single `async` loop per process; multiple runs can be in-flight per worker via `asyncio` tasks up to `max_concurrent_runs` per worker.

### Concurrency within a worker

```python
MAX_CONCURRENT_RUNS_PER_WORKER = 16
sem = asyncio.Semaphore(MAX_CONCURRENT_RUNS_PER_WORKER)

async def worker_loop():
    while True:
        work = await queue.pop()
        if not work: continue
        async with sem:
            asyncio.create_task(handle(work))
```

Concurrency cap prevents one run from monopolizing a worker but also bounds memory. Value is tuned to the worker's CPU/memory profile.

### Worker health

Each worker heartbeats to Redis every 5s. A missed heartbeat for 30s marks the worker dead; its claimed-but-unacked messages are reclaimable by others via XCLAIM after `idle_timeout=60s`. The crashed worker's in-flight steps therefore resume on another worker from the last committed checkpoint — not from the mid-step point, which has no durable state.

---

## The atomic step commit

After every step, **all** of the following must commit or none do:

1. `steps` row (new, immutable)
2. `runs` row update (budget, status, checkpoint_version, updated_at)
3. `agent_states` row (upsert, new checkpoint_version)
4. `messages` rows (appended)
5. `memory_records` inserts (implicit memory writes)
6. `events` rows (emitted during the step)

Implementation: a single Postgres transaction wraps all six. External side-effects (tool calls, LLM calls, event-bus publishes) happen *before* this transaction — they are the result we are committing.

```
BEGIN;
INSERT steps ...;
UPDATE runs SET checkpoint_version = $v, budget_used = $u, status = $s ...;
INSERT INTO agent_states (run_id, checkpoint_version, blob) VALUES ($run, $v, $blob)
  ON CONFLICT (run_id) DO UPDATE SET checkpoint_version = EXCLUDED.checkpoint_version, blob = EXCLUDED.blob;
INSERT INTO messages ...;
INSERT INTO memory_records ...;
INSERT INTO events ...;
COMMIT;
```

After commit, we publish `step.completed` to the event bus (a fire-and-forget redis publish). Subscribers (SSE fan-out, eval pipeline) consume from the bus.

**If the commit fails** (e.g., serialization error), we retry the transaction up to 3 times. If it still fails, we mark the run `failed` with code `checkpoint_failure` — we do NOT continue without a checkpoint. This is the invariant from the README: `never continue past a checkpoint failure`.

**Why external calls happen before the commit, not inside:**
- LLM calls are slow (seconds); holding a transaction open that long is suicide under load.
- Tool calls may have external side-effects that we can't roll back anyway.
- So we design the system to be **re-entrant**: if a worker crashes between an external call and the commit, another worker restarts from the prior checkpoint. The re-done external call is idempotent (via tool idempotency keys for write tools, or harmless for read tools).

### Exactly-once illusion

We are not truly exactly-once. We are at-least-once on external side effects (a write tool might be called twice if a worker crashes at just the wrong moment) and exactly-once on state commit. Idempotency keys on write tools bridge the gap. Documented explicitly.

---

## Agent state persistence

`AgentState` blob is stored two ways:

1. **Latest** — `agent_states` table, one row per run, the current state. Used for resume and debugging.
2. **Historical** — embedded inside each `Step` row's payload. Enables point-in-time replay.

```sql
CREATE TABLE agent_states (
  run_id        TEXT PRIMARY KEY,
  tenant_id     TEXT NOT NULL,
  checkpoint_version  INT NOT NULL,
  state_blob    JSONB NOT NULL,
  state_size    INT NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

State blobs over 1 MB spill to object storage, with a URI reference in `state_blob`. (Rare — most blobs stay under 256 KB.)

### Schema evolution for state

If we change the `AgentState` schema (add a field, rename a field), we:

1. Bump a `state_schema_version` number (inside the blob).
2. Write a lazy migration: on load, detect old version, upgrade, save back.
3. Keep migration code indefinitely — long-running paused runs may still hold old versions.

---

## Event bus

Purpose: fan out events to (a) SSE subscribers in API servers, (b) observability sinks, (c) eval pipelines.

**v1: Redis Pub/Sub.** Ephemeral — a subscriber that isn't connected misses the event. That's fine because all events are also committed to the `events` table; subscribers can catch up by reading from Postgres if needed.

**v2: Redis Streams** (same as the work queue, different stream). Gives us persistence + late subscribers + consumer groups for horizontal-scale sinks.

Topics:
- `tenant.{tenant_id}.runs.{run_id}` — per-run stream consumed by one SSE connection.
- `tenant.{tenant_id}.runs.*` — per-tenant firehose.
- `all.events` — platform-wide, for ops/observability.

Wildcard subscriptions are a Pub/Sub feature; under Streams we explicitly aggregate at the sink.

### SSE fan-out

API server subscribes to `tenant.{t}.runs.{r}` when a client opens SSE. As events arrive on Pub/Sub, the server writes them to the SSE response. On client disconnect, the server unsubscribes. A keep-alive `:heartbeat\n\n` every 15s prevents intermediaries from closing the socket.

### Replay

A late-arriving SSE client can request `?from=<event_id>` to get events from a point. API server reads from the `events` table for events with `id > from` and streams them before switching to live Pub/Sub. This is replay on demand.

---

## Persistence details

### Postgres

Tables (abbreviated):

```
tenants
principals
agents
tools
runs                 — partitioned by created_at month
steps                — partitioned by run_id hash (or created_at month)
messages             — partitioned by run_id hash
agent_states         — one row per run, updated in place
memory_records
memory_embeddings    — only if not using pgvector column
events               — partitioned by at month; hot partition in fast storage
budgets_used_rollup  — materialized view for tenant cost dashboards
tool_invocations_audit
```

### Indexes (just the ones that matter)

```
runs (tenant_id, status, created_at DESC)
runs (agent_id, created_at DESC)
steps (run_id, ordinal)
messages (run_id, created_at, id)
memory_records (tenant_id, namespace, kind, deprecated_at) WHERE deprecated_at IS NULL
memory_records (tenant_id, subject_id) WHERE subject_id IS NOT NULL
events (trace_id, at)
events (run_id, at)
```

### Partitioning strategy

- `runs`, `steps`, `messages`, `events` — partitioned by month on `created_at` or `at`. Hot month stays on SSD; colder months move to cheaper storage and eventually detach to cold archive.
- Old partitions do NOT drop automatically — they detach. Drop requires explicit ops action to comply with tenant data retention contracts.

### Connection pooling

PgBouncer in transaction mode. Workers hold connections only for the commit transaction window; API servers hold briefly for reads. Pool size = (expected concurrent connections × 1.2).

### Read replicas

Traces and historical queries hit a read replica. Run-state reads (critical path in worker) hit primary — replication lag during resume would cause wrong-state loads.

---

## Redis

Two Redis roles — separable into two clusters if needed, one cluster at v1 scale.

- **Queue cluster** — Redis Streams for work queue + Pub/Sub for event bus. Persistent (AOF + replication).
- **Cache cluster** — short-term memory buffers, response cache, idempotency cache, rate limiters. Can be less durable.

Key prefixes:
```
queue:{tenant}:work        — work stream
bus:run:{run_id}           — pub/sub channel
stm:{run_id}:summary       — short-term summary hot copy
stm:{run_id}:recent        — recent message list (LIST)
idemp:{tool_id}:{key}      — idempotency cache
resp_cache:{hash}          — response cache
ratelimit:{scope}:{key}    — token buckets
lock:ns:{namespace}:{sub}  — memory-write lock
health:provider:{pid}      — provider health score
health:worker:{wid}        — worker heartbeat
```

### Eviction
Queue and bus keys: never evict.
Cache keys: `allkeys-lru`. Critical caches (idempotency) live in a separate logical DB that does NOT evict.

---

## Object storage

S3 (or GCS / Azure Blob). Used for:
- Large tool inputs/outputs (>64 KB).
- State-blob spill (>1 MB).
- Full event-log archive (cold tier).
- File attachments to runs (user-uploaded PDFs, images).

URIs stored in database point to the objects. Signed URLs for external access.

Bucket layout:
```
s3://bucket/tenant_id=<tid>/run_id=<rid>/file.<ext>
s3://bucket/tenant_id=<tid>/traces/<yyyy>/<mm>/<dd>/<trace_id>.ndjson.gz
s3://bucket/tenant_id=<tid>/states/<run_id>/checkpoint_<n>.json.gz
```

Retention and lifecycle rules align with tenant plans.

---

## Vector DB

See [04 — Memory Systems](./04-memory-systems.md) for retrieval logic. v1 uses pgvector (same Postgres cluster, fewer moving parts). Switch to Qdrant / Pinecone when:
- Namespaces exceed ~10M records and IVFFlat/HNSW rebuilds get expensive.
- Query QPS pressures Postgres planner.
- Multi-region replication of indices is needed.

The memory manager's interface is unchanged — swap is a config change.

---

## Determinism guarantees

The runtime is deterministic given:
- A frozen model (temp=0, stable seed where supported).
- A frozen memory corpus.
- A frozen tool catalog and tool implementations.
- A captured clock (see below).

What's NOT deterministic:
- Wall-clock time (unless captured).
- `uuid.uuid4()` or similar (replace with seeded RNG).
- Anything depending on external state that changed.

### Clock injection

The runtime wraps `datetime.now()` behind an injected clock. In production, it's wall time. In replay, it's read from the trace. No code calls `datetime.now()` directly — linted.

### RNG injection

Same rule for random numbers. A `Run` carries a `run_seed`; all randomness is derived via `hash(run_seed, call_site_label)`.

---

## Pause / resume / cancel

### Pause
API: `POST /runs/{id}:pause`. Sets a flag in Redis; worker checks at the top of each step. On detection, completes the current step (to keep state consistent), commits, marks run `paused`, emits `run.paused`.

### Resume
API: `POST /runs/{id}:resume`. Flips status `paused → queued`, enqueues a work item with hint `resume`.

### Cancel (graceful)
Same as pause but terminal — marks `cancelled` after the current step completes.

### Cancel (force)
API: `POST /runs/{id}:cancel?force=true`. Sets `cancelled_dirty` flag; worker exits the step mid-way after best-effort rollback. Marks run `cancelled_dirty`. Reserved for ops use; clients should prefer graceful.

### Inject (human-in-the-loop)
API: `POST /runs/{id}:inject {content: "...", as: "user"}`. Appends a user message, enqueues work with hint `human_injected`. Worker loads state, sees the new message, continues.

---

## Backpressure and load shedding

Three levels:

1. **Per-tenant concurrency cap.** Max N runs per tenant executing simultaneously. Extra runs wait in queue.
2. **Per-model rate limit.** The provider layer already gates this; excess waits.
3. **Global load-shed.** At system-wide saturation, low-priority runs are delayed; an honest 503 is returned on `POST /runs` when the queue is saturated beyond a threshold.

Priorities:
- Interactive (SSE-streaming, human waiting) > background (eval, batch).
- Paying tier > free tier.
- Handoffs inherit parent's priority.

---

## Time slicing

A single step should fit in "seconds to tens of seconds." Longer than 60 seconds in one step is a smell. If a step's `duration_ms` exceeds a per-agent threshold:

- Log `step.slow`.
- If repeat, suggest "break this tool into smaller tools" to operators.
- The worker does NOT preempt mid-step — an LLM call in-flight for 45 seconds is still valid; we just track it.

---

## Multi-region considerations

v1 is single-region. Notes for v2:

- Postgres logical replication to secondary regions (read-only).
- Workers per region pull from a region-local queue.
- A run is sticky to the region where it was created (for data-residency simplicity).
- Global tenant metadata replicates; tenant data stays in region.

---

## Deployment shape

Everything is a container.

- `api-server` image: FastAPI + uvicorn workers (4 per container); health endpoint `/healthz`.
- `worker` image: Python + our runtime; health endpoint on admin port.
- `cron` image: scheduled background jobs (APScheduler or a cron sidecar).
- `migration` init-container: runs Postgres migrations to the required version before API/worker start.

Orchestrator: Kubernetes in production, docker-compose for local dev. Helm chart shipped alongside.

More on deployment: [13 — Deployment and Scaling](./13-deployment-and-scaling.md).

---

## Local development

```
docker-compose up postgres redis minio pgvector
poetry run uvicorn api.main:app --reload
poetry run python -m worker --concurrency=4
poetry run python -m cron
```

A seeded dev database with a `hello-world` agent, 3 tools, and 5 golden traces. `make dev` does all of the above.

---

## Failure modes

| Failure | Detection | Response |
|---------|-----------|----------|
| Worker crash mid-step | Heartbeat loss + XPENDING | XCLAIM after idle timeout → another worker resumes from last checkpoint |
| Postgres primary down | Health check | API returns 503; workers back off; leader-elected failover; resume |
| Redis queue down | Worker pop fails | Workers idle; API queues in-memory briefly then 503s |
| Event bus down | Publish fails | Events still committed to `events` table; late subscribers catch up |
| Disk full on object store | Spill write fails | State under 1 MB stays in Postgres; over-1MB runs fail explicitly; alert |
| Clock skew across workers | Observability (comparisons) | Workers use NTP; within-step timings derived from monotonic clock |

---

## Metrics (operational, not business)

- `worker.concurrent_runs` (gauge per worker)
- `worker.step_duration_ms` (histogram)
- `queue.depth` (gauge)
- `queue.age_seconds` (gauge — oldest item wait time)
- `checkpoint.duration_ms` (histogram; commit transaction time)
- `checkpoint.retry_count` (counter)
- `resume.count` (counter — reclaims after crash)
- `pg.connection_pool.in_use` (gauge)
- `redis.memory_bytes` (gauge)

Alerts:
- Queue age > 60s for > 5m.
- Worker reclaim rate > 1%/min.
- Checkpoint retry rate > 0.1%.
- Run failure rate > 2%/5m (per tenant and aggregate).

---

## Upgrade / migrations

- **Schema migrations** via Alembic. All backward-compatible; new columns with defaults; old columns deprecated for a release before removal.
- **Code upgrades** rolling-deploy:
  1. Pause scale-in of old workers.
  2. Deploy new workers alongside old.
  3. Drain old workers: stop taking new work, finish in-flight steps.
  4. Remove old workers.
  5. Same for API.
- **Breaking changes** to AgentState go through a version bump + lazy migration on load (see earlier).

---

## What this layer does NOT decide

- Prompt shape (planner + compiler).
- Tool logic (tools).
- Safety policy (policy engine).
- Pricing and billing (accounting rolls up from here, but tariffs are elsewhere).

This layer is plumbing. Good plumbing is boring, and that is the point.

---

## Next: [09 — Observability and Tracing](./09-observability-and-tracing.md). We've been emitting events everywhere; now we define what a complete trace looks like and how to use it.
