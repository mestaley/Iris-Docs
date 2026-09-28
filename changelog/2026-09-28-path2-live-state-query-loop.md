# 2026-09-28 — Path 2: cortex-logical live-state query loop (light-status grounding)

## Executive summary
Fixed the light-status hallucination at the root. cortex-logical now executes READ-only query tools
(get_entity_state, get_occupancy, get_*) *itself* between pass1 and pass2 — via the fabric's
TOOL_REQUEST→TOOL_RESULT round-trip against live HA/world-ai — and feeds the result into pass2, so
"are the kitchen lights on?" is answered from ground truth instead of confabulated. Realizes Matt's
principle (query fast state live, never prefetch it). **Staged; needs `sudo systemctl restart
iris-cortex-logical`.** Built on `feat/web-tasks-tools` (the actually-deployed cortex-logical base),
NOT `feat/punt-escalation` — see the branch-tangle note below.

## Changes by repo
### iris-fusion-fabric — branch `feat/web-tasks-tools` (cortex-logical)
- `src/cortex_logical/service.py` — **changed**: `ReadToolBroker` (DDS round-trip, bridges bus thread↔
  asyncio loop, `READ_TOOL_TIMEOUT_S=2s`); `_format_tool_result`; `_process_curated` splits pass1 tool
  calls by `side_effects` (WRITE→fusion actuates unchanged; READ→cortex executes inline), feeds read
  results to pass2, `proposed_actions=write_actions` (reads never reach fusion); actuation-only skip now
  only fires for pure write (a read query runs pass2 to voice the grounded answer); worker + main wire
  the broker + subscribe TOOL_RESULT.
- `src/cortex_logical/prompt.py` — **changed**: `build_wrapper_messages` takes `tool_results`, replayed
  as `role:tool` messages after the pass-1 assistant turn.
- `tests/test_two_pass.py` — **added** `test_read_query_executes_live_and_grounds_pass2`; `test_worker.py`
  fake updated for the new arg. 60 green.

### iris-docs
- `changelog/2026-09-28-path2-live-state-query-loop.md` — this entry.

## Deploy / live state
- **STAGED**: `sudo systemctl restart iris-cortex-logical`. Verified in the deployed test run: a status
  query executed the read live and voiced "The kitchen lights are on." with `proposed_actions=[]`.
- **Incident (contained, no live regression):** my first attempt scp'd the Path 2 files built on
  `feat/punt-escalation`, which clobbered core-ai's on-disk cortex-logical (deployed from
  `feat/web-tasks-tools`, with recent_conversation + get_tasks). Caught before any restart (running
  process untouched), restored from `wt-web-tasks`, and re-applied Path 2 onto the correct base.

## Follow-ups / owed
- **BRANCH RECONCILIATION (Matt's option B, agreed for "after"):** the deployed `/opt/iris-fusion-fabric`
  is a hybrid of uncommitted files from `feat/events-queue` (checkout) + `feat/web-tasks-tools`
  (cortex-logical) + `feat/punt-escalation` (fusion/tool-registry/curator, this session) + others. No
  single branch reflects what's running. This must be merged into one coherent, committed, deployable
  branch before the owed origin pushes. High priority — it's fragile.
- Owed pushes now span: `feat/web-tasks-tools` (Path 2), `feat/punt-escalation` (sole-actuator etc.),
  world-ai `fix/actuation-settle-verify`, iris-docs.
- Formal reflex-code removal; cortex-logical dropped-`service` tool-call reliability (fusion repair holds it).
