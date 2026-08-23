# ISSUE-005: Cortex speech isn't grounded in tool results → "articulate lie" family
- Status: open
- Severity: high
- Area: cognition
- Discovered: 2026-08-22 (pipeline map; long-standing per `BACKLOG.md:13`)
- Evidence: `BACKLOG.md:13` ("cortex blind to ToolResults"); diagnosis-surface analysis in ARCHITECTURE.md

## Symptom
Iris asserts success she can't actually see: claims multiple lights changed when some didn't,
says "now playing X" on Spotify without confirmation, hallucinates calendar entries. A felt-
regression contributor — she *sounds* confident and wrong, which erodes trust more than silence.

## Root cause (one fault, three faces)
The creative cortex streams prose optimistically **before** and **without** grounding in the
turn's `tool_results`. The same single defect surfaces in three different services, so it reads
as three separate bugs:
- **cortex-creative** — confabulates the success sentence.
- **fusion-executive** — reconciles/voices it without a grounding check.
- **tool-registry** — the actual result exists but never gets fed back into the spoken text.

This is the concrete example of "diagnosis surface wider than fault surface": three suspects,
one cause.

## Why it matters for the "too complicated" question
This is the highest-value *correctness* fix after recall/multi-intent. It's not an architecture
problem — the tool results exist; they're just not on the path to the spoken sentence. Grounding
the say-text in `tool_results` (or gating optimistic prose until results land for actuation turns)
kills all three faces at once.

## Fix direction
- Feed `tool_results` back into the say composition for actuation/query turns; make the confident
  success sentence conditional on the verified result (the actuation-verify plumbing already
  exists — ISSUE-style success/failure is known at fusion).
- Consider: creative may stream *conversational* prose optimistically, but *claims of outcome*
  ("done", "now playing", "turned off") must be gated on results.

## Relationship
Downstream of the same honesty gap that the actuation-verify work addressed for lights; this
generalizes it to speech for all tool-backed claims.
