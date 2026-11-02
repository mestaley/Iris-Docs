# Clean-Sheet "Layman's Jarvis" — Architecture (2026-11)

> A from-first-principles design for a local-first home voice assistant for a technical
> homeowner. Written blind to any existing system on purpose, so the reasoning isn't anchored.
> Opinionated by design. Model/benchmark choices reflect the open-model landscape as of late 2026.

---

## 0. Design philosophy (the five rules everything else obeys)

1. **Latency is the product.** A correct answer at 4 s feels broken; a slightly worse answer at
   800 ms feels magic. Every architectural choice is scored first on perceived latency.
2. **Determinism at the edges, intelligence in the middle.** The LLM is allowed to be fuzzy about
   *intent*. It is never allowed to be fuzzy about *facts that have a ground truth* — entity IDs,
   device state, whether an action succeeded. Those are resolved and verified by boring code.
3. **Local-first, not local-dogmatic.** ~97% of real home-assistant turns (control, timers, music,
   weather, calendar, everyday Q&A) are handled better locally than via cloud (latency, privacy,
   offline resilience). The long tail of genuinely hard knowledge is where dogma hurts — handle it
   with tools and an honest punt, plus one *optional, explicit, rate-limited* cloud escape hatch.
4. **Boring beats clever.** Prefer a debuggable state machine and a rules engine over an emergent
   cognitive architecture. Elaborate memory-consolidation / affect loops add latency and failure
   surface and tend to corrupt the simple stuff. Earn complexity with a measured win.
5. **Never lie about the house.** The single worst failure is confidently claiming a light turned
   on when it didn't. Grounding + verification is a first-class subsystem, not a nicety.

---

## 1. Physical & compute topology

```
                        LAN (wired where possible)
  ┌──────────────┐     ┌──────────────────────────────┐     ┌───────────────────┐
  │ Room endpoints│◀──▶│  audio-node  (8c CPU, 16GB,    │◀──▶│  gpu-node          │
  │ (Pi-class ×N) │ WS  │  NO GPU)                      │ gRPC│  2× RTX 5060 Ti +  │
  │ mic+spk+AEC   │     │  - wake fallback / VAD mux    │     │  Coral Edge TPU    │
  │               │     │  - DIALOG MANAGER (brain-stem) │     │  (32GB VRAM total) │
  └──────────────┘     │  - HA bridge + resolver        │     │  - STT / TTS / LLM │
                        │  - timers/reminders/state DB   │     │  - embeddings      │
        ┌───────────────│  - memory (SQLite + vectors)   │     └───────────────────┘
        │               └──────────────────────────────┘
   ┌────▼─────┐
   │ Home     │  (already on LAN; REST + WebSocket event stream)
   │ Assistant│
   └──────────┘
```

**Why split audio-node from gpu-node.** The dialog manager, timers, reminders, HA control and
the entity resolver are the "brain stem" — they must stay responsive even while the GPU box is
rebooting, swapping a model, or thermally throttling. Put them on the cheap GPU-less node so a
"set a 10-minute timer" / "turn off the kitchen light" never depends on GPU uptime. The GPU box is
a pure inference appliance (STT/TTS/LLM/embeddings) behind gRPC; if it's down, the system degrades
to a deterministic command-only assistant rather than dying.

### 1.1 VRAM allocation (the opinionated part)

| GPU | Resident | VRAM | Role |
|-----|----------|------|------|
| **GPU0 (16GB)** | **Qwen3-30B-A3B-Instruct**, AWQ/INT4, vLLM | ~11GB weights + ~4GB KV | The brain. MoE: 30B total / ~3B active → dense-14B-class quality at 8B-class decode speed (~90–130 tok/s). |
| **GPU1 (16GB)** | **Parakeet-TDT-0.6B-v3** (STT) ~2GB · **Kokoro-82M** (TTS) <1GB · **Qwen3-Embedding-0.6B** + **Qwen3-Reranker-0.6B** ~2GB · **Qwen3-4B-Instruct** (router/guard) INT4 ~4GB | ~9–10GB | Everything real-time + the small always-on model. ~6GB headroom for KV / bursts. |

