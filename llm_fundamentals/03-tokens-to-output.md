# 03 — From Tokens to Output

Goal: trace a prompt end-to-end. Tokens go in, tokens come out. Understand what "prefill," "decode," "logits," "sampling," and "streaming" mean in practice, and why generation has two very different cost phases.

---

## 1. The two phases of generation

When you send a prompt to an LLM, the server does **two distinct things**:

```
                    [Prompt: 2000 tokens]
                            │
                            ▼
              ┌─────────────────────────┐
              │        PREFILL          │   One forward pass over
              │  (compute-bound phase)  │   all 2000 prompt tokens.
              └─────────────┬───────────┘   Takes ~hundreds of ms.
                            │
                            ▼
                    First output token
                            │
                            ▼
              ┌─────────────────────────┐
              │         DECODE          │   One forward pass per
              │  (memory-bound phase)   │   generated token.
              │                         │   Takes ~20–50 ms each.
              │   Repeats until stop    │
              └─────────────────────────┘
```

- **Prefill**: process the whole prompt at once, producing one output token. This is compute-bound on matrix multiplications — the GPU is busy.
- **Decode**: generate one token at a time, autoregressively. Each step is memory-bound — the GPU mostly waits on moving weights through memory.

The metrics you see ("Time To First Token" or TTFT, and "Tokens Per Second" or TPS) measure these two phases separately.

| Metric | What it measures | Typical |
|---|---|---|
| TTFT | Prefill latency (first token) | 200–1500 ms |
| TPS (output) | Decode throughput | 30–200 tok/sec |

For a 500-token response: `TTFT + 500/TPS = ~800 ms + ~5 s = ~6 s` for a typical Claude/GPT-4-tier model.

---

## 2. Embedding: tokens become vectors

Given input token IDs `[791, 8415, 7731, 389, 279, 38427]`:

```python
# embedding_table shape: [vocab_size, hidden_dim] e.g., [100000, 4096]
token_embeddings = embedding_table[token_ids]
# shape: [num_tokens, hidden_dim] = [6, 4096]
```

This is just a table lookup — no multiplication, no parameters computed. Six tokens become six 4096-dim vectors. These are the input to the first transformer layer.

---

## 3. Forward pass: what happens through the stack

For each of the N layers (doc 01 §3), the six token vectors are updated:

```
After layer 1:   [6, 4096]    (same shape — each vector refined)
After layer 2:   [6, 4096]
...
After layer N:   [6, 4096]    (final hidden states)
```

Inside each layer:
1. **Attention** lets each token look at the others (causally — token 4 can see 1–4, not 5–6).
2. **FFN** processes each token independently through the MLP.

All 6 tokens are processed **in parallel** — this is one of the key wins over RNNs. Training does this in parallel across thousands of sequences in a batch; inference does it for your one sequence.

---

## 4. From hidden states to logits

After the final layer, we have `[6, 4096]` hidden states. We want to predict the **next** token — the 7th — so we look only at the last position's hidden state:

```python
last_hidden = hidden_states[-1]   # [4096]
logits = last_hidden @ W_unembed  # [vocab_size] = [100000]
```

`logits[v]` = a real number, the model's unnormalized score for token `v` coming next. Higher = more likely.

Example of what logits might look like:
```
Top 10 logits for next token after "The cat sat on the":
  'mat'     12.3
  'sofa'     9.8
  'floor'    9.1
  'bed'      8.6
  'ground'   8.2
  'chair'    7.9
  'couch'    7.5
  'table'    7.1
  'porch'    6.8
  'roof'     6.3
```

Still not probabilities. We need to softmax.

---

## 5. Softmax → probabilities, then sampling

Softmax converts raw logits into a probability distribution summing to 1:

$$
p(v) = \frac{e^{\text{logit}(v)}}{\sum_{v'} e^{\text{logit}(v')}}
$$

For the logits above, the resulting probabilities might be:

```
'mat'     0.73
'sofa'    0.09
'floor'   0.04
...
```

Now we need to pick *one*. Several strategies:

### 5.1 Greedy (temperature = 0)

Always pick `argmax`. Deterministic, but boring — produces the same output every time, often low-diversity, sometimes repetitive ("The cat sat on the mat. The cat sat on the mat. The cat sat on the mat."). Reliable for tasks with a single correct answer.

### 5.2 Temperature

Scale logits by `1/T` *before* softmax:

$$
p(v) = \frac{e^{\text{logit}(v) / T}}{\sum_{v'} e^{\text{logit}(v') / T}}
$$

| Temperature | Effect |
|---|---|
| T → 0 | Greedy (one winner takes all). |
| T = 1.0 | Softmax as-is. The "natural" distribution the model learned. |
| T > 1 | Flattens distribution — more diverse, more "creative," more nonsense. |

At T=1.5, low-probability tokens get real weight. At T=3, output starts resembling randomness. At T=0.2, it's nearly greedy.

