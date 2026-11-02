# Issues

One file per issue: `ISSUE-NNN-short-slug.md`. Header block, then evidence and notes.

```
# ISSUE-NNN: Title
- Status: open | in-progress | fixed | wontfix
- Severity: high | medium | low
- Area: audio/recall, cognition, actuation, memory, infra, ...
- Discovered: YYYY-MM-DD
- Evidence: file:line, log refs, unit names
```

Append notes with dates; change `Status:` in place rather than deleting. When fixed, keep
the file and record the fix + commit.

## Index
- ISSUE-001 — Speaker gate threshold cuts into genuine speaker → first-try silence (high, open)
- ISSUE-002 — Multi-intent collapse: single `tool_intent` drops actuate+query turns (high, open)
- ISSUE-003 — Consolidation cluster is a write-only dead-end (medium, open)
- ISSUE-004 — Dead code: wake-arbiter, fabric_brain bridge, legacy shells, dup VAD, etc. (low, open)
- ISSUE-005 — Cortex speech not grounded in tool results → "articulate lie" family (high, live)
- ISSUE-006 — Iris should say a backend ERRORED (503), not vaguely deflect (medium, open)
- ISSUE-007 — Path-2 actuation: ambient/state-expression commands ("make it cozy") (medium, open)
- ISSUE-008 — Web surface Phase 1: tasks & reminders, hybrid (new initiative, per ADR-002)
- ISSUE-009 — Perceived latency is governed by VAD endpointing, not cortex first-audio (medium, open)
- ISSUE-010 — Deep-mode output grooming (medium, deferred)
- ISSUE-011 — Proactive rail / creative retired (see STATE.md)
- ISSUE-012 — No-wakeword follow-up & conversation continuity
- ISSUE-013 — Conversation tightening (10-day audit)
- ISSUE-014 — Calendar: reads real events then self-interrupts with "schedule is clear" — logical confabulates empty @conf 1.00, un-retired creative speaks then gets overridden, fusion calls it disagreement=False (high, open)
- ISSUE-015 — Time-bound notifications (calendar/weather/deliberation) expire silently when no presence is detected in their window (occupancy-gated hold → expiry, attempts=0); user-set reminders NOT affected (expiry=None). (low, open)
- ISSUE-016 — TV control is Plex-only; broaden to power/input/transport/volume (depends on how the TV is exposed in HA). (medium, open)
- ISSUE-017 — "Iris, STOP": a verbal hard-abort of the current turn (halt speech, cancel in-flight turn, keep_listening=NO) — distinct from acoustic barge-in. (medium, open)
