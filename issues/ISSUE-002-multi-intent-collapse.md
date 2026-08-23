# ISSUE-002: Multi-intent collapse — single `tool_intent` drops heterogeneous turns
- Status: FIXED (thalamus) + STAGED on core-ai, awaiting restart + soak — iris-fusion-fabric commit 2ecf318 makes prefetch flags independent of tool_intent + adds compound few-shots. VALIDATED live against phi3.5 (3/3 compounds emit actuation+correct prefetch; controls/query/conv/memory unchanged). No contract change. End-to-end cortex verbalization of both halves to be confirmed on a live turn after `sudo systemctl restart iris-thalamus`.
- Severity: high
- Area: cognition/routing
- Discovered: 2026-08-22
- Evidence: `contracts/.../thalamus.py:26,64`; `curator/.../service.py:248-263,422`; `cortex-logical/.../prompt.py:330-343,360-378`

## Symptom
Matt: "there was a time I could shoot off 3-4 unrelated things and they happened." Now a compound utterance that **mixes an actuation with a query** ("turn off the kitchen lights, start some jazz, **and what's the weather?**") drops the query half silently.

## What still works vs. what broke
- WORKS: multiple targets of the *same* action ("living room **and** kitchen lights to 15%"). Proven end-to-end, turn `turn-8d75949a...` 2026-08-21 (fast-path fired 2 reflex calls; logical emitted 2; dedup dropped only the duplicates).
- BROKEN: *heterogeneous* intents in one turn, especially actuate + query.

## Root cause (single most-responsible line)
The thalamus routes the **entire turn** into one `tool_intent` enum — `Literal["none","query","actuation","memory"]` (`thalamus.py:26`, carried singular at `:64`). A phi3.5-mini pre-classifier collapses the compound to one dominant intent *before cognition runs*. The curator then treats the rails as mutually exclusive: reflex fires only `if tool_intent == "actuation"` (`curator/service.py:422`), and query answers require prefetched ambient (cortex is forbidden from calling query tools, `thalamus.py:60`, `cortex-logical/prompt.py:360-378`). One enum can't request both rails → the weaker intent is dropped.

## The contract is NOT the ceiling
`CortexResponse.proposed_actions` is a `list` (`cortex.py:314`); cortex keeps all N tool calls; fusion dispatches all of them (`fusion-executive/service.py:591-631`). The frozen contracts already support N heterogeneous calls. The bottleneck is purely upstream (routing + curator gating + a same-action-only cortex prompt).

## Fix direction (ranked by invasiveness — no contract change needed)
1. Prompt-only: broaden cortex COMPOUND guidance (`prompt.py:330-343`) beyond same-action multi-target. Fixes actuate+actuate; NOT actuate+query.
2. **Routing + curator (the real fix):** `RoutingDecision` already carries independent prefetch booleans orthogonal to `tool_intent`. (a) thalamus few-shot so a compound sets `tool_intent=actuation` AND `prefetch_weather=true`; (b) stop treating the rails as XOR in curator (`service.py:248-263,422`) — allow prefetch + actuate on one turn. Restores "3-4 unrelated things" without touching any frozen contract.
3. Optional/most-invasive: make `tool_intent` a `list` — unnecessary given the prefetch booleans already give the orthogonal rail.
