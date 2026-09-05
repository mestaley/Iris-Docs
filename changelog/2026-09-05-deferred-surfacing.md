# 2026-09-05 — ADR-004 deferred surfacing (deep-mode side) + a rail finding

## Executive summary
Started ADR-004's deferred-surfacing increment: deep-mode now hands its deliberation to Volition's
events channel for **deferred proactive delivery** ("been chewing on that…") instead of a same-turn
`SpeechUtterance` barge-in over the answer Matt already heard. The deep-mode POST side is done +
tested (11 green). Discovered on the way that the whole **proactive delivery rail is broken with
creative retired** (fusion waits for creative on system turns; the brief only ever reached
cortex-creative) — filed ISSUE-011. That blocks the *voicing* half and also silently degrades
welcome-home/briefing.

## Changes by repo
- **iris-fusion-fabric `feat/deep-mode`** — `deep-mode/.../service.py`: `_deliberate_and_deliver`
  routes to `_deliver_deferred` (POST a `deliberation` PendingEvent to `:8015/events`, priority low,
  holdable, TTL) instead of the barge-in when `deferred_delivery` (default True); deep prompt gains a
  deferred follow-up framing. `config.py`: `deferred_delivery`, `intake_url`, `intake_timeout_s`,
  `deliberation_ttl_s`. +1 test.
- **iris-docs** — `issues/ISSUE-011-proactive-rail-creative-retired.md` (new).

## Deploy / live state
- **STAGED:** deep-mode synced to core-ai. Restarting iris-deep-mode is a NET WIN even before the
  rail fix — it stops the jarring same-turn barge-in; deliberations queue (silently) until ISSUE-011
  voices them. **Owed:** `sudo systemctl restart iris-deep-mode` (optional now / required for the flow).
- Deferred delivery won't VOICE until: (a) ISSUE-011 (proactive rail with creative retired), and
  (b) `VOLITION_EVENTS_DRY_RUN=false`.

## Follow-ups
- **ISSUE-011** — repair the proactive rail (recommend: fusion voices `proactive_brief` verbatim for
  system turns). This is the next increment; it also restores welcome-home/briefing.
- Then flip `VOLITION_EVENTS_DRY_RUN=false` and validate end-to-end (deliberation surfaces later).
- Origin pushes owed.
