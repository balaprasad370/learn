# Fine-Tuning Hands-On

> When to fine-tune vs prompt vs RAG, the algorithms (SFT / LoRA / QLoRA / DPO / KTO / ORPO), data prep, training mechanics, and deployment. Differentiator topic — many engineers skip this; senior candidates can speak to it crisply.

---

## 1. The first question: should you fine-tune at all?

Decision tree before touching any training code:

1. **Have you tried a strong model with a well-engineered prompt + few-shot examples + tools?** If not, do that first. 80% of "we need to fine-tune" turns out to be prompt + retrieval problems.
2. **Have you tried RAG?** If the task is "answer using my documents", fine-tuning won't add knowledge reliably and will hallucinate confidently. RAG injects facts at query time; fine-tuning adjusts behavior.
3. **Do you have ≥500-1000 high-quality labeled examples?** Below that, fine-tuning rarely beats few-shot + retrieval.
4. **Is the task narrow + repetitive?** Classification, extraction, formatting, style transfer, function-calling on niche schemas — yes. Open-ended reasoning — no, use bigger model.
5. **Can you afford the lifecycle cost?** Fine-tuning means re-training when base model upgrades, eval suite, deployment infra, drift monitoring.

### When fine-tuning **does** win
- **Narrow domain classification / extraction** — e.g., medical entity tagging, custom intent taxonomy.
- **Style / format consistency** — voice, structured output adherence, brand tone.
- **Latency / cost** — distill GPT-4-class behavior into a 7B model you can serve cheaply.
- **Function-calling on bespoke schemas** — small model fine-tuned on your tools beats prompted big model for accuracy + cost + latency.
- **Refusal calibration / safety** — domain-specific refusal patterns.
- **Privacy** — keep data + model on-prem / sovereign cloud.

### When fine-tuning **loses**
- Knowledge injection (use RAG).
- Tasks needing reasoning/chain-of-thought beyond base model's reach.
- Small dataset, high variance.
- Fast-changing distribution (data drifts faster than you can re-train).

> Senior framing: "Prompt → few-shot → RAG → tools → fine-tune" — escalate only when each prior tier is exhausted.

---

## 2. The training landscape

| Method | What it does | When to use |
|---|---|---|
| **SFT (supervised fine-tuning)** | Learn from (input, output) pairs | Foundational; first thing you try |
| **LoRA (Low-Rank Adaptation)** | Train small adapters; freeze base | Efficient, swappable, multi-task |
| **QLoRA** | LoRA on a 4-bit quantized base | Largest model on smallest GPU; trade speed |
| **DoRA** | LoRA variant with magnitude decomposition | Sometimes better quality at similar cost |
| **DPO (Direct Preference Optimization)** | Learn from (chosen, rejected) pairs without RL | Replace RLHF for most teams; stable + simple |
| **KTO (Kahneman-Tversky Optimization)** | Like DPO but binary feedback (good / bad) | When you have thumbs-style data, not pairs |
| **ORPO** | Combines SFT + preference in one stage | Reduces pipeline; one-shot alignment |
| **RLHF (PPO + reward model)** | Classic RL on a learned reward | Mostly research / frontier labs; complex |
| **GRPO** | Group-relative PO; popularized by DeepSeek | Reasoning models; verifiable rewards |
| **Distillation** | Student learns to imitate teacher | Cheap+fast model from expensive teacher's outputs |

For 95% of production needs: **SFT + LoRA**, optionally followed by **DPO** for alignment. RLHF has mostly been displaced by DPO/ORPO outside frontier labs.

---

## 3. SFT mechanics

Training format (chat models): a list of messages with a `loss_mask` so loss is computed only on assistant turns, not user / system.

```python
{
  "messages": [
    {"role": "system", "content": "You are a medical entity extractor."},
    {"role": "user", "content": "Patient is 45M, BP 140/90, on metoprolol."},
    {"role": "assistant", "content": "{\"age\": 45, \"sex\": \"M\", \"bp\": \"140/90\", \"medications\": [\"metoprolol\"]}"}
  ]
}
```

