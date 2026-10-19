# Voice Agent System — Design

Status: Draft v0.1 · Owner: TBD · Last updated: 2026-10-19

## 1. Goal

A real-time conversational voice agent that people can talk to over a **phone call** or a **browser/mobile app**. It listens, understands, takes actions through tools (look up an order, book a slot, answer from a knowledge base), and talks back naturally — fast enough that it feels like a conversation, and able to be interrupted mid-sentence.

### Requirements

| Area | Target |
|---|---|
| Voice-to-voice latency (user stops talking → agent audio starts) | p50 ≤ 800 ms, p95 ≤ 1.5 s |
| Barge-in (user interrupts → agent stops speaking) | ≤ 200 ms |
| Channels | PSTN/SIP telephony, WebRTC (web + mobile) |
| Concurrency | 1,000 simultaneous calls at launch, horizontally scalable |
| Availability | 99.9% for call acceptance; graceful degradation when a vendor fails |
| Languages | English at launch; pluggable STT/TTS per language |
| Compliance | PII redaction in logs, recording consent, configurable retention |

### Non-goals (v1)

- Training our own speech or language models.
- Multi-party calls (more than one human speaker).
- Outbound dialing campaigns.

## 2. Architecture overview

The core is a **streaming cascade**: speech-to-text → LLM → text-to-speech, with every stage streaming so stages overlap instead of running one after another. Each provider sits behind an interface, so a vendor can be swapped (or a speech-to-speech model tried) without touching the orchestrator.

```
 Caller ──PSTN──► Telephony provider (SIP trunk) ─┐
                                                  │  audio frames (20 ms, 16 kHz PCM / μ-law)
 Browser/App ──WebRTC──► Media edge (SFU) ────────┤
                                                  ▼
                                   ┌──────────────────────────────┐
                                   │       Session Worker         │  one per active call
                                   │  ┌────────┐   ┌───────────┐  │
                         audio in ─┼─►│  VAD + │──►│ Streaming │──┼──► partial / final transcripts
                                   │  │  turn  │   │    STT    │  │
                                   │  │ detect │   └───────────┘  │
                                   │  └───┬────┘         │        │
                                   │      │ barge-in     ▼        │
                                   │      │      ┌──────────────┐ │      ┌──────────────┐
                                   │      │      │ Dialogue     │◄┼─────►│ Tool gateway │──► CRM, calendar,
                                   │      │      │ orchestrator │ │      └──────────────┘    orders, KB (RAG)
                                   │      │      │ (LLM, state) │ │
                                   │      │      └──────┬───────┘ │
                                   │      │   token stream│        │
                                   │      ▼             ▼        │
                        audio out ◄┼── Playout ◄── Streaming TTS   │
                                   │   buffer   (sentence chunks) │
                                   └──────────────┬───────────────┘
                                                  │ events
                         ┌────────────────────────┼─────────────────────────┐
                         ▼                        ▼                         ▼
                   Session store             Event log /              Metrics & traces
                   (Redis)                   transcripts (Postgres,   (OpenTelemetry)
                                             object storage)
```

### Control plane vs. data plane

- **Data plane** — Session Workers. Stateful, latency-critical, hold one live call each (many per process via async I/O). Pinned to a call for its whole lifetime.
- **Control plane** — stateless API services: agent configuration (prompts, voices, tools), call routing, admin dashboard, analytics. Not on the latency path.

## 3. Components

### 3.1 Transport / media ingress

- **Telephony**: a SIP trunk / programmable voice provider streams call audio to us over a bidirectional WebSocket (8 kHz μ-law). We resample to 16 kHz for STT.
- **WebRTC**: a media SFU (e.g. LiveKit-style) terminates WebRTC, giving echo cancellation, jitter buffering, and Opus from the client. The Session Worker joins the room as a participant.
- A **Media Adapter** interface normalises both into a stream of 20 ms PCM frames in and out, so nothing downstream knows which channel the call came from.

### 3.2 Voice activity detection and turn detection

