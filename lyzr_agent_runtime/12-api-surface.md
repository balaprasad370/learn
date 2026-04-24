# 12 — API Surface

Prerequisite: the runtime internals are defined in [01](./01-high-level-architecture.md)–[11](./11-evaluation-and-testing.md). This document defines the **external boundary**: what clients (applications, SDKs, UIs, other agents) see and are allowed to call. Nothing beyond this boundary is a supported integration point. Everything inside can be refactored without notice.

The API is the promise. The implementation is not.

---

## 1. Design principles for the API

1. **REST for control, SSE for stream, WebSocket only when bidirectional.** One transport per concern. Do not overload HTTP with long-lived bidirectional semantics.
2. **Every mutating call is idempotent.** The client supplies `Idempotency-Key`; the server caches the first response for 24 h and replays it.
3. **Every run has an opaque ID.** Clients never construct IDs. They receive ULIDs and round-trip them.
4. **Explicit versioning in the URL.** `/v1/...`. Breaking changes mean `/v2`. Never in-place-breaking.
5. **Errors are structured.** RFC 7807 `application/problem+json`. `type`, `title`, `status`, `detail`, `instance`, plus runtime-specific fields (`trace_id`, `error_code`, `retryable`).
6. **Pagination is cursor-based, never offset.** Offset pagination diverges under concurrent writes; cursor pagination does not.
7. **Read endpoints are cacheable.** `ETag` + `If-None-Match`. Writes invalidate via event stream, not polling.
8. **No hidden state.** Anything that mutates a run is a documented endpoint with a request body you can log.

---

## 2. Authentication and authorization

### 2.1 Authentication methods

| Method | When used | Header |
|---|---|---|
| API key | Server-to-server, scripts, CI | `Authorization: Bearer lyz_sk_...` |
| Session cookie | Browser-based studio UI | `Cookie: lyzr_session=...` (HttpOnly, Secure, SameSite=Lax) |
| OAuth2 bearer | Partner integrations, delegated access | `Authorization: Bearer <jwt>` |
| mTLS | High-assurance enterprise deployments | client certificate |

API keys are prefixed so leaks in logs are greppable: `lyz_sk_live_...` (secret, live), `lyz_pk_live_...` (publishable, live — never grants run writes), `lyz_sk_test_...` (test mode, separate billing + data).

### 2.2 Scopes

Keys have scopes. The runtime enforces scopes **before** invoking the policy engine (defense in depth — denied by scope means the request body is never parsed).

| Scope | Grants |
|---|---|
| `runs:write` | Create, pause, resume, cancel, inject |
| `runs:read` | Read run, list runs, tail events |
| `agents:write` | Create, update, delete agent configs |
| `agents:read` | Read, list agents |
| `tools:write` | Register/update tool manifests |
| `tools:read` | Read tool registry |
| `memory:read` | Read memory records |
| `memory:write` | Delete/forget memory records (no direct insert — inserts happen through runs) |
| `traces:read` | Read spans, events, cost accounting |
| `admin` | Tenant configuration, key management, quotas |

Deny-by-default: absence of scope = 403. Compound actions (handoff crossing tenants) require scope on both sides.

### 2.3 Tenant isolation

`X-Tenant-Id` header is **not trusted** from clients — the tenant is derived from the key. The header exists only for admin keys that can operate on behalf of a tenant.

---

## 3. URL structure and versioning

```
https://api.lyzr.ai/v1/{resource}[/{id}[/{sub-resource}]]
```

- `v1` is the **major** version. Breaking changes bump to `v2`. Parallel versions coexist for **≥ 12 months**.
- Minor changes (additive fields, new optional parameters, new endpoints) ship without version bump. Clients MUST tolerate unknown fields.
- Deprecations are signaled via `Sunset` and `Deprecation` headers (RFC 8594). No silent removals.

### 3.1 Regions

Multi-region deployments expose region-scoped hostnames:

