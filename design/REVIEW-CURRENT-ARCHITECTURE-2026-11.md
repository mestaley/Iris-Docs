# Iris — Current Architecture & Hardware Review (2026-11-02)

_Read-only audit. Every claim below was verified against the live hosts (SSH), the deployed code
(`/opt/iris-fusion-fabric` on core-ai), and the live systemd/docker/Ollama/Qdrant state on
2026-11-02, except where explicitly flagged as "per docs / unverified." Where this review and
`STATE.md` / `ARCHITECTURE.md` disagree, the live hosts win — and they disagree in several
material places (noted inline). This document supersedes the stale host table in `STATE.md`._

Auditor note on method: hosts were probed live (nvidia-smi, ollama, qdrant, systemd, docker, pve-03
`lspci`/`qm config`). A real voice-turn soak happened to be in the journald window (15:15–15:17Z
today, Matt testing "what time is it / weather / calendar / basement lights"), so the latency
numbers in §8 are **measured from production turns**, not budgets.

---

## 0. TL;DR of what changed since the docs were last trued-up

- **Hardware is NOT what `STATE.md` says.** core-ai has **2× RTX 5060 Ti 16GB** (not "2× RTX 5080"),
  both passed through to VM 201. The 5080s/4080 are already pulled and shelved for sale. voice-ai
  has **no GPU** and is vestigial.
- **A Google Coral Edge TPU (USB) is now plugged into pve-03** — present, enumerated, **not passed
  to any VM, and unused.** (Coordinator-flagged; see §1.4.)
