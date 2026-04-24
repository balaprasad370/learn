# 04 — Memory Systems

Agents without memory are chatbots. Memory is what makes them agents.

This doc defines the memory taxonomy, the storage primitives, the retrieval strategy, the write path, the forgetting policy, and the reading protocol during prompt assembly.

Reading prerequisite: [02 — Data Model](./02-data-model-and-contracts.md). The `MemoryRecord` type is assumed.

---

## Taxonomy — five kinds of memory

Memory is not one thing. The runtime distinguishes five kinds, each with distinct semantics, lifespans, and retrieval paths. Borrowing (loosely) from cognitive science because the names are clear, not because we claim biological fidelity.

| Kind | What it holds | Lifetime | Typical store | Written by | Read by |
|------|----------------|----------|---------------|-------------|---------|
| **Short-term (STM)** | The running conversation — recent messages + scratch | Run-scoped | Redis / in-process | Runtime | Planner every step |
| **Working** | Planner scratchpad — partial results, variables | Run-scoped | In-memory / state | Planner, executor | Planner |
| **Episodic** | Specific events that happened (runs, interactions) | Long | Vector + OLTP | Reflector, post-run hook | Planner via retrieval |
| **Semantic** | Facts the agent "knows" | Long | Vector + OLTP | Reflector, offline import | Planner via retrieval |
| **Procedural** | Skills: templates, patterns, rules of thumb | Long | OLTP + optional vector | Humans + reflector | Planner and executor |

You could collapse these into fewer boxes; you should not. Each has a different write trigger and retrieval strategy. Merging them hides bugs.

---

## Short-term memory (STM)

**What it is:** the messages of the current run plus the running summary and any "in-flight" context.

**Components:**
- `messages` — the raw Message list for this run. Storage: Postgres (durable) + in-memory window (hot).
- `running_summary` — a compressed text representation of messages older than the hot window. Storage: part of `AgentState`.
- `working_set` — a dict of typed, named values the planner has written. E.g. `{"candidate_urls": [...], "user_intent": "..."}`. Part of `AgentState`.

**Why split:** The raw message list is the ground truth for replay. The summary and working set are derived views optimized for prompt assembly.

### Windowing + summarization

The LLM context is finite. As messages accumulate, older ones are summarized. Policy:

```
Let recent_budget = max tokens for raw messages (e.g., 4000).
Let summary_budget = max tokens for running summary (e.g., 800).

On each step:
  window = last K messages whose cumulative tokens ≤ recent_budget
  older  = messages[:start_of_window]
  if older and we haven't summarized through start_of_window:
     summary = summarize_llm(older, existing_summary) → fits in summary_budget
```

### Summarization contract

The summarizer is an LLM call with its own prompt. It is **not** the planner. It is asked to:

- Preserve entities, decisions, and open questions.
- Drop small-talk and repeated content.
- Output in a compact, bulleted format.
- Never invent.

Summarizer input:
```
{
  "existing_summary": str | null,
  "new_messages": [ Message, ... ],
  "target_token_budget": int,
  "preserve_hints": [ "entities", "user_goals", "tool_results" ]
}
```

The summarizer's output replaces `running_summary`; the summarized messages are NOT deleted — they remain in Postgres for replay and can be re-summarized with different policies later.

### Working set

Typed entries in the working set survive for the run and nothing longer. Examples:
- `"user_location": "Bengaluru"`
- `"candidate_papers": [ {id, title, score}, ... ]`
- `"budget_remaining_at_last_check": 0.72`

Planners read/write via typed accessors:
```python
state.working.get("user_location", default=None)
state.working.set("candidate_papers", value, schema=CandidateListSchema)
```

Working set entries are **NOT** sent to the LLM implicitly. They are sent only when explicitly rendered into the prompt (usually via planner-specific template variables). This is an explicit-over-implicit choice.

---

## Long-term memory (LTM): episodic, semantic, procedural

LTM survives beyond a single run. All three subtypes share the `MemoryRecord` schema and differ by `kind` and retrieval strategy.

### Episodic

