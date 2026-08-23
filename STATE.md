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
