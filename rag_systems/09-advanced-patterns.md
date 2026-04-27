# 09 — Advanced Patterns

> What you reach for when baseline RAG hits a quality ceiling. Don't enable any of these without an eval set telling you which slice is failing.

By doc 08 you have a measured baseline: hybrid retrieval, reranker, grounded prompt. On a focused corpus and answerable queries, that hits 85–95% on most metrics. The remaining ground is hard:

- Multi-hop questions that need information from multiple docs in sequence.
- Knowledge graphs where relationships matter as much as content.
- Self-correcting pipelines that retrieve again when the first pass fails.
- Long documents where chunk-level retrieval breaks down.
- Multi-modal corpora (text + images + tables).

Every pattern below is a **named technique** you'll see in papers, blog posts, and other engineers' resumes. The point is to know which is which and when to reach for which.

---

## 1. Anthropic Contextual Retrieval

The simplest "advanced" technique. Mentioned briefly in doc 02; covered in full here because it's such a high-leverage default.

**Problem:** chunks lose context when isolated. A chunk reading "The default is 30 seconds" is useless without knowing what "the default" refers to.

**Solution:** before embedding, prepend an LLM-generated 1–2 sentence context describing where the chunk sits in the parent document.

```
ORIGINAL CHUNK:
"The default is 30 seconds, but can be increased to 300 with the
HTTP_TIMEOUT environment variable."

CONTEXTUALIZED CHUNK (for embedding only):
"From the Lyzr Runtime configuration guide, section on Network Settings:
The default is 30 seconds, but can be increased to 300 with the
HTTP_TIMEOUT environment variable."
```

The LLM-generated prefix is fed both to the embedder *and* to BM25. Anthropic's measurements: **35% reduction in retrieval failures**, up to **67% with reranking**.

**Cost:** one cheap LLM call per chunk at ingest (Haiku-tier). ~$1 per 100k chunks. Use prompt caching: pass the full doc once, generate context for each chunk in the same long-context call.

**When to add:** as soon as your eval shows recall ceiling on chunks that "look right but get missed." This is among the highest ROI advanced techniques and should be your first stop.

---

## 2. Agentic RAG

Treat the LLM as a planner that calls retrieval as a tool, possibly multiple times, deciding what to query next based on what it found.

```
USER → Planner LLM
         ├── tool: retrieve(query)        ← may call multiple times
         ├── tool: structured_search(filter)
         ├── tool: web_search(query)      ← out-of-corpus fallback
         └── final answer with citations
```

**What changes vs. linear RAG:**
- No fixed retrieval step. The LLM decides when, what, and how often.
- Multi-hop is handled organically (retrieve, read, retrieve again).
- Heterogeneous sources composed at query time (internal docs + tickets + web).
- Cost and latency 2–5× single-pass.

**Implementation:** function/tool calling on a strong model (Sonnet, GPT-5). Tools include:
- `search_docs(query: str, top_k: int = 10) -> list[Chunk]`
- `search_with_filter(query, filter: dict) -> list[Chunk]`
- `read_full_doc(doc_id) -> str`  ← for "I need the whole document, not chunks"
- `lookup_entity(name) -> dict`  ← for KG-style lookups
- `web_search(query)` ← out-of-corpus

The planner sees tool descriptions and picks among them. Ground its initial system prompt with the rules from doc 07 (cite, refuse if not found, etc.).

**When to use:** long-tail of complex queries. Combine with a router (doc 05.8): cheap classifier sends easy queries to linear RAG, hard ones to agentic.

**Failure modes specific to agentic:**
- **Loop / no progress.** Set max iterations (often 3–6). After cap, force final answer.
- **Wandering.** Planner queries 10 things, none relevant. Add "if you've called retrieve twice without useful results, refuse" to the system prompt.
- **Cost runaway.** Per-request budget (`max_tool_calls`, `max_tokens`). Hard cap latency.

---

## 3. Self-RAG

The model is trained (or prompted) to emit special tokens that decide:
- **Should I retrieve?** Some queries are answerable from priors.
- **Is this retrieved chunk relevant?** Filter at the model level, not just reranker.
- **Is my answer supported by the chunk?** Self-grounding check.

Original paper: training a Llama-2 to emit `[Retrieve]`, `[Relevant]`, `[Supported]` tokens as part of generation. With prompting on a strong model (Sonnet, GPT-5) you can approximate the same behavior without fine-tuning:

