# 06 — Reranking and Context Construction

> The two layers between "I have candidates" and "the model reads them well."

By doc 05 you have ~50–100 candidate chunks per query. Three things now decide whether the answer is correct:

1. **Reranking** — re-ordering candidates so the genuinely best chunks land at the top.
2. **Context construction** — assembling those chunks into a prompt the LLM reads carefully.
3. **Position handling** — fighting "lost in the middle."

Skipping rerank is the single most common reason a RAG demo looks great and the production version doesn't.

---

## 1. Why rerank at all

Bi-encoders (the dense embedders from doc 03) have a structural ceiling. They never see query and doc together — they map both into a fixed space and measure distance. Two failure modes are baked in:

- **Surface-level similarity wins.** Doc that shares vocabulary with query scores high even if it's tangential.
- **Subtle relevance loses.** Doc that answers the query with different wording scores low.

A **cross-encoder** sees query and doc concatenated. It computes attention across them and outputs a single relevance score. Result: 10–25 points of NDCG@10 over a bi-encoder, on average, across BEIR-style benchmarks. The gap is often the difference between "decent demo" and "production-grade."

Cost: you can't precompute. Every (query, doc) pair is one forward pass at query time. Latency: ~5–50ms per pair, depending on model and length. So you **rerank a small candidate set (50–100), not the whole corpus.**

The retrieval-then-rerank pattern lets you have both: bi-encoder over millions of chunks for the candidate set, cross-encoder over a hundred for the final order.

---

## 2. The reranker model landscape

### 2.1 API rerankers

| Model | Provider | Latency (top-100) | Pricing | Notes |
|---|---|---|---|---|
| Cohere Rerank 3.5 | Cohere | ~200–400ms | $2/1k searches | The de-facto default. Multilingual. Strong. |
| Voyage rerank-2 / rerank-2-lite | Voyage | ~150–400ms | $0.05/1M tokens | Strong; lite is much cheaper. |
| Jina Reranker v2 | Jina | ~150–300ms | low | Multilingual, available open + API. |
| Mixedbread mxbai-rerank-v1 | Mixedbread | n/a | open or API | Apache-licensed. |

### 2.2 Open-source rerankers (self-host)

| Model | Size | Notes |
|---|---|---|
| **bge-reranker-v2-m3** | 568M | Multilingual, very strong open default. |
| **bge-reranker-large** | 560M | English-focused, established. |
| **mxbai-rerank-large-v1** | 435M | Apache 2.0, top of its size class. |
| **jina-reranker-v2-base-multilingual** | 278M | Compact, fast. |
| **MiniLM cross-encoder (sentence-transformers)** | 22M | Tiny, fast, decent. CPU-friendly baseline. |

A single A10G runs `bge-reranker-v2-m3` at ~100 pairs in ~150ms (batch 32). That's enough for most production at modest QPS.

### 2.3 Late-interaction (ColBERT / ColPali)

Per-token vectors with MaxSim scoring. Not exactly a reranker — more a hybrid first-stage. But it sits in the same quality tier as cross-encoders at lower latency for long documents. ColBERT v2, Jina ColBERT v2, ColPali (multimodal/PDF page images).

Use ColBERT-style when:
- Documents are long and a cross-encoder hits its input limit.
- You can afford the ~100× storage of multi-vector indexes.
- You want a single retrieval+rerank stage without two model calls.

### 2.4 LLM-as-reranker

Send (query, doc_1, ..., doc_K) to a small LLM and ask it to rank.

```
Rank the following passages by relevance to the query.
Output: comma-separated IDs in best-to-worst order.
```

- **Pro:** zero training, easy to iterate. With a smart model (Sonnet, GPT-4-mini), often beats fine-tuned rerankers.
- **Con:** higher latency and cost than dedicated rerankers; sensitive to prompt; positional bias (LLMs over-rank early-listed candidates).
- Worth testing against Cohere/BGE on your eval set. Sometimes the simplest stack wins.

---

## 3. Picking a reranker — decision flow

