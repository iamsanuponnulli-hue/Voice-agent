# Voice Booking Agent — Design Spec

Status: **v1.0 — decisions locked** · Last updated: 2026-10-19

A voice agent that answers customers, checks availability, and books, reschedules or cancels appointments for a small business. It is built as a **showcase for prospective clients in the UAE**: each prospect gets their own re-skinned agent (their business name, services, staff, hours) that they can talk to in the browser, then watch the booking, transcript and recording appear in an admin dashboard.

---

## 1. Decisions

All open questions were answered on 2026-10-19. This table is the source of truth; the rest of the document follows from it.

| # | Question | Decision |
|---|---|---|
| D1 | Use case | **Appointment booking**: book, reschedule, cancel, answer questions about services, prices, hours, location |
| D2 | Business type | **Configurable per client**: one codebase, each business defined by config (services, staff, hours, FAQs, voice, greeting) |
| D3 | Purpose | **Demo / showcase**, not production. Compliance work deferred until a paying client needs it |
| D4 | Market | **UAE** prospects. Timezone `Asia/Dubai`, currency AED, phone numbers `+971` |
| D5 | Channels | **Web (browser) at launch.** Phone in Phase 3 via a `+971` number from Telnyx once a company is registered. Twilio has no UAE voice numbers |
| D6 | Language | **English at launch**; Arabic (Gulf) in Phase 4. Language is a config field from day one |
| D7 | Model priority | **Best quality where it matters most: speech-to-text and voice.** Use a small, fast language model; booking needs reliable tool use, not deep reasoning (revised 2026-10-19) |
| D8 | Framework | **LiveKit Agents** (Python) |
| D9 | Hosting | **LiveKit Cloud** for media + **one small VM** for agent worker, API, database, dashboard |
| D10 | Appointment storage | **Built-in Postgres database** with our own scheduling logic (no external calendar) |
| D11 | Confirmation | **WhatsApp** message after booking/reschedule/cancel (Twilio WhatsApp; Sandbox until a company is registered) |
| D12 | Recording | **Yes**, audio recorded with a short notice in the greeting; playback in the dashboard |
| D13 | Showcase UI | **Admin dashboard**, including a "Try the agent" page (the web channel) |
| D14 | Company / KYC | **None yet**: everything in Phases 1–2 works without a registered company |

## 2. Scope

### In scope (v1 demo)

- Talk to the agent in the browser, with interruptions handled naturally.
- The agent can book, reschedule and cancel appointments, check availability, and answer FAQs from the business config.
- Real-time availability from our own database, with no double bookings.
- WhatsApp confirmation to the customer's number.
- Admin dashboard: bookings calendar, call list with transcript, recording and per-turn latency, business config editor, "Try the agent" page.
- Multiple businesses (one per prospect), each with a shareable demo link.

### Out of scope (v1)

- Phone calls (Phase 3), Arabic (Phase 4).
- Payments, deposits, insurance checks.
- Outbound calls and reminder calls.
- Compliance work (UAE PDPL, health-data rules), SSO, multi-user roles. A single admin login is enough.
- Syncing with external calendars (Google, Outlook).

## 3. Model and vendor choices

| Stage | Choice | Why | Fallback |
|---|---|---|---|
| Language model | **Claude Haiku 5.5** (`claude-haiku-5-5`), `effort: "low"`, streaming, prompt caching | Booking is a narrow task: understand intent, call a few tools, confirm. Haiku is the fastest current model (lowest time to first token, which matters most in voice) and ~40× cheaper than Opus | Quality gate (§13): if Haiku fails the eval suite, switch the turn model to **Claude Sonnet 5.5** (`claude-sonnet-5-5`, `effort: "low"`); config change only |
| Speech-to-text | **Deepgram Flux** (streaming, English) | Built for voice agents: native end-of-turn events, low latency, keyterm boosting for service and staff names | AssemblyAI Universal-Streaming |
| Text-to-speech | **ElevenLabs**, latest low-latency model (Flash family) | Most natural voices in independent rankings; strong Arabic for Phase 4 | Cartesia Sonic (lowest time-to-first-audio) |
| Voice activity detection | **Silero VAD** (LiveKit plugin) | Small, runs on CPU in the worker | — |
| End-of-turn detection | **LiveKit turn-detector model** + Flux end-of-turn events | Decides whether the caller has finished speaking from the words, not only the silence | Silence-only (700 ms) |
| Noise cancellation | **LiveKit Cloud enhanced noise cancellation** | Removes background noise and other voices before speech-to-text | — |
| Eval: simulated caller | `claude-sonnet-5-5` | Cheap and capable enough to play a caller | — |
| Eval: judge | `claude-opus-5-5`, `effort: "high"` | Grading runs offline, so it can use the strongest model | — |