Rationale: the workload is **latency-bound, not throughput-bound** (one or two concurrent users,
not a datacenter). A single MoE model that decodes fast beats a bigger dense model that decodes
slowly. Keeping STT+TTS+embeddings off the brain GPU means a long LLM generation never starves the
TTS stream — the voice loop and the thinking loop don't contend.

### 1.2 The sellable GPUs (2× RTX 5080 16GB + 1× RTX 4080)

**Recommendation: sell all three.** They are 16GB cards — same VRAM ceiling as the 5060 Tis, so
they don't unlock a bigger model; they only add compute this workload won't use at one/two users.
The cash is worth more than the marginal tokens/sec. **One exception worth considering:** keep a
single 4080 as a *warm spare / burst card* if you want the option to A/B a **dense Qwen3-32B
(AWQ, ~20GB across two cards)** for max-reasoning mode, or to run STT and the brain fully isolated.
That's a ~$600–900 opportunity cost against maybe 10–15% better hard-reasoning quality that the
cloud escape hatch (§4.4) already covers more cheaply. Default: **sell, don't hoard.**

---

## 2. Audio front-end

**Endpoints (Raspberry-Pi-class, one per room).** Each does three things on-device and nothing
heavy:

1. **Wake word — `microWakeWord` on an ESP32-S3 / Pi.** Runs locally, ~20 ms, tiny model, no
   network round-trip. Train a custom "Hey Jarvis"-style phrase from the Piper-synthetic pipeline.
   This is the right place for wake detection: it's always-on, must be instant, and must never
   ship raw ambient audio off the device until triggered (privacy).
2. **Acoustic echo cancellation (AEC) + beamforming on-device.** This is non-negotiable for
   barge-in. Use a mic array with hardware/DSP AEC (XMOS XVF-class dev boards, or ReSpeaker arrays
   + `speexdsp`/WebRTC-APM). The speaker's own output is the echo reference so the device can hear
   the user *over its own TTS*. Without per-endpoint AEC, barge-in is impossible and you get a
   half-duplex walkie-talkie.
3. **Stream after wake.** On wake, open a WebSocket to audio-node and stream 16 kHz mono PCM with
   voice-activity flags. Stop streaming on endpoint.

**VAD / endpointing — on audio-node.** Use **Silero VAD v5** (CPU, ~1 ms/frame) for speech
presence, plus a **semantic/turn-end heuristic** layered on the STT partials (see §3): endpoint
when (a) VAD silence exceeds an adaptive hangover (180–400 ms, longer if the partial transcript
ends mid-clause) OR (b) STT emits a stable final. Adaptive hangover is the single biggest felt-
latency lever — too short truncates "turn off the … kitchen light"; too long feels sluggish.

**Barge-in.** While TTS is playing, the endpoint keeps its mic open (AEC removes the echo) and runs
a lightweight VAD. Local double-talk detection → send a `user_speaking` event to audio-node →
audio-node (a) cancels the TTS stream immediately (<50 ms), (b) sends a cancel to the LLM
generation, (c) transitions dialog state back to LISTENING. The user never has to wait for Jarvis
to finish a sentence.

**Multi-room routing.** audio-node owns a **session arbiter**. Each endpoint has a room label.
On simultaneous wakes, the loudest/closest wins the turn (RMS + optional speaker-ID); others are
suppressed for a short window to avoid double-answers across open-plan spaces. Replies are routed
back to the endpoint that won. Follow-up turns (no wake word, §5) stay bound to that endpoint.

### 2.5 Google Coral Edge TPU — where it earns its place (and where it doesn't)

A Coral Edge TPU (~4 TOPS, int8 tflite only, in the GPU server) is a power-efficient home for
**always-on, small, fixed classifiers** — exactly the always-on work you don't want burning CPU
cores or waking the GPUs. It cannot touch LLM/STT/TTS. Put it to work on three things:

- **Server-side wake-word second stage / false-trigger guard (earns it).** The primary wake word
  stays on-device (microWakeWord, §2), but the ESP32-class first stage is deliberately loose to
  avoid misses. A Coral-hosted **openWakeWord/second-stage verifier** re-checks the ~1 s pre-roll
  the endpoint ships on trigger, and *rejects* false wakes (TV, "fires" vs "Iris") before STT/LLM
  ever spin up. Runs continuously at near-zero CPU/GPU and near-zero idle watts.
