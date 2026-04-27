# 03 — Embeddings

> Turning text into numbers a computer can compare.

If chunking decides *what* you store, embeddings decide *how* you find it. Get this layer wrong and no amount of reranking saves you — the right chunk has to land in the candidate set first.

---

## 1. What an embedding actually is

An embedding is a fixed-length vector of floats that represents the meaning of a piece of text.

```
"The cat sat on the mat."
        │
        ▼  (embedding model)
        │
[0.0123, -0.0481, 0.2103, ..., 0.0017]   ← 1024 floats
```

Two texts with similar meaning end up with vectors that point in similar directions. That is the entire premise. You search by computing the embedding of the query and finding the chunks whose vectors are nearest.

**Key properties:**
- **Deterministic** — same input → same vector (if the model and version are pinned).
- **Dense** — every dimension is non-zero (vs. sparse vectors like BM25/SPLADE which are mostly zero).
- **Fixed dimension** — every text, no matter the length, produces a vector of size `d` (commonly 384, 768, 1024, 1536, 3072).
- **Continuous** — small change in input → small change in vector. (Mostly. Embeddings are not perfectly smooth, but close enough to be useful.)

**What an embedding is *not*:**
- Not a hash. Two completely different texts can land near each other if they share semantics.
- Not the model's internal token embeddings. Those are per-token and contextless. A sentence embedding is derived from a forward pass through an encoder.
- Not magic. A bad model on out-of-domain text gives garbage vectors that look fine to you (they are floats!) but cluster wrong.

---

## 2. How embedding models are built

Three families:

### 2.1 Bi-encoders (the standard)

A transformer encoder (BERT-style or decoder-only with last-token pooling) processes text once. Output is pooled (mean, CLS, or last-token) into one vector.

```
Query → encoder → vector_q
Doc   → encoder → vector_d
score = cosine(vector_q, vector_d)
```

- **Pro:** Doc vectors precomputed once. Query is a single forward pass. Search is fast vector lookup.
- **Con:** The model never sees the query and document together. It scores by proximity in a learned space, which loses fine-grained interaction.

This is what you use for retrieval. OpenAI, Cohere, Voyage, BGE, E5, Jina, Nomic — all bi-encoders.

### 2.2 Cross-encoders (rerankers)

The model takes `[query, doc]` together and produces a single relevance score.

```
[query || doc] → encoder → score
```

- **Pro:** Far more accurate. The model sees both texts and computes attention across them.
- **Con:** No precomputation. You must run a forward pass per (query, doc) pair at query time. ~50–500ms per pair. Unusable for scanning millions of docs.

You use cross-encoders to **rerank** the top 20–100 candidates a bi-encoder retrieves. Covered in doc 06.

### 2.3 Late-interaction (ColBERT)

A hybrid. Encode each token of query and doc separately, then at search time compute MaxSim per query token over doc tokens.

```
For each query token q_i:
    score_i = max over doc tokens d_j of (q_i · d_j)
Total score = sum(score_i)
```

- **Pro:** Per-token interaction → cross-encoder-like quality at much lower cost. Strong on long, complex docs.
- **Con:** ~100× more storage (one vector per token, not per chunk). Specialized index (PLAID). Operationally heavier.

ColBERT v2, ColPali (multimodal), Jina ColBERT v2 are the production-ready options. Use when your bi-encoder + reranker stack hits a quality ceiling and you can afford the storage.

---

## 3. The model landscape (2024–2026)

### 3.1 Closed-source / API

| Model | Dim | Max input | Price /1M tokens | Notes |
|---|---|---|---|---|
| OpenAI `text-embedding-3-small` | 1536 (Matryoshka to 512) | 8191 | $0.02 | Default cheap option. Solid English. |
| OpenAI `text-embedding-3-large` | 3072 (Matryoshka to 256–3072) | 8191 | $0.13 | Best OpenAI quality. Heavier. |
| Cohere `embed-english-v3.0` / `embed-multilingual-v3.0` | 1024 | 512 (truncates!) | $0.10 | Strong on retrieval; supports `input_type` (query vs. document). |
| Cohere `embed-v4.0` | up to 1536 | 128k | $0.12 | Long-context, multimodal (text+image). |
| Voyage `voyage-3` / `voyage-3-large` | 1024 / 1024 | 32k | $0.06 / $0.18 | Top of MTEB on many tasks. Good for code. |
| Voyage `voyage-code-3` | 1024 | 32k | $0.18 | Code-specialized. |
| Google `text-embedding-004` / `gemini-embedding-001` | 768 / 3072 | 2k–8k | low | Strong multilingual. |
| Mistral `mistral-embed` | 1024 | 8k | $0.10 | EU data residency option. |

