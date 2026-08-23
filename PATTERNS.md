# Iris — Patterns

Engineering patterns that have proven valuable and should be reused, not re-invented. When you find yourself solving a problem that feels familiar, check here first.

This is short by design. If a pattern needs more than a paragraph or two, it probably belongs in a `decisions/` ADR or `ARCHITECTURE.md`.

> _Ported into iris-docs 2026-08-23 from the earlier `Architecture/PATTERNS.md`, spot-checked
> against live code (still-true). Inline **UPDATE 2026-08-23** notes mark examples whose
> specifics have since changed; the patterns themselves stand._

---

## The bus carries cognition, HTTP carries I/O

The DDS bus connects services that *think together* — thalamus, curator, cortices, fusion-executive, memory-substrate, tool-registry. Anything that touches the physical world or external services lives behind an HTTP endpoint: HA, Spotify, weather, Sonos, Kokoro, voice nodes, iris-stt-gpu.

When you're tempted to put an I/O service on the bus, ask: does it need pub/sub semantics with other thinking services? Usually no. HTTP is the right answer for "call a function, get a result." DDS is the right answer for "broadcast something other thinkers might care about."

---

## Proxy through world-ai for external integrations

Tool-registry does NOT call HA, Spotify, weather APIs, or other external services directly. All external integration goes through `world-ai.staley.home:8001/...` (HA, Spotify) or `:8002/...` (Meridian: weather, news, sports, etc.).

This concentrates auth in one service and lets us evolve the proxy without rewiring tool-registry. Brain already did this; tool-registry now matches. **New external integrations should follow this pattern unless there's a documented reason not to.**

> **UPDATE 2026-08-23:** the pattern stands; the Spotify example is now dead — world-ai bridges playback through Music Assistant and holds no Spotify creds (those fields were pruned, world-ai `68369bf`).

---

## Cyclone DDS in Docker → host networking

Any service that joins the DDS bus from a Docker container must use `network_mode: host`. Cyclone announces its source IP in discovery; bridge IPs (`172.18.x.x`) are unreachable from peers.

Alternative: explicit unicast peers + `ExternalNetworkAddress` config + port publishing. Works but operationally heavier. Host networking is the documented convention.

---

## One job per host

Every Iris host has exactly one role. Voice-ai does GPU audio I/O. Audio-ai does CPU audio pipeline. Core-ai does cognition. World-ai does external integrations. Memory-ai does persistence.

When two roles share a host (as world-ai did pre-Phase-1.x with both "integrations" AND "GPU TTS"), the resulting host becomes harder to reason about, harder to upgrade, harder to fail-isolate. The single-role discipline pays itself back in operational simplicity.

Corollary: when a new responsibility appears, ask which existing role it belongs to rather than which host has spare CPU. If it doesn't fit any existing role, that's evidence a new host is the right answer — not that an existing host should absorb a second job.

---

## Verify migration claims with code, not docs

When DECISIONS.md says "we migrated from X to Y," the migration is complete only when the actual call path is verified end-to-end:

- `grep` of the consumer's environment to confirm the new endpoint
- Log inspection of the consumer to confirm requests reach the new component
- Inspection of running configuration matches expectations

Documentation captures intent at write-time, not necessarily reality at read-time. The Phase 1.0 discovery was: a documented "Parakeet replaces Whisper" migration had landed on voice-ai but NOT on audio-ai's asr-consumer. The doc-vs-reality drift went undetected for 10 days, blamed on Parakeet, and only surfaced via STT failure on real noisy audio.

A second instance during the Loki rollout: the database schema on world-ai was consolidated from `occupancy_events` to `sensor_events WHERE event_type='occupancy'` at some unknown earlier date, but `occupancy_poll.py` was never updated and was crashlooping every 30s. The "migration" was incomplete on the consumer side.

**Add `verify_migration.sh` scripts when migrations are committed** — short shell scripts that grep the relevant config, curl the relevant endpoints, and assert the actual state matches the documented state. Cheaper to write than four hours of mistaken debugging. The pattern generalizes beyond container-version migrations: schema migrations, endpoint renames, contract changes — anything where a producer/source changes shape — must be verified against every consumer end-to-end before declaring complete.

---

## Per-tool timeout overrides in dispatch

`tool_registry/dispatch.py::PER_TOOL_TIMEOUT_S` maps tool names to minimum timeouts. Effective timeout is `max(caller_request, tool_minimum)`. Slow tools (e.g., `select_content` at 15s) get protected from caller default (5s).

When adding a new tool, decide its realistic upper-bound latency and add it to the table. Don't trust callers to know.

---

## Honor backend success flags

When wrapping a backend HTTP call in tool dispatch, check if the response dict has its own `success: False`. Many backends return graceful failures (validation errors, downstream 500s) as dicts rather than exceptions. Hardcoded `success=True` lies to callers.

Pattern:
```python
backend_success = True
if isinstance(result, dict) and "success" in result:
    backend_success = bool(result["success"])
```

---

## Enum rendering for prompt-driven tool schemas

