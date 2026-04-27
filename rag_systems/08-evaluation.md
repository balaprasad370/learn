# 08 — Evaluation

> If you can't measure it, you'll regress it. The part of RAG most teams skip and regret.

Every change you make from doc 02 onward — chunking, embedding model, hybrid weight, reranker, prompt — moves quality somewhere. Without evaluation, every "improvement" is vibes. With it, you can A/B confidently and roll back when a deploy breaks recall.

This doc is the playbook: what to measure, on what data, with what tooling, and at what cadence.

---

## 1. Two evaluations, not one

RAG splits into **retrieval** and **generation**, and each needs its own metrics. Conflating them is the most common eval mistake — a regression in answer quality could be retrieval, generation, or both, and you can't tell from a single end-to-end score.

```
                ┌──────────────┐         ┌──────────────┐
query ─────────►│  Retrieval   │ chunks ►│  Generation  │ answer ──►
                └──────────────┘         └──────────────┘
                       ↑                         ↑
                eval with Recall@K,       eval with faithfulness,
                MRR, nDCG, hit rate       answer relevance,
                                          citation precision
```

Pipeline-level evals catch drift; component-level evals tell you *which* component drifted.

---

## 2. The eval dataset — the foundation

Without a labeled eval set, none of the rest matters. Three rules:

1. **Real queries, not synthetic.** Pull from production logs, support tickets, user interviews. Synthetic queries drift from reality.
2. **Diverse.** Cover easy, hard, multi-hop, refusal, ambiguous, multilingual (if relevant).
3. **Stable.** Once labeled, freeze. Versioned with `eval_set_v1.json`, `v2.json`. Don't edit in place — your trend lines become meaningless.

### 2.1 What each eval row contains

```json
{
  "id": "q-0042",
  "query": "How do I configure HNSW parameters in pgvector?",
  "expected_answer": "Set m and ef_construction in WITH clause...",
  "relevant_chunk_ids": ["chunk-1234", "chunk-5678"],
  "metadata": {
    "category": "how-to",
    "difficulty": "medium",
    "language": "en",
    "requires_multi_hop": false
  }
}
```

- `relevant_chunk_ids` — the ground truth for retrieval metrics. Required.
- `expected_answer` — for answer-quality metrics. Optional but valuable.
- `metadata` — slice metrics by category. The single most valuable thing once you have >100 queries.

### 2.2 How big

- **Smoke test (CI):** 20–50 queries. Fast, runs every PR.
- **Full eval (per release):** 200–1000 queries. Runs nightly or pre-deploy.
- **Production drift:** 50 sampled live queries / day, scored by LLM-as-judge or human spot-check.

You can start with **30 hand-curated queries**. Most teams who say "we don't have an eval set" really mean "we never sat down for an afternoon to write 30 queries."

### 2.3 Building it without ground truth

Chicken-and-egg. You can't label relevance without reading every chunk. Three shortcuts:

**Bootstrap from existing Q&A.** FAQs, docs with sections, support tickets with resolutions — each is a (query, answer-doc) pair. Auto-labels.

**LLM-generated synthetic Qs from chunks.** For each chunk, prompt an LLM: "write a question that this chunk answers." The chunk is the relevant chunk. Caveat: synthetic distribution skews toward what the chunk literally says — under-represents real user phrasing.

**Human-in-the-loop labeling.** Pick 50 production queries. For each, surface top-10 retrieval results, have a human (you, a domain expert) mark relevant/not. 2–3 hours, lasts months.

The right answer is "all three." Synthetic for breadth, FAQ-derived for grounding, human-labeled for the ground truth that you trust.

---

## 3. Retrieval metrics

You retrieve top-K chunks. The eval checks how well your top-K covers the labeled-relevant chunks.

### 3.1 Recall@K
Fraction of relevant chunks appearing in the top-K.

```
Recall@10 = |retrieved_top_10 ∩ relevant| / |relevant|
```

The single most important retrieval metric. If Recall@10 is 0.6, no reranker on earth saves you — the answer wasn't in the candidate set.

Track at multiple K: Recall@5, @10, @50, @100. The slope tells you whether your over-fetch budget is enough.

### 3.2 Precision@K
Fraction of top-K that's relevant.

```
Precision@10 = |retrieved_top_10 ∩ relevant| / 10
```

Mostly useful when you'd send all top-K to the LLM verbatim. With reranking it matters less; recall + reranker quality together do the work.

### 3.3 MRR (Mean Reciprocal Rank)
Rank of the first relevant result, averaged over queries.