- **Acoustic event detection (earns it, big).** A **YAMNet-int8** model on the Coral runs an
  always-on audio-scene classifier on a low-bitrate ambient tap: smoke/CO alarm, glass break,
  doorbell, baby cry, dog bark, running water. These become proactivity/notification signals
  (§8) and safety alerts — all without the CPU or GPUs doing continuous audio work. Privacy-safe:
  it emits a *class label*, never audio off the device.
- **Presence/vision for proactivity (earns it, opt-in).** A cheap USB camera + **MobileNet-SSD /
  person-detection int8** on the Coral gives room-level **person-present + "just sat/stood"**
  signals — the exact trigger behind "it's dim and you just sat down — lamps?" (§8). It emits only
  a boolean/box count, never frames, so it's a privacy-preserving occupancy sensor that also
  dog-filters (person-class vs not). Opt-in per room.

**Where it does *not* earn its place:** VAD (Silero on CPU is already ~1 ms/frame — moving it to
the Coral buys nothing and adds a transfer hop); any LLM/STT/TTS work (not supported); and the
primary wake word (that belongs on the endpoint MCU, not a server-side accelerator — the Coral is
the *second* stage, not the first). Net: the Coral absorbs the always-on "listening for the world"
duties, keeping the CPU free for the dialog loop and letting the GPUs idle (and down-clock) until a
real turn arrives.

---

## 3. STT

**Model: NVIDIA Parakeet-TDT-0.6B-v3** on GPU1.
- English, ~6.3% avg WER on the Open ASR Leaderboard — beats Whisper large-v3 (~7.4%) at ~¼ the
  size, and runs far faster. For **near-field command speech** (close mic, short utterances) the
  effective WER is excellent and the vocabulary is home-domain-biased via a small boosting list
  (room names, device aliases, media/artist names injected from the HA registry + music library).
- **Streaming**: feed 200–400 ms chunks; use partials for the turn-end heuristic, take the final on
  endpoint. Finalization latency after the user stops: **~80–150 ms**.
- **Fallback: Whisper-large-v3 (faster-whisper / CTranslate2)** kept on disk, loaded on demand for
  (a) non-English speakers / multilingual family members, (b) long dictation. It trades latency for
  language coverage — not the default path.

Why not run STT on the CPU audio-node? Parakeet on GPU finalizes in tens of ms and leaves the CPU
free for the real-time dialog loop. If the GPU box is down, audio-node falls back to a tiny CPU
Whisper (base/distil) for command-only survival mode — worse WER, but the house still responds.

---

## 4. The brain

### 4.1 Two-tier model strategy

- **Tier-S "reflex/router" — Qwen3-4B-Instruct (INT4, GPU1, always resident).** TTFT ~120–180 ms.
  Handles: intent classification, device-control tool calls, timers, the "which tier?" routing
  decision, and short canned conversational turns. ~70–80% of turns never touch the big model.
- **Tier-L "reasoning" — Qwen3-30B-A3B-Instruct (GPU0).** Handles multi-step reasoning, open Q&A,
  summarization, anything the router flags as non-trivial. Decode ~90–130 tok/s (MoE).
- **Tier-X "max reasoning" (optional)** — dense Qwen3-32B loaded on demand across both idle GPUs,
  or the cloud escape hatch (§4.4). For the rare "really think hard about this" request.

Routing is done by Tier-S in the same forward pass that parses intent: it emits a tool call *or*
a `route: {tier, reason}` token. Cheap, no extra round-trip.

### 4.2 Reliable tool / function calling

This is where open models quietly fail, so it's engineered hard:

1. **Constrained decoding.** All tool calls are generated under a **GBNF grammar / JSON-schema
   constraint** (vLLM structured outputs or llama.cpp grammars). The model physically cannot emit
   malformed JSON or hallucinate a tool name — the sampler masks illegal tokens. This alone raises
   tool-call validity from "usually" to "always well-formed."
