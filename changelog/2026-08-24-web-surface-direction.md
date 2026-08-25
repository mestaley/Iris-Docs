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

## Evening — V2 declared, implementation kicked off (parallel)
- **Iris V2 declared** (STATE.md "Versioning"): V1 = two-cortex parallel-pluralism voice fabric;
  V2 = serial-spine punt/escalation cognition (ADR-003) + multi-surface web (ADR-002). The
  cognition break + the interface break together = V2.
- **Bridge spike PASSED (live):** a synthetic `web`-origin FeatureStream was accepted and
  voiced-as-text by live fusion (`say="It's 69 degrees and partly cloudy."`, origin_node=web, 3.4s)
  — the one flagged integration risk is resolved; the BrainAdapter seam is proven. Ran via the eval
  harness on core-ai (scratch `spike_web.py`); non-actuating query, no audio to any device.
- **Two implementation builders — BUILT + TESTED + COMMITTED** (worktree-isolated, feature
  branches off `iris-fusion-fabric`, nothing deployed). NB: volition + tool-registry + the new
  web-gateway are all **components of the iris-fusion-fabric monorepo** (volition is not a separate repo).
  - **Cognition** `feat/punt-escalation` (`4fa64d6`, worktree `wt-punt`): `PuntRequest` contract +
    `iris/punt/requested` topic (frozen contracts untouched) · reactive shadow-punt in fusion
    (log-only default, `punt_publish=False`) · isolated action-biased front-door `decide()` prototype.
    Tests: front-door 9/9, contract 3/3, bus 1/1, shadow-punt 3/3. = migration step 1 (measure punt freq).
  - **Web** `feat/web-tasks-tools` (`33b48f3`, worktree `wt-web-tasks`): volition `TaskStore` +
    tasks/projects/conversation_turns tables + `/tasks`/`/projects`/`/conversations` routes (additive
    `task_store` param, default None = unchanged) · 4 fabric tools (get/create/update_task, get_reminders,
    mirroring set_reminder) · new `web-gateway` component (BrainAdapter Protocol + CurrentFabricAdapter
    using the proven spike pattern + `POST /chat` blocking/SSE). Tests: 27 new green, no regressions.
  - **Heads-up:** the web builder overlaid its 8 changed files onto core-ai `/opt/iris-fusion-fabric`
    to run tests (additive, inert until restart) — do a clean branch deploy; `git checkout .` there
    first if you want the deployed tree pristine. (Cognition builder used a temp copy — deployed tree untouched.)
- **New guidance captured:** Matt prefers Iris bias toward ACTION/assertiveness over conservative
  ask-first — "I can touch a light switch," a false-positive is cheap. Memory `iris-bias-toward-action`
  + ISSUE-007 updated (Path-2 leans assertive). Loosen Volition/actuation conservatism generally.

