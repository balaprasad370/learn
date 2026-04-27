# 02 — Ingestion and Chunking

Goal: get from "raw files in S3" to "clean, retrievable chunks with rich metadata." This is the highest-leverage stage in RAG, and the one most teams underinvest in. Bad chunks = bad retrieval, no matter how clever your embedder is.

---

## 1. The pipeline shape

```
  Source ──► Loader ──► Parser ──► Cleaner ──► Splitter ──► Enricher ──► Indexer
   │           │          │           │            │            │           │
  PDFs       fetch       text      strip       chunks       metadata     vector
  HTML       schedule    extract   noise       overlap     summaries     store
  Code       auth        OCR       dedupe      strategy    embeddings
  DB rows    polling     tables    normalize
  APIs       webhook     equations
```

Each box is a real engineering decision. Skip too many and your retrieval will struggle.

---

## 2. Source loaders

### 2.1 Common sources and their gotchas

| Source | Loader options | Gotcha |
|---|---|---|
| **PDFs** | PyPDF, pdfplumber, PyMuPDF, Unstructured, LlamaParse, Marker | Layout, tables, scanned pages, headers/footers. |
| **HTML / web** | BeautifulSoup, trafilatura, Playwright (for JS) | Boilerplate (nav, ads, sidebars). Strip aggressively. |
| **Markdown** | Native; `markdown-it-py` for AST | Easy. Preserve heading hierarchy. |
| **Code** | tree-sitter, Pygments | Don't chunk mid-function. Keep imports. |
| **Word / .docx** | python-docx, mammoth | Track-changes, comments, embedded media. |
| **Spreadsheets** | openpyxl, pandas | Each row may be its own "doc"; preserve column headers. |
| **PowerPoint** | python-pptx | Slide titles + speaker notes are gold. |
| **Confluence / Notion / Google Docs** | Their APIs | Auth + rate limits. Dedupe across versions. |
| **JIRA / Linear / GitHub Issues** | APIs | Threaded conversations; preserve order. |
| **Slack / Teams** | Export APIs | Attribution matters; preserve user + timestamp. |
| **Databases** | SQL queries | Per row → text serialization (JSON or sentence form). |
| **Audio / video** | Whisper for ASR | Transcribe first; chunk transcript. |
| **Images** | Vision models or OCR | Describe in text or use multimodal embeddings. |

### 2.2 Incremental ingestion

For production, **don't re-ingest everything every night.** Track:

- **Source ID** (URL, file path, DB primary key).
- **Version hash** (content hash or last-modified).
- **Tenant ID** if multi-tenant.
- **Last-indexed timestamp.**

On the next run, fetch only changed/new sources, re-process those, and **delete** their old chunks. Tombstone deleted source documents — your vector store should also drop those vectors.

This sounds obvious; it's where many teams accumulate stale data and silent retrieval bugs.

---

## 3. Parsing — the layer most teams underestimate

Garbage in = garbage out. The quality of text extraction sets a hard ceiling on RAG quality.

### 3.1 PDF parsing — the eternal headache

PDFs are a presentation format, not a content format. The text "January Revenue: $1.2M" might internally be three separate text objects positioned next to each other on the page. Extracting that as a coherent string requires layout analysis.

**Tier 1 — fast and dumb:**
- `PyPDF2`, `pypdf`. Good for clean digital PDFs (whitepapers, code docs). Fails on tables, multi-column layouts, scanned PDFs.
- Extracts text in arbitrary order; sometimes scrambles columns.

**Tier 2 — layout-aware:**
- `pdfplumber` — good at tables.
- `PyMuPDF` (fitz) — fast, layout-aware, extracts images.
- **Unstructured.io** — multi-format, returns typed elements (`Title`, `NarrativeText`, `Table`, `ListItem`).
- **LlamaParse** (LlamaIndex) — LLM-assisted parsing, excellent on complex PDFs.
- **Marker** — open-source, very good for academic / book PDFs to markdown.

**Tier 3 — scanned PDFs (need OCR):**
- **Tesseract** — open source, decent on clean scans, weak on layouts.
- **AWS Textract**, **Azure Document Intelligence**, **Google Document AI** — managed OCR with table/form extraction. Pay per page; high quality.
- **Mistral OCR**, **GPT-4o vision-based extraction** — LLM-driven OCR, very strong on hard documents.

