# Implementation Plan — Voice Booking Agent MVP

Last updated: 2026-10-19 · Builds the MVP specified in [DESIGN.md](DESIGN.md) (zero-cost profile, §1.1)

**Goal:** a link you can send a UAE prospect. They talk to an agent branded as their business, book an appointment by voice, receive a WhatsApp confirmation, and see the booking, transcript, recording and latency in a dashboard. It runs at $0/month.

**Estimate:** ~22 working days for one developer, in 9 milestones. Each milestone ends in something you can run and demo; nothing is "done" until its check passes.

```
M0 Setup ─► M1 Talking skeleton ─► M3 Tools in the agent ─► M5 Recording + WhatsApp ─► M7 Deploy ─► M8 Demo polish
              │                      ▲                         ▲                          ▲
              └► M1b UAE/Arm checks   M2 Scheduling core ──────┘                          │
                                      M4 Dashboard ───────────────────────────────────────┤
                                      M6 Evals ───────────────────────────────────────────┘
```

M2 can start alongside M1, and M4 and M6 alongside M3–M5.

---

## Before coding: accounts (you, ~1 hour)

| Account | What to do | Gives us |
|---|---|---|
| LiveKit Cloud | Sign up, create project `voice-agent-dev` | `LIVEKIT_URL`, `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET` |
| Deepgram | Sign up (no card) and confirm the $200 credit shows | `DEEPGRAM_API_KEY` |
| Anthropic Console | Add $5 credit and create a key | `ANTHROPIC_API_KEY` |
| Oracle Cloud | Sign up for Always Free, home region **US East (Ashburn)**; a card is needed for verification but is not charged | VM in M7 |
| Meta for Developers | Create app → add WhatsApp → note the test number; add your own phone as a recipient | `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID` |
| DuckDNS | Pick a subdomain, e.g. `yourname-voice.duckdns.org` | Domain for M7 |
| Sentry | Create two projects (python, nextjs) | `SENTRY_DSN_*` |

The home region cannot be changed later, so choose Ashburn. Keep all keys in a password manager; they go into `.env` only, never into git.

---

## M0 — Repository setup (1 day)

**Tasks**
1. Create the monorepo layout from DESIGN §14: `agent/`, `api/`, `dashboard/`, `evals/`, `seed/`, `deploy/`.
2. Python 3.12 with **uv** workspaces for `agent` and `api`; **ruff** (lint + format), **pyright** (basic), **pytest**.
3. `dashboard/`: Next.js 15 + TypeScript + Tailwind, **pnpm**, ESLint.
4. `deploy/docker-compose.dev.yml` runs Postgres 16 only for local development.
5. `.env.example` with every key from the accounts table plus `DATABASE_URL`, `INTERNAL_TOOL_SECRET`, `ADMIN_EMAIL`, `ADMIN_PASSWORD_HASH`, `PUBLIC_BASE_URL`.
6. `Makefile`: `dev`, `test`, `lint`, `seed`, `eval`, `deploy`.
7. GitHub Actions CI: lint + tests for Python and dashboard on every push.
8. `README.md`: how to run locally in 5 commands.

**Done when:** `make lint test` passes locally and in CI on an empty skeleton.

---

## M1 — Talking skeleton (4 days)

An agent you can talk to, with no booking yet. Test it in the browser with the free **LiveKit Agents Playground**, so no frontend is needed.

**Tasks**
1. `agent/main.py`: LiveKit Agents worker and `AgentSession` with:
   - Silero VAD, LiveKit turn detector;
   - Deepgram **Flux** STT and Deepgram **Aura-2** TTS (pick a natural English voice);
   - `preemptive_generation=True`.
2. `agent/llm_claude.py`: a custom `llm_node` using the official `anthropic` SDK:
   - `claude-haiku-5-5`, `effort: "low"`, streaming, `max_tokens=1024`;
   - `cache_control` after the system prompt;
   - refusal → retry once on `claude-sonnet-5-5`;
   - streams sentences to TTS as they complete.