### 3.2 Open-source (self-host)

| Model | Dim | Size | Notes |
|---|---|---|---|
| BGE-M3 (BAAI) | 1024 | 568M | Dense + sparse + ColBERT-style multi-vector in one model. Multilingual. The Swiss Army knife. |
| BGE-large-en-v1.5 | 1024 | 335M | Pure English, strong baseline. |
| E5-mistral-7b-instruct | 4096 | 7B | LLM-as-encoder. Top quality, expensive to run. |
| nomic-embed-text-v1.5 | 768 (Matryoshka 64–768) | 137M | Apache 2.0, 8k context. Good for cost-sensitive self-host. |
| jina-embeddings-v3 | 1024 (Matryoshka) | 570M | Strong multilingual, task-conditioned. |
| GTE-large-en-v1.5 | 1024 | 434M | Long context (8k) at modest size. |
| arctic-embed-m-v1.5 (Snowflake) | 768 | 109M | Compact, strong on retrieval. |
| stella_en_1.5B_v5 | 8192 (Matryoshka) | 1.5B | Top open MTEB scores, large. |

**Picking from open-source:**
- Tight latency/RAM budget → `nomic-embed-text-v1.5` or `arctic-embed-m`.
- Multilingual → `BGE-M3` or `jina-embeddings-v3`.
- Highest quality, willing to spend GPU → `stella_en_1.5B_v5` or `E5-mistral-7b`.

---

## 4. Dimensions — pick your size

Dimension count drives **storage, RAM, and ANN index speed**. Quality scales sub-linearly.

| Dim | 1M chunks (float32) | 1M chunks (int8) | Notes |
|---|---|---|---|
| 384 | 1.5 GB | 384 MB | Tiny, very fast. Fine for English-only, simple retrieval. |
| 768 | 3 GB | 768 MB | Sweet spot for many open models. |
| 1024 | 4 GB | 1 GB | Common modern default (BGE, Voyage, Cohere). |
| 1536 | 6 GB | 1.5 GB | OpenAI small. |
| 3072 | 12 GB | 3 GB | OpenAI large. Diminishing returns vs. cost. |
| 4096 | 16 GB | 4 GB | LLM-as-encoder territory. |

**Rule of thumb:** going from 384 → 1024 typically buys 3–5 points of recall. From 1024 → 3072, often <1 point. Your money is better spent on a reranker.

### Matryoshka embeddings

Modern models (OpenAI v3, Nomic, Jina v3, Stella) train so that the **first k dimensions** of the full vector are themselves a usable embedding. You get one model that ships any dimension you slice to.

```python
full = embed("text")          # e.g., 3072 dims
small = full[:512]             # still meaningful, much smaller
```

This lets you: store the full vector for offline/batch reranking, query with a sliced version for speed, or A/B different dims without re-embedding. Always normalize after slicing.

---

## 5. Distance metrics

How "similar" gets computed once you have vectors.

| Metric | Formula | Use when |
|---|---|---|
| **Cosine similarity** | `(a · b) / (\|a\| · \|b\|)` | Default for embeddings. Direction matters, magnitude doesn't. |
| **Dot product** | `a · b` | When vectors are already normalized (then identical to cosine). Faster. |
| **L2 (Euclidean)** | `\|a − b\|` | Rare in NLP. Used in some image embeddings. |

**Practical rule:** every modern text embedding model is trained for cosine. Normalize once at write time (`v / ||v||`), then use **dot product** at query time — same result, no division per query.

```python
import numpy as np
def normalize(v):
    return v / np.linalg.norm(v)
```

Most vector DBs let you pick the metric per index. **Pick wrong → silent quality collapse.** Cohere's docs explicitly say "use cosine"; pgvector defaults to L2 — change it.

---

## 6. Query vs. document — they are not symmetric

Many models distinguish *what kind of text* you're embedding.

- **Cohere v3:** pass `input_type="search_query"` for queries, `"search_document"` for chunks. Different prompts, different vectors. Skip this and recall drops by 5–15 points.
- **E5:** prefix the text. `"query: How do RAG systems work?"` vs. `"passage: RAG systems combine retrieval with generation..."`.
- **BGE / Nomic:** similar prefix or instruction system.
- **Voyage / OpenAI:** symmetric — same model call for both.