```
https://us-east.api.lyzr.ai/v1/...
https://eu-west.api.lyzr.ai/v1/...
https://in-south.api.lyzr.ai/v1/...
```

Tenant data does not cross regions. A run created in `us-east` cannot be read via `eu-west`. The root `api.lyzr.ai` is a router that 307-redirects to the tenant's home region.

---

## 4. Resource model

The public resources:

| Resource | Path prefix | Notes |
|---|---|---|
| Agents | `/agents` | Agent configurations (versioned) |
| Runs | `/runs` | Individual executions |
| Tools | `/tools` | Tool registry |
| Datasets | `/datasets` | Evaluation datasets |
| Evals | `/evals` | Eval runs |
| Memories | `/memories` | Long-term memory records |
| Traces | `/traces` | Spans, events, costs for a run |
| Webhooks | `/webhooks` | Event subscriptions |
| Keys | `/keys` | API key management (admin) |

Sub-resources are scoped under their parent: `/runs/{id}/events`, `/runs/{id}/steps`, `/agents/{id}/versions`.

---

## 5. Runs API (the hot path)

Everything else is supporting. 90% of client traffic is against `/runs`.

### 5.1 Create a run

```http
POST /v1/runs
Authorization: Bearer lyz_sk_live_...
Idempotency-Key: 01HXYZ...
Content-Type: application/json

{
  "agent_id": "agt_01HX...",
  "agent_version": "7",
  "input": {
    "messages": [
      {"role": "user", "content": "Summarize Q3 revenue across segments."}
    ]
  },
  "inputs": { "report_date": "2026-09-30" },
  "budget": {
    "max_tokens": 200000,
    "max_dollars": 2.50,
    "max_wall_seconds": 180,
    "max_steps": 40,
    "max_tool_calls": 60
  },
  "metadata": {
    "user_id": "u_42",
    "session_id": "s_abc"
  },
  "callbacks": {
    "webhook_url": "https://client.example.com/lyzr/events",
    "events": ["run.completed", "run.failed", "hitl.requested"]
  },
  "mode": "async"
}
```

Response (202 Accepted for async, 200 OK for sync completion):

```json
{
  "run_id": "run_01HX...",
  "status": "queued",
  "agent_id": "agt_01HX...",
  "agent_version": "7",
  "created_at": "2026-04-23T10:15:03.441Z",
  "stream_url": "https://api.lyzr.ai/v1/runs/run_01HX.../events",
  "trace_id": "trc_01HX..."
}
```

**Modes:**

| Mode | Behavior | Use when |
|---|---|---|
| `async` (default) | Returns 202 immediately with `run_id`. Client watches via SSE or webhook. | Any real workload. |
| `sync` | Blocks up to `max_wall_seconds`, returns the terminal state. Returns 504 if timeout. | Tiny deterministic flows, eval harness. |
| `stream` | Returns `text/event-stream` directly from POST. Connection is the run. | Interactive UIs that will lose nothing if the socket drops. |

**Idempotency-Key** is required for `POST /runs`. Re-submitting the same key within 24 h returns the cached response (same `run_id`). Different body + same key = `409 Conflict`.

### 5.2 Get run state

```http
GET /v1/runs/{run_id}
If-None-Match: "W/\"etag-v3\""
```

Returns the current snapshot. Sub-resource includes are opt-in via `?include=`:

```
GET /v1/runs/run_01HX...?include=steps,last_message,budget
```

Without includes, responses are small and cacheable. With includes, they're complete but larger. The default is minimal — this is a hot endpoint.

```json
{
  "run_id": "run_01HX...",
  "agent_id": "agt_01HX...",
  "status": "running",
  "started_at": "...",
  "updated_at": "...",
  "step_count": 7,
  "last_step_id": "stp_01HX...",
  "budget_used": { "tokens": 41203, "dollars": 0.1942, "wall_seconds": 22, "steps": 7, "tool_calls": 3 },
  "budget_remaining": { "tokens": 158797, "dollars": 2.306, ... },
  "output": null,
  "error": null,
  "trace_id": "trc_01HX..."
}
```

