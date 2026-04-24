# 05 — Tools and Function Calling

A tool is an externally callable function the agent can invoke mid-reasoning. Tools are how the agent stops being a text generator and starts being useful.

Reading prerequisite: [02 — Data Model](./02-data-model-and-contracts.md) (`Tool`, `ToolCall`, `ActionResult`) and [03 — Planning](./03-planning-and-execution.md) (how the planner emits a `ToolCall`).

---

## The anatomy of a tool

A tool is five things:

1. **Identity** — a globally-unique id (`web_search@1.2.0`), a human name, a version.
2. **Contract** — input schema, output schema, error catalog, side-effect class.
3. **Description** — natural-language text the LLM reads to decide *whether* and *how* to call.
4. **Transport** — how we actually execute it (in-process, HTTP, MCP, message queue).
5. **Policy** — rate limits, timeouts, auth requirements, allowlist membership.

If any of the five is missing, the tool does not exist. Half-defined tools are the single most common source of silent agent failures.

---

## Tool definition

```python
class Tool(BaseModel):
    id: str                            # "<name>@<semver>"
    name: str                          # slug, [a-z0-9_]{3,40}
    version: str                       # semver
    description: str                   # LLM-facing — see "The description hierarchy" below
    input_schema: JsonSchema
    output_schema: JsonSchema
    error_catalog: list[ErrorCode]
    side_effects: Literal["none", "read", "write", "external_write"]
    idempotency_key_expr: str | None   # e.g., "args.request_id"
    timeout_seconds: int               # hard deadline
    max_retries: int                   # per invocation
    retry_on: list[ErrorCode]
    rate_limit: RateLimit | None
    cost_hint: CostHint | None         # for budget pre-check
    transport: ToolTransport
    transport_config: dict
    auth_ref: str | None               # secret reference in the vault
    observability: ObservabilityConfig
    deprecated_at: datetime | None
    tags: list[str]
```

### `input_schema` and `output_schema`

Both are JSON Schema draft 2020-12. They are the *canonical* contract — the LLM's function-calling schema is derived from them, not the other way around.

**Rules:**
- Every property has a `description` — this is what the LLM sees for arg meaning.
- No `additionalProperties: true`. Ever. That is how you ship prompt-injection-as-tool-args.
- Nullable fields are `{"type": ["string", "null"]}`, not `"type": "string"` with a sentinel.
- Enums are closed — `{"enum": [...]}`.
- Output schema must validate every possible return value, including error shapes.

### `side_effects` — tells the runtime what it can retry

- `none`: pure function (e.g., `add(a, b)`). Parallelizable, cacheable, retryable freely.
- `read`: observes external state but changes nothing (`web_search`, `get_user_by_id`). Retryable; cache with TTL.
- `write`: changes internal state the runtime owns (`write_note`). Retryable iff idempotency key present.
- `external_write`: changes external state (`send_email`, `create_jira_ticket`). Retry only if the external system guarantees idempotency from our key.

Planners can call `none` and `read` tools in parallel. `write` and `external_write` serialize on resource identity.

### `idempotency_key_expr`

A JMESPath expression evaluated against `args` to produce an idempotency key. The invoker dedupes repeated calls with the same key within a TTL window. Required for `external_write`; otherwise optional.

Example: `"request_id"` → use `args.request_id` as the key. If the planner re-calls `send_email(request_id="abc", ...)` after a network hiccup, the second call short-circuits to the first call's result.

---

## The description hierarchy — why this field decides everything

The `description` is the single most important field. Every other field is machine-readable; `description` is the LLM's entire window into the tool's purpose.

A good description answers five questions:

1. **What does this tool do?** One sentence.
2. **When should the agent call it?** Triggers and use cases.
3. **When should it NOT?** Non-use cases.
4. **What do the args mean?** Per-arg gotchas.
5. **What does it return?** Output shape and caveats.

**Good:**
```
web_search: Search the public web for current information. Use this when
the user asks about events, facts, or entities the model may not know
(after your knowledge cutoff). Do NOT use for mathematical computation
or code generation — use the `python` or `calculator` tool instead.

Args:
  query (str): A concise search query, 3–10 words ideal.
  time_range (str?): ISO date range filter, e.g., "2025-01-01..2026-01-01".
  num_results (int, default 5): 1..10.

Returns: a list of {title, url, snippet, published_at} items,
ordered by relevance. Empty list if no matches.
```

