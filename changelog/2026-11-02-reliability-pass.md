# 2026-11-02 — Reliability & polish pass (make Iris solid)

## Executive summary
Matt chose "reliability & polish" as the first make-it-useful move. Found that most of it was already
done-but-unprotected or already-recovered: (1) Volition's boot-time "connection failed" errors were a
boot-ordering race — world-ai is healthy and Volition recovered on its own (no fix needed). (2) The
Iris↔"Virus" ASR fix, a no-wake await-window race fix, a voice-node clock-skew fix, and latency-hangover
tuning were all ALREADY LIVE on audio-ai but uncommitted — captured them in git (protect). (3) Verified
masking the hollow consolidation/dream stack is safe for the live turn (curator has no dependency;
fusion's memory_write is fire-and-forget), and prepared the exact teardown (needs Matt's sudo).
No behavior changed on core-ai this session; audio-ai code was committed (not pushed).

## Changes by repo
### iris-conditioning (audio-ai, LIVE — committed locally, push owed via bastion)
- `docker-compose.yml` — **committed** (99ad566): STT/TTS repoint off retired voice-ai →
  `asr-consumer.WHISPER_HOST=iris-stt` (mesh alias), `router/speech-output` TTS→`127.0.0.1:8880`
  (host-net containers can't resolve the compose alias).
- `consumers/asr/app/asr_consumer.py` — **committed** (19816b2): `SPEECH_END_HANGOVER_MS` env-tunable
  (ISSUE-009 perceived-latency tax; compose runs 700ms).
- `consumers/intent/app/intent_consumer.py` — **committed** (19816b2): leading iris-homophone rescue
  ("Iris,…"→"Fires,…" rescued when a command follows — the Iris/Virus fix); wall-clock timing for
  no-wake windows (vg-02 clock skew silently broke await/followup, ISSUE-012); gated-off CONVERSATION_MODE.
- `consumers/speech_output/app/main.py` — **committed** (19816b2): arm+stamp the await window on next
  idle, never immediately (was opening the no-wake window ~10s early while Iris was still asking, ISSUE-012).
- These were running live but had never been committed — a backup gap (no hardware rollback now that the
  surplus GPUs are sold). Untracked artifacts left alone (a contracts wheel, docker-compose.enroll.yml, scripts/).

### iris-docs
- `changelog/2026-11-02-reliability-pass.md` — **added**: this entry.

## Deploy / live state
- No core-ai changes. Audio-ai commits are LOCAL (VMs can't push; push via bastion `ssh -A`).
- Volition confirmed working (boot-race errors recovered); world-ai healthy (:8001 health 200).

## Follow-ups / owed — Matt's short list
1. **Tear down the hollow consolidation/dream stack** (core-ai sudo). Units live in `/etc/systemd/system`
   so `mask` fails; rename-aside (as with creative) + disable the digest timer:
   ```
   sudo systemctl stop dream-digest.timer iris-memory-substrate iris-memory-substrate-http \
     iris-reflection-coordinator iris-dream-mode iris-episode-buffer iris-chunk-store
   sudo systemctl disable dream-digest.timer iris-memory-substrate-http iris-dream-mode
   cd /etc/systemd/system
   for u in iris-memory-substrate iris-memory-substrate-http iris-reflection-coordinator \
            iris-dream-mode iris-episode-buffer iris-chunk-store dream-digest; do
     sudo mv "$u.service" "$u.service.disabled"; done
   sudo systemctl daemon-reload
   ```
   Safe: live turn path (thalamus/curator/cortex/fusion/tool-registry) is independent. Reversible.
2. **Push the audio-ai commits** (protect): from bastion `ssh -A audio-ai 'cd /opt/iris-conditioning && git push origin <branch>'` (99ad566, 19816b2).
3. **Push the deployed brain** — wt-integration `integration/deployed-2026-09-28` (~11 commits
   8507afe..fd2f441). Biggest backup gap: the running fabric is unpushed code on one NVMe, no HW rollback.
4. **Push world-ai `fix/block-lock-domain` (f848fe0) + restart world-ai** → makes the lock-domain block
   actually loaded in the running proxy (currently it is NOT; deadbolt is safe only because it isn't mapped).
5. Minor: Volition boot-order hardening (start After= world-ai reachable) to silence the harmless boot warnings.
