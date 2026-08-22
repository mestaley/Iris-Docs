# ISSUE-004: Dead / vestigial code — wake-arbiter unwired, reflection-coordinator disabled
- Status: open
- Severity: low
- Area: infra / diagnosis-surface
- Discovered: 2026-08-22
- Evidence: `ARCHITECTURE.md:346` ("wake-arbiter → STT handoff NOT implemented"); no consumer of `:8090` in iris-conditioning/iris-audio-fabric; `iris-reflection-coordinator` UnitFileState=disabled

## Why this matters
Dead paths inflate the "diagnosis surface" — when something breaks, they're extra suspects that can't actually be the cause. Removing them is pure clarity gain.

## Items
1. **wake-arbiter (:8090)** — referenced in docs but never wired for address gating; the real address decision is entirely `intent_consumer.score_transcript` + the speaker gate. The `keep_listening` field fusion sets is never consumed (STATE/BACKLOG notes), which is why follow-ups rely on the `.await` file-stamp window instead of a hot mic. DELETE the service + doc references, or actually wire `keep_listening` if the hot-mic follow-up is wanted.
2. **reflection-coordinator** — running but unit `disabled` (won't survive reboot). Decide with ISSUE-003: if the consolidation cluster is nuked, this goes with it.
3. **Duplicate VAD** — standalone `vad_consumer.py` duplicates the Silero VAD that asr-consumer already runs internally (`asr_consumer.py:137-148`). Removing it changes nothing about recall/precision.

## Note
Confirm each against live runtime before deleting (some are needs-runtime-check, per the pipeline/dead-code map — pending agent 4).