```
SYSTEM: For each user question:
  1. Decide if you need to retrieve. If not, answer directly.
  2. After retrieving, evaluate each chunk's relevance before using it.
  3. After answering, verify each claim is supported by the chunks.

Output format:
{
  "needs_retrieval": bool,
  "queries": [..],
  "evaluated_chunks": [{id, relevant: bool, why: str}, ..],
  "answer": "...",
  "self_check": "all claims supported" | "...claims unsupported: [...]"
}
```

**Why it helps:**
- Skips retrieval when not needed (general knowledge questions, math).
- Filters bad chunks the reranker missed.
- Catches its own hallucinations.

**Cost:** more output tokens per query, sometimes a re-call if self-check fails. Trade-off makes sense for high-stakes domains.

---

## 4. CRAG (Corrective RAG)

A retrieval-quality classifier sits between retrieval and generation, deciding what to do based on confidence:

```
retrieve → classifier → {
  "correct"  → use chunks as-is
  "ambiguous"→ refine query, retrieve again, fuse
  "incorrect"→ fall back to web search / parametric LLM
}
```

The classifier is a small model trained (or prompted) to score retrieval quality based on (query, chunks). Three actions:
- **Correct:** proceed normally.
- **Ambiguous:** rewrite query (paraphrase, decompose, HyDE) and retrieve again. Fuse old and new candidates.
- **Incorrect:** corpus doesn't have it. Fall back to a web-search tool, or refuse.

**When to use:** corpora with known coverage gaps. Customer support where many questions fall outside the docs and you want graceful fallback.

---

## 5. GraphRAG

Move from "chunks of text" to "entities and relationships."

**Build phase:**
1. Extract entities and relations from each chunk via LLM. ("ACME Corp acquired Foobar Inc in 2023.")
2. Build a graph: nodes = entities, edges = relations, properties = chunks they came from.
3. Compute community summaries: cluster the graph, summarize each cluster.

**Query phase (Microsoft GraphRAG style):**
1. Map step: each community summary attempts to partially answer.
2. Reduce step: combine partial answers into final.

Or local query: extract entities from query, walk the subgraph, retrieve attached chunks.

**Why it helps:**
- **Multi-hop questions.** "Who replaced the CFO that resigned in 2023?" — walking edges (CFO → resigned 2023 → ?) is exactly what graphs are for.
- **Aggregation across many sources.** "What products has ACME launched in Asia?" — gather all (ACME, launched, _, Asia) edges.
- **Explainability.** You can show *why* an answer was reached as a path through the graph.

**Cost:** building the graph is expensive — one LLM call per chunk for extraction, plus clustering and community summaries. For a 100k-chunk corpus, days of LLM time and a few thousand dollars.

**Stack:** Microsoft `graphrag` library, LlamaIndex `KnowledgeGraphIndex`, Neo4j with vector ext, or roll-your-own with Qdrant + a graph DB.

**When to use:** corpora where relationships are first-class — biomedical, legal, financial filings, enterprise data with lots of named entities. Overkill for support docs.

---

## 6. Hierarchical / RAPTOR

(Mentioned in doc 02. Expanded here.)

RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval) builds a tree:

```
                  [whole-corpus summary]
                  /         |          \
        [cluster A summ]  [B summ]   [C summ]
        /     |     \      ...
   [chunk] [chunk] [chunk]
```

- Cluster chunks (k-means or HDBSCAN on embeddings).
- LLM-summarize each cluster.
- Cluster the summaries. Summarize again. Recurse until you have one root summary.

**Query:** retrieve across all levels (chunks + summaries) in one fused index. Or: traverse top-down, picking which subtree to descend.

**Why it helps:**
- Broad/abstract questions ("what's the overall structure of...") match summaries naturally.
- Specific questions still match leaf chunks.
- One index handles both granularities.

**Cost:** O(n) LLM calls for summaries during ingest. Index ~30% larger than chunks-only.

**When to use:** mixed query types over a single coherent corpus (a book, a documentation set). Less useful for short, fragmented corpora.

---

## 7. Multi-vector indexing

Store **multiple representations** per chunk, query against all.

Common variants:
- **Chunk + summary embedding:** embed the chunk and its LLM summary; index both, dedupe at retrieval.
- **Chunk + hypothetical questions:** for each chunk, LLM generates 3 questions it answers; embed the questions, point them at the chunk.
- **Multi-perspective:** embed the chunk verbatim, the chunk in formal English, the chunk in casual English. Catches stylistic mismatch with queries.

**Trade-off:** index size grows ~3–5×; recall on hard queries jumps measurably.

