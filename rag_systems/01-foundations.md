# 01 — Foundations

Goal: build a precise mental model of what RAG is and isn't. Distinguish it from fine-tuning and long-context. Understand the standard pipeline and the failure modes each stage introduces.

---

## 1. The pure definition

**Retrieval-Augmented Generation** = at query time, a separate retrieval system fetches relevant content from a knowledge store, and that content is inserted into the LLM's prompt before it generates the answer.

The LLM is unchanged — same weights, same API. Only the **prompt** is augmented. RAG is a prompting pattern, not a model architecture.

```
Without RAG:
  prompt = system + user_question
  answer = llm(prompt)

With RAG:
  context = retrieve(user_question, knowledge_base, k=10)
  prompt  = system + context + user_question + "answer using only the context"
  answer  = llm(prompt)
```

Two systems, glued at the prompt boundary. The retrieval system is yours to design; the LLM is a hosted (or self-hosted) commodity.

---

## 2. The three problems RAG solves

### 2.1 Knowledge cutoff

Every LLM has a training cutoff date. Models trained on data through October 2024 don't know what happened in March 2026. For domains where freshness matters — news, prices, internal product changes, competitive intelligence — the model is structurally blind without retrieval.

### 2.2 Private knowledge

Anything not on the public internet is invisible to a pretrained LLM:
- Your company's internal docs, runbooks, SLAs.
- Your customer's account history.
- Your code base, ticket tracker, design docs.
- Your domain-specific manuals (medical guidelines, legal precedents, engineering standards).

The model knows *general* medical knowledge but not *your hospital's* patient-discharge protocol. RAG bridges this.

### 2.3 Hallucination

When asked a factual question outside its knowledge, an LLM will confabulate plausible-sounding answers. RAG provides ground-truth context, and a well-prompted RAG system can be instructed to say "I don't have that information" when retrieval returns nothing relevant — far more reliable than asking the bare model to be honest.

---

## 3. RAG vs alternatives

### 3.1 RAG vs fine-tuning

| | RAG | Fine-tuning |
|---|---|---|
| What it teaches | Facts, knowledge | Behavior, style, format |
| Update cost | Cheap (re-index) | Expensive (re-train) |
| Freshness | Real-time possible | Stale until next training run |
| Citations | Natural (cite retrieved chunks) | Impossible directly |
| Sensitive data control | Centralized in your store | Baked into weights — leakable, hard to delete |
| Cost per query | Higher (more tokens in context) | Lower (no retrieved context) |
| Best for | "What does our company say about X?" | "Always respond in this exact JSON format" |

**The rule:** facts → RAG, behavior → fine-tune. They're complementary; many production systems use both.

### 3.2 RAG vs long-context (just stuff everything in)

Modern models support 200k–2M token windows. Why retrieve at all?

| | RAG | Long-context stuffing |
|---|---|---|
| Cost per query | Low (small context) | Very high (everything billed) |
| Latency | Fast (small prefill) | Slow (large prefill) |
| Quality on retrieval | High (focused context) | Degrades with length ("lost in the middle") |
| Quality on global reasoning | Limited (only sees retrieved bits) | Better (sees everything) |
| Knowledge size limit | Effectively unlimited | Hard cap at model's context window |
| Cacheable? | Each query unique | Stable prefix caches well |

**The rule:** if your knowledge fits in 50k tokens **and you'll query the same content many times**, prompt caching + long context can beat RAG. Above that, or with diverse query patterns, RAG wins on cost and latency.

In practice, advanced systems do **both**: RAG for narrowing the candidate set, then long-context stuffing of the top-N retrieved docs for reasoning depth.

### 3.3 RAG vs SQL / API / function calling

If your data is **structured** (rows in a database, fields in an API), RAG is the wrong tool. RAG searches *text*. To answer "how many tickets were closed last week", you don't want the model fumbling through retrieved ticket dumps — you want a SQL query.

Pattern: use **agents with tools** (one of which may be RAG over unstructured docs, another may be SQL over structured data). The agent decides which tool to call. RAG is one capability, not the whole solution.

### 3.4 RAG vs in-context learning (few-shot)

Few-shot examples teach format/behavior. RAG provides facts. Different jobs.

You can combine them: a RAG prompt typically has (system instructions + few-shot examples of how to use retrieved context + retrieved chunks + question).

---

## 4. The eight stages of a RAG pipeline

Most production RAG systems decompose into:

```
1. Ingestion        — fetch source documents (S3, web, DB, APIs)
2. Parsing          — extract raw text from PDF/HTML/etc.
3. Chunking         — split into retrievable pieces
4. Embedding        — vectorize each chunk
5. Indexing         — store vectors + metadata for fast search
                      ─────── (offline pipeline ends here) ───────
6. Retrieval        — for each query, find top-K candidates
7. Reranking        — reorder by true relevance
8. Generation       — build prompt and call LLM
```

Stages 1–5 run **offline / async**, on a schedule (re-index nightly, or on document change). Stages 6–8 run **online / per query**.

Each stage has its own failure mode:

| Stage | Common failure |
|---|---|
| Ingestion | Source missed entirely; permissions block. |
| Parsing | PDF tables become garbage; OCR noise; encoding errors. |
| Chunking | Relevant info split across chunks; chunks lack context. |
| Embedding | Domain-specific terms misrepresented; multilingual gaps. |
| Indexing | Stale index; ANN settings sacrifice recall. |
| Retrieval | Wrong K; pure semantic misses keyword matches. |
| Reranking | Skipped, or reranker mismatched to domain. |
| Generation | Model ignores context; hallucinates citations; bad refusal. |

**The accuracy lift over naive RAG comes from fixing each of these in turn.** Doc 10 is the exhaustive playbook.

---

## 5. Naive RAG, in 30 lines

To anchor what we're improving, here's the absolute minimum implementation:

```python
import openai, chromadb

# === OFFLINE: index your docs ===
client = chromadb.Client()
collection = client.create_collection("docs")

for doc_id, text in load_documents():
    chunks = [text[i:i+1000] for i in range(0, len(text), 1000)]  # naive split
    embeddings = openai.embeddings.create(
        model="text-embedding-3-small", input=chunks
    ).data
    collection.add(
        ids=[f"{doc_id}-{i}" for i in range(len(chunks))],
        embeddings=[e.embedding for e in embeddings],
        documents=chunks,
    )

# === ONLINE: answer a query ===
def answer(query: str) -> str:
    q_emb = openai.embeddings.create(
        model="text-embedding-3-small", input=[query]
    ).data[0].embedding
    results = collection.query(query_embeddings=[q_emb], n_results=5)
    context = "\n\n---\n\n".join(results["documents"][0])
    prompt = f"Answer using only the context.\n\nContext:\n{context}\n\nQ: {query}\nA:"
    return openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    ).choices[0].message.content
```

This works on a clean dataset and a few easy questions. It will fail in interesting ways on:

- Long docs where the answer spans chunks.
- Acronyms not in the embedding model's training vocabulary.
- Tables, code blocks, equations that break under naive chunking.
- Questions that need ranking by recency, not just similarity.
- Adversarial queries ("ignore prior instructions...").
- Multi-hop questions where chunk A says X and chunk B says Y and the answer needs both.

Every doc in this folder addresses a specific subset of those failures.

---

## 6. The three-generation taxonomy of RAG systems

A useful mental model from the research literature:

### 6.1 Naive RAG

The 30-line implementation above. Single embedding model, single vector store, single retrieval call, single LLM generation. No reranking, no query rewriting, no metadata. Accuracy ceiling: ~50% on real-world QA datasets.

### 6.2 Advanced RAG

Add improvements at each stage:
- **Pre-retrieval:** query rewriting, HyDE, query decomposition.
- **Retrieval:** hybrid (BM25 + vector), metadata filters.
- **Post-retrieval:** reranking, contextual compression, deduplication.
- **Generation:** structured prompts, citations, refusal handling.

This is where most production systems live. Accuracy ceiling: 75–90%.

### 6.3 Modular RAG

RAG as composable building blocks:
- **Routing**: classify query, dispatch to specialized retrievers.
- **Multi-step**: agent loops that retrieve, reason, retrieve again.
- **Self-correction**: model evaluates retrieval quality, requests more.
- **Hybrid sources**: vector store + knowledge graph + SQL + web search, fused.
- **Verification**: separate model checks faithfulness before serving.

This is "agentic RAG" / Self-RAG / CRAG / GraphRAG territory (doc 09). Accuracy ceiling: 90%+ but with significant complexity, latency, and cost.

The right level for you is the simplest one that meets your accuracy bar. Don't jump to modular RAG when advanced RAG would do — the operational cost is real.

---

## 7. The retrieval/generation split is load-bearing

