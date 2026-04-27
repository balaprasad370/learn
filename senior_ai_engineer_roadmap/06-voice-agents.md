# Voice Agents

> Real-time voice AI: STT → reasoning → TTS, plus barge-in, turn-taking, telephony, and the latency engineering that makes a 700ms response feel human and a 2s response feel broken.

> Direct relevance: **Giga**, **Sarvam (Voice)**, **Bland**, **Vapi**, **Deepgram**, **Retell**, **PolyAI**, **Observe.AI**, **Haptik**.

---

## 1. The pipeline

Classic stack:
```
Mic / phone → VAD → STT → [Endpoint detect] → LLM → TTS → speaker / phone
                ↑                                          │
                └──────────── Barge-in interrupt ──────────┘
```

Components:
- **VAD (Voice Activity Detection)** — is someone talking? Silero VAD, WebRTC VAD, Picovoice Cobra.
- **STT / ASR** — speech to text. Streaming or batch.
- **Endpoint / turn detection** — has the user finished their thought?
- **LLM** — reasoning, tool calls, response generation.
- **TTS** — text to speech. Streaming preferred.
- **Barge-in** — when user starts talking, stop TTS playback immediately.

Newer architectures collapse parts of this:
- **Speech-to-speech models** (GPT-4o realtime, Gemini Live, Moshi, Sesame CSM-1B) take audio in and emit audio out, removing the STT/TTS hop. Lower latency, less control.
- **Hybrid** — keep STT/TTS for control + observability, use speech-to-speech only on subsets.

---

## 2. Latency — the only metric that matters

Target end-to-end **turn latency** (user finishes speaking → first audio out): **<800ms** for natural conversation. **<500ms** feels human. **>1.2s** feels like IVR.

Budget breakdown (cascaded pipeline):

| Stage | Budget | Notes |
|---|---|---|
| VAD + endpoint detect | 100-300ms | Aggressive endpointing reduces this but increases interruptions |
| STT final transcript | 100-300ms | Streaming STT helps; final = endpoint + tail |
| LLM TTFT | 200-500ms | Cached prompts; small/fast model for routing; speculative |
| TTS TTFB (time to first byte) | 100-300ms | Streaming TTS critical; first chunk only |
| Network + jitter buffer | 50-150ms | WebRTC + edge POP |
| **Total** | **~600-1000ms** | First audible byte after user stops |

Knobs that move the needle:
- **Streaming everything** — STT partials → LLM as they arrive, LLM tokens → TTS as they emit.
- **Speculative LLM** start — kick off an LLM draft on partial STT, cancel/restart on revision.
- **Smaller / closer models** — Haiku-class for routing, Sonnet for hard turns; co-locate model in same region as user.
- **TTS chunking** — emit speech as soon as you have a complete clause; don't wait for full sentence.
- **Prefetch / prewarm** — keep workers hot, model loaded, WebSocket open.
- **Parallel tool calls** during reasoning — overlap with TTS prep.
- **Cache** common openings, brand voice intros, hold messages.

---

## 3. Speech-to-text (STT / ASR)

### Categories
- **Hosted streaming APIs** — Deepgram, AssemblyAI, Speechmatics, Google STT, Azure Speech, AWS Transcribe.
- **Open / self-host** — OpenAI Whisper (large-v3, turbo), Faster-Whisper, NVIDIA NeMo (Parakeet, Canary), distil-whisper, WhisperX (with diarization).
- **Realtime-native** — Deepgram Nova-3, AssemblyAI Universal-Streaming, Speechmatics Ursa.

### Decision factors
- **Streaming vs batch** — voice agents need streaming (partial transcripts within ~200ms).
- **Word error rate (WER)** — domain-specific; benchmark on your audio (accents, jargon, noise).
- **Languages** — Indian languages: Sarvam, AI4Bharat IndicWhisper, Google/Azure for breadth; Whisper for general multilingual.
- **Diarization** — who said what; usually batch only or hosted.
- **Punctuation + casing** — improves LLM downstream quality.
- **Latency** — measure P95 first-partial and final-after-endpoint, not just average.
- **Cost** — $0.0025–$0.015 per minute hosted; self-host pays for GPU + ops.

### Endpointing
The hardest part. Two signals:
- **Silence-based** — N ms of silence → end of turn. Simple, but cuts off slow speakers.
- **Semantic** — combine silence with prosody / model prediction of "is this a complete utterance". LiveKit's "turn detector" model and Deepgram's smart endpointing are examples.

Tunable per use case: support call center can be aggressive (~300ms silence); thoughtful interview style (~700ms). Dynamic endpointing based on detected question vs answer is state of the art.

---

## 4. Text-to-speech (TTS)

### Categories
- **Hosted, low-latency** — ElevenLabs (Flash / Turbo), Cartesia (Sonic), Deepgram Aura, PlayHT, Rime, Resemble.
- **Indian / multilingual** — Sarvam Bulbul, Smallest.ai, Coqui-derivatives.
- **Open** — XTTS-v2, F5-TTS, Kokoro, OpenVoice, Sesame CSM-1B (open weights speech model).
- **Big-tech** — Google (Studio voices, Chirp 3 HD), Azure Neural, AWS Polly Generative.

