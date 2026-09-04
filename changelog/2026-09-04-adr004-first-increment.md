# 2026-09-04 — ADR-004 first increment: front door never stalls

## Executive summary
Drafted ADR-004 (automaticity vs. attention — retire "punt", salience-gated parallel deliberation
that refines + surfaces deferred) + a THESIS section, then landed the cheap first increment in
fusion: kill the "give up and stall." Dropped `self_uncertain` from the escalation gate (it was a
give-up signal, not salience), made the front door voice any present say regardless of confidence,
and replaced the think-stall filler with an honest retry-invite that fires only on a genuinely empty
front door. STAGED — needs `sudo systemctl restart iris-fusion-executive`.

## Changes by repo
- **iris-docs** — `decisions/ADR-004-automaticity-and-attention.md` (new), ADR-003 annotated,
  `THESIS.md` new "Automaticity vs. attention" section.
- **iris-fusion-fabric `feat/punt-escalation`** — `fusion-executive/.../service.py`:
  `_proactive_punt_reason` drops `self_uncertain` (gate = affect/complexity only); `_synthesize`
  `logical_valid = bool(say)` (no confidence floor); `CANNED_FALLBACK` → honest retry-invite.
  `synthesis.py:_select_say` same treatment (the HTTP/eval path). Punt tests updated. 41 green.

## Deploy / live state
- STAGED to core-ai. **Owed:** `sudo systemctl restart iris-fusion-executive`. After restart: a
  low-confidence/uncertain turn gets Iris's immediate best answer (not the filler), and the deep tier
  no longer spins up on `self_uncertain` (only affect/complexity).

## Follow-ups (the real project, per ADR-004)
- **Salience/attention mechanism** — what earns conscious deliberation, computed cheaply (the hard part).
- **Deferred/proactive deep surfacing** — wire deep-mode → Volition, off the voice hot path (retires
  the same-turn barge-in).
- **Expand the automatic layer** — more of the sensory fabric handled subconsciously, out of the prompt.
- Origin pushes still owed.