Hyperparameters that actually matter:
- **Learning rate**: 1e-5 to 5e-5 for full FT; 1e-4 to 3e-4 for LoRA. Too high → catastrophic forgetting; too low → no learning.
- **Epochs**: 1-3 for SFT on most data. >3 risks overfitting; eval per-epoch and pick best.
- **Batch size**: as large as memory allows; gradient accumulation for effective larger batches.
- **Max sequence length**: pick the smallest that covers your distribution; cost scales with seq².
- **Warmup**: 3-10% of steps; cosine decay typical.
- **Packing**: pack multiple short examples into one sequence to maximize GPU utilization.

---

## 4. LoRA / QLoRA — the workhorse

LoRA freezes the base model and learns small low-rank update matrices `ΔW = A·B` injected into attention (and sometimes MLP) layers. ~0.1-1% of params trained.

### Why LoRA wins for production
- **Memory**: 8B model SFT needs ~80GB; LoRA fits on 24GB.
- **Storage**: adapter is 10-200MB vs full model 16GB+; ship many per use case.
- **Multi-tenant serving**: vLLM `--enable-lora` swaps adapters per request → one base model serves many fine-tunes.
- **Composable**: stack adapters, swap, A/B easily.

### LoRA hparams
- **Rank (r)**: 8-64 typical. Higher rank = more capacity = more risk of overfitting + bigger adapter.
- **Alpha**: scaling factor; convention `alpha = 2 × rank`.
- **Target modules**: `q_proj`, `k_proj`, `v_proj`, `o_proj` minimum; add MLP for harder tasks.
- **Dropout**: 0.05-0.1 helps regularize.

### QLoRA
LoRA on top of a 4-bit quantized (NF4) base. Lets you fine-tune a 70B model on a single 48GB GPU. Trade-off: ~30% slower training; quality usually within 1-2% of full LoRA.

Frameworks: **PEFT** (HuggingFace), **Unsloth** (2× faster, lower memory), **Axolotl** (config-driven, very common), **TRL** (HF; SFT + DPO + reward), **LLaMA-Factory** (broad model + recipe support).

---

## 5. Preference optimization — DPO and friends

After SFT, alignment via preference data. You have pairs:
- *prompt*, *chosen* response, *rejected* response.

DPO directly optimizes a closed-form objective derived from RLHF, no reward model + no PPO needed. Far more stable than PPO.

Variants:
- **DPO** — original; canonical pair-based.
- **IPO** — fixes DPO's overfitting on noisy preferences.
- **KTO** — works on binary good/bad labels (no pairs needed); fits naturally to thumbs-up data.
- **ORPO** — single-stage SFT + preference; skip the SFT stage.
- **SimPO** — reference-free; simpler.

### Data needs
- Thousands of pairs (5k-50k typical).
- Quality matters more than quantity; noisy preferences poison alignment.
- Sources: human raters, synthetic via stronger model judging, model-vs-model contests.

### When to use
- Style alignment (more formal / friendlier / shorter).
- Refusal calibration.
- Format adherence.
- Reducing specific failure modes (over-apologetic, hallucinating, off-topic).

---

## 6. Data prep — the actual hard part

> 80% of fine-tuning success is data quality. Senior signal: you obsess over data, not hyperparameters.

### Sources
- **Production traces** (best): real distribution, real edge cases. Curate, don't dump.
- **Hand-written gold** for missing slices.
- **Synthetic** from a stronger model, with human review of a sample. Cheap to generate, easy to over-rely on.
- **Public datasets** for general capability if you're worried about regression (e.g., a slice of OpenOrca, UltraChat).

### Cleaning
- Dedup near-duplicates (MinHash / embedding clustering). Duplicates hurt generalization.
- Filter by length distribution; drop outliers.
- Schema-validate every example (especially if output is structured).
- Strip PII before training; never train on customer data without consent + legal review.
- Balance by class / intent; oversample rare classes carefully or use class weights.

