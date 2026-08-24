# Web Surface — Phase 1 Design (tasks / reminders / conversations)

_Design doc for [ISSUE-008](../issues/ISSUE-008-web-surface-phase1.md), per
[ADR-002](../decisions/ADR-002-web-surface.md) + [ADR-003](../decisions/ADR-003-local-serial-cognition.md).
Resolves the open decisions with concrete schema, API, and the **Turn-Entry Contract** (§6).
Grounded in the live code (Events subsystem, tool-registry, the DDS turn path)._

> **Sequencing (updated 2026-08-24): PARALLEL build with the cognition rearch, NOT web-first.**
> The CRUD half runs fine on the current brain; the conversational/research half is only worth
> building on the NEW cognition — so **UI + cognition build together and land on the new brain.**
> The **Turn-Entry Contract (§6)** is the shared seam that makes this work: the UI builds against it
> now (validated end-to-end via Adapter A on the current fabric), and the new front-door lands
> behind the *same* contract (Adapter B) — cutover is a config flip. **Implementation-ready:** the
> vertical slice in §7.1 is new code buildable **tonight, alongside the live system** (no
> deploy/cutover tonight — that needs Matt's sudo).

## 0. Summary of decisions
| Question | Decision |
|---|---|
| Store | **Extend the existing volition events service** (core-ai, `:8015`, WAL sqlite `state/events.db`) with new `tasks`/`projects`/`conversations` tables — NOT a new `iris-tasks` service. |
| Data model | Reminders stay `pending_events` (kind=reminder); tasks + projects + conversation-log get their own tables. |
| Shared state | Voice and web both go through the volition intake (`:8015`) and new fabric tools — one owner, one store. |
| **Turn-Entry Contract (§6)** | A **brain-agnostic** `POST /chat` + internal `BrainAdapter` interface. Identical whether it routes to the current fabric (**Adapter A**: synthetic `FeatureStream` "web node" → `FusionDecision.say`, no TTS) or the new front-door (**Adapter B**). The cognition swaps in *behind* the contract — the seam of the parallel build. |
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

## 6. The Turn-Entry Contract — the brain-agnostic shared interface  ⭐ CRITICAL
This is THE seam of the parallel build (UI + cognition together). The web gateway and frontend speak
ONLY this contract; the cognition — **current fabric today, the new front-door next** — swaps in
*behind* it. Implementation conforms to the interface (architecture-is-sovereign,
[[HISTORICAL-DECISIONS]] HD-10). **The contract is identical for both brains; only the adapter
differs.** (This section is reconciled against the cognition worker's turn-entry spec — keep the
shapes below authoritative for the web side.)

### 6.1 Client-facing (web gateway HTTP)
```
POST /chat
  req:  { "session_id": str|null, "text": str, "surface": "web", "stream": bool }
        # session_id null → gateway allocates a new one (returned in the response)
        # stream=true → SSE/WebSocket: delta frames then one final; stream=false → one JSON reply
  resp (stream):  event: delta  data: {"turn_id","text_delta"}                      # 0..N
                  event: final  data: {"turn_id","session_id","say","actions":[…]}  # exactly 1
  resp (block):   { "turn_id","session_id","say","actions":[…] }
GET  /chat/sessions            → conversation list (proxies /conversations, §5)
GET  /chat/sessions/{id}       → the turns (history + resume source)
```
`say` = the canonical final reply (may supersede optimistic deltas via a barge-in correction — same
reconciliation as voice). `actions` = what Iris did, for UI echo ("✓ turned on the lights").

### 6.2 Internal turn-submission interface (the swap point)
```python
class ResumeContext(TypedDict):
    session_id: str
    recent_turns: list[dict]        # last N conversation_turns (user_text/iris_say), oldest→newest

class TurnDelta(TypedDict):   turn_id: str; text_delta: str
class TurnResult(TypedDict):  turn_id: str; session_id: str; say: str; actions: list; done: bool

class BrainAdapter(Protocol):
    async def submit_turn(
        self, text: str, session_id: str, resume: ResumeContext | None,
    ) -> AsyncIterator[TurnDelta | TurnResult]:
        # yields 0..N TurnDelta then EXACTLY ONE TurnResult(done=True)
        ...
```
The gateway holds ONE `BrainAdapter`, chosen by config. Everything else (HTTP, streaming, sessions,
CRUD, frontend) is identical across brains — moving from Adapter A→B is a **config flip, zero UI change**.

### 6.3 Adapter A — current fabric (DDS peer; buildable today)
A `DomainParticipant(domain_id=42)` bus peer (host networking, [[PATTERNS]]) — like the eval harness
(HD-6). Does NOT touch retired `fabric_brain`/`/decide_v2`/brain-`:8000`.
- **submit_turn →** publish a synthetic **"web node"** `FeatureStream` on
  `iris/perception/feature_stream`: `asr=AsrResult(text, confidence=1.0, asr_model="web")`,
  `speaker=SpeakerVerification(verified_id="matt", confidence=1.0)` (web session authenticated),
  `spatial=SpatialContext(origin_node="web", origin_room="web")`, `affect=neutral`,
  `trace={"session_id": session_id, "surface":"web"}`. If `resume`, seed `recent_turns` (§6.5).
- **yield ←** subscribe by `turn_id`: `SPEECH_UTTERANCE` → `TurnDelta`s; `FUSION_DECISION.say` →
  the final `TurnResult`. No Kokoro/gRPC — the adapter is the text sink.
- Downstream (thalamus→curator→cortices→fusion→tool-registry) runs UNCHANGED; a web turn is a turn
  whose node is `web`. Actuations + shared memory work.

### 6.4 Adapter B — new front-door (conforms to §6.2; built by the cognition rearch)
The new cognition's front-door exposes a turn entry (the cognition worker specs the exact internal
shape — reconcile there). It takes `text` + `session_id` + `resume` and yields the SAME
`TurnDelta`/`TurnResult`. Because §6.1/§6.2 are identical, the web surface needs **zero changes** to
move to Adapter B. This is how UI + cognition build in parallel and **land together on the new
brain**: the UI builds against the contract now (Adapter A validates the whole path end-to-end), the
new front-door lands behind the same contract, cutover = flip the adapter.

