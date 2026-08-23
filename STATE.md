# Iris — Current State

_Last verified: 2026-08-22 (live SSH sweep of core-ai / audio-ai / world-ai / voice-ai)._

## Current focus
Whole-system review: **"continue down this path, or nuke half of it?"** — **DECIDED, see ADR-001
(accepted 2026-08-23).** Verdict: keep the streaming trunk, retune the gate, close two capability
gaps, prune the vestigia. All four investigations (recall/gates, multi-intent, live-subsystem
harm, pipeline/dead-code) landed 2026-08-22/23. The felt regression traced to four things, none
of them "the architecture is too complex": recall tuning (ISSUE-001), a routing pre-classifier
eating multi-intent (ISSUE-002), ungrounded outcome claims (ISSUE-005), and a diagnosis surface
wider than the fault surface (ISSUE-004). Next action = lower `SPEAKER_THRESHOLD` and measure.

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
