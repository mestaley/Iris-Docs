# 2026-08-23 — Iris whole-system review, ADR-001 implementation, docs consolidation

## Executive summary
Matt asked whether the system had grown too complex ("continue or nuke half?"). A 4-agent review
concluded: **keep the streaming trunk, prune the vestigia** — the felt regression was a handful of
specific issues, not the architecture. Recorded as **ADR-001** and implemented the same day.
**Deployed live:** speaker-gate threshold 0.35→0.28 (audio-ai), creative outcome-grounding + a
consolidation write hard-gate (core-ai). **Reverted:** the multi-intent dual-rail routing change —
it over-prefetched in production (caught in the first soak). **Banked:** voiceprint re-enrollment
(scripted; the real fix for Matt still repeating himself at cos ~0.26). Also stood up the
**`iris-docs` SSOT repo** and **merged Matt's legacy `/opt/repos/Architecture/` docs** into it
(THESIS, PRINCIPLES, PATTERNS, historical decisions, ~25 backlog items). New issues filed:
ISSUE-001…007. This changelog directory + workflow was created at the end of the session.

## Changes by repo

### iris-docs (master) — NEW repo, the cross-repo single source of truth
- `README.md`, `STATE.md`, `ARCHITECTURE.md` — **added** — SSOT scaffold + live-verified architecture map.
- `decisions/ADR-001-continue-vs-nuke.md` — **added** — the keep-trunk/prune decision + implementation log.
- `decisions/HISTORICAL-DECISIONS.md` — **added** — 11 still-in-force decisions ported from legacy `DECISIONS.md`, 5 contradicted-by-live flagged. (`f48ee29`)
- `THESIS.md` — **added** — canonical thesis (clean live copy; legacy `Architecture/Thesis.md` was corrupted) + status footnote. (`f48ee29`)
- `PRINCIPLES.md` — **added** — Jarvis target, working principles, what-Iris-is-NOT, working agreement. (`f48ee29`)
- `PATTERNS.md` — **added** — ~20 engineering patterns ported + 3 stale examples annotated. (`f48ee29`, `4c97934`)
- `issues/ISSUE-001…007` — **added** — speaker-gate recall, multi-intent collapse, consolidation dead-end, dead-code, cortex-blind-to-tool-results, honest-backend-errors, Path-2 actuation.
- `BACKLOG.md` — **changed** — merged ~25 legacy items (deduped), 5 ISSUE candidates, Davis flag (resolved). (`f48ee29`, `4c97934`)
- `changelog/` — **added** — this directory + its `README.md` (workflow spec) + this entry.

### iris-fusion-fabric (feat/events-queue) — STAGED/LIVE on core-ai
- `dream-mode/src/dream_mode/service.py` — **changed** — consolidation write hard-gate: `on_trigger` forces shadow unless `DREAM_CONSOLIDATION_WRITE_ENABLED=true`. LIVE. (`13d66b5`)
- `dream-mode/tests/test_service.py` — **changed** — +3 guard tests (7/7 green). (`13d66b5`)
- `cortex-creative/src/cortex_creative/prompt.py` — **changed** — grounding guardrail: acknowledge actions generally, never assert an unseen outcome (Spotify track / device state). LIVE. (`34f2400`)
- `thalamus/src/thalamus/prompt.py` — **changed then REVERTED** — dual-rail prefetch independence; over-prefetched live, reverted same day. Net: no change. (`2ecf318` → `cc116f3`)

### iris-conditioning (feat/prosodic-affect) — LIVE on audio-ai
- `docker-compose.yml` — **changed** — `SPEAKER_THRESHOLD` 0.35→0.28 (asr-consumer). LIVE (container recreated). (`b086e59`)
- `scripts/reenroll_voiceprint.sh` — **added** — sudoless voiceprint re-enrollment tool (backup→enroll→restore→verify). Deployed to audio-ai, not yet run. (`11e5994`)
- `docker-compose.enroll.yml` — **added** — enroll-mode compose override used by the script. (`11e5994`)

### world-ai (feat/presence-fusion)
- `app/config.py` — **changed** — removed 3 dead Spotify credential fields (dead since the Music Assistant refactor; zero reads). Committed, not yet deployed (config-only, no runtime effect). (`68369bf`)

_(Carried in from earlier in this session, still owed to origin: iris-fusion-fabric `40a20f1`
doubled-domain HA service fix; iris-conditioning `261e4db` leading-homophone rescue; world-ai
`12561ec` basement radar sensor swap. All deployed live earlier; see the address-ASR memory.)_

## Deploy / live state
- **LIVE now:** speaker threshold 0.28 (audio-ai); creative grounding + dream-mode write-gate (core-ai, restarted 15:20 UTC); `iris-dream-mode-trigger.timer` disabled.
- **Reverted:** thalamus dual-rail (over-prefetch); thalamus restarted on the reverted prompt.
- **Committed, not deployed:** world-ai Spotify-creds prune (config-only).
- **Staged tool, not run:** the re-enrollment script on audio-ai.

## Follow-ups / owed
- **Owed origin pushes** (automation can't push): iris-fusion-fabric {`13d66b5`,`34f2400`,`cc116f3`,`40a20f1`}; iris-conditioning {`b086e59`,`11e5994`,`261e4db`}; world-ai {`68369bf`,`12561ec`}.
- **Run the re-enrollment** in a quiet room (ISSUE-001) — the durable recall fix.
- **Redo multi-intent** with an eval harness + tune the working-memory field (ISSUE-002).
- **ISSUE-006** (honest backend errors), **ISSUE-007** (Path-2 actuation), and the 4 other promoted candidates in BACKLOG.
- Deferred doc merges: DDS topic-table reference appendix, PROMPTS.md re-derived from live code, Core-Substrate-as-history.
