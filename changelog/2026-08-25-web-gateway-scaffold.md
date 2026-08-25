# 2026-08-25 — Web-gateway scaffold (the web surface)

## Executive summary
Scaffolded `iris-web-gateway`, a new repo: the browser surface for Iris. It's thin
presentation+transport — owns no data. `/api/*` proxies CRUD to the volition events intake
(`:8015`, which already owns tasks/projects/reminders/conversations); `/chat` bridges to the
live cognition via the brain-agnostic **Turn-Entry Contract** (WEB-SURFACE.md §6). The live
adapter (`CurrentFabricAdapter`, Adapter A) is a long-lived DDS peer on domain 42 that
publishes a synthetic `web`-origin FeatureStream and reads back `FusionDecision.say`, matching
the proven 2026-08-24 bridge spike. A `FakeAdapter` runs the entire HTTP/frontend/session stack
with no DDS and no Google, so it tests anywhere (5 tests green on bastion). Google OAuth
(authlib, iris@ G-Suite) gates a single allowlisted email. Bootstrap+htmx frontend (task board,
conversation history, chat). The live adapter's FeatureStream shape was validated against the
real contracts on core-ai. **Nothing deployed** — vendoring contracts+bus, the host-net docker
build, and the careful web-origin spike are Matt's sudo step. Earlier in the same session the
conversation-log subscriber went LIVE (separate changelog).

## Changes by repo

### iris-web-gateway — NEW repo, branch `main` — commit 06eea8e
- `pyproject.toml` — **added.** hatchling package `web_gateway`; deps fastapi/uvicorn/httpx/
  jinja2/authlib/itsdangerous/pydantic-settings; `dev` extra = pytest(+asyncio). Fabric packages
  are NOT registry deps — vendored into the image only.
- `src/web_gateway/config.py` — **added.** `WEB_GATEWAY_*` settings: adapter (fabric|fake),
  intake_url, turn_timeout, resume_turns, Google OAuth creds, allowed_emails, session_secret.
- `src/web_gateway/app.py` — **added.** FastAPI factory; lifespan-managed adapter.start()/aclose()
  + shared httpx client; SessionMiddleware; mounts auth/pages/proxy/chat routers + static; /health.
- `src/web_gateway/adapters/base.py` — **added.** `BrainAdapter` Protocol + ResumeContext/
  TurnDelta/TurnResult — THE seam (Adapter A today, Adapter B behind the same contract later).
- `src/web_gateway/adapters/fabric.py` — **added.** `CurrentFabricAdapter`: one long-lived
  FusionBus peer (domain 42), subscribes FUSION_DECISION + SPEECH_UTTERANCE once, dispatches by
  turn_id to per-turn asyncio.Queues via call_soon_threadsafe (DDS handlers run off-loop);
  `_build_feature_stream` mirrors the eval `build_feature_stream` (web origin_node/room, matt
  speaker, neutral affect, exact wake); yields deltas then one final TurnResult; per-turn timeout.
- `src/web_gateway/adapters/fake.py` — **added.** Echo adapter (2 deltas + 1 final) for dev/CI.
- `src/web_gateway/adapters/__init__.py` — **added.** `build_adapter()` — the one brain-choice point.
- `src/web_gateway/proxy.py` — **added.** `/api/{path}` → intake `:8015`, allowlisted prefixes
  (tasks/projects/conversations/reminders/events); 502 on intake unreachable; require_user dep.
- `src/web_gateway/routes/chat.py` — **added.** POST /chat (blocking now, SSE stubbed),
  GET /chat/sessions[/{id}] (proxy /conversations); allocates session_id; seeds ResumeContext
  from the intake's recent turns.
- `src/web_gateway/routes/pages.py` — **added.** HTML shell (redirects to /login when signed out).
- `src/web_gateway/auth.py` — **added.** Google OIDC (authlib), /login /auth/callback /logout,
  email-verified + allowlist check, httpOnly signed session; AUTH_ENABLED=false opens it for dev.
