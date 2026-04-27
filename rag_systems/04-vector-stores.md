# 04 — Vector Stores

> Where vectors live, and how you find the nearest ones in milliseconds instead of minutes.

A vector store is two things glued together: a **storage layer** (vectors + payload + filters) and an **ANN index** (algorithm that returns approximate nearest neighbors fast). Get either wrong and your system either lies (low recall) or stalls (high latency).

---

## 1. Why you need an index at all

Brute-force search is `O(N · d)` per query. For 1M chunks × 1024 dims = ~4ms on a modern CPU using SIMD. Fine.

For 100M chunks = ~400ms per query. Already painful.
For 1B chunks = 4+ seconds. Dead.

**Approximate Nearest Neighbor (ANN)** indexes trade a small amount of recall (typically 95–99% of true nearest) for 100–1000× speedup. Every production vector DB is ANN under the hood.

The ceiling on quality you set in doc 03 (the right embedding) is now bounded again by the index's recall. A 99%-recall index loses 1% of true top-K results. If that 1% contains your actual answer, you've capped accuracy you can never recover.

---

## 2. ANN algorithms — the four you need to know

### 2.1 HNSW (Hierarchical Navigable Small World) — the default

A multi-layer graph. Each vector is a node connected to its M nearest neighbors. Top layer is sparse (long jumps), bottom layer is dense (local refinement). Search greedily descends layers.

**Knobs:**
- `M` (graph degree, typically 16–64): higher → better recall, more RAM.
- `ef_construction` (build-time candidate list, typically 100–500): higher → better graph, slower build.
- `ef_search` (query-time candidate list, typically 50–500): higher → better recall, higher latency. **Tune this per query.**

**Properties:**
- 95–99% recall at sub-10ms latency on 10M vectors with reasonable RAM.
- Index lives in RAM. Storage ≈ vectors + ~M × 4 bytes per node (~50% overhead for M=32).
- **Updates are expensive but supported** in modern implementations (Qdrant, Weaviate, pgvector 0.6+). Deletes are tombstones until rebuild.
- Simple to operate. The default for almost every production system.

### 2.2 IVF (Inverted File) — partitioned

Cluster vectors into `nlist` Voronoi cells (k-means at build time). At query, find the `nprobe` nearest cells and brute-force within them.

**Knobs:**
- `nlist` (cell count, often `sqrt(N)`): more cells → smaller cells → faster search but worse recall.
- `nprobe`: how many cells to scan. Higher → better recall, slower.

**Properties:**
- Cheaper to build than HNSW. Lower memory.
- Worse recall/latency frontier than HNSW for most workloads.
- Good when vectors don't fit in RAM (combine with PQ — see below).
- FAISS's bread and butter.

### 2.3 PQ (Product Quantization) — compression

Not an index, but commonly combined with IVF. Split the vector into `m` sub-vectors, train a small codebook (e.g., 256 centroids) for each. Store each chunk as `m` byte codes.

A 1024-dim float32 vector (4 KB) becomes 64 bytes at m=64. **64× compression.**

**Cost:** approximate distances (lookup table of squared distances). Recall drops 5–15% vs. raw vectors. Mitigated by reranking top-K with full vectors (which you keep on disk).

Use IVF-PQ when you have hundreds of millions to billions of vectors and RAM is the bottleneck.

### 2.4 ScaNN (Google), DiskANN (Microsoft) — disk-friendly

- **ScaNN** — Google's algorithm, anisotropic vector quantization. Strong recall/latency. Used in Vertex AI Vector Search.
- **DiskANN** (Vamana graph) — designed to run from SSD with a small RAM footprint. Powers Azure AI Search, Milvus's DiskANN index. Right when corpus is huge and RAM-bound.

You won't usually pick these directly — you pick a DB that uses them.

### Recall vs. latency vs. RAM — the three-way tradeoff

| Index | Recall (typ.) | Latency (10M, 1024d) | RAM | Build time |
|---|---|---|---|---|
| Brute force (Flat) | 100% | ~50ms | low (vectors only) | none |
| HNSW (M=32, ef=100) | 98–99% | 2–5ms | high (graph in RAM) | minutes–hours |
| IVF-Flat (nprobe=10) | 90–95% | 5–10ms | medium | fast |
| IVF-PQ | 85–93% | 5–15ms | low (compressed) | fast |
| DiskANN | 95–98% | 3–8ms | low (most on SSD) | medium |

**Pick HNSW unless you have a reason not to.** The reasons: corpus too big for RAM (→ DiskANN/IVF-PQ), or write workload too high for HNSW updates (→ IVF-Flat).

---

## 3. Filters — the silent quality killer

Real RAG always filters: `tenant_id = X`, `language = en`, `published_after = 2024-01-01`, `doc_type IN ('blog','docs')`. The DB has to combine ANN search with structured filters.