A subtle but important point: **retrieval and generation can fail independently**. Diagnosing which one is broken is the most important debugging skill in RAG.

```
                           ┌──────────────┐
   Query                   │  RETRIEVAL   │
   "What's our refund"  ──►│              │── top-5 chunks
   policy after 30 days?   │  Did we get  │   (some relevant?)
                           │  the right   │
                           │  chunks?     │
                           └──────┬───────┘
                                  │
                                  ▼
                           ┌──────────────┐
                           │  GENERATION  │
                           │              │── final answer
                           │  Did the LLM │
                           │  actually    │
                           │  read them?  │
                           └──────────────┘
```

Two failure axes:

| Retrieval | Generation | Outcome |
|---|---|---|
| Got the right chunks | Used them correctly | ✅ correct answer |
| Got the right chunks | Ignored or hallucinated | ❌ but the index is fine — fix prompt/model |
| Missed the right chunks | "Used" what was there | ❌ but the LLM is fine — fix retrieval |
| Missed the right chunks | Hallucinated | ❌ both broken |

**Always evaluate both separately.** Doc 08 covers metrics for each. A common mistake is staring at bad answers and rewriting prompts when the real problem is retrieval — or vice versa.

---

## 8. The accuracy ceiling concept

There's a hard ceiling: if relevant chunks aren't retrieved, *no amount of LLM smartness can recover them*. The LLM can only work with what it sees.

So in any RAG system, retrieval recall@K sets a fundamental ceiling. If recall@10 = 70%, then the LLM-side accuracy is bounded at ~70% (it might do worse, but cannot consistently do better).

**Optimization order matters:**

1. **First, maximize retrieval quality.** Improve recall@K, then precision@K. This raises the ceiling.
2. **Then, improve generation.** Better prompts, better citations, better refusals. This approaches the ceiling.
3. **Finally, combine in agentic loops.** Use the model to fix retrieval gaps adaptively.

Skipping step 1 and pouring effort into prompt engineering is the most common waste of time in RAG projects.

---

## 9. RAG anti-patterns to avoid early

Stuff people commonly get wrong in their first attempt:

1. **One chunk size for everything.** PDFs, code, chat logs, and tables all need different chunking strategies. (doc 02)
2. **Pure vector search with K=3.** Misses keyword-driven matches; K too low for safety net. (doc 05)
3. **No reranking.** Embedding similarity is a proxy; rerankers are the actual judges of relevance. (doc 06)
4. **Stuffing K=20 chunks into the prompt.** Lost-in-the-middle; cost balloons; signal-to-noise drops. (doc 06)
5. **No metadata in chunks.** Can't filter by date, author, document type, tenant. (doc 02)
6. **No evaluation harness.** "Looks good in demo" is not a quality signal. (doc 08)
7. **Ignoring re-indexing.** Index drifts from source; queries return stale info. (doc 10)
8. **Using vector DB for source of truth.** It's a cache; your authoritative store is elsewhere.
9. **Trusting retrieved content.** Adversarial users can poison the index via uploaded docs. (doc 10)
10. **Skipping multi-tenancy at the index level.** Bolting it on later is painful.

Mark these. Every doc in this folder will revisit at least one of them.

---

## 10. Where this folder takes you

By the end:
- You can design a production RAG pipeline end-to-end with awareness of each component's tradeoffs.
- You can pick a vector store, embedding model, and retrieval strategy for a given domain and budget.
- You can measure RAG accuracy properly, identify which stage is bottlenecking, and apply the right fix.
- You understand when to add reranking, query rewriting, metadata filtering, agentic loops — and when not to.
- You can take a 30%-accurate baseline to a 90%+ system with documented improvements.

This is the same kind of practical depth that the LLM fundamentals folder gave for transformers. The bias is the same: enough theory to reason, enough engineering detail to build.

---

## 11. Summary

RAG = retrieval + augmented generation, glued at the prompt boundary. It exists to overcome knowledge cutoff, private knowledge, and hallucination. The pipeline has 8 stages, each with its own failure modes. Naive implementations cap at ~50% accuracy; advanced/modular RAG reaches 90%+ via layered, measured improvements. Always separate retrieval failures from generation failures when debugging. Pick the simplest pipeline level that meets your accuracy bar. Evaluation is non-negotiable — it's the difference between iteration and guessing.

Next: how raw documents become retrievable chunks — parsing, chunking, and metadata.

---

Next: [02 — Ingestion and Chunking](./02-ingestion-and-chunking.md).
