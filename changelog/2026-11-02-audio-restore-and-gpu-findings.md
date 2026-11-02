# 2026-11-02 — Audio stack restored (STT+TTS back on CPU); GPU/passthrough ground truth

## Executive summary
Picked up the slim-rig migration. Verified **memory-ai retirement** held (live path clean, no failed
units; Phase 2 done). Got SSH to pve-03 working (config alias `pve-03`, not the FQDN). Found the
**GPU/passthrough reality**: pve-03 has **2× RTX 5060 Ti** (no 3060 Ti — it's in Matt's workstation),
**both already passed to core-ai**, Ollama pinned to GPU0 so GPU1 is free. Matt chose **Option C**
(collapse voice-ai into core-ai). While executing it, discovered the audio stack was **already CPU on
audio-ai** all along (`iris-stt` = parakeet-0.6b-onnx :10300, `kokoro-tts` = kokoro-fastapi-cpu :8880);
voice-ai was vestigial and the only breakage was the conditioning consumers still pointing at the dead
`voice-ai.staley.home`. **Repointed them → STT and TTS are LIVE again** (TTS synth verified ~3.5×
realtime on CPU). No GPU STT build needed; the `nvidia-container-toolkit` install on core-ai is moot
for audio (harmless). Net: Iris can hear/speak again (pending a real voice turn to confirm WER), GPU1
is now pure surplus, and voice-ai is ready to retire.

## Changes by repo
### iris-conditioning (LIVE on audio-ai, `/opt/iris-conditioning/docker-compose.yml` — not a git commit; VM-local compose)
- `docker-compose.yml` — **changed**: `asr-consumer.WHISPER_HOST` `voice-ai.staley.home` → `iris-stt`;
  `router-consumer.TTS_URL` → `http://kokoro-tts:8880/v1/audio/speech`; `speech-output.KOKORO_URL` →
  `http://kokoro-tts:8880`. Recreated the three containers (`--no-deps`). Timestamped backups alongside.
  Fixes STT/TTS both pointing at the retired voice-ai host (ConnectionRefused).

### iris-docs — branch `main`
- `design/SLIM-RIG-RUNBOOK.md` — **changed**: target table → 2× 5060 Ti (GPU0 LLM / GPU1 surplus);
  recorded hardware reality + Option C; Phase 2 ✅ (memory-ai retired); Phases 4–5 rewritten to the CPU
  reality (audio already CPU on audio-ai; repoint done; voice-ai retire; GPU1 sell-or-keep fork).
- `changelog/2026-11-02-audio-restore-and-gpu-findings.md` — **added**: this entry.

### auto-memory
- `iris-host-topology.md` — **changed**: corrected to 2026-11-02 ground truth — 2× 5060 Ti both on
  core-ai (GPU0 LLM, GPU1 surplus); memory-ai retired; voice-ai vestigial; audio-ai runs the real
  STT/TTS as CPU containers + the consumer repoints.

## Deploy / live state
- **LIVE on audio-ai**: STT (`iris-stt`, parakeet-0.6b CPU) + TTS (`kokoro-tts` CPU) reachable and
  repointed; asr-consumer `whisper=iris-stt:10300`, voice-node workers up; Kokoro synth 200 OK.
- **core-ai**: nvidia-container-toolkit installed (Matt) + docker nvidia runtime registered; GPU1
  reachable from containers. Not used by anything yet. Ollama still GPU0.
- **memory-ai**: shut down; consolidation stack (memory-substrate ×2, reflection-coordinator,
  dream-mode) inactive. Live turn path unaffected.
- Not yet done: real-voice-turn confirmation; voice-ai VM retirement; GPU1 sell-or-keep; front-door
  model decision.

## Follow-ups / owed
- **DECISION 2026-11-02: GPU1 KEPT** on core-ai for a local deep tier → Iris stays all-local, the
  **OpenAI/cloud failover is CANCELLED** (not built). Only the 2×5080+4080 are sold; both 5060 Ti stay.
- **Matt:** confirm a real voice turn (hear→think→speak) when home; retire voice-ai VM 205 (reallocate
  6c/16GB to core-ai).
- Still owed from before: push world-ai `fix/block-lock-domain` (f848fe0) + restart world-ai; the
  iris-docs `main` commits (ahead of origin).
