# RAG Systems — From 0 → 100% Accuracy

A self-contained learning track on Retrieval-Augmented Generation: why it exists, what the components are, when to use it (vs alternatives), and how to actually build one — including the explicit accuracy-improvement playbook that takes a hello-world RAG (~30% accuracy) to a production system (~90%+).

Written for an engineer who already understands LLMs at the level of `D:\jobs\llm_fundamentals\`. If you don't, read that folder first — RAG is a pattern *on top of* LLMs, and the failure modes only make sense once you understand tokenization, context, and inference.

---

## The four questions, answered up front

### Why RAG?

LLMs have three hard limits:

1. **Knowledge cutoff.** A model trained through Oct 2024 doesn't know April 2026 events.
2. **Private knowledge.** Your company's docs, policies, customer history, internal tools — never in any pretraining corpus.
3. **Hallucination.** When the model doesn't know, it makes up plausible answers with high confidence.

RAG fixes all three by **letting the model look things up at query time** instead of relying on memorized weights. You retrieve the most relevant passages from a knowledge store, paste them into the prompt, and instruct the model to answer using only those passages with citations.

### What is RAG?

RAG = **Retrieval** (find the relevant content) + **Augmented Generation** (give it to an LLM as context). Two systems glued together:

```
       ┌──────────────────────────┐
       │   Your knowledge base    │   PDFs, docs, code, tickets, DB rows
       └──────────────┬───────────┘
                      │
                      ▼
              ┌───────────────┐
              │   Ingestion   │   Parse, chunk, embed, index
              └───────┬───────┘
                      ▼
              ┌───────────────┐
              │  Vector store │   Searchable index
              └───────┬───────┘
                      │
   user query ───►   ▼
              ┌───────────────┐
              │   Retriever   │   Find top-K relevant chunks
              └───────┬───────┘
                      ▼
              ┌───────────────┐
              │   Reranker    │   Reorder by true relevance
              └───────┬───────┘
                      ▼
              ┌───────────────┐
              │  Prompt build │   Stuff context into the prompt
              └───────┬───────┘
                      ▼
              ┌───────────────┐
              │      LLM      │   Generate grounded answer + citations
              └───────────────┘
