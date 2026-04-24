# 07 — LLM Provider Layer

The runtime must be provider-agnostic. Not out of idealism — out of survival. Providers change pricing, rate-limit, go down, deprecate models, and ship incompatible tool-call formats. This layer is the moat.

Reading prerequisite: [02 — Data Model](./02-data-model-and-contracts.md) and [03 — Planning](./03-planning-and-execution.md).

---

## What this layer does

1. **Translates** between the runtime's internal chat/tool/embedding format and each provider's native format.
2. **Streams** tokens and tool-call fragments uniformly.
3. **Counts** tokens and dollars per call.
4. **Retries** on transport errors with correct backoff.
5. **Routes** a given request to a provider+model by policy.
6. **Caches** where safe and useful.
7. **Falls back** when a provider is unavailable.
8. **Normalizes** safety/refusal signals.

What it does NOT do: decide WHAT to send. Prompt construction lives in the planner/compiler.

---

## The internal contract

Every provider implements this interface. Nothing above this layer knows what provider it is talking to.

```python
class LLMProvider(Protocol):
    provider_id: str                          # "anthropic", "openai", "vertex", ...
    supported_models: set[str]

    async def chat(self, req: ChatRequest) -> ChatResponse: ...
    def stream_chat(self, req: ChatRequest) -> AsyncIterator[ChatStreamChunk]: ...
    async def embed(self, req: EmbedRequest) -> EmbedResponse: ...
    async def count_tokens(self, req: TokenCountRequest) -> int: ...
    async def health(self) -> HealthStatus: ...
    def supports(self, capability: Capability) -> bool: ...
```

### `ChatRequest` (normalized input)

```python
class ChatRequest(BaseModel):
    model: str                                # provider's model id
    messages: list[Message]                   # runtime's canonical messages
    system: str | None                        # provider-agnostic system block
    tools: list[ToolSchema]                   # normalized function schemas
    tool_choice: ToolChoice                   # auto | required | none | {"name": ...}
    response_format: ResponseFormat | None    # text | json | json_schema
    sampling: SamplingParams                  # temp, top_p, max_tokens, stop
    stream: bool
    metadata: dict[str, str]                  # provider-side tags (tenant, run_id)
    cache_hint: CacheHint | None              # let provider cache prefix if supported
    timeout_seconds: int
    seed: int | None                          # for providers that support deterministic sampling
```

### `ChatResponse` (normalized output)

```python
class ChatResponse(BaseModel):
    id: str
    provider_id: str
    model: str
    finish_reason: FinishReason               # stop | length | tool_use | content_filter | error
    message: Message                          # the assistant's reply
    tool_calls: list[ToolCall]                # may be empty
    usage: Usage                              # prompt_tokens, completion_tokens, cache tokens
    cost_usd: Decimal
    raw_provider_id: str                      # provider's own response id for support cases
    safety_signals: list[SafetySignal]        # refusals, blocks, redactions
    latency_ms: int
```

### `ChatStreamChunk` (normalized stream)

A provider stream yields a sequence of these:

```python
class ChatStreamChunk(BaseModel):
    kind: Literal[
        "message_start", "content_delta", "tool_call_delta",
        "tool_call_end", "message_end", "usage", "error"
    ]
    delta: Any
    index: int                                # for interleaved tool calls
    at: datetime
```

The stream adapter emits canonical chunks regardless of how the provider's native stream looks (Anthropic's `message_delta`/`content_block_delta` events, OpenAI's `delta.tool_calls[]` slices).

---

## Provider adapters

One adapter per provider. Each is a shallow module:

```
providers/
  anthropic/
    client.py          # HTTP + auth + retries
    translate.py       # normalize ↔ native
    stream.py          # native stream → canonical stream
    pricing.py         # model → $/token
  openai/
    client.py
    translate.py
    stream.py
    pricing.py
  gemini/
    ...
  bedrock/
    ...
  local/
    ollama.py
    vllm.py
```

### Translation rules

The adapter is the only place "provider-specific knowledge" lives. Everything above sees canonical types.

