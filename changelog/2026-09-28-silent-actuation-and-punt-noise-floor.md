# 2026-09-28 — Silent-on-clean-actuation + affect-punt noise floor (ISSUE-013 A & C)

## Executive summary
Tackled the top ISSUE-013 items. The big one is architectural, from Matt's own reframing: the
fast path actuates and the **async supervisor is the sole correctness voice** (silent when the
reflex was right, a spoken correction when wrong) — but fusion's *synchronous* path was also
voicing the outcome ("Done." + the false "can't reach it"), pre-empting the supervisor and being
chatty. Pulled the synchronous voicing: a clean reflex actuation now emits **no speech at all**
(the action is the feedback), which both **dissolves the perceived-latency problem** for the common
case and **deletes the got-away / can't-reach-it / "Done." bug class at the root**. Also added a
content floor so degenerate ASR ("six"/"yeah"/mumbles) no longer gets deep-punted. Both committed on
fusion `feat/punt-escalation`, **staged pending `sudo systemctl restart iris-fusion-executive`.**
This supersedes the earlier synchronous-voicing fixes (the "Done." ack + got-away tuning from
2026-09-10/11) — they were treating symptoms of the two-voices confusion.

## Changes by repo

### iris-fusion-fabric — branch `feat/punt-escalation`
- `fusion-executive/src/fusion_executive/service.py` — **changed** (silent actuation, commit after
  the noise-floor one): replaced `_should_confirm_actuation` (voiced "Done.") with
  `_silent_reflex_actuation(reflex_owned, strategy)`; `_fire` now stays silent (emits no utterance,
  still publishes the FusionDecision for memory) on a reflex-owned + `neither_confident` turn, and
  the synchronous failed-actuation correction is gated to NON-reflex actuations only (reflex failures
  defer to the supervisor). Emit block gated on a new `speak` flag.
- `fusion-executive/src/fusion_executive/service.py` — **changed** (affect-punt floor): added
  `_PUNT_MIN_WORDS=4`; `_proactive_punt_reason` requires ≥4 content words before an affect-driven deep
  punt, so prosody-tagged 1-2 word noise ("six"/"yeah") isn't escalated to the deep tier.
- `fusion-executive/tests/test_proactive_punt.py` — **changed**: replaced the `_should_confirm_actuation`
  tests with `_silent_reflex_actuation` tests; added `test_degenerate_input_does_NOT_punt_on_affect`.
- `fusion-executive/tests/test_reflex_dedup.py` — **changed**: `test_reflex_failure_still_voices_correction`
  → `test_reflex_failure_defers_to_supervisor_silently` (asserts no synchronous correction utterance +
  a SuperviseRequest is published). 55 tests green.

## Deploy / live state
- **STAGED (pending restart, Matt sudo):** `sudo systemctl restart iris-fusion-executive` activates
  both changes. Until then, prod still chirps "Done." and can deep-punt noise.
- Access note: the SSH agent had dropped over the multi-day gap (passphrase-encrypted internal key,
  no agent) — Matt reloaded it (`ssh-add`), connectivity restored. No code impact.

## MAJOR discovery: creative was never actually retired (fixed 2026-09-28)
Tracing the first light commands after the restart showed **every turn `creative_dominant`,
`creative_conf=0.95`** — cortex-creative is alive and voicing ("Right you are. Getting the basement
lights."), despite ISSUE-011/012 + memory claiming it retired Aug 25. Root cause: the Aug-25
"retirement" only set `ENABLE_CREATIVE=false`, which gates `ready_to_fire` but **not synthesis** —
the service kept running, its response still arrived, and `_fire`/`_synthesize` still picked it
(creative voice bias). So retirement never took: creative was the live voice on every turn (the
persistent "leads with Right" tic + actuation chatter), and the silent-actuation change above was
**inert** (turns were never `neither_confident`).
- `fusion-executive/src/fusion_executive/service.py` — **changed**: `_fire` now ignores any creative
  response when `ENABLE_CREATIVE` is off, so a still-running / accidentally-restarted creative can't
  win the voice via synthesis. Retirement is now authoritative from fusion's side. +1 test.
- **Action for Matt:** `sudo systemctl restart iris-fusion-executive` activates the fix (creative
  ignored → logical is the voice → clean actuations go silent). Then, to actually free GPU1 for the
  deep tier: `sudo systemctl stop iris-cortex-creative && sudo systemctl disable iris-cortex-creative`
  (the code fix means the voice is fixed even before this; stopping it is for the GPU/deep-tier).
- Docs corrected: ISSUE-011 (see its 2026-09-28 note); memory updated.

## Behavior change to watch
- Genuine reflex-failure correction now depends entirely on the **async supervisor** (cortex-logical
  judge, which receives the failed result in the SuperviseRequest) — there's no deterministic
  synchronous net anymore. If a real failure ever goes uncorrected/silent, that's a supervisor-prompt
  reliability issue (cortex-logical), not fusion.

## Post-restart trace (validated) + light-status hallucination
After the fusion restart + `stop iris-cortex-creative`: **creative is inactive**, and pure commands
("basement lights off", "to 25%") now go `neither_confident` → **`silent (reflex actuation, supervisor
owns correctness)`** with the supervisor logging `reflex CORRECT`. The model is live and working.

Tracing status *questions* surfaced a two-bug hallucination (turn-49f10037 "are the kitchen lights on
or not"): (1) the reflex fired `control_device(turn_on)` — **asking toggled the lights on**; (2) logical
answered "The kitchen lights are off" from nothing — its `get_entity_state` call returned AFTER it had
already committed (cortex query-tool results are dropped) and ambient had NO light state.
- `curator/src/curator/fastpath.py` — **changed** (Fix A): `_is_status_query()` pre-gate; `reflex_actuate`
  skips (no model call, no fire) when the text asks ABOUT state, so asking never toggles. Word-boundary
  command detection so "turned on" ≠ the "turn" command verb; polite "can you turn…?" still actuates.
- `curator/tests/test_fastpath.py` — **changed**: +2 tests; un-staled the actuation-tools set assertion
  (flagged: reflex is offered set_reminder/mute — a mis-route risk, review later). **Needs iris-curator restart.**

**Fix B (light-status grounding) — deferred to next increment, per Matt's principle:** *"the only
prefetched data should be external APIs, and only slowly-changing things like weather."* So the fix is
NOT to prefetch device state — it's to make cortex query-tool results **flow back** (a synthesis pass
that waits for + incorporates `get_entity_state`) so logical answers fast state from live truth, and to
STRIP any fast-state prefetch that exists. Recorded as memory `iris-prefetch-only-slow-external`. This
is a real cortex-logical change (the query-loop), scoped as the next step.

## Follow-ups / owed
- Restart above.
- ISSUE-013 status: **A (latency) — resolved by silent-actuation** (no speech = nothing to wait on);
  **C (deep-punt noise) — done**; **B (ASR min-content guard) — dropped as unsafe** (the noise rides
  the follow-up/await windows where short answers are legitimate, so a blanket drop would kill real
  follow-ups); **D (backyard-light entity mapping) — still open** (spot-check).
- Origin pushes still owed (automation can't push): fusion `feat/punt-escalation`, world-ai
  `fix/actuation-settle-verify`, iris-docs. `/opt/repos/_push-staging/` has bundles + push-all.sh.
