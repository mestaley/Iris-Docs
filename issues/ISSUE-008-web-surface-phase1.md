# ISSUE-008: Web surface — Phase 1 (tasks & reminders, hybrid)
- Status: open (scoping; per [ADR-002](../decisions/ADR-002-web-surface.md))
- Severity: n/a (new initiative — the highest-value near-term direction)
- Area: new surface / web
- Depends on: Events subsystem (`iris-volition`, `:8015`, live), `memory-substrate` (live)

## Goal
A LAN web app that makes tasks & reminders genuinely useful daily, built as the eventual
research-assistant shape (hybrid: CRUD + a "same Iris" chat box). No larger model required.

## Phase 1 scope
1. **Data model** (reminders + tasks + projects):
   - `reminder`: text, when (time-anchored) — already exists via Events; expose + render.
   - `task`: title, status (todo/doing/done), notes, optional due, project_id, timestamps.
   - `project`: name, optional tags/color. Tasks group under projects.
   - Store: sqlite to start (co-located with Events in `iris-volition`, or a small `iris-tasks`
     service — decide first). Must be the SAME store the fabric reads/writes.
2. **Backend API** — CRUD for tasks/projects; reminders via the existing Events intake.
3. **Fabric tools (shared state)** — `get_tasks`, `create_task`, `update_task`, `get_reminders`
   in tool-registry so **voice can see and change the same list** ("Iris, what's on my list?",
   "add milk to the groceries project"). This is the P2 seed and the thesis's non-negotiable.
4. **Chat box (minimal, growing)** — a text chat that routes to the fabric. Start minimal (even
   a stub or thin path), but wire it toward the ADR-002 text→fabric bridge option (a) so it
   becomes "the same Iris," not a separate assistant. Don't let it diverge.
5. **Frontend** — simple, self-hosted: task list grouped by project, add/edit/complete,
   reminders view, chat panel.
6. **Hosting + auth** — LAN, single-user, light auth (not zero). Pick a host.

## Suggested build order
1. Decide the store (extend Events vs `iris-tasks`) + the task/project schema.
2. Backend CRUD + the fabric tools (get/create/update) — voice gets tasks even before the UI ships.
3. Minimal frontend over the API (tasks + projects + reminders), no chat yet.
4. Text→fabric bridge + chat box (ADR-002 §2 — the meatiest piece; can trail the CRUD UI).
5. Auth + hosting hardening.

## Explicitly NOT Phase 1
Research assistant, document ingestion, larger-model reasoning, Drive workspace — that's P3
(hardware-gated). Keep the P1 schema/bridge from painting P3 into a corner, but don't build it.

## Open decisions (from ADR-002)
- Tasks store location (Events sqlite vs new `iris-tasks` service).
- Text→fabric bridge shape (synthetic-FeatureStream web-node vs thin cortex endpoint).
- Host + auth mechanism + frontend stack.
A short design doc (like `EVENTS.md` / `PRESENCE_FUSION.md`) is warranted before building — offer it when P1 kicks off.