**Example — tool-call translation (Anthropic → canonical):**
```
{ "type": "tool_use", "id": "toolu_01abc", "name": "web_search",
  "input": {"query": "Bengaluru pop"} }
→
ToolCall(id="toolu_01abc", tool_id="web_search@1.2.0", args={"query": "Bengaluru pop"})
```

Note the `tool_id` resolution — the adapter resolves the registry version from `name` at translate-time; this is where we catch LLM hallucinations of tool names.

**Example — stream normalization:**

OpenAI yields:
```
{ "delta": { "tool_calls": [ { "index": 0, "function": { "arguments": "\"que" } } ] } }
```

Adapter yields:
```
ChatStreamChunk(kind="tool_call_delta", index=0, delta={"args_chunk": "\"que"})
```

The runtime's stream consumer does not know or care which provider produced the delta.

---

## Tool-schema translation

Providers all accept JSON-Schema-like function definitions but with subtle differences.

| Feature | Anthropic | OpenAI | Gemini | Bedrock (Claude) |
|---------|-----------|--------|--------|------------------|
| JSON Schema draft | 2020-12 (subset) | 2019-09 (subset) | OpenAPI-ish | Same as Anthropic |
| Enum support | yes | yes | partial | yes |
| Tuple schemas | no | no | no | no |
| Oneof / anyOf | limited | limited | limited | limited |
| Parallel tool calls | yes | yes (gpt-4o+) | yes (1.5+) | yes |
| Tool required | `tool_choice` | `tool_choice` | function_calling_config | `tool_choice` |

The runtime's canonical `ToolSchema` uses JSON Schema 2020-12 as the lingua franca. The adapter downgrades or normalizes for providers that don't fully support it; if a feature cannot be represented, the adapter either:

- **Degrades gracefully** (e.g., drop a `oneOf` and flatten to the union), logging `schema.downgraded`, or
- **Refuses the request** with an explicit error (`capability_missing`) if the degrade would change semantics.

Silent degrades are banned. Choose between graceful-with-log or loud-fail at adapter-authoring time, per feature, and write it down.

---

## The Router

The router picks a provider + model for a given `ChatRequest`.

```python
class Router(Protocol):
    async def resolve(
        self,
        request: ChatRequest,
        agent: Agent,
        ctx: RouterContext,
    ) -> ResolvedCall:
        ...
```

### Resolution order
1. If the agent pins a specific `(provider, model)`, use it.
2. Else, consult the **model alias** table: e.g., `"reasoning-high"` → `(anthropic, claude-opus-4-7)` this week.
3. Check provider health. If primary is unhealthy, try fallbacks in order.
4. Check capability: does the model support tools? streaming? vision? If not, route to a capable model or fail.

### Health check
A background probe hits each provider every 30 seconds (tiny request, cheap). If error-rate-over-sliding-window exceeds threshold, provider is marked `degraded`; after a second threshold, `unhealthy`. Unhealthy providers are bypassed in routing for a cool-down period.

### Fallback rules
Agents can declare fallbacks in config:
```yaml
planner:
  model: claude-opus-4-7
  fallbacks:
    - gpt-5
    - claude-sonnet-4-6
```

Fallback is attempted on:
- `unhealthy` status on the primary.
- Persistent 5xx from primary after retries.
- Context length exceeded on primary (route to a longer-context model).