### 5.3 List runs

```http
GET /v1/runs?agent_id=agt_01HX...&status=running,failed&created_after=2026-04-01T00:00:00Z&page_size=50&cursor=...
```

Response:
```json
{
  "data": [ { "run_id": "...", "status": "...", ... }, ... ],
  "next_cursor": "eyJ0IjoxNzE0...",
  "has_more": true
}
```

Filters: `agent_id`, `status` (CSV), `created_after`, `created_before`, `metadata.{key}=value` (max 3 metadata filters per query, all ANDed). Sort is fixed: `created_at DESC`.

### 5.4 Control operations

All control operations require `runs:write` and are idempotent on the logical transition (re-pausing a paused run is 200 OK, not 409).

| Endpoint | Transitions from → to | Notes |
|---|---|---|
| `POST /runs/{id}/pause` | `running` → `paused` | Takes effect at next step boundary. Returns when the transition is acknowledged, not when it completes. |
| `POST /runs/{id}/resume` | `paused` → `running` | Optional body can override budget for remainder. |
| `POST /runs/{id}/cancel` | any non-terminal → `canceled` | Reason required in body: `{"reason": "user_request"}`. Terminal and irrevocable. |
| `POST /runs/{id}/inject` | `paused`, `awaiting_hitl` | Body is a message to inject into STM. Triggers resume. |
| `POST /runs/{id}/reply` | `awaiting_hitl` | Specifically for HITL responses — approval/denial/edit. |

Example inject:
```http
POST /v1/runs/run_01HX.../inject
{
  "message": {"role": "user", "content": "Actually, focus on APAC only."},
  "resume": true
}
```

### 5.5 Event stream (SSE)

```http
GET /v1/runs/{run_id}/events
Accept: text/event-stream
Last-Event-ID: evt_01HX...
```

Response:
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
X-Accel-Buffering: no

:ok

event: step.started
id: evt_01HX001
data: {"run_id":"run_01HX...","step_id":"stp_01HX...","step_index":3,"started_at":"..."}

event: planner.token
id: evt_01HX002
data: {"run_id":"...","step_id":"...","delta":"Looking"}

event: planner.token
id: evt_01HX003
data: {"run_id":"...","step_id":"...","delta":" at"}

event: tool.invoked
id: evt_01HX020
data: {"run_id":"...","step_id":"...","tool":"search_docs","args":{...},"action_id":"..."}

event: step.completed
id: evt_01HX040
data: {"run_id":"...","step_id":"...","status":"ok","duration_ms":1820}