**Bad:**
```
Searches the web.
```

Rule: a tool description short enough to skim is a tool the LLM will misuse.

---

## Tool Registry

A central, versioned, tenant-aware registry.

```python
class ToolRegistry(Protocol):
    async def register(self, tool: Tool) -> None: ...
    async def get(self, tool_id: str) -> Tool: ...
    async def list_for_agent(self, agent: Agent) -> list[Tool]: ...
    async def deprecate(self, tool_id: str, reason: str) -> None: ...
    async def search(self, q: str, tags: list[str]) -> list[Tool]: ...
```

**Versioning rules:**
- A Tool is immutable once registered. Changes = new version.
- An agent pins tool versions (e.g., `web_search@^1`). The registry resolves to the highest compatible at agent-publish time; the resolved version is frozen on the `Agent` record.
- Deprecation emits a warning to agents still pinning the old version. After a grace period, pinned-to-deprecated tools fail agent validation at publish time.

---

## Transports

A tool's code must run somewhere. The transport determines where.

### `inproc`
The tool function is a Python callable registered in the worker process.

- **Pros:** fastest (no serialization), easiest to build for MVP.
- **Cons:** ships with the worker; upgrading a tool = redeploying the worker.
- **Use for:** low-volatility platform tools (math, string ops, internal lookups).

### `http`
The tool is an HTTP endpoint; the invoker sends a signed JSON request.

- **Pros:** language-agnostic, independently deployable, easy to scale per-tool.
- **Cons:** network overhead, requires cert management, authn/z.
- **Use for:** customer tools, microservices, third-party integrations wrapped in an adapter.

### `mcp` (Model Context Protocol)
Tool lives on an MCP server; the runtime connects to the server on tool-registration and multiplexes invocations over the MCP session.

- **Pros:** standard protocol, easy to adopt existing MCP ecosystems, richer capability discovery.
- **Cons:** another protocol to operate; early ecosystem, tooling still settling.
- **Use for:** connector ecosystems (Slack, Gmail, Notion, etc.) where someone else provides the MCP server.

### `queue`
Tool is invoked asynchronously via a queue; the runtime waits on the result topic.

- **Pros:** long-running tools (minutes to hours), batching, backpressure.
- **Cons:** orchestration is harder, pause/resume semantics non-trivial.
- **Use for:** document pipelines, long ETL, human-in-the-loop review steps.

---

## The tool invoker

The invoker is the single component that executes a `ToolCall` and returns an `ActionResult`. It has ~200 lines of strict contract:

```python
class ToolInvoker(Protocol):
    async def invoke(self, call: ToolCall, ctx: InvocationContext) -> ActionResult: ...
```

### Step-by-step

```
1. RESOLVE
   tool = registry.get(call.tool_id)
   if tool.deprecated_at:
       emit warning, continue

2. AUTHORIZE
   policy.check_tool_call(tool, call, ctx)
   # may reject here → ActionResult.ok=false, error.code=policy_violation

3. VALIDATE INPUT
   jsonschema.validate(call.args, tool.input_schema)
   # failure → ActionResult.ok=false, error.code=invalid_args
   # this is a "planner bug" — feed back into next step for repair

4. IDEMPOTENCY CHECK
   if tool.idempotency_key_expr:
       key = eval(expr, call.args)
       cached = idemp_cache.get(tool.id, key)
       if cached: return cached.result

5. RATE LIMIT CHECK
   if rate_limiter.exceeded(tool.id, ctx.tenant): 
       backoff or reject → ActionResult.ok=false, error.code=rate_limited

6. BUDGET CHECK
   if budget.cannot_afford(tool.cost_hint):
       return ActionResult.ok=false, error.code=budget_exceeded

7. EXECUTE
   with timeout(tool.timeout_seconds):
       with observability.span("tool.invoke", attrs=...):
           raw = await transport.call(tool, call.args, ctx)
   on timeout → error.code=tool_timeout, retryable if not idempotent-external
   on transport error → map to ErrorCode; decide retryable

8. VALIDATE OUTPUT
   jsonschema.validate(raw, tool.output_schema)
   failure → error.code=invalid_output
   this is a "tool bug" — NOT retryable; surface to planner as-is

9. RETRIES
   if error.retryable and retries < tool.max_retries:
       backoff(retries); goto 7

10. CACHE
    if tool.idempotency_key_expr:
        idemp_cache.put(tool.id, key, result, ttl)

11. EMIT + RETURN
    emit tool.completed (or tool.failed)
    return ActionResult
```

