# Backlog

Prioritized. Promote items to `issues/` once they're concrete enough to act on.

## Now (post first-soak, 2026-08-23 — see ADR-001 + issues)
1. **Re-enroll Matt's voiceprint** (ISSUE-001) — script ready at `audio-ai:/opt/iris-conditioning/scripts/reenroll_voiceprint.sh`; run when quiet. 0.28 helped but genuine turns still reject at cos ~0.26.
2. **Redo multi-intent with an eval harness** (ISSUE-002) — the dual-rail prompt over-prefetched live and was REVERTED. Also tune the working-memory field's prefetch stickiness (baseline phi3.5 over-prefetch predates the change).
3. **ISSUE-006** — say a backend errored (503) instead of vaguely deflecting.
4. Prune the confirmed-dead set (ISSUE-004); consolidation writer already gated (ISSUE-003).

## Owed / housekeeping
- Push owed commits from Matt's shell: `12561ec` (world-ai), `40a20f1` (fusion-fabric), `261e4db` (conditioning).
- Basement lux replacement decision — standalone lux sensor, or basement stays radar-only? (affects LightingArbiter basement offers)
- Task #31 (banked): arm the no-wake await window for proactive reply-inviting turns (e.g. "go for it" to a briefing offer). Verify `FOLLOWUP_WINDOW_S=6`.
- Task #29: fuse text valence (DistilBERT SST-2) into AffectInfo `source="fused"`.

## Watch (latent, not currently biting)
- `tool-registry control_device` schema advertises a `'data'` object but dispatch reads flat args (ARCHITECTURE.md:346). Actuation works today because the cortex sends flat args; would bite if a model followed the advertised schema. Reconcile the schema with dispatch.
- Confirm the two `probably-dead` rows before deleting: grep `fusion-executive/.../service.py` for httpx/`recall_precedent` call sites; grep tool-registry for `current_weather` dispatch. Now doable — SSH to core-ai is live again.

## Later / speculative
- Presence census go-live once dog-baseline calibration holds.
- Migrate remaining scattered canonical docs (EVENTS.md, PRESENCE_FUSION.md, per-repo STATE/BACKLOG) into this repo or link them. _(THESIS/PATTERNS/PRINCIPLES/legacy-DECISIONS merged 2026-08-23.)_
- Deferred merge: fold the legacy `Architecture/ARCHITECTURE.md` DDS **topic table + HTTP edge table + RTOS priorities + Mermaid diagrams** into a `reference/DDS-FABRIC.md` appendix (correct the retired router-TTS→speech-output leg; add vg-03 bedroom node). The tight hot-path narrative stays in `ARCHITECTURE.md`; this is the deep reference layer. Also port `Architecture/PROMPTS.md` rationale + the 7 regression test cases into a `PROMPTS.md` **re-derived from live `prompt.py` files** (the old bodies are stale). Preserve `Architecture/Core Substrate.md` as a clearly-labeled sync-era/roadmap history doc (the three-loop model + four-pass dream-mode spec = the consolidation roadmap ADR-001 protects).

---

## Merged from legacy `Architecture/` backlog (2026-08-23 review)
_Deduped against the issues + items above. Grouped by theme. The legacy files stay in
`/opt/repos/Architecture/` as source; only still-open items are carried here._

### Promote to ISSUE (bigger than a backlog line; not covered by ADR-001)
- **Path-2 actuation** (ambient/state-expression): "too dark", "make it cozy", "I'm winding down" → cortex reasons over sensor state + time + occupancy and emits multi-action plans. Distinct from the done Path-1 direct-imperative fast-path. A real capability gap and a THESIS frontier.
- **Time-anchored intentions / reminders scheduler**: "remind me at X / when Y." The Events subsystem is the substrate; no user-facing scheduled-intention path yet.
- **Right-to-forget / memory purge**: "forget that." Grep confirms zero implementation. A privacy primitive against the LIVE episodic tier — independent of the gated substrate work, so pull it out so it actually gets built.
- **Contention-surfacing as offered alternative**: "Turning on the kitchen lights. If you want softer, say so." THESIS keystone (the weaver rendering two perspectives); a lighter version could ride the existing dual-cortex output without full substrate. Decide rather than silently bury.
- **Cognitive event stream** (`emit_cognitive_event`, Loki `event_class=cognitive`): designed, never built; partly overlapped by the turn-tracing dashboard. Keep-or-kill decision now that consolidation is gated.