event: run.completed
id: evt_01HX100
data: {"run_id":"...","status":"succeeded","output":{...},"cost":{...}}
```

**Reconnection semantics:** the client sends `Last-Event-ID` on reconnect. The server replays events with `id >` that value. Events are retained in the stream for 24 h after terminal state; after that, reconnect returns 410 Gone. Clients that need longer retention must use webhooks.

**Heartbeat:** `:hb` comment lines every 15 s when there are no events. Load balancers must be configured for idle timeout > 60 s.

**Filtering:** `?event_types=step.*,tool.*,run.*` to subscribe to a subset. Typed client libraries should default to subscribing to everything and filter locally — server-side filter is an optimization for thin clients.

### 5.6 Step and action inspection

```http
GET /v1/runs/{run_id}/steps?cursor=...
GET /v1/runs/{run_id}/steps/{step_id}
GET /v1/runs/{run_id}/steps/{step_id}/actions/{action_id}
GET /v1/runs/{run_id}/trace
```

These are read-only, paginated, cacheable. The trace endpoint returns the OTel-shaped span tree (see doc 09).

---

## 6. Agents API

### 6.1 Lifecycle

Agents are **versioned**. You update by creating a new version; you do not mutate a deployed version. This is non-negotiable because running agents may reference a specific version number, and eval baselines depend on immutability.

```http
POST /v1/agents                               # Create (version 1)
GET  /v1/agents/{id}                          # Get latest version
GET  /v1/agents/{id}@{version}                # Get a specific version
GET  /v1/agents/{id}/versions                 # List versions
POST /v1/agents/{id}/versions                 # Create new version
POST /v1/agents/{id}/versions/{v}/promote     # Mark a version as "production" alias
DELETE /v1/agents/{id}                        # Soft delete (archives; runs against it still work)
```

Create body (minimal):
```json
{
  "name": "Revenue Analyst",
  "description": "Summarizes revenue reports.",
  "instructions": "You are a ...",
  "model": {"provider": "anthropic", "model": "claude-opus-4-7"},
  "tools": ["search_docs", "sql_query"],
  "memory": {"ltm_enabled": true, "namespaces": ["analyst:global"]},
  "policies": ["pii_redact", "tool_allowlist"],
  "budget_defaults": {"max_tokens": 200000, "max_dollars": 2.0}
}
```

The server returns with a generated `agent_id`, `version: 1`, and a full config echo including defaults applied. The config is the source of truth — when you read it back, you get what the runtime will execute, not what you sent.

### 6.2 Aliases

Runs may reference `agt_01HX.../production` or `agt_01HX.../staging`. Alias resolution is snapshotted at run creation — promoting a new version mid-run does not affect in-flight runs.

---

## 7. Tools API

Tools are registered once, used by many agents.

```http
POST /v1/tools                                # Register a tool
GET  /v1/tools                                # List
GET  /v1/tools/{name}                         # Get manifest
PUT  /v1/tools/{name}                         # Update (creates new version)
DELETE /v1/tools/{name}                       # Deregister (soft)
```

The manifest shape is defined in doc 05. Key API concern: tool deregistration does **not** break running agents immediately. A deregistered tool is tombstoned; runs that reference it fail with a clear `tool_deregistered` error, not a generic 500.

---

## 8. Webhooks

For clients that cannot hold an SSE connection, or need durable delivery across long silences.

### 8.1 Subscription

```http
POST /v1/webhooks
{
  "url": "https://client.example.com/lyzr",
  "events": ["run.completed", "run.failed", "hitl.requested"],
  "filters": {"agent_id": "agt_01HX..."},
  "secret": "whsec_...",
  "active": true
}
```

### 8.2 Delivery

Outgoing webhook request:
```http
POST https://client.example.com/lyzr
Content-Type: application/json
User-Agent: Lyzr-Webhook/1.0
X-Lyzr-Event: run.completed
X-Lyzr-Delivery: whd_01HX...
X-Lyzr-Signature: t=1714000000,v1=3a8f...
X-Lyzr-Timestamp: 1714000000

