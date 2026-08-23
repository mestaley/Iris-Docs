# ISSUE-007: Path-2 actuation — ambient / state-expression commands
- Status: open (promoted from backlog 2026-08-23; surfaced in the legacy-docs review)
- Severity: medium (capability gap, not a defect)
- Area: cognition / actuation
- Evidence: not built (grep); legacy `Architecture/BACKLOG.md` + `Core Substrate.md`; ADR-001's fix list does not touch it

## What it is
Two distinct actuation paths:
- **Path-1 (DONE, live):** direct imperative — "turn the basement lights on", "dim the kitchen".
  The curator fast-path (qwen2.5:7b on world-ai) handles these as a reflex.
- **Path-2 (NOT built):** ambient / state-expression — "it's too dark in here", "make it cozy",
  "I'm winding down", "it's freezing". No device or action is named. Iris must **reason over
  current state** (lighting band, time of day, occupancy, recent activity) and synthesize a
  multi-action plan (dim these, warm the color temp, maybe the lamp not the overheads), then
  actuate — or offer.

## Why it matters
This is the THESIS's interesting frontier — cognition producing situated action from an
expressed *state* rather than a parsed *command*. It's what separates "a persistent household
intelligence that reads the room" from "a voice-controlled switch." ADR-001's plan (recall /
multi-intent / grounding / prune) deliberately does not cover it — it's net-new capability.

## Design direction (sketch — needs its own design pass, not a hot-patch)
1. **Detection:** thalamus/curator must recognize a state-expression turn (no imperative verb, no
   named entity) and route it to a reasoning path, NOT the reflex fast-path (which needs a target).
   Likely a new `tool_intent` value or a curator branch — respect [[HISTORICAL-DECISIONS]] HD-4
   (prefetch dual-locus) and, critically, avoid the ISSUE-002 over-prefetch trap.
2. **Grounding:** the turn needs lighting band + time + occupancy + the room's controllable
   entities in ambient BEFORE the cortex reasons (curator prefetch — lighting/occupancy already exist).
3. **Reasoning → plan:** cortex-logical emits a multi-action `control_device` plan from the grounded
   state. `CortexResponse` already carries N actions (no contract change).
4. **Ask-vs-act:** per the conservative Volition stance, an ambient inference should likely OFFER
   ("It's dim in here — want me to bring up the lamps?") before acting, at least initially. Ties to
   the Volition LightingArbiter (already live for dim+occupied+lights-off offers) — Path-2 may be
   its natural generalization.
5. **Validate with the eval harness** (HD-6): state-expression cases at production temperature.

## Relationship
- Generalizes the live Volition LightingArbiter (see the `iris-proactive-lighting-scope` memory).
- Depends on the same grounded-context plumbing as ISSUE-005/006 (cortex must see real state).
- A candidate first real exercise of the "reads the room" thesis commitment.
