# 05 — Retrieval Strategies

> The highest-leverage layer in the entire RAG stack. Most accuracy gains live here.

By doc 04 you have chunks, embeddings, and an index. You can already do `embed(query) → top-K`. That is *naive retrieval*, and on real workloads it sits at maybe 50–70% recall.

This doc is about everything that closes the gap from there to the high 90s. The trick is that **the user's query is rarely the right thing to search with**. Almost every advanced retrieval technique is some flavor of "fix the query before, fix the results after."

---

## 1. The retrieval failure modes you're fighting

Six recurring reasons the right chunk doesn't come back:

| Failure | Example |
|---|---|
| **Vocabulary mismatch** | Query: "annual revenue 2024." Doc says: "FY24 top-line." |
| **Exact-term miss** | Query: "error E_TIMEOUT_507." Dense embedding generalizes; loses the literal code. |
| **Underspecified query** | Query: "How does it work?" — semantically empty in isolation. |
| **Compound query** | "Compare X and Y on dimensions A, B, C." Five different chunks needed. |
| **Wrong granularity** | Best answer is in a 200-word section; you indexed 2000-word pages. |
| **Stale priors** | Doc updated; old version still in the index. |

Each technique below targets one or more of these.

---

## 2. The retrieval ladder (cheap → expensive)

```
1. Dense retrieval (vector top-K)            ← baseline
2. + Hybrid (BM25 + dense, RRF fusion)       ← +5–15 pts
3. + Query rewriting / expansion             ← +3–10 pts
4. + Multi-query / decomposition             ← +5–15 pts on hard queries
5. + Metadata / self-querying filters        ← +5–20 pts on filterable corpora
6. + HyDE                                    ← +2–8 pts on underspecified queries
7. + Reranker (cross-encoder)                ← +10–25 pts (covered in doc 06)
8. + Iterative / agentic retrieval           ← +5–15 pts on multi-hop (covered in doc 09)
```

Climb the ladder in this order. Stop when your eval (doc 08) says you're hitting target.

---

## 3. BM25 — the sparse baseline you must keep

Term-frequency × inverse-document-frequency, with document-length normalization.

```
score(q, d) = Σ_{t in q} IDF(t) · (tf(t,d) · (k1+1)) / (tf(t,d) + k1·(1 − b + b·|d|/avgdl))
```

Don't memorize the formula. Remember:
- **Rewards rare terms** that appear in a doc.
- **Penalizes long docs** (saturating, not killing).
- **Zero-shot.** No training, no embedding model.
- **Catches what dense misses:** product codes, error strings, person names, version numbers, jargon.

BM25 alone is a surprisingly hard baseline. On TREC and BEIR datasets it routinely beats first-gen dense embeddings. **Always run it as a sanity check.** If your dense system can't beat BM25 on your own data, your embedding model is wrong for the domain.

Implementations:
- **Postgres** — `tsvector` + `ts_rank_cd`. Built-in. Great with pgvector.
- **Elasticsearch / OpenSearch** — BM25 is the default scoring.
- **Tantivy / Lucene / Whoosh** — embedded.
- **rank_bm25** (pure Python) — fine for small corpora.

---

## 4. Hybrid retrieval — fuse dense + sparse

The single highest-ROI move after baseline dense. Run **both** retrievals, fuse the result lists.

### 4.1 Reciprocal Rank Fusion (RRF) — the default

```python
def rrf(results_lists, k=60):
    # results_lists = [[doc_id, ...], [doc_id, ...]]
    scores = {}
    for results in results_lists:
        for rank, doc_id in enumerate(results):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])
```

- **Score-free.** Doesn't matter that BM25 scores and cosine scores live on different scales.
- `k=60` is the canonical constant from the original paper. Rarely worth tuning.
- Works for **any number of result lists** — fuse dense + sparse + a reranker + a metadata-filtered query, all of them.

### 4.2 Weighted score fusion

```python
final = α · normalize(dense) + (1 − α) · normalize(sparse)
```

Min-max normalize each score list to [0,1] first. Tune α on a held-out eval set. More fragile than RRF (depends on score distributions) but can win when one signal is reliably stronger.

### 4.3 Rank-aware fusion via reranker

Don't fuse — **concatenate the candidate sets** (e.g., top-50 dense ∪ top-50 sparse), then let the reranker (doc 06) sort them. Often beats RRF when the reranker is strong.

**Production stack pattern:**
```
query
  ├──► BM25 → top 50 ──┐
  └──► dense → top 50 ─┴──► concat (dedupe) → reranker → top 10 → LLM
```

This is what Anthropic's contextual-retrieval blog post and most well-tuned production systems converge to.

---

## 5. Query rewriting

The user's query is the worst version of itself for retrieval. Rewriting beats it into shape.

### 5.1 Spell / typo correction

Cheap. Often a 5–10 point recall lift on consumer queries. Use a small LLM or a spelling library; gate on edit distance.

### 5.2 Synonym / expansion

```
"VPC peering not working" 
  → ["VPC peering broken", "VPC peering connectivity issue", "AWS VPC peering troubleshooting"]
```

