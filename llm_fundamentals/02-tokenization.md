# 02 — Tokenization

Goal: understand exactly what a "token" is, why the same text has different token counts on different models, and why tokenization decisions matter for cost, latency, and capability.

---

## 1. The core question

A transformer (doc 01) operates on integer token IDs: `[791, 8415, 7731, 389, 279, 38427]`. But your input is text: `"The cat sat on the mat"`.

**Tokenization is the process of turning the string into integers, and back.** Nothing more, nothing less. But the choices inside that process shape cost, vocabulary, multilingual capability, and which tasks the model is good at.

---

## 2. Why not just use characters or words?

Two bad options bracket the design:

### Character-level
Vocab = ~128 (ASCII) or ~150 k (Unicode). Pros: no out-of-vocab. Cons: sequences get *very* long (5× longer than word-level for English). Attention is O(n²) in length, so 5× longer = 25× compute. Fatal at scale.

### Word-level
Vocab = every word in the language. Pros: natural. Cons: explodes — millions of "words" if you count inflections, names, typos, new words, code identifiers, URLs. Everything you've never seen before becomes `<UNK>` and your model can't generate it.

The winning compromise is **subword tokenization**: break rare words into pieces, keep common words whole.

- `"the"` → 1 token
- `"antidisestablishmentarianism"` → `"anti" + "dis" + "establishment" + "arian" + "ism"` → 5 tokens (roughly)
- `"xcorbulator"` → `"x" + "cor" + "bul" + "ator"` → 4 tokens (handles made-up words)

Subword tokenization guarantees coverage: any UTF-8 string can be encoded, because in the worst case, the tokenizer falls back to per-byte tokens. No `<UNK>`, ever.

---

## 3. BPE (Byte Pair Encoding) — the dominant algorithm

Virtually all modern LLMs use a variant of BPE: GPT-2/3/4 (tiktoken), Claude, Llama, Mistral. Gemini uses SentencePiece-Unigram, which is close enough conceptually.

### 3.1 How BPE is *built* (training the tokenizer)

BPE is trained on a corpus *before* the model itself is trained. Steps:

1. Start with a vocabulary of all single characters (or bytes) present in the corpus.
2. Count all adjacent symbol pairs across the corpus.
3. Find the most frequent pair. Merge it into a new symbol. Add that merged symbol to the vocabulary.
4. Repeat step 2–3 until the vocabulary reaches the target size (e.g., 100 000).

Conceptual example on the tiny corpus `"low low lower lowest"`:

```
Start (chars): l, o, w, e, r, s, t

Pair counts: (l,o)=3, (o,w)=3, (w,e)=1, (e,r)=1, (w,e)=1, (e,s)=1, (s,t)=1
Merge (l,o) → "lo"

Re-tokenize: lo w  lo w  lo w e r  lo w e s t
Pair counts now: (lo,w)=3, (w,e)=2, (e,r)=1, (e,s)=1, (s,t)=1
Merge (lo,w) → "low"

Re-tokenize: low  low  low e r  low e s t
... continue ...
```

After training, you have:
- A **vocab** (~50 k–200 k symbols, depending on the model).
- A **merge table** — ordered list of pair-merge rules.

### 3.2 How BPE is *applied* (tokenizing a new string)

Given the merge table, to tokenize new text:

1. Split the text into bytes (or characters, depending on variant).
2. Greedily apply merges in the order they were learned — each merge rule combines two adjacent symbols into one.
3. Stop when no more merges apply. The resulting sequence is the token stream.

Real example with tiktoken (GPT-4 encoding):

```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")

enc.encode("Hello, world!")
# → [9906, 11, 1917, 0]

[enc.decode([t]) for t in [9906, 11, 1917, 0]]
# → ['Hello', ',', ' world', '!']
```

Note the *space is part of the token* — `' world'` not `'world'`. This is common in byte-BPE: leading whitespace binds to the following word. It's why `" the"` (with leading space) and `"the"` (without) are different tokens and `"Hello"` at start-of-string vs `" Hello"` mid-sentence have different IDs.

---

## 4. Byte-level BPE — the "no UNK, ever" trick

Classic BPE starts from characters. **Byte-level BPE** (GPT-2 onward) starts from the 256 possible bytes. This means any sequence of any bytes — including garbage binary, emoji, obscure Unicode — can be represented. No special "unknown" token needed.

Downside: a single emoji like "🎉" is 4 UTF-8 bytes, and if it's not common enough to have merged into a single token during training, it'll cost you 4 tokens. Rare emoji, rare language scripts, and decorative unicode can be surprisingly expensive.

Example:

```python
enc.encode("🎉")       # → [11410, 231, 231, 232] — 4 tokens for one emoji
enc.encode("€")        # → [82445, 120] — 2 tokens
enc.encode("hello")    # → [15339] — 1 token, 5 characters
```

---

## 5. SentencePiece and Unigram — the alternative

**SentencePiece** is a tokenizer library (originally from Google) that supports BPE *and* Unigram. Gemini, T5, XLNet, Llama 1/2 use SentencePiece. Llama 3 switched to tiktoken-style BPE (with 128 k vocab).

