# iris-docs

Single source of truth for the Iris fabric: current state, known issues, backlog,
architecture, and decisions. Lives on bastion at `/opt/repos/iris-docs`.

**Why this exists:** state was scattered across `STATE.md`, `THESIS.md`, `BACKLOG.md`,
`ARCHITECTURE.md`, `EVENTS.md`, `PRESENCE_FUSION.md` in six repos plus ~30 memory files.
When something broke there was no one place that said "here's what's live, here's what's
known-broken." That fragmentation *is* the complexity problem. This repo shrinks it.

## Layout

| File / dir | What it holds |
|---|---|
| `THESIS.md` | The canonical intent — what Iris fundamentally is. The evaluation filter for all work. |
| `PRINCIPLES.md` | Vision (Jarvis target), methodological commitments, what Iris is NOT, working agreement. |
| `STATE.md` | The live dashboard — hosts, running services, what's live / observe-only / dead, current focus. Update when reality changes. |
| `ARCHITECTURE.md` | The per-turn hot path, service inventory, and where a turn actually goes now. |
| `PATTERNS.md` | Durable engineering patterns (each with the incident that motivated it). Check before re-inventing. |
| `BACKLOG.md` | Prioritized backlog, one line per item. |
| `issues/` | One file per known issue (`ISSUE-NNN-slug.md`), with status + evidence. |
| `decisions/` | Architecture Decision Records (`ADR-NNN-slug.md`) — one per irreversible-ish call. `HISTORICAL-DECISIONS.md` holds the still-in-force decisions ported from the legacy log. |

## Conventions
- Plain markdown only — greppable, diffable, git-native. That's the point.
- Every claim about live behavior cites `file:line`, a systemd unit, or a log ref.
- Issues and ADRs are append-mostly: change `Status:`, don't delete history.
- Absolute dates, never "last week."

## Relationship to auto-memory
The `~/.claude/.../memory/` files are Claude's working recall. This repo is the
durable, human-owned record. When they disagree, this repo wins; memory gets corrected.