**Examples:**
- "On 2026-04-20, user asked about Bengaluru population; answer was 13.6M."
- "On 2026-04-21, tool `jira_create` failed with auth error for workspace X."

**Purpose:** the agent can say "last time we did X" instead of re-discovering it.

**Write trigger:**
- End-of-run hook: for each run, an LLM call decides which steps are worth remembering. Not every step. Writing everything is noise.
- Explicit write action: the planner can emit `Action{memory_op: write_episode(content=...)}`.

**Retrieval:** semantic (vector) + time filter (most recent N) + subject filter (about this user).

### Semantic

**Examples:**
- "The ACME corp uses PostgreSQL in production (stated by user, 2026-04-15)."
- "Customer X prefers morning meetings."

**Purpose:** durable facts. Retrieved whenever relevant to the current turn.

**Write trigger:** reflection loop after a run. Extract atomic facts from the run's trace, check for contradictions, write.

**Retrieval:** semantic, optionally with `subject_id` filter when the turn is about a specific entity.

**Contradiction handling:** when a new semantic record would contradict an existing one, the reflector marks the older one `deprecated_at=now` and writes the new one with `supersedes=old_id`. Both remain for audit; only the newer is retrieved by default.

### Procedural

**Examples:**
- "When the user asks for a weekly report, render it as markdown with sections: Summary, Highlights, Risks, Next."
- "Always escalate tier-1 tickets with severity >= 4 to the on-call."

**Purpose:** behavioral rules and templates that shape responses.

**Write trigger:** humans (admin UI) or explicit reflector decisions — rarely automatic.

**Retrieval:** by tag/namespace, not primarily semantic. These are usually small in number, so a tag-based fetch is cheaper and more predictable.

---

## Namespaces and scoping

Every long-term record has a **namespace** and a **scope**.

### Namespace
Free-form string used for partitioning. Examples: `"research"`, `"customer_support"`, `"engineering_on_call"`. One agent can read from multiple namespaces.

### Scope chain
```
global        — platform-wide (rare; e.g., "company-wide glossary")
tenant        — this customer
agent         — this specific agent config
subject       — about a specific entity (user id, account id)
run           — scoped to a single run (short-term only)
```

Retrieval walks the chain from narrow to broad. Narrow matches win ties. Example: if `tenant`-scope and `global`-scope each return a fact about "vacation policy," the tenant one wins.

---

## Storage layout

```
Postgres:
  memory_records (id, tenant_id, agent_id, subject_id, kind, namespace,
                  content, importance, access_count, last_accessed_at,
                  expires_at, metadata, created_at, supersedes,
                  deprecated_at, embedding_id)
  memory_embeddings (id, record_id, model, dim, vector)   -- or use pgvector column

Vector DB (pgvector OR external):
  index per (tenant_id, namespace) for hot paths
  metadata fields replicated for filtered search

Object store:
  optional content overflow when a record's content > 32 KB (rare; we prefer chunking)
```

### Why pgvector first
Simplifies deployment (no extra service), enables joint queries with filters (`WHERE tenant_id = $1 AND kind = 'semantic' AND deprecated_at IS NULL ORDER BY embedding <=> $vec LIMIT 10`), keeps transactional consistency with OLTP. We only move to Pinecone / Qdrant / Weaviate when index size or QPS demands it.

### Chunking

Long content is chunked before embedding. Strategy:

- **Recursive character splitter** with overlap (default: 512 tokens, 64 token overlap).
- Chunk boundaries respect sentence/paragraph ends when possible.
- Each chunk is its own MemoryRecord linked to a parent via `metadata.parent_id`.
- Retrieval returns chunks; the memory manager can "widen" a chunk into its parent if the planner requests more context.

---

## Embeddings

Embeddings are a concern of the LLM provider layer (see [07](./07-llm-provider-layer.md)). Memory only stores the vector plus the model that produced it.

**Rules:**
- The embedding model is versioned. Vectors from model A and model B are not interchangeable.
- Agents pin to an embedding model in config. Changing models requires re-embedding the namespace.
- Dimension mismatch at query time raises a loud error, not a silent empty result.