3. `agent/prompts/system.md`: a persona and speaking-style rules (DESIGN §5.3), using a hard-coded demo business.
4. Greeting with the recording notice, played immediately at join.
5. Barge-in settings from DESIGN §5.4: minimum interruption 250 ms or 2 words, backchannels ignored. Interruptions are recorded in history as "the caller heard up to …".
6. `agent/telemetry.py`: per-turn stamps (end-of-turn, STT final, LLM first token, TTS first audio). Logged as JSON for now, moved to the database in M3.
7. Silence handling (8 s → "Are you still there?"; 16 s → goodbye) and a 10-minute cap.
8. Provider settings in config (`STT_PROVIDER`, `TTS_PROVIDER`, `LLM_MODEL`), so ElevenLabs/Sonnet can be switched on later without code changes.

**Done when:**
- a 3-minute conversation in the Playground feels natural;
- interrupting the agent stops it within ~250 ms;
- logs show p50 voice-to-voice ≤ 1.4 s;
- `usage.cache_read_input_tokens > 0` from turn 2.

### M1b — Risk checks (½ day, as soon as M1 talks)

1. **UAE networks:** open the Playground on e& mobile data, du mobile data and a home broadband line in the UAE. Record whether audio connects and the latency. If blocked, check that LiveKit's TURN/TLS on port 443 fallback works. If it is still blocked, stop and decide: escalate phone (Telnyx) or demo only on known-good Wi-Fi.
2. **Arm:** on a free Oracle A1 VM (or any `aarch64` Docker image), run `uv sync` for `agent/` and start the worker. Confirm the LiveKit, Silero and turn-detector (onnxruntime) wheels install. If not, fall back to the laptop host for demos and note it.

---

## M2 — Scheduling core (4 days; can run in parallel with M1)

Pure business logic, no audio. This is where correctness lives, so it gets the most tests.

**Tasks**
1. SQLAlchemy 2 models + **Alembic** migration for the DESIGN §7.1 schema, including the `btree_gist` extension and the **exclusion constraint** on `appointment`.
2. `api/app/scheduling/availability.py`: a pure function `(working_hours, time_off, appointments, service, rules, now, range, part_of_day) → slots`, implementing DESIGN §7.2 (granularity, buffers, minimum notice, horizon, spread across days, max 6).
3. `api/app/scheduling/booking.py`: `book`, `reschedule`, `cancel`, `find_by_phone`. Each runs in one transaction. A `slot_taken` conflict returns 2 alternatives; cancellation respects the notice window.
4. Phone normalisation to E.164 for UAE numbers (`050…`, `+971 50…`, `00971…`) using the `phonenumbers` library.
5. `seed/*.yaml` with 3 businesses (dental clinic, salon, physiotherapy): realistic UAE services, AED prices, Sat–Sun weekends, staff, hours with breaks. `make seed` loads them.
6. Tests:
   - unit tests for availability edge cases: overlapping breaks, an appointment touching a slot boundary, end of day, time off spanning days, the minimum-notice cut-off, `Asia/Dubai` timezone;
   - a **property test** (Hypothesis): random bookings never overlap;
   - a concurrency test: two simultaneous bookings for one slot give exactly one success.

**Done when:** `pytest api/` passes with the concurrency and property tests included, and `make seed` creates 3 bookable businesses.

---

## M3 — Tools in the agent (3 days)

**Tasks**
1. FastAPI app: `/internal/tools/*` endpoints for the 9 tools in DESIGN §6, authenticated with `INTERNAL_TOOL_SECRET`. Each returns short JSON.
2. `agent/tools.py`: JSON schemas with `strict: true` and `eager_input_streaming: true`; validate inputs before calling; run parallel tool calls concurrently and return all results in one message.
3. Dynamic system message per call: today's date plus the next 14 dates with weekday names, the business name, services (if ≤ 20), hours and policies. The static prompt stays byte-stable for caching.
4. `agent/confirmation.py`: the side-effect guard. `book`, `reschedule` and `cancel` return `needs_confirmation` unless the caller confirmed after a read-back.
5. Fillers: a cached "One moment, let me check that" plays if a tool takes > 600 ms, at most once per turn.
6. Call lifecycle:
   - `POST /api/demo/{slug}/session` creates the `call` row and room token and dispatches the agent with metadata;
   - the agent loads the business config by id;
   - telemetry writes `call_event` rows;
   - `end_call` hangs up after the goodbye.