{"event_type":"run.completed","run_id":"...","data":{...}}
```

Signature: HMAC-SHA256 over `timestamp.payload` using the subscription secret. Clients MUST verify and reject timestamps older than 5 minutes.

**Retry policy:** 7 retries over 24 h with exponential backoff (1m, 5m, 15m, 1h, 4h, 12h, 24h). After terminal failure, the delivery is recorded with status `failed` and can be replayed via `POST /webhooks/{id}/deliveries/{delivery_id}/redeliver`.

**At-least-once delivery.** Idempotency on the consumer side is required (use `X-Lyzr-Delivery` as dedupe key). We guarantee we don't drop; we do not guarantee we don't duplicate.

---

## 9. WebSocket: bidirectional sessions

WebSocket is used for **one thing**: bidirectional agent sessions where the client is a participant, not an observer. Example: a voice agent where the client streams audio in and receives tokens out. Do not use WebSocket for simple "watch a run" — that's SSE.

```
wss://api.lyzr.ai/v1/sessions/{session_id}/ws
```

Subprotocol: `lyzr.v1`. Messages are framed JSON envelopes:

```json
{"type": "user.message", "id": "msg_01HX...", "content": "..."}
{"type": "tool.response", "id": "...", "action_id": "...", "result": {...}}
{"type": "hitl.reply", "id": "...", "decision": "approve"}
```

Server → client:
```json
{"type": "planner.token", "delta": "..."}
{"type": "tool.request", "action_id": "...", "tool": "...", "args": {...}}   # when the tool lives on the client
{"type": "run.completed", "output": {...}}
```

The "tool lives on the client" pattern is important for mobile/desktop agents: the client registers a tool via the WS subprotocol, and when the runtime wants to invoke it, the request crosses back over the socket. The runtime waits (with timeout from the tool manifest) for the `tool.response` message.

---

## 10. Error responses

RFC 7807 `application/problem+json`:

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/problem+json
Retry-After: 12

{
  "type": "https://docs.lyzr.ai/errors/rate_limited",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "Tenant exceeded 1000 runs/minute.",
  "instance": "/v1/runs",
  "error_code": "rate_limited",
  "retryable": true,
  "trace_id": "trc_01HX...",
  "request_id": "req_01HX..."
}
```

### 10.1 Error code catalog (public)

| `error_code` | HTTP | Retryable | Description |
|---|---|---|---|
| `invalid_request` | 400 | no | Body failed schema validation. |
| `unauthenticated` | 401 | no | Missing/invalid credentials. |
| `forbidden` | 403 | no | Authenticated but missing scope or tenant access. |
| `not_found` | 404 | no | Resource does not exist (or not visible). |
| `conflict` | 409 | no | Idempotency key collision with different body, or version conflict. |
| `precondition_failed` | 412 | no | `If-Match` mismatch. |
| `payload_too_large` | 413 | no | Body > tenant cap (default 1 MB). |
| `unprocessable_entity` | 422 | no | Syntactically valid but semantically rejected (e.g., agent references missing tool). |
| `rate_limited` | 429 | yes | Tenant or global rate limit. `Retry-After` provided. |
| `tenant_quota_exceeded` | 429 | no | Monthly cap hit. Not retryable until billing cycle resets. |
| `internal_error` | 500 | yes | Unexpected server error. `trace_id` provided for support. |
| `service_unavailable` | 503 | yes | Planned downtime or overload. `Retry-After` provided. |
| `upstream_error` | 502 | yes | Upstream LLM/tool provider failure. |
| `timeout` | 504 | yes | Upstream timeout. |

Sync mode also surfaces **run-level terminal errors** at the HTTP layer:

| `error_code` | HTTP | Description |
|---|---|---|
| `run_budget_exhausted` | 200 (body status=failed) | Run hit budget cap. Not an API error; the run ran correctly, it just stopped. |
| `run_policy_blocked` | 200 | A policy guard terminated the run. |
| `run_tool_failed` | 200 | A non-recoverable tool error. |

Rule: API-layer failures = HTTP 4xx/5xx. Run-layer failures = HTTP 200 with `status: "failed"`. Don't conflate.

---

## 11. Pagination

**Cursor-based, opaque cursors, forward-only by default.** Cursors encode `(sort_key, id)`; they are base64 JSON signed with the tenant's cursor secret so tampering is detectable.

```http
GET /v1/runs?page_size=50
→ {"data": [...], "next_cursor": "eyJ0Ijo...", "has_more": true}

GET /v1/runs?page_size=50&cursor=eyJ0Ijo...
→ ...
```

- `page_size`: 1–200, default 50.
- Bidirectional traversal (`prev_cursor`) is provided only for endpoints where it's cheap. Listing runs does not support it.
- Cursors expire after 1 h. Stale cursor → `410 Gone`.

---

## 12. Idempotency

