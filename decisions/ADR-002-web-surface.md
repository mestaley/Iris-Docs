# ADR-002: Iris web surface — tasks/reminders now, research assistant later
- Status: accepted (direction); Phase 1 scoped in [ISSUE-008](../issues/ISSUE-008-web-surface-phase1.md)
- Date: 2026-08-24

## Context
Voice is surface #1. [THESIS.md](../THESIS.md) names web-chat-at-the-desk as the eventual
surface #2 — *"the same Iris across both, with shared accumulated memory and consistent
character."* Matt: the one thing that would make Iris super useful is a web interface — **tasks
and reminders first, a research assistant ultimately** (when a larger model is runnable).

Not greenfield: tasks/reminders already have a **live backend** — the Events subsystem in
`iris-volition` (durable sqlite queue + delivery arbiter, intake `:8015`), and `set_reminder` is
already a working voice tool that enqueues into it. Phase 1 is a screen over substrate that
partly exists.

## Decision
Build the web surface, **phased**:
- **P1 — Tasks & reminders.** Needs no larger model; ships the daily-useful part fast.
- **P2 — Voice ↔ web shared state.** A reminder set by voice shows on the screen; a task added
  on the web, Iris knows by voice. "Same Iris, one state."
- **P3 — Research assistant.** Gated on a larger model + document ingestion (the Drive-workspace
  backlog item; the custody-agreement validation target). The horizon payoff.

Two design choices (Matt, 2026-08-24):
- **Architecture = HYBRID.** Direct CRUD against a tasks/reminders store for reliable task
  management, PLUS a cortex-backed chat box for conversational "same Iris" interaction. The chat
  box is the P3 shape started early.
- **Data model = reminders + tasks + projects.** Reminders (time-anchored, exists) + tasks
  (persistent: title, status, notes, optional due) + projects (grouping/tags).

**Non-negotiable (thesis):** shared state. The tasks/reminders store is the SAME store voice
reads/writes (exposed to the fabric as tools — `get_tasks`, `create_task`, `get_reminders`, …),
and the chat box reasons over the same `memory-substrate`. One Iris, two surfaces — never a
second, diverging assistant.

## Key architectural pieces / open questions (resolve in ISSUE-008 or a design doc)
1. **Tasks/projects store** — extend the Events sqlite in `iris-volition`, or a new `iris-tasks`
   service? _Lean: co-locate with Events, sqlite to start (matches that choice); revisit if it
   grows._
2. **Text→fabric bridge (the meatiest piece).** For the chat box to be *the same Iris*, text must
   enter the fabric and a text response come back — today the fabric originates from voice
   conditioning and speaks via gRPC to voice nodes. Options:
   - (a) A web **gateway** publishes a synthetic `FeatureStream` (a "web node") and subscribes to
     the turn's decision, rendering **text instead of TTS**. Reuses the full cognition pipeline —
     the right, thesis-aligned answer — but needs an HTTP↔DDS bridge + a text response sink.
     _(Retired `fabric_brain`/`/decide_v2` HTTP paths and the dormant brain `:8000` are prior art
     to study, not revive.)_
   - (b) A thinner text endpoint hitting a cortex directly. Simpler; risks divergence from
     voice-Iris.
   - _Recommend growing toward (a); the P1 chat box can start minimal and mature. This bridge is
     reusable infrastructure — it unlocks any future non-voice surface._
3. **Hosting + auth.** Local-first, LAN, single-user. A small new service (backend API + frontend).
   Auth light but **not zero** — it's the first persistent networked surface. Host TBD (core-ai? a
   dedicated ui host? `ui-01` is a basement touchscreen/display, not a server).
4. **Frontend stack** — self-hosted, simple; TBD.

## Consequences
- Delivers daily usefulness (a task list) without waiting on hardware; P3 rides the same shape.
- Introduces Iris's **first persistent networked surface** → a new auth/security surface for a
  local-first system. Keep it LAN-scoped; treat it as outward-facing when deciding auth.
- The text→fabric bridge is reusable infrastructure for every future surface.
- Relates to: [ISSUE-007](../issues/ISSUE-007-path2-ambient-actuation.md) (reasoned actuation),
  the intentions-scheduler backlog item (this surface becomes its UI), and the Drive-workspace +
  custody-agreement validation target (P3).

## Not deciding yet
Frontend framework, exact host, auth mechanism, and whether tasks live in Events vs a new service
— those land in ISSUE-008 / a design doc once P1 starts. This ADR fixes the *direction* and the
two shape decisions.