```
Need multilingual?
  Yes → Cohere Rerank 3.5 (API) or bge-reranker-v2-m3 (self)

Cost-constrained at high QPS?
  Yes → bge-reranker-v2-m3 self-hosted, or Voyage rerank-2-lite

Latency-critical (<100ms reranking budget)?
  → MiniLM cross-encoder or Jina Reranker v2-base; trade ~3–5 pts quality

Long docs (>2k tokens per chunk)?
  → ColBERT v2 / Jina ColBERT v2; or chunk-window the doc and max over scores

Want to skip the dedicated model?
  → LLM-as-reranker with Sonnet/Mini. Slower, but no extra infra.
```

For a typical mid-stage RAG project: **start with `bge-reranker-v2-m3` self-hosted or Cohere Rerank 3.5 API.** Both are battle-tested defaults.

---

## 4. The reranker call shape

Two contracts.

**Cohere-style API:**
```python
results = co.rerank(
    model="rerank-v3.5",
    query=user_query,
    documents=[c.text for c in candidates],
    top_n=10,
)
# results.results = [{"index": int, "relevance_score": float}, ...]
```

**Self-hosted (sentence-transformers cross-encoder):**
```python
from sentence_transformers import CrossEncoder
model = CrossEncoder("BAAI/bge-reranker-v2-m3")
pairs = [(user_query, c.text) for c in candidates]
scores = model.predict(pairs, batch_size=32)
ranked = sorted(zip(candidates, scores), key=lambda x: -x[1])[:10]
```

**Always pass the full chunk text**, not just the title. The cross-encoder needs to read the body.

**Watch the input length.** Cross-encoders truncate at 512 tokens by default. If your chunks are longer, either: (a) chunk smaller from the start, (b) use a long-context reranker (`bge-reranker-v2-gemma`, `mxbai-rerank-large`), or (c) sliding-window per chunk and take the max.

---

## 5. Stage budgets

```
top 50 (BM25) ──┐
                 ├── concat → dedupe → 100 candidates → rerank → top 10 → LLM
top 50 (dense) ─┘
```

Why 100 in, 10 out:
- Cross-encoder cost is linear in K. 100 pairs × 10ms = 1s on CPU, ~150ms on GPU. Fine.
- Quality plateau: top-3 to top-10 reranked is usually >95% of the recall ceiling.
- Past 100 candidates, the marginal one is almost certainly noise.

If you can afford it, **rerank top-200 → top-10** for hard queries. Rare to see further gains past that.

---

## 6. Score thresholds and "I don't know"

Rerankers return calibrated-ish scores (0–1 range). Use them for two things:

### 6.1 Cutoffs
If the top reranked score is below a threshold (often 0.3–0.5 depending on model), **return "I don't have enough information"** instead of pushing weak chunks to the LLM. Hallucinations love empty context.

```python
top = ranked[0]
if top.score < THRESHOLD:
    return "I don't have information about that in the knowledge base."
```

This single check kills a class of hallucinations and tells the user honestly.

### 6.2 Adaptive K
Instead of always returning top-10, return all chunks with `score > threshold`, capped at K_max. Some queries genuinely need 2 chunks; others 8.

Calibrate the threshold per model on your eval set. Don't use cross-eval thresholds — Cohere Rerank's 0.5 is not BGE Rerank's 0.5.

---

## 7. Context construction — assembling the prompt

You have your top-K reranked chunks. How you arrange them in the prompt matters as much as which chunks you picked.

### 7.1 The standard template

```
SYSTEM:
You are a helpful assistant. Answer the user's question using ONLY the
context provided below. If the context does not contain the answer, say so.
Cite sources by [doc_id].

CONTEXT:
[1] (source: docs.lyzr.ai/agents, page 3)
<chunk text>

[2] (source: docs.lyzr.ai/runtime, page 12)
<chunk text>

...

USER QUESTION:
{query}
```

Critical pieces:
- **System instructions clearly bound to the context.** "Use ONLY the context" is the single most important instruction in a RAG prompt.
- **Numbered/labeled chunks.** Lets the model cite. Lets you parse citations back.
- **Source metadata visible.** Lets the model judge authority and reject irrelevant context.
- **User query at the end.** Models follow recent instructions more strongly. Place the query last.

