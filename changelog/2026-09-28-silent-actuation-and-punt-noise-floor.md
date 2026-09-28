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

## Behavior change to watch
- Genuine reflex-failure correction now depends entirely on the **async supervisor** (cortex-logical
  judge, which receives the failed result in the SuperviseRequest) — there's no deterministic
  synchronous net anymore. If a real failure ever goes uncorrected/silent, that's a supervisor-prompt
  reliability issue (cortex-logical), not fusion.

## Follow-ups / owed
- Restart above.
- ISSUE-013 status: **A (latency) — resolved by silent-actuation** (no speech = nothing to wait on);
  **C (deep-punt noise) — done**; **B (ASR min-content guard) — dropped as unsafe** (the noise rides
  the follow-up/await windows where short answers are legitimate, so a blanket drop would kill real
  follow-ups); **D (backyard-light entity mapping) — still open** (spot-check).
- Origin pushes still owed (automation can't push): fusion `feat/punt-escalation`, world-ai
  `fix/actuation-settle-verify`, iris-docs. `/opt/repos/_push-staging/` has bundles + push-all.sh.