```

Eight stages, each with its own design decisions and failure modes. This folder walks every stage.

### When to use RAG?

| Situation | Use RAG? | Alternative |
|---|---|---|
| Question depends on **private** or **frequently-updated** data | ✅ Yes | Fine-tuning gets stale; long-context is expensive per call. |
| Knowledge fits in **< 50k tokens** total | ⚠️ Maybe | Just stuff it all in context. Skip the retrieval complexity. |
| Need to learn **new behavior or style** (not facts) | ❌ No | Fine-tune or DPO. |
| Need **structured reasoning over a small fixed dataset** | ❌ No | SQL / API / function call. RAG over a 10-row table is silly. |
| Question is **ambiguous, multi-step, or agentic** | ⚠️ RAG-as-tool | Wrap RAG inside an agent (doc 09). |
| Need answers that **cite sources** for trust/audit | ✅ Yes | Long-context + careful prompting can also work. |
| Need **freshness** (live data, latest news) | ✅ Yes | RAG with regular re-indexing. Fine-tuning is hopeless. |

Default: if the answer requires content the model wasn't trained on **and** that content is large or changing, RAG. Otherwise, simpler patterns.

### How to from 0 → 100%?

A naive RAG built in a weekend will hit ~30–40% accuracy on a moderately hard QA dataset. Production-grade RAG hits 85–95%. The gap is closed by **layered improvements**, each measured. Doc 10 has the full playbook; the short version:

| Stage | Lift | Cost | Effort |
|---|---|---|---|
| Naive vector RAG (chunk → embed → cosine → stuff) | baseline 30–40% | low | 1 day |
| + Hybrid search (BM25 + vector + RRF) | +10–15% | low | 1 day |
| + Reranking (cross-encoder or LLM) | +10–15% | latency cost | 1 day |
| + Better chunking (semantic, parent-child) | +5–10% | medium | 2–4 days |
| + Query rewriting / HyDE / multi-query | +5–10% | LLM cost | 2 days |
| + Metadata filtering & query routing | +5–10% | medium | 3–5 days |
| + Contextual retrieval (chunk-level context) | +5–10% | indexing cost | 2 days |
| + Domain embedding fine-tuning | +5–15% | one-time | 1–2 weeks |
| + Evaluation harness + iterative loop | compound | ongoing | forever |

Stack them all and you go from 30% → 90%+. Skip evaluation and you'll never know which step did what.

---

## What you'll be able to answer after reading this folder

1. What is the difference between an embedding and a token? Between a bi-encoder and a cross-encoder? (docs 01, 03, 06)
2. How do you choose between fixed-size, recursive, semantic, and parent-child chunking? (doc 02)
3. Which embedding model should I use for English vs multilingual vs code? Why does dimension matter? (doc 03)
4. What's the difference between FAISS, pgvector, Pinecone, Qdrant, Weaviate, Milvus? When does each win? (doc 04)
5. Why does hybrid search beat pure vector search? What's RRF? (doc 05)
6. What is HyDE? When does query rewriting hurt instead of help? (doc 05)
7. Why is reranking the highest-leverage single addition to most RAG systems? (doc 06)
8. How do you measure RAG accuracy without an LLM judge biasing results? (doc 08)
9. What is GraphRAG, agentic RAG, contextual retrieval — and when does each pay off? (doc 09)
10. My RAG works on tests but fails in production — how do I diagnose where the breakdown is? (doc 10)

---

## Reading order

| # | Title | What you'll learn | Time |
|---|---|---|---|
| 01 | [Foundations](./01-foundations.md) | What RAG is precisely; the retrieval/generation split; RAG vs fine-tune vs long-context. | 30 min |
| 02 | [Ingestion and Chunking](./02-ingestion-and-chunking.md) | Parsing PDFs/HTML/code/tables; chunking strategies; metadata; OCR; layout-aware parsing. | 40 min |
| 03 | [Embeddings](./03-embeddings.md) | What embeddings are; popular models; dimensions; cosine vs dot vs L2; MTEB; fine-tuning. | 35 min |
| 04 | [Vector Stores and Indexing](./04-vector-stores.md) | pgvector, FAISS, Pinecone, Qdrant, Weaviate, Milvus, Chroma, LanceDB. HNSW, IVF, PQ. | 40 min |
| 05 | [Retrieval Strategies](./05-retrieval-strategies.md) | BM25, semantic, hybrid, RRF, HyDE, multi-query, query decomposition, self-querying. | 50 min |
| 06 | [Reranking and Context Construction](./06-reranking-and-context.md) | Cross-encoders, ColBERT, Cohere/Voyage rerankers, lost-in-the-middle, context packing. | 35 min |
| 07 | [Generation, Prompting, Citations](./07-generation-and-citations.md) | Prompt patterns, citation formats, hallucination control, structured outputs. | 30 min |
| 08 | [Evaluation](./08-evaluation.md) | Recall@k, MRR, NDCG, RAGAS, faithfulness, context precision/recall, golden datasets. | 45 min |
| 09 | [Advanced Patterns](./09-advanced-patterns.md) | Agentic RAG, Self-RAG, CRAG, GraphRAG, hierarchical, contextual retrieval, multimodal. | 50 min |
| 10 | [Production Playbook (0 → 100%)](./10-production-playbook.md) | The accuracy improvement roadmap, failure-mode catalog, latency/cost/security/ops. | 60 min |

Total: ~7 hours end-to-end. Build a hello-world RAG alongside doc 02–05 and you'll absorb 3× more.

---

## Glossary — every keyword, in one place

Skim this; come back to it as you read.

**Retrieval / search:**
- **RAG** — Retrieval-Augmented Generation.
- **Retriever** — the system that finds top-K relevant chunks for a query.
- **BM25** — Best Matching 25, the canonical lexical (keyword) retrieval algorithm.
- **TF-IDF** — Term Frequency × Inverse Document Frequency. BM25's predecessor.
- **Lexical / sparse search** — keyword-based retrieval; matches surface terms.
- **Semantic / dense / vector search** — embedding-based retrieval; matches meaning.
- **Hybrid search** — combines lexical and semantic, fuses with RRF or weighted scores.
- **RRF** — Reciprocal Rank Fusion. Standard hybrid-fusion technique.
- **Top-K** — number of results retrieved per query (e.g., K=10).
- **Recall@K** — fraction of relevant documents found in the top K.
- **MRR** — Mean Reciprocal Rank.
- **NDCG** — Normalized Discounted Cumulative Gain.
- **Hit Rate** — % of queries with at least one relevant doc in top K.

**Embeddings:**
- **Embedding** — a vector (typically 384–3072 floats) representing meaning.
- **Embedding model** — model that turns text into an embedding (text-embedding-3-large, BGE, E5, Voyage, Cohere Embed, etc.).
- **Bi-encoder** — encodes query and document separately; cheap; standard for retrieval.
- **Cross-encoder** — encodes query+doc together; expensive but accurate; standard for reranking.
- **Late interaction (ColBERT)** — middle ground; per-token embeddings with late comparison.
- **Cosine similarity / dot product / L2 distance** — distance metrics between embeddings.
- **MTEB** — Massive Text Embedding Benchmark; the standard leaderboard.
- **SPLADE** — learned sparse embeddings (best of both worlds: keyword + semantic).
- **Matryoshka embeddings** — embeddings that work at multiple truncated dimensions.

**Indexing / vector stores:**
- **Vector store / vector database** — system that stores embeddings and supports nearest-neighbor search.
- **HNSW** — Hierarchical Navigable Small World; the dominant ANN algorithm.
- **IVF** — Inverted File Index; partitions vectors into clusters.
- **PQ** — Product Quantization; lossy compression of vectors for memory savings.
- **ScaNN** — Scalable Nearest Neighbors (Google).
- **FLAT** — exact (brute-force) search; baseline for accuracy.
- **ANN / k-NN** — Approximate / exact Nearest Neighbor search.
- **pgvector** — Postgres extension for vector search (default for many).
- **FAISS** — Facebook AI Similarity Search; library, not a database.
- **Pinecone, Qdrant, Weaviate, Milvus, Chroma, LanceDB** — managed/embeddable vector DBs.

**Chunking:**
- **Chunk** — a piece of a source document, sized to be embedded and retrieved.
- **Fixed-size chunking** — equal-character chunks with overlap.
- **Recursive chunking** — split on natural boundaries (paragraphs, sentences) recursively.
- **Semantic chunking** — split where embedding similarity drops (topic shifts).
- **Sentence-window** — index sentences; return surrounding context window.
- **Parent-child / small-to-big** — index small chunks; return larger parent chunks for context.
- **Hierarchical chunking** — multi-level summaries.
- **Late chunking** — embed the whole doc once, then derive chunk embeddings (preserves cross-chunk context).
- **Overlap / stride** — bytes/tokens shared between adjacent chunks.

**Query strategies:**
- **HyDE** — Hypothetical Document Embeddings; LLM writes a fake answer; embed *that* and retrieve.
- **Multi-query** — LLM rewrites the query into N variants; union the results.
- **Query expansion** — add synonyms or related terms.
- **Query decomposition** — break complex query into sub-queries.
- **Query rewriting** — reformulate ambiguous queries.
- **Self-querying retriever** — LLM extracts metadata filters from the query.
- **Step-back prompting** — generate a more general question first.
- **Routing** — classify query type and dispatch to the right retriever / data source.

**Reranking:**
- **Reranker** — second-pass scorer that reorders top-K results by true relevance.
- **Cross-encoder reranker** — small transformer that scores (query, doc) pairs.
- **LLM reranker** — use a generic LLM to rerank.
- **Cohere Rerank, Voyage Rerank, BGE-reranker, ms-marco-MiniLM** — popular models.

**Generation / output:**
- **Grounding** — making the answer reference the retrieved evidence.
- **Citation** — explicit reference back to source chunks.
- **Hallucination** — answer not supported by evidence.
- **Faithfulness** — degree to which output is grounded in context.
- **Answer relevance** — degree to which output addresses the query.
- **Context precision / recall** — ranked context quality metrics from RAGAS.

**Evaluation:**
- **RAGAS** — open-source eval framework for RAG (faithfulness, answer relevance, context precision, context recall).
- **TruLens, ARES, DeepEval** — alternative eval frameworks.
- **LLM-as-judge** — using an LLM to score answers (with caveats).
- **Golden dataset** — curated (query, ideal answer, ideal sources) records for testing.
- **Lost in the middle** — model degradation on facts placed mid-context.
- **Needle in a haystack** — retrieval evaluation across context positions.

**Advanced patterns:**
- **Naive RAG** — single-shot retrieve-then-generate.
- **Advanced RAG** — adds query/post-retrieval optimizations.
- **Modular RAG** — composable building blocks.
- **Agentic RAG** — agent decides whether/what/how to retrieve.
- **Self-RAG** — model decides to retrieve, then self-critiques output.
- **CRAG (Corrective RAG)** — model evaluates retrieval quality; falls back to web if poor.
- **GraphRAG** (Microsoft) — knowledge-graph-backed retrieval for global reasoning.
- **Contextual retrieval** (Anthropic) — prepend chunk-specific context before embedding.
- **Multi-modal RAG** — retrieve and condition on images/audio/video.

**Production / ops:**
- **Freshness** — how stale the index is relative to the source.
- **Re-indexing** — periodic rebuild of the index.
- **Multi-tenancy** — isolating tenant data in a shared index.
- **Prompt injection via context** — adversarial content in retrieved chunks.
- **PII leakage** — sensitive data in retrieved context.
- **Cost per query** — embedding + retrieval + reranking + generation cost.
- **Cache** — query cache, semantic cache, embedding cache.

---

## Mental model summary

RAG is a four-step pipeline glued onto an LLM: **(1) ingest** documents into chunks, **(2) embed and index** them, **(3) retrieve and rerank** the top results for each query, **(4) generate** an answer grounded in those results. Each step has 3–5 alternative implementations with measurable accuracy and latency tradeoffs. The art of RAG is choosing the right combination for your data, your query distribution, and your accuracy/cost budget — and proving it with evaluation rather than vibes. Naive implementations work for demos; production RAG stacks ~6–10 layered improvements measured against a golden dataset.

That paragraph is what each subsequent doc unpacks.

---

Next: [01 — Foundations](./01-foundations.md).