Rule of thumb:
- **Code, math, factual extraction** → T=0 or 0.2.
- **Summaries, structured outputs** → T=0.3–0.5.
- **Creative writing, brainstorming** → T=0.7–1.2.

### 5.3 Top-K

After scaling, consider only the top K tokens; zero out the rest. If K=50, you're sampling from the 50 most likely options. Prevents "long-tail garbage" where a 1-in-a-million token sneaks in.

### 5.4 Top-P (nucleus sampling)

Sort tokens by probability. Keep the smallest set whose cumulative probability ≥ P. Sample from that. If P=0.9, you ignore the long tail past the 90th percentile of probability mass.

Top-P is usually preferred over top-K because the "right number of plausible continuations" varies by context — sometimes 3, sometimes 30. Top-P adapts; top-K is fixed.

### 5.5 Repetition and frequency penalties

OpenAI-style APIs expose:
- `frequency_penalty`: subtract from the logit of tokens that have already appeared, proportional to frequency.
- `presence_penalty`: subtract a flat amount from any token that has appeared at all.

Both combat the "infinite loop" failure mode.

### 5.6 Typical sampling stack

Standard recipe (what you get when you don't touch the knobs):

```
temperature = 0.7
top_p = 0.9
top_k = 40 (or not specified)
frequency_penalty = 0.0
presence_penalty = 0.0
```

This is fine for most generative tasks. For structured output (JSON, code), drop temperature to 0.2 or 0.

---

## 6. Autoregressive decode: one token at a time

Once we've sampled the next token (say, `'mat'`, ID 12345), we append it to the sequence and run the forward pass *again*:

```
Iteration 1 (prefill):
  Input:  [791, 8415, 7731, 389, 279]   # "The cat sat on the"
  Output: next = 12345                   # "mat"

Iteration 2 (decode step 1):
  Input:  [791, 8415, 7731, 389, 279, 12345]
  Output: next = 13                      # "."

Iteration 3 (decode step 2):
  Input:  [791, 8415, 7731, 389, 279, 12345, 13]
  Output: next = 791                     # "The"

...until the model emits an end-of-sequence token or we hit max_tokens.
```

Naively, each decode step is a full forward pass over the entire sequence so far. For a 2000-token prompt generating 500 tokens, that would be 500 forward passes over 2000+500 = 2500 tokens on average = enormously expensive.

**This is where KV caching saves us.** See doc 05 §2.

---

## 7. Stop conditions

Generation stops when:

1. **End-of-sequence token** emitted. Each model has specific stop tokens: `<|endoftext|>`, `<|eot_id|>`, `<|im_end|>`. The API handles stripping these from the output.
2. **Custom stop strings** matched. `stop=["\n\nUser:"]` — the server tokenizes each stop string and checks if the generated token sequence ends with it.
3. **`max_tokens` reached.** The hard cap. Most APIs default to a generous value; for production, set it explicitly so a runaway generation can't burn your budget.
4. **Tool call emitted** (in function-calling mode). The model emits a special tool-call token sequence; the API ends generation, returns the call, expects a tool result in the next turn.

When generation stops, the API returns the output tokens with a `stop_reason`: `end_turn`, `stop_sequence`, `max_tokens`, `tool_use`, etc.

---

## 8. Streaming

Instead of waiting for the full response, you receive each token (or small group of tokens) as it's generated:

```python
# OpenAI SDK
for chunk in client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    stream=True,
):
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
```

```python
# Anthropic SDK
with client.messages.stream(
    model="claude-opus-4-7",
    messages=[...],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 8.1 Why streaming matters

- **Perceived latency.** User sees output at ~word-per-100-ms instead of waiting 6 seconds for a paragraph.
- **Early termination.** Your UI or agent can cancel mid-stream if the output is going wrong, saving tokens.
- **Composable pipelines.** Streaming lets you pipe tokens into downstream logic (sentence-by-sentence summarization, live tool-call detection).

### 8.2 The protocol: Server-Sent Events

Under the hood, streaming is HTTP SSE:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

event: content_block_delta
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"The"}}

event: content_block_delta
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":" cat"}}

event: content_block_delta
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":" sat"}}

event: message_stop
data: {"type":"message_stop"}
```

SDKs parse this for you. When you self-host a model (vLLM, TGI), the server speaks this same protocol.

### 8.3 Token boundaries are not word boundaries

Because tokens are subwords, a single "word" often arrives as multiple chunks. Don't try to parse streaming output by splitting on spaces — buffer until you see punctuation or a newline, or use a proper SSE parser.

---

## 9. Function calling / tool use — how it actually works

"Tool use" is not magic. It's the model emitting a specific token sequence that the API recognizes as "call this tool with these arguments."

Modern models are trained on chat templates that include tool-call markers. When you send:

```json
{
  "model": "claude-opus-4-7",
  "messages": [{"role": "user", "content": "What's the weather in Paris?"}],
  "tools": [{
    "name": "get_weather",
    "description": "Get current weather.",
    "input_schema": {"type": "object", "properties": {"city": {"type": "string"}}}
  }]
}
```

The API stuffs the tool definition into the system prompt (using a model-specific format), then generates. The model may emit:

```
<|tool_call|>{"name": "get_weather", "arguments": {"city": "Paris"}}<|/tool_call|>
```

The API intercepts this, stops generation, and returns:
```json
{
  "stop_reason": "tool_use",
  "content": [{"type": "tool_use", "name": "get_weather", "input": {"city": "Paris"}}]
}
```

You call the tool, send the result back:
```json
{"role": "tool", "tool_use_id": "...", "content": "15°C, partly cloudy"}
```

The model continues from there, now with the tool result in its context.

This is why tool use is "just" more tokens — schemas, calls, and results all consume context. A tool-heavy agent burns through tokens quickly.

---

## 10. Structured outputs

Two approaches to force JSON / schema-compliant output:

### 10.1 JSON mode

API flag that biases the model toward valid JSON. Not a hard guarantee unless combined with schema.

### 10.2 Constrained decoding / schema-guided generation

At each decode step, the sampler is given a mask of "tokens that would violate the JSON schema given what's been emitted so far" — those tokens get logit `-∞` and can never be sampled. This guarantees output is valid JSON matching your schema.

OpenAI calls this "Structured Outputs" (`response_format={"type": "json_schema", "json_schema": {...}}`). Anthropic Claude supports it via tool use (you define a tool with your schema and force the model to call it). Self-hosted options: Outlines, LM Format Enforcer, llama.cpp's grammars.

Constrained decoding is the reliable way to get structured output. It's a small latency cost (computing the mask each step) but eliminates the "model returned almost-JSON" class of bug.

---

## 11. Determinism — or the lack of it

Even with `temperature=0` and a `seed`:

1. **Floating-point non-associativity.** GPU kernels may sum values in different orders depending on batch composition. `(a + b) + c` ≠ `a + (b + c)` in float. Result: identical requests in different batches can produce different outputs.
2. **MoE routing.** In Mixture-of-Experts models (doc 07 §3.6), the choice of which experts run for a token depends on batch-level routing decisions.
3. **Provider backend changes.** Your prompt today and tomorrow may hit slightly different model versions or hardware — providers rarely guarantee bit-identical outputs across time.

For evaluation and reproducibility, this matters. Practical advice:
- Pin `temperature=0` and a `seed`.
- Use the same `model_version` string in requests (where providers support it).
- Expect ~2% output divergence across time even at T=0.

---

## 12. Latency breakdown for a real request

Let's trace a realistic agent call:

```
POST /v1/messages  (2000 tokens prompt, 500 max output tokens)

  0 ms   Request hits API
  5 ms   Auth, rate limit, tenant routing
 10 ms   Queue pick, worker assigned
 25 ms   GPU loaded with weights (cache miss); if cached, ~1 ms
         ┌─── PREFILL ───────────────────────────────────┐
         │  2000 tokens × ~20 TFLOPs/token               │
         │  on 8×H100 at ~10 PFLOPs each = 50 ms         │
         └───────────────────────────────────────────────┘
 75 ms   First output token ready  ← TTFT
         ┌─── DECODE (per token) ────────────────────────┐
         │  Weights streamed through HBM                 │
         │  ~1.8 GB/token on 70B FP8 at 3 TB/s = ~0.6 ms │
         │  In practice ~15 ms/token due to overhead     │
         └───────────────────────────────────────────────┘
75 ms    output[0] → client (streaming starts)
90 ms    output[1]
...
7575 ms  output[499]
7580 ms  end_turn
```

Key takeaways:

1. **First-token latency (~75 ms here, 500+ ms in reality for warm systems)** is dominated by prefill + queue + model load. Caching prefixes (doc 08) can drop this dramatically.
2. **Per-token latency (~15 ms)** is dominated by memory bandwidth, not compute. Reducing parameter count (quantization, MoE active-expert count, smaller model) helps; reducing prompt size does not help per-token latency.
3. **Total latency** is dominated by output length at long outputs. Telling the model "be concise" shaves seconds. "Respond in 1–2 sentences" is a real performance optimization.

---

## 13. Summary

A prompt flows through the transformer in two phases: **prefill** (all prompt tokens at once, compute-bound, produces the first output token) and **decode** (one output token at a time, memory-bound, continues until stop). Each step ends with logits — unnormalized scores over the vocabulary — which are shaped by temperature, top-p, and top-k into a distribution that's sampled. The KV cache (next doc) makes decoding cheap. Tool use and structured outputs are decoding with extra constraints or post-processing, not separate mechanisms. Streaming exposes the decode phase token-by-token, improving perceived latency. Determinism is partial at best.

You now know, at the engineering level, what happens between "POST /messages" and "the model replied." Next, we zoom in on the constraint that defines a model: the **context window** — what it is, why it's bounded, and how models extend it.

---

Next: [04 — Context Windows and Position](./04-context-windows.md).
