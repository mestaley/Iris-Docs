# ADR-003: Local serial-spine cognition; contention deferred to a rented lab
- Buildable design (incremental migration, code-grounded): [`design/COGNITION-REARCH.md`](../design/COGNITION-REARCH.md)
- Status: accepted (direction); **PARALLEL build with the web surface** (ADR-002 / ISSUE-008), coordinated by the shared text→fabric-bridge contract
- Date: 2026-08-24
- **2026-09-04 — the "punt"/escalation model here is REFRAMED by [ADR-004](ADR-004-automaticity-and-attention.md)**: "punt" was selector-thinking; it becomes a salience-gated *parallel* deliberation that refines (not replaces) and surfaces *deferred* (not same-turn barge-in). The serial-spine, local-first, and creative-retirement decisions below still stand.

## Context
The two-cortex design — logical `qwen2.5:14b` + creative `gemma3:12b` — does not deliver the
thesis's cognitive pluralism. "Creative" is a **voice-only rendering role**: no tools, no
world-state reasoning (`CLAUDE.md`: "creative doesn't propose tools"). A rendering slot is
structurally incapable of contributing a second *cognitive* perspective, whatever model sits in
it — so the pluralism was never actually built (one reasoner + one stylist, labeled a weave). And
Gemma can't fill the reasoning role either: a 12B next to the 14B is too thin a perspective to
earn the weave, and it's the source of the "articulate lie" (ISSUE-005 — it speaks before it can
see tool results). Multi-turn design discussion (2026-08-24) reframed the whole cognition stack.

Hardware: core-ai = 2× RTX 5080 **16 GB** each (GPU0 logical+embed, GPU1 gemma+phi3.5); world-ai
holds the `qwen2.5:7b` fast-path.

## Decision
**Build the most capable, efficient LOCAL cognition with the VRAM at hand — a serial spine, not
parallel pluralism.**
1. **Fold the thalamus into a capable tool-calling front-door model.** Routing becomes a
   *tool-call* ("handle it now vs. escalate"), not a phi3.5 *classification* — which fixes the
   ISSUE-002 routing weakness and removes the fastpath↔logical double-fire (the front door is the
   reflex; it escalates deliberately). The front door owns routing + tool-calling + **voice** +
   every turn it can handle itself.
2. **A MoE slow path** as the escalation/deep tier for genuinely hard turns (on the freed GPU1).
3. **Retire the creative-as-renderer cortex** (Gemma out; frees ~10 GB). Voice moves to the
   front-door/logical model — utilitarian voice is fine; the flourish was never valued.

**The thesis fixes the goal (situational intelligence), not the topology.** It is explicit —
"cortex configuration is not load-bearing," "substrate implementation is not predetermined."
Parallel pluralism was one *mechanism*; serialism is a legal realization of the same goal.

**Reconception of "cognitive pluralism" (Matt, 2026-08-24) — it's escalation, not parallel
contention.** Pluralism here means *the right depth of cognition for the turn*, and the
intelligence lives in the **punt decision** — NOT many models arguing in parallel, and NOT one
model grinding extra passes it's no better equipped for (a 2nd pass of the same model isn't
smarter; punting to a bigger model is). Architecture: the front door handles the turn in one
efficient pass, and **knows when to punt** to the deep tier.

**Framing (the elevator pitch):** this is effectively a **Mixture-of-Experts lifted from the
token/weight level to the turn/system level** — but the gating is an **explicit, inspectable,
programmable punt policy** (affect / `escalate` tool / verify-backstop), not a learned black-box
softmax, and the "experts" are heterogeneous whole models on a priority-scheduled DDS bus. Sparse-
activation efficiency *without surrendering routing control to the weights* — the ADR-001
diagnosis-surface instinct applied to cognition. In ways it's more than a real MoE: heterogeneous
(even rentable) experts, a router that sees affect/history/failure-feedback, and *reactive* routing
(try-then-punt after seeing a weak answer — which a real MoE, committing before it computes, cannot
do). Recursion worth noting: the deep expert is itself a real MoE — explicit gating outside, learned
gating inside.

