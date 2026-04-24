# LLM Fundamentals — A Clear Picture

A self-contained learning track covering transformers, tokens, token usage, inference optimizations, and the evolution story of modern language models. Written for a working engineer who wants mental models that hold up — not marketing copy, not research-paper density.

---

## What you'll be able to answer after reading this

1. What actually happens when a transformer reads your prompt and produces a token? (doc 01, 03)
2. Why is a "word" sometimes 1 token, sometimes 4 tokens, and how does that affect cost? (doc 02)
3. What is the context window, physically — why is 1 M tokens hard, and what tricks make it possible? (doc 04)
4. Why does the first token take ~500 ms but subsequent tokens take ~20 ms each? (doc 03, 05)
5. What is a KV cache, and why does every inference engine talk about it? (doc 05)
6. What's the difference between GPT-3, GPT-4, GPT-4o, and GPT-5 — beyond "it's smarter"? (doc 07)
7. What changed when "reasoning models" (o1, o3, Claude extended thinking) appeared? (doc 07)
8. How does pretraining, SFT, and RLHF actually produce a model that follows instructions? (doc 06)
9. How do you estimate tokens and cost for a new prompt before you run it? (doc 08)
10. What is prompt caching, and why does it cut costs 50–90% on agent workloads? (doc 08)

If you can answer all ten with a whiteboard and no Googling, you're done with this folder.

---

## Reading order

Each doc builds on the prior. Skipping is possible but discouraged.

| # | Title | What you'll learn | Time |
|---|---|---|---|
| 01 | [Transformer Architecture](./01-transformer-architecture.md) | Attention, multi-head attention, the decoder-only stack, positional information. | 45 min |
| 02 | [Tokenization](./02-tokenization.md) | BPE, how tokenizers are built, why the same text has different token counts on different models. | 25 min |
| 03 | [From Tokens to Output](./03-tokens-to-output.md) | Embedding, forward pass, logits, sampling (temperature, top-p, top-k), streaming. | 35 min |
| 04 | [Context Windows and Position](./04-context-windows.md) | Why attention is O(n²), RoPE, ALiBi, sliding window, long-context tricks. | 30 min |
| 05 | [Inference Optimizations](./05-inference-optimizations.md) | KV cache, quantization, batching, speculative decoding, FlashAttention, PagedAttention. | 50 min |
| 06 | [Training Pipeline](./06-training-pipeline.md) | Pretraining, SFT, RLHF, DPO, constitutional AI, distillation. | 40 min |
| 07 | [Model Generations](./07-model-generations.md) | The evolution of GPT, Claude, Gemini, Llama. What each generation added. Reasoning models. | 60 min |
| 08 | [Token Economics](./08-token-economics.md) | Counting, estimating, pricing mental model, prompt caching, practical cost optimization. | 30 min |

Total: ~5 hours end-to-end. Most people will skim some sections and spend longer on others.

---

## Prerequisites

- Comfort with Python and linear algebra at the "matrix multiplication" level. You do not need calculus or a deep-learning background.
- Mental model of "neural network = parameterized function you fit to data." You do not need to derive backprop.
- A curiosity about the difference between *using* an LLM (call the API, get a response) and *reasoning about one* (predict how it'll behave, estimate cost, choose between models).

## Non-goals

- **Not a research review.** We cite concepts, not papers. If you want papers, the [Anthropic](https://www.anthropic.com/research), [OpenAI](https://openai.com/research), and [DeepMind](https://deepmind.google/research/) blogs are where to start.
- **Not a training tutorial.** You will not fine-tune a model by reading this. We cover training *concepts* so you understand model behavior — the "how to train" is a different folder.
- **Not a PyTorch textbook.** Code snippets are for illustration, not production use.
- **Not about one specific model.** We describe the concept and note how Claude, GPT, Gemini, and Llama differ. When a specific model's behavior matters, it's called out.

---

## How to use this folder

1. **Linear read, first time.** The order exists for a reason.
2. **Tinker as you go.** Load a tokenizer (`tiktoken` or Hugging Face `AutoTokenizer`) and tokenize a few strings while reading doc 02. Call the provider APIs with `stream=True` while reading doc 03. Without hands-on, the concepts stay abstract.
3. **Come back for reference.** After the first pass, individual docs are short enough to re-read when a specific question comes up (e.g., "how does prompt caching work again?" → doc 08 §4).

---

## Mental model summary (the whole folder in one paragraph)

A language model is a very large **next-token predictor**. Text is first broken into **tokens** by a tokenizer — not words, not characters, but statistically chosen subword chunks. The model is a **transformer**: a stack of layers where each layer updates every token's vector by *attending* to the others. Attention is quadratic in the number of tokens, which is why long contexts are expensive. During inference, the model processes the prompt once (**prefill**) and then generates one token at a time (**decode**), caching intermediate state (**KV cache**) so each new token only costs the new computation. Training has three phases — **pretraining** (predict next token on the internet), **supervised fine-tuning** (predict next token on curated examples), and **reinforcement from human or AI feedback** (align outputs to human preferences). Each new model generation moves on some combination of: **scale** (more parameters, more data), **context length** (from 4 k → 200 k → 1 M tokens), **multimodality** (images, audio, video), **reasoning** (internal thinking before answering), **tool use** (calling external functions), and **cost efficiency** (cheaper per token at equal quality). Your practical job, as an engineer, is to match the right model to the task and manage token spend — which is ~90% of variable cost.

That paragraph is what you should be able to unpack in increasing detail after reading the docs.

---

Next: [01 — Transformer Architecture](./01-transformer-architecture.md).