7. Keyterm boosting: send service and staff names to Deepgram Flux per business.

**Done when:** in the Playground you can, by voice, book, then reschedule, then cancel an appointment for the seeded dental clinic. The database matches each step, and asking for a taken slot gets alternatives. Saying "yes" before the read-back never books.

---

## M4 — Dashboard (5 days; can start after M2)

**Tasks**
1. Auth: single admin login (argon2 hash in env, httpOnly session cookie). Every route except `/try/*` is protected.
2. **Try the agent** `/try/{slug}?k={demo_key}`:
   - business-branded page with a big mic button (`@livekit/components-react`);
   - live captions for both sides from LiveKit transcription streams;
   - a booking card that appears when a booking event arrives;
   - a "Connecting…" state and clear mic-permission errors.

   Unguessable `demo_key` per business.
3. **Bookings**: week/day calendar per staff member (e.g. FullCalendar); create, edit and cancel manually through the same API functions as the agent.
4. **Calls**: list (date, duration, outcome, summary) and detail (timestamped transcript, audio player, tool calls, per-turn latency bars).
5. **Messages**: list of `take_message` entries.
6. **Business settings**: name, greeting, voice picker with preview, services, staff, hours, time off, FAQs, policies. Saving bumps `config_version`.
7. **Businesses**: list, "New from template" (dental/salon/physio), copy demo link.
8. Responsive enough to demo on a phone.

**Done when:** a new business made from a template in under 10 minutes is immediately bookable at its `/try` link, and the booking appears in the calendar without a refresh (poll every 5 s).

---

## M5 — Recording, summaries, WhatsApp (2 days)

**Tasks**
1. Recording: the agent worker mixes caller + agent audio to OGG at `/data/recordings/{call_id}.ogg`. The API serves it through short-lived signed URLs.
2. Call summary: after a call ends, a background job calls `claude-sonnet-5-5` with the transcript and stores one paragraph plus an outcome label.
3. WhatsApp: `api/app/notify/whatsapp.py` using the **Meta Cloud API test number**:
   - sent after a book/reschedule/cancel commits;
   - free-form text if the 24-hour window is open, otherwise the `hello_world` template;
   - result stored in `notification`;
   - failures never affect the call.

   Create a permanent System User token, because test tokens expire after 24 hours.
4. Dashboard shows a "WhatsApp sent / preview" chip on each booking.
5. Nightly job: delete recordings and transcripts older than 30 days.

**Done when:** booking by voice delivers a WhatsApp to your registered phone within ~10 s, and the call page plays the recording and shows the summary.

---

## M6 — Evals (3 days; can start after M3)

**Tasks**
1. **Text-mode harness** (cheap, fast): drive the same agent logic with text turns, skipping STT and TTS, against a fresh seeded database. A `claude-sonnet-5-5` "caller" plays each scenario's persona.
2. **25 scenarios** in `evals/scenarios/*.yaml` (DESIGN §13), each with a persona, goal and expected database end-state.
3. **Judge** (`claude-opus-5-5`, effort high) scores the rubric. The database end-state is checked in code; the judge never decides it.
4. **Audio subset**: 5 scenarios run through real STT/TTS, with the caller voice from Aura-2 and noise mixed in, to catch hearing problems.
5. `make eval` prints a pass table and saves results to `evals/results/`.
6. Model gate: run once on `claude-haiku-5-5`, and once on `claude-sonnet-5-5` for comparison. Record the decision in DESIGN §1.