---

## The retrieval algorithm

For each step, the retrieval pipeline produces a list of `MemoryRecord`s to include in the prompt (see layer 3 of the prompt assembly in [03](./03-planning-and-execution.md)).

```
query = build_query(state)
# Hybrid: semantic + lexical for robustness to acronyms, ids, etc.

semantic_hits = vector_search(
    query.embedding,
    filters = { tenant_id, namespace in namespaces, kind in {semantic, episodic},
                deprecated_at is null, expires_at > now or null, subject_id match },
    top_k = 20,
    min_score = 0.6,
)

lexical_hits = bm25_search(query.text, same filters, top_k = 10)

fused = reciprocal_rank_fusion(semantic_hits, lexical_hits)

# Rerank with a small cross-encoder if configured (optional)
if reranker:
    fused = reranker.rerank(query.text, fused, top_k=10)

# Recency boost
fused = apply_recency_boost(fused, halflife_days=30)

# Cap at retrieval budget
return fused[: retrieval.top_k]
```

**Why hybrid:** pure semantic fails on identifiers ("ticket-4891") and rare tokens. Pure lexical fails on paraphrase. Both together cover the common failure modes.

**Reciprocal rank fusion (RRF)** is the cheap and correct way to merge:
```
score(doc) = Σ 1 / (k + rank_in_list_i(doc)),  k = 60 is fine
```

### Query construction

The query passed to retrieval is NOT the raw user input. It is a **query plan** built from:

1. Current user message (verbatim).
2. The planner's "thought" from the previous step, if any.
3. Named entities extracted from the conversation (fast NER or LLM).
4. Active goals from the working set.

All four are concatenated into the query text; the embedding is computed on this combined text. For hybrid search, the same text is tokenized for BM25.

### Filter precedence

When multiple filters apply, they compose as AND. For scope, the OR is done at the namespace list level (e.g., `namespace IN ('agent:x', 'tenant:y', 'global')`) and then scope precedence is applied as a post-sort, not inside the vector query (simpler, and vectors are already ranked).

---

## The write path

Writes happen in three places. Each has a distinct policy.

### Path 1: implicit write during a run (short-term → long-term bridge)

Not every run's content is worth remembering. A "salience filter" runs at end-of-run:

```
candidates = [ step for step in run.steps if is_notable(step) ]
                 # notable = user confirmed info, external fact retrieved, new preference stated, etc.

for c in candidates:
    record = draft_memory(c)        # LLM-aided distillation
    if record passes dedup check:
        write(record, kind=auto_classify(record))
```

De-duplication: before writing, query LTM with the new record's embedding; if an existing record has cosine similarity > 0.92 and same namespace/subject, skip the write (or merge metadata).

### Path 2: explicit write during a run

The planner can emit `Action{memory_op: write(...)}`. This happens when the planner is instructed to track things (e.g., "remember the user's preferences").

Explicit writes are subject to the same policy and dedup pipeline as implicit writes.

### Path 3: admin/bulk import

Humans upload documents, FAQs, policies. Pipeline:

```
file → chunker → embedder → writer → memory_records
```

Each upload has a `batch_id` in metadata for later rollback.

---

## Forgetting

Memory without forgetting bloats into noise. The runtime has three forgetting mechanisms.

### Expiration
`expires_at` is a hard TTL. Records past it are excluded from retrieval and garbage-collected by a background job.

### Deprecation
When a fact is superseded, the old record is marked `deprecated_at=now`. Retrieval excludes deprecated records by default. Audit/history can still see them.

### Importance-decay eviction
Each record has `importance ∈ [0,1]` and `access_count`. Periodic job scores:

```
score = importance * recency_factor(created_at) * log(1 + access_count)
```

Records below a threshold after the namespace hits a size cap are evicted. Eviction is *soft* (marked, moved to cold storage) for 90 days before hard-deletion.

**Never silently delete records that are referenced by a Run's trace.** Traces are replayable for 90 days; memory records written during replayable runs must remain visible.

---

## Per-turn memory "budget"

