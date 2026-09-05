# 2026-09-06 — Repair the proactive rail (ISSUE-011)

## Executive summary
Fixed the proactive delivery rail that creative-retirement broke (ISSUE-011): welcome-home,
briefing, and the ADR-004 deferred deliberation had no voice because the rail routed the composed
brief only through cortex-creative. Fusion now voices `proactive_brief` **verbatim** for
system-originated turns — reading it off the FeatureStream trace it already processes, firing as
soon as the brief is present, and bypassing synthesis so a considered deliberation isn't paraphrased.

## Changes by repo — iris-fusion-fabric `feat/punt-escalation`
- `fusion-executive/.../service.py`: `_TurnState.proactive_brief`; `on_feature_stream` grabs it off
  `fs.trace`; `ready_to_fire` fires a system turn on the brief alone; `_fire` voices it verbatim
  (skips the no-response return + the synthesis paraphrase); loop-guard (never punt a system turn).
- `tests/test_retire_creative.py`: +2 (ready_to_fire on brief w/o creative; not-ready without brief).
  Fusion suite 43 green.

## Deploy / live state
- STAGED. **Owed:** `sudo systemctl restart iris-fusion-executive` — then EVERY proactive turn voices
  its brief (welcome-home, briefing, deferred deliberation), not the 25s fallback.
- For deferred deliberations to actually DELIVER: also `sudo systemctl restart iris-deep-mode` (the
  deferred POST side) and set `VOLITION_EVENTS_DRY_RUN=false` (events delivery is dry-run).

## Follow-ups
- Validate the rail live (publish a canned proactive brief → fusion voices it verbatim).
- Then the full deferred loop: hard turn → deep queues deliberation → delivered later, voiced.
- Origin pushes owed.
