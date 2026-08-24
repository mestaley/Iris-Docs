# Cognition Rearchitecture — design (fleshes out ADR-003)

_Buildable design for the serial-spine / punt-escalation cognition ("a system-level MoE with an
explicit programmable router"). Code-grounded, with an **incremental, reversible** migration path —
this touches the live cognition core, so never big-bang. See [ADR-003](../decisions/ADR-003-local-serial-cognition.md)
for the decision + rationale; this is the how._

## 0. Current state (live-verified 2026-08-24)
- **GPUs:** core-ai 2× RTX 5080, **16303 MiB each**.
  - GPU0: `qwen2.5:14b` 9.7 GB + `nomic-embed` 0.6 GB → **5.4 GB free**.
  - GPU1: `gemma3:12b` 10.0 GB + `phi3.5` 4.0 GB → **2.8 GB free**.
  - world-ai (separate GPU): `qwen2.5:7b` fast-path.
- **Pipeline:** conditioning → `thalamus` (phi3.5, routing) → `curator` (prefetch + reflex fast-path + v0/v1 gating) → `cortex-logical` (qwen2.5:14b, two-pass native tools) ‖ `cortex-creative` (gemma3:12b, voice) → `fusion-executive` → speech-output.
- **RTOS (SCHED_FIFO prio):** fusion 80 · thalamus 70 · curator 65 · logical 60 · creative 55 · memory 25 · tool-registry 20. Best-effort (SCHED_OTHER): dream-mode, reflection-coordinator, episode-buffer. → **the deep tier belongs at SCHED_OTHER**, exactly like dream-mode: it never preempts the FIFO real-time path.
- **HARD CONSTRAINT — `CortexResponse` is FROZEN** (`contracts/.../cortex.py:294`, v0.7.2 `extra=forbid`; audio-ai's conditioning consumes `FusionDecision`, which embeds it). *"do not add fields here"* (cortex.py:317). v1b already learned the reflex via a **dedicated topic** (`FASTPATH_FIRED`, topics.py:137 / `FastPathFired`, cortex.py:110) rather than a new field. **The punt signal must follow that pattern — a dedicated DDS topic, not a CortexResponse field.**
- **Signals that already exist** (usable without a contract change): `confidence`, `self_reported_uncertainty: float|None` (cortex.py — "affect-aware self-reporting, cortex may flag its own uncertainty"), `needs_clarification: bool`. The front door's uncertainty punt input is already on the wire.
- **The reflex already has a defer path:** `FastPathDecision.deferred` (world-ai `schemas/fastpath.py:38`) = "declined to act (ambient/ambiguous → a cortex owns it)." **The escalate is an extension of `deferred`**, not a new concept — `{act | defer-to-cortex | escalate-to-deep}`.

## 1. Target architecture
A **1-D-plus-1-D expert pool** with an explicit router:
- **Tier axis (depth):** reflex (world-ai 7b) → front-door cortex (14b-class) → deep tier (MoE, SCHED_OTHER).
- **Specialty axis (later, extensible):** general reasoning · creative (a Gemma-class specialist may return, routed by *kind*, if it out-evals the general tier) · future code/research.
- **Voice ownership moves to the front door.** Gemma retires; the front-door/logical model streams its own pass-1 prose for first-audio (it already produces `proposed_say`). Utilitarian voice is acceptable (Matt).
- **The router = the punt policy** (§3), explicit and programmable — the whole point vs a weight-baked MoE.

## 2. Front-door consolidation (thalamus → tool-call)
Today routing is a *classification* (`thalamus` phi3.5 emits `RoutingDecision`; the weak link behind ISSUE-002 over-prefetch). Target: routing is a **tool-call** the front door makes natively.
- The front-door model, per turn, either (a) calls the device/query tool + speaks (handles it), (b) defers, or (c) **calls an `escalate` tool** / sets the punt signal (§3). Prefetch stops being a separate classifier decision — the model just calls the query tool it needs (grounded by curator's entity/ambient injection, per [[PATTERNS]] "curator injects everything").
- **Reflex-fold tunable (NEEDS MATT):** keep the world-ai `7b` as a sub-second reflex *in front of* the 14b front door (fastest "lights on", ~0.5 s; two hops), **or** fold it in and let the 14b be the single front door (~1s on trivial turns, one hop, simpler). Recommend: **keep the 7b reflex initially** (it already works, `deferred` is the escalate hook) and measure whether the 14b-only path is fast enough to retire it. Cheap to A/B.
- **RoutingDecision's fate:** curator still needs prefetch/gating inputs. Option A — the front door emits an equivalent structured pre-pass (routing-as-tool-call output feeds curator). Option B — curator derives prefetch from the front door's actual tool calls (it already filters already-satisfied tools). Lean B (fewer moving parts); resolve in migration step 3.

## 3. The punt / escalate mechanism (the core problem — metacognition)
**Signal transport:** a new **dedicated topic `iris/punt/requested`** + a `PuntRequest` contract (mirrors `FastPathFired` — keeps `CortexResponse`/`FusionDecision` frozen wire stable). Carries `turn_id`, `reason` (enum: `self_uncertain|affect|complexity|verify_failed|explicit_tool`), `context_ref`. The deep tier subscribes; its answer returns as a late `CortexResponse`/barge-in on existing topics.

**Punt inputs (hybrid — proactive on cheap-strong signals, reactive as backstop):**
| Signal | Kind | Source (grounded) | Notes |
|---|---|---|---|
| `escalate` tool called | proactive | front-door tool-call | cleanest; leans on self-judgment |
| `self_reported_uncertainty` high / `confidence` low | proactive | exists on CortexResponse | cheap; poorly calibrated → don't use alone |
| affect (frustration/emotional register) | proactive | **`fs.affect` is LIVE** (prosody, `PROSODY_OBSERVE_ONLY=false`); flows FeatureStream→curator→fusion | thesis-relevant; hard-conversation targets always punt |
| structural complexity | proactive | front door / curator (compound, ambiguous, long) | heuristic |
| verify/supervise failure | **reactive** | logical supervisor + actuation-verify (fusion `on_tool_result`, the correction path cortex.py:202) | reliable; pays one failed attempt |

**Proactive vs reactive:** proactive punts before computing (fast, needs good difficulty estimation); reactive punts after a weak/failed answer (reliable, +latency). Ship reactive first (it's a safety net and reuses the supervisor), layer proactive signals as they prove out.

**DDS + RTOS mapping (near-zero friction — ADR-003 §"why this fits"):**
- Deep tier = a new service `iris-deep-mode` (or reuse a cortex worker) at **SCHED_OTHER** (best-effort, like dream-mode) — never preempts the FIFO real-time path.
- Punt = publish to `iris/punt/requested`; deep tier is just another subscriber that produces a **late** answer.
- Late answer **barges in** via the existing fusion barge-in/correction machinery (`interrupt_prior=True`, cause=`correction`; fusion already handles late stragglers + tombstones, service.py:361/488). A deep-tier answer is a low-priority stream barging the high-priority one — the mechanism already exists.

## 4. VRAM / model-fit plan
Free Gemma (10 GB, GPU1) and relocate phi3.5 (4 GB) to GPU0 (has 5.4 GB free) → **GPU1 becomes a clean ~16 GB for the deep tier.**
| GPU | Resident | Fit |
|---|---|---|
| GPU0 (16 GB) | 14b front door 9.7 + nomic-embed 0.6 + phi3.5 4.0* | ~14.3 GB ✓ (*only if phi3.5 still needed; if thalamus fully folds, drop it → 10.3 GB, roomy) |
| GPU1 (16 GB) | Deep MoE | see below |
| world-ai | 7b reflex (front, tunable) | unchanged |

Deep MoE fit on a clean 16 GB card:
- **Qwen3-30B-A3B-shaped, 4-bit ≈ ~18 GB → does NOT fit** 16 GB (flag).
- **3-bit ≈ ~13 GB → fits** with KV headroom. **FLAG:** 3-bit degrades quality; but this is the *slow* tier and (per ADR-003) the "is a bigger deep tier worth it" question is what the **rented** experiment answers — so run the local deep tier at whatever fits, and use rented big-VRAM to test the ceiling.
- **Expert-offload** (MoE cold experts → system RAM) is viable *because it's the slow best-effort tier* — lets a 4-bit run with a latency hit that's acceptable here.
- **On-demand load** (not resident) is also acceptable for the deep tier — it only fires on punts.
- **NEEDS MATT — exact deep model + quant.** A 24 GB card would make 4-bit clean; the 5080s are 16 GB, so local deep = 3-bit-or-offload until/unless hardware changes.

## 5. Migration path (incremental, reversible — never break live)
Each step ships and rolls back independently. Current baseline = 2-cortex (logical+creative) live.

1. **Reactive-punt backstop, deep tier OFF.** Add the `iris/punt/requested` topic + `PuntRequest` contract (additive, frozen contracts untouched). Wire the *reactive* signal only (supervise/verify failure publishes a punt) but with **no subscriber** — pure shadow logging of "this turn would have punted." **Rollback:** ignore the topic. _Value: measures punt frequency before building anything._
2. **Retire Gemma → single-voice front door.** Make `cortex-logical` the voice owner (stream its pass-1 prose); stop invoking `cortex-creative` on live turns (leave the service deployable). Frees GPU1. **Rollback:** re-enable creative invocation (one flag). _This is the ISSUE-005 root fix (voice now sees tool results) and the ADR-001 diagnosis-surface win — shippable on its own even if nothing else lands._
3. **Fold thalamus into the front door** (routing-as-tool-call). Front door emits routing via tool-calls; curator derives prefetch from actual calls (§2 option B). Keep phi3.5/thalamus deployed as fallback for one soak, then relocate/retire. **Rollback:** re-enable thalamus routing. _Fixes ISSUE-002 at the root; validate on the eval harness first (§6) given the over-prefetch history._
4. **Stand up the deep tier** (`iris-deep-mode`, SCHED_OTHER) subscribing to `iris/punt/requested`, answering via barge-in. Start with the reactive punt only. **Rollback:** stop the service — punts fall back to the front-door answer (no barge-in). _Deep tier failure is always safe: worst case you get the front door's answer, which is the status quo._
5. **Layer proactive punt signals** (affect, `escalate` tool, complexity) once reactive is trusted. **Rollback:** disable per-signal.
6. **(Later, separate) Reflex-fold decision** and **specialty experts** — each additive, each eval-gated.

Ordering rationale: step 2 (retire Gemma) delivers the biggest correctness win (grounded voice) and is the most reversible, so it goes first; the deep tier (step 4) is last because it's the only genuinely new capability and it's safe-by-construction (punt falls back to the front door).

## 6. Eval-harness plan for the punt policy
Reuse HD-6 (eval-as-bus-peer, real turns/tools, no mock seam) + the 7 creative regression cases + the multi-intent cases ISSUE-002 owes. Measure:
- **Punt precision/recall** — on a labeled set of "easy vs genuinely-hard" turns, does the policy punt the hard ones and leave the easy ones? (Shadow mode from migration step 1 gives the raw frequency; add labels.)
- **Does the deep tier actually improve punted turns?** A/B: front-door-only answer vs. deep-tier answer on punted turns, judged for quality/grounding. **This is the question the RENTED experiment scales up** (local 3-bit deep vs rented big/diverse deep) — production stays 100% local (thesis boundary).
- **No latency regression on non-punted turns** — the 90%+ that never escalate must stay as fast as today (proactive punt must be cheap; reactive adds latency only on the turns that were already going to be reworked).
- **Voice quality after Gemma** — the 7 creative cases must not regress below "acceptable utilitarian" (Matt doesn't value flourish, but shouldn't get worse-than-serviceable).

## 7. Open decisions for Matt
1. **Reflex-fold tunable:** keep the world-ai 7b as a sub-second front reflex, or fold into the 14b (~1s trivial turns)? (Recommend: keep, measure, retire if the 14b is fast enough.)
2. **Exact models:** front door (a 14b-class native-tool-caller — stay Qwen family, or diversify?) and the **deep MoE + quant** (30B-A3B at 3-bit local, offload, or on-demand?). 16 GB forces 3-bit/offload for a dense-ish deep tier; a 24 GB card would change this.
3. **Voice register after Gemma:** confirm utilitarian voice from the front door is acceptable, or does a creative specialist (Gemma-class, routed by kind) come in earlier than "later"?
4. **RoutingDecision fate** (curator prefetch): derive from the front door's tool-calls (lean) vs. keep a structured pre-pass.
5. **Deep-tier residency:** resident on GPU1, or on-demand load per punt (saves VRAM/heat, adds first-punt latency)?

_Nothing here is built or deployed — design only. All frozen contracts stay frozen (punt rides a new dedicated topic). Every step above is individually shippable and reversible._
