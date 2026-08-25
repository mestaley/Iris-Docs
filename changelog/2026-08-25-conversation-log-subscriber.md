# 2026-08-25 — Conversation-log subscriber (web-surface keystone)

## Executive summary
Built the conversation-log subscriber inside iris-volition — the persistent verbatim
transcript the web surface will render and resume from. It joins each turn's user text
(`FeatureStream`) and Iris's reply (`FusionDecision`) by `turn_id` and writes them to the
existing `conversation_turns` table via `TaskStore.record_turn` (idempotent merge-upsert).
Surface (`voice` | `web`) is derived from `origin_node`, so voice and web turns share one
history. Committed on `feat/web-tasks-tools` and **synced to core-ai's working tree**; STAGED
pending `sudo systemctl restart iris-volition` (Matt). No contracts changed — subscribe-only,
read side of the frozen `FeatureStream`/`FusionDecision` topics. Full volition suite green
(129 passed, +3 new).

## Changes by repo

### iris-fusion-fabric — branch `feat/web-tasks-tools` (worktree wt-web-tasks) — commit 7595914
- `volition/src/volition/events/conversation_log.py` — **added.** `ConversationLogger`:
  caches per-turn `(session, surface, room)` from the FeatureStream in a bounded
  OrderedDict (512), records the user side on `on_feature_stream` and completes the reply
  side on `on_decision`. Every handler wrapped in try/except — a logging failure can never
  break a live turn. Surface = `web` if `origin_node` starts with `web`, else `voice`;
  session from `trace.session_id` with a `sess-<turn_id>` fallback.
- `volition/src/volition/service.py` — **changed.** Import `PERCEPTION_FEATURE_STREAM` +
  `FeatureStream`; import `ConversationLogger`; after the FusionBus is built, when a
  `task_store` exists, subscribe `PERCEPTION_FEATURE_STREAM`→`on_feature_stream` and
  `FUSION_DECISION`→`on_decision` and log "conversation-log subscriber enabled".
- `volition/tests/test_conversation_log.py` — **added.** 3 tests: voice turn joins
  user+reply, `web:`-origin marks surface web, reply with no cached FeatureStream still
  records (defaults). All green.

## Deploy / live state
- **STAGED on core-ai:** `conversation_log.py`, `service.py`, `test_conversation_log.py`
  scp'd to `/opt/iris-fusion-fabric/volition/…`; deployed-module import verified against the
  real installed bus/contracts. **Owed: `sudo systemctl restart iris-volition` (Matt)** — after
  restart, voice turns begin landing in `conversation_turns`.
- Nothing reverted. No other host touched.

## Follow-ups / owed
- **Origin push owed** (automation can't push): commit 7595914 on `feat/web-tasks-tools`.
- **Restart owed** (Matt): `iris-volition` on core-ai.
- **Next:** the web gateway — docker container (host networking, core-ai), Bootstrap+htmx
  frontend (task board + conversation history/resume + chat box), Google OAuth (authlib,
  restricted to matt@mattstaley.com), text→fabric bridge via the proven `web`-origin
  FeatureStream seam. Conversation resume replays recent `conversation_turns` into cortex
  context (ADR-002 / ISSUE-008).
