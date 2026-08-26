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

## Deploy / live state — ALL LIVE + VALIDATED 2026-08-26
- **Both fixes LIVE and confirmed on real turns.** turn-2c00dc57 confirmed fix #1 (punt suppressed,
  no barge-in). turn-7cfb202e ("kitchen lights to 30%", 14:45, after the final fusion restart) is
  fully clean: cortex-logical fired the ack (`pass1 empty on fastpath-satisfied turn — confirming
  say='Done.'`, ctx.fastpath present this turn) → fusion `logical_dominant say='Done.'` (2793ms) →
  voiced "Done.", no filler, no punt, deep-mode silent.
- The cortex ack (fix #2) and the fusion override are complementary: cortex handles it cleanly when
  its context carries the reflex; fusion is the reliable backstop when the race hides it (both
  deployed, both unit-tested). The reflex actuation itself was always correct.
- Aside (NOT actioned, pre-existing): the reflex logs `room=basement` for a kitchen light and the
  reply routed to vg-02 — a room-hint / node-routing quirk unrelated to these fixes.

## Follow-ups / owed
- **Matt sudo:** restart iris-fusion-executive + iris-cortex-logical.
- **Origin pushes owed:** `feat/punt-escalation` (fusion), `feat/web-tasks-tools` (cortex), iris-docs.
- **ISSUE-010** (deferred): deep-mode output grooming + qwen3-deep warm-keep; and the standing
  question of whether the deep tier belongs on the voice path at all.

---

# 2026-08-26 (later) — Web chat had no memory: recent_conversation fetch bug

## Executive summary
First real web chat: Iris didn't remember earlier turns. Root cause found live — the curator's
`_fetch_recent_conversation` treated the intake's `/conversations/{id}` response as a bare list,
but it returns `{"session_id":..., "turns":[...]}`. So `isinstance(rows, list)` was always False →
returned `[]` → the recent_conversation slot NEVER injected → no literal context on any web/resumed
turn. The turns WERE being logged correctly (conversation-log subscriber fine) and the fetch WAS
hitting the intake (200s) — it just parsed the wrong shape. Same wrong assumption also broke the
gateway's resume seeder and history-view. Fixed all three; frontend session-reuse verified working
(the SSE `final` frame carries session_id; the earlier "new session per turn" was stale state).

## Changes by repo
- **iris-fusion-fabric `feat/web-tasks-tools`** — `curator/src/curator/service.py`: extract `.turns`
  from the intake response (tolerates a bare list too). `curator/tests/test_recent_conversation.py`:
  fake now returns the real `{turns:[...]}` shape. 9 green. **STAGED — needs `sudo systemctl restart
  iris-curator`.**
- **iris-web-gateway `main`** — `routes/chat.py` (`_resume_context`) + `static/app.js` (`openSession`):
  same `.turns` extraction so resume + the sidebar history render. Container rebuilt + healthy.

## Deploy / live state
- Gateway container: rebuilt, live. Curator fix: STAGED, needs iris-curator restart.
- After restart: within a web (or resumed) conversation, the cortex gets a RECENT CONVERSATION block
  and Iris carries context turn-to-turn.