Three approaches:

| Strategy | How | Pro | Con |
|---|---|---|---|
| **Pre-filter** | Filter first, then brute-force the survivors | Exact, simple | Slow if filter is selective (you scan everything) |
| **Post-filter** | ANN first, then filter the top-K | Fast | If filter is selective, you may return <K results — recall collapse |
| **Filter-aware ANN** | Index integrates filter conditions during traversal | Fast + correct | Requires DB support |

**Production rule:** if your queries always filter by `tenant_id`, the DB needs to know. Either:
- Use a filter-aware index (Qdrant payload filtering, Weaviate filtered HNSW, pgvector with `WHERE`).
- Shard by tenant (one collection per tenant if N is small; namespace by tenant in a single collection if N is huge).
- Pre-filter and accept the scan if the tenant has few docs.

A common mode of failure: post-filter with `top_k=10` and a selective filter returns 1 result. The index thought 10 vectors were close; 9 didn't match the filter. **Fix:** either over-fetch (top_k=100 then filter), or move to a filter-aware DB.

---

## 4. The vector DB landscape

Three categories: **add to your existing DB**, **dedicated vector DB**, **managed cloud service**.

### 4.1 In-DB extensions

#### pgvector (PostgreSQL)
- ANN: HNSW (since 0.5.0) and IVFFlat.
- Filters: native SQL `WHERE` — true filter-aware retrieval.
- Strengths: you already run Postgres. Joins, transactions, backups, ACLs all just work. One database to operate.
- Limits: scales well to ~10M vectors per node before HNSW build time and RAM bite. Beyond that, partition or move.
- **Default pick for any project under 10M chunks** if you have Postgres.

```sql
CREATE EXTENSION vector;
CREATE TABLE chunks (
  id BIGSERIAL PRIMARY KEY,
  tenant_id UUID,
  embedding vector(1024),
  text TEXT,
  metadata JSONB
);
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops)
  WITH (m = 32, ef_construction = 200);
CREATE INDEX ON chunks (tenant_id);

SELECT id, text, 1 - (embedding <=> $1) AS score
FROM chunks
WHERE tenant_id = $2
ORDER BY embedding <=> $1
LIMIT 20;
```

#### Other in-DB
- **Elasticsearch / OpenSearch** — `dense_vector` field with HNSW, plus full-text BM25 in the same query. Great for hybrid search out of the box.
- **MongoDB Atlas Vector Search** — HNSW, integrates with regular MongoDB. Simple if you already use Atlas.
- **Redis (RediSearch)** — vectors with HNSW or FLAT. Good for low-latency / cache-tier retrieval.
- **ClickHouse** — has vector indices (HNSW). Strong if your retrieval lives next to analytics.

### 4.2 Dedicated open-source vector DBs

#### Qdrant (Rust)
- HNSW with strong filter integration ("payload filtering"). Filter-aware traversal — fast and correct.
- Quantization: scalar (int8), binary, product. Live tunable.
- Multi-tenancy via collections or payload index. Strong RBAC.
- gRPC + REST. Solid Python/JS SDKs.
- **Operational sweet spot:** 10M–1B vectors, multi-tenant SaaS. Picks for: lean ops, high QPS, complex filters.

#### Weaviate (Go)
- HNSW + flat. Modules for embedding generation (run inside the DB).
- Strong on hybrid search, BM25 + dense.
- GraphQL + REST API. Multi-tenancy first-class.
- Quirks: schema-first; you define classes upfront. Good when your data model is clear.

#### Milvus (C++ / Go)
- HNSW, IVF-Flat, IVF-PQ, DiskANN, GPU indices (RAFT). Most flexible index choice.
- Distributed by default — separate compute and storage. Designed for **billions** of vectors.
- Heavier ops than Qdrant. Worth it at very large scale.
- Zilliz Cloud is the managed offering.

#### Vespa (Java)
- Yahoo's engine. Vectors + tensors + structured + text in one query language.
- HNSW, sophisticated multi-stage ranking pipelines.
- Steepest learning curve. Right for "we are building search-as-a-product."

#### Chroma, LanceDB, Marqo, Vald
- **Chroma** — embedded, Python-first. Great for prototypes, notebooks, single-host.
- **LanceDB** — embedded, columnar (Lance format). Good for ML pipelines that already use Arrow/Parquet. Cheap on-disk, good for TB-scale.
- **Marqo** — opinionated stack, runs embedding + index together.
- **Vald** — Kubernetes-native, used heavily in Japan.

### 4.3 Managed cloud / serverless

#### Pinecone
- Closed-source SaaS. Two products:
  - **Serverless** — pay per query + storage. Auto-scales. No index tuning.
  - **Pod-based** — old model, fixed-size pods.