Add expansions either by LLM or by a static synonym map. Run each variant, fuse with RRF.

### 5.3 Conversational rewrite (turn-aware)

In a chat: the user says "what about for Postgres?" The retrieval needs the prior turn's context.

**Pattern:** before retrieval, run a small LLM with the conversation history and produce a **standalone, fully-specified query**.

```
SYSTEM: Rewrite the user's last message as a standalone search query 
        using context from the conversation. Output only the query.

INPUT:
  USER: How do I configure connection pooling?
  ASSISTANT: For pgbouncer, set...
  USER: what about for Postgres?

OUTPUT:
  How to configure connection pooling for Postgres
```

Cheap (a Haiku/Mini model in <300ms) and dramatically improves multi-turn retrieval.

### 5.4 Step-back prompting

Rewrite the specific query into a more general one to retrieve broader context.

```
"Why does my Python 3.11 asyncio event loop hang on subprocess.communicate()?"
  → STEP-BACK: "How do asyncio and subprocess interact in Python?"
```

Retrieve both the specific and the step-back query, fuse. Helps when the corpus has more conceptual material than literal Q&A pairs.

### 5.5 Hypothetical Document Embeddings (HyDE)

Counterintuitive but effective.

1. Ask an LLM to **answer** the query (hallucinations OK).
2. Embed the *fake answer*.
3. Search with that embedding.

The fake answer is closer in embedding space to real answer-shaped passages than the question itself is. Strong on underspecified or question-form queries, weak on factual queries where the LLM might hallucinate the wrong direction.

```python
fake_answer = llm("Write a short factual answer to: " + query)
results = vector_db.search(embed(fake_answer), top_k=20)
```

Cost: one extra LLM call per query. Diminishes once you have a strong reranker.

---

## 6. Multi-query / query decomposition

When one query can't retrieve everything you need.

### 6.1 Multi-query (parallel paraphrase)

Generate N (typically 3–5) paraphrases of the user query, retrieve each, fuse.

```
"How do RAG systems handle long documents?"
  → "Strategies for chunking long documents in RAG"
  → "Hierarchical retrieval for long context RAG"
  → "RAG over long PDFs"
```

Cheap insurance against a single phrasing being unlucky in embedding space.

### 6.2 Decomposition (sub-questions)

For compound queries, ask the LLM to break the query into sub-questions, retrieve for each, then concatenate or aggregate the contexts.

```
"Compare LangChain and LlamaIndex on agent support, retrieval, and observability."

  → Q1: "What agent capabilities does LangChain provide?"
  → Q2: "What agent capabilities does LlamaIndex provide?"
  → Q3: "How does LangChain handle retrieval?"
  → ... (six sub-queries total)
```

Each sub-query retrieves independently. The final prompt to the answer LLM gets all the context, organized.

This is the foundation of **multi-hop** and **agentic** retrieval (doc 09).

---

## 7. Self-querying / metadata filtering by LLM

Your corpus has structured metadata (date, author, type, language). Users almost never write filter clauses. **The LLM can extract them for you.**

```
"Show me Anthropic blog posts from 2024 about agent evaluation."
  
  ↓ LLM as router/parser
  
  query: "agent evaluation"
  filter: {source = "blog.anthropic.com", year = 2024}
```

Implementation: a small LLM call with a JSON schema describing the available filter fields. The output filter goes into the vector DB query.

**Why huge:** before this, all that metadata sat unused. Now selective filters cut the candidate set 100× before ANN search even runs, often turning a 60% recall query into 95%.

LangChain calls this `SelfQueryRetriever`; LlamaIndex has a similar `MetadataFilters` flow. Or just write your own — it's ~30 lines.

---

## 8. Routing — different queries, different retrievers

Real corpora are heterogeneous. A "support docs" query and a "marketing copy" query want different indexes.

Two patterns:

**Rule-based router** — keywords / regex / classifier. Cheap, brittle.

**LLM router** — a small classifier LLM picks one of N retrievers (or "none, answer directly").

```
query → router LLM → "search:product_docs" | "search:codebase" | "search:tickets" | "no_retrieval"
```

Done well, this is the foundation of **agentic RAG**. Done badly, it adds latency without quality.

---

## 9. Top-K — picking K

Two questions: how many to retrieve from the index (`K_initial`), how many to send to the LLM (`K_final`).

| Stage | Typical K | Why |
|---|---|---|
| Vector top-K | 50–200 | Cheap; over-fetch to give the reranker room |
| After hybrid fusion | 50–100 | Dedupe, filter, then trim |
| After reranker | 5–20 | This is what hits the LLM context |
| Final to LLM | 3–10 | Quality > quantity. More context ≠ better answer |

Common mistake: jamming `top_k=20` raw chunks into the LLM prompt. Quality usually peaks at `K_final = 5–8` post-rerank. Past that you trigger lost-in-the-middle (doc 06) and waste tokens.

---

## 10. Diversity / MMR — avoiding near-duplicates

If your top-10 are all near-paraphrases of the same source, the LLM has one signal repeated, not ten signals.