Getting turn-taking right matters more than any other single factor for how natural the agent feels.

- **VAD** (small on-device model, e.g. Silero-class) runs on every frame: speech / not-speech.
- **End-of-turn detection** combines:
  1. Silence duration after speech (base threshold ~500 ms, adaptive).
  2. A **semantic end-of-turn model** over the partial transcript ("I want to book for…" → not done; "Yes, that's right." → done). This lets the silence threshold drop to ~200 ms when the utterance is clearly complete, and stretch to ~1.2 s when it clearly isn't.
- **Speculative start**: when end-of-turn probability crosses a lower threshold, the orchestrator starts the LLM call early on the current transcript. If the user keeps talking, the speculative response is cancelled and discarded before any audio plays.

### 3.3 Barge-in (interruption)

While the agent is speaking:

1. VAD fires on user speech that lasts longer than ~150 ms (filters coughs and "mm-hm" backchannels; a short list of backchannel words can be ignored outright).
2. The Playout buffer is flushed immediately and TTS + LLM streams are cancelled.
3. The orchestrator records **what the user actually heard**: the conversation history keeps the assistant message truncated at the last word that was played (tracked through TTS word timestamps), not the full generated text. Otherwise the agent "remembers" saying things the user never heard.

Telephony echo: the agent's own voice can leak back into the inbound audio. Rely on WebRTC AEC for app calls; for phone calls, gate VAD sensitivity while the agent speaks and use the provider's echo cancellation where available.

### 3.4 Speech-to-text (STT)

- Streaming STT with **partial** (interim) and **final** results; partials drive turn detection and speculative LLM calls.
- Custom vocabulary / keyword boosting per agent (product names, account terms).
- Interface:

```python
class STT(Protocol):
    async def stream(self, frames: AsyncIterator[bytes], lang: str) -> AsyncIterator[TranscriptEvent]: ...
    # TranscriptEvent: text, is_final, confidence, start_ms, end_ms
```

### 3.5 Dialogue orchestrator

The brain of a session. A per-call async state machine:

```
LISTENING ──end of turn──► THINKING ──first audio──► SPEAKING ──playout done──► LISTENING
    ▲                          │                         │
    └──────── barge-in ────────┴──────── barge-in ───────┘
```

Responsibilities:

- Build the LLM request: system prompt (persona, policies, voice style rules), conversation history, relevant retrieved context, available tools.
- **Stream** the LLM output; forward text to TTS sentence by sentence as soon as each clause is complete.
- Run the tool loop: when the model calls a tool, execute it through the Tool Gateway, feed the result back, continue. If a tool will take more than ~700 ms, play a short filler ("Let me check that for you…") so there is no dead air.
- Maintain structured **session state** (caller identity, collected slots, current task) alongside raw history, so long calls can summarise old turns without losing key facts.
- Enforce guardrails (see §7) and decide on **escalation to a human** (explicit request, repeated failure to understand, sensitive topics).

**Model choice.** Latency dominates, so the default turn model is a fast model (e.g. `claude-haiku-5-5`) with streaming and prompt caching on the static system prompt + tool definitions to cut time-to-first-token. Agents that need harder reasoning can be configured to use a larger model (e.g. `claude-sonnet-5-5`), accepting higher latency, or to route only complex turns to it.

**Voice-specific prompting.** Responses are spoken, not read: short sentences, no markdown/lists/URLs, numbers and dates written to be read aloud, one question at a time, confirm critical values ("That's 4-1-5, 5-5-5, 0-1-9-9 — correct?").

### 3.6 Tool gateway

- Agents declare tools as JSON-schema functions; the gateway owns the actual integrations (CRM, calendar, order DB, payments handoff, knowledge-base search).
- Per-tool timeout, retries, and circuit breaker; every call is logged with arguments (PII-redacted) and latency.
- **Side-effecting tools** (book, cancel, refund) require the model to have read back and confirmed the details with the caller in the conversation; the gateway can enforce this with a `requires_confirmation` flag checked against session state.
- **Knowledge base (RAG)**: documents chunked and embedded into a vector index; retrieval exposed as a `search_knowledge_base` tool rather than stuffed into every turn, so we only pay its latency when needed.

