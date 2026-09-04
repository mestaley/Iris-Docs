# ADR-004: Automaticity vs. attention — kill "punt", weave a salience-gated parallel deliberation

- Status: **proposed** (direction, 2026-09-04) — reframes the escalation model of [ADR-003](ADR-003-local-serial-cognition.md); ADR-003's serial spine and local-first stance stand. **First increment LANDED 2026-09-04** (fusion `feat/punt-escalation`, staged): see bottom.
- Origin: Matt, driving. "Why are we punting at all… I'm thinking about one thing, not where to turn, not the lights — that's automatic. Iris needs to do that with the entire sensory fabric."
- Date: 2026-09-04

## Context — "punt" smuggled selector-thinking back in
ADR-003 introduced a **punt**: the fast front door decides it's out of its depth and *escalates* to
the deep tier. In practice that is a **selector** model — pick the better model, discard the loser —
and the thesis is explicit that fusion is a **weaver, not a selector** ([THESIS](../THESIS.md):
"MAITI = async streams interleaved; fusion=weaver not selector; the cortex stutter is a feature").
Punt drifted from the thesis, and it produces exactly two ugly outcomes, both observed live:

1. **Front-door result dropped → stall.** On a punt the front door yields and Iris says a filler
   ("Hm. Let me think about that.") while the deep tier computes. This is the "I give up" model; the
   filler is its symptom (see the whole 2026-08-26 → 09-01 fight against openers/fillers).
2. **Front-door executes, deep result computed then discarded.** Feels like waste — "why run it?"

Neither is intrinsic; both come from treating deliberation as a **replacement** for the fast answer
instead of a **parallel refinement** of it.

## Decision
**Retire "punt" as a concept.** Replace the escalation model with an **automaticity-vs-attention**
architecture, gated by **salience**, with deliberation that **refines and surfaces deferred** rather
than replacing and barging.

### 1. Three tiers by role, not by confidence
- **Automatic layer (reflex / fast-path):** massively parallel, always-on, subconscious. Presence,
  lighting, routine actuation, ambient sensory fabric. Handles the *vast majority* of turns and
  **never touches cognition** — the cerebellum keeping you in-lane while your cortex is elsewhere.
  **Actions commit here, fast, and deliberation never un-commits them.** (Driving: the turn signal,
  the lane, the lights — automatic.)
- **Front door (stream of consciousness):** singular. Answers the one thing in focus **now**, and
  **never stalls** — it always gives its immediate best, never a filler placeholder.
- **Deep (deliberation):** engaged **rarely**, **in parallel**, for the genuinely hard/salient thing.
  It **refines** the verbal/reasoning layer; it does not replace the answer and does not re-actuate.

### 2. Parallel, not sequential — speculation, not hand-off
The deep tier does **not** need the front door's answer ("does big daddy care what the first model
thinks?" — mostly no). Run them concurrently. The front-door result is never dropped; a discarded
deep result is **speculative execution** (a CPU throws away every mispredicted branch — not waste,
because speculation is cheap relative to the win **when deliberation is rare**).

### 3. Gate on SALIENCE, not front-door uncertainty
Today the proactive gate fires on `self_uncertain` — "the front door felt shaky." That is a
**give-up** signal and it is backwards. The right question is attention: **is this the one thing that
deserves conscious deliberation, while everything else runs automatically?** Affect and complexity
are salience signals; `self_uncertain` is not. **Drop `self_uncertain` from the gate.** The hard,
unsolved work is a real **attention/salience mechanism** (not a confidence threshold) — that is where
effort should go, not punt-threshold tuning.

### 4. Deliberation surfaces DEFERRED, not same-turn (for voice)
The barge-in-10s-later-and-override is jarring *for voice* precisely because it violates the driving
model: in the moment you want the automatic answer *now*, not a correction while merging. So the deep
tier's output should **not** land in the live turn. Like a person: quick take now, and later —
*"been chewing on that thing you asked…"* — the considered version, delivered **proactively through
the Volition/events channel we already built.** Same-turn correction is reserved for the **reflective
surface** (web/research), where a multi-second deliberate answer is what was asked for. This also
retires the "deep tier on the voice hot path" tension (it always fit the web surface better).

## Consequences
- **Expand the automatic layer** — the biggest, least-glamorous lever. Most of the sensory fabric
  should be handled subconsciously and **never clutter the conscious prompt** (also attacks the
  over-prefetch noise problem from a new angle — ISSUE-002).
- **Singular attention ⇒ a serial resource**: one deep thought at a time (a scheduler), while the
  automatic fabric is parallel (a fan-out).
- Deep-mode becomes a **producer of deferred proactive events**, not a live-turn barge-in — wiring it
  to Volition rather than to `iris/speech/utterance` on the hot path.
- ADR-003's escalation/`PuntRequest` mechanism is **reframed**: the topic can carry a *salience*
  escalation to a deferred deliberator; the "front door gave up" semantics are gone.

## Open (the real work)
- **The salience/attention mechanism** — what earns conscious deliberation, computed cheaply. Unsolved.
- **Cost model** — parallel speculative deep is only affordable if it's rare; the gate must keep it rare.
- **Deferred-delivery UX** — how/when a later "I've been thinking…" surfaces without being intrusive
  (ties to the Volition arbiter + [[iris-bias-toward-action]]).

## First increment (cheap, high-signal, no attention mechanism yet)
**Drop `self_uncertain` from the proactive-punt gate, and make the front door never emit the filler.**
That alone stops the "give up and stall" behavior today (fusion `_proactive_punt_reason` +
`neither_confident`/`CANNED_FALLBACK` → always the front door's best say). The full parallel/salience/
deferred model lands incrementally after.

## First increment — DONE (2026-09-04, staged)
Landed in fusion (`feat/punt-escalation`, needs `sudo systemctl restart iris-fusion-executive`):
- `_proactive_punt_reason`: **`self_uncertain` removed** — gate fires only on salience (affect / complexity).
- `_synthesize` + `synthesis._select_say`: **voice any present say** (no confidence floor) — the front door never suppresses its immediate answer for a filler.
- Fallback text: think-stall → honest retry-invite ("Sorry — that one got away from me. Say it again?"), fired ONLY on a genuinely empty front door. 41 fusion tests green.

Still ahead (the real project): the **salience/attention mechanism**; **deferred/proactive** deep surfacing (wire deep-mode → Volition, off the voice hot path); **expand the automatic layer**.
