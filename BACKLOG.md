# Backlog

Prioritized. Promote items to `issues/` once they're concrete enough to act on.

## Now (from the 2026-08-22 review — see ADR-001)
1. Lower `SPEAKER_THRESHOLD` 0.35 → ~0.28 and measure recall (ISSUE-001). Biggest felt win, lowest risk.
2. Dual-rail routing so actuate+query compounds both fire (ISSUE-002).
3. Prune dead code: consolidation cluster, wake-arbiter, duplicate VAD, reflection-coordinator (ISSUE-003/004).

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
- Migrate scattered canonical docs (THESIS.md, EVENTS.md, PRESENCE_FUSION.md, per-repo STATE/BACKLOG) into this repo or link them from here.
