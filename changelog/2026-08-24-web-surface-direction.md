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
Nothing deployed. The bridge spike touched the live bus read-only-ish (injected one non-actuating
query turn; no state changed). All builder work is new code on feature branches / worktrees,
awaiting Matt's review + deploy.

## Follow-ups / owed
- When P1 kicks off: a short design doc (EVENTS.md-style) resolving the open decisions in ADR-002/ISSUE-008 — tasks store (extend Events vs new `iris-tasks`), the text→fabric bridge shape, host + auth + frontend stack.
- Still carried from 2026-08-23: run the voiceprint re-enrollment; push owed origin commits; the queued deferred doc merges.