**Unigram language model tokenizer** is a probabilistic variant:
- Start with a huge vocab (e.g., 1 M candidate pieces).
- For each candidate, estimate how much removing it would hurt the corpus likelihood.
- Prune the worst N%. Repeat until you hit the target vocab size.

Difference in practice: Unigram tokenizers tend to split whitespace and punctuation slightly differently. At 100 k+ vocab, for English and code, the choice is not very important. For languages like Japanese or Chinese, Unigram sometimes gives cleaner segmentation.

---

## 6. Special tokens

Beyond text pieces, the tokenizer has **special tokens** for structure:

| Special token | Purpose |
|---|---|
| `<|im_start|>`, `<|im_end|>` (OpenAI) | Mark the beginning/end of a role-tagged turn ("chat template"). |
| `<|start_header_id|>`, `<|end_header_id|>` (Llama 3) | Same idea, different spelling. |
| `<|endoftext|>` (GPT-2) | Signal end of a document during training. |
| `[INST]`, `[/INST]` (Llama 2) | Instruction delimiters. |
| `<|tool_call|>`, `<|tool_result|>` (modern function-calling) | Structured tool use markers. |
| `<|FIM_PREFIX|>`, `<|FIM_MIDDLE|>`, `<|FIM_SUFFIX|>` | Fill-in-the-middle for code models. |

These tokens have dedicated IDs in the vocabulary and **cannot appear in user input** — the tokenizer (or the API server) filters them. If a user's prompt contains the literal string `<|im_start|>`, it gets re-tokenized as the *bytes* of those characters, not as the special marker. This is how prompt-injection defenses at the format layer work (doc 10 in the agent runtime folder).

---

## 7. Chat templates

Different models expect different format wrapping around turns. Examples:

**Llama 3 chat template:**
```
<|begin_of_text|>
<|start_header_id|>system<|end_header_id|>
You are a helpful assistant.
<|eot_id|>
<|start_header_id|>user<|end_header_id|>
What is 2+2?
<|eot_id|>
<|start_header_id|>assistant<|end_header_id|>
```

(model is expected to complete from here with the answer, then emit `<|eot_id|>`.)

**OpenAI / tiktoken chat template (roughly):**
```
<|im_start|>system
You are a helpful assistant.
<|im_end|>
<|im_start|>user
What is 2+2?
<|im_end|>
<|im_start|>assistant
```

When you call the chat API, the SDK applies the template on your behalf. When you *fine-tune* or self-host, you need to know the exact template — a misplaced special token means the model has no idea where turns begin.

Hugging Face tokenizers ship `tokenizer.apply_chat_template(messages)` which handles this correctly per model.

---

## 8. Vocabulary sizes across models

| Model family | Vocab | Notes |
|---|---|---|
| GPT-2 | 50 257 | Byte-level BPE. Original tiktoken. |
| GPT-3.5, GPT-4 | 100 277 (cl100k_base) | Expanded from GPT-2. |
| GPT-4o | 200 019 (o200k_base) | Re-tokenized. Better for many languages. |
| Claude 3, 4 | ~200 000 | Not public but similar scale. |
| Llama 2 | 32 000 | SentencePiece, small. English-heavy. |
| Llama 3 | 128 256 | Much larger; better multilingual. |
| Gemini | ~256 000 | SentencePiece Unigram; very large. |

**Trend: vocab is growing.** More vocab → shorter sequences for the same text → cheaper inference (attention is O(n²)), but larger embedding tables (memory). For multilingual and code-heavy workloads, bigger vocab is a clear win.

---

## 9. Why the same text costs different tokens on different models

Same text, four tokenizers:

```
Text: "The quick brown fox jumps over the lazy dog."

GPT-2 (50k):     11 tokens
GPT-4 (100k):    10 tokens
GPT-4o (200k):    9 tokens
Llama 3 (128k):   9 tokens

Text: "आप कैसे हैं" (Hindi: "How are you")

GPT-2:           ~32 tokens (per-byte fallback, brutal)
GPT-4:           ~13 tokens
GPT-4o:           4 tokens
Gemini (256k):    3 tokens
```

Bigger vocab and better non-English coverage = fewer tokens per sentence = cheaper and faster.

**For code:**
```
Text: "def hello_world():\n    print('Hello')"

GPT-2:    15 tokens (spaces and newlines fragment)
GPT-4:    11 tokens
GPT-4o:    9 tokens
Llama 3:   9 tokens
```

Modern tokenizers explicitly include common code patterns (`    ` 4-space indent, `def `, `function `, `);`, etc.) as merge targets. GPT-2 predates this awareness.

---

## 10. Token counting in practice

**For OpenAI/Anthropic-like models:** use `tiktoken` for OpenAI, Anthropic's tokenizer for Claude. Python:

```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
n_tokens = len(enc.encode(my_text))
```

```python
import anthropic
client = anthropic.Anthropic()
count = client.messages.count_tokens(
    model="claude-opus-4-7",
    messages=[{"role": "user", "content": my_text}],
)
# count.input_tokens
```