### Splits
- **Train / dev / test** — strict separation; never leak test into train (especially from prod traces).
- Reserve **regression eval set** that is never used for training across iterations.
- Reserve **adversarial test set** (jailbreaks, edge cases, ambiguity) — tracked separately.

### Sizing
- Classification / extraction: 500-5k can suffice.
- Style / format: 1k-10k.
- Multi-task or open-ended: 10k+.
- Big jumps in quality usually come from *better* data, not 10× more data.

### Labels
- For SFT: examples should be the **gold output**, not "approved-by-rater" outputs (those drift). Have a clear style guide; raters trained on it.
- For DPO: labels are **preferences**, easier to source than gold. Multiple raters per pair if budget allows.

---

## 7. Compute + cost

Rough envelope (open weights):

| Model | Method | GPU | Time / 10k examples | Cloud cost (rough) |
|---|---|---|---|---|
| 7-8B | LoRA | 1× A100 40GB | 2-4h | $5-15 |
| 7-8B | QLoRA | 1× 24GB consumer (3090/4090) | 4-8h | self-host or ~$5 spot |
| 70B | LoRA | 4-8× A100 80GB | 4-8h | $100-300 |
| 70B | QLoRA | 1-2× A100 80GB | 8-16h | $40-100 |
| 70B | Full FT | 16+× H100 | 8-24h | $1000+ |

Hosted fine-tuning (OpenAI, Anthropic, Together, Fireworks, Replicate, Anyscale):
- Per-token training cost ($2-25 per million training tokens depending on provider + model).
- Per-token inference cost premium for fine-tuned models (sometimes 2-3× base).
- Trade-off: managed infra, drift you don't control, lock-in.

### Self-host vs hosted
- **Self-host** when: you have GPU access, want adapter portability, want multi-LoRA serving, privacy / sovereignty.
- **Hosted** when: PoC stage, no infra team, OK with provider lock-in.

---

## 8. Tooling

- **Hugging Face TRL** — SFT, DPO, KTO trainers. Reference implementations.
- **PEFT** — LoRA, QLoRA, prompt-tuning, IA3.
- **Unsloth** — drop-in faster training, lower memory.
- **Axolotl** — YAML-driven; community recipes for popular models.
- **LLaMA-Factory** — broad support; UI; dozens of methods.
- **Torchtune (PyTorch)** — official; growing.
- **NVIDIA NeMo** — enterprise; multi-node; full alignment suite.
- **DeepSpeed / FSDP** — multi-GPU + offload for big models.
- **WandB / MLflow** — experiment tracking. Mandatory.

For preference data labeling: **Argilla**, **Label Studio**, **SuperAnnotate**, **Surge AI** (managed).

---

## 9. Training pipeline (production)

A reproducible fine-tune pipeline:

1. **Data ingest** — pull from sources, dedupe, redact, validate schema.
2. **Curation** — sample, label gaps with humans + synthetic, balance.
3. **Eval set freeze** — hold-out, never train on.
4. **Train** — versioned config (Axolotl YAML / Hydra config), pinned base model, deterministic seed.
5. **Eval** — domain eval + general regression eval (MMLU subset, HellaSwag) to catch capability loss.
6. **Register** — model registry (HF private, MLflow, WandB), tag with dataset + recipe + commit.
7. **Deploy** — to vLLM / SGLang / hosted; LoRA adapter swap if multi-tenant.
8. **Canary + ramp** — same flow as base-model rollout.
9. **Monitor** — quality, latency, cost, drift; trigger re-train on signal.

Re-training cadence: monthly / quarterly typical; event-driven (data drift, base model upgrade) preferred over calendar.

---

## 10. Deployment patterns

### Single-tenant
Fine-tune lives as a deployed model; serve via vLLM or hosted endpoint. Simple.

### Multi-tenant via LoRA
Base model loaded once; per-tenant LoRA adapters loaded on demand and swapped per request (vLLM `--enable-lora`). Massive cost win — one GPU serves dozens of fine-tunes.

