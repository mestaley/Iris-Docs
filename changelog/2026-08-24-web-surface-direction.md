# 2026-08-24 — Web surface direction (tasks/reminders → research assistant)

## Executive summary
Matt named the next big direction: a **web interface** — tasks & reminders first, a research
assistant ultimately (when a larger model is runnable). Captured as a direction decision, not
code. It's thesis surface #2 ("web chat at the desk, same Iris") and rides existing substrate
(the live Events subsystem in iris-volition + `set_reminder` voice tool). Two shape decisions
made: **hybrid** architecture (direct CRUD + a cortex-backed chat box) and a **reminders + tasks
+ projects** data model. No build yet — scoped for a future session.

Also a long cognition-architecture design discussion (~5 turns) that landed a clear direction
and a roadmap sequencing.

## Changes by repo
### iris-docs (master)
- `decisions/ADR-002-web-surface.md` — **added, then extended** — direction ADR: phased P1 tasks/reminders → P2 voice↔web shared state → P3 research assistant; hybrid architecture; reminders+tasks+projects; text→fabric bridge as the key open piece. Extended with **voice-conversation continuity** (UI exposes voice conversations, resumable in text → durable conversation log + session-aware bridge).
- `issues/ISSUE-008-web-surface-phase1.md` — **added, then extended** — Phase 1 scope + build order; added the **conversations view + resume** pillar.
- `decisions/ADR-003-local-serial-cognition.md` — **added** — cognition direction: fold thalamus into a capable tool-calling front-door model (routing-as-tool-call), MoE slow path, retire Gemma/creative; serial spine; contention/pluralism deferred to a **rented experimentation lab** (prove-before-you-buy-local); local-first boundary held. Sequenced AFTER the web surface.
- `STATE.md` — **changed** — added the 3-step roadmap sequencing (web → local cognition rearch → rented experiments).
- `decisions/README.md`, `issues/README.md` — **changed** — indexed ADR-002/003 + ISSUE-008.
- `changelog/2026-08-24-web-surface-direction.md` — **added** — this entry.

## Deploy / live state
None — planning/direction only. Nothing deployed or changed in the running system.

## Follow-ups / owed
- When P1 kicks off: a short design doc (EVENTS.md-style) resolving the open decisions in ADR-002/ISSUE-008 — tasks store (extend Events vs new `iris-tasks`), the text→fabric bridge shape, host + auth + frontend stack.
- Still carried from 2026-08-23: run the voiceprint re-enrollment; push owed origin commits; the queued deferred doc merges.