**The core hard problem is the punt decision — metacognition: how does a model know a turn is
beyond it?** (Models are overconfident exactly when wrong, so this is non-trivial.) Punt-signal
design space:
- **Escalate-as-a-tool** — the front door calls an `escalate`/`deliberate` tool when it judges
  the turn hard (routing-as-tool-call pointed at *depth*). Clean; leans on self-judgment.
- **Structural difficulty** — compound/multi-step, ambiguity, novelty-vs-routine, long context.
- **Affect (thesis-relevant, live)** — prosodic affect is LIVE; frustration / emotional register
  is a strong punt signal. Hard-conversation validation targets should always punt. Ties the punt
  decision to the thesis's affect commitment.
- **Self-confidence** — cheap but poorly calibrated; don't trust alone.
- **Reactive / failure-driven** — attempt, then punt if the answer fails / verify flags it.
  Reliable but pays the failed attempt's latency.
Tension: **proactive** punt (fast, needs good difficulty estimation) vs **reactive** (reliable,
slower). Lean hybrid — proactive on the cheap strong signals (affect, obvious complexity, an
explicit `escalate` call), reactive as backstop. The punt is **serial and late-tolerant**: the
deep tier runs async and **barges in** with the better answer when it lands (reuse the existing
barge-in). Existing primitives to build on: `needs_clarification` (→ a structured
`needs_escalation`), the logical supervisor (reactive-escalation primitive), and barge-in.

**The punt bridge (Matt, 2026-08-25) — speak a GENERATED holding line while the deep tier thinks.**
A punt means a slow answer; the user shouldn't sit in silence. On punt, the front door (14b) does a
quick second pass to generate a **situation-specific** "thinking out loud" bridge — NOT a canned
array/dict of "one moment" fillers (generic + repetitive kills the persistent-presence feel).
Speak the bridge immediately; the deep tier's real answer **barges in** when it lands. Architecture
fit (same as everything): bridge = a normal `FusionDecision.say` published on punt; deep answer =
a late `FusionDecision` correction/barge-in — the existing machinery. It's literally the thesis's
"async streams of consciousness interleaved" made audible: fast acknowledgment, then deep answer.
Design notes: the bridge must ACKNOWLEDGE + buy time in character, never promise a specific outcome
it doesn't yet have ("the custody schedule's a tangle — let me actually work through it", not "the
answer is X"); the bridge pass is short-output so it's ~sub-second (covers the multi-second deep
wait). Scoped: only PUNTED turns get a bridge — normal front-door turns just answer. Lands WITH
the deep-tier build (step 4).

**Contention / pluralism = deferred to a RENTED experimentation lab, not local hardware.**
It is the unproven capability — never worked as built, and possibly not what Matt wants. Don't
commit scarce local VRAM to it. Two reasons to rent rather than cram it locally:
- Local 16 GB could never test pluralism *fairly* (two crammed small models is a strawman). If
  contention pays off at all, it's with capable, *diverse* models — only rentable at scale.
- **Prove-before-you-buy:** the rented experiment is what decides whether contention ever earns
  local silicon. Commit a GPU to it only if the experiment proves its value.

**Local-first boundary (thesis, load-bearing).** Production Iris stays **100% local**
(front-door + MoE). The rented GPU is a **lab, not the product** — renting to decide what to build
≠ renting to run Iris. Guardrail: keep real household data off rented iron; experiment with
sanitized versions of the validation-target scenarios. If contention ever proves indispensable,
the thesis-consistent endgame is buying the local GPU for it.

## Extensible by expert (a property of the decision, not a someday-maybe)
The explicit router makes the pool extensible along a **second axis — specialty, not just tier.**
Punt on *depth* (escalate hard turns) AND route on *kind* (creative → a creative expert; later
code/research/etc.). Adding a specialist is **additive and non-disruptive** — a route + an expert,
no rearchitecture (impossible in a monolith or a weight-baked MoE). Concretely: **Gemma (or the
like) may re-enter later as a creative specialist** for creative-centric turns — it was never a bad
model, only mis-cast as always-on general voice. Discipline (ADR-001, again): a specialist earns
its slot only if **measurably better at its specialty than the general tier** (eval-tested) — it
returns on evidence, not nostalgia; a capable deep tier may write well enough that no creative
expert is warranted.

## Open tunable
Front-door latency: keep the world-ai `7b` as a sub-second reflex *in front of* the 14B-class
front door, or fold everything into the one model and accept ~1s on trivial turns. Cheap to try both.