2. **Small, flat tool surface.** ~12–15 tools, not 100. Verbs, not device IDs:
   `set_device_state(area, device_type, attribute, value)`, `query_device_state(...)`,
   `play_media`, `set_timer`, `create_reminder`, `get_weather`, `web_search`, `recall_memory`,
   `remember_fact`. Fewer tools → far higher selection accuracy.
3. **Targets are natural language, resolved downstream.** The model says
   `area="kitchen", device_type="light", attribute="on"` — it **never** emits an HA `entity_id`.
   The deterministic resolver (§6) owns IDs. This removes the #1 source of actuation hallucination.
4. **Two-shot repair.** If a call fails schema/resolution, the error is fed back once for a
   correction turn. After two failures, punt verbally ("I couldn't find a light called X").
5. **Separate the "say" from the "do."** The model's spoken text is generated *after* tool results
   come back (or streamed as a pre-action acknowledgement that is explicitly provisional), so it
   never narrates success before verification (§6).

### 4.3 Answering hard questions without cloud

The honest local ceiling: a 30B-A3B/14B-class model is a strong generalist and a weak specialist.
Strategy:

- **Default local answer** for general knowledge, reasoning, how-to, cooking, kids' homework, etc.
- **Tool-augment before giving up.** For anything time-sensitive or factual-lookup-shaped
  (news, prices, "who won", specs), route to **`web_search` → local SearXNG meta-search → fetch
  top N → rerank (Qwen3-Reranker-0.6B) → read & summarize locally.** This closes most of the gap
  without a cloud LLM; the cloud isn't doing the knowing, the web is, and the local model just reads.
- **Calibrated honesty.** The model is prompted and tuned to *punt* rather than confabulate:
  "I'm not sure and I can't verify that offline — want me to look it up?" A wrong-but-confident
  answer is treated as a worse failure than "I don't know." This is a feature, not an apology.
- **Graceful offline.** With no internet: control/timers/music/local-knowledge still work;
  web-dependent queries say so plainly.

### 4.4 The one sanctioned cloud escape hatch

**Optional, off by default, explicit.** A single `ask_cloud(question)` tool, gated by: (a) user
opt-in per-query ("I'll have to phone a friend for this — ok?") or a standing "hard mode" toggle,
(b) a rate limit + monthly budget cap, (c) a privacy scrubber that strips names/locations/entity
data before sending. Used for the genuine long tail (deep technical, niche, long-context synthesis).
Expected usage: **low single-digit percent of turns.** When offline or over budget, it transparently
falls back to §4.3's local-tool path + punt. This is the pragmatic answer to the hard constraint:
near-fully-local, with a clearly-bounded pressure valve instead of dogma.

---

## 5. Dialog management

A small, inspectable **state machine** on audio-node (the brain stem), not an LLM agent loop:

```
IDLE ──wake──▶ LISTENING ──endpoint──▶ THINKING ──first clause──▶ SPEAKING
  ▲                │                                                  │
  │                └── barge-in / user_speaking ◀─────────────────────┘
  │                                                                   │
  └──────────── follow-up window closes (no speech) ◀── SPEAKING ends ┘
```

- **Barge-in** (SPEAKING→LISTENING): cancel TTS + LLM gen in <50 ms (§2).
- **Follow-up window:** after a reply, keep the mic "hot" on the winning endpoint for ~6–12 s so
  the user can continue **without the wake word** ("…and the lamp too"). Extended if the reply was
  a question. This is the single biggest "feels conversational" win.
- **Confirmations:** reversible/cheap actions (lights, scenes, media, thermostat within a band)
  execute immediately — bias to action; a mistake is "I can flip a switch back." **Risky actions
  (locks, garage, thermostat outside a safe band, anything flagged) require a spoken confirm** and
  are rate-limited. Locks get special treatment (§6.4).
- **Interruptibility of long answers:** long replies are chunked by clause and spoken
  progressively; "stop"/"never mind"/barge-in halts cleanly.
- **Pre-action acknowledgement:** for actions that take >300 ms to verify, Jarvis may say a short
  provisional ("on it") then confirm/correct after verification, rather than narrating success up
  front.

---

## 6. Smart-home actuation & grounding (the anti-lying subsystem)