The retrieval pipeline's output is bounded by a **memory token budget** set per-agent. Typical default: 1500 tokens. The compiler picks records greedily by score until the budget is filled.

If a single record exceeds the budget, the compiler widens to fit one record and drops the rest, logging a `memory.budget_exceeded_single` event for observability.

---

## Consistency and concurrency

Two runs from the same user can race on writes. Policy:

- **Reads are eventually consistent.** A write made during Run A may not be visible to Run B's next step if it happened milliseconds ago — we don't block reads on writes.
- **Writes are serialized per (namespace, subject).** A lightweight Postgres row lock on a "namespace_subject_lock" table. Writers outside this scope don't serialize.
- **Dedup is best-effort.** Two simultaneous near-identical writes may both persist; the post-run reconciliation job later merges them.

---

## Memory operations surface (what the planner can do)

```python
class MemoryManager(Protocol):
    # reads
    async def query(self, q: MemoryQuery) -> list[MemoryHit]: ...
    async def get(self, record_id: MemoryId) -> MemoryRecord | None: ...
    async def list_namespace(self, ns: str, limit: int) -> list[MemoryRecord]: ...

    # writes
    async def write(self, draft: MemoryDraft) -> MemoryRecord: ...
    async def update(self, record_id: MemoryId, patch: MemoryPatch) -> MemoryRecord: ...
    async def deprecate(self, record_id: MemoryId, reason: str) -> None: ...

    # short-term (run-scoped)
    async def append_message(self, run_id: RunId, msg: Message) -> None: ...
    async def working_set(self, run_id: RunId) -> MutableMapping[str, Any]: ...
    async def refresh_summary(self, run_id: RunId) -> None: ...
```

The planner does not access Postgres or the vector DB directly. It calls the Memory Manager. This is a hard rule.

---

## Failure modes

| Failure | Symptom | Response |
|---------|---------|----------|
| Vector DB unavailable | Retrieval returns empty | Run continues with STM only. Log `memory.query_unavailable`. Degraded mode flag set. |
| Embedding API throttled | Write stalls | Queue writes; backoff. If queue overflows, drop lowest-importance writes first. |
| Dim mismatch | Exception at query | Fail loudly. Fix is operational (re-embed namespace). |
| Contradictory writes | New write disagrees with existing | Reflector resolves via `supersedes`; no silent overwrite. |
| Namespace not found | Empty result | Not a failure; returns empty. |
| Budget overflow on retrieval | Too many hits for prompt | Greedy truncation by score. Emit budget event. |

---

## Observability for memory

Every memory operation emits an event (see [09](./09-observability-and-tracing.md)). Per-run memory summary:

```
memory.queries.count
memory.hits.count
memory.hits.avg_score
memory.writes.count
memory.evictions.count
```

Plus per-query latency histograms.

**Debug view:** every `memory.retrieved` event carries the full list of hits with scores. The debugger UI shows which memories were included in which step, with a highlight diff showing how retrieval changed over the run.

---

## Eval hooks for memory

- **Retrieval regression tests:** given a fixed memory corpus, a set of queries should return a fixed top-K. Breaks loudly if an embedding model changes, a chunker changes, or fusion parameters change.
- **Dedup soak test:** write 10K near-duplicates; assert corpus size stays within 1.1× unique count.
- **Forgetting correctness:** after eviction, the evicted records must not appear in live retrieval; they must appear in cold-storage audit queries.

---

## The explicit non-feature list

These are **not** memory concerns and belong elsewhere:

- **Prompt caching.** Provider-level. See [07](./07-llm-provider-layer.md).
- **Tool output caching.** A tool can be configured idempotent and cacheable; that's [05](./05-tools-and-function-calling.md).
- **User profile.** A "user" is a subject; preferences are memory records. There is no special "user profile" table.
- **Knowledge base** (the marketing word for "documents you uploaded"). That is procedural/semantic LTM in a specific namespace. No separate system.

---

## Next: [05 — Tools and Function Calling](./05-tools-and-function-calling.md). The planner proposes tool calls; memory stores their results; now we define what a tool is and how it executes.