Vendor rankings for speech-to-text and text-to-speech come mostly from vendor-published benchmarks. Phase 1 includes a bake-off on our own recorded test phrases (UAE-accented English, names, phone numbers) before the choice is final.

**Speech is where accuracy is spent.** Mishearing a name or phone number, or a robotic voice, ruins a demo more than any reasoning gap. That is why speech-to-text and text-to-speech use top-tier vendors, while the language model is the smallest one that passes the eval suite.

**Keep the model's job small.** The code does the hard parts, so a small model is enough:
- The dynamic system message lists today's date and the next 14 days with weekday names ("Tue 21 Oct"). The model never calculates dates.
- `check_availability` does all slot logic.
- The side-effect guard (§6) enforces confirmation.
- The API sends WhatsApp confirmations.

**How the Claude call is made.** Use a custom `llm_node` in LiveKit Agents that calls the **official `anthropic` Python SDK** directly, not the generic LiveKit plugin, so we control effort, caching, refusal handling and tool streaming:

- `client.messages.stream(...)` with `model` taken from config (default `"claude-haiku-5-5"`), `max_tokens=1024`, `output_config={"effort": "low"}`, and adaptive thinking (the default). Effort `low` keeps thinking minimal; raise it only if evals show reasoning errors.
- Refusals: Haiku 5.5 has no server-side fallback. On `stop_reason: "refusal"`, retry the turn once on `claude-sonnet-5-5`. Always check `stop_reason` before speaking.
- Tools: `strict: true` and `eager_input_streaming: true`. Validate each tool input against its schema before running it. `tool_choice` stays `auto`.
- Prompt caching: tools and system prompt are byte-stable per business and version. Put a `cache_control` breakpoint after the system prompt and another on the latest turn. Dynamic facts (today's date, the caller's known details) go in a mid-conversation `system` message, never in the top-level system prompt. Confirm `usage.cache_read_input_tokens > 0` from turn 2.
- History is append-only: the full `response.content` goes back each turn. When the caller interrupts, add a short note to the next user turn saying which sentence they actually heard. Never edit earlier assistant messages, because preserved thinking rejects edited history.
- Text the model writes between tool calls comes back as hidden `thinking` blocks, so the agent does not speak it. Fillers such as "One moment, let me check that" are played by the orchestrator (§5.4).

## 4. Architecture

```
  Prospect's browser                           LiveKit Cloud (nearest edge)
  ┌───────────────────────┐   WebRTC audio    ┌──────────────────────────┐
  │ Dashboard (Next.js)   │◄─────────────────►│  Room per conversation   │
  │  "Try the agent" page │                   │  noise cancellation      │
  └──────────┬────────────┘                   │  Egress → recording      │
             │ HTTPS                          └────────────┬─────────────┘
             ▼                                             │ WebRTC (agent joins room)
  ┌─────────────────────────── One VM (Docker Compose, US-East) ─────────────────────────┐
  │                                                        ▼                             │
  │  ┌──────────────┐   dispatch    ┌───────────────────────────────────────────────┐   │
  │  │ API (FastAPI)│──────────────►│ Agent worker (LiveKit Agents, Python)         │   │
  │  │ - auth       │               │  Silero VAD → Deepgram Flux → turn detector   │   │
  │  │ - room tokens│◄──tool calls──│       → Claude (custom llm_node) → ElevenLabs │   │
  │  │ - scheduling │   (HTTP)      │  barge-in, fillers, event log, latency stamps │   │
  │  │ - config CRUD│               └───────────────────────────────────────────────┘   │
  │  │ - WhatsApp   │                                                                    │
  │  └──────┬───────┘                                                                    │
  │         ▼                                                                            │
  │  ┌──────────────┐   ┌──────────────┐                                                 │
  │  │ Postgres 16  │   │ Caddy (TLS)  │                                                 │
  │  └──────────────┘   └──────────────┘                                                 │
  └──────────────────────────────────────────────────────────────────────────────────────┘
       External: Anthropic API · Deepgram · ElevenLabs · Twilio (WhatsApp) · Cloudflare R2 (recordings)
```