ColBERT (doc 03.2.3) is the per-token version of this idea taken to its logical extreme.

---

## 8. Iterative / multi-hop retrieval (named patterns)

### 8.1 IRCoT (Interleaved Retrieval with Chain-of-Thought)
Generate one reasoning step → retrieve based on that step → continue reasoning → retrieve again. Output the final answer when the chain converges.

### 8.2 Self-Ask
Force the model to explicitly ask itself sub-questions:
```
Question: Who replaced the CFO that resigned during the 2023 audit?
  Are follow-up questions needed: yes
  Follow up: Who was the CFO during the 2023 audit?
  Intermediate answer: Jane Doe.
  Follow up: Who replaced Jane Doe as CFO?
  Intermediate answer: John Smith.
So the final answer is: John Smith.
```

Each "Intermediate answer" comes from a retrieval step. The structure is the prompt scaffolding; you parse the follow-ups, retrieve, inject answers, continue.

### 8.3 ReAct + retrieve-tool
The classic agent loop (Reason → Act → Observe). With retrieval as a tool, this collapses into agentic RAG (section 2 above).

**When to use any of these:** when eval shows multi-hop slice failing. Don't enable globally — gate via a query classifier.

---

## 9. Long-context "RAG-less" and the fallback question

Models with 1M+ token context (Gemini 2.5 Pro, GPT-5, Claude 4.x) can fit a small corpus directly into the prompt. Tempting to skip retrieval entirely.

**When it works:**
- Corpus stable and small (<500k tokens).
- Latency-tolerant (huge contexts are slower).
- Quality acceptable despite lost-in-the-middle effects.

**When it doesn't:**
- Large corpora. Cost scales linearly per query.
- Frequent updates. Re-cache cost on every doc edit.
- Multi-tenant. You'd pay per tenant per query.
- Multi-hop. Long context still has lost-in-the-middle.

**Hybrid:** RAG to narrow to ~50–100 chunks (~50k tokens), then dump them all into a long-context model and let it reason. Bypass reranking when the model can handle the whole candidate set. Quality beats narrow top-K on hard queries; cost roughly same as aggressive K_final.

---

## 10. Multimodal RAG

Corpora with images, tables, diagrams, scanned PDFs.

### 10.1 Pure-text path
Run OCR / VLM captioning on images at ingest. Treat captions as text chunks. Cheap, lossy.

### 10.2 Multimodal embedding path
Use a multimodal embedder (CLIP, Cohere embed-v4, Voyage multimodal, Jina-CLIP). Index images and text in the same vector space. At query time, the same embedding works for both.

### 10.3 ColPali / page-image path
Render each PDF page as an image. Use ColPali (late-interaction over patches) for retrieval. Skip OCR entirely. Ground-breaking for documents with complex layouts (financial reports, scientific papers, scanned legal docs).

### 10.4 Generation
Send retrieved images directly to a vision-capable model (Claude, GPT-4V, Gemini). Ask for citation as "page 12, Figure 3."

**When to choose:**
- Mostly text with occasional images: caption-and-text-index path.
- Image-heavy or layout-critical: ColPali or multimodal embedder.
- Mixed: hybrid (text path for text, ColPali for layout-heavy docs, fuse).

---

## 11. Domain adaptation

When general embeddings/rerankers don't match your jargon:

1. **Synthetic Q-doc pairs from your corpus.** LLM generates 3–5 questions per chunk. Use as training data.
2. **Hard-negative mining.** Find chunks the current embedder thinks are close but are wrong. Add as negatives in contrastive training.
3. **Fine-tune the embedder.** Sentence-transformers + LoRA on a 500M model. Hours, not days.
4. **Fine-tune the reranker.** Often higher leverage than embedder, since reranker sees more of the input.
5. **Continued pretraining of the generator.** Last resort; expensive; usually unnecessary if RAG is working.

Track gains on the eval set. Domain fine-tuning typically buys 5–15 NDCG points if the general model was below 0.5 NDCG@10 in domain. Less if you were already above 0.7.

---

## 12. Streaming / incremental ingest

For corpora that update constantly (chat, news, tickets):

- **Append-only index.** Most vector DBs support incremental insert. New chunks indexed in seconds.
- **Soft delete** for stale docs (tombstone on `valid_until`). Hard-delete in batches.
- **Time-decay scoring** at query time (doc 05.12).
- **Re-embed on edit** when chunk text changes — pin chunk IDs to source content hashes.

For multi-tenant systems: per-tenant pipelines. A noisy enterprise tenant ingesting 10k docs/min should not back up a quiet one.