### Hybrid (RAG + fine-tune)
Fine-tune for **format / style / refusal**, RAG for **facts**. Common production combo: a small fine-tuned model that knows the brand voice + tool schemas, RAG injects current data per query.

### Distillation cascade
Use GPT-5-class to label data, fine-tune a 7B-class model on it, serve at 1/100 cost. Common for: classification, summarization, structured extraction.

---

## 11. Failure modes

| Failure | Symptom | Cause / fix |
|---|---|---|
| **Catastrophic forgetting** | Fine-tune is great at task, dumb at everything else | Mix in general capability data; lower LR; use LoRA |
| **Overfitting** | Train loss great, eval bad | Reduce epochs; more diverse data; regularization |
| **Hallucinating "in-style"** | Confidently wrong, in your fine-tuned voice | Add RAG; teach model to refuse; eval refusal cases |
| **Drift after base upgrade** | Adapter brittle when base model updates | Pin base version; re-train on new base; abstraction layer |
| **Eval looks good, prod regresses** | Train data isn't matched to prod | Re-sample prod traces; refresh data quarterly |
| **PII leak** | Fine-tune memorizes + regurgitates | Redact pre-training; differential privacy; eval for memorization |
| **Hard slowdowns** | Multi-LoRA serving slows under heavy adapter switching | Pin top-N adapters in memory; batch by tenant |

---

## 12. When fine-tuning **changes the conversation** at interview

If a senior interviewer asks "would you fine-tune for this?", the right answer is rarely "yes immediately". Show you know the cost / value:

> "Before fine-tuning I'd want to confirm prompt + few-shot + tools + RAG can't get us there. If we still need it, I'd plan: data curation pipeline, eval freeze, LoRA SFT on a 7B base, optional DPO if we have preference data. Multi-tenant via vLLM adapter swap. Re-train trigger on drift or base upgrade. Cost vs continuing to call an API — break-even calc on volume."

That single paragraph signals senior more than reciting LoRA hparams.

---

## 13. RLHF / GRPO and the reasoning-model wave

DeepSeek-R1 and successors popularized **GRPO** (group-relative PO with verifiable rewards) for training reasoning behavior. You probably won't run this in product, but you should know:
- It works because reasoning has **verifiable rewards** (math right/wrong, code passes/fails) — no human labels needed.
- For domains with verifiable rewards (your codegen pipeline, your SQL accuracy, your function-calling success), GRPO-style training is increasingly practical.
- For non-verifiable domains, DPO remains the default.

If you can articulate this trend, you're ahead of most candidates.

---

## 14. Senior interview talk track

"Walk me through fine-tuning a model for X."
1. **Justify** — name what RAG / prompt / tools couldn't solve.
2. **Method** — SFT (LoRA / QLoRA), optional DPO; size of base model; rationale.
3. **Data** — sources, sizing, curation, dedupe, redaction, splits, eval freeze.
4. **Training** — hparams (LR, rank, epochs, packing); experiment tracking.
5. **Eval** — domain + capability regression + adversarial; gates before deploy.
6. **Deploy** — vLLM with adapter swap; canary + ramp.
7. **Lifecycle** — re-train trigger; base-model upgrade plan; drift monitoring.
8. **Cost / break-even** — vs continuing API calls; shipping at $X volume justifies $Y training spend.

If they ask "fine-tune vs RAG", show you understand: RAG injects knowledge at query time, fine-tuning shapes behavior. Hybrid is common.

---

## 15. Self-check
- Can you explain the prompt → RAG → fine-tune escalation and when to stop at each?
- Can you list 5 LoRA hparams and their effects?
- Do you know what DPO replaces from classic RLHF and why it's preferred?
- Can you sketch a fine-tune pipeline (data → train → eval → deploy → monitor)?
- Can you describe multi-LoRA serving and why it's a cost win?
- Can you back-of-envelope the break-even between hosted API and self-hosted fine-tune?

---

**Next:** `12-interview-prep.md` — system design framework, behavioral STAR, comp negotiation playbook for the ₹50 LPA - ₹1 Cr band.