**Maximal Marginal Relevance (MMR):**

```
score = λ · sim(query, chunk) − (1 − λ) · max sim(chunk, already_selected)
```

Greedily pick chunks that are relevant *and* dissimilar to what's already chosen. λ around 0.5–0.7 is typical.

Alternative: cluster the candidates by embedding, take the top from each cluster. Same idea, different mechanics.

Use MMR when your corpus has lots of duplicated/templated content (FAQs, generated docs, repeated boilerplate).

---

## 11. Iterative / multi-hop retrieval

Some questions can't be answered by any single chunk. You need to retrieve, read, then retrieve again based on what you learned.

Classic example:
> "Who replaced the CFO that resigned during the 2023 audit?"

Hop 1: retrieve "who was the CFO during the 2023 audit." Read: "Jane Doe."
Hop 2: retrieve "who replaced Jane Doe as CFO." Read: "John Smith."
Answer: "John Smith."

**Patterns:**
- **IRCoT (Interleaved Retrieval with Chain-of-Thought)** — LLM thinks one step, retrieves, thinks again.
- **Self-Ask** — LLM explicitly asks itself sub-questions and answers each via retrieval.
- **ReAct + retrieve-tool** — agent loop with retrieval as a tool. Most flexible. Most expensive.

Trade-offs: 2–4× the latency, much better quality on multi-hop. Don't enable globally — gate with a "needs_multi_hop" classifier or let the user opt in for hard queries.

Covered in detail in doc 09.

---

## 12. Time-aware retrieval

Many RAG corpora have a strong recency signal (news, support tickets, changelogs). Treat time as a first-class score component.

**Recency boost:**
```
final = sim_score · exp(−λ · age_in_days)
```

`λ` tuned per domain — news might decay over weeks, legal docs over decades.

**Time-window pre-filter:**
```
WHERE created_at > NOW() - INTERVAL '90 days'
```

When the user's query has time markers ("last quarter," "yesterday"), use self-querying to extract them into filters.

**Versioning:** if your corpus has explicit versions (API v1, v2, v3), store the version, filter to "active" or "user's selected" version. Otherwise the LLM cites deprecated material.

---

## 13. Personalization

If your system has per-user history (past chats, projects, preferences), it can rerank or filter retrieval per user.

- **User-aware rewrite:** include "the user is working in the X project" in the rewrite prompt.
- **User-specific subindex:** boost results from docs the user has interacted with.
- **Negative signals:** down-weight chunks the user has marked unhelpful.

Avoid the trap of personalizing the index itself — keep one global index, layer personalization at query-time.

---

## 14. Putting it together — a strong default stack

For most production RAG, the following stack hits high-90s on a focused eval set:

```
user query
  │
  ├── (chat?) → conversational rewrite
  │
  ├── (need filters?) → self-query → extract metadata filter
  │
  ├── (compound?) → decompose into 2–4 sub-queries
  │
  └── for each sub-query:
         ├── BM25 top-50
         ├── dense top-50
         ├── (optional) HyDE → dense top-50
         └── concatenate, dedupe → reranker top-K_final
  │
  ▼
  Build context (doc 06) → LLM
```

You don't have to enable every step. The order to add them:
1. Hybrid (BM25 + dense + RRF) — biggest single win after baseline.
2. Reranker (doc 06) — second biggest.
3. Conversational rewrite — only if you have multi-turn.
4. Self-query — only if your metadata is rich.
5. Decomposition / multi-hop — only when eval shows compound queries failing.
6. HyDE — last; sometimes redundant once a strong reranker is in place.

Always measure (doc 08) before adding the next layer.

---

## 15. The keyword cheat-sheet

- **BM25** — sparse term-based scoring; the baseline.
- **TF-IDF** — older sparse scoring (term-freq × inverse-doc-freq).
- **SPLADE** — neural sparse, expanded with related terms.
- **Hybrid retrieval** — fuse dense + sparse.
- **RRF (Reciprocal Rank Fusion)** — the default fusion algorithm.
- **HyDE** — hypothetical document embeddings; embed an LLM-written fake answer.
- **Multi-query** — N paraphrases of the user query.
- **Decomposition** — break compound query into sub-questions.
- **Self-query / SelfQueryRetriever** — LLM extracts structured filters from natural language.
- **Step-back prompting** — generalize the query for broader context.
- **MMR (Maximal Marginal Relevance)** — diversity-aware reranking.
- **Multi-hop / iterative retrieval** — retrieve → read → retrieve again.
- **IRCoT, Self-Ask, ReAct** — multi-hop agent patterns.
- **Routing** — pick which retriever(s) to use per query.
- **Top-K / over-fetch / re-rank** — the K-staging vocabulary.

---

## 16. What you take into doc 06

You now have a candidate set of, say, 100 chunks. Most of them are decent; some are great; the order is approximate. The next layer **re-orders them so the best chunks land at positions where the LLM will actually read them**, and constructs the prompt that makes the model use them.

This is where reranking and the lost-in-the-middle problem live.

Next: [06 — Reranking and Context Construction](./06-reranking-and-context.md)