### 3.7 Text-to-speech (TTS)

- Streaming TTS that accepts text incrementally and returns audio chunks plus **word-level timestamps** (needed for barge-in truncation).
- Text normaliser before TTS: expand abbreviations, currency, phone numbers, times.
- Voice, speed, and pronunciation lexicon configured per agent.
- Common phrases (greetings, fillers, "one moment") pre-rendered and cached.

### 3.8 Playout buffer

Paces audio frames out in real time, tracks the playback cursor (what the user has heard so far), and supports instant flush on barge-in.

### 3.9 Alternative: speech-to-speech models

Native audio-in/audio-out models can cut latency further and keep prosody, but give less control over turn detection, tool reliability, transcripts for compliance, and per-stage vendor choice. The `Pipeline` interface allows a `SpeechToSpeechPipeline` implementation to be A/B-tested against the cascade per agent; the cascade is the v1 default.

## 4. Latency budget

Voice-to-voice, from the moment the user stops speaking:

| Stage | Budget (p50) | Notes |
|---|---|---|
| End-of-turn decision | 200–300 ms | Semantic model lets us cut silence wait |
| STT final transcript | ~100 ms | Mostly overlapped with turn detection |
| LLM time to first sentence | 250–350 ms | Fast model, prompt caching, speculative start |
| TTS time to first audio | 100–150 ms | Streaming, first clause only |
| Network + jitter buffer | 50–100 ms | Co-locate workers with media edge and vendors |
| **Total** | **~700–900 ms** | |

Rules that keep us inside it: stream everything, start each stage on partial input from the previous one, keep vendors in the same region as Session Workers, keep persistent connections to STT/LLM/TTS warm, and never block the audio loop on logging or analytics (fire events to a queue).

## 5. Data and state

| Data | Store | Lifetime |
|---|---|---|
| Live session state (turn state, slots, history) | In worker memory, checkpointed to Redis every turn | Call duration (+ short TTL for reconnects) |
| Agent configs (prompt, voice, tools, model) | Postgres, versioned; cached in workers | Permanent, every change versioned |
| Call records, transcripts, tool calls, metrics | Postgres (metadata) + object storage (transcripts) | Per retention policy (default 90 days) |
| Recordings | Object storage, encrypted, only with consent | Per retention policy |
| Knowledge-base embeddings | Vector index | Rebuilt on document change |

Every call writes an append-only **event log** (`call.started`, `turn.user_final`, `llm.first_token`, `tool.called`, `tts.first_audio`, `barge_in`, `call.ended`, …). Transcripts, analytics, and latency dashboards are all derived from it.

## 6. Scaling and deployment

- Session Workers run as containers on Kubernetes, autoscaled on **active sessions per pod** (not CPU). Each pod handles ~50–100 concurrent calls since the heavy work happens at vendors; VAD/turn models are small enough for CPU.
- A **call router** assigns new calls to the least-loaded worker; a call stays on its worker for its lifetime.
- **Draining**: on deploy, a pod stops accepting new calls and exits once its calls finish (max ~30 min drain), so deploys never drop live calls.
- Multi-region: route callers to the nearest region; each region has its own workers, Redis, and vendor endpoints.
- If a worker dies mid-call, the router restarts the session on a new worker from the last Redis checkpoint; the agent apologises briefly and continues.

## 7. Safety, privacy, and guardrails

- **Recording consent** announced at call start where legally required; recording disabled otherwise.
- **PII redaction** (card numbers, SSNs, etc.) on transcripts before they are stored or logged. Payment details are never handled by the LLM — hand off to a PCI-compliant DTMF/IVR payment flow.
- **Authentication** before account-specific actions (verify caller via OTP or known details), tracked as a flag in session state that the Tool Gateway checks.
- **Prompt-injection hygiene**: tool results and KB content are treated as data; side-effecting tools are gated by confirmation and auth flags, not by model judgement alone.
- **Human handoff**: warm transfer to a live agent queue with a generated summary of the call so the caller doesn't repeat themselves.
- Hard limits: max call duration, max consecutive tool failures, abuse detection → polite end of call.

