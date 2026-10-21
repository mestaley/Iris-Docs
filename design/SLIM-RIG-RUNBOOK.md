# Slim-rig migration — runbook

Companion to [`SINGLE-GPU-CONSOLIDATION.md`](SINGLE-GPU-CONSOLIDATION.md) (the decision). Execution checklist
for the 3-box end state. Downtime acceptable (Matt: "fine to brick Iris a few days"). **Do not sell any GPU
until Phase 6 passes** — it's the only irreversible step.

## Target
| Node | Role | GPUs | Storage |
|---|---|---|---|
| pve-03 | **Iris** (core-ai fabric + voice-ai audio + world-ai) | 5060 Ti (LLM+embed) · 3060 Ti (STT+TTS) | +1TB NVMe (local Qdrant) |
| pve-01 | **Infra** (monitor, DNS, bastion, …) | — | — |
| pve-02 | **Plex** | A400 (moved from pve-01) | existing ZFS pool |
Deep/hard-turn tier → **cloud API** (escalation-only). Front door → an **8B** (Hermes-3-8B / qwen3-8B); qwen2.5:14b retired.

## Phase 0 — prep while up ✅ (automation, 2026-10-15)
- [x] Qdrant snapshots (iris_memory/mappings/tools) → `pve-03:~/iris-qdrant-backup/` (4.5MB).
- [x] Full code backstop bundle (`iris-fabric-integration-FULL.bundle`, all branches) → same dir.
- [ ] **Owed origin push** (11 commits, `8507afe..fd2f441` on `integration/deployed-2026-09-28`) — Matt, from bastion `ssh -A`:
  `cd /opt/repos/wt-integration && git push origin integration/deployed-2026-09-28` (+ world-ai `fix/actuation-settle-verify`, iris-docs).
- [x] **STT test set generated** (8 Kokoro-synth clips of real commands + ground-truth text) → `pve-03:~/iris-qdrant-backup/stt-test/`. Current STT is **NVIDIA Canary-1B-v2 ONNX** (`wyoming_iris_stt.py`, :10300, ~8.2GB, **containerized** — onnx-asr not on host), TTS Kokoro (:8880, ~1.2GB). 8GB audio card needs STT ≤ ~6GB (leave room for Kokoro). **WER measurement deferred to Phase 5** on the real 3060 Ti (higher-fidelity than a live-box eval, and gated by the Phase-6 sell-gate). Candidates, ranked by confidence: **canary-1b-v2 `quantization=int8`** (~4GB, same model → minimal WER delta — preferred) · **parakeet-tdt-0.6b-v2** (~2–3GB, top English WER). Both expected to fit; risk is low.
- [ ] Stage (don't deploy) software branches: **ChatGPT/OpenAI** deep-tier failover (punt→OpenAI API, escalation-only), curator Qdrant URL→localhost, cortex model pin→8B, STT→chosen trimmed model.

## Phase 1 — infra pve-02 → pve-01  (Matt; pve-01 ready)
- [ ] Migrate monitor (Grafana/Loki/Prom), DNS, bastion, remaining infra LXCs to pve-01.
- [ ] **Verify DNS + monitoring live on pve-01 before touching pve-02** — the rebuild needs name resolution.

## Phase 2 — retire memory-ai  (Matt)
- [ ] Final Qdrant snapshot if changed since Phase 0 (the data rides to the pve-03 NVMe; Phase-0 copy is the baseline).
- [ ] Stop memory-ai + memory-substrate. (Consolidated/pgvector tier is hollow — drop, don't migrate; rebuild later if wanted.)
- [ ] pve-02 now clear.

## Phase 3 — pve-02 → Plex  (Matt)
- [ ] Move the **A400 from pve-01 → pve-02** (passthrough to the Plex LXC). Verify NVENC in-guest (`ffmpeg … h264_nvenc`).
- [ ] Plex LXC on the existing ZFS pool.

## Phase 4 — pve-03 GPU swap (brick window)  (Matt)
- [ ] Graceful shutdown of core-ai / voice-ai / world-ai.
- [ ] Pull **both 5080s + the 4080** → shelf (NOT sold yet). Keep the 5060 Ti. Seat the **3060 Ti**. Install the **1TB NVMe**.
- [ ] **Re-map passthrough** (PCI IDs change): `hostpci` 5060 Ti → core-ai VM; 3060 Ti → voice-ai VM. `qm set` / verify `qm config`.

## Phase 5 — power up + deploy  (automation, once pve-03 is back on the network)
- [ ] Qdrant on the NVMe: install, restore the 3 snapshots, bind to localhost; curator Qdrant URL → `127.0.0.1:6333`.
- [ ] **STT WER measurement on the 3060 Ti** (the deferred A/B): run `~/iris-qdrant-backup/stt-test/` clips through canary-1b-v2 int8 and parakeet-tdt-0.6b-v2; pick the one that fits ≤~6GB with acceptable WER; wire it into `wyoming_iris_stt.py` (`--model` / `quantization`). Confirm STT + Kokoro resident together < 8GB.
- [ ] Deploy staged branches: 8B front door, trimmed STT, **OpenAI** deep-tier failover (API key in env, escalation-only), localhost Qdrant.
- [ ] Delete stale local models: `qwen3-deep` (cloud now), `gemma3` (creative retired), `qwen2.5:14b` (ceiling is 8B).
- [ ] Bring fabric up; walk the checklist: actuation turn · weather/query (grounded, single, no barge-in) · STT transcription accuracy · TTS playback · recall from NVMe Qdrant.

## Phase 6 — validate, then the irreversible step
- [ ] Soak ~1 day. Confirm stable across real turns.
- [ ] **THEN sell** the 2× 5080 + 4080. (Keep on the shelf until here so you can roll back by re-seating + reverting passthrough.)

## Rollback (any time before Phase 6)
Re-seat the pulled GPUs in pve-03, revert passthrough, restore the Qdrant snapshots, re-point curator to the old Qdrant (or localhost), flip cortex model back. All config; the data is backed up in Phase 0.

## Safety invariant — carry across the rebuild
- **`lock` domain stays blocked.** world-ai `fix/block-lock-domain` (commit f848fe0) hard-blocks the
  z-wave deadbolt at the `/ha/service` proxy (actuate + read), default `config.blocked_domains=lock`.
  Cherry-pick onto whichever world-ai branch is live, and **never reseed a `lock.*` entity** into
  `iris_mappings`/`iris_tools` during the Qdrant restore. See PRINCIPLES.md → Safety invariants.

## Open items / decisions
- ~~STT test audio~~ — **DONE**: 8 Kokoro-synth clips staged on pve-03; WER measured in Phase 5 on the 3060 Ti.
- ~~Cloud provider~~ — **ChatGPT/OpenAI** (Matt has credits). Escalation-only, reuse the punt path; API key in env.
- **8B front door:** confirm Hermes-3-8B (incumbent trial) vs qwen3-8B — decide before Phase 5 deploy.
