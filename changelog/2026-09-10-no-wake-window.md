# 2026-09-10 — No-wake follow-up window: resurrect + arm for offers + conversation mode (ISSUE-012)

## Executive summary
Dug into the no-wake-word follow-up mechanism (agent-mapped end-to-end) and found it had been DEAD
since ~Aug 13 — armed only by `creative.needs_clarification`, orphaned when creative retired (a
second casualty like the proactive rail). Built Phase 1 (fusion: resurrect + arm for front-door
questions AND proactive offers) and Phase 2 (conditioning: gated open-ended conversation window).
Filed ISSUE-012 with the full plan incl. Phase 3 (offer-before-deliver, the recall-queue slice).

## Changes by repo
- **iris-fusion-fabric `feat/punt-escalation`** — `fusion-executive/.../service.py`:
  `_should_invite_reply(logical, origin, proactive_kind)` arms the await marker + `keep_listening`
  on a front-door question (`logical.needs_clarification`) or a proactive offer (`proactive_kind==
  "offer"`); dropped the empty `streaming_speech_nodes` guard; reads `proactive_kind` off the trace.
  +4 tests (suite 50). **Owed: restart iris-fusion-executive** — restores reactive no-wake answering.
- **iris-conditioning** — `consumers/intent/app/intent_consumer.py`: `CONVERSATION_MODE` (gated OFF)
  rolling window — a session opened by a verified user turn admits back-and-forth without a wake word,
  anchored purely on the user's turns (Iris's speech can't sustain it), closes on user silence;
  speaker gate still rejects a confidently-different speaker. **Deployed inert to audio-ai** (bind-mount
  + `docker restart intent-consumer`, clean). Flip `CONVERSATION_MODE=true` to try.
- **iris-docs** — `issues/ISSUE-012-*.md` (new).

## Deploy / live state
- Phase 1 (fusion): STAGED — needs `sudo systemctl restart iris-fusion-executive`.
- Phase 2 (conditioning): LIVE but INERT (CONVERSATION_MODE=false). Flip the env + restart the
  container to enable open chat.

## Follow-ups
- After the fusion restart: reactive no-wake answering works again (Iris asks → you answer, no "Iris").
- Phase 3 = offer-before-deliver (recall-queue): deep-mode emits an offer, holds the payload, confirm
  → deliver. Depends on the recall-queue store.
- Phase-2 refinements: half-duplex barge-in (ISSUE-009), long-session re-verification.
- Origin pushes owed (now incl. iris-conditioning).