## Deploy / live state
- **Both tonight-slices STAGED onto core-ai + validated (2026-08-24), inert until Matt restarts:**
  - **Tasks:** volition (TaskStore wired into `service.py` — branch commit `0ef5e6f`) + tool-registry source synced; validated — volition imports clean, all 4 task tools registered. Components are editable installs, so source overlay is picked up on restart. **Restart: `iris-volition` + `iris-tool-registry`.**
  - **Punt:** contracts/bus/fusion-executive source synced; fusion-executive venv refreshed (`uv sync --reinstall-package` recopied contracts+bus with `PuntRequest`/`iris/punt/requested`). Validated via the deployed `.venv/bin/python` (systemd's exact launcher): imports OK, `punt_shadow_enabled=True`, `punt_publish=False`. **Restart: `iris-fusion-executive`** → `journalctl -u iris-fusion-executive | grep SHADOW-PUNT`.
  - Nothing hit the live bus; running services unaffected until restart (only fusion-executive's venv was refreshed; other services keep their own contracts copies).
- **Caveats / proper-deploy follow-ups:** contracts left at 0.22.0 with new content (version drift — bump on the clean branch deploy); `uv sync` pruned dev deps from fusion's venv (harmless at runtime; `uv sync --dev` to run tests there); the deployed tree now carries worktree-branch overlays — do a clean `git checkout` of `feat/punt-escalation` + `feat/web-tasks-tools` for the real deploy.
- Bridge spike earlier touched the live bus read-only-ish (one non-actuating query turn; no state changed).

## Follow-ups / owed
- When P1 kicks off: a short design doc (EVENTS.md-style) resolving the open decisions in ADR-002/ISSUE-008 — tasks store (extend Events vs new `iris-tasks`), the text→fabric bridge shape, host + auth + frontend stack.
- Still carried from 2026-08-23: run the voiceprint re-enrollment; push owed origin commits; the queued deferred doc merges.

## 2026-08-25 continuation — voice-tasks bug fixed (found in live testing)
Live test surfaced: "add X to my list" confabulated success (nothing created) and "what's on my list"
hallucinated a grocery list. Root cause = a two-catalog gotcha: the builder registered the 4 task tools
in `tool-registry/definitions.py` (dispatch) but NOT in `curator/tool_catalog.py` (the cortex-facing
catalog — its own docstring says you must update both). So the cortex never saw the tools → confabulated.
Fix (`feat/web-tasks-tools` `5976d3d`, curator only): (1) added the 4 defs to `curator/tool_catalog.py`
→ cortex can call `create_task`; (2) curator now prefetches `get_tasks` into AMBIENT on list-query turns
(`_wants_tasks` keyword trigger) → creative owns the voice and reads only ambient, so it now grounds on
the real list instead of inventing one. Staged+validated on core-ai (catalog 15→19, `_wants_tasks` correct);
restart `iris-curator` only. Store/API/routes were never the problem (proven via direct API round-trip).
Lesson: adding a fabric tool = update BOTH catalogs; query tools must reach ambient, not just be callable.

## 2026-08-25 — Cognition migration step 2: retire creative (ADR-003), staged
Matt chose "retire Gemma → grounded voice" (then wants the MoE on the freed GPU1 next).
Implemented as a flag: wired the existing `enable_creative` (was only in the dead legacy app.py)
into the LIVE `ready_to_fire` — when off, a normal turn fires on the logical response alone
(the fire path already synthesizes from logical when creative is absent, lines ~584). Grounded
voice owns every turn → kills the ISSUE-005 confabulation at the root. `feat/punt-escalation`
`4409158`; +2 tests green; staged on core-ai, INERT (default enabled).
ACTIVATION (Matt sudo): add `FUSION_EXECUTIVE_ENABLE_CREATIVE=false` to /etc/iris-fabric/fusion-executive.env
→ restart iris-fusion-executive → stop iris-cortex-creative (frees GPU1 / gemma3:12b for the MoE).
TRADEOFF: logical does NOT stream (only creative did), so first-audio goes ~0.2s → ~2.5-3s (grounded
but slower start). Follow-up if it bothers: add SpeechUtterance streaming to cortex-logical.
NEXT (queued): stand up the deep MoE tier on the freed GPU1 (migration step 4).

## 2026-08-25 — Deep tier: model + service (migration step 4, in progress)
Matt: retire Gemma worked → stand up the MoE deep tier on freed GPU1.
- **Model:** ollama has no q3 for qwen3:30b-a3b (smallest q4_K_M=19GB, won't fit 16GB). Decision (Matt):
  import a Q3_K_M GGUF. Downloading `bartowski/Qwen_Qwen3-30B-A3B-Thinking-2507-GGUF` Q3_K_M (14.1GB,
  fits) → will `ollama create qwen3-deep` on :11435 (GPU1). Thinking variant = the deliberator.
- **phi3.5 → GPU0** (frees GPU1): thalamus is env-driven — `THALAMUS_OLLAMA_URL=http://127.0.0.1:11434`
  + restart iris-thalamus (Matt).
- **`iris-deep-mode` service BUILT + tested** (`feat/deep-mode` `a28e6f0`, worktree wt-deep-mode, off
  feat/punt-escalation): SCHED_OTHER DDS peer, caches CONTEXT_CURATED by turn_id (LRU 256), subscribes
  iris/punt/requested, runs the configurable deep model, **barges in via a direct SpeechUtterance**
  (cause=correction, interrupt_prior) — the barge-in-after-fire risk RESOLVED (fusion's tombstone is on
  the coordinator, not the speech topic; speech-output consumes utterances regardless). 6/6 tests. Gated
  OFF (`DEEP_MODE_ENABLED=false`). Live barge-in spike recommended before enabling.
- **Gap to fire:** only reactive `verify_failed` punts exist + `punt_publish=false` → deep tier idles
  until a punt fires. For a first live test, manually publish a PuntRequest (web-spike style). Proactive
  punt signals = migration step 5.
- **Fold analysis** (ADR-003 open tunable): two SEPARATE folds — thalamus→14b (retires phi, win) vs
  7b-reflex→14b (frees world-ai GPU but slows "lights on" ~0.5→2.5s; gate = a named use for that GPU).

## 2026-08-25 — DEEP TIER PROVEN END-TO-END (live spike) 🎯
Model finalized: **instruct-2507** (not thinking — thinking was too slow/verbose, timed out >120s uncapped)
+ **ChatML template** (raw-GGUF import defaulted to `{{ .Prompt }}` → garbage prefix; re-created FROM the
blob with an explicit Qwen3 template + `<|im_end|>` stop) + **num_predict cap** (`DEEP_MODE_MAX_TOKENS=400`,
deep_client, commit 0d358e6). qwen3-deep on GPU1 (:11435): warm answers in ~0.4s, ~160 tok/s, 15.3/16GB.
**Live spike (a genuinely hard turn):** front-door gave a generic answer; deep-mode deliberated ~13s and
BARGED IN (`cause=correction, interrupt_prior=True`) with a deeper, GROUNDED answer that reframed the
problem AND pulled Davis's preschool date + Matt's work calendar out of the CuratedContext. The whole
escalation pipe — reflex → 14b front door → 30B-A3B deep MoE, woven by barge-in — works end-to-end.
Tunables: bump the 400 cap for richer deliberation; the ~13s wait is what the punt-bridge (generated
holding line) covers. Still gated behind manual punts (proactive punt signals = step 5).