### Decision factors
- **TTFB (time to first byte)** — the only latency that matters live. Cartesia and ElevenLabs Flash hit ~75-200ms.
- **Streaming chunking** — does the API stream audio as you stream text in?
- **Voice quality** — naturalness, prosody, emotion. Subjective; do A/B with users.
- **Voice cloning** — instant clone (3-30s sample) vs fine-tuned voice. Compliance: get explicit consent + watermark.
- **Languages + accents** — Indian English, Hindi, Tamil, Telugu — Sarvam, Smallest.ai, ElevenLabs multilingual all in play.
- **Cost** — hosted: $0.10–$0.30 per 1k chars typical; self-host: GPU bound.
- **Pronunciation control** — SSML, phoneme overrides, lexicons for brand names / technical terms.

### Streaming TTS pattern
1. LLM streams tokens.
2. Buffer until clause boundary (comma, period, conjunction) or N tokens.
3. Send buffered text to TTS streaming endpoint.
4. Receive audio chunks; pipe to playback.
5. Continue while LLM keeps generating.

This overlap is what hides the LLM generation time inside TTS playback.

---

## 5. Barge-in and turn-taking

When the user starts talking while the agent is speaking:
1. VAD fires within ~50-100ms of voice onset.
2. **Stop TTS playback immediately** (drain audio buffer, kill remaining synth).
3. **Cancel in-flight LLM generation** (the response is now stale).
4. Decide: was this a real interruption or backchannel ("uh-huh", "yeah")? Some pipelines classify; cheap version is "ignore <300ms utterances".
5. Start fresh STT capture.

Engineering implications:
- TTS playback buffer should be small (200-400ms) so cancellation is responsive without underflow.
- LLM and TTS calls need cancellation tokens propagated end-to-end.
- Track interruption rate as a KPI — too high = agent is too verbose / too slow / overconfident.

---

## 6. Real-time transports

| Option | Use when |
|---|---|
| **WebRTC** | Browser / mobile in-app voice. Low latency, NAT traversal, jitter buffer built in. **LiveKit**, **Daily**, **Agora**, **100ms** are the typical stacks. |
| **WebSockets (PCM / Opus frames)** | Custom desktop/mobile clients, simpler than WebRTC but you handle jitter, packet loss, etc. |
| **SIP / RTP** | Telephony. Phone calls go through carriers as SIP; you need a media gateway. |
| **PSTN via Twilio / Telnyx / Plivo / Exotel** | Easiest path to real phone numbers; they bridge SIP↔your media server. |

**LiveKit Agents** and **Pipecat** are the two open-source orchestration frameworks that abstract over transport + STT + LLM + TTS. **Vapi**, **Retell**, **Bland** are hosted equivalents. **Daily Bots** uses Pipecat under the hood.

---

## 7. Telephony specifics

Phone audio is **8kHz 16-bit mono** (PSTN narrowband) or **16kHz wideband** (Opus over SIP / VoIP). STT/TTS quality drops vs studio audio — pick models tuned for telephony or upsample carefully.

Other phone-call concerns:
- **Carriers** — Twilio (global, easy), Telnyx (cheaper SIP), Plivo, Vonage; in India: Exotel, Knowlarity, MyOperator; international SIP termination requires regulatory homework.
- **Numbers** — local DIDs vs toll-free; A2P 10DLC in US; TRAI rules in India for outbound.
- **Spam labeling** — outbound calls from new numbers get flagged; warm up volume; STIR/SHAKEN attestation.
- **Call recording + consent** — varies by jurisdiction (two-party consent in IN/CA/some US states); play disclosure, store recordings encrypted.
- **DTMF** — keypad input; bridge to tools like "press 1 for sales" if needed.
- **Hold music / fillers** — buy a few seconds during long tool calls so silence doesn't feel like a drop.

---

## 8. Speech-to-speech models

GPT-4o Realtime, Gemini Live, Moshi (Kyutai), Sesame CSM-1B, Step-Audio, Qwen2-Audio.

- **Pros:** lower latency (one model hop), better prosody / interruption handling, model "hears" tone (sarcasm, emotion).
- **Cons:** less control (no transcript-level guardrails), harder to log / eval, fewer language options, cost per minute is higher than cascaded today, harder to swap voices.
- **Hybrid** — many production systems use cascaded for the main flow but enable speech-to-speech for specific personas or low-latency moments.

---

## 9. Architecting the agent loop for voice

