# ADR-001: Continue the current architecture vs. nuke half of it
- Status: proposed (pending Matt's call; one investigation still landing)
- Date: 2026-08-22

## Context
Matt: "I still feel everything has regressed — Occam's Razor — have I made this too
complicated? In the beginning I could have a conversation and she'd answer the first time,
and I could fire off 3-4 unrelated commands and they happened." A four-agent review was run
to answer, per subsystem: does it add felt value net of its cost (recall loss, latency, live
corruption, diagnosis burden)?

## Finding
The felt regression is **not** "the whole architecture is too complex." It's two specific
structural costs plus one dead-weight cluster:
1. **Recall (ISSUE-001):** two precision gates in series took first-try response ~96% → ~78%.
   A tuning fix, not an architecture fix.
2. **Multi-intent (ISSUE-002):** a single-`tool_intent` pre-classifier architected out a
   capability the cortex still supports natively. A routing/curator fix, no contract change.
3. **Dead weight (ISSUE-003/004):** consolidation cluster (355 unread records, no read path),
   wake-arbiter (never wired), duplicate VAD, a disabled reflection-coordinator.

The live, hot-path, value-earning pieces are healthy: episodic memory, working-memory field,
prosodic affect (now live). Consolidation — the thing the thesis note feared was corrupting
prefs live — writes only to an isolated tier nothing reads.

## Decision (proposed)
**Don't nuke the trunk. Prune the dead limbs and fix the two structural costs.**
- KEEP: episodic memory, working-memory field, prosodic affect, volition (audit for noise),
  presence substrate.
- NUKE / hard-gate: consolidation + dream-mode + reflection-coordinator + consolidated tier;
  wake-arbiter; duplicate VAD.
- FIX: lower speaker threshold toward EER (ISSUE-001); dual-rail routing for multi-intent (ISSUE-002).
- Ordering: recall tune first (biggest felt win, lowest risk) → multi-intent → then prune dead code.

## Consequences
- Recovers "answers the first time" and "do 3 things at once" — the two things Matt actually
  misses — without a rewrite.
- Removing the dead cluster shrinks the diagnosis surface (fewer suspects when something breaks),
  which is the real complexity cost he's feeling.
- Open: the pipeline/dead-code map (agent 4) will confirm the full dead-code list and latency
  budget before anything is deleted. This ADR is not accepted until that lands and Matt signs off.