**Two SEPARATE folds (Matt, 2026-08-25) — don't conflate:**
- **Fold 1 — thalamus (phi3.5) → the 14b (step 3).** Routing-as-tool-call (front door calls the
  query tools it needs itself; reworks the curator prefetch flow). Retires phi3.5 (frees ~4GB on
  its GPU, kills a hop, fixes ISSUE-002). Does NOT free world-ai. **Clear win — do it.**
- **Fold 2 — the world-ai `7b` reflex → the 14b.** THIS is what frees world-ai's GPU. Cost: trivial
  actuations ("lights on") go ~0.5s → ~2.5s (the most common command, slower). The endpointing win
  (ISSUE-009) does NOT erase this — it's ~2s of real processing after the endpoint.
  **Decision gate: what would world-ai's freed GPU be FOR?** If nothing pending → keep the reflex
  (snappy lights). If for a named use (a creative-specialist expert / vision / Frigate) → fold it.
  Lean: keep the reflex for now, measure the 14b front-door speed after Fold 1, fold only on a named use.
- Both are post-deep-tier. Interim: phi3.5 → GPU0 (frees GPU1 for the MoE now); Fold 1 later retires
  phi3.5 outright (freeing that GPU0 slot too).

**DECIDED (Matt, 2026-08-25): Fold 1 ONLY.** Do the thalamus→14b fold (retire phi3.5, routing-as-tool-call).
Do NOT do Fold 2 — keep the world-ai 7b reflex so trivial actuations ("lights on") stay sub-second; world-ai
GPU stays as-is. The 3-tier speed ladder holds: 7b reflex → 14b front door → MoE deep.

## Why this fits the existing substrate (DDS + RTOS) — near-zero friction
The punt/escalation model maps directly onto what's already built — a strong argument for it
(the primitives exist; the substrate was arguably ahead of the cognition model):
- **DDS (pub/sub, async):** the punt is a *message* — the front door publishes an escalation; the
  deep tier is another subscriber that produces a **late** answer that barges in on the existing
  speech/decision topics. DDS exists for "streams arriving at different times."
- **RTOS (SCHED_FIFO):** the tiers are a **priority hierarchy** — fast front door at high priority
  (bounded latency), deep MoE at low/best-effort priority (takes as long as it needs, never
  preempts the real-time path). Textbook mixed-criticality scheduling; the punt is a priority handoff.
- **Barge-in / "async streams interleaved":** the streaming machinery fit parallel-contention
  poorly (half of why fusion's v0/v1/v2 reconciliation is a mess) and fits escalation naturally —
  a late deep stream barging a fast one. The substrate was built for *this*.

## Consequences
- Simpler, more legible cognition: a serial ladder replaces the parallel v0/v1/v2 fusion race →
  smaller diagnosis surface (the ADR-001 win, continued).
- Grounds the voice (the voice model now sees tool results before speaking) → attacks ISSUE-005 at
  the root, not just via the prompt guardrail.
- Frees ~10 GB (Gemma) — which is also what makes a bigger *reasoning* model / the MoE fit.
- The contention question stays **open but isolated** (rented), decided by evidence, not conviction.
- Supersedes the two-cortex arrangement. Relates to ISSUE-002 (routing), ISSUE-005 (voice grounding).

## Not deciding yet / sequenced
- Exact models (front-door: a 14B-class tool-caller; slow: a MoE, e.g. Qwen3-30B-A3B-shaped) —
  size precisely against the freed 16 GB when implementation starts.
- **Order: PARALLEL with the web surface (ADR-002), not after it.** (Corrected 2026-08-24 —
  Matt: the UI's CRUD half is fine on the current brain, but its conversational/research half is
  worthless on top of the confabulating two-cortex stack, so they build together and land on the
  new brain.) The two builds are coordinated by ONE shared interface: the **text-turn-in /
  response-out / session-attach contract** (the bridge). The UI builds against it; this cognition
  rearchitecture swaps in BEHIND it — architecture-is-sovereign (HD-10), commit to the interface,
  both implementations conform. Build the new cognition as NEW code alongside the live 2-cortex
  (production/failsafe coexistence) and cut over only when validated (needs Matt's sudo).
