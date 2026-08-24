# Web Surface — Phase 1 Design (tasks / reminders / conversations)

_Design doc for [ISSUE-008](../issues/ISSUE-008-web-surface-phase1.md), per
[ADR-002](../decisions/ADR-002-web-surface.md). Resolves the open decisions with concrete
schema, API, and the session-aware text→fabric bridge. Grounded in the live code (Events
subsystem, tool-registry, the DDS turn path). **Design only — nothing here is built yet.**_

## 0. Summary of decisions
| Question | Decision |
|---|---|
| Store | **Extend the existing volition events service** (core-ai, `:8015`, WAL sqlite `state/events.db`) with new `tasks`/`projects`/`conversations` tables — NOT a new `iris-tasks` service. |
| Data model | Reminders stay `pending_events` (kind=reminder); tasks + projects + conversation-log get their own tables. |
| Shared state | Voice and web both go through the volition intake (`:8015`) and new fabric tools — one owner, one store. |
| Bridge | A **new web-gateway service** (DDS participant on domain 42) that publishes a synthetic `FeatureStream` ("web node") and reads back `FusionDecision.say` / `SpeechUtterance` as **text**. No TTS, no reviving `fabric_brain`. |
| Chat/CRUD split | Gateway serves the frontend + the chat bridge; CRUD is proxied to `:8015`. Data logic stays in the owner (volition), gateway is thin presentation+transport (the world-ai-broker pattern, [[PATTERNS]]). |
| Host | Web-gateway on **core-ai** (must be on the DDS bus for the bridge; `:8015` is local). |
| Auth | Single-user password → httpOnly session cookie, LAN-only. Not zero (it actuates the house + reads family data). |
| Frontend | FastAPI backend (consistent with intake) + a light frontend (htmx or Svelte); chat streams over SSE/WebSocket. |

## 1. Why extend the events service, not a new one
The Events subsystem is already exactly the shape a task store needs (`volition/src/volition/events/`):
- **Durable WAL sqlite** (`queue.py`: `EventQueue`, `state/events.db`) — survives reboot.
- **A FastAPI intake on `:8015`** (`intake.py`: `make_intake_app`) already serving CRUD-ish routes
  (`POST /reminders`, `POST/GET/DELETE /events`, `/mute`, `/briefing/*`), run by the volition
  service (`service.py` → uvicorn).
- **On the DDS bus + a fabric-managed systemd unit.** Reminders (`kind=reminder` in
  `pending_events`) already live here; `set_reminder` (voice) POSTs `:8015/reminders`.
A new `iris-tasks` service would duplicate the sqlite, the FastAPI app, the deploy unit, and
create a second store to keep in sync — for zero benefit. Extend. Tasks are a *different model*
from the pending-event queue (a todo list, not things that "fire"), so they get **separate
tables**, not shoehorned into `pending_events`. Revisit a split only if volition bloats.

## 2. Data model (new tables in `state/events.db`)
Reminders are unchanged (`pending_events`, `kind=reminder`; `fire.at` = due time; the UI renders
these as the "reminders" view via `GET /events?status=pending`). New:

```sql
CREATE TABLE IF NOT EXISTS projects (
  id          TEXT PRIMARY KEY,          -- new_id("proj")
  name        TEXT NOT NULL,
  tags        TEXT,                       -- JSON array, optional
  color       TEXT,                       -- optional
  created_at  REAL NOT NULL,
  updated_at  REAL NOT NULL,
  archived    INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS tasks (
  id          TEXT PRIMARY KEY,          -- new_id("task")
  title       TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'todo',   -- todo | doing | done
  notes       TEXT,
  due         REAL,                       -- optional epoch; NULL = no time anchor
  project_id  TEXT REFERENCES projects(id),
  source      TEXT NOT NULL DEFAULT 'web',     -- web | voice
  created_at  REAL NOT NULL,
  updated_at  REAL NOT NULL,
  completed_at REAL
);
CREATE INDEX IF NOT EXISTS idx_tasks_status  ON tasks(status);
CREATE INDEX IF NOT EXISTS idx_tasks_project ON tasks(project_id);
```
A `task` with a `due` MAY also enqueue a `pending_events` reminder (kind=reminder, dedup_key=task
id) so a due task fires like any reminder — reusing the delivery arbiter. One-way: task owns the
truth; the reminder is a derived fire.

