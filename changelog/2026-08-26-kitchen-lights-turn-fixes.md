# 2026-08-26 — Kitchen-lights turn: spurious-punt fixes

## Executive summary
Reviewed the "interesting" voice output on **turn-3ca38ed3** ("turn the kitchen lights to 20%").
The light was actuated + verified correctly by the fastpath reflex, but the VOICE was doubly wrong:
Iris first said the filler *"Hm. Let me think about that."*, then ~10s later a deep-tier answer
**barged in** with a rambling monologue leaking the raw entity_id and chain-of-thought. Root cause:
`cortex-logical` returned an **empty pass1** on the reflex-satisfied turn, which fusion read as
`neither_confident` (the filler) AND `self_uncertain` → escalated the proactive punt (which Matt
flipped live) to the slow, ungroomed deep tier. Fixed both drivers; logged the residual deep-mode
problems as ISSUE-010. **STAGED on core-ai — needs `sudo systemctl restart iris-fusion-executive
iris-cortex-logical`.**

## Changes by repo

### iris-fusion-fabric — branch `feat/punt-escalation` (worktree wt-punt) — commit (fusion)
- `fusion-executive/src/fusion_executive/service.py` — **changed.** New `_actuation_succeeded(tool_results,
  reflex_results)` helper; the proactive-punt block now suppresses escalation when a
  control_device/control_media call verified `success=True` this turn (fusion-executed OR reflex) —
  nothing to deliberate over a completed action. Suppression is logged so shadow measurement stays honest.
- `fusion-executive/tests/test_proactive_punt.py` — **changed.** +4 tests (`_actuation_succeeded`:
  fusion result, reflex result, failed, none). Full fusion suite 36 green.

### iris-fusion-fabric — branch `feat/web-tasks-tools` (worktree wt-web-tasks) — commit (cortex)
- `cortex-logical/src/cortex_logical/service.py` — **changed.** New `_FASTPATH_ACK = "Done."`; when
  pass1 is empty AND the fastpath already actuated this turn, emit a brief confirmation
  (proposed_say="Done.", confidence 0.8, needs_clarification False) instead of the empty,
  self_reported_uncertainty=1.0 response that fusion mis-read. Reflex owns the action, logical owns the ack.
- `cortex-logical/tests/test_two_pass.py` — **changed.** +1 test (empty pass1 on a fastpath turn →
  confirmation, not uncertain). Full cortex-logical suite 57 green.

### iris-docs — branch `main`
- `issues/ISSUE-010-deep-mode-output-grooming.md` — **added.** The two residual deep-mode problems
  (rec #3/#4 from the review): ungroomed output leaking entity_ids + chain-of-thought, and the
  qwen3-deep cold-start (8.61s) worsening the barge-in latency. Both deferred per Matt ("just do 1 and 2").
- `STATE.md`, `changelog/2026-08-26-…` — **changed/added** (this).

### iris-fusion-fabric — branch `feat/punt-escalation` (wt-punt) — commit (fusion confirm override)
- `fusion-executive/src/fusion_executive/service.py` — **changed.** Live re-test (turn-2c00dc57,
  "lights to 35%") showed the cortex-logical ack does NOT fire: the reflex outcome often isn't on the
  CuratedContext logical ran on (known race), so `ctx.fastpath` is None and logical returns empty →
  fusion voiced the filler again. Added `_should_confirm_actuation()`: when synthesis fell to
  `neither_confident` but a device actuation succeeded this turn (and none failed), override say →
  "Done." (say-only, mirrors the failed-actuation correction). This is the RELIABLE fix (fusion has
  the ground truth); the cortex-logical ack stays as the cleaner path when its context does carry the
  reflex. +4 tests, fusion suite 40 green.

## Validation (live)
- **turn-2c00dc57** ("turn the kitchen lights on to 35%", 14:31): reflex actuated+verified.
  **Fix #1 confirmed LIVE** — `PROACTIVE-PUNT suppressed reason=self_uncertain (actuation succeeded
  this turn)`, deep-mode silent, no barge-in. **Fix #2 (cortex-logical) did NOT fire** (ctx.fastpath
  race) → filler still voiced → drove the fusion-side confirm override above. Re-test pending after
  the next fusion restart.

## Deploy / live state
- **Fix #1 (punt suppression) LIVE + confirmed** (fusion restarted 2026-08-26 14:31).
- **Fusion confirm override STAGED** — service.py synced; needs one more
  `sudo systemctl restart iris-fusion-executive`, then re-test a light command (expect say="Done.",
  no filler). cortex-logical fix #2 already deployed (harmless; fires only when ctx.fastpath present).
- Nothing else changed. The reflex actuation itself was always correct.

## Follow-ups / owed
- **Matt sudo:** restart iris-fusion-executive + iris-cortex-logical.
- **Origin pushes owed:** `feat/punt-escalation` (fusion), `feat/web-tasks-tools` (cortex), iris-docs.
- **ISSUE-010** (deferred): deep-mode output grooming + qwen3-deep warm-keep; and the standing
  question of whether the deep tier belongs on the voice path at all.