Fallback is NOT attempted on:
- Content policy refusal (that is a genuine signal, not a transport issue).
- 400-level validation errors (our request is malformed — don't paper over it).

Every fallback emits `llm.fallback` with reason.

---

## Retries and backoff

At the provider adapter level:

```
Retry on: 429, 500, 502, 503, 504, connection errors, read timeouts
Do not retry: 400, 401, 403, 404 (request-shape problems — fix upstream)
               content_policy_refusal
Backoff: jittered exponential, base 0.5s, cap 8s, max 3 attempts
After final failure: raise mapped error to router; router decides fallback.
```

Retries consume budget (tokens of successful retries are counted; failed attempts are not). If a provider bills on failures, the adapter's pricing module reflects that.

---

## Rate limit handling

Providers expose rate limits per API key / project. The adapter:

- Reads rate-limit headers if provided (`x-ratelimit-remaining-requests`, etc.).
- Maintains a local view of remaining quota to avoid hitting the wall.
- Implements a **token bucket** per (provider, model, tenant) and admits requests when tokens are available; else queues up to a cap.
- On 429 with `retry-after`, waits precisely that long (with jitter).

The router sees rate-limit pressure as part of `health()`; at high pressure, routing distributes load to alternate models.

---

## Streaming protocol — end-to-end

```
API client → (SSE)   ← API server
                      ← event bus
                      ← worker emitting canonical stream chunks
                      ← provider adapter emitting canonical chunks
                      ← provider native stream
```

The worker consumes the provider's stream and emits `llm.stream` events. SSE subscribers receive them live. Chunks are also appended to a buffer; once `message_end` arrives, the full response is assembled and validated.

**If the stream drops mid-response:**
- Worker marks the call `error`; retries once from scratch (streams aren't resumable cross-providers).
- Partial content already streamed to the client is annotated `{truncated: true}`; the client can decide to wait for retry or show the truncated state.

**If the client disconnects mid-response:**
- Worker does NOT stop; streaming to disk continues. A late reconnecting client receives buffered chunks.

---

## Embeddings

```python
class EmbedRequest(BaseModel):
    texts: list[str]
    model: str
    input_type: Literal["query", "document"]  # some providers differentiate
    dimensions: int | None                    # some providers allow truncation

class EmbedResponse(BaseModel):
    model: str
    vectors: list[list[float]]
    usage: Usage
    cost_usd: Decimal
```

Embedding calls go through the same router + retry stack but use a separate rate-limit bucket (providers typically meter them separately from chat).

---

## Token counting

Three ways to know token count:
1. **Provider-reported** — authoritative, only available after the call.
2. **Model-specific tokenizer** — bundled (e.g., `tiktoken` for OpenAI, Anthropic's official counter) for pre-call budget checks.
3. **Character-based heuristic** — fallback for unknown models: `tokens ≈ chars / 3.5` (tune per language).

Budget pre-checks use (2) or (3); the authoritative (1) settles the books after the call. If the pre-check underestimated by more than 15%, a `token_estimate_drift` event fires for calibration.

---

## Cost accounting

The pricing table is a config file shipped with the adapter:

```yaml
anthropic:
  claude-opus-4-7:
    input_per_mtok: 15.00
    output_per_mtok: 75.00
    cache_read_per_mtok: 1.50
    cache_write_per_mtok: 18.75
```

On each response, `cost_usd = (input_tokens * rate + output_tokens * rate + cache_tokens * rate) / 1_000_000`. Cache-read and cache-write are separate line items.

Pricing tables are versioned by effective date. A historical run's cost is computed at the pricing of its `run.created_at`, not current pricing — preserves audit correctness when pricing changes.

---

## Caching

Three distinct caches live at or under this layer.

### 1. Prompt cache (provider-native)
Anthropic and (less so) OpenAI support prompt caching with prefix-based reuse. The adapter:
- Marks cache breakpoints via `cache_control` blocks in Anthropic's format.
- Chooses stable prefixes: identity layer + capability layer + long-term memory layer — everything that changes per-turn sits AFTER the breakpoint.

The compile-prompt function knows the cache model and emits messages with breakpoints inserted.

### 2. Response cache (runtime-level, optional)
Cache by `hash(canonicalized ChatRequest)` → `ChatResponse`. Useful for:
- Eval reruns (make test suites deterministic and cheap).
- Deterministic planner steps (temperature=0, same inputs).

**Default: off.** Enabled explicitly per-agent or per-test. Never auto-cache production planner calls — too easy to poison with a stale response.

### 3. Embedding cache
Text → vector is pure and deterministic per-model. Cache aggressively by `hash(text) + model_id`. TTL: effectively infinite (clear only on model version change).

---

## Idempotency

Provider APIs sometimes accept an idempotency key (e.g., OpenAI's `idempotency_key` header). The adapter injects one generated from the runtime's `request_id` (a ULID per `ChatRequest`). On retry, the same key goes up — providers dedupe server-side, sparing us double-billing in the "response arrived but connection dropped" case.

---

## Safety and refusal handling

Providers sometimes refuse or redact. Canonical handling:

- **Soft refusal** (e.g., the LLM's own "I can't help with that" text): surfaced as content. The planner sees it and decides whether to retry with a rephrase or end the run.
- **Provider block** (e.g., a safety header saying the request was blocked without a completion): mapped to `finish_reason = content_filter`, `safety_signals = [...]`. The runtime does NOT auto-retry; emits `llm.blocked` and surfaces to policy.

Policy may (e.g.) allow a retry with redaction; see [10 — Safety and Governance](./10-safety-and-governance.md).

---

## Deterministic mode

For eval and replay, the runtime supports a "deterministic mode" per call:

- `temperature = 0`
- `seed = <stable>`
- `stream = false`
- Response cache enabled for the call

Providers that genuinely support deterministic sampling with seeds produce byte-stable outputs; others produce semantically stable outputs which may still replay OK for eval purposes. Determinism is best-effort across providers — documented limitation.

---

## Model registry

```python
class ModelRegistry(Protocol):
    def get(self, model_id: str) -> ModelInfo: ...
    def list(self, capability: Capability | None = None) -> list[ModelInfo]: ...

class ModelInfo(BaseModel):
    id: str
    provider_id: str
    family: str                                # "claude-4", "gpt-5", ...
    context_window: int
    max_output_tokens: int
    capabilities: set[Capability]              # {"tools", "stream", "vision", "json_mode"}
    pricing_ref: str                           # -> pricing table entry
    effective_from: datetime
    deprecated_at: datetime | None
```

Model aliases (`"reasoning-high"`, `"fast-cheap"`) map to model ids via a config file, updated by ops. Agents that pin an alias are insulated from the specific model behind it — ops can swap `"reasoning-high"` from `claude-opus-4-7` to `claude-opus-5-0` on the release day with no agent config changes.

---

## Local / self-hosted models

Adapters for `ollama`, `vllm`, `tgi`, and `transformers-inference`:

- Translate to OpenAI-compatible APIs where possible (most local runtimes expose this).
- Pricing = 0 unless self-hosting cost is configured.
- Tokenizer bundled with the model (not a heuristic) for accurate counts.
- Streaming supported.

**Limitations:**
- Tool-call quality and JSON-mode conformance vary — runtime should be able to fall back to "function-call-by-prompt" (parse a structured response from plain text) when native tool calling is unreliable.

---

## Observability

Per call emitted events:
- `llm.request` (redacted prompt, model, params)
- `llm.stream` (chunks, for streaming)
- `llm.response` (tokens, cost, latency, finish_reason)
- `llm.error` (on failure)

Metrics:
- `llm.latency.ms` (histogram, labels: provider, model)
- `llm.tokens.prompt` / `llm.tokens.completion` (counter, labels)
- `llm.cost_usd` (counter)
- `llm.errors.count` (counter, labels: provider, code)
- `llm.fallback.count` (counter)
- `llm.cache.hit_rate` (gauge)

---

## Error-code mapping (cross-provider)

Provider-specific errors normalize to canonical `ErrorCode`:

| Canonical | Anthropic | OpenAI | Retryable |
|-----------|-----------|--------|-----------|
| `rate_limited` | 429 | 429 | yes |
| `context_length` | invalid_request with "prompt too long" | `context_length_exceeded` | no — fallback to longer model |
| `auth_failed` | 401 | 401 | no |
| `content_filter` | `stop_reason=content_filtered` | `content_filter` finish reason | no |
| `overloaded` | 529 | 500 | yes |
| `transport_error` | network | network | yes |
| `bad_request` | 400 | 400 | no |

The adapter owns this mapping. Upstream code writes decisions against canonical codes only.

---

## Versioning the layer itself

This layer evolves when providers change. Rules:

- Add new providers behind a feature flag; dark-launch with shadow traffic before enabling by routing.
- Adapter protocol version is bumped when the internal contract (`ChatRequest`, `ChatResponse`) changes. All adapters must implement the new version or be quarantined.
- Pricing tables are effective-dated — past runs remain correctly costed.

---

## Next: [08 — Runtime Infrastructure](./08-runtime-infrastructure.md). We've defined what an agent does and how it talks to models; now: where it runs, how it persists, and how it scales.
