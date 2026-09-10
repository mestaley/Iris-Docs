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

---
## Fix: early-stamp race (2026-09-10)
Re-test showed Phase 1 fusion WORKED (Iris asked "which lights?" → fusion emitted `await-marker →
vg-02`), but the window opened ~10s early: speech-output stamped `<node>.await` at question-START
(16:13:33) not question-END (16:13:43), because the await marker races the reply utterance
(different DDS topic) and the node still looked idle. Fix: speech-output always ARMS and stamps on
the coming idle, never immediately (fusion always voices a say on an invite_reply turn). Deployed to
audio-ai (`speech-output` restarted). Now the 18s window opens when Iris finishes asking.
(Also observed: no ASR captured for the answer in that shortened window — re-test answering promptly.)

---
## ROOT CAUSE: clock skew (2026-09-10) — three bugs on the path
Reactive no-wake answering was broken by three stacked bugs, found by tracing three failed re-tests:
1. **Early-stamp race** (speech-output): the await marker raced the reply on a different DDS topic, so
   `<node>.await` stamped at question-START not END, burning ~10s of the window. Fixed: always arm,
   stamp on the coming idle. LIVE.
2. **needs_clarification inconsistent** (fusion): cortex-logical set the flag for "which lights?" but
   not "how would you like them set?" → no arm. Fixed: `_should_invite_reply` also arms when the say
   is a question to the user AND no action was taken. STAGED (needs iris-fusion-executive restart).
3. **Clock skew — THE blocker** (intent-consumer): the window compared the wall-clock `.await`/`.idle`
   stamps against the utterance's `timestamp_us` = the NODE's clock, which ran ~90 min behind on vg-02.
   `await_us <= ts_us` was `16:57 <= 15:27` → always false; the window could NEVER match (broke await,
   followup, AND conversation mode). Fixed: time all windows in wall-clock (`produced_at_us`, fallback
   now). LIVE. Commit on iris-conditioning.

## Latent infra issue surfaced
**vg-02's clock is ~90 min off wall-clock** — a voice-node NTP problem. The wall-clock window fix makes
the no-wake windows robust to it, but the skew likely affects other node-timestamp-based logic and
should be corrected (NTP on the Pi voice nodes). New backlog item.

## Deploy state
- speech-output (stamp) + intent-consumer (wall-clock) fixes: LIVE on audio-ai.
- fusion ("?"-fallback arming): STAGED — **restart iris-fusion-executive**, then re-test.