**Why US-East for the VM.** Anthropic, Deepgram and ElevenLabs all serve from US regions. Putting the agent next to them keeps the three per-turn vendor calls fast; the UAE↔US distance is paid once on the audio path (~200–250 ms round trip) instead of on every vendor hop. Phase 3 re-measures an EU region.

**Agent logic calls the API for every tool.** The agent worker holds no business logic. Scheduling rules live in the API, so the dashboard and agent can never disagree, and the logic can be tested without audio.

## 5. Conversation design

### 5.1 Session start

1. The browser asks the API for a room token: `POST /api/demo/{business_slug}/session`.
2. The API creates the `call` row and returns the token plus room name. It dispatches the agent with metadata `{business_id, config_version, call_id, channel: "web"}`.
3. The agent loads the business config, starts recording (LiveKit Egress, audio only) and plays the greeting immediately, before any LLM call (pre-rendered and cached per config version):
   > "Hi, thanks for calling Bright Smile Dental. Just so you know, this call is recorded. How can I help you today?"

### 5.2 Booking flow (happy path)

1. **Intent**: book / reschedule / cancel / question.
2. **Service**: match to a configured service. If ambiguous, offer at most 3 options.
3. **Staff preference** (only if the business has more than one staff member for that service): "Do you have a preference for who you see?"
4. **Time**: ask for a preferred day/time, call `check_availability`, offer **at most 2–3 slots** out loud ("I have Tuesday at 10 a.m. or 2:30 p.m. — which works better?").
5. **Customer details**: full name, then mobile number. Read the number back in groups ("That's oh-five-oh, one-two-three, four-five-six-seven — correct?"). Normalise to E.164 (`+97150…`).
6. **Confirm everything** in one sentence: service, staff, day, date, time. Wait for an explicit yes.
7. `book_appointment` → say the booking is confirmed, and that a WhatsApp confirmation is on its way.
8. "Anything else I can help with?" → `end_call` after goodbye.

**Reschedule / cancel**: look up by mobile number (`find_appointments`). If more than one booking is found, read the soonest first. Confirm which one, then act.

### 5.3 Speaking style (system prompt rules)

- One or two short sentences per turn; one question at a time.
- No lists, markdown, URLs or emojis. Write everything as it should be spoken: "two thirty p.m.", "Tuesday the twenty-first", "one hundred and fifty dirhams".
- Never invent availability, prices or policies; only use tool results and the business config.
- If unsure what was said, ask once to repeat; after two failed attempts on the same item, offer to take a message.
- Stay on topic; politely decline unrelated requests.
- Use British/UAE-friendly English spelling and phrasing. Weekends are Saturday–Sunday unless the business config says otherwise.

### 5.4 Turn-taking and latency tricks

- **End of turn**: Silero VAD + LiveKit turn detector with minimum endpointing delay 300 ms, maximum 1,500 ms. Flux end-of-turn events can shorten the wait.
- **Barge-in**: caller speech lasting ≥ 250 ms or ≥ 2 words stops agent audio at once and cancels the LLM and TTS streams. Short backchannels ("mm-hm", "okay", "yeah") are ignored while the agent speaks.
- **Fillers**: if a tool call has not returned in 600 ms, play a cached filler ("One moment, let me check that.") Never play more than one filler per turn.
- **Preemptive generation**: on (LiveKit `preemptive_generation`). The LLM starts on the interim transcript and the result is discarded if the caller keeps talking.
- **Silence**: after 8 s of silence, "Are you still there?"; after another 8 s, goodbye and `end_call`.
- **Max call length**: 10 minutes, then a polite wrap-up.

## 6. Tools (agent → API)

All tools are JSON-schema with `strict: true`, executed by the agent worker over HTTP against the API (`/internal/tools/*`, authenticated with a shared secret). Every result is short, plain JSON that the model can read directly. Every call is logged with latency.