### 6.1 Entity registry sync
audio-node subscribes to HA's **WebSocket event stream** and mirrors the entity registry, area
registry, and current states into a local SQLite table, kept live. Each entity gets: `entity_id`,
`friendly_name`, `area`, `domain`, `device_class`, plus a **user-editable alias table**
("the good lamp" → `light.living_room_corner`).

### 6.2 Resolution (NL → entity_id), deterministic
The resolver takes `{area, device_type, attribute}` from the model and resolves to concrete
`entity_id`s by: exact/alias match → area+domain filter → fuzzy + embedding similarity
(Qwen3-Embedding) as a tiebreaker → **disambiguation question if ambiguous** ("the ceiling light or
the lamp?"). The LLM never sees or invents IDs. Resolution is logged with its confidence.

### 6.3 Execute → verify → confirm (never trust the 200 OK)
1. Call the HA service (`light.turn_on`, etc.).
2. **Verify against ground truth**, not against the HTTP response: wait for the `state_changed`
   event for the target entity (with a ~1–2 s timeout), confirming the *end state* matches intent
   (not merely "a change happened").
3. Only then produce the spoken confirmation. On timeout/mismatch: **speak a correction** ("the
   kitchen light didn't respond — it still shows off"). This closes the "HA accepted ≠ device
   obeyed" gap (unreachable Zigbee bulb, offline switch).

### 6.4 Never hallucinate state — and the lock rule
- **Device state is always queried LIVE, never prefetched/cached into the prompt.** Caching device
  state is the root cause of "the light is on" hallucinations. Only genuinely slow *external* data
  (weather forecast) is prefetched; fast local entity reads are done on demand and fed back.
- **Locks / safety domain:** read-and-write both require explicit, authenticated confirmation.
  A conservative default is correct here even though the house bias is "toward action" — the cost
  asymmetry flips for physical security. The lock domain can be allow-listed per-user and is the
  one place where ask-first beats act-first.

---

## 7. Memory & personalization (local, boring, effective)

Two stores, both on audio-node, both local:

1. **SQLite — structured truth.** Preferences (`prefers thermostat 68 at night`), people & voice
   profiles, device aliases, routines, reminders/tasks/timers, and an append-only **turn/event
   log**. This is authoritative, editable, and diffable. Most "personalization" is here, not in a
   vector DB.
2. **Vector store — `sqlite-vec` (same file) or LanceDB.** For semantic recall of free-form facts
   and past-conversation snippets. Embeddings: **Qwen3-Embedding-0.6B**; retrieval is **hybrid**
   (BM25/FTS5 + vector) then reranked (Qwen3-Reranker-0.6B), top-k injected into Tier-L context.

**Write policy (deliberately conservative).** Facts are stored **additively and non-destructively**
— a new preference supersedes by recency, it does not delete the old row (so a bad write is
reversible and auditable). No background "consolidation" job rewriting memories into a lossy
summary — that pattern silently corrupts preferences and is explicitly rejected. Remembering is
mostly an explicit tool call (`remember_fact`) plus a light heuristic for obvious durable
preferences, reviewed in a visible log the user can edit.

**Speaker identity.** ECAPA-TDNN voiceprint per family member (and a reject class for TV/guests)
gates *who* Jarvis responds to and *whose* preferences apply. Primary user is the default; family
members get their own preference scope.

**Privacy.** Everything local; SQLite on an encrypted volume; raw audio discarded after STT unless
the user opts into retention; the cloud hatch scrubs PII. No telemetry leaves the LAN.

---

## 8. Proactivity (helpful, not a nag)

A **deterministic rules/offers engine** on audio-node, fed by HA events (occupancy, illuminance,
media state, door/motion), time, calendar, **and the Coral-derived signals (§2.5): person-present/
"just sat down" from vision, and acoustic events (doorbell, alarm, baby cry)**. Flow: **signal →
candidate offer → gate → (maybe) speak → learn.** The Coral presence signal is what makes the
canonical "you just sat down — want the lamps?" trigger reliable and dog-proof without a cloud or
GPU in the loop.

Gating (all must pass):
- **Presence + recency:** only to a person actually in the room who was recently active (dodges the
  empty-room and sleeping-user cases; a passed speaker-gate turn proves it's a real person, not the
  dog).
- **Cooldown & budget:** per-room rate limit + a daily proactivity budget. Repeated declines of a
  given offer type back it off (bandit-style).
- **Reversible & cheap only:** proactivity may *offer* ("it's dim and you just sat down — lamps?")
  and, if configured, *do* small reversible things; it never proactively touches locks/thermostat-
  out-of-band/anything risky.
- **Do-not-disturb windows** (sleep, calls, focus) hard-mute offers.
- **Offer-before-act** for anything beyond trivial, with one-word accept.

Start pure observe-only (log would-be offers, show the user what it *would* have said) and only
enable speaking per-rule once the false-positive rate is acceptable. Proactivity earns trust slowly.

---

## 9. Latency budget

**Target: wake → first audio of reply.**
- Simple control ("turn on the lamp") — **≤ 900 ms** to spoken confirmation.
- Conversational reply — **first audio ≤ 1.3 s**, then streaming.

Breakdown (simple command, happy path):

| Stage | Budget | Notes |
|-------|--------|-------|
| Wake detect (on-device) | ~20 ms | microWakeWord, no network |
| User speaks + endpoint | (user time) + **180–300 ms** | adaptive VAD hangover — dominant tunable |
| STT finalize | **80–150 ms** | Parakeet streaming, final on endpoint |
| Route + tool call (Tier-S) | **120–200 ms** | TTFT of 4B, constrained decode of one call |
| Resolve + HA execute | **50–150 ms** | local resolver + service call |
| Verify (state_changed) | **(overlapped)** | verification overlaps TTS prep; provisional "ok" if slow |
| TTS first audio (Kokoro) | **150–300 ms** | time-to-first-chunk, streamed |
| **Total (excl. user speech)** | **~0.6–1.0 s** | feels immediate |

How we hit it: everything streams and overlaps — STT partials drive endpointing; the small model
runs at low TTFT; tool execution and TTS warm-up overlap verification; TTS is synthesized and
played **clause-by-clause** so audio starts before the full sentence exists. The brain GPU and the
voice GPU never contend. For Tier-L answers, speak the first clause as soon as it's generated.

---

## 10. Reliability & ops

- **Graceful degradation ladder:**
  1. All up → full assistant.
  2. GPU box down → audio-node **command-only survival mode**: wake + tiny CPU-Whisper + Piper CPU
     TTS + resolver + HA control + timers. "I can control the house but I can't think hard right now."
  3. Internet down → local everything; web/cloud queries say so.
  4. HA down → Jarvis reports it plainly instead of pretending to actuate.
- **Failure modes designed for:** unreachable device (caught by verify §6.3), ASR mishear of wake
  (false-trigger budget + speaker gate), double-fire across rooms (session arbiter), TTS dropout
  (Piper fallback voice chain), model OOM (health-checked, auto-reload, Tier-S can answer alone).
- **Observability:** structured logs shipped to a local **Loki + Grafana**, keyed by a `turn_id`
  that threads every stage (wake→STT→route→tool→verify→TTS). One dashboard = per-turn drill-down
  with end-to-end latency and where the ms went; one = grounding (verify failures, resolution
  confidence); one = proactivity (offers made/accepted/declined). Metrics via Prometheus
  node/GPU exporters. If you can't see a turn, you can't fix it.
- **TTS primary/fallback:** Kokoro-82M on GPU (54 voices, best quality/compute) as primary;
  **Piper on the CPU audio-node** as the always-available fallback (~35 ms TTFB, lower quality) so
  the house is never mute. Guard the clause-splitter against abbreviations/decimals (U.S., Inc., 3.5).

---

## 11. The 3–4 hardest problems & concrete approaches

**(1) Full-duplex barge-in on cheap endpoints.** Hardest because it spans hardware, DSP, and
cancellation plumbing. Approach: mandate **per-endpoint AEC** (hardware DSP mic arrays — the echo
reference is the local speaker feed), run a **double-talk VAD** that stays active during playback,
and build **end-to-end cancellation** (endpoint → audio-node cancels TTS stream + LLM gen in
<50 ms). Don't try to do AEC server-side; the reference and the mic must be co-located. Accept that
sub-$15 endpoints can't do this — spend on the mic array, save on everything else.

**(2) Never lying about device state.** Approach: deterministic NL→entity resolver (model never
sees IDs), **verify against the `state_changed` end-state** not the HTTP 200, speak corrections on
mismatch, and **query state live, never prefetch/cache it** into the prompt. This is a whole
subsystem (§6), and it's the one that earns trust or destroys it.

**(3) The local reasoning ceiling / hard Q&A without cloud.** Approach: tiered routing (don't waste
the big model on trivia, don't ask the small model to reason), **tool-augment via local web search
+ rerank + local reading** before conceding, **calibrated honest punts** over confabulation, and a
**single bounded, opt-in cloud hatch** for the true long tail. The design treats "I don't know, want
me to look?" as a correct answer, which is what makes near-fully-local viable.

**(4) Tool-calling adherence + latency under real use.** Approach: **grammar-constrained decoding**
(malformed/hallucinated calls are impossible), a **small flat tool surface**, verbs-not-IDs,
two-shot repair, and streaming/overlap so adherence doesn't cost felt latency. Plus speaker-gating
and the session arbiter so multi-room/family use doesn't cause double-answers or mis-attributed
memory.

---

## 12. Phased build plan (usable fast, then layered)

- **Phase 0 — Voice loop skeleton (days).** One endpoint + HA. microWakeWord → audio-node → GPU
  STT (Parakeet) → Tier-S model with ~4 tools (control/query/timer/weather) → resolver + verify →
  Kokoro TTS. Target: reliably turn lights on/off with verification, set timers, tell weather.
  This is already a genuinely useful assistant.
- **Phase 1 — The brain (1–2 weeks).** Add Tier-L (30B-A3B), constrained tool-calling grammar,
  routing, full tool surface, local web-search tool, honest-punt behavior. Now it answers questions.
- **Phase 2 — Conversation (1–2 weeks).** Barge-in (AEC endpoints), adaptive endpointing, follow-up
  window, multi-room session arbiter, speaker ID gate. Now it *feels* like Jarvis.
- **Phase 3 — Memory & personalization (1 week).** SQLite truth store + sqlite-vec hybrid recall,
  `remember_fact`/`recall_memory` tools, per-person preference scopes, editable memory log.
- **Phase 4 — Proactivity (ongoing).** Rules/offers engine in observe-only, then enable per-rule as
  false-positive rates allow. Learn from accept/decline.
- **Phase 5 — Hardening (ongoing).** Full degradation ladder, Loki/Grafana turn-tracing,
  CPU-survival mode, optional cloud hatch, lock-domain allow-listing, backup/restore of the DB.

---

## 13. Where "purity" helps vs hinders (honest note to the commissioner)

**Local-first helps** — overwhelmingly — on latency, privacy, offline resilience, and cost. For
control, timers, music, weather, everyday Q&A, and personalization, local is not a compromise; it's
*better* than cloud. Keep that rule hard.

**100%-local dogma hinders** only at the edges: the genuine long tail of hard/niche knowledge, and
fast-moving facts. The pragmatic fix isn't to abandon local — it's (a) tool-augmented local
(web-search + read) and (b) one small, explicit, budgeted, PII-scrubbed cloud hatch that's off by
default. Refusing even that costs you real capability for an ideological win the user won't feel.
Treat "I don't know, want me to look it up?" as success, and the dogma mostly stops mattering.

**Over-engineered cognitive architecture hinders more than it helps.** Elaborate memory
consolidation, affect/emotion loops, multi-agent "cortex" deliberation, emergent planning — these
add latency, failure surface, and (for memory consolidation specifically) a real risk of *silently
corrupting the simple preference data that actually drives personalization*. They are seductive
because they feel like "real intelligence," but the user experiences them as slowness and
flakiness. **Invest cognition only where it pays at the turn level:** a great resolver, great
retrieval, great grounding/verification, great endpointing. Keep the control plane a boring,
inspectable state machine. The magic the user wants is *fast, correct, and never lies about the
house* — not a mind. Build the mind only after the brain stem is flawless, and only where a
measured win justifies the complexity it adds.

---

*End. This is a clean-sheet proposal; model choices should be re-benchmarked on the actual
5060 Ti hardware before freezing (Open ASR / BFCL numbers above are blog-sourced and indicative).*
