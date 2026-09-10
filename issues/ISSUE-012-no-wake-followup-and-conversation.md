# ISSUE-012: No-wake follow-up window — resurrect, arm for offers, and open-ended conversation

- Status: **Phase 1 + Phase 2 built 2026-09-10** (staged/gated); Phase 3 (offer-before-deliver) is
  the recall-queue slice, not yet built. Prereq for [design/RECALL-QUEUE.md](../design/RECALL-QUEUE.md).
- Area: cognition — fusion (arming) + conditioning intent-consumer (the window) + speech-output (stamp)

## Background (from the 2026-09-06 mechanism map)
The no-wake-word window is a file-stamp handshake: fusion emits an await marker on
`iris/speech/await` → speech-output stamps `<node>.await` on idle → intent-consumer admits a no-wake
turn inside the window. Two windows existed: AWAIT (18s, "Iris asked a question") and FOLLOWUP (6s,
"one quick add-on"), both single-shot, neither re-arms. In-window turns are speaker-gated leniently
(`GATE_TRUSTS_WINDOW`: trust short/unverifiable audio, reject only a confidently-different enrolled
speaker — the ISSUE-001 mitigation).

**The regression:** the AWAIT window was armed ONLY by `creative.needs_clarification`; creative was
retired 2026-08-25, so the whole no-wake follow-up went DEAD (`.await` stamps stale since ~Aug 13) —
a second creative-retirement casualty alongside the proactive rail (ISSUE-011). A second guard
required the node in `streaming_speech_nodes` (empty in prod), a belt-and-suspenders reason it stayed
dead. `FusionDecision.keep_listening` was vestigial (nothing acted on it).

## Phase 1 — resurrect + arm for questions AND offers ✅ (built, fusion `feat/punt-escalation`)
`_should_invite_reply(logical, origin, proactive_kind)`: arm the await marker (and set
`keep_listening`) when the FRONT DOOR asked a genuine question (`logical.needs_clarification`, which
cortex-logical already emits) OR a proactive turn is a reply-inviting OFFER (`proactive_kind ==
"offer"`). Dropped the `streaming_speech_nodes` guard. Reads `proactive_kind` off the trace. +4 tests.
**Owed:** `sudo systemctl restart iris-fusion-executive` — restores reactive no-wake answering.

## Phase 2 — open-ended conversation (rolling window) ✅ (built GATED, conditioning intent-consumer)
A third window for back-and-forth without a wake word each turn. A session is **opened only by a
verified user turn**; admits continued turns while `ts <= last_user_turn + CONVERSATION_IDLE_S`
(default 45s), **anchored purely on the user's turns** — Iris's own/proactive speech never extends it,
so it can't self-sustain (the TV can neither open nor keep a session). Speaker gate still rejects a
confidently-different enrolled speaker in-window. Closes on user silence. **Gated OFF**
(`CONVERSATION_MODE`); deployed inert to audio-ai. Flip `CONVERSATION_MODE=true` (intent-consumer
env) to try it. Residual risk (documented): ambient within an active session could be admitted —
mitigated by the verified-open + different-speaker-reject backstop; that's why it's gated + user-anchored.

Open Phase-2 refinements (deferred): half-duplex/barge-in (interrupt Iris mid-reply — ties to
ISSUE-009 VAD); periodic re-verification on a longer burst for very long sessions.

## Phase 3 — offer-before-deliver (the recall-queue slice, NOT built)
Make deep-mode emit a short OFFER ("I've got a deeper read on that — want it?", `proactive_kind=
"offer"`) instead of the full analysis; HOLD the analysis keyed by turn_id in the recall queue;
when the confirming turn arrives in the (Phase-1-armed) await window, deliver the held payload. This
is the recall-queue confirm-resolution — see design/RECALL-QUEUE.md. Depends on Phase 1 (offer arms
the window) + the recall-queue payload store.