- **cortex-creative is GONE** (systemd unit `not-found`, fully removed — not merely "disabled").
  **deep-mode is LIVE** (running, not in STATE's service list). The front door is now **hermes3:8b**,
  not qwen2.5:14b.
- **The "retired" consolidation/reflection stack is running again.** After the 13:41Z reboot today,
  memory-substrate, memory-substrate-http, reflection-coordinator, dream-mode, episode-buffer and
  chunk-store all came back up via `iris-fabric.target` (the units are `disabled` but the target
  `Wants=` them). They run **hollow** — pgvector (memory-ai) is down and `iris_memory` is nuked.
- Audio (STT+TTS) is **CPU on audio-ai** and was **repointed off the dead voice-ai host today**;
  it is live again.

---

## 1. Hardware & host topology

### 1.1 Physical hypervisor — pve-03 (`pve-03`, Proxmox, Arrow Lake-S / Z890)
Single physical box now hosts the entire Iris production cluster. Verified via `lspci -nn` + `qm`:

| PCI addr | Device | Disposition |
|---|---|---|
| `00:02.0` | Intel Arrow Lake-S iGPU (`8086:7d67`) | host console |
| `01:00.0/.1` | **NVIDIA RTX 5060 Ti 16GB** (GB206 `10de:2d04`) | → VM 201 `hostpci0`, `x-vga=1` (primary) = **GPU0** |
| `81:00.0/.1` | **NVIDIA RTX 5060 Ti 16GB** (GB206 `10de:2d04`) | → VM 201 `hostpci1` = **GPU1** |
| `02:00.0` | Kingston NV3 NVMe (DRAM-less) | VM disks |
| `82:00.0` | Realtek Killer E5000 5GbE | host net |
| `83:00.0` | Intel Wi-Fi 7 BE200 | host |
| USB bus 002 | **Global Unichip `1a6e:089a`** = Google Coral Edge TPU | host only, unused (§1.4) |

The two 5060 Ti are in **IOMMU groups 13 / 24** (per runbook) — cleanly separable, which is why both
pass to one VM independently. No 3060 Ti in this box (it lives in Matt's SolidWorks workstation).

### 1.2 VMs on pve-03 (from `qm list` / `qm config`)

| VMID | Name | vCPU | RAM | GPUs | Role | Status |
|---|---|---|---|---|---|---|
| 201 | core-ai1 | 8 | 24 GB | **both 5060 Ti** (hostpci0 01:00 x-vga, hostpci1 81:00) | Cognition fabric, Ollama×2, Qdrant, web-gateway | running |
| 202 | audio-ai | 8 | 16 GB | none | Conditioning + STT + TTS (all docker, CPU) | running |
| 205 | voice-ai | 6 | 16 GB | none | **Vestigial** — nothing listening on :10300/:8880, no containers, `nvidia-smi` fails (no driver/GPU) | running but DEAD |
| 210 | world-ai | 4 | 8 GB | none | Sensors / Meridian / HA proxy | running |

Infra LXCs (monitor-01 Grafana/Loki, infra-01 Pi-hole/chrony, bastion-01 where this audit runs,
memory-ai) are **per docs** also parked on pve-03 after the pve-01 retirement; not re-verified this
session. **memory-ai is shut down** (Phase 2, 2026-10-21) — pgvector gone.

### 1.3 Slim-rig end state (target, per `SLIM-RIG-RUNBOOK.md` + `SINGLE-GPU-CONSOLIDATION.md`)
Three boxes: **pve-03 = Iris** (core-ai fabric + audio + world-ai, GPU0 LLM/embed/thalamus, GPU1
local deep tier, Kokoro TTS on CPU, +1TB NVMe for Qdrant — NVMe **not yet installed**), **pve-01 =
Infra**, **pve-02 = Plex** (A400 moved from pve-01). **Decision 2026-11-02: stay all-local, keep
BOTH 5060 Ti, cancel the OpenAI/cloud deep-tier failover.** Kept-vs-sold: **keep** both 5060 Ti +
the Coral; **sell** the 2× RTX 5080 + 1× RTX 4080 (shelved, held until Phase-6 soak passes — the
only irreversible step). Open slim-rig TODOs: confirm a real voice turn's WER; retire VM 205 and
hand its 6c/16GB to core-ai; install the 1TB NVMe; pick the front-door model (hermes3:8b incumbent
vs qwen2.5:14b, which now fits 16GB again).

### 1.4 Google Coral Edge TPU (USB) — present, unused (coordinator-flagged)
- **Detected:** `lsusb` on pve-03 → `Bus 002 Device 002: ID 1a6e:089a Global Unichip Corp.` This is
  the **USB Accelerator form factor** (not the M.2/PCIe Apex variant — nothing on `lspci`, no
  `apex`/`gasket` kernel module, no `/dev/apex0`).
- **State `1a6e:089a`** is the pre-firmware DFU enumeration — the device re-enumerates as
  `18d1:9302 Google` only after `libedgetpu` loads firmware into it. It is sitting in the
  **un-initialized state, i.e. nothing has ever driven it** this boot.
- **Passthrough:** none. No `usb*` line in any VM's `qm config` (201/202/205/210). It is bound to the
  **pve-03 host only** and reachable by no guest.
- **Practical role:** int8 TFLite only, ~4 TOPS. Good for wake-word / keyword spotting, lightweight
  always-on audio/vision classifiers, Frigate-style object detection. **Not usable for LLM
  inference** (no general matmul/fp path, 8MB SRAM). 
- **Verdict:** present-but-unused. Plausible future home = a host-local wake-word / KWS front-end or
  a Frigate camera-detection sidecar, but it would first need to be USB-passed to a VM (likely
  audio-ai for wake-word, or a new Frigate LXC) and have a tflite pipeline written. Today it is dead
  weight — document it, don't count on it.

---

## 2. Cognition pipeline (core-ai fabric)

### 2.1 Bus
DDS (Cyclone/`iris_fusion_bus`) on **domain 42**, pub/sub topics (`iris/perception/...`,
`iris/context/...`, `iris/speech/utterance`, `iris/dream/trigger`, etc.). Everything cross-service is
async fire-and-forget on the bus; the only synchronous-blocking hops are the HTTP/gRPC side-calls
(Ollama `/api/chat`, Qdrant, embed, world-ai `/ha/service`, Meridian, Wyoming ASR).

### 2.2 Services running on core-ai (live `systemctl`, verified)
`iris-fabric.target` active. Running units:
`iris-thalamus`, `iris-curator`, `iris-cortex-logical`, **`iris-deep-mode`**, `iris-fusion-executive`,
`iris-tool-registry`, `iris-volition`, `iris-memory-substrate`, `iris-memory-substrate-http`,
`iris-episode-buffer`, `iris-chunk-store`, `iris-reflection-coordinator`, `iris-dream-mode`.
**`iris-cortex-creative` = `not-found` (removed).**

| Service | Role | On hot path? | Live verdict |
|---|---|---|---|
| thalamus | Routing; working-memory field; phi3.5 @ :11434 | YES | healthy |
| curator | Sensory integration, memory recall, **tool retrieval** (Qdrant), v0→v1 context | YES | healthy; `recall=no`/`recalled=0` on live turns (memory empty) |
| cortex-logical | **Front door**, two-pass native-tools, sole actuator, hermes3:8b @ :11434 | YES | healthy |
| deep-mode | Slow MoE deliberator, qwen3-deep @ :11435 (GPU1), barges in on punts | on punt only | running; GPU1 idle (model not resident now) |
| fusion-executive | Arbiter; dispatches actuation; emits FusionDecision + MemoryWrite | YES | healthy; creative disabled |
| tool-registry | I/O dispatch → world-ai/Meridian/HA; settle-verify | YES (on tool call) | healthy |
| cortex-creative | — | — | **DELETED** (unit gone) |
| volition | Proactive (lighting/arrival/briefing/events) | side-channel | LIVE (§7) |
| memory-substrate (+http :8013) | Episodic recall (DDS) + consolidated tier-2 (pgvector) | nominally YES | **HOLLOW** — pgvector down, `iris_memory` nuked (§5) |
| episode-buffer / chunk-store / reflection-coordinator / dream-mode | Reflection/consolidation "second brain" | NO | **running but inert** (§7) |

**Important correction to STATE.md:** its "Live fabric services" list omits `deep-mode` and still
implies `cortex-creative` is a disabled-but-present service. Reality: creative is **removed**,
deep-mode is **the** second cognitive tier, and the "retired" consolidation units are **running
again** (see §7 — they auto-start via the target on reboot even though individually `disabled`).

### 2.3 Two-pass native-tools cortex pattern (cortex-logical)
Confirmed in `/opt/iris-fusion-fabric/cortex-logical/src/cortex_logical/{prompt.py,service.py,tool_schema.py}`:
- **Pass 1** — Ollama **native tools API** attached, no format constraint. Model emits
  `message.tool_calls` and/or content. Tool calls are parsed (`_parse_tool_calls`) into
  `CortexProposedAction[]`.
- **Pass 2** — the pass-1 assistant message (content + tool_calls) is **replayed verbatim** with the
  tool **results** appended, and the model writes the final grounded `say`.
- **Actuation short-circuit:** when pass 1 is a pure device-control call, **pass 2 is skipped**
  (`category=pass2_skipped_actuation_only`) and the front door goes **silent** — the supervisor /
  settle-verify owns correctness. Measured: these turns finish in ~0.75–0.86s (pass1 only).
- **v0 gating:** cortex skips the ungrounded `v=0` context and only answers on `v>=1` (`GATED —
  skipping (will respond on ungated v>0)`), i.e. it waits for curator's grounded context.

### 2.4 End-to-end turn flow (observed on live turns today)
1. Pi voice node → RTP/Opus :5004 → rtp-ingest → audio-fabric gRPC ring (audio-ai).
2. conditioning: vad-consumer → asr-consumer (Wyoming → iris-stt :10300) → intent-consumer →
   affect-consumer → publishes **FeatureStream** on the bus.
3. thalamus routes; **curator** fetches ambient grounding + does **tool retrieval** (Qdrant
   `iris_tools`, scoped to ~7 candidate tools) and publishes `CONTEXT_CURATED` v0 (fast) then **v1**
   (grounded, ~1.3s measured).
4. **cortex-logical** runs two-pass; emits actions and/or `say`.
5. **fusion-executive** reconciles (creative off → logical owns the voice), dispatches actuation via
   tool-registry → world-ai `/ha/service`, writes FusionDecision + MemoryWrite, publishes
   `iris/speech/utterance`.
6. speech-output (audio-ai) clause-chunks → Kokoro :8880 → SpeechSink gRPC → voice node (barge-in armed).
7. On a hard/uncertain turn, fusion publishes a **punt** → **deep-mode** (qwen3-deep) deliberates and
   barges in with a `cause=correction` utterance.

---

## 3. Models (Ollama on core-ai)

Two Ollama instances, GPU-pinned by separate systemd units (verified `systemctl cat`):

| Instance | Port | `CUDA_VISIBLE_DEVICES` | `OLLAMA_KEEP_ALIVE` | Serves |
|---|---|---|---|---|
| `ollama.service` | 11434 | **0** (GPU0) | **-1** (pinned resident) | front door + thalamus + embed |
| `ollama-creative.service` | 11435 | **1** (GPU1) | default | deep tier (qwen3-deep) |

Note the second unit is still **named** `ollama-creative` (historical — creative is gone); it now
hosts the deep tier on GPU1.

**Models pulled** (`ollama list`): `hermes3:8b` (4.7GB), `qwen3-deep:latest` (14GB), `phi3.5` (2.2GB),
`gemma3:12b` (8.1GB — orphaned, was creative), `qwen2.5:14b` (9.0GB — former front door, now idle),
`nomic-embed-text` (274MB).

**Resident now** (`/api/ps`, GPU0 = 9.6GB used, GPU1 = 2MiB idle):
- **hermes3:8b** (5.2GB VRAM) — **front door**, `CORTEX_LOGICAL_MODEL=hermes3:8b`.
- **phi3.5** (4.0GB) — thalamus routing, `THALAMUS_OLLAMA_URL=…:11434`.
- **nomic-embed-text** (0.6GB) — curator embeddings for Qdrant (768-dim).
- **GPU1 empty** — `qwen3-deep` (deep tier, `DEEP_MODE_MODEL_URL=…:11435`) loads on demand for punts
  and is not resident between them.

Env pins (`/etc/iris-fabric/*.env`): `CORTEX_LOGICAL_MODEL=hermes3:8b` ·
`DEEP_MODE_ENABLED=true DEEP_MODE_MODEL_NAME=qwen3-deep DEEP_MODE_MODEL_URL=http://127.0.0.1:11435` ·
`CURATOR_TOOL_RETRIEVAL_ENABLED=true CURATOR_QDRANT_URL=http://127.0.0.1:6333` ·
`FUSION_EXECUTIVE_ENABLE_CREATIVE=false`. qwen2.5:14b and gemma3:12b are **pulled but unused** — free
disk to reclaim, or (per runbook) qwen2.5:14b is the candidate if Matt re-picks a 14B front door now
that GPU0 is a full 16GB.

---

## 4. Tool / actuation layer

- **Catalog** (`tool-registry/.../definitions.py`, ~16 tools → matches Qdrant `iris_tools`=19 incl.
  variants): `control_device`, `get_entity_state`, `get_current_weather`, `get_hourly_forecast`,
  `get_daily_forecast`, `get_weather_alerts`, `get_news`, `get_wikipedia`, `recall_memory`,
  `set_reminder`, `get_reminders`, `mute_notifications`, `unmute_notifications`, `get_tasks`,
  `create_task`, `update_task`, `briefing_ready`.
- **Tool retrieval via Qdrant** (`iris_tools`, 768-dim cosine, 19 points): curator embeds the turn
  (nomic) and scopes the catalog to ~7 candidates before the cortex sees them — this is the fix for
  Hermes tool-overload (`fd2f441`: `tool_retrieval_enabled=true` "required with Hermes"). Observed
  live: `tool-retrieval scoped -> 7 tools: [control_device, select_content, get_entity_state, …]`.
- **ADR-005 name-based `control_device`:** the cortex emits `control_device(name=…, service=…)`;
  resolution to an `entity_id` happens via `iris_mappings` (Qdrant, 56 points). Observed live:
  `coerced control_device -> entity=light.basement_lights service=turn_on brightness_pct=20`.
  Side-effecting tools are **issued by fusion**, not curator (curator only prefetches reads).
- **world-ai `/ha/service` proxy + settle-verify** (`app/services/ha_service.py`): world-ai holds the
  **only** HA token. control_device → HA, then **polls HA for the end-state** (first read immediate,
  later reads give flaky zigbee/zwave mesh devices time to settle, 8s budget / `_CONTROL_DEVICE_TIMEOUT_S`).
  Un-verifiable services (media_play, volume_set) trust HA's 200. This is what makes "HA 200 ≠ done"
  honest and lets Iris voice a correction when a device doesn't actually change.
- **`lock`-domain safety block (VERIFIED in code):** `world-ai/app/config.py` →
  `blocked_domains: str = "lock"` (default), normalized to a frozenset in `ha_service.py`. The proxy
  (sole HA-token holder) blocks the z-wave deadbolt for **both actuate and read**. This is the hard
  enforcement of the MEMORY safety invariant. Code is committed on branch `fix/block-lock-domain`
  (`f848fe0`) — **but that push + a world-ai restart are still owed** (see §9); need to confirm the
  running world-ai actually has it loaded, not just the checkout.

---

## 5. Data stores

- **Qdrant on core-ai** (docker, :6333, local; curator points at `127.0.0.1:6333`). Collections:
  - `iris_mappings` — **56 points**, 768-dim cosine — name→entity_id resolution for ADR-005.
  - `iris_tools` — **19 points**, 768-dim cosine — tool-retrieval catalog.
  - **`iris_memory` — GONE** (intentionally nuked; not recreated). Episodic vector recall therefore
    returns nothing — consistent with the `recalled=0` seen on every live turn.
  - Dataset ~1.4GB, backed up to `pve-03:~/iris-qdrant-backup/` (Phase 0, 4.5MB snapshots).
- **pgvector / memory-substrate consolidated tier:** **DOWN.** memory-ai VM retired 2026-10-21; the
  `:8013` HTTP service is up and `/health` returns ok, but it is **hollow** — no pgvector backend
  behind it, no consolidated records reachable. This is a health-endpoint-lies situation: the service
  looks alive but has nothing to serve.

---

## 6. Audio stack

All of STT/TTS/conditioning runs on **audio-ai (VM 202), CPU, docker-compose at `/opt/iris-conditioning`.**
Live `docker ps` (all Up):
- **rtp-ingest** (:5004/udp) → **audio-fabric** (gRPC ring :50051) → **vad-consumer** →
  **asr-consumer** → **intent-consumer** → **affect-consumer**.
- **iris-stt** — `istupakov/parakeet-tdt-0.6b-v2-onnx` via `wyoming_iris_stt.py`, Wyoming :10300,
  **CPU**. (The docs' old "Canary-1B on voice-ai GPU" claim is **stale/wrong** — confirmed by the
  2026-11-02 changelog; it was always parakeet-0.6b CPU here.)
- **kokoro-tts** — `ghcr.io/remsky/kokoro-fastapi-cpu`, :8880, **CPU**. Verified ~3.5× realtime
  (~1.0s for ~11 words); fabric streams per-clause so first-audio is a fraction of that.
- **speech-output** — clause-chunked Kokoro playback → SpeechSink gRPC → voice node :8766 (barge-in).
- **wake-arbiter** (:8090) — still **Up but vestigial** ("never wired", ISSUE-004 / dead-code).

**Recent repoint (2026-11-02, LIVE):** `asr-consumer`, `router-consumer`, `speech-output` had been
pointing at the dead `voice-ai.staley.home`; they were repointed to local compose
(`WHISPER_HOST=iris-stt`, `KOKORO_URL/TTS_URL=http://127.0.0.1:8880`). Subtlety that bit and is now
documented: `router-consumer` + `speech-output` are `network_mode: host`, so they must use
`127.0.0.1:8880` (they can't resolve the compose alias); `asr-consumer` is a mesh container so it
uses the `iris-stt` alias. This compose is **VM-local, not a git commit** — it exists only on
audio-ai, which is a backup/reproducibility gap.

**Speaker identification:** ECAPA voiceprint gate (in conditioning), `SPEAKER_THRESHOLD=0.35` per
STATE — gates Iris to respond only to Matt. It drops ~22% of genuinely-addressed turns (ISSUE-001,
needs a re-enroll in a quiet room). Not re-measured this session.

**Voice nodes (Pi):** `iris-voice-node-v2` (vg-01 kitchen, vg-02 basement) + `iris-voice-node-xvf3800`
(vg-03 master bedroom, ReSpeaker XVF3800). Streaming requires three allowlists + node maps to agree
(the "five-allowlist drift" class is the #1 "heard but mute" trap).

---

## 7. Memory / consolidation / proactivity

### 7.1 Consolidation / reflection ("second brain") — DESIGNED-but-INERT, and quietly re-animated
The design (dream-mode nightly consolidation, reflection-coordinator default-mode network,
episode-buffer short-term, chunk-store hippocampal write, memory-substrate tier-2) is **not
contributing to any live turn**. Live reality after today's 13:41Z reboot:
- All six units are **`disabled`** individually (memory-substrate, reflection-coordinator,
  episode-buffer, chunk-store) or **`enabled`** (memory-substrate-http, dream-mode) — **but all are
  currently RUNNING**, pulled up by `iris-fabric.target`'s `Wants=`. So the Phase-2 "teardown" does
  **not survive a reboot** — the stack silently came back. This contradicts STATE/runbook which
  describe it as stopped.
- They run **hollow:** `iris_memory` nuked, pgvector down, `dream-mode` idle ("awaiting triggers",
  nightly 04:30 timer), recall returns 0. This is pure **operational debt + false-retirement risk**:
  the stack burns RAM/handles, can corrupt prefs if ever re-triggered (ISSUE-003: destructive
  supersession, constant affect `[0,0,0,0,0,1]`, degenerate embeddings), and gives a false
  impression of "retired." **Recommend: mask the units (`systemctl mask`) or remove them from the
  target `Wants=`, not just `disable`.**

### 7.2 Episodic memory (hot path)
Nominally "recall before cortex, write after," but with `iris_memory` gone and pgvector down it is
**effectively a no-op** on live turns (`recall=no`, `recalled=0` every turn). The `recall_memory` tool
exists in the catalog but has nothing to hit.

### 7.3 Volition (proactive) — genuinely LIVE
`/etc/iris-fabric/volition.env` + live logs confirm proactivity is **on**, not dry-run, for most rails:
- `VOLITION_DRY_RUN=true` (global welcome-home gate) but **lighting/arrival/briefing/events
  `DRY_RUN=false`**.
- **Events substrate LIVE** — durable sqlite (`state/events.db`), intake :8015, counts
  `{delivered:320, cancelled:5, expired:31}`.
- **Arrival greeter LIVE** (window 900s, absence ≥1800s, node→room map).
- **Morning briefing LIVE** (07–11 window). Observed firing today: `BRIEFING OFFER (first) →
  [basement/vg-02, kitchen/vg-01]` at 13:42Z, then `briefing offer lapsed unanswered → done for today`.
- **Lighting arbiter LIVE** (rooms=['dining_room'], dim bands, 1800s cooldown, night 23–06) — **but
  throwing `lighting fetch (dining_room) failed: All connection attempts failed`** at startup (can't
  reach world-ai lighting route). Worth chasing — the lighting rail may be silently inert.

---

## 8. Latency & quality (MEASURED from the 15:15–15:17Z live soak today)

Front door = **hermes3:8b on GPU0**, two-pass. Representative measured turns:

| Turn | Type | pass1 | pass2 | cortex elapsed | fusion decision | notes |
|---|---|---|---|---|---|---|
| "what time is it" | chat | 759ms | 1357ms | **2117ms** | — | warm |
| "what's the weather" | 1 read tool | 608ms | 1768ms | **2405ms** | — | reads get_daily_forecast |
| "calendar today" | 1 read tool | 631ms | 2082ms | **2730ms** | 3997ms | |
| "living room lights off" | actuation | 808ms | skipped | **808ms** | 2147ms (say "Done.") | pass2 skipped, front door silent |
| "basement to 20%" | actuation | 861ms | skipped | **861ms** | 2218ms (actuated_ok) | ADR-005 coerce + settle-verify |
| "basement to 40%" (correction) | actuation | 747ms | skipped | **747ms** | 2049ms | re-actuation follows the correction cleanly |

- **Curator v1** publishes in ~1.3s (`publishing v1 after 1355ms`). **Fusion** decisions land in ~2–4s.
- **Warm** query turns: ~2.1–2.7s cortex, fusion ~2–4s → spoken completion in roughly the ~4–5.5s
  perceived-latency band the docs describe (dominated by VAD endpointing + fusion stacking, not
  first-audio). **Actuation** turns are fast (~0.8s cortex, silent).
- **Cold start:** the first turn after the 13:41 restart logged **pass1 ≈ 7.9s** (model warm-up), the
  next ≈ 668ms — so `OLLAMA_KEEP_ALIVE=-1` on GPU0 matters; deep-tier (GPU1, not resident) will pay a
  cold load on its first punt.
- **Deep tier:** fires only on punts (`FUSION_EXECUTIVE_PROACTIVE_PUNT_ENABLED=true`); not exercised
  in this window, historically ~13s to barge-in.

### Quality issues caught live
- **ASR homophones persist:** one turn transcribed "Hello, **Virus**. Do you know what time it is?"
  — wake-word/"Iris" mishearing still happens on parakeet-0.6b (it was handled correctly downstream
  here, but it's the ISSUE-001/address-gate family).
- **Courtesy phrase → confused apology:** a bare "**Thank you.**" produced fusion
  `neither_confident say='Sorry — that one got away from me. Say it again?'` — the front door returns
  `say=None` for social closers and fusion fills an apology. That's a de-nag / closer-filter gap
  (should be a silent or minimal ack, not "say it again").
- `recalled=0` on every turn is expected given §5 but means **zero long-term memory** is in play.

---

## 9. Honest assessment

### What genuinely works well
- **The live voice turn works and is reasonably fast.** Grounded answers (weather/calendar/time),
  clean name-based actuation with settle-verify, correction handling ("on second thought… 40%"), and
  the silent-on-clean-actuation model are all behaving as designed. Measured warm latencies are
  solid for a local 8B.
- **Tool-retrieval + ADR-005 name-based control** is a real architectural win — it fixed Hermes tool
  overload and decoupled the cortex from entity_ids. Qdrant-local is simple and fast.
- **The lock-domain safety block is correctly placed** (single HA-token chokepoint, blocks read+write).
- **Consolidation to one physical box with two separable GPUs** is a clean end state — zero
  cross-GPU contention (LLM on GPU0, deep tier on GPU1), all-local preserved.
- **Volition events rail** (320 delivered) is quietly doing useful work.

### Broken / half-done / stale
- **STATE.md is materially stale** (wrong GPUs, missing deep-mode, wrong creative status, wrong
  consolidation status). ARCHITECTURE.md still describes the Canary/voice-ai audio path and the
  two-cortex creative model — both obsolete. These are the SSOT and are misleading right now.
- **The "retired" consolidation stack auto-restarts via the target** and runs hollow — false
  retirement. `disable` ≠ stopped here. Mask it or cut it from the target.
- **memory-substrate-http /health lies** — ok while the pgvector tier behind it is gone.
- **Episodic/long-term memory is effectively absent** (`iris_memory` nuked, pgvector down) — Iris has
  no durable recall right now, only the session's recent-conversation slot.
- **Volition lighting rail errors at startup** ("All connection attempts failed" to world-ai) —
  likely silently non-functional.
- **"Thank you" → "Sorry, say it again"** and **"Iris"→"Virus"** are live, reproducible quality nicks.
- **audio-ai compose is VM-local only** (not committed anywhere) — the repoint that restored audio
  exists on one disk. Reproducibility/backup gap.
- **wake-arbiter (:8090) still Up** on audio-ai, still vestigial (ISSUE-004).

### Operational pain / tech debt / risks
- **Owed origin pushes (automation can't push):** wt-integration `integration/deployed-2026-09-28`
  (~11 commits `8507afe..fd2f441`), world-ai `fix/block-lock-domain` (`f848fe0`) + a world-ai
  restart to load the lock block, iris-conditioning `feat/prosodic-affect` (7 ahead), iris-docs main.
  **The deployed brain is running code that exists only on this box + one worktree.** If pve-03's
  NVMe (a DRAM-less Kingston NV3) dies, substantial unpushed work is at risk.
- **Single-box blast radius:** the entire cluster (all four VMs) is on one pve-03. One host failure =
  total Iris outage. The slim-rig design accepts this ("fine to brick Iris a few days").
- **Config locality:** a voice node still needs multiple allowlists across three services to agree;
  the "heard but mute" diagnosis surface is wide.
- **Branch sprawl:** ~8 worktrees under `/opt/repos` (wt-*), fabric repo on `feat/events-queue` while
  the deployed code is a different branch in `wt-integration` — easy to edit the wrong tree.
- **Coral TPU** bought/installed but unconnected to anything — spend with no pipeline yet.

### Biggest risks, ranked
1. **Unpushed deployed code on a single DRAM-less NVMe** — back it up / push before the Phase-6 GPU sale.
2. **Lock-domain block may not be live** — code is on an unpushed branch pending a world-ai restart;
   confirm the running proxy actually enforces it (safety invariant).
3. **Hollow consolidation stack re-animating on boot** — mask it so it can't silently corrupt prefs
   if a trigger ever fires.
4. **No durable memory** — the thesis keystone (affect-tagged precedent recall) is still open; Iris
   currently has no long-term recall at all.
5. **Audio config lives on one VM disk, uncommitted** — a lost audio-ai = reconstruct the repoint from memory.