### 6.5 Session-attach / resume
`session_id` is the conversation identity, **shared across surfaces** — voice turns already carry it
in `trace.session_id`; web turns carry the same. A voice conversation from `/conversations` resumes
on web by submitting with its `session_id`. Resume = same `session_id` + `ResumeContext` (recent
`conversation_turns`). Honest caveat (both adapters): the working-memory field is RAM-only/per-process,
so a resumed turn doesn't inherit live field state — the adapter seeds from `recent_turns` + memory
recall (keyed by session_id, which the fabric already does); the field re-warms as the chat continues.
Durable cross-surface field state = post-P1 (session-keeper, [[BACKLOG]]).

### 6.6 The one integration risk — spike FIRST
The streaming path gates nodes by allowlist across conditioning/fusion/speech-output
([[ARCHITECTURE]]). A `web` node bypasses the *speech* allowlists (no TTS) but **fusion must accept
and voice-as-text a `web`-origin turn**, not drop it. Spike this before building the gateway out:
publish one canned `web` FeatureStream, confirm a `FusionDecision.say` returns. **Caution:**
publishing to the LIVE bus triggers a REAL turn (a command would actuate) — spike with a
non-actuating query ("what's the weather") and a dedicated test session; the bus is shared even
though there's no cutover tonight.

## 7. Build order

### 7.1 Vertical slice — buildable TONIGHT (new code, alongside live, NO deploy)
All new modules/files; nothing wired into a running service, so no restart/sudo needed tonight
(wiring + deploy is Matt's, later). Each is unit-testable in isolation against a scratch sqlite.
1. **`TaskStore` + schema + Pydantic `Task`/`Project` models** — new module beside `EventQueue`
   (same `sqlite3`/WAL pattern). Unit tests against a temp db. *(no deploy)*
2. **Task/project router** — a new `APIRouter` (the `/tasks` + `/projects` routes, §3) that will
   mount into `make_intake_app` later; test standalone by spinning the app on a scratch db. *(no deploy)*
3. **Fabric tools** — `get_tasks`/`create_task`/`update_task`/`get_reminders`: new methods on the
   events backend + entries in `definitions.py` + `dispatch.py` + `PER_TOOL_TIMEOUT_S`. Unit-test
   like the existing tool-registry tests. *(no deploy — dispatch changes go live only on restart)*
4. **Turn-Entry Contract skeleton (§6)** — the new web-gateway service: FastAPI `POST /chat`, the
   `BrainAdapter` Protocol, and **`CurrentFabricAdapter`** (DDS peer, domain 42). Code-complete;
   `stream=false` blocking mode first (simplest), SSE later. *(new service, not started as a unit)*
5. **The `web`-origin spike (§6.6)** — the ONE live-touch checkpoint: from the new gateway, submit a
   **non-actuating** turn ("what's the weather") on a dedicated test session; confirm a
   `FusionDecision.say` returns and that fusion doesn't drop the `web` node. If it drops, that's the
   first real fix (fusion `web`-origin acceptance). **Careful — shared live bus.**

This slice proves the whole path (store → tools → contract → real brain round-trip) end-to-end on
the CURRENT fabric, which is exactly what de-risks landing Adapter B on the new brain.

### 7.2 Full order (parallel with the cognition rearch; lands on the new brain)
6. **Conversation-log subscriber + `/conversations` routes** (§5) — history queryable.
7. **Frontend** — tasks by project, reminders, conversations/history, chat panel (SSE streaming).
8. **Resume** (§6.5) — wire chat `session_id` to a selected conversation; seed `recent_turns`.
9. **Adapter B** — swap `CurrentFabricAdapter` → the new front-door behind the SAME contract
   (config flip). UI + cognition converge here; this is the "land on the new brain" moment.
10. **Auth + hosting hardening.**
Deploy of anything in 1–10 (wiring into volition/tool-registry, starting the gateway unit) is a
Matt-sudo step; tonight is authoring + isolated tests + the one careful spike.

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
