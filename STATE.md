# Iris — Current State

_Last verified: 2026-08-22 (live SSH sweep of core-ai / audio-ai / world-ai / voice-ai)._

## Current focus
Whole-system review DECIDED (ADR-001) and **implemented 2026-08-23** — keep the streaming
trunk, retune the gate, close two capability gaps, gate the dead writer, prune what's safely
isolated. Status after first live soak (2026-08-23 15:03-15:20):
- **Step 1 (speaker 0.28)** — live, HELPED but INSUFFICIENT: genuine Matt still rejected at cos ~0.26 on vg-02 → re-enroll voiceprint (BANKED, room too noisy). Did NOT admit TV. (ISSUE-001)
- **Step 2 (grounding)** — live, WORKING ("Getting the basement lights" = clean honest ack). (ISSUE-005)
- **Step 3 (dual-rail multi-intent)** — **REVERTED** (cc116f3, thalamus restarted 15:20): caused live over-prefetch; baseline phi3.5+working-memory-field over-prefetch is a separate pre-existing tuning problem. Redo with an eval harness. (ISSUE-002)
- **Step 5 (consolidation write gate)** — live + trigger timer disabled.
- **Step 4** — Spotify creds pruned; rest staged (ISSUE-004).
New: **ISSUE-006** (say a backend 503 honestly, don't deflect) — captured, deliberate fix pending.
Note: Meridian weather flapped 502/503 ~15:05, recovered ~15:08 (transient).
Next: re-enroll voiceprint when quiet; redesign multi-intent + field tuning with evals; push owed commits.

**Docs consolidated 2026-08-23:** Matt's legacy `/opt/repos/Architecture/` notes reviewed (4-agent
fan-out) and merged — added `THESIS.md` (canonical), `PRINCIPLES.md`, `PATTERNS.md`,
`decisions/HISTORICAL-DECISIONS.md`, and ~25 still-open items into `BACKLOG.md` (incl. 5 ISSUE
candidates: Path-2 actuation, intentions scheduler, right-to-forget, contention-surfacing,
cognitive-event-stream). Deferred: DDS topic-table reference appendix, PROMPTS.md re-derived from
live code, Core-Substrate-as-history. One flag for Matt: the Davis soft-vs-hard prompt rule.

## V2 progress (live)
- **Conversation-log subscriber LIVE 2026-08-25** (iris-volition restarted): every turn's
  user text (FeatureStream) + Iris reply (FusionDecision) is joined by turn_id and persisted to
  `conversation_turns` (surface voice|web) — the transcript the web surface renders/resumes.
  Branch `feat/web-tasks-tools` (commit 7595914, push owed). 129 volition tests green.
- **Web-gateway LIVE on core-ai 2026-08-26** — container up (`~/iris-web-gateway`, docker host-net,
  restart:unless-stopped), `/health` ok, `CurrentFabricAdapter live on the bus`. Bridge PROVEN: a
  web-origin turn round-tripped a grounded weather answer (silent — web bypasses speech nodes).
  Browser access still needs the Cloudflare hostname + an auth model (Google client for Model A, or
  Cloudflare Access for Model B — AUTH.md). The /chat bridge + /api proxy to intake :8015 are wired.