## 8. Failure handling

| Failure | Behaviour |
|---|---|
| STT/TTS vendor errors or slow | Circuit breaker fails over to secondary vendor |
| LLM timeout (> 3 s to first token) | Play filler, retry once on fallback model, then offer human transfer |
| Tool failure | Model is told the tool failed and apologises / offers alternative; never invents results |
| Low STT confidence | Ask the caller to repeat or confirm, instead of guessing |
| Repeated misunderstanding (3 turns) | Offer human transfer |
| Worker crash | Resume from Redis checkpoint on another worker |

## 9. Observability and evaluation

**Per-turn metrics**: end-of-turn delay, STT latency, LLM TTFT, TTS TTFA, voice-to-voice latency, barge-in count, tool latency/error rate. One OpenTelemetry trace per call, one span per stage per turn.

**Per-call outcomes**: task completed, transferred to human, abandoned, call duration, caller sentiment.

**Evaluation**:
- **Offline**: a library of scripted conversations replayed through the pipeline using a **simulated caller** (an LLM playing a persona, speaking via TTS so the full audio path is exercised, including accents, background noise and interruptions). An LLM judge scores each call against a rubric (task success, policy adherence, tone, no hallucinated facts). Run on every prompt/config change as a regression gate.
- **Online**: sample real calls for human review; A/B test prompt, model, and voice changes by agent config version.

## 10. Proposed repository layout

```
voice-agent/
├── docs/                    # this design, ADRs
├── services/
│   ├── session-worker/      # data plane: pipeline, orchestrator, media adapters
│   │   ├── media/           # telephony + WebRTC adapters, playout buffer
│   │   ├── turn/            # VAD, end-of-turn model, barge-in
│   │   ├── providers/       # STT, LLM, TTS interfaces + vendor implementations
│   │   ├── orchestrator/    # state machine, prompt building, tool loop
│   │   └── tests/
│   ├── call-router/         # assigns calls to workers
│   ├── tool-gateway/        # integrations, auth/confirmation gating
│   └── control-api/         # agent config CRUD, call records, analytics API
├── web/                     # admin dashboard + web/mobile client SDK
├── evals/                   # simulated callers, scenarios, judge rubrics
└── infra/                   # Kubernetes manifests, Terraform
```

Suggested stack: Python (asyncio) for the session worker — the best ecosystem for audio/ML and vendor SDKs; TypeScript for the dashboard and client SDK.

## 11. Roadmap

1. **Phase 1 — Walking skeleton (2–3 wks)**: WebRTC only, one STT/LLM/TTS vendor each, silence-based turn detection, barge-in, a single hard-coded agent, call logging. Goal: measure real end-to-end latency.
2. **Phase 2 — Useful agent (3–4 wks)**: tool gateway with 2–3 integrations, RAG knowledge base, semantic turn detection, speculative LLM start, agent config in the control API.
3. **Phase 3 — Phone + production (4 wks)**: telephony adapter, human handoff, PII redaction, consent, failover vendors, autoscaling and draining, dashboards.
4. **Phase 4 — Quality loop (ongoing)**: simulated-caller eval suite as CI gate, A/B testing, speech-to-speech pipeline experiment, more languages.

## 12. Open questions

1. Primary use case and first customer — support line, appointment booking, sales qualification? This drives which tools come first.
2. Which channel matters most at launch: phone or app?
3. Build on a voice-agent framework (e.g. LiveKit Agents, Pipecat) versus our own pipeline? Recommendation: use a framework for media transport and the pipeline plumbing in Phase 1, keep our own orchestrator and turn logic behind interfaces.
4. Hosting constraints (cloud, region, data residency) and compliance scope (HIPAA, PCI, GDPR).
5. Budget per call-minute — this decides vendor tiers and how often larger models can be used.