- Strengths: simple. You don't think about HNSW knobs. Filters, namespaces, hybrid (sparse-dense).
- Limits: no self-host. Cost scales with query volume — can get expensive at heavy QPS.
- **Default pick when you don't want to operate infrastructure.**

#### Vertex AI Vector Search (Google) / Azure AI Search / AWS OpenSearch Serverless
- Cloud-vendor managed. Convenient if you're already in their ecosystem.
- Vertex uses ScaNN under the hood. Strong recall/latency, supports streaming updates.
- Azure AI Search: hybrid (vector + BM25 + semantic ranker) tightly integrated. Good for enterprise with M365 data.

#### Turbopuffer
- Newer entrant, object-storage-backed (S3). Aggressive cost/$ for cold corpora.
- Right when most queries hit a small hot subset and you want to keep huge cold storage cheap.

---

## 5. Pick a vector store — a decision flow

```
START: How many chunks, end of year 2?
    < 1M  → Postgres + pgvector. Done.
    1M–10M → Postgres + pgvector if you already have Postgres.
             Otherwise Qdrant or Pinecone Serverless.
   10M–100M → Qdrant (self-host) or Pinecone Serverless or Weaviate.
             Postgres possible but partitioning gets fiddly.
   100M–1B+ → Milvus, Vespa, Vertex Vector Search, or partitioned Qdrant.

Sub-decision: filters?
    Heavy structured filtering → Qdrant, pgvector, or Elasticsearch.
    Mostly unfiltered           → any.

Sub-decision: hybrid (dense + sparse)?
    Critical for your domain → Weaviate, Elasticsearch, OpenSearch, or Qdrant.
    Optional / external rerank → any.

Sub-decision: ops appetite?
    Zero ops → Pinecone Serverless / Vertex / Azure AI Search.
    Light    → Qdrant Cloud or Weaviate Cloud.
    Heavy OK → self-host anything.
```

A first-time RAG project with <5M chunks should default to **pgvector** unless there's a hard reason otherwise. Most "we need a real vector DB" claims don't survive contact with reality at small scale.

---

## 6. Schema design

Treat your vector record as **vector + structured payload + raw text + provenance**.

```python
{
  "id": "uuid",
  "vector": [...1024 floats...],
  "text": "the chunk text actually shown to the LLM or returned in citations",
  "metadata": {
      "tenant_id": "...",
      "doc_id": "...",
      "doc_title": "...",
      "doc_url": "...",
      "section_path": ["Chapter 3", "3.2 Configuration"],
      "page": 47,
      "language": "en",
      "doc_type": "manual",
      "created_at": "2025-08-12T...",
      "updated_at": "2026-01-02T...",
      "embed_model": "voyage-3",
      "embed_version": "2024-09",
      "chunk_strategy": "recursive-512-50",
      "checksum": "sha256:..."
  }
}
```

**Why every field matters:**
- `tenant_id`, `language` — filter scopes.
- `doc_id`, `section_path`, `page` — citations and "show source."
- `embed_model`, `embed_version`, `chunk_strategy` — re-embed migrations and A/B testing.
- `checksum` — incremental re-ingestion (skip unchanged chunks).
- `updated_at` — freshness ranking, TTL eviction.

**Index your filter fields.** A vector DB without secondary indices on payload will table-scan to filter. Qdrant: declare payload indexes per field. pgvector: regular Postgres indexes on the metadata columns.

---

## 7. Hybrid storage (dense + sparse) inside the DB

If you fuse BM25 + dense (you should — see doc 05), you have two options:

**Option A — one DB does both:** Elasticsearch/OpenSearch, Weaviate, Qdrant (named vectors with sparse), Vespa. Single query returns fused results.

**Option B — two systems:** Postgres for BM25 (`tsvector`) + pgvector for dense, fuse in app code. More moving parts but works fine and keeps both layers debuggable.

Pinecone supports sparse-dense in one vector — store BM25 (or SPLADE) sparse + dense in the same record. Single API call.

---

## 8. Multi-tenancy patterns

Three strategies, in order of isolation:

| Pattern | Example | Pro | Con |
|---|---|---|---|
| **Field on row** | `WHERE tenant_id = X` (pgvector, Qdrant payload) | Simple, cheap | One bug = data leak; noisy neighbor possible |
| **Collection per tenant** | One Qdrant collection per tenant | Strong isolation | Doesn't scale to 100k tenants (collection overhead) |
| **Namespace** | Pinecone namespace, Qdrant payload index | Logical isolation, scales | Still shared physical resources |
| **Cluster per tenant** | Separate DB per enterprise customer | Total isolation | Most expensive, hard to operate |

For SaaS at 100s–10ks of tenants, the standard is **field on row + filter-aware index + per-tenant secondary index**. Wire the tenant_id into every read at the application layer (use a query helper, not raw queries).

