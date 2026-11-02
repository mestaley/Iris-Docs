# ISSUE-017: "Iris, STOP" — a verbal hard-abort of the current turn (keep_listening = NO)

- Status: open
- Severity: medium
- Area: conditioning (intent/wake) + fabric (fusion-executive turn control) + speech-output (playback halt)
- Discovered: 2026-11-02 (Matt)

## Summary
When Iris starts rambling — a long answer, a false trigger off the TV, the wrong reply — Matt wants a
spoken **"Iris, stop"** that **abruptly ends the turn**: cut TTS playback immediately, cancel/abandon any
in-flight cognition for that turn, and **do not await a follow-up** (`keep_listening = NO`, close the
conversation window). A hard kill switch, distinct from the normal flow.

## What exists vs what's missing
- **Acoustic barge-in exists**: speaking over Iris interrupts playback (streaming speech rewrite —
  confirmed both gateways). But that's "start a new turn by talking," not an explicit abort.
- **Missing**: a recognized **STOP command** that (a) halts current speech, (b) does NOT start a new
  cognitive turn / does NOT leave the mic open awaiting an answer, (c) explicitly sets the turn's
  continuation to NO so Iris goes quiet rather than listening for more.

## Desired behavior
- Phrases: "Iris, stop" / "stop" / "nevermind" / "quiet" (addressed) → immediate halt.
- Fast path — recognized as early as possible (ideally at the conditioning/intent layer, like an address
  gate), not routed through full cognition, so the stop is near-instant.
- Effects: flush/stop speech-output for the target node; cancel the in-flight turn (if any) so a delayed
  reply doesn't resume talking after the stop; set `keep_listening = NO` / close the await/followup/
  conversation window so the turn ends cleanly and silently.
- Idempotent/safe when nothing is playing (just ensures silence + closed window).

## Notes / unknowns (to resolve when picked up)
- Where to intercept: a dedicated reflex in `iris-conditioning` (intent-consumer) vs a fabric-level
  control message. Earliest wins for latency.
- Needs a speech-output "halt now + drop queued clauses for node" path, and a fusion/turn "abandon turn
  <id>" signal.
- Interaction with the (gated-off) CONVERSATION_MODE and the await/followup windows — STOP must force all
  of them closed.
- Careful not to swallow a legitimate "stop" that's part of a command ("stop the kitchen timer") — STOP
  as abort should be the bare/addressed form.

Filed 2026-11-02 (Matt) to track; not scheduled.
