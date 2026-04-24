# 07 — Model Generations

Goal: a clear picture of what actually changes with each new model upgrade. Not "it's smarter." The specific capability axes each generation pushes on — and for the major families (GPT, Claude, Gemini, Llama, DeepSeek/Qwen), what each generation added.

---

## 1. The six axes of model progress

Every new model release improves on some mix of:

| # | Axis | What "better" looks like |
|---|---|---|
| 1 | **Raw capability** | Higher scores on benchmarks (MMLU, GPQA, HumanEval, SWE-bench, math olympiad). |
| 2 | **Context length** | Bigger window. From 4k → 200k → 1M → 10M. |
| 3 | **Multimodality** | New input/output modalities: images, audio, video, generative modalities. |
| 4 | **Reasoning** | Explicit "thinking" phase with self-correction — o-series, extended thinking. |
| 5 | **Agentic capability** | Tool use, long-horizon task completion, computer use. |
| 6 | **Efficiency** | Lower cost per token, faster per token, smaller models matching bigger ones. |

A release may move one axis heavily (GPT-4 → GPT-4 Turbo was mostly axis 6) or several at once (Gemini 1.5 → 2.0: context, multimodality, reasoning, efficiency).

When reading release notes, map each change to one of these axes. The landscape becomes tractable.

---

## 2. Benchmarks: what the numbers mean

Because "better" gets quantified against specific benchmarks, you need to know which benchmark measures what:

