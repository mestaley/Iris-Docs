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