**Rule:** test 10 representative documents from your real corpus through 2–3 parsers. Measure manually. The "best" parser is corpus-specific.

### 3.2 Tables in PDFs

Tables are where parsers fail most often. Three approaches:

1. **Extract as Markdown / HTML.** Modern parsers (Unstructured, LlamaParse, Marker) emit markdown tables. The LLM understands these natively.
2. **Serialize each row as a sentence.** "Q1 revenue was $1.2M, Q2 was $1.5M..." — easier for embedding.
3. **Keep table as a separate retrievable unit** with its own metadata (source, headers, surrounding context).

For numerical / financial tables, option 2 or 3 usually retrieves better than option 1 because embeddings handle prose better than tabular markup.

### 3.3 Code files

Don't treat code like prose:

- **Use AST-aware splitting** (tree-sitter). Chunk by function or class, not by character count.
- **Preserve imports** at the top of every chunk so context survives.
- **Include file path and language** in metadata.
- For documentation-style retrieval (API references), keep docstrings + signatures; don't dump entire bodies if not needed.

Specialized libraries: `langchain.text_splitter.Language` enums, `tree-sitter-languages`.

### 3.4 HTML / web pages

Web pages are mostly junk wrapped around a small core of useful content.

- **trafilatura** — purpose-built for content extraction; strips boilerplate; handles paywalls/cookies banners.
- **Readability.js** (or its Python ports) — what reader-mode browsers use.
- **BeautifulSoup with custom selectors** — when you control the source.

