# Operating Cost Estimate — per business, per day

Last updated: 2026-10-19 · Companion to [DESIGN.md](DESIGN.md)

> This is the **production** cost per client business. The MVP showcase runs on free tiers at $0/month; see DESIGN.md §1.1.

This estimates what it costs **us** to run the voice booking agent for one UAE business in production: phone line, speech, AI and messaging. It is not the price we charge. All prices are in USD, with AED at 3.6725.

> **Confidence.** Vendor prices come from public pricing pages and third-party summaries dated mid-to-late 2026. Several could not be confirmed on an official page (marked *unverified*). No UAE-specific call-volume data was found, so the volumes below are assumptions based on dental-practice benchmarks. Re-check against vendor consoles before quoting a client.

## 1. Assumptions

### Call volume

| Business profile | Example | Appointment calls / day | Avg call length |
|---|---|---|---|
| Small | Salon, single-dentist clinic | 30 | 3 min |
| Medium | Multi-chair dental / physio clinic | 60 | 3 min |
| Busy | Polyclinic, large spa | 120 | 3 min |

- Benchmarks for dental practices put volume at **40–80 calls/day**; human receptionist calls average **4–6 min** in those benchmarks. These benchmarks are US-based.
- An AI booking call is assumed to average **3 min**: no hold time and no small talk, but read-backs of name and number add a little. A 5-minute sensitivity case is in §4.
- **60%** of calls end in a booking, reschedule or cancellation, which triggers a WhatsApp confirmation. The rest are questions, messages or hang-ups.
- Opening hours do **not** change the cost: every vendor charges per use, so a 24/7 agent costs the same as one that runs 9 a.m. to 9 p.m. for the same number of calls. Evening calls are common in the UAE and are captured at no extra cost.

### Conversation shape (per call minute)

- The agent speaks about **40%** of the time, at ~900 characters per minute of speech, so **~360 characters of TTS per call minute**.
- ~6 conversational turns per minute and ~10 Claude requests per minute (tool calls add round trips). Each request is ~5k input tokens, mostly cached, and ~150 output tokens.

## 2. Unit prices

### Per call minute

| Item | Vendor / basis | Price used | Per call minute |
|---|---|---|---|
| Speech-to-text | Deepgram Flux, $0.008/min of audio *(unverified, one source)* | $0.008/min | **$0.0080** |
| Text-to-speech | ElevenLabs Flash, $0.05 per 1k characters × 360 chars | $0.05/1k chars | **$0.0180** |
| Language model | Claude Haiku 5.5, $0.10 / $0.50 per M tokens; ~10 requests with caching | — | **$0.0030** |
| Media / agent hosting | LiveKit Cloud: SIP caller ~$0.004/min + agent WebRTC $0.0005/min *(SIP rate unverified)* | — | **$0.0045** |
| Phone line | Telnyx inbound to a UAE number ~$0.0032/min *(template rate, unverified)*, rounded up | $0.005/min | **$0.0050** |
| Call recording | LiveKit Egress audio + R2 storage *(estimate)* | — | **$0.0050** |
| **Total** | | | **≈ $0.044 / min** |

### Per call or per booking

| Item | Basis | Cost |
|---|---|---|
| WhatsApp confirmation | Meta UAE utility rate ~$0.0157 + Twilio fee $0.005 | **$0.021 per booking/change** |
| Call summary | Claude Sonnet 5.5, ~2k tokens in, ~200 out | **$0.006 per call** |

### Fixed monthly (per business)

| Item | Monthly | Notes |
|---|---|---|
| UAE `+971` number (Telnyx) | $150 | Vendor page says "from $150"; requires company KYC |
| LiveKit Cloud paid plan | ~$50 | *Unverified.* The free tier (1,000 agent minutes) is too small for production |
| VM (agent + API + DB + dashboard) | $25 | Can be shared across several businesses |
| **Total** | **$225 / month = $7.50 / day** | |

## 3. Daily cost per business (3-minute average call)

| Profile | Calls | Call minutes | Voice pipeline | WhatsApp | Summaries | Fixed | **Total / day** | **AED / day** | Per call | **Per month (30 d)** |
|---|---|---|---|---|---|---|---|---|---|---|
| Small | 30 | 90 | $3.91 | $0.37 | $0.18 | $7.50 | **$11.97** | **AED 44** | $0.40 | $359 · AED 1,319 |
| Medium | 60 | 180 | $7.83 | $0.75 | $0.36 | $7.50 | **$16.44** | **AED 60** | $0.27 | $493 · AED 1,811 |
| Busy | 120 | 360 | $15.66 | $1.49 | $0.72 | $7.50 | **$25.37** | **AED 93** | $0.21 | $761 · AED 2,795 |

Where the money goes (medium clinic): fixed costs ~46% (mostly the UAE number), ElevenLabs ~20%, Deepgram ~9%, Claude ~3%.

## 4. Sensitivity

| Change (medium clinic) | Total / day |
|---|---|
| Baseline (60 calls × 3 min) | $16.44 · AED 60 |
| Calls average 5 min | $21.66 · AED 80 |
| Arabic voice (ElevenLabs Multilingual, 2× TTS price) | $19.68 · AED 72 |
| All per-minute vendor prices 50% higher than assumed | $20.35 · AED 75 |
| Web channel only (no UAE number, no telephony) | ~$9.82 · AED 36 |
| Fixed costs shared across 5 client businesses (VM + LiveKit plan) | ~$14.44 · AED 53 |

## 5. Takeaways

- **One AI booking call costs us about $0.20–0.40 (AED 0.75–1.50) all-in.** The voice pipeline is ~$0.13 for a 3-minute call; the rest is the fixed phone number and platform spread over the day's calls.
- **Fixed costs dominate for small businesses.** The UAE phone number alone is $5/day. Sharing the VM and LiveKit plan across clients is the main lever as we sign more businesses.
- **Of the per-minute costs, speech is the biggest share and the LLM is negligible.** Moving from Opus to Haiku mattered for speed, not money. If TTS spend needs cutting later, Cartesia or Deepgram's own TTS (~$0.045 per 1k chars) are the levers. They come at some cost to voice quality, which we chose to protect.
- **Pricing headroom.** At ~AED 1,800/month to run a medium clinic, a subscription priced well above that still costs the clinic far less than an extra full-time receptionist in the UAE.