```
MRR = mean(1 / rank_of_first_relevant)
```

MRR=1.0 means the first hit is always relevant. MRR=0.2 means the first relevant result averages position 5. Good for "the model only reads top-1" workflows.

### 3.4 nDCG@K (Normalized Discounted Cumulative Gain)
Like Recall@K but **rewards relevant chunks at higher positions** more than lower ones, and supports graded relevance (very-relevant vs. somewhat-relevant).

```
DCG@K = Σ (rel_i / log2(i+1))   for i in 1..K
nDCG@K = DCG@K / ideal_DCG@K
```

Best metric when you have graded relevance (0=irrelevant, 1=somewhat, 2=highly). For binary labels, Recall@K is usually enough.

### 3.5 Hit Rate / Hit@K
Did *any* relevant chunk appear in top-K? Binary per query, averaged.

```
Hit@10 = mean(1 if any relevant in top_10 else 0)
```

The simplest, most intuitive metric. Pair with Recall@K for the full picture.

### 3.6 Targets

| Metric | Bad | OK | Good | Excellent |
|---|---|---|---|---|
| Hit@10 | <0.7 | 0.7–0.85 | 0.85–0.95 | >0.95 |
| Recall@10 | <0.5 | 0.5–0.7 | 0.7–0.85 | >0.85 |
| MRR@10 | <0.3 | 0.3–0.5 | 0.5–0.7 | >0.7 |
| nDCG@10 | <0.4 | 0.4–0.6 | 0.6–0.8 | >0.8 |

These are rough — domain matters. Customer-support FAQ retrieval can hit 0.95 Recall@10. Open-domain biomedical QA struggles to crack 0.7. Build the trend on your own data; absolute targets are guidance.

---

## 4. Generation metrics

Now you have an answer. How good is it, given the retrieved context?

### 4.1 Faithfulness (groundedness)
Are all claims in the answer supported by the retrieved context?

How to measure:
- **LLM-as-judge:** prompt a strong model with `(claims, context) → list_of_unsupported_claims`. Faithfulness = 1 − (unsupported / total claims).
- **NLI-based:** for each claim, run a Natural Language Inference model on `(context, claim)` → entailment / neutral / contradiction. Score = fraction entailed.

Faithfulness is the **anti-hallucination KPI**. Track it ruthlessly. Production target: >0.9.

### 4.2 Answer relevance
Does the answer actually address the user's question?

LLM-as-judge: "On a 1–5 scale, how directly does this answer address the question?" Average across the eval set.

A faithful answer that answers the wrong question scores low here. Catches refusals, off-topic responses, partial answers.

### 4.3 Context relevance / context precision
Of the chunks you retrieved and sent to the LLM, what fraction was actually useful?

LLM-as-judge per chunk: "Does this chunk contain information that helps answer the query?" Precision = useful_chunks / total_chunks.

Low context precision → retrieval is over-fetching irrelevant material. The LLM is doing too much filtering work. Tighten K or improve reranker.

### 4.4 Context recall
Of the information needed to answer the question, what fraction did the context contain?

Compare expected_answer to retrieved context. LLM-as-judge: "What fraction of the facts in the expected answer appear in the context?"

This is the **end-of-pipeline retrieval check** — answers "did we retrieve enough to answer correctly," even if the chunk-level labels are imperfect.

### 4.5 Answer correctness
Compares generated answer to expected_answer.

Three levels:
- **String match / F1** — for short factual answers ("Paris", "1989"). Brittle for prose.
- **Semantic similarity** — embed both, cosine. Cheap, OK signal.
- **LLM-as-judge** — "On a 1–5 scale, how correct is the candidate answer compared to the reference?" Most reliable for long-form.

### 4.6 Citation metrics
- **Citation precision:** fraction of citations that point to chunks actually supporting the claim.
- **Citation recall:** fraction of claims that have a citation when one is available in context.
- **Citation accuracy:** fraction of cited chunks that contain the cited fact (verifiable via span match).

These are often skipped and shouldn't be — citation drift is one of the most common silent regressions.

### 4.7 Refusal correctness
Two slices of your eval set:
- **Answerable queries:** is refusal rate low? (Should be ~0% on these.)
- **Unanswerable queries:** is refusal rate high? (Should be ~100% on these.)

Build at least 20 unanswerable queries: questions about topics not in your corpus, factual queries with deliberately misleading retrieved chunks. Your refusal rate on this slice is a critical safety KPI.