Always strip: navigation, ads, footers, "related articles," cookie banners, share buttons, comment threads (unless that's the content).

### 3.5 Office formats (.docx, .pptx, .xlsx)

- **.docx** — `python-docx` for structured content, `mammoth` to convert to clean Markdown/HTML.
- **.pptx** — `python-pptx`. Extract slide title + body + speaker notes. Speaker notes often have the actual content; titles alone are weak retrieval signal.
- **.xlsx** — pandas. Treat each sheet (or each row in a key sheet) as its own retrievable unit.

### 3.6 Multimodal: images, audio, video

- **Images:** describe via vision LLM (`gpt-4o`, Claude vision) and store the description as the retrievable text. Or store the image vector directly using a multimodal embedding (CLIP, SigLIP, Voyage Multimodal).
- **Audio:** Whisper or AssemblyAI for transcription. Chunk by speaker turn or 30s windows. Preserve timestamps.
- **Video:** ffmpeg → frames + audio. Transcribe audio + describe sampled frames. Or use a unified video model.

Multimodal RAG is mostly RAG-over-text-derived-from-multimedia. True end-to-end multimodal RAG (embedding raw images/audio in the same space as text and retrieving across modalities) is doc-09 territory.

---

## 4. Cleaning — the boring critical step

After parsing, run aggressive cleaning:

- **Normalize whitespace.** Collapse runs of spaces, normalize line endings.
- **Fix encoding artifacts.** `â€™` → `'`, mojibake from bad source files.
- **Remove repeated headers/footers.** "Confidential — Page X of Y" on every page hurts retrieval.
- **Deduplicate near-duplicates.** Use MinHash / SimHash. The same content appearing 50× confuses retrieval (same answer keeps winning).
- **Strip junk:** signatures, footnotes-as-noise, watermarks.
- **Language detection.** Tag each chunk; mismatched-language queries can be filtered or routed.
- **PII detection (optional).** Redact or tag for downstream policy.

The output should be **clean, semantically meaningful prose with structure preserved (headings, lists, tables).**

---

## 5. Chunking — the design decision that matters most

A "chunk" is one indexed unit. Each chunk gets one embedding and is retrieved as a unit. **Chunk size and boundary choice are the single biggest knob in RAG quality.**

### 5.1 Why chunking exists at all

Why not embed whole documents?

- **Embedding context limits.** Embedding models have their own context windows: 512–8192 tokens typically. Documents exceed this.
- **Retrieval precision.** A 50-page PDF embedded as one vector represents an average of 50 pages — too coarse to retrieve the specific paragraph that answers a question.
- **LLM context budget.** Even at long context, you don't want to dump 50-page docs; you want focused passages.

### 5.2 Chunk size — the tradeoff

- **Small chunks (100–300 tokens).** High retrieval precision (chunks are semantically focused). Risk: relevant content split across chunks; LLM lacks context.
- **Medium chunks (500–1000 tokens).** Sweet spot for most use cases. Balance of precision and context.
- **Large chunks (1500–3000 tokens).** More context per chunk; lower precision; fewer chunks fit in the LLM context budget.

There is no universal answer. The right chunk size depends on:

- **Question complexity.** "What's our refund policy?" → small chunk fine. "Compare our refund policy to our cancellation policy" → larger.
- **Document structure.** Tightly-written API docs → small chunks. Long narrative documents → larger.
- **Embedding model.** Some embedders perform better at certain lengths; check the model's training-time context.
- **Reranker presence.** With a strong reranker (doc 06), you can retrieve more chunks and let the reranker pick the best — favoring smaller chunks at retrieval time.

**Default for greenfield: 512–800 tokens with 50–100 token overlap.** Tune from there.

### 5.3 Chunk overlap

Overlap = N tokens shared between adjacent chunks. Why? A sentence that begins at the end of chunk K and ends at the start of chunk K+1 would otherwise have its meaning truncated in both.

- **0% overlap** — minimum cost; risk of meaning splits.
- **10–20% overlap** — standard.
- **>50% overlap** — wasteful; storage and retrieval bloat.

Use overlap proportional to chunk size: 100 tokens overlap on 1000-token chunks is a fine default.

### 5.4 Chunking strategies — the menu

| Strategy | How it splits | When it wins | Pitfalls |
|---|---|---|---|
| **Fixed-size character/token** | Every N tokens | Quick start; uniform corpora | Cuts mid-sentence, mid-table |
| **Recursive character** | Tries paragraphs → sentences → words | General-purpose, language-agnostic | Still arbitrary on edge cases |
| **Sentence-based** | Split on sentence boundaries (NLTK, spaCy) | Q&A on prose | Loses paragraph context |
| **Markdown-aware / structural** | Preserve sections by `#`, `##` headers | Markdown corpora, technical docs | Sections wildly variable in size |
| **Semantic chunking** | Split where embedding similarity drops | Topical narrative, blog posts | Slow; embedder-dependent |
| **Sentence-window** | Index single sentences; return surrounding window | Precise retrieval + context for LLM | More indexing complexity |
| **Parent-child / small-to-big** | Index small; return parent (larger) chunk | Best of both worlds; common in production | Two-tier storage |
| **Hierarchical / summary trees** | Multi-level summaries | Global reasoning over large corpora | Heavy preprocessing; LlamaIndex / RAPTOR |
| **Late chunking** | Embed whole doc once, derive chunk vectors | Preserves cross-chunk context | Needs long-context embedder |
| **AST-based (code)** | Function/class boundaries | Code RAG | Language-specific tooling |

### 5.5 Recursive character splitter (the workhorse default)

LangChain's default. Logic:

```
Try splitting on "\n\n" (paragraphs).
  If chunk still > max_size, try splitting on "\n" (lines).
    If still > max_size, try ". " (sentences).
      If still > max_size, try " " (words).
        Last resort: hard character cut.
Recursively merge small pieces back to fill chunks up to max_size, with overlap.
```

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,         # in characters or tokens (configurable)
    chunk_overlap=100,
    separators=["\n\n", "\n", ". ", " ", ""],
    length_function=len,    # or a tokenizer
)
chunks = splitter.split_text(document_text)
```

This is the right default. Move beyond it only after measuring.

### 5.6 Semantic chunking

Use the embedder itself to find topic boundaries:

1. Split into sentences.
2. Embed each sentence.
3. Compute cosine similarity between adjacent sentences.
4. Split where similarity drops below a threshold (or below the local 95th percentile of distances).

Pros: chunks correspond to topical units.
Cons: slower (extra embedding pass), requires tuning the threshold, can produce wildly variable chunk sizes.

LlamaIndex offers `SemanticSplitterNodeParser`. Try it on your corpus; sometimes a noticeable lift, sometimes neutral.

### 5.7 Parent-child / small-to-big

The single most useful "advanced" chunking pattern:

- **Index level (small):** 200-token chunks, embedded for retrieval precision.
- **Return level (large):** 1500-token parent chunks containing the small chunk, sent to the LLM for context.

Workflow:
```
Split doc into 1500-token parent chunks.
For each parent, split into 200-token children.
Index the children (with parent_id metadata).
At query: retrieve top-K children. Look up unique parents. Send parents to LLM.
```

Why it works: small chunks retrieve precisely (specific sentences match queries); large parent chunks give the LLM room to reason. Best of both.

LangChain: `ParentDocumentRetriever`. LlamaIndex: similar via metadata.

### 5.8 Sentence-window

Variant of small-to-big:
- Index every sentence.
- Each sentence's metadata stores its position in the document.
- At query: retrieve top-K sentences. For each, return a window of N sentences before and after as context.

Cleaner than parent-child for documents without clear paragraph structure.

### 5.9 Hierarchical (RAPTOR-style)

For corpora where global reasoning matters (long books, large reports):

1. Chunk the document normally.
2. Cluster nearby chunks; have an LLM summarize each cluster.
3. Cluster the summaries; summarize again. Repeat to a tree.
4. Index every level (chunks + cluster summaries + section summaries).
5. At query, retrieve from any level — sometimes a high-level summary is the best answer.

Pricey to build; powerful for "what are the main themes of this 500-page report" queries.

### 5.10 Late chunking

A 2024 trick that resolves a fundamental tension. Normal chunking loses cross-chunk context (a chunk doesn't know what came before). **Late chunking** does the opposite:

1. Run the *whole document* through a long-context embedder, getting one embedding per token.
2. Derive chunk embeddings by averaging the per-token embeddings within each chunk's range.

The chunk embeddings carry context from the entire document. Better for documents with strong cross-references.

Requires a long-context embedder (Jina v3, Voyage Context, Cohere Embed v4 in long mode). New technique; check the embedder's docs.

---

## 6. Metadata — the most underrated retrieval lever

Every chunk should carry rich metadata. This enables filtering, routing, ranking, and explainability.

### 6.1 Core metadata fields

| Field | Why |
|---|---|
| `source_id` | Which document. |
| `source_url` / `path` | Citations. |
| `source_type` | PDF, HTML, code, ticket, etc. |
| `title` / `section` | Restore context for LLM. |
| `created_at`, `modified_at` | Recency filtering, time-aware queries. |
| `author` | Attribution; permissions. |
| `tenant_id` | Multi-tenant isolation. |
| `language` | Mismatched-language filtering. |
| `tags` / `categories` | Domain routing. |
| `chunk_index` | Position within source. |
| `parent_chunk_id` | For parent-child retrieval. |
| `permissions` | ACL filtering. |
| `version` | Versioned docs (e.g., API v1 vs v2). |

### 6.2 Synthetic metadata (LLM-generated at ingest time)

For higher precision, run an LLM over each chunk to generate:

- **Summary** (1–2 sentences, embedded separately for high-level retrieval).
- **Keywords / entities** (for hybrid keyword retrieval).
- **Question candidates** ("what questions does this chunk answer?") — embed those instead of the chunk for retrieval; powerful trick.
- **Type tags** (definition, example, FAQ, policy, etc.).

Cost: one cheap LLM call per chunk at ingest time. Indexing cost goes up; query-time accuracy goes up more. Worth it for any high-traffic RAG system.

### 6.3 Contextual retrieval (Anthropic)

A 2024 technique that significantly raises accuracy: before embedding, prepend a short LLM-generated context to each chunk explaining what it's about, where in the document it sits, and what surrounds it.

```
[Context: This chunk is from the "Refund Policy" section of the 2026 Q1
employee handbook, immediately following the discussion of cancellation
fees.]

