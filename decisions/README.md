# Decisions (ADRs)

One file per decision: `ADR-NNN-short-slug.md`. Record the *why*, not just the *what*, so
future-you doesn't relitigate it.

```
# ADR-NNN: Title
- Status: proposed | accepted | superseded by ADR-MMM
- Date: YYYY-MM-DD
- Context: what forced the decision
- Decision: what we're doing
- Consequences: what this costs / enables
```

## Index
- ADR-001 — Continue vs. nuke half (accepted; implemented 2026-08-23)
- ADR-002 — Iris web surface: tasks/reminders + voice-conversation continuity → research assistant (accepted direction; P1 in ISSUE-008) — **PARALLEL build with ADR-003**
- ADR-003 — Local serial-spine cognition (front-door + MoE); contention deferred to a rented lab (accepted direction; **PARALLEL** with ADR-002, coordinated by the shared bridge contract)
- ADR-004 — Automaticity and attention (salience-gated deferred deliberation; reframes ADR-003's punt model)
- ADR-005 — Name-based actuation contract: accept a device name, resolve name→entity_id server-side, retire the cortex coercion shim (accepted 2026-09-30; impl in progress) — design: `design/ACTUATION-CONTRACT.md`
- ADR-006 — Scenes are Iris-owned (YAML + a deterministic `activate_scene` tool), not HA scenes; built on ADR-005; mechanism decoupled from (future) Volition triggering (accepted 2026-11-02; impl pending)
- `HISTORICAL-DECISIONS.md` — still-in-force decisions ported from the legacy log