| Benchmark | Measures | Saturation state (2026) |
|---|---|---|
| **MMLU** | Broad knowledge, 57 subjects, multiple choice | Saturated — top models at 90%+. |
| **MMLU-Pro** | Harder MMLU, more distractors | Top ~80%, still discriminating. |
| **GPQA Diamond** | Expert-level science (PhD-ish questions) | Top ~70–85%, discriminating. |
| **HumanEval** | Simple Python coding | Saturated — top 95%+. |
| **MBPP** | Basic programming problems | Saturated. |
| **SWE-bench Verified** | Real GitHub issue resolution | Discriminating — 50–80% for frontier. |
| **AIME / USAMO** | High-school to olympiad math | Reasoning models: 80%+, non-reasoning: 20–60%. |
| **HLE (Humanity's Last Exam)** | Frontier-hard, expert-curated | Discriminating — 20–50% for frontier. |
| **LiveCodeBench** | Contest coding, contamination-resistant | Discriminating. |
| **ARC-AGI / ARC-AGI-2** | Abstract visual reasoning | Non-reasoning near zero; reasoning models breakthrough. |
| **Tau-bench / Agentic benchmarks** | Multi-turn tool use | Discriminating — this is where differentiation is now. |

Saturated benchmarks don't tell you anything useful. When GPT-4 hits 87% on MMLU and Claude Opus 4.7 hits 92%, that 5-point gap is mostly noise. Discriminating benchmarks (GPQA, SWE-bench, AIME, HLE) are what frontier labs actually race on.

---

## 3. Things every new model adds (the standard upgrade menu)

Before going family-by-family, here's the checklist of things that **typically** get better release-to-release. When a lab drops a new model, expect some subset of:

### 3.1 Training data recency
The training cutoff moves forward. "GPT-4o knows about events through October 2023" → new release pushes to March 2024, then late 2024, etc. Affects current-events Q&A noticeably.

### 3.2 Training data scale + quality
More tokens, better curated. Larger code-to-text ratio. More synthetic data. More multilingual.

### 3.3 Context window growth
Architectural shift (RoPE extension, sliding window + global, Mamba hybrids) plus training on long sequences. Often doubled or 5×'d between generations.

### 3.4 New modalities
Image input (vision) was the big 2023 addition. Audio in/out (2024). Video understanding (2024–2025). Native image generation (2025). Each is a separate training and serving stack.

### 3.5 Tool use / function calling
Every generation gets better at structured tool calls, parallel tool calls, multi-turn tool use. Agentic benchmarks are where this shows.

### 3.6 Mixture of Experts (MoE)
Replace the FFN in each transformer layer with a bank of "experts," router picks 2 of N per token. E.g., 8 experts × 22B each = 176B parameters total, but only 2 × 22B = 44B active per token. Same per-token compute, much bigger capacity. GPT-4, Mixtral, DeepSeek-V3, many recent frontier models use MoE.

### 3.7 Post-training improvements
- Better preference data.
- New RL techniques (DPO variants, constitutional iterations).
- Reasoning post-training (when applicable).
- Red-team-driven safety fixes.

### 3.8 Reasoning mode
Extended thinking / "o1-style" reasoning. Either a separate model variant or a toggle on a standard model.

### 3.9 Efficiency and cost
Same quality, lower price. Or same price, better quality. Driven by:
- Distillation to smaller models.
- Better inference stack (FP8, paged attention, speculative decoding).
- Architectural efficiency (GQA, MoE, Mamba hybrids).

### 3.10 Safety and refusal calibration
Iteration on what to refuse, what to allow. Usually quieter in the marketing; loud in evals.

### 3.11 API surface additions
Structured outputs (JSON schema), prompt caching, batch APIs, fine-tuning support, real-time audio APIs, computer-use APIs.

### 3.12 Multilingual coverage
Larger vocab, more balanced training data, explicit post-training in target languages.

Map any release to this list. Most upgrades hit 4–8 of these at once.

---

## 4. OpenAI's GPT family

### 4.1 GPT-2 (2019)
- 1.5B params max, 1024 context.
- Byte-level BPE tokenizer, learned absolute position.
- Released in stages due to "safety concerns" — in retrospect modest.
- Proof that scale + next-token prediction produces something eerily good at completions.

### 4.2 GPT-3 (2020)
- 175B params, 2048 context, same architecture as GPT-2 at 100× scale.
- First to demonstrate **in-context learning** at scale — few-shot examples in the prompt teach the model new tasks without fine-tuning.
- Base model only at first; API-only. No chat capability.

### 4.3 InstructGPT → ChatGPT (2022)
- Same base model; added SFT + RLHF (doc 06 §5).
- **Turned GPT-3 into a usable assistant.** The breakthrough was post-training, not the base.
- ChatGPT (Nov 2022) was essentially InstructGPT plus a chat UI and scale.

### 4.4 GPT-3.5 Turbo
- Cheaper, faster variant.
- 16k context variant followed.
- The default "chat" model before GPT-4 went mainstream.

### 4.5 GPT-4 (2023)
- Multimodal (text + image input, text output).
- Rumored MoE architecture (8 × 220B experts) — OpenAI never confirmed.
- 8k and 32k context variants.
- Much better reasoning and safety than GPT-3.5.
- First model to cross the "reliably useful for professional work" bar.

### 4.6 GPT-4 Turbo (late 2023)
- Same capability, lower cost, faster, 128k context.
- Training cutoff moved forward.
- Pure efficiency release.

### 4.7 GPT-4o (2024)
- **Omni-modal**: native text + image + audio (in and out).
- New tokenizer (o200k_base, 200k vocab) — much better at non-English.
- Faster and cheaper than GPT-4 Turbo.
- Real-time audio API (voice mode).

### 4.8 GPT-4o mini
- Distilled smaller sibling. Matched GPT-3.5-Turbo quality at much lower cost.
- The template for "mini" models across the industry.

### 4.9 o1 (Sept 2024)
- **First reasoning model**. Internal chain-of-thought before output.
- Massive jumps on math and coding benchmarks.
- Slower and more expensive per call (thinking tokens add up).
- Hidden thinking tokens — users see final answer only.

### 4.10 o3, o3-mini, o3-pro (2025)
- Iteration on o1. Better reasoning, more reliable, cheaper.
- `o3-mini` brought reasoning to the "cheap tier" of the API.
- `o3-pro` explicitly tunable "thinking budget."

### 4.11 GPT-4.5 (early 2025)
- The "last big pretrain" per OpenAI's framing. Best non-reasoning model at the time.
- Very expensive; deprecated relatively quickly as reasoning models took over.

### 4.12 GPT-5 (2025)
- Unified model combining strong base capability with on-demand reasoning.
- Improvements across all six axes.
- Multiple sizes (nano, mini, standard, pro) sharing family traits.

Takeaway across the GPT line: the **architecture hasn't changed dramatically** since GPT-2. What's changed: scale (100×+), training data (500×+), post-training (added RLHF, then reasoning RL), multimodality (added images, then audio), efficiency (MoE, smaller distilled siblings), and interface (chat, function calling, structured outputs, real-time audio).

---

## 5. Anthropic's Claude family

### 5.1 Claude 1 (early 2023)
- First public Claude.
- Constitutional AI–trained from the start.
- 100k context (industry-leading at the time; GPT-4 had 8k).
- Emphasis on "harmlessness" and long-document handling.

### 5.2 Claude 2 / 2.1 (mid-late 2023)
- Improved reasoning and coding.
- 200k context in 2.1.
- Still text-only.

### 5.3 Claude 3 (March 2024)
- Three sizes: Haiku, Sonnet, Opus.
- Vision added (image input).
- 200k context on all sizes.
- Opus became a GPT-4-class model.
- Introduced the "family of sizes" pattern Anthropic has kept since.

### 5.4 Claude 3.5 Sonnet (mid 2024, refreshed late 2024)
- Upgraded mid-tier model that, surprisingly, outperformed Claude 3 Opus on many tasks.
- First to introduce **Computer Use API** (Oct 2024) — model controls a screen/mouse/keyboard as a tool.
- Very strong on coding benchmarks.

### 5.5 Claude 3.5 Haiku
- Fast, cheap tier with quality exceeding original Claude 3 Opus on many benchmarks.
- Template for: distilled small models that leapfrog older large models.

### 5.6 Claude 3.7 Sonnet (Feb 2025)
- First Claude with **extended thinking** — explicit reasoning mode toggleable per call.
- Notable leap on complex coding tasks.

### 5.7 Claude 4 (Opus, Sonnet) (2025)
- Big jump in agentic capability — sustained multi-hour tasks with tools.
- Strong SWE-bench performance, production-usable for code-agent workloads.

### 5.8 Claude 4.5 / 4.6 / 4.7 (late 2025 – early 2026)
- Continued iteration on reasoning, agentic tool use, cost.
- Each version tightens benchmarks and adds API features (prompt caching improvements, longer thinking budgets, better structured outputs).

Claude family signature traits:
- **Heavy emphasis on safety and constitutional AI**; noticeable in refusal behavior.
- **Strong at long-document and code tasks**.
- **Fewer hallucinations** relative to GPT family on many tasks (a post-training emphasis).
- **Pioneer on Computer Use and agentic tool use APIs**.

---

## 6. Google's Gemini family

### 6.1 PaLM / PaLM 2 (2022–2023)
- Pre-Gemini era. 540B PaLM, then smaller-but-better PaLM 2.
- Underlying tech for Bard (Google's chat product).

### 6.2 Gemini 1.0 (late 2023)
- Three sizes: Nano (on-device), Pro, Ultra.
- Natively multimodal from the ground up (vs grafted later like GPT-4).
- 32k context initially.

### 6.3 Gemini 1.5 Pro (Feb 2024)
- **1 million token context** — industry-shocking at release.
- MoE architecture.
- Massive lift in long-context capability compared to anyone else.
- Later expanded to **2 million**.

### 6.4 Gemini 1.5 Flash
- Cheap/fast sibling. Distilled.
- Enabled low-cost long-context workflows.

### 6.5 Gemini 2.0 (late 2024 / early 2025)
- "Agentic era" framing.
- Multimodal live API (real-time voice + video).
- Native tool use (search, code execution).
- Flash, Flash-Lite, Pro variants.

### 6.6 Gemini 2.5 / 3.0 (2025–2026)
- Added reasoning mode.
- Further context growth (experimental 10M+ variants).
- Better native multimodality (video understanding, image generation).

Gemini family signature traits:
- **Longest context windows** in the industry.
- **Native multimodality** including video, audio, and generation.
- **Tight Google product integration** (Search grounding, YouTube understanding).
- **Strong on-device story** via Gemini Nano and Gemma distillations.

---

## 7. Meta's Llama family (open weights)

The open-weights leader and the template for the open-source ecosystem.

### 7.1 Llama 1 (Feb 2023)
- 7B, 13B, 33B, 65B.
- Leaked first, then released as "research only." Defined the modern open-weights era.
- SentencePiece tokenizer (32k).

### 7.2 Llama 2 (July 2023)
- Commercially usable license.
- SFT + RLHF chat variants.
- Sliding window + grouped query attention introduced in the 70B.
- Kicked off the open-source fine-tuning explosion (Alpaca → Vicuna → countless variants).

### 7.3 Llama 3 (April 2024)
- 8B, 70B.
- Tiktoken-style BPE (128k vocab) — big multilingual improvement over Llama 2's 32k.
- 15T training tokens (Chinchilla-over-trained for inference efficiency).
- Performance at 70B approaching GPT-4 on many benchmarks.

### 7.4 Llama 3.1 (July 2024)
- **405B** variant, matching frontier on many benchmarks.
- 128k context.
- 8B and 70B also updated.

### 7.5 Llama 3.2 / 3.3
- Vision variants (11B, 90B multimodal).
- Small models for edge (1B, 3B).
- Iteration on 70B without a new 405B.

### 7.6 Llama 4 (2025)
- MoE architecture.
- Very large total parameters with modest active parameters.
- Multimodal natively.

Llama family signature traits:
- **Open weights** — downloadable, fine-tunable, self-hostable.
- **Reference architecture** for the open-source ecosystem. When "a Llama-style model" is said, people know exactly what's meant.
- **Drives the ecosystem**: vLLM, Ollama, llama.cpp, countless fine-tunes, all revolve around Llama.

---

## 8. DeepSeek, Qwen, and the Chinese frontier

Chinese labs pushed hard on open-weights frontier models from 2023 onward.

### 8.1 Qwen (Alibaba, 2023–)
- 1.5, 2, 2.5, 3 series through 2025.
- Qwen 2.5 72B and Qwen-Coder highly competitive with Llama / Claude Haiku class.
- Multilingual strength, especially Chinese.

### 8.2 DeepSeek (2023–)
- DeepSeek-V2, V3: strong MoE open-weights models.
- **DeepSeek-R1 (Jan 2025)**: first major open-weights reasoning model. Recipe published in detail (doc 06 §8.3). Rivaled o1-level reasoning on open weights.
- Showed that frontier reasoning could be trained with far less compute than US labs had implied.

### 8.3 Mixtral, Mistral (French, though often grouped)
- Mixtral 8x7B (Dec 2023): first great open-weights MoE.
- Mistral Small, Medium, Large (hosted).

---

## 9. Reasoning models — the 2024–2026 phase shift

Not a family but a *style* of model. Every major lab now has a reasoning variant:

| Family | Reasoning variant(s) |
|---|---|
| OpenAI | o1 → o3 → future o-series; GPT-5 unified |
| Anthropic | Claude 3.7+ with extended thinking toggle |
| Google | Gemini 2.0 Thinking → 2.5 with reasoning |
| DeepSeek | R1, R1-Zero |
| Alibaba | Qwen QwQ, Qwen 3 reasoning variants |
| Meta | Llama 4 with reasoning (later variant) |

What changed when reasoning landed:

1. **Benchmarks reset.** Math, coding, logic benchmarks jumped 10–40 points overnight when reasoning models arrived. Non-reasoning models look flat-footed on hard problems.
2. **Cost and latency diverged from capability.** A reasoning model may use 10–100× more output tokens on a hard problem. The *right* tradeoff depends on task.
3. **New product patterns.** "Think harder on this" became a UX concept. Budget-tunable reasoning.
4. **Agentic workflows benefited enormously.** Tool-using agents that can reason through tool outputs before the next action are dramatically more reliable.

Reasoning is not a replacement for normal models. For short-turn chat or creative generation, non-reasoning models are faster, cheaper, and often better (less "overthinking" output). For math, code, agents, science: reasoning wins.

---

## 10. Multimodality timeline

| Year | Capability becoming mainstream |
|---|---|
| 2023 | Text-only frontier (GPT-4, Claude 2) |
| 2023 late | Text + image input (GPT-4V, Claude 3) |
| 2024 | Text + image + audio (GPT-4o Realtime, Gemini Live) |
| 2024 late | Video understanding (Gemini 1.5+, GPT-4o updates) |
| 2025 | Native image generation via LLM (GPT-4o image gen, Gemini image) |
| 2025 late | Computer use (screen control as a tool) |
| 2026 | Video generation, real-time agentic video |

Each new modality is typically a **separate encoder** stitched into the model:

- **Image**: vision transformer (ViT) encoder → projected into the LLM's token stream.
- **Audio**: audio encoder (Whisper-like) → same.
- **Video**: sampled frames + temporal encoding.
- **Image generation**: separate decoder (diffusion or autoregressive image model) conditioned on LLM outputs.

The LLM backbone stays; the surrounding encoders/decoders change. True "omnimodal" unified architectures (one model doing everything natively) are the 2025–2026 frontier.

---

## 11. Efficiency improvements — the quiet revolution

Not headline-grabbing but economically the biggest deal. Cost per million tokens has dropped ~10× per year for equivalent quality:

| Year | GPT-4-class input price ($/M) |
|---|---|
| 2023 Q1 | $30 |
| 2023 Q4 (Turbo) | $10 |
| 2024 Q2 (4o) | $5 |
| 2024 Q4 (4o mini for GPT-3.5 class) | $0.15 |
| 2025 | Frontier: $2–3; mid-tier: $0.10–0.30 |
| 2026 | Sub-dollar frontier input common |

Drivers: MoE, distillation, FP8 inference, speculative decoding, continuous batching, paged attention, prompt caching (doc 05 + 08). The *consumer* of this effort is anyone building on top — your cost to serve your users drops passively each year if you're willing to move models.

---

## 12. The "what's added" checklist — applied

Pick any recent release and evaluate it against this list. For example, **Claude 4.7** (hypothetical April 2026 refresh):

| Axis | Change |
|---|---|
| Training cutoff | Moved forward ~6 months |
| Context | 200k standard, 1M beta |
| Multimodality | Image input improved; audio roadmap |
| Reasoning | Extended thinking defaults smarter; budget knobs |
| Agentic | Better tool-call reliability; computer-use improvements |
| Efficiency | Output tokens ~20% cheaper; prompt caching TTL extended |
| Safety | Refusal calibration refinements from red-team rotation |
| API | Structured outputs expanded to nested schemas; new batch API |

No revolutionary change. Eight incremental improvements = the release. This is what every modern model upgrade looks like.

---

## 13. How to reason about which model to use

Given the landscape, a practical selection matrix:

| Task type | Recommendation |
|---|---|
| Chat, Q&A, summarization | Mid-tier (Sonnet, GPT-4o, Gemini Flash). Cheaper than frontier, plenty capable. |
| Code generation, short | Mid-tier. |
| Code: multi-file refactor, agentic | Frontier with reasoning (Claude Opus/Sonnet extended thinking, GPT-5, o-series). |
| Math, logic, science | Reasoning model, period. |
| Creative writing | Non-reasoning mid-tier or frontier. Reasoning models over-think fiction. |
| Translation / multilingual | Models with large, multilingual tokenizers (GPT-4o, Gemini, Llama 3+). |
| Long-document analysis | Gemini 1.5/2 Pro (long context) or Claude with careful prompt engineering. |
| Fast, cheap, bulk classification | Small distilled models (Haiku, 4o mini, Llama 3 8B). |
| On-device / edge | Gemini Nano, Phi, Llama 3.2 1B/3B, MLC deploys. |
| Function-calling-heavy agents | Claude or GPT-5. Benchmarks move but these two lead agentic tool use. |
| Pure research / benchmarking | Frontier reasoning; pin versions for reproducibility. |

Update this matrix quarterly. Rankings shift with releases.

---

## 14. What's next (2026 onward, speculative)

Not predictions — things the field is visibly investing in:

1. **Native agents.** Models designed from the ground up for multi-hour task completion, not just multi-turn chat.
2. **Better long-context reasoning.** Closing the gap between "can retrieve at 1M" and "can reason over 1M."
3. **Multimodal reasoning.** Reasoning chains that include image/video reasoning steps, not just text.
4. **Cheaper frontier capability.** The cost-per-token of frontier models keeps falling; smaller models keep matching older frontiers.
5. **On-device frontier capability.** Running GPT-4-class models on a phone. Requires 5–10× more efficient architectures.
6. **Personalized models.** Per-user or per-organization fine-tunes that stay aligned. Currently clumsy; improving.
7. **Verifiable reasoning.** Reasoning chains that you can check — chains of symbolic operations, proofs, tool traces.

Each is already visible in research or product announcements. None is fully production-ready in early 2026.

---

## 15. Summary

Model upgrades move on six axes: capability, context, multimodality, reasoning, agentic, and efficiency. Each major lab has a family (GPT, Claude, Gemini, Llama, Qwen, DeepSeek) that iterates on a largely stable architecture — decoder-only transformer with modern optimizations — while pushing on those axes.

The architectural story since GPT-2 is surprisingly incremental: GQA, RoPE, SwiGLU, RMSNorm, MoE for capacity. The capability story is mostly **scale + data + post-training + inference efficiency**. The most surprising recent addition — reasoning models — came from a simple RL objective with ground-truth rewards, not a new architecture.

When a new model launches, don't read the marketing. Read the model card. Map changes to the six axes. Re-check your "which model for which task" matrix. The field moves fast but it moves predictably now.

Next and last: **token economics** — how to count, estimate, and reduce your token bill in practice.

---

Next: [08 — Token Economics](./08-token-economics.md).
