# 2026-08-24 — Web surface direction (tasks/reminders → research assistant)

## Executive summary
Matt named the next big direction: a **web interface** — tasks & reminders first, a research
assistant ultimately (when a larger model is runnable). Captured as a direction decision, not
code. It's thesis surface #2 ("web chat at the desk, same Iris") and rides existing substrate
(the live Events subsystem in iris-volition + `set_reminder` voice tool). Two shape decisions
made: **hybrid** architecture (direct CRUD + a cortex-backed chat box) and a **reminders + tasks
+ projects** data model. No build yet — scoped for a future session.

## Changes by repo
### iris-docs (master)
- `decisions/ADR-002-web-surface.md` — **added** — direction ADR: phased P1 tasks/reminders → P2 voice↔web shared state → P3 research assistant; hybrid architecture; reminders+tasks+projects; the text→fabric bridge as the key open piece; shared-state-is-non-negotiable.
- `issues/ISSUE-008-web-surface-phase1.md` — **added** — Phase 1 scope + suggested build order (store → fabric tools → CRUD UI → chat bridge → auth), explicitly excluding P3.
- `decisions/README.md`, `issues/README.md` — **changed** — indexed ADR-002 / ISSUE-008 (and marked ADR-001 accepted).
- `changelog/2026-08-24-web-surface-direction.md` — **added** — this entry.

## Deploy / live state
None — planning/direction only. Nothing deployed or changed in the running system.

## Follow-ups / owed
- When P1 kicks off: a short design doc (EVENTS.md-style) resolving the open decisions in ADR-002/ISSUE-008 — tasks store (extend Events vs new `iris-tasks`), the text→fabric bridge shape, host + auth + frontend stack.
- Still carried from 2026-08-23: run the voiceprint re-enrollment; push owed origin commits; the queued deferred doc merges.
