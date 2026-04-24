# 08 — Token Economics

Goal: a practical, engineer-level handle on counting, estimating, and reducing your token spend. Closes the folder with the stuff you'll use every day.

---

## 1. What you actually pay for

At the billing layer, every LLM API has three or four token meters:

| Meter | What it counts | Typical rate |
|---|---|---|
| **Input tokens** | Every token in the request (system + prior turns + current user) | $1–$15 / M |
| **Output tokens** | Every token the model generates | $4–$75 / M |
| **Cached input tokens** | Input tokens served from prompt cache (providers that support it) | $0.10–$1.50 / M (~10% of fresh input) |
| **Cache write tokens** | Cost to initially store a cache entry (Anthropic bills this; OpenAI doesn't) | ~1.25× normal input |

**Output is 4–5× the price of input on most hosted frontier models.** This one fact drives most optimization decisions.

Rough pricing matrix (early 2026, approximate):

| Model | In $/M | Out $/M | Cached-in $/M | Notes |
|---|---|---|---|---|
| Claude Opus 4.7 | $15 | $75 | $1.50 | Premium tier. |
| Claude Sonnet 4.x | $3 | $15 | $0.30 | Workhorse. |
| Claude Haiku 4.5 | $0.80 | $4 | $0.08 | Cheap tier. |
| GPT-5 | ~$10 | ~$30 | ~$1 | Flagship. |
| GPT-5 mini | ~$0.40 | ~$1.60 | ~$0.04 | Cheap. |
| o3 / reasoning | $15+ | $60+ (incl. hidden reasoning tokens) | — | Extra "thinking" tokens billed. |
| Gemini 2.5 Pro | $1.25–2.50 | $5–10 | $0.30–$0.60 | Cheaper at short ctx, more at long. |
| Gemini Flash | $0.10 | $0.40 | $0.025 | Very cheap. |
| Llama 3.1 70B (hosted) | $0.80 | $0.80 | — | Flat rate. Open-weights. |

Prices change; structure doesn't. Memorize the structure (in/out ratio, cache ratio, reasoning premium).

---

## 2. Counting tokens — accurately, cheaply, and when not

### 2.1 Exact count

- **OpenAI**: `tiktoken` library. `enc.encode(text)` is the canonical count. Free, fast, offline.
- **Anthropic**: `client.messages.count_tokens(...)` — one API call per count. Or Anthropic's published tokenizer in their SDK where available. Slight latency, but accurate.
- **Gemini**: `client.models.count_tokens(model, contents)`.
- **Open models (Llama, Qwen)**: `AutoTokenizer.from_pretrained(...)` from Hugging Face. Offline.

Rule: **always count before a large prompt or fine-tune**. Tokenizer surprises cause budget blow-ups.

### 2.2 Approximation when you can't count

| Content type | Chars / token (rough) |
|---|---|
| English prose | 4.0 |
| English + punctuation | 3.8 |
| Source code | 3.3 |
| JSON (schema-heavy) | 2.8 |
| Base64, hex | 2.0 |
| Japanese, Chinese | 1.5 |
| Hindi, Arabic, Thai | 2.5 |
| Emoji-heavy | 1.0 |

Rule of thumb for English chat: **total_tokens ≈ total_chars / 4**. A 10 KB English doc ≈ 2500 tokens.

### 2.3 Don't count for every request

On the hot path, counting every prompt adds latency. Sample: count 1-in-100 requests for cost-accounting, and trust the provider's response metadata for the other 99. Every provider returns `usage: {input_tokens, output_tokens}` in the response. That's the billable amount, exactly.

---

## 3. Cost estimation before you run

For a new feature:

```
cost_per_call = (input_tokens × input_rate) + (output_tokens × output_rate)

monthly_cost = cost_per_call × calls_per_user × users_per_month × retry_multiplier
```

Worked example: chatbot serving 10 000 users, each sending 20 messages/month:

- Each message: 2 000 input tokens (history + system) + 300 output tokens.
- Model: Claude Sonnet 4.x ($3/M in, $15/M out).
- Per message: (2000 × $3/M) + (300 × $15/M) = $0.006 + $0.0045 = **$0.0105**.
- Monthly: $0.0105 × 20 × 10 000 = **$2 100/month**, before any optimizations.

Now flip to GPT-4o mini: ($0.15 + $0.60)/M. Per message: $0.0003 + $0.00018 = $0.00048. Monthly: **$96**. 22× cheaper.

The exercise above is what product/engineering should run **before** building. The right model for a given task is 10–50× cheaper than the wrong one at equal quality.

---

## 4. Prompt caching — the biggest single lever

If you only adopt one optimization from this doc, make it prompt caching.

### 4.1 What it is

When you call an LLM with a large, repeated prefix (system prompt, few-shot examples, large context), providers can cache the prefix's computed KV state (doc 05 §2) on the GPU. The next call with the same prefix skips the prefill for that portion.

Effects:
- **Cost**: cached input tokens are billed at ~10% of fresh input.
- **Latency**: prefill for the cached portion goes from seconds to tens of ms.

### 4.2 Provider details

| Provider | Cache granularity | TTL | Markup on write | Savings |
|---|---|---|---|---|
| **Anthropic** | Explicit `cache_control` markers in messages | 5 min (refreshed on hit), 1 h available | 1.25× | 90% on read |
| **OpenAI** | Automatic on prompts ≥ 1024 tokens | 5–10 min | No extra write cost | ~50% on read |
| **Gemini** | Explicit `cached_content` object, reusable across calls | Configurable (minutes to hours) | One-time cache cost | 75%+ on read |
| **Groq / Together / Fireworks** | Automatic | Varies by provider | — | Varies |

### 4.3 Anthropic caching — concretely

```python
response = client.messages.create(
    model="claude-opus-4-7",
    system=[
        {
            "type": "text",
            "text": "<very long system prompt with tools, examples, policies...>",
            "cache_control": {"type": "ephemeral"},  # ← cache this block
        }
    ],
    messages=[
        {"role": "user", "content": "What's my balance?"}
    ],
)
```

Every subsequent call with the *exact same* system block within 5 min gets 90% off that portion.

Rules:
- Cache block must be ≥ 1024 tokens (Opus/Sonnet) or 2048 (Haiku) to be eligible.
- You can cache up to 4 "breakpoints" per request (system, messages[i], etc.).
- Cache miss = full prefill + write cost. First call pays 1.25×; next 4 min cost 0.1×.
- Break even: 2 cache hits in a 5-min window beats no caching.

### 4.4 When caching pays off enormously

- **Agentic loops.** Same system prompt + tools + prior turns, called 10× in a run. Every call after the first is ~90% off on the repeated content.
- **RAG with stable "instructions" section.** Cache the instructions, vary the retrieved content.
- **Few-shot prompts.** Cache the examples; vary the query.
- **Multi-user chat** where many users hit the same bot. Each user's session runs against a cached system/instruction prefix.

### 4.5 When it doesn't help

- Unique prompts every call (no shared prefix).
- Prefixes below the minimum.
- Latency-critical single-call flows where you'll only ever call once.

A real agentic workload with 10 steps per run, 1000 runs/day, 2k system prompt:
- Without cache: 10 × 1000 × 2000 input tokens = 20M tokens/day on system alone.
- With cache (first step pays full, next 9 get 90% off within TTL): ≈ 20M × (0.1 + 0.9 × 0.1) = 3.8M effective.
- On Claude Sonnet: $60/day → $11.40/day. **~80% reduction**.

Over a year, that's thousands of dollars from five lines of `cache_control`.

---

## 5. Choosing the right model ("routing")

A production system that uses one model for every task is leaving money on the table.

### 5.1 Tiered routing

Segment traffic by complexity:

```
Incoming request
      │
      ▼
[Classifier / heuristic]
      ├── simple   → Haiku / 4o-mini / Flash  (~$0.50/M avg)
      ├── medium   → Sonnet / 4o / Gemini Pro (~$8/M avg)
      └── hard     → Opus / GPT-5 / reasoning (~$45/M avg)
```

Classifier options:
- A small cheap model (Haiku) categorizes the request.
- Heuristics (word count, keyword presence, prior classification).
- Escalation: try cheap model first; if confidence low or task flagged, retry with larger model.

Typical mix for a chat product: 70% handled by small, 25% by medium, 5% by large. Average cost drops 5×+ vs "always use frontier."

### 5.2 Reasoning vs non-reasoning routing

Reasoning models are massively more expensive per call (more output tokens, higher output rate). Use them only when:
- The task involves math, logic, or multi-step planning.
- The benchmarks for your task show a reasoning advantage.
- Latency budget allows 10–60 seconds.

For chat, summarization, and straightforward Q&A, a non-reasoning model is faster, cheaper, and often better-calibrated.

---

## 6. Reducing output tokens (the highest-leverage lever)

Because output is 4–5× input cost, cutting output cuts cost disproportionately.

### 6.1 Prompt for concision

Tell the model: "Answer in 1–2 sentences." "Respond in ≤100 words." "No preamble; answer directly."

These instructions reliably reduce output by 30–60%. Model compliance is good with modern alignment.

### 6.2 Max tokens cap

Set `max_tokens` to the length you actually need. A default of 4096 when your use case is 200-token answers is wasted headroom (doesn't directly cost if unused, but prevents pathological runaways).

### 6.3 Structured output

Asking for JSON with specified fields naturally trims explanation prose. A tool-call-style response ("call `submit_answer` with `{...}`") is much shorter than a conversational explanation.

### 6.4 Stop sequences

`stop=["\n\n---", "END_OF_ANSWER"]` lets you forcibly cut the output at a known marker. Useful when the model tends to over-explain after a good answer.

### 6.5 Remove meta-commentary

Instruct: "Don't explain your approach." "Don't ask follow-up questions." "Don't summarize at the end."

Each removes a chunk of output.

### 6.6 Streaming with early termination

In agents, as soon as the model has committed to a tool call or a "no" answer, close the stream. Saves any remaining tokens the model would have generated. Your SDK or client-side handler decides when to `abort()`.

---

## 7. Reducing input tokens

Input is cheaper per token than output, but often larger in total. Optimizations:

### 7.1 Tight system prompts

System prompts bloat quickly as features are added. Audit periodically. Remove dead instructions, consolidate examples, move rarely-used guidance to tool descriptions instead of the main prompt.

### 7.2 Summarize prior turns

In long conversations, the full history grows quadratically with turn count. Strategies:
- Summarize every N turns into a compressed "conversation so far" paragraph.
- Keep the last K raw turns verbatim.
- Drop turns that weren't referenced (requires light analysis).

Claude's prompt-caching TTL makes this less urgent for short-to-medium conversations. For long conversations (> 30 turns), summarization is still essential.

### 7.3 RAG instead of context stuffing

If your system prompt contains a big reference doc, move it to a vector store and retrieve only the relevant chunks. 200 k of full doc → 5 k of top-3 chunks. Quality usually *improves* because the model isn't distracted.

### 7.4 Trim few-shot examples

If the model has been fine-tuned for the task (or is capable enough zero-shot), few-shot examples are waste. Remove or reduce from 8 examples to 2–3.

### 7.5 Compress with structured format

Verbose prose: "The user is named John Smith and is 35 years old and lives in San Francisco and works at Acme Corp."
Compressed: `{"name":"John Smith","age":35,"city":"SF","employer":"Acme"}`

Same information, ~40% fewer tokens. The model parses JSON fine; your ops cost drops.

### 7.6 Strip markdown chrome

If you're feeding the model scraped HTML-to-markdown, it's often bloated with decorative formatting. Strip and re-format tersely.

---

## 8. Batching — when and when not

Providers offer **batch APIs** (OpenAI Batch, Anthropic Message Batches, Gemini Batch):

- Submit up to 10 k requests in a single file.
- Processed within 24 hours.
- 50% discount on input and output.

Use when:
- Offline: evals, data generation, classification of backlogs, document processing.
- Not latency-sensitive.

Don't use for:
- User-facing requests (24h SLA is too long).
- Real-time agent loops.

For async eval runs: batch API is *always* the right call. Saves half the cost with zero work.

---

## 9. Token-economics for agents

Agents are a special case because they make many calls per user action. A 5-step agent is 5× the token cost of a 1-shot chat.

### 9.1 What blows up in agents

1. **Context growth.** Each step, the model sees the prior steps' outputs and tool results. By step 5, context is 5× step 1.
2. **Tool result bloat.** A search tool returning a 10 KB JSON blob adds ~3 k tokens to every subsequent step.
3. **Planning verbosity.** Models with extended thinking may emit 1 000+ "thinking" tokens per step.
4. **Retries.** Tool failures + retries compound.

### 9.2 Agent-specific tactics

- **Cache the system prompt and tool definitions.** These are stable across steps — huge cache hit rate.
- **Truncate tool results.** Tool manifests should declare `max_output_tokens`. Transport layer truncates with a clear "<truncated>" marker.
- **Summarize between steps.** After step N, compress "what we've done" into 200 tokens; reset the raw history.
- **Budget enforcement.** Every run has a `max_tokens`, `max_dollars`, `max_steps` cap. Hit any, stop. (This is exactly what the Lyzr agent runtime folder doc 02 specifies.)
- **Step-level model routing.** Use a cheap model for "format this output" steps; frontier only for planning steps.
- **Short circuit.** If the model has produced a confident final answer, don't run more steps looking for edge cases. Agents without confidence thresholds run too long.

### 9.3 Typical agent cost

A non-optimized 5-step agent on Claude Sonnet: ~$0.20 per run.

With caching + truncation + routing: ~$0.04 per run.

At 100 k runs/month: $20 000 → $4 000. The difference is an engineer-week of optimization work, every month forever.

---

## 10. Fine-tuning vs prompting — the cost question

Fine-tuning trades upfront training cost for lower per-call cost:

- **Shorter prompts possible** (less instruction, less few-shot).
- **Smaller models viable** (a fine-tuned 8B can match a zero-shot 70B on narrow tasks).
- **Upfront cost:** $50–$5000 depending on model and data size.

Break-even for fine-tuning vs prompting with caching:

- Tenant-specific task with stable behavior: consider fine-tuning above ~100 k requests/month.
- Occasional or general-purpose: prompting + caching almost always wins. Fine-tunes drift as base models improve; prompts don't.

Rule: **don't fine-tune until you've exhausted prompting, caching, and routing.** Most token savings in production come from the boring three, not fine-tunes.

---

## 11. Self-hosting vs hosted — the cost tradeoff

Hosted (Anthropic, OpenAI, Gemini):
- Pay per token.
- Zero ops.
- Frontier quality.

Self-hosted (vLLM / Llama 70B on H100s):
- Pay per GPU-hour.
- Full ops responsibility.
- Open-weights capability only (usually 1–2 tiers behind frontier).

Break-even analysis (reference: Llama 3.1 70B at FP8):
- Self-hosted throughput: ~10 000 tokens/sec aggregate on 4×H100.
- Cost at $16/hr: $0.40/hr for 10k tok/s × 3600 = 36M tokens/hr → $0.011/M tokens amortized at full utilization.
- Hosted equivalent (Llama 70B on Together/Fireworks): ~$0.80/M tokens.
- Break-even utilization: ~1.3% (i.e., self-hosting wins easily if you can fill 2% of a GPU).

But:
- Utilization is rarely 100%. Real-world self-hosting averages 30–60% utilization.
- Ops cost (monitoring, on-call, capacity planning) is real.
- You can only self-host open-weights models (no Claude, no GPT-4).

Practical threshold: self-host above ~10 B tokens/month on a specific open model, with an ops team. Below that, hosted wins.

---

## 12. Measurement and observability

You can't optimize what you don't measure. Essential metrics:

- **Per-call cost** (input, output, cached, total), tagged by endpoint, feature, model, user tier.
- **Cache hit rate** for each cacheable prefix.
- **Output token distribution** (p50, p95, p99). Long tails = runaway generations.
- **Per-feature cost per day/week**. Watch for creep — new system prompts or tool definitions quietly push costs up.
- **Cost per user session / per active user**. The "is this feature paying for itself" metric.
- **Model mix** — what % of traffic on each tier. Drift toward expensive models is an anti-pattern.

Store usage rows per call in a database. Build a dashboard. Alert on weekly cost deltas > 20%.

This is what doc 09 in the agent runtime set specifies; applying here: **every call logs `{model, input_tokens, output_tokens, cache_read, cache_write, cost_dollars, feature, user_tier}`**. Without this, optimizations are guesswork.

---

## 13. A practical cost-reduction playbook

When your LLM bill comes in and it's too high, work through this list in order:

1. **Measure.** Top 10 endpoints by spend. Top 10 users by spend. Which models, which features.
2. **Route.** Are simple tasks hitting frontier models? Move them to a cheaper tier.
3. **Cache.** Are you sending the same system prompt 1000 times a day? Add `cache_control`.
4. **Trim output.** Audit `max_tokens`, concision instructions. Look at output token p95.
5. **Trim input.** System prompt bloat? Conversation history growing unbounded?
6. **Batch where possible.** Non-real-time workloads → batch API.
7. **RAG instead of stuff.** Long reference docs in prompt → vector retrieval.
8. **Tool result truncation.** Big tool outputs hurting every subsequent step.
9. **Short-circuit agents.** Add confidence thresholds for early termination.
10. **Re-evaluate model choice.** Re-run evals with current cheaper models. Landscape moved since you first picked.

Each step typically lands 10–50% savings on *some* portion of traffic. Stacked, 5–10× total reduction over a quarter is realistic for a system that hasn't been optimized before.

---

## 14. Anti-patterns that eat tokens

Quick list of common waste:

- **Massive system prompts that grow monotonically.** Review quarterly.
- **No caching.** Still the most common single issue.
- **Reasoning model for non-reasoning tasks.** 10× output cost for zero quality gain.
- **Default `max_tokens: 4096`** when actual outputs are 200.
- **No output length guidance.** Models ramble by default.
- **Uncompressed tool results.** 50 KB of JSON fed raw into the next step.
- **Chatty agents.** "Let me think about this..." preamble × 10 steps.
- **Wrapper SDKs adding chrome.** Some wrappers inject their own verbose system prompts on top.
- **Retries on non-retryable failures.** Burns tokens without success.
- **Hidden `temperature=1`** on tasks that would be deterministic + cacheable at T=0.

If any of these describe your system, you have easy savings available.

---

## 15. Summary (the whole folder, in money)

A prompt becomes tokens (doc 02) which flow through a transformer (doc 01) during prefill and decode (doc 03) limited by a context window (doc 04), accelerated by inference optimizations (doc 05), trained in three stages (doc 06), improved across six axes with each new model (doc 07), and **billed per token** at rates where output is 4–5× input.

Your levers, in order of impact:

1. **Right-size the model.** 10–50× on simple tasks.
2. **Prompt caching.** 50–90% on agentic / repeated-prefix workloads.
3. **Trim output.** 30–60% on verbose defaults.
4. **Trim input.** 20–50% on long histories and bloated system prompts.
5. **Batch async work.** 50%.
6. **Self-host** when you have the scale and ops maturity. 10×+.

Measure before, measure after. Every optimization that looks good in a doc has to be verified in production telemetry. That's the discipline.

---

**End of the LLM Fundamentals folder.**

Reading order:
- [00 — README / Index](./README.md)
- [01 — Transformer Architecture](./01-transformer-architecture.md)
- [02 — Tokenization](./02-tokenization.md)
- [03 — From Tokens to Output](./03-tokens-to-output.md)
- [04 — Context Windows and Position](./04-context-windows.md)
- [05 — Inference Optimizations](./05-inference-optimizations.md)
- [06 — Training Pipeline](./06-training-pipeline.md)
- [07 — Model Generations](./07-model-generations.md)
- [08 — Token Economics](./08-token-economics.md)  ← you are here