Implementation: a `TaskStore` class beside `EventQueue` (same db file, same connection pattern —
`sqlite3`, `row_factory=Row`, WAL). Pydantic `Task`/`Project` models in contracts or volition.

## 3. HTTP API (extend `make_intake_app`)
Add to the `:8015` app (same service voice already uses):
```
GET    /tasks?status=&project=          → list
POST   /tasks           {title, notes?, due?, project_id?, source}   → create
PATCH  /tasks/{id}      {status?, title?, notes?, due?, project_id?}  → update
DELETE /tasks/{id}
GET    /projects  ·  POST /projects  ·  PATCH /projects/{id}  ·  DELETE /projects/{id}
GET    /conversations?limit=&before=    → list sessions (id, started_at, last_at, preview, turn_count)
GET    /conversations/{session_id}      → the turns (see §5)
```
Reminders reuse the existing `POST /reminders`, `GET /events?status=pending`, `DELETE /events/{id}`.

## 4. Fabric tools (shared state — voice sees the same list)
Add to tool-registry so voice can read/change tasks ("Iris, what's on my list?", "add milk to
groceries", "mark the taxes task done"). Pattern mirrors the existing `set_reminder`
(`tool-registry/src/tool_registry/backends/events.py` → POSTs `:8015`; declared in
`definitions.py`; dispatched in `dispatch.py`):
- `get_tasks(status?, project?)` → `GET :8015/tasks`
- `create_task(title, notes?, due?, project?)` → `POST :8015/tasks` (source="voice")
- `update_task(id_or_title, status?, …)` → `PATCH :8015/tasks/{id}` (resolve title→id server-side)
- `get_reminders()` → `GET :8015/events?status=pending`
Add `PER_TOOL_TIMEOUT_S` entries (~3s, like the other events tools). These land voice-side value
**before any UI ships** — build them first (see §7).

## 5. Conversation log + history + resume
### What exists vs what's missing
- `FeatureStream` (`iris/perception/feature_stream`) carries the **user text** (`asr.text`) +
  `turn_id` + `trace{session_id}`.
- `FusionDecision` (`iris/fusion/decision`) carries **Iris's reply** (`say: str`) + `turn_id`.
- `SPEECH_UTTERANCE` streams the reply as deltas (for token-streaming into the chat).
- `EPISODE_CLOSED` groups turns; `memory-substrate` stores consolidated/semantic memory.
Missing: a **verbatim, per-turn, display-oriented conversation store**. Memory is semantic, not a
transcript; Loki turn-traces are ops logs, not a queryable UI source; episodes are groupings.
### Add: a conversation-log subscriber
A small DDS subscriber (in the volition service, or a tiny sibling) that joins on `turn_id`:
`FeatureStream.asr.text` (user) + `FusionDecision.say` (Iris) + `trace.session_id` + timestamps →
writes a `conversations` / `conversation_turns` table in `state/events.db`:
```sql
CREATE TABLE IF NOT EXISTS conversation_turns (
  turn_id     TEXT PRIMARY KEY,
  session_id  TEXT NOT NULL,
  surface     TEXT NOT NULL,      -- voice | web
  user_text   TEXT,
  iris_say    TEXT,
  room        TEXT,
  created_at  REAL NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_conv_session ON conversation_turns(session_id, created_at);
```
This powers `GET /conversations*` — browsable history, per §3.
### Resume (pick up a voice conversation in text)
The web chat sets `trace.session_id` to the existing conversation's session (§6 bridge). The
fabric already keys memory recall by `session_id`, so recall carries over. **Caveat (real):** the
working-memory field (thalamus) is RAM-only/per-process — a resumed turn won't inherit live field
state. P1 resume = same `session_id` + recent `conversation_turns` replayed into the turn's context
(the gateway can prepend the last N turns as context, or curator can be taught to seed from the
conversation log). The field re-warms as the text conversation continues. Full session-keeper
(durable cross-surface field state) is post-P1 — the old "session-keeper" idea, [[BACKLOG]].

## 6. The text→fabric bridge (the keystone)
The fabric already has a clean DDS **entry** (`FeatureStream`) and **exit** (`FusionDecision.say` /
`SpeechUtterance`). The gateway just speaks DDS — it is a **bus peer**, like the eval harness
([[HISTORICAL-DECISIONS]] HD-6). It does NOT touch `fabric_brain`/`/decide_v2`/brain-`:8000`
(retired, [[ARCHITECTURE]]).

**Text in — synthesize a "web node" FeatureStream** and publish on `iris/perception/feature_stream`:
```
FeatureStream(
  turn_id   = new_id("turn"),
  asr       = AsrResult(text=<typed message>, confidence=1.0, asr_model="web"),
  wake      = WakeContext(...synthetic web wake...),
  affect    = AffectInfo(neutral)         # or text-derived later
  speaker   = SpeakerVerification(verified_id="matt", confidence=1.0),   # web session is authenticated
  spatial   = SpatialContext(origin_node="web", origin_room="web", occupied_rooms=[...]),
  trace     = {"session_id": <conversation session>, "surface": "web"},
)
```
Everything downstream (thalamus → curator → cortices → fusion → tool-registry) runs **unchanged** —
a web turn is just a turn whose "node" is `web`. Actuations work. Shared memory works.

**Text out — subscribe by `turn_id`:**
- `SPEECH_UTTERANCE` → stream reply deltas to the browser (SSE/WebSocket) for live typing.
- `FusionDecision.say` → the canonical final text (source of truth; reconcile if a barge-in
  correction supersedes an optimistic delta — same as voice).
No Kokoro, no gRPC speech sink — the gateway is the text sink.

**Gateway shape:** a small FastAPI + cyclonedds service (`DomainParticipant(domain_id=42)`, host
networking per [[PATTERNS]]). Routes: `POST /chat {session_id?, text}` (publishes the turn, streams
back the reply), plus it serves the frontend and reverse-proxies the `:8015` CRUD. One new unit on
core-ai. A prototype interface sketch belongs in the build (§7 step 4), kept minimal first.

**Node allowlists:** the streaming path gates nodes by allowlist in three services (conditioning /
fusion / speech-output, [[ARCHITECTURE]]). The `web` node bypasses the *speech* allowlists (no TTS)
but fusion must accept a `web`-origin turn — verify/extend `FUSION_EXECUTIVE_STREAMING_NODES`-style
gating so a `web` turn is voiced-as-text, not dropped. **This is the one integration risk to spike first.**

## 7. Build order (value lands before the UI ships)
1. **`TaskStore` + tables + `:8015` task/project routes.** Pure CRUD over the existing service.
2. **Fabric tools** (`get_tasks`/`create_task`/`update_task`/`get_reminders`). ⇒ *voice can manage
   the task list with no web page yet.* Ship + soak this alone.
3. **Conversation-log subscriber + `/conversations` routes.** History is now queryable.
4. **Web gateway — the bridge.** Spike the `web`-node FeatureStream round-trip FIRST (publish a
   canned text turn, read back `say`) to de-risk the fusion `web`-origin gating (§6). Then `/chat`
   streaming.
5. **Frontend** — task list by project, reminders view, conversations/history view, chat panel.
6. **Resume** — wire chat `session_id` to a selected conversation; replay recent turns as context.
7. **Auth + hosting hardening.**

## 8. Open decisions for Matt
- **Host:** web-gateway on core-ai (needs the DDS bus). OK, or stand up a dedicated ui host on the bus?
- **Auth:** single-user password + session cookie (LAN) — acceptable, or use an existing reverse
  proxy / add basic-auth? (Off-LAN access later ⇒ real auth + TLS, out of P1.)
- **Frontend stack:** htmx (server-rendered, minimal) vs Svelte/SvelteKit (SPA) vs React. Recommend
  htmx for CRUD + a small WS/SSE chat widget — least build tooling, self-hosted, fast.
- **Store growth:** OK to grow the volition events service into the "state service" (events + tasks
  + conversations), or prefer a sibling process sharing `state/events.db` to keep volition lean?
- **Voice-turn `web` gating:** confirm fusion will voice-as-text a `web`-origin turn (the spike in §7.4).

## 9. NOT in Phase 1
Research assistant, document ingestion, larger-model reasoning, Drive workspace (P3, hardware-gated).
Full durable session-keeper (cross-surface field state) — P1 resume uses session_id + replay.
Keep the schema/bridge from painting P3 into a corner; don't build P3.