- **Web-gateway SCAFFOLDED 2026-08-25** (new repo `iris-web-gateway`, commit 06eea8e) — thin
  FastAPI: `/api/*` proxies CRUD to intake `:8015`; `/chat` bridges to live cognition via the
  Turn-Entry Contract (`CurrentFabricAdapter` = long-lived DDS peer, domain 42, publishes a
  synthetic web-origin FeatureStream, reads FusionDecision.say). FakeAdapter runs the whole
  stack with no DDS (5 tests green). Google OAuth (authlib, iris@ G-Suite, allowlisted email).
  Bootstrap+htmx frontend. FeatureStream shape validated against real contracts on core-ai.
  **SSE streaming wired** (commit 802db0b): `/chat` stream=true emits delta frames + one final;
  frontend fills the bubble progressively. Deep-tier barge-in over web via a default-off
  `CORRECTION_GRACE_S` knob (deep-mode's cause=correction SpeechUtterance is node-agnostic).
  Honest limit: cortex doesn't token-stream yet → a plain web turn is one final today.
  **NOT deployed** — vendor contracts+bus, docker host-net on core-ai, + the web-origin spike =
  Matt's sudo step (README). Adapter B (new front-door) swaps in behind the same contract later.
- **recent_conversation slot LIVE 2026-08-25** (curator + cortex-logical restarted clean, commit
  2d9161e on `feat/web-tasks-tools`) — curator fetches a session's recent turns from intake `:8015`
  by session_id and seeds a cortex `RECENT CONVERSATION` block, so a resumed conversation has literal
  continuity. Mode **web** (confirmed loaded) = web-origin turns only, so it's inert until the web
  gateway is deployed; voice path untouched. Also **LIVE**: the reflex-catalog fix (commit e8c4714)
  scoping the world-ai fast reflex to `category=='actuation'` (device control only, no task/reminder
  CRUD) — confirmed `actuation_tools()` = control_device/control_media/select_content. Suite 19/19.
- **Voice-managed tasks LIVE** (add + read, grounded) — web-surface P1 first capability.
- **Creative cortex retirement — HALF-DONE Aug 25, made authoritative 2026-09-28.** The Aug-25
  step only set `FUSION_EXECUTIVE_ENABLE_CREATIVE=false` (which gates `ready_to_fire`, NOT synthesis)
  and **never stopped cortex-creative** — so the service kept running, its response still arrived, and
  `_fire`/`_synthesize` kept picking it (`creative_dominant`). Creative was the LIVE voice on every
  turn through 2026-09-28 (the persistent "leads with Right" tic + actuation chatter). Fix: `_fire`
  now IGNORES creative when the flag is off (fusion `feat/punt-escalation`, 2026-09-28) → logical owns
  the grounded voice for real. **Staged: restart iris-fusion-executive; then stop+disable
  cortex-creative to free GPU1 for the deep MoE tier.** With silent-on-clean-actuation, pure reflex
  commands now go silent (the supervisor is the sole correctness voice — Matt's model).
  Tradeoff: first-audio ~0.2s→~2.5-3s (no logical streaming yet — backlogged). Reversible via the flag.
- **Deep tier (qwen3-deep, 30B-A3B instruct) PROVEN LIVE 2026-08-25** — deliberates on grounded context, barges in over the front-door answer (~13s). GPU1. Still fires only on MANUAL punts (proactive punt signals = step 5). See changelog.
- **Shadow-punt measuring** (step 1). Surprise finding: no-streaming *feels faster* → perceived latency
  is VAD endpointing, not cortex first-audio (ISSUE-009).
- **Spurious-punt fixes LIVE + VALIDATED 2026-08-26** (turn-3ca38ed3 review): a reflex-satisfied
  "lights to 20%" voiced a filler + a rambling deep-tier barge-in. Root cause: cortex-logical empty
  pass1 → fusion neither_confident + self_uncertain punt. Fixed at both layers — fusion suppresses the
  proactive punt on actuation success AND overrides the filler → "Done." when actuated_ok
  (`feat/punt-escalation`); cortex-logical confirms "Done." on a fastpath-satisfied empty pass1
  (`feat/web-tasks-tools`, fires when ctx.fastpath present). Confirmed clean on turn-7cfb202e (says
  "Done.", no filler, no barge-in). Suites 40 + 57 green. Residual deep-mode grooming + warm-keep →
  ISSUE-010 (deferred).
- **Web-chat memory FIXED + confirmed 2026-08-26**: recent_conversation fetch parsed the intake
  response as a bare list but it's `{turns:[...]}` — always returned [], so no context injected.
  Fixed (curator .turns extraction, iris-curator restarted); two-turn web test recalls ("the number
  was forty-two", recent_conversation now in ambient_keys). Gateway resume/history + app.js cache-bust
  also fixed. If a browser still forgets, suspect stale cached app.js (hard-refresh) or CF SSE buffering.

## Versioning — this is Iris V2 (declared 2026-08-24)
- **V1 (live now):** the async streaming DDS fabric with **two-cortex cognitive pluralism** —
  logical + creative running in parallel, fusion-as-weaver. Voice-only surface. (Its own
  predecessor was the synchronous `fabric_brain` era.)
- **V2 (in parallel build):** a fundamental architecture shift, not an increment —
  (a) **serial-spine punt/escalation cognition** (a system-level MoE with an explicit programmable
  router; front-door + deep-MoE, creative retired — ADR-003), and (b) **multi-surface** (the web
  surface: tasks/reminders/projects + resumable voice-conversation continuity, shared state across
  voice + web — ADR-002). The two-cortex parallel model → escalation model is the cognition break;
  voice-only → multi-surface is the interface break. Together = V2.
- Bridge de-risked 2026-08-24: a synthetic `web`-origin turn was accepted+voiced-as-text by live
  fusion ("It's 69 degrees and partly cloudy.", origin_node=web, 3.4s) — the `BrainAdapter` seam works.

## Roadmap sequencing (decided 2026-08-24)
**Web surface (ADR-002/ISSUE-008) and cognition rearch (ADR-003) build in PARALLEL** — the UI's
CRUD half is fine on the current brain, but its conversational/research half is worthless on the
confabulating two-cortex stack, so they build together and land on the new brain. Coordinated by
ONE shared interface: the **existing `FeatureStream` (in) → `FusionDecision.say` (out) DDS
contract** (both design workers converged on this independently). The web-gateway publishes a
synthetic "web node" FeatureStream and reads the reply as text; the new cognition swaps in behind
the same contract (architecture-is-sovereign). Design docs: `design/WEB-SURFACE.md`,
`design/COGNITION-REARCH.md`.
- **Shared spike FIRST (de-risks both):** confirm fusion will voice-as-**text** a `web`-origin
  turn rather than drop it (streaming-node allowlists gate voice nodes; `web` must be accepted).
- Then rented GPU experiments (later): "is a bigger deep tier worth local silicon" — prove-before-buy.
Plus the ADR-001 cleanup still in flight (re-enroll voiceprint, eval-backed multi-intent redo,
ISSUE-004/006 pruning + honesty). Deferred doc merges queued.

## Hosts (consolidated to 2 active nodes 2026-09-28 — pve-01 retiring)
| Host | Role | Node | Mgmt |
|---|---|---|---|
| core-ai | Fabric services (2× RTX 5080) | pve-03 | systemd `iris-*.service` (interactive sudo) |
| audio-ai | Conditioning consumers | pve-03 | docker (matt in docker group, sudoless) |
| world-ai | Sensors, Meridian, HA proxy (**no GPU** — reflex retired) | pve-03 | systemd `world-ai*.service` |
| voice-ai | STT / TTS + GPU (RTX 5060 Ti) | **pve-02** (was pve-01) | — |
| infra-01 | Pi-hole DNS + chrony NTP | pve-03 (was pve-01) | LXC |
| monitor-01 | Loki / Grafana | pve-03 (was pve-01) | LXC |
| bastion-01 | Where Claude runs; iris-docs lives here | pve-03 (was pve-01) | — |
| memory-ai | pgvector / storage | pve-02 | — |

**pve-01: retiring** (only net-tools left, stopped). Physical: RAM → pve-02, 5060 Ti → spare, `pvecm delnode`.
Parked infra (bastion/infra-01/monitor-01) on pve-03 → rebalance to pve-02 after its RAM upgrade.
No shared storage; migrations are `qm/pct migrate` (ZFS send). See `design/HOST-CONSOLIDATION*.md`.

## Live fabric services (core-ai)
`iris-fabric.target` active. **cortex-creative RETIRED** (2026-09-28, authoritative — fusion ignores it).
The **fast-path reflex is DISABLED** (`CURATOR_FASTPATH_ENABLED=false`; world-ai GPU-less) — cortex-logical
is the **sole actuator** (silent on completed actuation, spoken correction on failure). Running: chunk-store,
cortex-logical, curator, dream-mode, episode-buffer, fusion-executive, memory-substrate (+http),
reflection-coordinator, thalamus, tool-registry, volition.

## Subsystem status (live-verified)
| Subsystem | Status | On hot path? | Verdict |
|---|---|---|---|
| Episodic memory (memory-substrate, :8013) | LIVE | YES (recall before cortex, write after) | KEEP |
| Working-memory field (thalamus) | LIVE (`THALAMUS_FIELD_ENABLED`) | YES (routing) | KEEP — best value/complexity |
| Prosodic affect (openSMILE) | **LIVE** (`PROSODY_OBSERVE_ONLY=false`, commit 3244916; `PROSODY_ENABLED=true`) | YES (shapes synthesis) | KEEP — watch "warm" gating |
| Speaker gate (ECAPA) | LIVE, `SPEAKER_THRESHOLD=0.35` | YES (drops ~22% of addressed turns) | TUNE — see ISSUE-001 |
| Volition | LIVE-r than expected: lighting/arrival-greet/events/briefing `DRY_RUN=false`; welcome-home `DRY_RUN=true` | NO (side-channel proactive) | KEEP + audit noise |
| Presence / census (world-ai radar) | Observe-only, read-only | NO | KEEP (cheap) |
| **Consolidation / dream-mode / reflection-coordinator** | dream-mode active but trigger-only (nightly, currently idle); `dream-mode.env` absent → shadow defaults; reflection-coordinator unit **disabled** (running, won't survive reboot) | **NO** — consolidated tier (:8013, 355 records) has **no working read endpoint** | **NUKE / hard-gate** — see ISSUE-003 |
| wake-arbiter (:8090) | Documented, **never wired** ("NOT implemented") | NO | DELETE — see ISSUE-004 |

## Known-broken / owed
- Origin pushes owed (automation can't push): `12561ec` world-ai `feat/presence-fusion`;
  `40a20f1` iris-fusion-fabric `feat/events-queue`; `261e4db` iris-conditioning `feat/prosodic-affect`.
- Basement lost its lux entity when the MFS prototype was swapped for radar — `/lighting/basement`
  is presence-only until a standalone lux sensor is added.

## Recently fixed + live-confirmed
- Doubled-domain actuation (`light.light.turn_on` → false "couldn't reach it") — `_bare_service()`, confirmed on a real basement-lamp turn 2026-08-10.
- Leading "Iris"→"Fires" homophone drop — symmetric rescue, live on audio-ai.
- Basement presence restored via `sensor.basement_occupancy_*` radar.