Applies to all `POST` that create resources. Client sends `Idempotency-Key: <ULID or UUID>`. Server stores `(tenant_id, idempotency_key, request_fingerprint, response)` for 24 h.

| Condition | Response |
|---|---|
| Key + identical body within 24 h | Cached response replayed, same status. |
| Key + different body | `409 Conflict` with `error_code: idempotency_key_reuse_conflict`. |
| Key, in-flight (previous call hasn't returned) | `409 Conflict` with `error_code: idempotency_key_in_flight`. Client should retry after delay. |
| No key provided | `400 invalid_request` on mutating endpoints (we **require** idempotency keys for creation). |

Rationale for requiring the key: duplicate submission under network flakes creates duplicate runs that burn budget. Mandatory idempotency is cheap insurance.

---

## 13. Rate limits

Three layers, documented:

| Scope | Default | Header when near limit |
|---|---|---|
| Per key | 60 rps, 1000 rpm | `X-RateLimit-Remaining`, `X-RateLimit-Reset` |
| Per tenant | 500 rps, 10 k rpm | `X-RateLimit-Tenant-Remaining` |
| Per endpoint (writes to a single run) | 20 rps | `X-RateLimit-Endpoint-Remaining` |

Rate limit errors are 429 with `Retry-After`. Budget-based limits (monthly cap) are separate and return `tenant_quota_exceeded` which is **not** retryable.

---

## 14. Content negotiation

- `Accept: application/json` — default.
- `Accept: text/event-stream` — required for SSE endpoints.
- `Accept: application/problem+json` — for structured errors; accepted on all endpoints.
- `Content-Encoding: gzip, br` — supported on both request and response bodies.

Requests without explicit `Content-Type: application/json` on JSON endpoints → `415 Unsupported Media Type`. Strictness here prevents XSS-via-content-sniffing.

---

## 15. OpenAPI and discovery

- Machine-readable OpenAPI 3.1 spec at `https://api.lyzr.ai/v1/openapi.json`.
- Human docs at `https://docs.lyzr.ai/api/v1`.
- The spec is generated from the server's route definitions — it's the **source of truth** for SDK generation, not hand-written.
- Breaking changes to the spec require a PR-level review and an explicit version bump.

Changelog feed: `GET /v1/changelog` returns recent additive changes, deprecations, and regional status.

---

## 16. SDK shape

The runtime ships first-party SDKs in Python and TypeScript. Other languages (Go, Ruby, Java) are community-supported with generated bindings from OpenAPI.

### 16.1 Python SDK

```python
from lyzr import Lyzr

client = Lyzr(api_key=os.environ["LYZR_API_KEY"])

# Async (default) — returns a handle, stream events
run = client.runs.create(
    agent_id="agt_01HX...",
    input={"messages": [{"role": "user", "content": "..."}]},
    budget={"max_dollars": 2.0},
    idempotency_key=str(ulid.new()),
)

for event in run.stream():
    match event.type:
        case "planner.token":
            print(event.delta, end="", flush=True)
        case "tool.invoked":
            log.info("tool %s", event.tool)
        case "run.completed":
            print(event.output)
            break

# Sync (blocking) convenience
result = client.runs.create_and_wait(agent_id="...", input={...}, timeout=60)
print(result.output)

# Control
client.runs.pause(run.id)
client.runs.inject(run.id, message={"role": "user", "content": "actually..."})
client.runs.cancel(run.id, reason="user_request")
```

### 16.2 TypeScript SDK

```typescript
import { Lyzr } from "@lyzr/sdk";

const client = new Lyzr({ apiKey: process.env.LYZR_API_KEY! });

const run = await client.runs.create({
  agentId: "agt_01HX...",
  input: { messages: [{ role: "user", content: "..." }] },
  budget: { maxDollars: 2.0 },
  idempotencyKey: crypto.randomUUID(),
});

for await (const event of run.stream()) {
  if (event.type === "planner.token") process.stdout.write(event.delta);
  if (event.type === "run.completed") {
    console.log(event.output);
    break;
  }
}
```

### 16.3 SDK conventions

- **Types are exhaustive.** Event discriminated unions mean the compiler flags a missing handler for a new event type after SDK upgrade.
- **Automatic retries** for 429/5xx with exponential backoff and jitter. Configurable per call.
- **Idempotency keys auto-generated** if not supplied (ULID).
- **Streaming is iterator-native** — `for event in run.stream()` / `for await`, no callback hell.
- **No hidden globals.** Every call accepts explicit `client` context; the SDK never reads env vars at call time.
- **Telemetry hooks** — SDKs emit OTel spans for every API call if an OTel SDK is present. Off by default, opt-in.

---

## 17. Versioning policy (the contract with integrators)

1. **Within v1:** no breaking changes. Ever. Fields may be added; enum values may be added (clients must handle unknown enum values, per the changelog guidance). Fields may be deprecated with `Sunset` header, but still returned for ≥ 12 months.
2. **New major (v2):** parallel deployment. v1 remains until usage is < 1% of request volume for 3 consecutive months, then removed with 6 months' final notice.
3. **Deprecation signaling:**
   - `Deprecation: true` header on responses using a deprecated feature.
   - `Sunset: Sun, 01 Oct 2027 00:00:00 GMT` header indicating removal date.
   - Weekly digest email to integrators with active usage.
4. **Preview endpoints:** `/v1-preview/...`. No stability guarantee. Must be explicitly opted into via `Lyzr-Preview: 2026-04-01` header. Features graduate to `/v1` after ≥ 90 days preview with ≥ 3 external users.

---

## 18. What the API deliberately does not expose

- **Step-internal state** (planner chain-of-thought, policy-redacted fields, raw provider responses) — available only via traces with `traces:read` scope, never in the public run envelope.
- **Memory internals** — records are read/deleted, but raw vectors and index metadata are not. Those are runtime concerns.
- **Queue state** (worker assignments, retry counts) — available only through traces for debugging.
- **Agent-internal prompts post-composition** — you see the config; you don't see the final assembled prompt. (It's in traces.)
- **Direct provider pass-through** — no `POST /v1/anthropic/messages`. If you want raw provider access, use their API directly. Mixing tiers defeats the runtime's purpose.