### Cognition / correctness
- **RESOLVED 2026-08-23 (Matt):** the Davis soft/grounded anchor is INTENTIONAL — "Davis references are fine if grounded." The live curator `IDENTITY_ANCHOR` ("mention him only when he's directly relevant") is correct; do NOT restore the legacy hard "never mention Davis" rule. The only failure mode to watch is UNGROUNDED Davis drops (punchline reflex), which is the ISSUE-005 grounding family, not a separate identity rule.
- `entity_id` normalization guard — cortex-logical ~5% drops a compound-command entity_id missing its domain prefix (`dining_room_lights` vs `light.dining_room_lights`). Resolve canonical form from entity_mappings before building the ToolCall.
- Iris self-maintains entity mappings — every "the dining lights" clarification is free training data for the mapping table; never built. Cheap, compounding; attacks the entity_id pain above.

### Calendar correctness (Google Workspace otherwise done)
- Multi-day `status=free` must read as *presence* ("Davis with me"), not *unavailable*, so availability math isn't blocked.
- Custody schedule representation (recurring multi-day blocks, probably its own calendar). Family-specific, deferred.

### Integration (email-intake substrate is live)
- Package tracking via forwarded order confirmations to iris@ (regex → 17track).
- Drive shared workspace (`Iris/` read + journal writes) — Phase-5 precursor toward custody-agreement processing.
- Air quality integration is broken (OpenAQ 401) — drop or replace the provider.

### Sensing / home-automation
- HA WebSocket subscription instead of 30s polling (catches rapid button presses the poll misses).
- Occupancy-driven display sleep for the ui-01 basement touchscreen (V1 via HA automation).

### Ops / infra hygiene
- Propagate Docker `json-file` log-rotation to voice-ai/core-ai/world-ai/memory-ai/pve (only audio-ai has it). See [[PATTERNS]].
- Cross-host contracts-version propagation automation (a bump has broken pinned consumers; private index or bind-mount source).
- `bus.run_forever` absorbs SIGTERM/SIGINT → systemd stop waits out the timeout then SIGKILLs. One-line fix, every restart.
- Music Assistant entity rename cleanup, then delete the `SONOS_TO_MA_ENTITY` map in world-ai.
- File the HA Sonos `get_media` IndexError upstream (worked around via MA).

### Housekeeping (fold into ISSUE-004 dead-code family)
- `fusion-executive/curator.py` unused `curate()` helper — confirm dead + delete (active producer is `curator/service.py`).
- `cortex-logical/README.md` still documents the deleted HTTP shell.
- Centralize the three-way `identity_anchor` (canonical in curator, copy in reflection-coordinator, drifted in fusion — ISSUE-004 tracks the drifted one; this is the broader centralize task).
- Extract a shared `CortexWorker` module (both cortices duplicate worker + wakeup-channel fix).
- Stale `# Gibsonia, PA` comment in `world-ai/app/config.py` (coords already correct) — trivial.

### Observability
- Observability catalog — one doc: every signal, where it lives, how to query it.
- Loki per-stream retention (operational 30-90d vs cognitive/consolidation source longer).

### On-ice — do NOT re-file as active (gated by ADR-001)
The whole consolidation/reflection/substrate roadmap (dream-mode, neutral semantic store, two-track
LTM writers, Hopfield substrate research, affect-delta cross-turn tagging, persona modes, Amygdala
v0, real blend synthesis). Writer stopped, code preserved. Prosodic affect itself IS live; only the
cross-turn delta that fed consolidation is on ice. Revisit only when the read-back is deliberately wired.