Voice constrains the agent design:
- **Short responses.** Long answers feel like monologues. Cap to 2-3 sentences then ask a question.
- **Token budget = audio budget.** ~150 words/min spoken; a 200-token response is ~50 seconds of audio. Tune accordingly.
- **No markdown.** Headers, bullets, code blocks all sound terrible. Use plain prose, optionally rendered visually if a screen exists.
- **Tool calls during talking.** Run async during TTS so when the agent finishes a sentence, the tool result is ready for the next sentence.
- **Filler phrases** during long tool calls — "Let me check that" — generated by a tiny model or pre-canned, played over the gap.
- **State machine over freeform agent.** Many production voice agents are essentially a finite state machine (greeting → intent → fulfillment → confirmation → close) with LLM-generated turns inside each state. More predictable than open-ended ReAct.

---

## 10. Eval for voice

In addition to LLM-quality evals (helpfulness, accuracy, safety), voice systems need:
- **Audio-level WER** on your domain.
- **Endpointing precision/recall** — false cuts vs missed cuts.
- **Turn latency P50/P95** — end-to-end measured client-side.
- **Interruption rate** — interruptions per call.
- **Task success rate** — did the call complete the goal (booking, lookup, transfer)?
- **Conversational flow metrics** — average turns per call, abandonment rate, time-to-resolution.
- **CSAT / sentiment** sampled from transcripts.
- **Transcript replay tools** — internal UI to scrub through call audio + transcript + LLM trace + tool calls for debugging.

Voice eval is **labor-intensive** — listening to calls is the only honest signal for naturalness. Sample + human-rate; don't trust LLM-as-judge on prosody.

---

## 11. Failure modes

| Failure | Mitigation |
|---|---|
| **Cuts user off** | Tune endpointing later; lower silence threshold; semantic endpointing |
| **Dead air** | Filler phrases on tool latency >800ms; confirm receipt earlier |
| **Robotic / monotone** | Better TTS voice; SSML prosody; speech-to-speech for hot paths |
| **Hallucinates phone numbers / dates** | Strict tool calls for any structured info; never let LLM invent |
| **Won't stop talking** | Cap response tokens; prompt "respond in <=2 sentences" |
| **Misses accents / code-switching** | Multilingual STT; in India, code-switch (Hinglish) is the norm — pick models trained on it |
| **Drops on bad network** | WebRTC reconnect; jitter buffer tuning; opus FEC; degrade to text fallback |
| **Echo / feedback** | Echo cancellation in WebRTC; never play TTS into the same mic stream |
| **PII leak in logs** | Redact transcripts before storage; encrypt audio at rest; access controls |
| **Latency spikes from cold model** | Keep warm pool; prewarm on call answer event |

---

## 12. Cost mental model

Per minute, cascaded pipeline (rough Western pricing):
- STT: $0.005-$0.010
- LLM: $0.01-$0.05 (varies wildly by model; tool-heavy increases)
- TTS: $0.05-$0.15 (ElevenLabs > Cartesia ~ Deepgram Aura)
- Telephony: $0.01-$0.05 PSTN; ~$0 for in-app WebRTC
- **Total**: ~$0.10-$0.30 per minute typical; $0.05 if you're aggressive with self-host + cheaper TTS.

Speech-to-speech (GPT-4o Realtime): ~$0.06/min input audio + ~$0.24/min output audio at list — much higher than cascaded but fewer moving parts.

Cost levers: cheaper TTS, smaller LLM, cache greetings, end calls promptly, batch transcripts for offline analytics.

---

## 13. Senior interview talk track (Giga / Sarvam Voice)

"Design a voice agent at scale" — structure:
1. **Pipeline** — cascaded vs speech-to-speech; pick cascaded for control + observability, name the trade-off.
2. **Latency budget** — write the per-stage budget; total target <800ms.
3. **Component picks** — STT (Deepgram Nova / Sarvam), LLM (small + smart split), TTS (Cartesia / ElevenLabs Flash), with reasons.
4. **Transport** — LiveKit / Pipecat for in-app; SIP via Twilio for phone; mention codec / sample-rate concerns.
5. **Barge-in** — VAD onset → cancel TTS + LLM; cancellation tokens through stack.
6. **State design** — FSM with LLM-generated turns; tool calls async during TTS; filler phrases over gaps.
7. **Eval** — turn latency P95, interruption rate, task success, sampled human ratings.
8. **Scale + ops** — warm pools, region affinity, jitter buffers, telephony number management, recording compliance.
9. **Failure modes** — name 3 (dead air, cuts off user, hallucinated info) with mitigations.
10. **Cost knobs** — TTS choice dominates; smaller LLM; speech-to-speech only where worth it.

The Giga / Sarvam / Vapi / Retell crowd will press hard on **latency math, barge-in implementation, and telephony specifics**. Have those reflexive.

---

## 14. Self-check
- Can you write the per-stage latency budget without looking it up?
- Do you know why streaming TTS with clause-level chunking is critical?
- Can you describe barge-in implementation including cancellation propagation?
- Can you compare cascaded vs speech-to-speech with concrete trade-offs?
- Do you know the Indian telephony providers and what changes for India outbound?
- Can you list 5 voice-specific failure modes and how you'd detect each in production?

---

**Next:** `07-evaluation-engineering.md` — eval beyond accuracy: trajectory, golden sets, eval harnesses, LLM-as-judge done right.