| Tool | Input | Returns | Notes |
|---|---|---|---|
| `get_business_info` | `topic` (hours, location, parking, policies, faq) | relevant text | Avoids putting rarely-used FAQs in every prompt |
| `list_services` | — | `[{id, name, duration_min, price_aed, staff_ids}]` | Also in system prompt if ≤ 20 services |
| `check_availability` | `service_id`, `date_from`, `date_to` (ISO dates, ≤ 14 days), `staff_id?`, `part_of_day?` (`morning`/`afternoon`/`evening`) | up to 6 slots `[{start, staff_id, staff_name}]`, local time | Server picks well-spread slots |
| `book_appointment` | `service_id`, `staff_id`, `start`, `customer_name`, `customer_phone` (E.164), `notes?` | `{appointment_id, start, staff_name, service_name}` or error `slot_taken` | Re-checks availability inside a transaction |
| `find_appointments` | `customer_phone` | upcoming appointments for that number | |
| `reschedule_appointment` | `appointment_id`, `new_start`, `staff_id?` | updated appointment or `slot_taken` | |
| `cancel_appointment` | `appointment_id`, `reason?` | `{cancelled: true}` | Respects `cancellation_notice_hours` |
| `take_message` | `customer_name`, `customer_phone`, `message` | `{saved: true}` | Stands in for "transfer to a human" in the demo; shown in dashboard |
| `end_call` | `reason` | — | Agent finishes speaking, then disconnects |

**Side-effect guard.** `book_appointment`, `reschedule_appointment` and `cancel_appointment` are only allowed after the orchestrator has seen the model read back the details and the caller say yes. The orchestrator tracks a `confirmed_action` flag in session state, set by a confirmation turn. If the model calls one without confirmation, the tool returns `{"error": "needs_confirmation"}` and the model asks.

WhatsApp confirmations are **not** a tool: the API sends them after any successful book/reschedule/cancel, so the model cannot skip or duplicate them.

## 7. Scheduling engine

### 7.1 Data model (Postgres)

```
business(id, slug, name, timezone='Asia/Dubai', language='en', config jsonb, config_version, created_at)
  config: greeting, voice_id, address, parking, policies, faqs[], cancellation_notice_hours,
          slot_granularity_min=15, buffer_min=0, booking_horizon_days=30, min_notice_hours=2
service(id, business_id, name, aliases[], duration_min, price_aed, active)
staff(id, business_id, name, active)
staff_service(staff_id, service_id)
working_hours(id, staff_id, weekday 0-6, start_time, end_time)        -- multiple rows = breaks
time_off(id, staff_id, starts_at, ends_at, reason)
customer(id, business_id, name, phone_e164, created_at)               -- unique(business_id, phone_e164)
appointment(id, business_id, service_id, staff_id, customer_id,
            during tstzrange, status: booked|cancelled|completed|no_show,
            source: voice|dashboard, call_id, notes, created_at, updated_at)
  EXCLUDE USING gist (staff_id WITH =, during WITH &&) WHERE (status = 'booked')   -- no double booking
call(id, business_id, channel, started_at, ended_at, outcome, recording_url, summary)
call_event(id, call_id, ts, type, payload jsonb)                      -- append-only event log
message(id, business_id, call_id, customer_name, phone, body, created_at)
notification(id, appointment_id, channel='whatsapp', status, provider_id, error, created_at)
```

### 7.2 Availability algorithm

For each eligible staff member and each day in the range:

1. Take that weekday's working-hours windows in the business timezone.
2. Subtract time off and existing `booked` appointments (each widened by `buffer_min`).
3. Generate start times on the `slot_granularity_min` grid where `start + duration` fits in a free window.
4. Drop slots earlier than `now + min_notice_hours` or after the booking horizon.
5. Return up to 6 slots, spread across days and times of day, matching `part_of_day` if given.

Booking runs in a single transaction. The exclusion constraint is the final guard against double booking: on conflict the tool returns `slot_taken` with 2 alternatives.

### 7.3 Seed data

A `seed/` folder with three example businesses (dental clinic, salon, physiotherapy). A new prospect's business can be created from the dashboard or by copying a seed YAML: `make new-business NAME=... TEMPLATE=dental`.

## 8. Admin dashboard

Next.js (TypeScript), served by the same VM; single admin login (email + password hashed with argon2, session cookie).

| Page | Contents |
|---|---|
| **Try the agent** `/try/{slug}` | Big mic button, live captions for both sides, a booking card that appears when a booking is made, optional live latency per turn. Shareable with a prospect via an unguessable link (no login) |
| **Bookings** | Week/day calendar per staff member; create/edit/cancel manually (source = dashboard) |
| **Calls** | List with outcome, duration and summary. Detail view shows the transcript with timestamps, audio player, tool calls, and a per-turn latency breakdown (end-of-turn, STT, LLM time to first token, TTS time to first audio) |
| **Messages** | Messages taken by the agent |
| **Business settings** | Name, greeting, voice picker with preview, services, staff, working hours, time off, FAQs, policies. Saving bumps `config_version` |
| **Businesses** | List and create businesses, one per prospect; copy the demo link |