### 7.2 What goes in the chunk's wrapper

For each chunk, you can include:
- The text itself.
- The doc title and section path (`Architecture > Memory > Vector Stores`).
- The URL or doc_id for citations.
- Date / version.
- Score (rare, but useful when debugging).

Keep wrappers tight. Verbose metadata steals tokens without helping the model.

### 7.3 The order of chunks in the prompt

This is where most teams accidentally tank quality.

---

## 8. Lost in the middle

Long-context LLMs read content unevenly. The seminal paper *Lost in the Middle* (2023, replicated since across models) showed a strong U-shape: accuracy is highest when the relevant info is at the **start** or **end** of the context, lowest in the middle.

The effect is real on every model — Claude, GPT, Gemini, open models — and gets worse with longer contexts.

**Mitigations:**

1. **Order by relevance, both ends first.**
   ```
   [most-relevant chunk]
   [3rd]
   [5th]
   [4th]
   [2nd]
   [least-relevant]
   ```
   Or simpler: top chunk first, second-best last, others in the middle.

2. **Don't pad context.** Sending 10 mediocre chunks because "more = better" measurably hurts. 3–5 strong chunks beats 10 mediocre ones.

3. **Move the query to the end** so it's adjacent to the most-recent attention.

4. **Repeat the question after the context** for very long contexts:
   ```
   CONTEXT: ...
   QUESTION (repeated): {query}
   ```

5. **Smaller chunks** are easier to position-rank; larger chunks dilute the signal.

For most production: order top-K by reranker score, prepend the query metadata, append the query. Re-evaluate position effect when you change models.

---

## 9. Compression / summarization of context