**Done when:** pass rate ≥ 90% on Haiku with zero hallucinated slots/prices, or a documented switch to Sonnet. A full run costs ≤ $3.

---

## M7 — Deploy to Oracle Always Free (2 days)

**Tasks**
1. Create an Ampere A1 VM (2 OCPU / 12 GB, Ubuntu 24.04 Arm) in Ashburn. If it is "out of capacity", retry over a few hours; meanwhile use the laptop + Cloudflare Tunnel fallback.
2. Open ports 80/443 in the security list **and** the Ubuntu firewall (`iptables` on Oracle images).
3. `deploy/docker-compose.yml` with `caddy`, `api`, `agent`, `dashboard`, `postgres`, all `restart: unless-stopped`. Arm64 images are built in GitHub Actions (buildx), or on the VM.
4. DuckDNS updater + Caddy automatic HTTPS for `yourname-voice.duckdns.org`.
5. Nightly `pg_dump` to `/data/backups` (keep 7), and a weekly copy off the VM (e.g. to your laptop via `make backup-pull`).
6. Sentry DSNs set; a `/healthz` endpoint; a free uptime ping (e.g. UptimeRobot) to catch outages.
7. `make deploy`: build → push images → `ssh vm 'docker compose pull && docker compose up -d'` → run migrations.

**Done when:** the `/try` link works from a UAE phone on mobile data, a booking survives a VM reboot, and `make deploy` is the only step needed for a new release.

---

## M8 — Demo polish (2 days)

**Tasks**
1. Voice selection: A/B 3 Aura-2 voices with 3 listeners; pick the default.
2. Tighten prompts using the M6 failures; re-run evals.
3. **Demo script** (`docs/DEMO_SCRIPT.md`): a 5-minute walkthrough to show prospects — book, interrupt, reschedule, the dashboard reveal.
4. **Prospect onboarding checklist:** create the business from a template, enter their services and hours, register their phone as a WhatsApp test recipient, send the link.
5. Landing copy on `/try`: business name, "Talk to our AI receptionist", a privacy line.
6. Optional: a toggle to switch TTS to ElevenLabs for a high-stakes demo ($5 month plan).

**Done when:** three people unfamiliar with the project each complete a booking from the link alone, without help.

---

## MVP acceptance checklist

- [ ] A prospect can book, reschedule and cancel by voice from a link, in English.
- [ ] No double bookings (concurrency test + exclusion constraint).
- [ ] The agent never books without a read-back and a "yes".
- [ ] WhatsApp confirmation arrives for registered numbers; the dashboard shows a preview for everyone.
- [ ] Dashboard: calendar, calls with transcript, recording, summary and latency, messages, settings, businesses.
- [ ] p50 voice-to-voice ≤ 1.2 s measured from a UAE network; barge-in ≤ 250 ms.
- [ ] Eval suite ≥ 90% pass, zero hallucinations.
- [ ] Monthly bill: $0 (Anthropic usage within the $5 top-up).

## Risk triggers and responses

| If… | Then… |
|---|---|
| WebRTC is blocked on UAE networks (M1b) | Demo on known-good Wi-Fi now; move the phone channel (Telnyx `+971`) ahead of other work |
| Arm wheels fail (M1b) | Run the agent on your laptop with a Cloudflare Tunnel for demos; keep API/dashboard on Oracle |
| Haiku fails evals (M6) | Switch `LLM_MODEL` to `claude-sonnet-5-5`; cost rises but stays a few dollars a month at demo volume |
| Oracle capacity is unavailable for days | Laptop fallback; retry another time of day |
| The Deepgram credit runs low | Check usage monthly in the console; at ~$0.02/call minute it should last 10,000 minutes |

## After the MVP (not in this plan)

Phone channel via Telnyx `+971`, verified WhatsApp Business sender, ElevenLabs by default, Arabic (Phase 4), paid VM and LiveKit plan, and production hardening per DESIGN §11.