When a tool parameter has an enum, render it inline in the cortex prompt as `param:type={val1|val2|val3}`. Models pattern-match to these strings verbatim, much more reliably than free-text descriptions.

Pattern (in cortex prompt builder):
```python
if p.enum:
    enum_str = "|".join(p.enum)
    param_strs.append(f"{p.name}:{p.type}{marker}={{{enum_str}}}")
```

---

## Curator injects everything cortices need

Cortices read CuratedContext only. They don't pull from databases, don't call external services, don't import shared catalogs. Curator gathers everything (tools, mappings, ambient data, memory, session state) and packages it into one immutable context object per turn.

**Why:** Cortices stay stateless and side-effect-free. Anything they need is in the context they receive. Reproducing a turn's behavior given a CuratedContext is deterministic (modulo model temperature).

If you find yourself wanting a cortex to call a service, the service's result probably belongs in curator's ambient instead.

---

## Filter at the structural layer, not the prompt layer

When the cortices must "not do X" and prompt-level instructions fail to suppress the behavior, the fix is to make X structurally impossible rather than to escalate prompt force.

Examples that worked:

- **Weather prefetch dedup.** Curator filters `available_tools` to exclude tools whose output is already in ambient. When weather is in ambient, weather tools disappear from the catalog. Logical can't re-fetch what it can't see. This succeeded after two failed attempts to instruct qwen2.5:14b via prompt to "not re-fetch already-fetched data."

- **Identity anchor injection.** Always-injected at the curator layer rather than being a prompt guideline. Cortices can't forget the character because they don't have to remember it — it's in their context every turn.

Examples where the prompt layer is the right fit:

- Tone and register (creative voice character — "dry, observant")
- Domain-specific conventions ("data first, then observation")
- Output format hints (JSON schema reminders)

The test: if you find yourself writing "the model should not do X" three times in a prompt and it still happens, the question is whether X can be made structurally impossible. Structural is deterministic; prompts are probabilistic.

---

## Single shared stateful model is an anti-pattern across per-node workers

When a model maintains internal state across calls (RNN hidden state, recurrent context buffer, anything with `model.reset_states()` or equivalent), a single shared instance across per-node workers corrupts state through interleaving. Calls from different node streams update the model's internal state in interleaved order until it eventually becomes invalid.

The fix is **per-node model instances** attached to per-worker state objects. The model files themselves can be shared via a host-side bind-mounted cache (one copy on disk); the in-memory model objects must be per-node.

Bind-mount pattern:

```yaml
# docker-compose.yml
volumes:
  - ./models/torch-hub:/root/.cache/torch/hub:ro
```

```python
# In process(), lazy-load on first chunk per worker
state = getattr(worker, "_my_state", None)
if state is None:
    state = MyState()
    model, _ = torch.hub.load(...)
    state.model = model
    worker._my_state = state
```

Concrete instance: Silero VAD in vad-consumer and asr-consumer. See DECISIONS.md for the full incident.

The pattern generalizes to anything with internal state: encoder-decoder transformer state, LSTM hidden states, stateful audio processors, any model where `model.forward()` modifies attributes of `self`. A `threading.Lock` around the model is NOT a fix — it prevents races but not state interleaving. Per-instance isolation is the only correct approach.

---

## Clarifying questions block actuation

When fusion picks a `say` from creative that is a clarifying question, AND logical produced tool calls for the same turn, the tool calls must be suppressed. Speaking a question while having already acted on a best guess is incoherent behavior.

The heuristic in fusion: `"?" in creative.proposed_say AND has_actions → suppress actions, set keep_listening=True`. Intentionally loose (catches questions mid-sentence). The right long-term fix is a structured `needs_clarification: bool` field on CortexResponse that creative sets explicitly.

> **UPDATE 2026-08-23:** the long-term fix SHIPPED — `needs_clarification: bool` is live on CortexResponse with a canonical `_NEEDS_CLARIFICATION_RULE`; fusion suppresses actuation on the flag, not on `"?"`. (`keep_listening` itself is now unwired — ISSUE-004.) Kept here as the worked example of "structured signal > surface-text inspection" — see the ported ADR on structured cortex→fusion signals.

Generalization: anywhere a cortex disagreement is structurally meaningful (one is asking a question, the other is acting; one is hedging, the other is confident), fusion should respect the disagreement rather than blindly combining outputs.

---

## Hard prompt rules > soft suggestions for model behavior

When a prompt change must enforce restraint, use direct prohibition ("DO NOT mention X unless..."), not soft framing ("try to avoid..."). Models give weight to imperative phrasing.

Worked: "DO NOT mention Davis unless the user explicitly mentions him."
Didn't work: "Don't reach for Davis as a punchline; mention him only when directly relevant."

Same intent, very different outcomes. The first works because models obey strong directives more reliably than they internalize style guidance.

Phase 1.x extension: hard rules ALSO need scoped trigger conditions. The GROUND TRUTH preamble had a hard "state these values when asked" rule that misfired on adjacent question shapes. The fix was to scope WHEN the rule applies, not just HOW strongly. "ONLY mention X if explicitly asked... For any other request, DO NOT mention." Hard rule plus explicit trigger plus explicit anti-trigger.

