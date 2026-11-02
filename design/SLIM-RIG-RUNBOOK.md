# Slim-rig migration — runbook

Companion to [`SINGLE-GPU-CONSOLIDATION.md`](SINGLE-GPU-CONSOLIDATION.md) (the decision). Execution checklist
for the 3-box end state. Downtime acceptable (Matt: "fine to brick Iris a few days"). **Do not sell any GPU
until Phase 6 passes** — it's the only irreversible step.

## Target
| Node | Role | GPUs | Storage |
|---|---|---|---|
| pve-03 | **Iris** (core-ai fabric + audio stack + world-ai) | 2× 5060 Ti 16GB: GPU0 = LLM+embed+thalamus · GPU1 = STT. Kokoro TTS on CPU. | +1TB NVMe (local Qdrant) |
| pve-01 | **Infra** (monitor, DNS, bastion, …) | — | — |
| pve-02 | **Plex** | A400 (moved from pve-01) | existing ZFS pool |

**Hardware reality (verified 2026-11-02):** pve-03 holds **2× RTX 5060 Ti** (PCI 01:00 / 81:00, IOMMU
groups 13 / 24 — cleanly separable), **both already passed to core-ai** (VM 201 hostpci0+hostpci1);
Ollama already pinned to GPU0 (`CUDA_VISIBLE_DEVICES=0`) so GPU1 is free. **No 3060 Ti** (it stays in
Matt's workstation for SolidWorks). voice-ai (VM 205) has no GPU.

**DECISION 2026-11-02 — Option C (collapse voice-ai into core-ai).** STT runs on core-ai GPU1,
LLM+embed+thalamus on GPU0 (separate cards → zero contention, the original single-GPU risk #1 is
gone), **Kokoro TTS moves to CPU**. voice-ai VM retired, freeing 6 cores + 16GB to the host (hand to
core-ai, which is 8c/23GB). **Reopened by having a full 16GB per role:** STT no longer needs trimming
(Canary-1B full fits GPU1); the front door is no longer capped at 8B (14B fits GPU0); a **local deep
tier becomes possible again**, so the cloud-API failover is now *optional*, not required.

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

## Phase 2 — retire memory-ai  ✅ (2026-10-21)
- [x] Live data already on core-ai local Qdrant (`iris_mappings` 56 + `iris_tools` 19, verified); `iris_memory` intentionally nuked; Phase-0 snapshots on pve-03 are the baseline. No new snapshot needed.
- [x] Stopped the memory-ai-dependent fabric stack first (clean teardown, no failed units): `iris-memory-substrate`, `-http`, `iris-reflection-coordinator`, `iris-dream-mode` → inactive. (Consolidated/pgvector tier dropped — hollow, not migrated.)
- [x] memory-ai shut down. Live voice path unaffected (curator → `127.0.0.1:6333`; thalamus/cortex/tool-registry never touched it). **pve-02 now clear** for Plex.
- Note: `pve-03` SSH was denied this session (`publickey,password`) — couldn't re-verify the snapshot backup there; the authoritative copy is live on core-ai regardless.

## Phase 3 — pve-02 → Plex  (Matt)
- [ ] Move the **A400 from pve-01 → pve-02** (passthrough to the Plex LXC). Verify NVENC in-guest (`ffmpeg … h264_nvenc`).
- [ ] Plex LXC on the existing ZFS pool.

## Phase 4 — GPU seating ✅ (done before 2026-11-02)
- [x] Surplus (2× 5080 + 4080) already pulled; **2× 5060 Ti** seated in pve-03, both passed to core-ai
  (VM 201). GPU0 (01:00, x-vga primary) + GPU1 (02:00 in-guest / 81:00 host). Ollama pinned to GPU0.
- [ ] 1TB NVMe for Qdrant — **not yet installed** (local Qdrant currently runs in core-ai's docker on
  the VM disk; fine for now, the dataset is ~1.4GB). Install when convenient; not blocking.

## Phase 5 — restore audio + retire voice-ai  (mostly done 2026-11-02)
**Discovery that rewrote this phase:** the audio stack was **already CPU on audio-ai** the whole time —
`iris-stt` (`istupakov/parakeet-tdt-0.6b-v2-onnx` via `wyoming_iris_stt.py`, :10300) and `kokoro-tts`
(`ghcr.io/remsky/kokoro-fastapi-cpu`, :8880), both in the `iris-conditioning` compose. The docs'
"Canary-1B on voice-ai GPU" was stale. voice-ai (VM 205) was vestigial (nothing on :10300/:8880). The
only breakage was that `asr-consumer`/`router-consumer`/`speech-output` still pointed at the dead
`voice-ai.staley.home`. **So STT/TTS never needed a GPU — the NVIDIA-container-runtime prereq is moot
for audio** (left installed on core-ai; harmless, reusable).
- [x] **Repointed the audio consumers** (automation, sudoless docker on audio-ai) from
  `voice-ai.staley.home` → the local compose services, recreated each:
  `asr-consumer.WHISPER_HOST=iris-stt` · `router-consumer.TTS_URL=http://kokoro-tts:8880/...` ·
  `speech-output.KOKORO_URL=http://kokoro-tts:8880`. Edited `/opt/iris-conditioning/docker-compose.yml`
  (backups alongside). asr-consumer logs `whisper=iris-stt:10300`, workers up (vg-01/vg-03), TCP OK.
- [x] **TTS verified**: Kokoro CPU synth 200 OK, valid 24kHz WAV, ~1.0s for ~11 words (~3.5× realtime);
  fabric streams per-clause so first-audio is a fraction of that.
- [ ] **Confirm on a real voice turn** (Matt, when home): STT accuracy + end-to-end hear→think→speak.
  parakeet-0.6b is a strong English ASR; WER-check against `~/iris-qdrant-backup/stt-test/` if suspect.
- [ ] **Retire voice-ai VM 205** (Matt) — now safe; reallocate its 6 cores + 16GB to core-ai (8c/23GB).
- [ ] **Front-door model decision** (GPU0 is a full 16GB, STT/TTS off-GPU): Hermes-3-8B (incumbent) vs
  qwen3-8B vs **keep qwen2.5:14b** (fits again). Decide, then deploy.
- [ ] **GPU1 (2nd 5060 Ti) is now pure surplus** — STT/TTS are CPU, LLM needs only GPU0. Decide: **sell
  it** (adds to the 2×5080+4080 pile) OR **keep on core-ai** for a 14B+ front door / a local deep tier
  (→ no cloud dependency, OpenAI failover unneeded). This is the main open fork.
- [x] Qdrant → localhost (curator repointed; memory-ai retired).

## Phase 6 — validate, then the irreversible step
- [ ] Soak ~1 day across real turns.
- [ ] **THEN sell** the 2× 5080 + 4080 (already shelved). Keep until here for rollback.

## Rollback (any time before Phase 6)
Re-point the audio consumers back to voice-ai, re-start the voice-ai STT/TTS, flip cortex model back,
re-point curator Qdrant. All config; data backed up in Phase 0. (The 5080s/4080 on the shelf are the
hardware rollback for the sell gate.)

## Safety invariant — carry across the rebuild
- **`lock` domain stays blocked.** world-ai `fix/block-lock-domain` (commit f848fe0) hard-blocks the
  z-wave deadbolt at the `/ha/service` proxy (actuate + read), default `config.blocked_domains=lock`.
  Cherry-pick onto whichever world-ai branch is live, and **never reseed a `lock.*` entity** into
  `iris_mappings`/`iris_tools` during the Qdrant restore. See PRINCIPLES.md → Safety invariants.

## Open items / decisions
- ~~STT test audio~~ — **DONE**: 8 Kokoro-synth clips staged on pve-03; WER measured in Phase 5 on the 3060 Ti.
- ~~Cloud provider~~ — **ChatGPT/OpenAI** (Matt has credits). Escalation-only, reuse the punt path; API key in env.
- **8B front door:** confirm Hermes-3-8B (incumbent trial) vs qwen3-8B — decide before Phase 5 deploy.