- `src/web_gateway/templates/{base,index}.html`, `static/app.js` — **added.** Bootstrap+htmx UI:
  tasks (add/toggle via /api), conversations (list + open history), chat panel (blocking /chat).
- `Dockerfile` — **added.** python:3.12-slim; installs the web stack then vendored contracts+bus;
  CMD uvicorn :8030. `docker-compose.yml` — **added.** host networking, env_file, CYCLONEDDS_URI.
- `scripts/vendor-fabric.sh` — **added.** copies contracts+bus from /opt/iris-fusion-fabric into
  ./vendor (gitignored) for the image build. `.env.example`, `.gitignore`, `README.md` — **added.**
- `tests/{conftest,test_gateway}.py` — **added.** 5 tests (health, chat blocking, session
  alloc/reuse, fake-adapter delta/final contract, proxy allowlist). Green.

### iris-docs — branch `main`
- `STATE.md` — **changed.** V2 progress: conversation-log subscriber LIVE; web-gateway scaffolded.
- `changelog/2026-08-25-web-gateway-scaffold.md` — **added** (this file).

### iris-web-gateway — branch `main` — commit 802db0b (SSE streaming)
- `routes/chat.py` (already emitted SSE) + `static/app.js` — **changed.** Frontend now POSTs
  `stream=true` and consumes the `text/event-stream` via fetch+ReadableStream: an SSE frame parser
  fills the reply bubble progressively on `event: delta` and replaces it with the final `say` on
  `event: final` (say supersedes deltas — voice's reconciliation).
- `adapters/fabric.py` — **changed.** Added `_await_correction`: after the front-door FusionDecision
  lands, if `correction_grace_s > 0`, surface the front-door answer as a delta then hold the turn
  open for a deep-tier `SpeechUtterance(cause="correction")` (published node-agnostically by
  deep-mode) to supersede it — the deep answer barges in over the web reply, exactly as voice.
- `config.py` + `.env.example` — **changed.** New `WEB_GATEWAY_CORRECTION_GRACE_S` (default 0 = off,
  front-door reply is final/fast; ~15 catches the ~13s deep answer on web).
- `tests/test_gateway.py` — **changed.** +SSE framing test (delta frames + exactly one final). 6 green.
- Honest limit documented: cortex doesn't token-stream yet and fusion emits per-turn SpeechUtterance
  only for allowlisted voice nodes, so a plain web turn yields one final today; progressive tokens
  flow through this same path unchanged once logical-streaming lands.

## Deploy / live state
- **Nothing deployed for the gateway.** Code-complete + unit-tested on the fake adapter; the live
  adapter's contract shapes validated against core-ai. Deploy = Matt's sudo step:
  `scripts/vendor-fabric.sh` → `docker compose build && up -d` on core-ai (host networking) →
  the web-origin spike (non-actuating query on a test session; confirm fusion voices the web turn).
- The conversation-log subscriber IS live (iris-volition restarted 2026-08-25) — separate entry.

## Follow-ups / owed
- **Origin push owed** (automation can't push): iris-web-gateway `main` 06eea8e (new repo — needs
  creating on the remote first); volition `feat/web-tasks-tools` 7595914; iris-docs.
- **Matt sudo:** vendor + docker build + `docker compose up -d` on core-ai; run the web-origin spike.
- **Google OAuth setup:** create a Web OAuth client under the iris@ project; redirect URI
  `${WEB_GATEWAY_BASE_URL}/auth/callback`; drop client id/secret + a random session secret in `.env`.
- **Next build:** SSE streaming (adapter deltas → frontend); the curator `recent_conversation` slot
  for full prompt-level resume (continuity today is via shared session_id + memory recall);
  vendor Bootstrap/htmx for a fully offline LAN; Adapter B swap when the new front-door lands.
