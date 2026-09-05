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

---
## Validation + deferred timing (2026-09-06)
- **Rail fix VALIDATED LIVE**: published a canned proactive brief to a non-speaker node — fusion
  fired immediately and voiced it **verbatim** (say == brief, strategy=logical_only_called). ISSUE-011
  confirmed fixed.
- **Events delivery is LIVE** — `/etc/iris-fabric/volition.env` already has `VOLITION_EVENTS_DRY_RUN=false`
  (independent of the master `VOLITION_DRY_RUN=true`). So the only remaining gate was delivery *timing*.
- **Deferred timing fixed** (deep-mode `feat/deep-mode`): the deliberation event was `holdable=True`,
  which the arbiter BATCHES into the morning/arrival digest — wrong for "been chewing on that…". Now
  non-holdable (its own turn at the next present tick) + `fire_at = now + 60s` so it lands a little
  later, not instantly. Needs `sudo systemctl restart iris-deep-mode`.

## Full deferred loop — now wired + live-ready
hard turn → deep-mode queues a deliberation (due +60s, non-holdable) → arbiter delivers at the next
present tick → **fusion voices the brief verbatim** (validated). Restart iris-deep-mode, then a live
end-to-end test WILL voice in the house ~1 min after a hard question.
