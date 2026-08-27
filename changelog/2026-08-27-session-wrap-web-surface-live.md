# 2026-08-27 — Session wrap: web surface live end-to-end (+ voice-turn fixes)

Cross-repo digest for the multi-day session (2026-08-25 → 27). Detailed per-topic entries:
`2026-08-25-conversation-log-subscriber.md`, `2026-08-25-web-gateway-scaffold.md`,
`2026-08-26-kitchen-lights-turn-fixes.md` (incl. the web-memory section). This is the summary.

## Executive summary
Built and shipped the **Iris web surface** (thesis surface #2) end-to-end and **live** on core-ai:
a browser page — task board, voice/web conversation history, and a **context-aware chat** — behind
Cloudflare Access/ZTNA at `iris.mattstaley.com`. It's a thin FastAPI gateway that proxies CRUD to
the volition intake (`:8015`) and bridges `/chat` to the live fabric over the brain-agnostic
Turn-Entry Contract (a DDS peer publishing synthetic `web`-origin FeatureStreams, reading back
`FusionDecision.say`). Confirmed in Matt's real browser: a multi-turn weather chat with working
turn-to-turn memory ("what about tomorrow" followed context). Along the way: the conversation-log
subscriber (the transcript store) and the curator `recent_conversation` slot (the memory mechanism)
went live, and a batch of **voice-turn cognition fixes** landed and were validated (spurious
deep-tier barge-in + filler on already-completed actuations; reflex tool-catalog scope). Everything
is committed locally; **all origin pushes are still owed** (automation can't push).

## Changes by repo

### iris-web-gateway — NEW repo, branch `main` (all owed to origin — remote not yet created)
- `06eea8e` scaffold — FastAPI factory, BrainAdapter Protocol + CurrentFabricAdapter (DDS peer) +
  FakeAdapter, `/api` CRUD proxy to `:8015`, `/chat` bridge, Bootstrap+htmx frontend, config, tests.
- `802db0b` SSE streaming — `/chat` stream=true (delta frames + one final), frontend consumes it;
  default-off `CORRECTION_GRACE_S` for deep-tier barge-in over web.
- `1572ffc` drop the trace-resume hack — curator owns recent_conversation; adapter passes only session_id.
- `aac675c` deploy + smoke scripts (`deploy.sh`, `smoke.sh`, `smoke_bridge.py`).
- `d4b97a0` auth for the Cloudflare-fronted https deploy — Secure-cookie derivation, `--proxy-headers`,
  Cf-Access identity passthrough; `AUTH.md` (Model A Google OAuth / Model B Cloudflare Access).
- `2e09b83` `DEPLOY.md` — one ordered deploy runbook.
- `3a8bf42` fix index 500 — modern Starlette `TemplateResponse(request, name, ctx)` + page-render test.
- `eee374f` handle the intake `{session_id, turns:[...]}` shape in resume + history-view (app.js).
- `813821e` cache-bust `app.js` (?v=2) so browsers pick up frontend changes.

### iris-fusion-fabric — branch `feat/web-tasks-tools` (worktree wt-web-tasks) — owed to origin
- `0ef5e6f` wire TaskStore into volition (`make_intake_app`).
- `5976d3d` voice-tasks fix — register task tools in curator catalog + prefetch get_tasks into ambient.
- `7595914` **conversation-log subscriber** — join FeatureStream+FusionDecision by turn_id → `conversation_turns`.
- `2d9161e` **curator recent_conversation slot** — fetch a session's recent turns by id, render a
  `RECENT CONVERSATION` block; mode `web` (voice path untouched).
- `e8c4714` reflex-catalog scope — `actuation_tools()` filtered to `category=='actuation'` (no task/reminder CRUD).
- `8eb5953` cortex-logical confirms "Done." on a fastpath-satisfied empty pass1 (not empty/uncertain).
- `66f6725` **web-memory fix** — recent_conversation fetch reads `.turns` from the intake response
  (was parsing the whole `{turns:[...]}` object as a list → always []).

### iris-fusion-fabric — branch `feat/punt-escalation` (worktree wt-punt) — owed to origin
- `bd7e666` suppress the proactive punt when a device actuation succeeded this turn (`_actuation_succeeded`).
- `133266a` fusion confirms → "Done." when the front door is empty on a reflex-completed turn
  (`_should_confirm_actuation`; reliable backstop for the ctx.fastpath race).

### iris-docs — branch `main` — owed to origin
- `issues/ISSUE-010-deep-mode-output-grooming.md` — deferred: deep-mode leaks entity_ids/chain-of-thought
  + qwen3-deep cold-start warm-keep.
- `STATE.md` + several `changelog/` entries — the running record above.

## Deploy / live state
- **LIVE on core-ai:** web-gateway container (`~/iris-web-gateway`, docker host-net, restart:unless-stopped,
  auth off = Cloudflare Access gates it); conversation-log subscriber (iris-volition); recent_conversation
  slot + reflex fix (iris-curator); punt-suppression + confirm-override (iris-fusion-executive);
  cortex-logical ack (iris-cortex-logical). All restarted + validated on real turns.
- **Proven:** the web→fabric→web bridge (grounded weather answer), the kitchen-lights turn ("Done.", no
  filler, no barge-in), and web-chat memory (Matt's browser, multi-turn weather follow-up).

## Follow-ups / owed
- **Origin pushes (all owed — automation can't push):** create the `iris-web-gateway` remote + push
  `main`; push `feat/web-tasks-tools` and `feat/punt-escalation` (iris-fusion-fabric) via bastion `ssh -A`;
  push `iris-docs`.
- **ISSUE-010** (deferred): deep-mode output grooming + qwen3-deep warm-keep; and whether the deep tier
  belongs on the voice path at all.
- **Security (Model B):** gateway origin binds `0.0.0.0:8030`, so a direct LAN hit bypasses ZTNA —
  acceptable "for now" (Matt); the lockdown (bind 127.0.0.1 / firewall) is a small compose/CMD change owed.
- **Minor, unactioned:** the reflex logged `room=basement` for a kitchen light — a room-hint/node-routing
  quirk, unrelated to the fixes.
- **Housekeeping:** gateway runs from `~/iris-web-gateway`, not `/opt` (matt's home; `/opt` needs root).
