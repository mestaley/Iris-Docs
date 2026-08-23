# ISSUE-004: Dead / vestigial code — wake-arbiter unwired, reflection-coordinator disabled
- Status: PARTIAL 2026-08-23. DONE: dead Spotify cred fields removed (world-ai commit 68369bf). CORRECTED: `recall_precedent` is NOT dead-to-delete — it's the read-side scaffolding for the roadmap's "close the read-back" work; per ADR-001 (keep consolidation code, gate the writer) it STAYS. STAGED (need approval + deploy, higher blast-radius than "dead code" implied — see below).
- Severity: low

## Implementation notes (2026-08-23)
Verification changed the picture — most removals are not isolated:
- **DONE** — Spotify cred fields (isolated, zero reads, config-only).
- **KEEP** — `recall_precedent` (roadmap scaffolding; ADR-001 keeps consolidation code).
- **STAGED, needs a dedicated change + Matt's deploy:**
  - **wake-arbiter service** (:8090) — whole component + docker-compose + any systemd; confirmed unwired, but removal is a deploy change.
  - **Duplicate `vad_consumer.py` / vad-consumer container** — confirm it's not a running dependency, then drop from compose.
  - **Dormant `fabric_brain` bridge + WAV `/play`** — retire the fallback once no node can un-flip; touches the router.
  - **Legacy `app.py` shells** (fusion-executive/tool-registry) — entangled with their own `test_app.py` (tool-registry's holds the `_bare_service` regression test); delete file+test together.
  - **Dead bus topics** (`PERCEPTION_AFFECT_BURST`, `AFFECT_INTERRUPT`, `SESSION_STATE_UPDATE`) — exported from the shared `bus` package; verify no cross-repo importer first.
  - **`keep_listening`** — a FROZEN-contract field woven through contracts/fusion/curator/cortex-logical/eval/tests. Decision needed: WIRE it (hot-mic follow-up) or remove it (contract change). Not a delete.
Each of these deserves its own reviewed branch, not a blind sweep.
- Area: infra / diagnosis-surface
- Discovered: 2026-08-22
- Evidence: `ARCHITECTURE.md:346` ("wake-arbiter → STT handoff NOT implemented"); no consumer of `:8090` in iris-conditioning/iris-audio-fabric; `iris-reflection-coordinator` UnitFileState=disabled

## Why this matters
Dead paths inflate the "diagnosis surface" — when something breaks, they're extra suspects that can't actually be the cause. Removing them is pure clarity gain.

## Items
1. **wake-arbiter (:8090)** — referenced in docs but never wired for address gating; the real address decision is entirely `intent_consumer.score_transcript` + the speaker gate. The `keep_listening` field fusion sets is never consumed (STATE/BACKLOG notes), which is why follow-ups rely on the `.await` file-stamp window instead of a hot mic. DELETE the service + doc references, or actually wire `keep_listening` if the hot-mic follow-up is wanted.
2. **reflection-coordinator** — running but unit `disabled` (won't survive reboot). Decide with ISSUE-003: if the consolidation cluster is nuked, this goes with it.
3. **Duplicate VAD** — standalone `vad_consumer.py` duplicates the Silero VAD that asr-consumer already runs internally (`asr_consumer.py:137-148`). Removing it changes nothing about recall/precision.

## Full dead-code list (pipeline map, 2026-08-22)
| Path | Evidence | Confidence |
|---|---|---|
| wake-arbiter → STT handoff (:8090) | `wake-arbiter/app/main.py:58-115` records to SQLite, never forwards to STT; no `/wake` caller anywhere | confirmed-dead |
| Synchronous `fabric_brain` bridge (call_brain + 15s wait + WAV /play) | `router_consumer.py:230-241` takes fire-and-forget; `:245-304` marked "Dormant… kept as fallback"; all 3 nodes flipped to streaming | confirmed-dead on live path (dormant fallback retained) |
| Legacy FastAPI `app.py` shells — fusion-executive :8010, tool-registry :8012 | units run `-m *.service` (DDS), never `app`; arch doc marks all `(L)` dead-in-prod | confirmed-dead in prod |
| cortex-logical :8011 / cortex-creative :8014 "shells" | no `app.py` exists; `host/port` is vestigial config with no listener bound | confirmed-dead (dead config) |
| `keep_listening` field | computed in fusion, plumbed to `router_consumer.py:255-304`, "only ever logged, never wired to mic re-arm" | confirmed-dead (consumer absent) |
| fusion-executive httpx clients + `recall_precedent` | LEGACY v1, not invoked by `service.py`; `recall_precedent` zero call sites | probably-dead |
| tool-registry `WorldAiBackend.current_weather` + `ha_url`/`ha_token` config | defined, never dispatched (weather → Meridian); proxy_url hardcoded | probably-dead |
| memory-substrate Qdrant backend | `service.py:20,103` in-memory dict; Qdrant never written | live-but-stub (storage vestigial) |
| Spotify creds in world-ai `config.py` | dead after Music Assistant refactor; TODO to delete | confirmed-dead |
| fusion-executive `config.py` identity_anchor (drifted copy) | dead unless fusion HTTP path activates (it doesn't) | probably-dead |
| Bus topics w/ no producer/consumer — `PERCEPTION_AFFECT_BURST`, `AFFECT_INTERRUPT`, `SESSION_STATE_UPDATE` | catalog-only "future"; `DREAM_REPORT` removed in bus 0.3.0 | confirmed-dead (declared, unwired) |
| Duplicate standalone `vad_consumer.py` | asr-consumer runs its own Silero VAD (`asr_consumer.py:137-148`) | confirmed-dead (redundant) |

Cutting the confirmed-dead set removes **zero** live-turn capability and roughly halves the "no-response" diagnosis surface (12 → ~6 suspects). Verify the two `probably-dead` httpx/weather rows with one grep of `service.py` call sites before deleting.