---

## 19. Failure modes at the API boundary

| Condition | What the client sees | What the runtime does |
|---|---|---|
| Client holds SSE too long; gateway kills idle conn | Stream closes; `Last-Event-ID` known | Reconnect replays from cursor. |
| Client disconnects mid-POST | Connection reset | Run already enqueued (if past the validation phase); client uses idempotency key to learn outcome. |
| Webhook endpoint returns 500 | Webhook retry schedule | Runtime retries with backoff; after 7 failures, alerts tenant admin. |
| Client submits stale agent_version | 422 with `agent_version_unavailable` | Runtime does not "fall back to latest" — explicit or nothing. |
| Tenant cap mid-run | SSE `run.failed` with `error_code: tenant_quota_exceeded` | Run is canceled at next step boundary; partial state preserved. |
| Regional outage | 503 on writes, 307 on reads (redirect to healthy region only if tenant is multi-region) | See doc 13. |

---

## 20. Summary

The API surface prioritizes one principle above all others: **the client should be able to write durable, correct integration code without reading runtime internals**. Idempotency keys make retries safe. Versioning guarantees give multi-month migration windows. Cursor pagination holds under concurrent writes. SSE + webhooks cover both interactive and durable delivery. Error responses are structured and categorized by retryability.

Integrators don't need to understand planning loops or memory consolidation to build on Lyzr. They need a small, consistent, well-documented surface. That's what this document defines.

---

Next: [13 — Deployment and Scaling](./13-deployment-and-scaling.md). We've defined what clients see; now we define how the runtime is packaged, shipped, and scaled.
