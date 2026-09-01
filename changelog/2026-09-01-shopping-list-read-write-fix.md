# 2026-09-01 — Shopping list: can't read it + polluted with actuations

## Executive summary
Matt: "she can't tell me what's on my list." Two bugs. READ: `get_tasks` had no renderer
in cortex-logical, so the prefetched list was raw-dumped and the cortex returned `say=None` →
fusion voiced the filler. WRITE: the list was polluted with device/media commands misfiled as
tasks ("Turn off living room lamps" ×7, "Play Frank Sinatra", basement-lights variants) — 16 of 18
"tasks" were actuations — because `create_task`'s description never forbade it. Fixed both; cleaned
the store live (kept the 2 real items). Read/write fixes STAGED — need cortex-logical + curator restart.

## Changes by repo — iris-fusion-fabric `feat/web-tasks-tools`
- `cortex-logical/src/cortex_logical/prompt.py` — new `_render_tasks`: renders get_tasks as a
  "THE LIST" block (titles by status, explicit "nothing on the list" when empty) instead of the raw
  else-dump that produced say=None. +2 tests (13 green).
- `curator/src/curator/tool_catalog.py` — `create_task` description hardened: NEVER for device/media
  commands (use control_device/control_media), store the item not the whole sentence.

## Deploy / live state
- **Store cleaned LIVE** (no restart): deleted 16 device/media tasks via `:8015` DELETE; renamed the
  2 keepers "Add X to the shopping list" → "X". List now = Greek yogurt, Swiffer dusters.
- **STAGED:** prompt.py + tool_catalog.py synced to core-ai. Need `sudo systemctl restart
  iris-cortex-logical iris-curator`.

## Follow-ups / owed
- **Matt sudo:** restart iris-cortex-logical + iris-curator; then a "what's on my list?" read verbalizes.
- **Watch:** whether the description alone stops actuations→tasks. The "×7 in one burst" (08-28 02:08)
  suggests a compound/retry misfire worth watching; if it recurs, add a dispatch-side guard (reject
  create_task whose title matches an actuation verb) — deferred pending recurrence.
- Origin pushes still owed (rides feat/web-tasks-tools + the other feature branches).

---
## Follow-on (same day): the add path
Live test after restart: READ works ("Your shopping list currently includes Greek yogurt and
Swiffer dusters"). But the ADD exposed two more bugs — it stored the whole sentence and voiced the
filler. Fixed:
- `volition/.../tasks_routes.py` — `_normalize_title` at the create route: "add X to the … list" →
  item ("Paper towels"); plain titles untouched. Every create path (voice + web). +1 test.
- `fusion-executive/.../service.py` — broadened `_actuation_succeeded` to the write tools
  (create_task/update_task/set_reminder/mute + actuation), so a completed task write with an empty
  front door confirms ("Done.") instead of the filler, and suppresses a spurious punt. +1 test.
- Store: renamed the stray "add paper towels…" test task → "Paper towels". List = Greek yogurt,
  Swiffer dusters, Paper towels.

**Owed (Matt sudo):** `sudo systemctl restart iris-volition iris-fusion-executive` — then the add
path stores clean + confirms.