The call summary is generated after the call by `claude-sonnet-5-5`: one paragraph plus an outcome label.

## 9. Latency

Measured from the caller stopping speaking to the first agent audio heard in a UAE browser.

| Stage | Budget p50 |
|---|---|
| End-of-turn decision | 300 ms |
| STT final (overlaps end-of-turn) | 50 ms |
| Claude time to first sentence (Haiku, cached prompt) | 300 ms |
| ElevenLabs time to first audio | 150 ms |
| UAE ↔ US audio path (both directions) | 230 ms |
| **Total** | **≈ 1.0 s** |

**Targets**: p50 ≤ 1.2 s, p95 ≤ 2.0 s, barge-in stop ≤ 250 ms.

**Latency gate (end of Phase 1).** Run 50 scripted turns. If total p50 is over 1.4 s, try an EU-region VM and Cartesia for text-to-speech. The model is not expected to be the bottleneck with Haiku.

Every turn writes latency stamps into `call_event`, so the dashboard shows the real numbers.

## 10. WhatsApp confirmations

- Twilio WhatsApp API. **Until a company is registered: Twilio WhatsApp Sandbox.** Recipients first send the sandbox join code, which suits demos to prospects on their own phones. After registration: a Meta-verified WhatsApp Business sender with approved templates.
- Template (utility): "Hi {name}, your {service} with {staff} at {business} is confirmed for {day, date} at {time}. To change it, just call us back." Similar templates for rescheduled and cancelled.
- Sent asynchronously by the API after commit; result stored in `notification`. A failure never affects the call.

## 11. Recording, data and privacy (demo level)

- Greeting includes the recording notice (D12). Recording: LiveKit Egress, audio only, OGG to Cloudflare R2 (private bucket). Played back in the dashboard through short-lived signed URLs.
- Secrets live in `.env` on the VM, never in the repo. The dashboard is HTTPS only, through Caddy.
- Retention: recordings and transcripts are deleted after 30 days by a nightly job. Prospects' demo data can be wiped per business.
- Deferred to production: UAE PDPL review, DHA/DOH rules for clinics, data residency in the UAE, consent records, PII redaction.

## 12. Failure handling

| Failure | Behaviour |
|---|---|
| Claude slow (> 2.5 s to first token) or 5xx | Play filler, retry once on `claude-sonnet-5-5`; then "Sorry, I'm having trouble — can I take a message?" |
| Refusal (`stop_reason: "refusal"`) | Retry the turn once on `claude-sonnet-5-5`; if it still refuses, steer back to booking |
| Deepgram / ElevenLabs error | LiveKit `FallbackAdapter` → AssemblyAI / Cartesia |
| Tool / API error | Model receives `is_error` result and apologises; never invents a booking |
| `slot_taken` | Offer the 2 alternatives returned |
| WhatsApp send fails | Logged; dashboard shows "not delivered"; call unaffected |
| Agent worker crash | Browser shows "connection lost — try again"; call marked `failed` |

## 13. Observability and evaluation

- **Logs**: structured JSON (structlog) with `call_id` on every line. **Metrics**: the per-turn stamps in `call_event`, plus a dashboard latency chart. **Errors**: Sentry, free tier.
- **Scenario evals** (`evals/`): 25 scripted scenarios run through the real pipeline, including the audio, using a simulated caller (`claude-sonnet-5-5` + TTS):
  - simple booking; booking with no staff preference; requested time unavailable; caller changes their mind mid-way;
  - interruptions during slot offers; mumbled or corrected phone number; reschedule; cancel inside the notice window;
  - unrelated question; caller asks for a human; silence; background noise.
- **Judge** (`claude-opus-5-5`, effort high) scores each run on:
  1. task completed correctly (checked against the database);
  2. no hallucinated slots or prices;
  3. confirmation before side effects;
  4. concise, spoken style;
  5. latency within target.
- Run on every prompt or config change. Gate: ≥ 90% pass, no hallucination failures.
- **Model quality gate (Phase 2):** run the suite on `claude-haiku-5-5` and `claude-sonnet-5-5`. Keep Haiku unless it misses the gate. Sonnet is the default fallback, and the turn model is a single config value.

