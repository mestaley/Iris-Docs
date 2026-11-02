# Architecture synthesis — current Iris vs a clean-sheet Jarvis (2026-11-02)

Compares two independent 2026-11-02 workers:
- [`REVIEW-CURRENT-ARCHITECTURE-2026-11.md`](REVIEW-CURRENT-ARCHITECTURE-2026-11.md) — grounded audit of the live system.
- [`CLEANSHEET-JARVIS-2026-11.md`](CLEANSHEET-JARVIS-2026-11.md) — a designer deliberately blind to the existing system and the thesis, briefed only on the goal + hardware + "minimal cloud."

Purpose (Matt): fresh insight on whether the project's own rules are limiting it. The clean-sheet was
told to ignore the thesis; the audit is pure fact. Where they **agree**, that's low-regret signal.

## 1. Where they converge (strong signal — the current design is right here)
A blind designer reinvented, nearly verbatim, several things Iris already does:
- **STT = parakeet-0.6b (CPU).** Identical. Both beat Whisper-large at ¼ the size for near-field commands.
- **TTS = Kokoro.** Identical (clean-sheet adds Piper-CPU as an always-available fallback — cheap, worth adopting).
- **Name-based actuation (verbs, not entity-IDs) + execute-then-verify against end-state + spoken correction.**
  The clean-sheet independently specified ADR-005 + settle-verify. Full validation of that line of work.
- **Device state queried LIVE, never prefetched/cached** — both call caching the hallucination root. Matches the existing rule.
- **Small, retrieved tool set for tool-calling adherence** — both land on it (Qdrant `iris_tools` today; grammar-constrained + small set in the clean-sheet).
- **Two-tier brain** (cheap reflex model for the common path + a bigger reasoner) — both have this shape.
- **Sell the 2×5080 + 4080** — 16GB ceiling unlocks no bigger model at 1–2 users. Validates the slim-rig plan.
- **Coral TPU** = wake-word 2nd-stage / acoustic events / presence — NOT core pipeline. Both agree.

## 2. Where they diverge (candidate changes)
1. **The cognitive-architecture layer.** Clean-sheet calls consolidation / affect / multi-agent "cortex"
   pluralism **net-negative**: latency + failure surface, and it *silently corrupts the simple preference
   data that actually matters*. The audit confirms the live reality — consolidation is hollow,
   auto-restarts after reboot, `/health` lies, episodic `recalled=0` every turn, no durable memory at all.
   **Both independent sources say this layer is costing more than it delivers.** This is the headline.
2. **Memory, rebuilt simply.** Clean-sheet: SQLite structured truth + `sqlite-vec` hybrid retrieval,
   additive/non-destructive, **no lossy background consolidation**. Replaces today's nuked-episodic +
   hollow-pgvector with something that works and can't corrupt prefs.
3. **Brain split.** Clean-sheet: ~4B reflex model resident for 70–80% of turns + Qwen3-30B-A3B MoE for
   reasoning. Current: hermes3:8b front door + qwen3-deep (also a 30B-A3B MoE) deep-tier-on-punt only.
   The clean-sheet's split is cleaner and would end the hermes-vs-14b indecision.
4. **Resilience topology.** Clean-sheet puts the dialog manager / HA bridge / resolver / timers / memory
   on the GPU-less node (a "brain stem") so basic smart-home control survives a GPU-box reboot; the GPU
   box is a pure inference appliance. Current: all of cognition is on the GPU box (core-ai) and all four
   VMs share one physical host = single blast radius (audit's words).
5. **The cloud question.** Clean-sheet: a *tiny* (low-single-digit-%), opt-in, budgeted, PII-scrubbed
   cloud escape hatch for the hard-knowledge tail — and argues 100%-local **dogma** hurts *only* there.
   Current direction (2026-11-02): all-local, OpenAI failover cancelled. This is the one real philosophy fork.

## 3. Verdict on "are my rules limiting development?"
- **Local-first: NOT limiting — keep it.** Both agree it's right for 95%+ of turns. The only cost is the
  hard-knowledge/reasoning tail, addressable with local web-search/RAG tools + (optionally) a bounded hatch.
- **The cognitive-architecture thesis (consolidation, affect, pluralistic cortex): YES, limiting.** This is
  where the fresh designer and the live audit agree the project is paying latency, complexity, and
  preference-corruption for little per-turn value. The honest answer: the local-first rule is fine; the
  *cognitive-architecture ambition* is the thing holding back a genuinely useful assistant. Spend cognition
  only where it pays per turn — resolver, retrieval, grounding, endpointing — and keep the control plane a
  boring, inspectable state machine.

## 4. What we can do — prioritized
### Tier 0 — urgent hygiene/safety (do regardless of direction)
- **Back up / push the deployed brain** before any GPU sale — the live system is unpushed code on one NVMe.
- **Confirm the lock-domain block is actually live** in the running world-ai (it's on unpushed branch
  `fix/block-lock-domain`, never deployed → NOT loaded; the deadbolt is safe today only because it isn't in
  mappings). Push + restart world-ai.
- **Mask the hollow consolidation stack** (it auto-restarts via `iris-fabric.target Wants=` and runs hollow;
  `disable` didn't stick across reboot).
- **Commit the audio-ai compose** — today's STT/TTS repoint that makes Iris hear/speak exists only on one VM disk, in no git repo.
- **Fix/disable the erroring Volition lighting arbiter** (`lighting fetch dining_room failed`).

### Tier 1 — low-regret convergent moves
- Finish slim-rig: retire voice-ai; sell 5080s/4080 (after the brain backup); reallocate freed cores/RAM.
- Initialize + wire the Coral (it's present but uninitialized/dead-weight) for wake-word 2nd-stage / acoustic events / presence.
- **Rebuild memory the simple way** (SQLite + sqlite-vec, additive) — retire the hollow consolidation for good.
- Add Piper-CPU as the TTS fallback.

### Tier 2 — strategic bets to decide
- Adopt the clean two-tier brain (resident reflex + MoE reasoner); settle the front-door model.
- Consider the "brain-stem on the audio node" resilience split.
- Decide the cloud-hatch question consciously (all-local vs a tiny bounded hatch for the hard tail).
- Decide how much of the cognitive-architecture layer to formally retire vs keep dormant.

## 5. Caveat
The clean-sheet is one model's opinion (no authority); the audit is grounded but was a single read-only
pass. Both are inputs for Matt's decision, not mandates. The convergences in §1 are the safest to bank;
§2–3 are the discussion.