For a handful of large enterprise tenants, **collection or cluster per tenant** is cleaner and lets you set per-tenant quotas/HNSW params.

---

## 9. Operations checklist

### Capacity
- Rough RAM for HNSW (rule of thumb): `vectors × dim × 4 bytes × 1.5` (graph overhead).
  - 10M × 1024 × 4 × 1.5 ≈ **60 GB**. That's one big box, not a cluster.
  - 100M × 1024 = **600 GB** → quantization or sharding mandatory.
- Use int8 vectors if you can — 4× RAM savings, ~1–3% recall hit.

### Backups
- Vectors are derived data. **Always** keep the source corpus + chunking config separately. Worst case you re-ingest.
- But re-ingesting 100M chunks is days of work + embedding cost. Snapshot the DB state regularly.

### Updates
- Pure appends are cheap.
- Updates: usually delete + insert. HNSW handles this; quality degrades after lots of churn → periodic rebuild.
- **Soft delete with tombstones** is normal during a re-embedding migration; hard-delete after cutover.

### Dual-write during migrations
- New embedding model → new collection. Dual-write all new chunks to both. When the new collection is fully backfilled, flip reads. Keep old around for rollback for a week. Then drop.

### Monitoring
- **Recall@K on a held-out eval set** — track this on every deploy. The single best leading indicator that your index is healthy.
- **p50/p95/p99 query latency** — separate by `nprobe` / `ef_search` if you tune dynamically.
- **Index size, RAM, disk** — alert before they bite.
- **Filter selectivity** — if many queries return <K results, you have a post-filter bug.

### Sharding
- Most DBs shard by hash of vector ID (random) or explicit field (tenant).
- Cross-shard ANN is "fan out, gather, merge top-K." Latency is max of shards, not sum.
- Don't shard prematurely. A single Qdrant node handles 100M+ vectors comfortably with right hardware.

---

## 10. Latency budget

For a typical RAG retrieve-then-rerank flow:

| Stage | Budget (ms) |
|---|---|
| Embed query | 30–80 (API) / 5–20 (self-host) |
| ANN search top-100 | 5–30 |
| Filter / fetch payload | 5–20 |
| Rerank top-100 → top-10 (cross-encoder) | 100–400 |
| Build prompt | 5 |
| LLM generation (streaming first token) | 200–1500 |
| **Total to first token** | **350–2000ms** |

The vector store is rarely the bottleneck — embedding API or LLM TTFT dominates. **But if your DB latency is >50ms on a 10M-chunk index, something is misconfigured** (wrong index, no warmup, cold disk, or post-filter scanning).

---

## 11. Common failure modes

| Symptom | Likely cause |
|---|---|
| Right docs are in DB, never retrieved | Wrong distance metric; missing `input_type`/prefix; index trained on different vector version |
| Recall craters after deploy | New embedding model with old index OR you re-built HNSW with too low `ef_construction` |
| Multi-tenant data leaks across tenants | Filter applied in app code only; index scans across all tenants and post-filters |
| Top-K returns <K results | Post-filter on selective predicate; over-fetch and re-filter |
| Latency spikes after a writes-heavy hour | HNSW deletes piling up; trigger compaction/rebuild |
| Same query, different results between calls | ANN non-determinism on tied scores; usually fine, but log query+result hash if you need reproducibility |
| Costs balloon on serverless | Re-embed migration ran without guardrails; or query path is calling the index per turn for chat history |

---

## 12. Keyword cheat-sheet

- **ANN** — approximate nearest neighbor.
- **HNSW** — hierarchical navigable small world graph. The default ANN.
- **IVF** — inverted file. Cluster-then-search.
- **PQ** — product quantization. Compress vectors to byte codes.
- **DiskANN / Vamana** — disk-resident graph index.
- **ScaNN** — Google's index used in Vertex.
- **Flat / brute-force** — no index, exact search.
- **Recall@K** — fraction of true top-K returned. The quality KPI.
- **ef_search / nprobe** — query-time recall/latency knobs.
- **Filter-aware index** — combines filter and ANN in traversal, doesn't post-filter.
- **Namespace / collection / index** — DB-specific terms for a logical bucket of vectors.
- **Payload** — non-vector data attached to a record (text, metadata).
- **Hybrid search** — fusing dense + sparse (BM25/SPLADE) results.
- **Quantization** — fp16 / int8 / binary / PQ at the storage layer.
- **Sharding** — splitting an index across nodes.

---

## 13. What you take into doc 05

You now have a place to store vectors and a way to find the nearest ones. The next question is **what query to embed in the first place**. Most retrieval failures are not "the index missed it" — they're "we embedded the wrong query."

The next doc is the most leverage-rich layer of the entire pipeline.

Next: [05 — Retrieval Strategies](./05-retrieval-strategies.md)