Every step emits events. The invoker is the only component that produces `tool.*` events.

---

## Parallel invocation

When the planner returns multiple calls marked `parallelizable=true`, the invoker runs them concurrently under a semaphore.

```python
async def invoke_many(calls: list[ToolCall], ctx: InvocationContext) -> list[ActionResult]:
    sem = asyncio.Semaphore(ctx.max_parallel)
    async def one(c):
        async with sem: return await self.invoke(c, ctx)
    return await asyncio.gather(*(one(c) for c in calls), return_exceptions=False)
```

**Constraints:**
- Side-effect class `write` / `external_write` calls against the same resource serialize via a mutex keyed on `(tool_id, idempotency_key)` — no interleaved writes.
- Per-tenant concurrency cap prevents one tenant from starving others.
- A failing parallel call does not cancel siblings. All results flow back to the planner next step.

---

## Handling tool results

Tool results flow back as `role=tool` messages, one per `tool_call_id`. The next planner iteration sees:

```
assistant: [tool_calls: [tc_1 web_search("..."), tc_2 fetch_url("...")]]
tool (id=tc_1): { "items": [...] }
tool (id=tc_2): { "status": "404 Not Found" }
```

The planner is expected to handle partial failures — it sees both results and adjusts.

---

## Error handling: error codes, not strings

The error catalog is a closed enum. Flow-control decisions never parse `error.message`.

```
invalid_args           — schema validation failed on input; planner bug
invalid_output         — schema validation failed on output; tool bug
tool_timeout           — wall-clock timeout exceeded
tool_rejected          — tool's business logic refused (returned ok=false)
rate_limited           — local or remote rate limit hit
auth_failed            — credential or permission issue
policy_violation       — policy engine rejected
transport_error        — network/5xx; retryable per provider adapter
upstream_unavailable   — dependent service unreachable
budget_exceeded        — would consume more than remaining budget
unknown                — last resort; logged with full stack
```

Each has a `retryable: bool` property that the invoker consults.