---

## 5. RAGAS — the framework

[RAGAS](https://docs.ragas.io/) is the most-used open-source RAG eval library. Implements the metrics above with sensible defaults.

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness, answer_relevancy,
    context_precision, context_recall,
    answer_correctness,
)

results = evaluate(
    dataset=eval_dataset,   # rows with question, contexts, answer, ground_truth
    metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall,
        answer_correctness,
    ],
)
```

Output is per-query and aggregate scores. Plug into a notebook or your CI.

Caveats:
- RAGAS uses LLM-as-judge for most metrics → **costs money** per eval run. ~$1–10 for a 200-query run depending on model.
- Judge model choice matters. Default is GPT-4-class; results can drift if you change judges.
- Pin the judge model for trend continuity.

Alternatives / complements:
- **TruLens** — broader observability + eval, with feedback functions.
- **DeepEval** — pytest-style assertions for RAG.
- **Phoenix (Arize)** — eval + tracing, good for production.
- **LangSmith** (LangChain) — managed eval + traces, paid.
- **Braintrust** — managed evals, comparison views.

---

## 6. LLM-as-judge — the hidden complexity

Most RAG quality metrics today are LLM-graded. That's powerful and dangerous.

### 6.1 What works
- Long-form answer scoring with a clear rubric.
- Faithfulness checks (claim-by-claim entailment).
- Pairwise comparison ("which answer is better?") — more reliable than absolute scoring.

### 6.2 What breaks
- **Position bias** — judge prefers the answer it sees first. Mitigation: randomize order, run both orderings, average.
- **Verbosity bias** — judge prefers longer answers. Mitigation: pairwise + explicit rubric mentioning conciseness.
- **Self-preference** — GPT-judge over-rewards GPT-generated answers; same for Claude. Mitigation: use a **third-party judge** for cross-model evaluation.
- **Score drift** — same prompt, same input, different scores across judge model versions. Pin the judge version.
- **Calibration** — the judge's "4 out of 5" doesn't mean what you think. Build your intuition on a few hundred manually-rated examples first.

### 6.3 Pairwise vs. absolute
Pairwise ("A or B?") is more reliable than absolute ("score 1–5"). Use it for ranking experiments. Use absolute for tracking a single system over time.

### 6.4 Human eval as the calibration anchor
Once a quarter, hand-grade 50 samples and compare to the judge. If the judge correlates >0.7 with human, trust the trend. If <0.5, your judge is broken — try a different model or prompt.

---

## 7. Eval cadence

Three loops at three speeds.

### 7.1 CI smoke (every PR)
- 20–50 queries, deterministic where possible.
- Retrieval-only: Hit@10, Recall@10. Generation-only on a few queries to catch prompt regressions.
- Runs in <2 minutes. Blocks merge if metrics drop >2 points.

### 7.2 Pre-deploy full (per release / nightly)
- 200–1000 queries.
- Full pipeline: retrieval + generation + faithfulness + answer relevance.
- Compare to the last release. Surface deltas per slice (category, difficulty, language).
- Reviewed before any production rollout.

### 7.3 Production drift (continuous)
- Sample 1–5% of live queries.
- Score with LLM-as-judge async (don't block the user).
- Track rolling daily and weekly metrics on a dashboard.
- Alert on: faithfulness drop >5pts, refusal rate spike or drop, latency p95 spike.

Drift is real. Index grows, embeddings stale, models update upstream, user query distribution shifts. A static deploy will silently regress.

---

## 8. The slice analysis

Aggregate metrics hide the action. Always slice.

```
Hit@10 by category:
  how-to:       0.92  ✓
  factual:      0.88  ✓
  comparison:   0.61  ⚠
  multi-hop:    0.42  ✗
  refusal:      n/a   (separately tracked)
```

That comparison/multi-hop slice tells you exactly where to focus next: enable decomposition (doc 05.6.2) for compound queries.

Standard slice axes:
- Query category (how-to, factual, comparison, multi-hop)
- Difficulty (easy/medium/hard)
- Language
- User segment (free vs. enterprise, new vs. returning)
- Doc type that the answer should come from
- Time of day / load
- Tenant (multi-tenant systems — one tenant's quality often differs)

---

## 9. The A/B / experimentation loop

When you change something, run the change against the unchanged baseline on the **same eval set** and report the delta.

```
                Baseline   Treatment   Δ