---

## Production / failsafe coexistence

Brain at port 8000 stays loaded as a failsafe even though fabric handles all traffic. One config flip reverts. Same pattern when migrating any production-critical component:

1. Build the replacement alongside the existing system
2. Validate the replacement
3. Cut traffic over with a single config edit
4. Keep the old system warm for one quiet week
5. Then retire

The cost of the parked old system is small (RAM, disk); the cost of having no fallback is large.

> **UPDATE 2026-08-23:** the pattern stands; its running example expired — the brain-at-:8000 / synchronous `fabric_brain` bridge failsafe is now confirmed dormant/vestigial and slated to prune (ADR-001, ISSUE-004). It served its "one quiet week" long ago. The lesson (keep the old path warm through a cutover) is still correct; just don't leave the warm path wired indefinitely — that's the diagnosis-surface cost ADR-001 is paying down.

**Exception worth noting:** When Matt explicitly says "no going back," skip the failsafe pattern. Some migrations are clean enough to commit to fully (Canary GPU STT was strictly better than CPU Parakeet on every axis — slower fallback wasn't worth keeping warm). Default is failsafe; deliberate-no-failsafe is a documented decision.

---

## Sane default values for tool argument validation

Tool backends should normalize invalid values to a safe default and continue, with a logged warning. Better than hard-failing on enum violations the model generates.

Example: `select_content` accepts `content_type` enum [playlist|artist|album|track]. If the model produces `"genre"`, normalize to `"playlist"` and proceed. Logged at warn level so we can spot the pattern but not block the user.

---

## Anker PowerConf only exposes the DSP-processed channel

Worth knowing architecturally: PowerConf firmware does NOT expose its raw mic array. The only capture device presented to USB is the post-DSP processed mono stream (post-AEC, post-beamforming, post-noise-suppression). Whatever consumer reads from PowerConf is getting Anker's hardware processing, no opt-out.

This is a design strength for our use case — Anker decided to not let people accidentally bypass their DSP. Means we don't have to choose channels or configure routing; PowerConf handles it before any software sees the audio. `/etc/asound.conf` just pins ALSA's `default` to PowerConf so PyAudio picks it up reliably.

Worth knowing in case future hardware swaps lose this property — some USB conference mics DO expose raw channels and the wrong one would dramatically degrade STT quality.

---

## Lock down auto-updates on production hosts

`unattended-upgrades` + NVIDIA + DKMS + PREEMPT_RT is a failure mode lottery. The right defense is to never let the lottery run: disable auto-updates, hold NVIDIA + linux-image families, do all package maintenance during explicit maintenance windows.

Pattern applied cluster-wide. New hosts get locked down immediately after NVIDIA driver install completes — BEFORE the first scheduled apt activity could fire.

Defense in depth: `systemctl disable --now` AND `systemctl mask` for `unattended-upgrades.service`, `apt-daily.timer`, `apt-daily-upgrade.timer`. Plus `apt-mark hold` for `nvidia-*`, `linux-image-*`, `linux-headers-*`, `linux-modules-*`. Plus `/etc/apt/apt.conf.d/99-iris-no-autoupdates` setting `APT::Periodic` keys to 0. Any one of these failing leaves the others in place.

---

## Disk planning for ML inference hosts

Ubuntu's autoinstaller default creates a tiny LVM logical volume (~38 GB) and leaves the rest of the volume group unallocated. For an ML host that downloads multi-GB models on first run, this fills up fast.

Default workflow after Ubuntu install on any ML host:

```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
df -h /
```

Do this BEFORE pulling any large container images or model weights. Cheaper than debugging "No space left on device" mid-deployment.

---

## Docker daemon log rotation is mandatory

Configure `/etc/docker/daemon.json` with json-file driver and bounded rotation on every host that runs Docker:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

Without this, a single misbehaving container can fill disk and break unrelated services. Phase 1.x discovered the failure mode: vad-consumer's broken state produced 100GB of log spam in a few days because Docker had no rotation policy. Audio-ai's root partition filled, asr-consumer couldn't write features, the conditioning pipeline was effectively dead.

Applies on container recreate, not just restart. Roll out cluster-wide.

---

## Logs lie about what's there until you ship them

The Phase 1.x Loki deployment found, within minutes of the dashboard going live:

- A polling service nobody remembered existed (world-ai-occupancy)
- A crashlooping script (occupancy_poll.py — schema mismatch)
- A token-rotation gap (three places, only one rotated)
- A prompt regression (cortex-creative violating two documented rules)
- A logging firehose burning CPU and disk for no diagnostic value (vad-consumer)
- A documentation drift (a third world-ai service not in ARCHITECTURE.md)

None of these were on the day's intended work list. All surfaced because logs that previously vanished into `/tmp` tmpfs or got lost in docker-compose stdout suddenly became queryable.

**Pattern:** First-time observability deployments find more bugs in the first hour than they were built to expose. Plan for it. Don't deploy logging the day before a demo. Don't ship Loki and immediately add 30 alerts — let the surface stabilize for a week so you can distinguish normal-noise from actionable-noise.