Example — `auth_failed` is NOT retryable (the creds won't fix themselves in 2 seconds); `transport_error` IS retryable; `tool_timeout` is retryable if `side_effects in {none, read}`, otherwise only if an idempotency key is present.

---

## Authentication and secrets

Tools NEVER see raw secrets. The tool config references a secret by name (`auth_ref: "google_calendar_oauth"`). At invocation time:

```
secret = vault.resolve(ctx.tenant_id, tool.auth_ref)
adapter.inject(request, secret)   # transport-specific injection
```

**Rules:**
- Secrets are tenant-scoped.
- Vault access is logged; the secret value is never logged.
- Rotated secrets take effect on the next invocation; in-flight invocations keep their resolved value.
- A tool without a configured secret, where one is required, fails at `authorize` step with `auth_failed`.

---

## Streaming tool outputs

Most tools return a complete response. Some stream (e.g., `run_sql` returning rows, `subscribe_events`). The transport layer supports an optional streaming return; the invoker exposes an async iterator to the executor.

**Semantics:**
- Streamed chunks are emitted as `tool.stream` events.
- The final assembled value is validated once at stream end.
- If the planner did not request streaming, and the tool streams, the invoker buffers and validates normally. (Streaming is opt-in at the planner level.)

---

## Observability

Every tool invocation emits a `tool.invoked` event at start and one of `tool.completed | tool.failed | tool.retried` at end, wrapped in an observability span.

**Span attributes:**
- `tool.id`, `tool.version`
- `tool.transport`
- `tool.args_hash` (NOT raw args — those may be PII)
- `tool.side_effects`
- `tool.retries`
- `tool.duration_ms`
- `tool.tenant_id`

Raw args and outputs are persisted to the Step record (with PII redaction) and may be separately persisted to object storage if over a threshold.

---

## Caching

Two distinct caches:

### Idempotency cache
Deduplicates repeated calls within a TTL. Keyed on `(tool_id, idempotency_key)`. Purpose: correctness (don't send two emails). TTL: hours.

### Result cache
Caches `(tool_id, canonicalized_args)` → result for read/none tools when configured. Purpose: latency + cost reduction. TTL: tool-specific (seconds to days).

**Never cache `write` or `external_write` results as results.** They go through the idempotency cache only.

---

## MCP bridge (detail)

The runtime supports MCP servers as first-class tool sources. At startup, for each configured MCP server:

```
1. Connect via MCP transport (stdio, HTTP+SSE, WebSocket).
2. Handshake; negotiate protocol version.
3. Call `tools/list`; receive tool descriptors.
4. For each descriptor, synthesize a Tool record in the registry with:
     - id = "<server_id>/<tool_name>@<version>"
     - transport = mcp
     - transport_config = { server_id, mcp_tool_name }
     - input/output schema from MCP.
5. Keep the session open; heartbeat.
```

Invocation routes through the MCP session — the transport adapter translates a `ToolCall` into an MCP `tools/call` request and maps the response back.

**Health:** a disconnected MCP server marks its tools as temporarily unavailable. Tools scheduled against an unavailable server fail with `upstream_unavailable` and the planner sees the failure.

---

## Sub-agents as tools

A sub-agent appears to the planner as a tool with special shape: its `input_schema` is the child agent's `RunInput` schema, its `output_schema` is the child's output schema. Calling it is a `Handoff` action (see [06](./06-multi-agent-coordination.md)), not a `ToolCall` — but the LLM-facing surface is similar.

Why distinguish:
- Handoffs deduct from run budget differently (child has its own sub-budget).
- Handoffs produce a child `Run` record with its own trace.
- Handoff side-effects are the union of the child's tool side-effects — policy inspects recursively.

---

## Failure modes and responses

| Failure | Detected where | Response |
|---------|----------------|----------|
| Tool schema not found | Registry resolve | Planner bug; inject correction; next step |
| Invalid args | Invoker step 3 | `invalid_args`; planner retries with correction |
| Tool times out | Step 7 | `tool_timeout`; retry if safe, else surface |
| Output doesn't match schema | Step 8 | `invalid_output`; NOT retryable; surface |
| Rate limit | Step 5 | `rate_limited`; backoff or surface |
| Auth failed | Step 7 | `auth_failed`; halt or surface; escalate to ops |
| Transport down | Step 7 | `transport_error`; retry, then surface |
| Policy rejects | Step 2 | `policy_violation`; emit event; halt if severity=high |

---

## Testing a tool in isolation

- **Contract tests:** golden input/output pairs that assert both schemas are valid and match.
- **Timeout tests:** run with a sleeping stub; assert timeout fires and error code is `tool_timeout`.
- **Idempotency tests:** call twice with the same key; assert one execution, two cached results.
- **Concurrency tests:** call in parallel with overlapping keys; assert mutex ordering is correct.
- **Failure injection:** inject each `ErrorCode` and assert the invoker's response matrix.

---

## Governance for tools

- **Allowlists per agent** — an agent sees only tools explicitly listed in its config.
- **Denylists per tenant** — platform-admin can globally disable a tool for a tenant.
- **Permission scopes** — a tool can declare required scopes; the principal starting the run must have them.
- **Audit** — every invocation writes a row to `tool_invocations_audit` with tenant, agent, principal, args hash, result status. Retained for compliance period.

---

## Naming and design rules (hard-won)

- **Verbs, not nouns.** `fetch_url`, not `url`. `search_customers`, not `customer_search`.
- **One tool, one purpose.** If an arg named `mode` switches behaviors, make two tools.
- **Return structured data, not prose.** The LLM can summarize; the tool should not.
- **Include metadata.** `created_at`, `source`, `confidence` — the planner can cite and reason better.
- **Prefer idempotent shapes.** "Upsert" beats "create/update" pairs.
- **No hidden pagination.** Return a `next_cursor` or accept `limit`; don't decide silently.

---

## Next: [06 — Multi-agent Coordination](./06-multi-agent-coordination.md). A tool is one-off; a sub-agent is another agent. That turns out to be a very different problem.