This is the single most common embedding mistake. Read the model card. If there's an `input_type` or prefix convention, **use it on every embed call**.

---

## 7. The MTEB benchmark

[MTEB (Massive Text Embedding Benchmark)](https://huggingface.co/spaces/mteb/leaderboard) is the standard scoreboard. ~56 datasets across retrieval, reranking, classification, clustering, semantic similarity.

**How to read it:**
- Filter to **Retrieval** tab — the others (classification, STS) don't predict RAG quality.
- Look at NDCG@10 average across retrieval datasets. That's the most RAG-relevant single number.
- Cross-check on the specific dataset closest to your domain (e.g., FiQA for finance, NFCorpus for medical, TREC-COVID for biomedical).
- Check the **size** column. A 7B model on top is irrelevant if you can only run a 500M model.

**Caveat:** MTEB is heavily gamed. Models trained on MTEB-adjacent data score artificially high. Always evaluate on **your own data** before committing (doc 08).

---

## 8. Specialized embedders

### 8.1 Multilingual

If your corpus or queries cross languages, you need a model trained on them. Single-language models will partially work (subword token overlap helps) but recall suffers.

- **BGE-M3** — 100+ languages, dense + sparse.
- **multilingual-e5-large-instruct** — strong, well-supported.
- **Cohere embed-multilingual-v3** — 100+ languages, single API.
- **jina-embeddings-v3** — 89 languages, task-conditioned.

Test cross-lingual retrieval explicitly: query in language A, doc in language B. That's where weak models collapse.

### 8.2 Code

Code has different statistics than prose: identifiers, syntax, repeated keywords. Use a code-aware model.

- **voyage-code-3** — top of HumanEval-retrieval style benchmarks.
- **jina-embeddings-v2-base-code** — open-source.
- **CodeBERT / GraphCodeBERT / UniXcoder** — older, still usable.

For code search you usually combine: embed function-level chunks (with docstrings), tag with language and module, hybrid-search with BM25 over identifiers (covered in doc 05).

### 8.3 Multimodal (text + image)

- **CLIP / SigLIP** — original cross-modal embedders. Image and text into shared space.
- **Cohere embed-v4** — text + image in one call, large enough for charts and diagrams.
- **Voyage multimodal-3** — same idea.
- **ColPali** — late-interaction over PDF page images. Handles scanned docs and complex layouts without OCR.

If your corpus is "PDFs full of charts," ColPali or a multimodal embedder often beats a text-only pipeline by a wide margin.

---

## 9. Sparse embeddings (and hybrid)

Dense embeddings capture meaning. **Sparse embeddings** capture exact terms.

- **BM25** — classical, no model. Term frequency + document length normalization. Decades-old, still a strong baseline.
- **SPLADE** (and SPLADE++) — neural sparse model. Outputs a sparse vector over the vocabulary, expanded with related terms the doc didn't literally contain. Best of both worlds.
- **uniCOIL, TILDE** — other neural sparse approaches.

Why care: dense models miss exact-match queries (product codes, error strings, rare names). Sparse catches them. Production RAG nearly always **fuses** dense + sparse — see doc 05 (RRF, hybrid retrieval).

BGE-M3 outputs both dense and sparse from one forward pass — compelling if you want both without running two models.

---

## 10. Fine-tuning embeddings

When to consider it:
- Your domain has heavy jargon (legal, medical, internal codenames) that no general model has seen.
- MTEB-leader models score ~0.4 NDCG@10 on your eval set and you've exhausted other knobs.
- You have ≥1k labeled (query, relevant_doc) pairs, ideally with hard negatives.

**Methods:**
- **Contrastive fine-tuning** (the standard) — train so positives are close, negatives are far. Loss: InfoNCE / MultipleNegativesRanking.
- **Sentence-Transformers** library — easy entry point. Wraps Hugging Face models.
- **LoRA on the embedding model** — cheap, few hundred MB of adapter weights.

**Hard-negative mining** is the secret sauce. Easy negatives (random docs) are uninformative. Hard negatives are docs the *current* model thinks are relevant but aren't. Re-mine periodically as the model improves.

**Synthetic data:** if you don't have labeled queries, use an LLM to generate (query, doc) pairs from your corpus. Quality varies; validate on held-out real queries.

**Cost:** fine-tuning a 500M-param model with 10k pairs on a single A100 takes hours, not days. Recall lift in-domain: typically 5–15 points NDCG@10. Worth it once you've ruled out cheaper improvements.

---

## 11. Operational concerns

### 11.1 Versioning

When you switch embedding models or fine-tune, **all existing vectors are invalidated**. You cannot mix vectors from different models in the same index — the spaces are incompatible.

**Pattern:** tag every vector with `embed_model_version`. To migrate:
1. Stand up a new index.
2. Backfill by re-embedding the corpus.
3. Dual-write during cutover.
4. Switch reads.
5. Drop old index.

For a 10M-chunk corpus at ~$0.02/1M tokens × ~500 tokens/chunk = ~$100 for OpenAI small. Self-hosted = a few GPU-hours. Budget for this — it will happen 2–3 times in any serious project.

### 11.2 Throughput and batching

Embedding APIs accept batches (Cohere up to 96, OpenAI up to 2048 inputs per call). Batch aggressively at ingest time — single-input calls waste latency and money.

For self-hosted: a single A10G GPU runs BGE-large at ~500 chunks/sec at batch 32. 10M chunks = ~5.5 hours. Plan accordingly.

### 11.3 Truncation

Most embedders have a max input (512, 2048, 8192 tokens). **Anything past the limit is silently truncated** — you get a vector for the first chunk only.

Always check: `len(tokenizer.encode(text)) <= model.max_input`. If your chunks exceed the limit, your chunking is wrong.

### 11.4 Cost mental model

For a typical RAG corpus of 1M chunks × 500 tokens:
- **OpenAI small** ($0.02/1M tokens): ~$10 to embed once. Trivial.
- **OpenAI large** ($0.13): ~$65.
- **Voyage-3-large** ($0.18): ~$90.
- **Self-host BGE on A10G**: ~$10–30 of GPU time. Wins at >5M chunks or frequent re-embedding.

**Query-time cost** is usually 100–1000× lower than ingest because each query is one short embedding.

---

## 12. Quantization at the vector level

Storing 1024 floats × 4 bytes = 4 KB per chunk. At 100M chunks, that's 400 GB of RAM for an in-memory ANN index. Too much.

Three compression options (vector DBs apply these for you, but know what's happening):

| Technique | Size reduction | Quality loss |
|---|---|---|
| **float32 → float16** | 2× | <1% recall |
| **float32 → int8** | 4× | 1–3% recall |
| **Binary quantization (1 bit per dim)** | 32× | 5–15% recall, recoverable with reranking |
| **Product Quantization (PQ)** | 8–64× | varies, used by IVF-PQ |

Common production stack: **store full-precision** (or int8) on disk, **build the ANN index on a quantized version** for fast filtering, then **rerank top-K with full-precision vectors** or a cross-encoder. Binary + reranking is a strong cost/quality point at very large scale.

---

## 13. The keyword cheat-sheet

- **Bi-encoder** — dual encoder, embeds query and doc independently.
- **Cross-encoder** — joint encoder, scores (query, doc) pair.
- **Late interaction / ColBERT** — per-token vectors, MaxSim scoring.
- **Dense embedding** — every dim non-zero, fixed size.
- **Sparse embedding** — mostly zero, vocab-sized (BM25, SPLADE).
- **Bag-of-vectors** — multi-vector representation (ColBERT, ColPali).
- **Pooling** — mean / CLS / last-token; how you collapse a sequence to one vector.
- **Normalization** — dividing by L2 norm so cosine ≡ dot product.
- **Matryoshka** — embeddings whose prefixes are themselves valid embeddings.
- **Symmetric / asymmetric retrieval** — whether query and doc use the same encoder/prompt.
- **MTEB** — standard benchmark; care about the Retrieval tab.
- **Hard negatives** — confusable wrong answers used for contrastive training.
- **Quantization** — fp16 / int8 / binary / PQ; trades recall for size.

---

## 14. What to take into doc 04

You now have vectors. The next problem: at 100M vectors, you can't compare the query against every one of them. You need an **approximate nearest neighbor (ANN) index** and a place to put it. That is the vector store.

Decisions you've effectively already made:
- **Dimension** locks the storage cost per chunk.
- **Distance metric** locks the index type.
- **Sparse-or-not** locks whether you need a hybrid index too.
- **Self-host vs. API** locks your latency floor and your bill.

Carry these into the next doc.

Next: [04 — Vector Stores](./04-vector-stores.md)
