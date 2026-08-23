# Iris — Architecture

_Verified 2026-08-22/23 against live code + `iris-fusion-fabric/docs/ARCHITECTURE.md`. Budgets are configured defaults, not measured p50s._

## Per-turn hot path (NOW — async streaming)

Legend: `==>` async DDS pub/sub (non-blocking) · `-->` synchronous-blocking HTTP/gRPC side-call.

```
USER speaks
  │ RTP/Opus UDP :5004                                       [async, node→audio-ai]
  ▼
rtp-ingest ==> audio-fabric ring (gRPC :50051)
  ▼
iris-conditioning/router  (VAD → ASR → intent → affect)
  │  ASR: Wyoming --> iris-stt-gpu :10300                    [BLOCKING, voice-ai]
  │  publishes FeatureStream, FIRE-AND-FORGET (router_consumer.py:230-241)
  ▼ PERCEPTION_FEATURE_STREAM (DDS) — fan-out
  ├──► thalamus     phi3.5 --> Ollama :11435   [budget 1500ms]
  ├──► curator      [turn_deadline 1500ms; routing wait ~600ms]
  │       ├─ MEMORY_RECALL ==> memory-substrate ==> result
  │       ├─ nomic embed --> Ollama :11434 ; Qdrant --> :6333   [BLOCKING]
  │       ├─ TOOL prefetch ==> tool-registry ==> result
  │       └─ FAST-PATH --> world-ai /fastpath/decide (qwen2.5:7b, ~400-550ms warm)
  │            curator FIRES reflex actuation itself
  │       CONTEXT_CURATED v0 (fast) ==> then v1 (grounded)
  ▼ CONTEXT_CURATED (DDS) — fan-out
  ├──► cortex-logical   qwen2.5:14b --> Ollama :11434  [budget 10000ms; ~2.5s/toolcall]
  └──► cortex-creative  gemma3:12b  --> Ollama :11435  [budget 6000ms]
          streams SpeechUtterance deltas AS GENERATED (v>=1) ==>┐
  ▼                                                             │
fusion-executive  [budget 2000ms]                              │ (parallel
  │  reconciles voice owner; on actuation:                     │  speech
  │    TOOL_REQUEST ==> tool-registry --> world-ai/meridian --> HA :8123  [tool timeout 8s]
  │  FUSION_DECISION ==> episode-buffer   MEMORY_WRITE ==> memory-substrate
  └────────────────────────────────────────────────────────┐  │
                                                            ▼  ▼
                  iris/speech/utterance (DDS) ==> speech-output (audio-ai)
                     clause-chunked Kokoro --> :8880  (first clause ~0.2s)
                     SpeechSink.Play gRPC ==> voice node :8766 (barge-in armed)
```

**Latency reconciliation** (~2.4s first-audio vs ~4.5–5.5s perceived): first audio is fast
because creative streams optimistically (~0.2s first clause), but *perceived completion*
stacks curator 1.5s + logical (up to 10s cap, ~2.5s/toolcall) + fusion 2s. The biggest single
budget is `cortex-logical default_latency_budget_ms=10000` (bumped from 5000).

**Sync-blocking hops:** ASR (Wyoming), each Ollama `/api/chat`, curator Qdrant/embed, the
fast-path call, tool-registry→world-ai/meridian→HA. Everything on the DDS bus is async.

## Service inventory

**core-ai fabric (systemd `iris-*.service`):** thalamus (routing, phi3.5) · curator (sensory
integration + memory recall + tool prefetch + reflex fast-path) · cortex-logical (qwen2.5:14b,
reasoning + fast-path supervisor) · cortex-creative (gemma3:12b, voice owner, streams prose) ·
fusion-executive (arbiter, dispatches actuation, canonical FusionDecision + MemoryWrite) ·
tool-registry (I/O dispatch to world-ai/meridian) · memory-substrate (DDS recall, **in-memory
only**, Qdrant deferred) · memory-substrate-http (:8013 consolidated tier-2 pgvector; only
caller is dream-mode) · episode-buffer · reflection-coordinator (unit **disabled**) · chunk-store ·
dream-mode (nightly, shadow default) · dream-mode-trigger (04:30 timer) · volition (proactive) ·
dream-digest (04:00 timer).

**audio-ai (docker):** rtp-ingest · audio-fabric (gRPC ring) · wake-arbiter (:8090, **vestigial**) ·
conditioning/router (turn originator: VAD→ASR→intent→affect) · speech-output (Kokoro streaming).

**voice-ai (pve-01):** iris-stt-gpu (Wyoming :10300) · kokoro-tts (:8880).

**voice nodes (Pi):** iris-voice-node-v2 (vg-01/02) · iris-voice-node-xvf3800 (vg-03 bedroom).

**world-ai:** FastAPI :8001 (HA/MA/Met.no grounding + qwen2.5:7b fast-path) · world-ai-presence
(radar/census streaming ingest) · meridian :8002 (weather/calendar/news/etc).

**storage (memory-ai/pve-02):** Postgres+pgvector :5432 · Qdrant :6333.

## Two planes

Only the **live-turn plane** touches a spoken reply:
`conditioning → curator → one cortex → fusion → speech-output`.

Everything else is the **reflection/consolidation plane** (episode-buffer,
reflection-coordinator, chunk-store, dream-mode, dream-digest, memory-substrate-http) — a whole
second brain that never touches a live turn — or confirmed-vestigial (see dead-code list).

## Diagnosis surface (the real complexity metric)

**"No response at all" — 12 independent suspects:** voice node · rtp-ingest · audio-fabric ring ·
conditioning router (incl. node not in `STREAMING_NODES` allowlist) · iris-stt-gpu · thalamus ·
curator · cortex-logical · cortex-creative (the actual voice owner) · fusion-executive (incl.
`FUSION_EXECUTIVE_STREAMING_NODES` allowlist) · speech-output · kokoro-tts. Plus the
allowlist-drift class that spans three services at once (five allowlists must agree per node).

**"Wrong action / wrong words" — 9 suspects:** iris-stt-gpu · thalamus · curator (v0/v1/v2 race) ·
world-ai fast-path · cortex-logical · cortex-creative · fusion-executive · tool-registry · downstream HA/MA.

**Key insight:** the "articulate lie" family (multi-light, Spotify "now playing", calendar
hallucination) is **one** root cause — cortex speech isn't grounded in `tool_results` — but it
surfaces across creative + fusion + tool-registry, reading as three bugs. **The diagnosis
surface is wider than the actual fault surface.** That, not the capability count, is the
complexity cost. See ISSUE-005.

## Then (synchronous) vs Now

THEN (`iris-audio-sync-bridge`): one cognitive hop — router → `call_brain()` blocked up to 15s
for exactly one `FusionDecision`, spoke it via batch TTS. Diagnosis: ~4 hops, readable from one
waiter log line; "exactly once"; a hard 15s latency ceiling.

NOW: fan-out across 8+ DDS services, 3-tier deadline cascade, two Ollama instances, a fast-path
model on a third host, a separate streaming speech channel.

- **Gained:** proactive speech (impossible before), voiced supervisor corrections, multi-utterance
  turns, ~0.2s first-audio, barge-in.
- **Lost:** single-line diagnosability (1 waiter → 12-suspect fan-out); "exactly once" determinism
  (v0→v1→v2 race can make spoken text lag the canonical record by a version); a hard latency
  ceiling (perceived latency is emergent across four budgets); configuration locality (a node now
  needs five allowlists across three services to agree).

Cutting the vestigial set removes **zero** live-turn capability and roughly halves the
no-response diagnosis surface.
