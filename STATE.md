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
- **Voice-managed tasks LIVE** (add + read, grounded) — web-surface P1 first capability.
- **Creative cortex RETIRED 2026-08-25** (migration step 2, `FUSION_EXECUTIVE_ENABLE_CREATIVE=false`
  + cortex-creative stopped): logical owns the grounded voice → ISSUE-005 confabulation dead at the
  root ("weather report so much better", Matt). GPU1 freed for the deep MoE tier (step 4, next).
  Tradeoff: first-audio ~0.2s→~2.5-3s (no logical streaming yet — backlogged). Reversible via the flag.
- **Deep tier (qwen3-deep, 30B-A3B instruct) PROVEN LIVE 2026-08-25** — deliberates on grounded context, barges in over the front-door answer (~13s). GPU1. Still fires only on MANUAL punts (proactive punt signals = step 5). See changelog.
- **Shadow-punt measuring** (step 1). Surprise finding: no-streaming *feels faster* → perceived latency
  is VAD endpointing, not cortex first-audio (ISSUE-009).

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

## Hosts
| Host | Role | Mgmt |
|---|---|---|
| core-ai | Fabric services | systemd `iris-*.service` (interactive sudo) |
| audio-ai | Conditioning consumers | docker (matt in docker group, sudoless) |
| world-ai | Sensors, Meridian, HA proxy | systemd `world-ai*.service` |
| voice-ai (pve-01) | STT / TTS + GPU | — |
| bastion-01 | Where Claude runs; iris-docs lives here | — |

## Live fabric services (core-ai, verified 2026-08-22)
`iris-fabric.target` active. Running: chunk-store, cortex-creative, cortex-logical,
curator, dream-mode, episode-buffer, fusion-executive, memory-substrate (+http),
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