---

## 13. Cache-augmented patterns

### 13.1 Question cache
Hash the user query → if seen recently with high-confidence answer, serve cached. Latency drops to <50ms; cost drops to ~zero.

Risk: stale answers when corpus updates. TTL the cache (5–60 minutes), bust on doc updates touching cited chunks.

### 13.2 Cluster cache
Cluster queries by embedding similarity (within ε). Serve a similar cached answer if it cites still-valid chunks.

More aggressive than exact-match cache; needs careful eval.

### 13.3 Prompt cache (Anthropic / OpenAI / Gemini)
Reuse the static prefix of long system prompts and tool specs. 90% discount on cache hits, lower TTFT.

In RAG specifically the *retrieved* context is per-query, so it's not cacheable — but the system prompt, tool descriptions, and few-shot examples are. Pay this 30 minutes of plumbing once; it amortizes.

---

## 14. Hybrid: combining patterns

Real production systems chain these:

```
                    ┌───────────────────┐
       query  ───►  │  Router (LLM)     │ ──┬── easy: linear RAG
                    └───────────────────┘   │
                                            ├── multi-hop: agentic RAG
                                            │
                                            ├── ambiguous: CRAG with rewrite
                                            │
                                            └── out-of-corpus: web fallback or refusal
                          
                          all paths share:
                          - Contextual retrieval at ingest
                          - Hybrid (BM25 + dense + RRF)
                          - Cross-encoder rerank
                          - Grounded prompt + citations
                          - Eval + tracing
```

**Don't build the full thing day one.** The order to add:

1. Baseline: chunking + dense + BM25 + RRF + reranker + grounded prompt + eval.
2. Contextual Retrieval — biggest single ROI of advanced techniques.
3. Self-querying / metadata filters — if your corpus has rich metadata.
4. Multi-query / decomposition — if compound queries fail.
5. Agentic / Self-RAG — if multi-hop fails after the above.
6. GraphRAG / RAPTOR — if your domain is relationship- or hierarchy-heavy.
7. Multimodal — if your corpus has it.
8. Domain fine-tuning — last resort, when you've maxed everything else.

---

## 15. Pattern → failure mode lookup

Reverse direction: when a slice fails, which pattern targets it?

| Failing slice | First thing to try |
|---|---|
| "Right chunk in corpus, never retrieved" | Contextual Retrieval; check input_type/prefix; fine-tune embedder |
| "Multiple paraphrases retrieve the same docs" | Multi-query expansion + RRF |
| "Compound queries get partial answers" | Decomposition |
| "Multi-hop questions go in circles" | Agentic RAG with retrieve-tool |
| "Out-of-corpus questions hallucinate" | Threshold + refusal; CRAG with web fallback |
| "Long-doc queries miss" | Hierarchical / parent-window / RAPTOR |
| "Image / table content invisible" | Multimodal embedder or ColPali |
| "Sources contradict, model picks one silently" | "Present conflict" rule + rerank by score; agentic verifies |
| "Answers stale relative to docs" | Re-embed cadence; doc-update bus to invalidate cache |
| "One tenant has bad quality" | Per-tenant eval slice; possibly per-tenant fine-tune |

---

## 16. Keyword cheat-sheet

- **Contextual Retrieval** (Anthropic) — LLM-prepended chunk context before embedding.
- **Agentic RAG** — LLM-driven planner calls retrieval as a tool.
- **Self-RAG** — model emits retrieve/relevant/supported decisions.
- **CRAG** — corrective RAG; retrieval-quality classifier picks fallback path.
- **GraphRAG** — entity-and-relation graph + community summaries.
- **RAPTOR** — recursive abstractive tree of summaries.
- **Multi-vector indexing** — multiple representations per chunk.
- **IRCoT / Self-Ask / ReAct** — multi-hop reasoning patterns.
- **Long-context "RAG-less"** — fit corpus directly into prompt.
- **Multimodal RAG / ColPali** — image-and-text retrieval.
- **HyDE** — hypothetical doc embedding (also doc 05).
- **Domain fine-tuning** — adapt embedder/reranker on synthetic Q-doc pairs.
- **Question cache / cluster cache / prompt cache** — caching layers.

---

## 17. What you take into doc 10

You now have a buffet of techniques. The final doc is the **playbook**: the actual sequence to apply them in, the failure-mode atlas to navigate by, and the production checklist that turns a working RAG into one that survives real users at scale.

Next: [10 — Production Playbook](./10-production-playbook.md)