## 14. Repository layout

```
voice-agent/
├── docs/DESIGN.md
├── agent/                     # LiveKit Agents worker (Python 3.12, uv)
│   ├── main.py                # entrypoint, AgentSession setup
│   ├── llm_claude.py          # custom llm_node using anthropic SDK
│   ├── prompts/               # system prompt template, fillers
│   ├── tools.py               # tool schemas + HTTP calls to API
│   ├── confirmation.py        # side-effect guard
│   ├── telemetry.py           # latency stamps → call_event
│   └── tests/
├── api/                       # FastAPI + SQLAlchemy 2 + Alembic
│   ├── app/routes/            # demo sessions, tools, dashboard CRUD, auth
│   ├── app/scheduling/        # availability, booking (pure functions, heavily tested)
│   ├── app/notify/whatsapp.py
│   ├── migrations/
│   └── tests/
├── dashboard/                 # Next.js 15 + @livekit/components-react + Tailwind
├── evals/                     # simulated caller, scenarios, judge rubric
├── seed/                      # example businesses (YAML)
├── deploy/                    # docker-compose.yml, Caddyfile, .env.example
└── Makefile                   # dev, test, seed, eval, deploy
```

## 15. Environments and cost

- **Local dev**: `docker compose up` (Postgres, API, dashboard) + `uv run agent/main.py dev` against a LiveKit Cloud dev project.
- **Demo server**: one VM (2 vCPU / 4 GB, Ubuntu 24.04, US-East, e.g. Hetzner Ashburn or DigitalOcean NYC) running Docker Compose; deploy by `make deploy` (rsync + `docker compose up -d`). Domain with HTTPS through Caddy.
- **Estimated running cost** (to be verified against live pricing):
  - Fixed: VM ~$10–25/month; LiveKit Cloud, Sentry and R2 on free tiers at demo volume.
  - Per conversation minute, about $0.044 all-in (ElevenLabs is the largest share, then Deepgram; Claude Haiku 5.5 is ~$0.003). Full per-business daily estimate in [COST.md](COST.md).

## 16. Roadmap and acceptance criteria

**Phase 1 — Talking skeleton (week 1–2)**
- Agent in a LiveKit room with Flux → Claude → ElevenLabs, barge-in, greeting, end of call; plain test page.
- Speech-to-text and text-to-speech bake-off; latency gate (§9).
- ✅ Done when: a 3-minute conversation works with p50 ≤ 1.4 s and barge-in works reliably.

**Phase 2 — Bookable demo (week 3–5)**
- Scheduling engine and tools; database; seed businesses; WhatsApp Sandbox; recording; dashboard (all §8 pages); 25-scenario eval suite; deploy to VM.
- ✅ Done when: a prospect can open `/try/{slug}`, book an appointment by voice, get a WhatsApp confirmation, and the admin sees booking, transcript, recording and latency. Eval pass rate ≥ 90%. p50 ≤ 1.2 s from a UAE network.

**Phase 3 — Phone (after company registration)**
- Telnyx `+971` number → LiveKit SIP inbound trunk; 8 kHz telephony tuning; re-measure EU vs US region; verified WhatsApp Business sender.

**Phase 4 — Arabic**
- Gulf Arabic speech-to-text bake-off (Deepgram, ElevenLabs Scribe, Speechmatics); ElevenLabs Arabic voice; language auto-detect at greeting; bilingual prompts and number reading.

## 17. Risks

| Risk | Mitigation |
|---|---|
| Browser voice (WebRTC) blocked or degraded on some UAE networks | Test on e& and du mobile data and home broadband in Phase 1; LiveKit falls back to TURN over TLS on port 443. If still blocked, prioritise Phase 3 (phone) |
| Haiku 5.5 makes booking mistakes (dates, skipped confirmation) | Dates computed in code, side-effect guard, eval gate; switch to Sonnet 5.5 by config |
| Speech-to-text gets UAE names and accents wrong | Keyterm boosting from staff and service names; always read back name and number; bake-off on accented samples |
| WhatsApp Sandbox needs recipients to opt in first | Explain in the demo script; register a business sender after company registration |
| Twilio cannot provide UAE voice numbers | Phase 3 uses Telnyx (or a local e&/du SIP trunk) |