**Heuristic when you can't run the real tokenizer:**

| Text type | Chars per token (approx) |
|---|---|
| English prose | 4.0 |
| Code (English comments) | 3.5 |
| JSON | 3.0 (punctuation-heavy) |
| Hindi / Arabic / Thai | 1.5–3.0 depending on tokenizer |
| Japanese / Chinese | 1.0–2.0 |
| Base64 / hex | 2.0 (opaque bytes) |

Rule of thumb: **1 token ≈ ¾ of an English word**, or 4 characters. A 500-word essay is ~650 tokens.

---

## 11. Weird edge cases

### 11.1 The "space at the start" problem

```
enc.encode("world")    # → [14957]
enc.encode(" world")   # → [1917]
enc.encode("hello world")   # → [15339, 1917]
                            # = "hello" + " world"
```

If you concatenate "hello" + "world" (no space) vs "hello " + "world" vs "hello" + " world" — different token IDs, different model behavior. Matters when templating prompts.

### 11.2 Glitch tokens

Some tokens exist in the vocabulary because a specific string appeared many times in the BPE training corpus, but rarely during model training (e.g., because that data was filtered later). The model has an *ID* for that token but no learned behavior. Famous example: GPT-3's ` SolidGoldMagikarp` — a Reddit username that got tokenized as a single token but was rare in training. Prompting it caused bizarre outputs.

Modern models are trained more carefully, but glitch tokens still lurk. Don't trust that every token in the vocabulary is a "good" token.

### 11.3 Tokenization mismatches during fine-tuning

If you fine-tune on data where the tokenizer splits `"foo_bar"` as `["foo", "_", "bar"]` but production data has the identifier appear as `" foo_bar"` with a leading space — different tokens, degraded performance. Always tokenize your fine-tuning data with the *exact* tokenizer you'll use at inference.

### 11.4 Numbers

Classic BPE tokenizes numbers character-ish: `"12345"` → `"123" + "45"` or similar, depending on merges. This is one reason LLMs are bad at arithmetic — each digit isn't its own token, and the tokenization of `"12345"` vs `"2345"` bears no systematic relationship. Some modern tokenizers (Llama 3, Gemini) force digit-level tokenization for this reason: `"12345"` → `"1" + "2" + "3" + "4" + "5"`.

### 11.5 Whitespace runs

`"    "` (4 spaces, common Python indent) is usually a single token in code-aware tokenizers but multiple tokens in older ones. Whitespace-heavy Python vs whitespace-light JavaScript → different effective context sizes.

---

## 12. Tokenization affects capability

A few real-world consequences of tokenization choices:

1. **Arithmetic and counting.** Models trained with non-digit-aware tokenizers (GPT-3, early GPT-4) are worse at multi-digit arithmetic than models with digit-level tokenization. The representation doesn't align with how arithmetic works.
2. **Reversal and character manipulation.** "Reverse the string `'abcdef'`" is hard because tokens don't correspond to characters. Same for "how many letters in banana" — the model sees `"banana"` as 1 token, not 6 chars.
3. **Multilingual capability.** A tokenizer trained on an English-heavy corpus fragments Hindi, Arabic, Thai into per-byte tokens. The model has less effective capacity (in token budget) for non-English, and training signal is diluted.
4. **Code identifiers.** A well-tuned code tokenizer merges common patterns (`self.`, `return `, `function(`). Less fragmentation → shorter sequences → better long-context behavior.

Tokenizer choice is not cosmetic. It's a model capability decision.

---

## 13. How tokenization interacts with pricing

Providers bill on tokens (input + output), priced separately:

| Provider (approx 2026) | Input $/M tokens | Output $/M tokens | Ratio |
|---|---|---|---|
| Claude Opus 4.7 | 15 | 75 | 1 : 5 |
| GPT-4o | 2.50 | 10 | 1 : 4 |
| GPT-4o mini | 0.15 | 0.60 | 1 : 4 |
| Gemini 1.5 Pro | 1.25 (short ctx) / 2.50 (long) | 5 / 10 | 1 : 4 |
| Llama 3.1 70B (hosted) | 0.80 | 0.80 | 1 : 1 |

**Output tokens are 4–5× more expensive than input** on most hosted frontier models. This matters: chopping a prompt by 10% saves less than chopping an output by 10%. "Make the model respond shorter" is often the highest-leverage cost lever (doc 08).

---

## 14. Summary

Tokenization turns strings into integers by applying a pre-trained merge table. Byte-pair encoding (BPE) and Unigram are the two common algorithms; both produce ~50 k–256 k sub-word tokens. Bigger, more modern vocabularies handle multilingual and code better. Every token is billed. The same text can cost 10% or 300% more depending on the tokenizer. Tokenization is not a cosmetic step; it shapes cost, capability (arithmetic, multilingual, code), and the shape of prompts.

You now know what a token physically is. Next, we'll follow a stream of tokens all the way through the transformer and out the other side, with sampling strategies that decide what the model *says*.

---

Next: [03 — From Tokens to Output](./03-tokens-to-output.md).