Refunds are processed within 7 business days of approval...
```

Embed the chunk with this prefix. Retrieval improves because each chunk's vector reflects its context within the larger document.

Anthropic reported 35–67% reduction in retrieval failures by combining contextual retrieval + hybrid + reranking. Best single improvement to advanced RAG in recent memory.

Implementation: use a cheap model (Haiku, Gemini Flash) and prompt caching to amortize the doc-level cost. Still pricey for huge corpora; selectively apply where accuracy matters.

---

## 7. Special cases

### 7.1 Tables

Three workable approaches (revisiting from §3.2):
1. **Markdown table** as the chunk text.
2. **Row-as-sentence** serialization.
3. **Hybrid:** index a textual summary, store the full table separately, retrieve the table by reference.

For numerical analysis, consider Text-to-SQL agents instead of RAG.

### 7.2 Code

- Index by function/class with file path + language metadata.
- Include docstring + signature; optionally exclude the body if your LLM is good at filling it in from the description.
- For "how do I do X?" queries, also index README / docs as separate chunks.

### 7.3 Conversations / threads

Slack threads, GitHub issue conversations, customer support tickets:
- Preserve thread structure. Retrieve the whole thread as one chunk if cohesive; otherwise chunk by message group.
- Include user IDs, timestamps, channel names in metadata.
- Embed the *summary* + first message rather than the whole thread for retrieval, then return the full thread as context.

### 7.4 Versioned documentation

API docs change. Indexing all versions equally causes the wrong version to be retrieved.

- Tag each chunk with `version`.
- Default queries to "latest" via metadata filter.
- Allow explicit version queries ("how was this in v3?") via filter dispatching.

### 7.5 Multi-language corpora

- Detect language at chunk level; store as metadata.
- Use a multilingual embedder (mE5, multilingual-E5, Cohere Embed v3 multilingual, BGE-M3).
- Allow query-language filtering or cross-lingual retrieval depending on use case.

---

## 8. Common chunking failures and fixes

| Failure | Symptom | Fix |
|---|---|---|
| Chunks cut mid-sentence | Garbled retrieved context | Use recursive splitter with sentence separator. |
| Tables fragmented | Numbers retrieved without column headers | Treat tables as atomic chunks; preserve headers. |
| Same content many times (versions, duplicates) | Same chunk wins every query | Dedupe at ingestion; add `version` metadata; filter. |
| Lost cross-chunk context | LLM answers without big-picture | Parent-child or contextual retrieval. |
| Code chunks miss imports | LLM hallucinates types | AST-based splitter; prepend imports. |
| All chunks roughly the same | Topic-mixed; weak retrieval | Semantic chunking or smaller chunks. |
| Long docs entirely uncovered | Recall fails on certain documents | Inspect parsing — likely text-extraction failure. |

---

## 9. A practical ingestion checklist

Before you ship any RAG system, walk through this:

- [ ] Source loaders are incremental (only re-process changed docs).
- [ ] Parsers tested on 10+ representative real documents from each format.
- [ ] Tables, code, equations preserved or handled explicitly.
- [ ] Cleaning normalizes whitespace, dedupes, strips boilerplate.
- [ ] Chunk size measured against your query patterns, not guessed.
- [ ] Overlap configured (10–20% typical).
- [ ] Strategy chosen with rationale (recursive default; semantic/parent-child if measured benefit).
- [ ] Every chunk carries source_id, source_url, type, timestamps, tenant_id.
- [ ] Permissions/ACL captured if multi-tenant or RBAC.
- [ ] Versioning if your sources change over time.
- [ ] Re-indexing schedule documented; old chunks deleted on source removal.
- [ ] Spot-check 50 random chunks visually before going live.

---

## 10. Summary

Ingestion and chunking are the foundation. Bad chunks set a hard ceiling on RAG quality that no amount of retrieval cleverness can overcome. Choose parsers per source format and validate on real documents. Default to recursive 500–800 token chunks with 10–20% overlap; advance to parent-child, semantic, or contextual retrieval when measurement justifies it. Carry rich metadata on every chunk — multi-tenancy, versioning, time-awareness, and citations all depend on it. Re-index incrementally; tombstone deletions; spot-check visually.

Now that we have clean chunks, we need to turn them into vectors. Next: embeddings — what they are, which model to pick, and the tradeoffs of dimension and distance metric.

---

Next: [03 — Embeddings](./03-embeddings.md).
