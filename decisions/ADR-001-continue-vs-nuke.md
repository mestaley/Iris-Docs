# ADR-001: Continue the current architecture vs. nuke half of it
- Status: accepted
- Date: 2026-08-23 (proposed 2026-08-22; all four investigations landed)

## Context
Matt: "I still feel everything has regressed — Occam's Razor — have I made this too
complicated? In the beginning I could have a conversation and she'd answer the first time,
and I could fire off 3-4 unrelated commands and they happened." A four-agent review answered,
per subsystem: does it add felt value net of its cost (recall loss, latency, live corruption,
diagnosis burden)?

## Finding
The felt regression is **not** "the whole architecture is too complex." The live-turn plane is
healthy and does more than the old one (proactive speech, corrections, multi-utterance, ~0.2s
first-audio, barge-in). The regression is four specific things:

1. **Recall (ISSUE-001):** two precision gates in series took first-try response ~96% → ~78%
   (measured over 13 days of logs; a genuine command rejected at cos 0.344 vs a 0.35 floor).
   A **tuning** fix, not an architecture fix.
2. **Multi-intent (ISSUE-002):** a single-`tool_intent` pre-classifier architected out a
   capability the cortex still supports natively (`CortexResponse` already carries N calls;
   fusion already dispatches all). A **routing/curator** fix, no contract change.
3. **Articulate lie (ISSUE-005):** cortex speech isn't grounded in `tool_results`, so she
   confidently claims outcomes she can't see. One fault wearing three faces. A **grounding** fix.
4. **Diagnosis surface (ISSUE-004):** a "no response" has 12 independent suspects; a whole
   reflection/consolidation plane + a pile of vestigial code (wake-arbiter, the dormant
   fabric_brain bridge, four legacy `app.py` shells, `keep_listening`, dead bus topics, dup VAD,
   355 unread consolidated records) never touches a live turn. **The diagnosis surface is wider
   than the fault surface — that is the real complexity cost Matt feels.**

The thing the thesis note feared — consolidation corrupting prefs live — is **not happening**:
it writes only to an isolated tier with no working read endpoint (ISSUE-003).

## Decision
**Don't nuke the trunk. Prune the dead limbs, retune the gate, and close the two capability gaps.**

- **KEEP (live, value-earning):** episodic memory-substrate, working-memory field, prosodic
  affect (now live), volition (live — audit for noise), presence substrate, the whole streaming
  live-turn plane.
- **NUKE / hard-gate (zero live-turn cost):** consolidation + dream-mode + reflection-coordinator
  + consolidated tier (355 unread records); wake-arbiter (:8090); dormant fabric_brain bridge;
  the four legacy `app.py`/shell configs; `keep_listening` (or wire it); dead bus topics;
  duplicate `vad_consumer.py`; Spotify creds + drifted identity_anchor in world-ai/fusion config.
- **FIX:** (a) lower `SPEAKER_THRESHOLD` 0.35 → ~0.28 and measure (ISSUE-001); (b) dual-rail
  routing so actuate+query compounds both fire (ISSUE-002); (c) ground say-text in `tool_results`
  (ISSUE-005).

## Ordering (biggest felt win first, lowest risk first)
1. **Speaker-threshold retune** — one env var, no redeploy, recovers "answers the first time."
2. **Ground outcome claims in tool_results** (ISSUE-005) — kills the "confidently wrong" trust hit.
3. **Dual-rail multi-intent** (ISSUE-002) — restores "do 3 things at once."
4. **Prune the confirmed-dead set** (ISSUE-004) — halves the no-response diagnosis surface.
5. **Hard-gate/delete the consolidation cluster** (ISSUE-003) — only after confirming nothing
   intends to wire the read-back soon; otherwise leave the code, stop the writer.

## Consequences
- Recovers the two things Matt actually misses (first-try response, multi-command) plus the
  trust-eroding "articulate lie" — without a rewrite.
- Halving the diagnosis surface is the durable win: the next time something breaks, ~6 suspects
  instead of 12.
- Cost: the consolidation/reflection plane is long-term memory ambition. Nuking it forfeits that
  roadmap; hard-gating (stop the writer, keep the code) preserves it at zero live cost — preferred.
- Verification owed before deletion: grep `service.py` for the two `probably-dead` httpx/weather
  rows; confirm the reflection plane is truly unread by any live consumer.

## Implementation log (2026-08-23)
All five steps implemented same day. Deploy needs Matt's sudo on core-ai (staged, inert until restart).
1. **Speaker threshold** — ✅ DEPLOYED LIVE. `SPEAKER_THRESHOLD` 0.35→0.28 on audio-ai (iris-conditioning b086e59); container recreated + verified.
2. **Ground outcome claims** — ✅ implemented + tested (34/34), STAGED. Creative-prompt guardrail (iris-fusion-fabric 34f2400); actuation-failure half was already live.
3. **Dual-rail multi-intent** — ✅ implemented + VALIDATED live against phi3.5, STAGED. Thalamus prompt (2ecf318); no contract change.
4. **Prune dead code** — ⚠️ PARTIAL. Spotify creds removed (world-ai 68369bf). recall_precedent KEPT (roadmap). Rest staged — higher blast-radius than expected (frozen-contract field, shared bus package, whole-service deploys). See ISSUE-004.
5. **Hard-gate consolidation writer** — ✅ code guard implemented + tested (7/7), STAGED. dream-mode (iris-fusion-fabric 13d66b5). Operational timer-disable needs sudo.

Commits owed to origin (automation can't push): iris-fusion-fabric {13d66b5, 2ecf318, 34f2400} + prior 40a20f1; iris-conditioning b086e59 + prior 261e4db; world-ai 68369bf + prior 12561ec.

Staged core-ai deploy (Matt's sudo), restart order safe as listed:
```
sudo systemctl restart iris-thalamus            # step 3 (multi-intent)
sudo systemctl restart iris-cortex-creative     # step 2 (grounding)
sudo systemctl restart iris-dream-mode          # step 5 (write gate active)
sudo systemctl disable --now iris-dream-mode-trigger.timer   # step 5 operational belt
```
Rollback: `git checkout` the component + restart; or revert the specific commit.

## Open threads (not blocking this decision)
- Basement lux replacement (radar-only vs standalone sensor).
- Task #31 (arm await window for proactive reply-inviting turns), Task #29 (fuse text valence).
- Push the three owed commits from Matt's shell.