When candidates exceed your token budget (or when you're paying per token):

### 9.1 Extractive compression
Keep only the sentences within each chunk that match the query. Implementations:
- **LLMLingua / LongLLMLingua** (Microsoft) — small model rates each token, drops low-importance ones.
- **Sentence-level cross-encoder filter** — score each sentence in each chunk, keep top sentences globally.

### 9.2 Abstractive compression
Send candidates to a fast LLM, ask for a summarized brief, send brief to the answering LLM. Halves input tokens, adds one LLM hop, risks losing detail. Use only when token budget is the hard constraint.

### 9.3 The cheaper alternative
**Smaller K and shorter chunks.** Fixing chunking is cheaper than wiring up a compression step. If you find yourself compressing, ask whether your chunks are too big.

---

## 10. Citations and grounding signals

Every production RAG should give citations. Two reasons: trust (users verify), debugging (you can see what the model used).

### 10.1 Inline citation pattern

Tell the model to cite per claim:
```
SYSTEM: After each factual claim, cite the source as [N] where N is the
chunk index from CONTEXT. Multiple citations [1][3] are allowed.
```

Output:
```
The Lyzr runtime supports three transports: REST, SSE, and WebSocket [1][2].
The default timeout is 30 seconds [3].
```

Post-process: parse `[N]` markers, look up the chunk's metadata, render as links.

### 10.2 Block citation
Append a "Sources" list at the end. Less granular but lower cognitive load on the model.

### 10.3 Span-level citation (advanced)
Have the model copy the relevant span verbatim alongside its claim. Easier to verify; more tokens.

### 10.4 Forced grounding via structured output
Use JSON schema-constrained output:
```json
{
  "answer": "...",
  "citations": [{"chunk_id": 1, "quote": "..."}]
}
```

Models follow schemas more reliably than prose instructions for citation. Use OpenAI structured outputs, Anthropic tool use, or grammar-constrained decoding.

---

## 11. Repacking — chunk → context blocks

Sometimes you don't want to send the chunk verbatim. Two common transformations:

### 11.1 Parent-window expansion
You retrieved a small chunk for precision; now expand to the surrounding section before sending to the LLM. (Doc 02's parent-child / sentence-window pattern.) The reranker still operates on the small chunk; only the final context uses the parent.

### 11.2 Deduplication / merging
After fusion + rerank, two chunks may be near-paraphrases. Drop the lower-scored one. If they're from the same doc and adjacent, merge them into one block.

### 11.3 Cross-doc grouping
Group chunks by source doc. Order docs by max-chunk score. Within a doc, keep chunk order. The model reads each source as a coherent unit.

---

## 12. Token budget arithmetic

You have a fixed prompt budget. Spend it deliberately.

```
Total prompt = system + history + context + query
```

For a typical 200k-context model with a 4k output reservation:
- System / instructions: 500–1500 tokens
- Conversation history: 1k–10k (truncated/summarized as it grows)
- Retrieved context: 2k–20k (10 chunks × 200–500 tokens)
- Query + scaffolding: ~200 tokens

**Stop adding chunks at the budget.** Don't truncate mid-chunk — drop the lowest-scored chunk entirely. A truncated chunk is worse than no chunk.

Count tokens with the **target model's** tokenizer (`tiktoken` for OpenAI, `anthropic.tokenizer` for Claude). They differ — your chunk that's 480 OpenAI tokens may be 540 Claude tokens. Plan for the larger.

---

## 13. Caching

Two huge wins:

### 13.1 Embedding cache
Hash the query (or the rewritten query) → if seen recently, reuse the embedding. Saves the embedding API call. Trivial Redis lookup.

### 13.2 Prompt cache (Anthropic / OpenAI / Gemini)
The system prompt + tools + (often) the retrieved context can be cached. Marks: cache breakpoint after the slow-changing content.

**For RAG specifically**, the retrieved context changes per query — so prompt caching doesn't help on a per-call basis. But:
- **Long static system prompts** with examples / tool specs / corpus-wide instructions: cache them.
- **Same query repeated** (popular questions, autocomplete suggestions): cache the full response.
- **Conversational RAG** where the conversation history grows: cache the prefix as it stabilizes.

Anthropic prompt cache: 5-min TTL, 90% cost discount on cache hits. OpenAI: similar. At scale this is real money — measure cache hit rate as a KPI.

---

## 14. Failure modes and what causes them

| Symptom | Likely cause |
|---|---|
| Right chunks retrieved, wrong answer generated | Bad ordering (lost in middle); chunks too long; missing system instruction "use ONLY context" |
| Model hallucinates despite good chunks | Threshold not enforced — empty/irrelevant chunks slipping through; system prompt allows external knowledge |
| Citations are wrong | Indices not stable across rerank stages; model copying numbers from chunk text instead of context labels — use distinct labels like `[doc-1]` not `[1]` |
| Cross-encoder slow | Running on CPU; batch size too low; chunks too long |
| Quality dropped after deploy | Reranker swapped or model version bumped; threshold no longer calibrated |
| LLM picks the obviously-wrong-but-confident chunk | Reranker over-ranks chunks with strong topical surface words; tighten prompt to "if multiple chunks conflict, prefer the more specific one" |
| Same chunk appears twice in context | Dedupe step missing after fusion |

---

## 15. Keyword cheat-sheet

- **Cross-encoder** — model that scores (query, doc) jointly. The reranker.
- **Bi-encoder** — model that scores via independent embeddings. The retriever.
- **Late interaction / ColBERT** — per-token scoring; middle ground between bi- and cross-encoder.
- **Cohere Rerank / BGE Reranker / Jina Reranker / mxbai** — production reranker options.
- **LLM-as-reranker** — using an LLM to order candidates.
- **Score threshold / cutoff** — minimum reranker score to keep a chunk.
- **Lost in the middle** — degraded recall for content in the middle of long contexts.
- **Parent-window expansion** — retrieve small, present large.
- **MMR** — diversity-aware ordering (also doc 05).
- **LLMLingua** — extractive prompt compression.
- **Citation grounding** — inline `[N]` references back to chunks.
- **Prompt cache** — caching the static prefix of a prompt for cost/latency.
- **Token budget** — total tokens of system + history + context + query.

---

## 16. What you take into doc 07

You now have ordered, well-positioned context. The next decision is the system prompt itself: how do you instruct the model to use this context faithfully, refuse when it can't, and produce structured output that downstream code can trust?

Generation is where you finally cash in everything you built upstream. Or where you throw it away if the prompt is sloppy.

Next: [07 — Generation and Citations](./07-generation-and-citations.md)
