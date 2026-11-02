# 2026-11-02 — Scene adherence: make the model CALL activate_scene

## Executive summary
Post-deploy, scenes said "activating" but nothing happened. Trace (turn-1e7b9d77, "movie time"):
`activate_scene` WAS retrieved into the cortex tool set, but Hermes-3-8B emitted `actions=0` and just
*said* "I'll set up the living room for a movie night" — it acknowledged instead of calling the tool
(a tool-calling adherence gap, not a wiring bug; the scene/expander/retrieval all work). Fix: rewrote the
`activate_scene` description to be imperative — frame it as an ACTION, "you MUST call activate_scene …
do NOT instead reply that you'll set it up", plus phrase→scene matching hints. Built; **pending deploy**
(curator restart only).

## Changes by repo
### iris-fusion-fabric (via wt-integration) — branch `feat/scene-adherence` (commit 0e26b6d)
- `curator/src/curator/tool_catalog.py` — **changed**: `_activate_scene_tool()` description reframed
  imperative (action-not-chat, "MUST call", don't-just-acknowledge, phrase→scene hints); `scene` param
  description tightened. Logic unchanged (still only offered when scenes are configured).

## Deploy / live state
- **Built, pending deploy.** Branch `feat/scene-adherence` (0e26b6d), unpushed.
- Model-facing description is built live by `get_catalog()` → a **curator restart** picks it up. No reseed
  (retrieval already matches "movie time"), no tool-registry change, no new deps.

## Follow-ups / owed — deploy (Matt)
1. Merge `feat/scene-adherence` → deployed branch; deploy to core-ai (bundle flow).
2. `sudo systemctl restart iris-curator`.
3. Verify by voice: "Iris, movie time" → the lights change (cortex log shows `actions=1`, not a verbal ack).
- **If it STILL just acknowledges** (actions=0): the 8B isn't honoring the tool even with imperative text.
  Next levers then: a prompt-level nudge in cortex-logical, or handle scene phrases as a lower-layer
  reflex/intent. Flag it and we escalate.