Hit@10           0.872      0.901      +0.029
Recall@10        0.745      0.782      +0.037
nDCG@10          0.681      0.715      +0.034
Faithfulness     0.913      0.918      +0.005
Answer rel.      0.847      0.835      −0.012
Cost/query       $0.022     $0.031     +$0.009
P95 latency      820ms      1140ms     +320ms
```

A change is "ship" if quality up, latency/cost acceptable, no slice regresses badly. The discipline of running this table before every deploy is what turns RAG from craft to engineering.

For online A/B (real users, not eval set): use shadow mode first (run treatment in parallel, log, don't show to user), then ramp traffic 1% → 10% → 50% → 100%. Track CTR, refusal rate, latency, downstream conversion.

---

## 10. Continuous improvement loop

```
1. Look at the latest production drift dashboard.
2. Find the worst slice (e.g., refusal rate dropped on enterprise tenant).
3. Sample 10 failing queries.
4. Diagnose: retrieval miss? bad rank? bad answer? prompt issue?
5. Form a hypothesis. Add the failing queries to the eval set.
6. Implement a fix.
7. Run the new eval set. Compare deltas across slices.
8. Ship if better; rollback if worse.
9. Repeat next week.
```

Two cultural points:
- **The eval set must grow.** Every production failure becomes a test row. After 6 months you have hundreds of curated edge cases.
- **The eval set must be honest.** Don't rewrite hard queries to make scores go up. The point is to catch what's broken, not to feel good.

---

## 11. Tracing — single-query observability

Per-query tracing is the debugging side of evaluation. For each request, capture:

- The user query verbatim.
- All rewrites / decompositions.
- Retrieved chunks (IDs + scores) per stage (BM25, dense, fused, reranked).
- Final context sent to the LLM.
- LLM input + output (with reasoning if visible).
- Latency per stage.
- Cost per stage.
- Final answer + citations.
- (Optional) async LLM-as-judge score on faithfulness.

Tools:
- **Langfuse** — open-source, full LLM tracing.
- **Phoenix (Arize)** — open-source, OpenTelemetry-native.
- **LangSmith** — paid, deep LangChain integration.
- **OpenTelemetry + Grafana / Honeycomb** — DIY, integrates with general infra.

When a user reports "the answer is wrong," tracing tells you in seconds: was the chunk retrieved? was it ranked top? did the model use it? Without tracing, every bug is a 30-minute archeology project.

---

## 12. Eval anti-patterns

| Anti-pattern | Why it's bad |
|---|---|
| One number ("our answer quality is 87%") | Hides retrieval vs. generation, hides slice failures |
| Eval set = synthetic LLM-generated only | Drifts from real users; over-rewards the very models you used to generate it |
| Editing eval rows in place | Trend lines lie; "improvements" might be relabels |
| Same model judges itself | Self-preference bias; over-reports quality |
| Eval only at deploy time | Drift goes unnoticed for weeks |
| No refusal slice | Hallucinations on out-of-corpus queries hide |
| No latency / cost in the table | You ship a quality win that doubles your bill |
| "Our users say it's better" | n=3 vibes are not eval |

---

## 13. Keyword cheat-sheet

- **Recall@K** — fraction of relevant chunks in top-K.
- **Precision@K** — fraction of top-K that's relevant.
- **MRR** — mean reciprocal rank of the first relevant result.
- **nDCG@K** — position-weighted recall, supports graded relevance.
- **Hit@K** — binary "any relevant in top-K?"
- **Faithfulness / groundedness** — claims supported by context.
- **Answer relevance** — answer addresses the question.
- **Context precision / recall** — usefulness of retrieved context.
- **Answer correctness** — generated vs. expected.
- **Citation precision / recall / accuracy** — citation-level metrics.
- **Refusal correctness** — right behavior on unanswerable queries.
- **LLM-as-judge** — using an LLM to score outputs.
- **Pairwise eval** — A-vs-B comparison; more reliable than absolute scoring.
- **RAGAS / TruLens / DeepEval / Phoenix / LangSmith** — eval frameworks.
- **Tracing** — per-query observability of every pipeline stage.
- **Slice analysis** — metrics broken down by category/segment.
- **Drift** — quality changes over time even with no code changes.

---

## 14. What you take into doc 09

You can now measure quality precisely and per-component. The next doc covers the techniques you reach for **once your baseline RAG is well-evaluated and still hitting a ceiling** — the advanced patterns.

Don't enable them blindly. Let your eval slices point at what's failing, then pick the pattern that targets that failure.

Next: [09 — Advanced Patterns](./09-advanced-patterns.md)
