# 2026-11-02 — Architecture review: current-state audit + clean-sheet Jarvis + synthesis

## Executive summary
At Matt's request, ran two independent workers in parallel: (1) a grounded read-only audit of the live
architecture + hardware, (2) a clean-sheet "layman's Jarvis" design deliberately blind to the existing
system and the thesis (to test whether the project's own rules are limiting it), minimal-cloud, on the
real hardware. Both delivered; a synthesis compares them. **Key finding:** the blind designer reinvented
Iris's actuation layer and audio stack almost verbatim (validation), and both the clean-sheet *and* the
live audit independently flag the **cognitive-architecture layer (consolidation / affect / multi-cortex)
as net-negative** — hollow, auto-restarting, corrupting prefs, zero per-turn value. Verdict: local-first
is NOT the limiter (keep it); the cognitive-architecture ambition is. No code/live changes this session —
docs only. Several urgent hygiene/safety items surfaced (see follow-ups).

## Changes by repo
### iris-docs — branch `main`
- `design/REVIEW-CURRENT-ARCHITECTURE-2026-11.md` — **added** (worker 1): grounded audit of hosts, GPUs
  (2×5060 Ti both on core-ai; 5080s/4080 shelved; Coral USB TPU present but uninitialized/unused),
  cognition pipeline, models, tools/actuation, data stores, audio (all CPU on audio-ai), memory, Volition,
  measured latency, and a blunt works/broken/debt assessment.
- `design/CLEANSHEET-JARVIS-2026-11.md` — **added** (worker 2): first-principles design — two-tier local
  brain (4B reflex + Qwen3-30B-A3B MoE), parakeet STT / Kokoro+Piper TTS, microWakeWord + AEC + Silero,
  grammar-constrained verbs-not-IDs actuation with end-state verify, SQLite+sqlite-vec memory (no
  consolidation), deterministic proactivity, Coral for wake-word/acoustic/presence, bounded cloud hatch,
  4-rung degradation ladder, phased build.
- `design/ARCHITECTURE-SYNTHESIS-2026-11.md` — **added**: convergences (low-regret), divergences
  (candidate changes), the "are my rules limiting me" verdict, and a Tier-0/1/2 action plan.
- `changelog/2026-11-02-architecture-review.md` — **added**: this entry.
- (also tracked now: pre-existing untracked `design/ACTUATION-CONTRACT.md`, `design/SINGLE-GPU-CONSOLIDATION.md`.)

## Deploy / live state
- No live or code changes this session. Audit was read-only. Earlier-today audio repoint (STT/TTS) remains LIVE.

## Follow-ups / owed
- **Tier 0 (urgent, from the audit):** back up/push the deployed brain before any GPU sale (unpushed code,
  one NVMe); confirm the lock-domain block is actually loaded in the running world-ai (unpushed branch →
  not deployed); **mask** the hollow consolidation stack (auto-restarts via target Wants=); commit the
  audio-ai compose (lives only on the VM disk); fix/disable the erroring Volition lighting arbiter.
- **Owed pushes (automation cannot push):** wt-integration `integration/deployed-2026-09-28`
  (~11 commits 8507afe..fd2f441); world-ai `fix/block-lock-domain` (f848fe0) + restart; iris-conditioning
  `feat/prosodic-affect` (7 ahead); iris-docs `main`.
- **Decisions for Matt:** rebuild memory simply (SQLite+sqlite-vec) vs keep chasing consolidation; the
  two-tier brain + front-door model; brain-stem resilience split; all-local vs a bounded cloud hatch.
- STATE.md + ARCHITECTURE.md are materially stale (audit) — refresh when direction settles.
