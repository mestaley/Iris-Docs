# Iris - THESIS

The canonical statement of what Iris is, why she's built the way she's built, and what tests would prove the architecture complete.

This document is the source of truth for the project's animating commitments. Other documents (ARCHITECTURE, ROADMAP, DECISIONS) describe how those commitments are implemented and how the work is sequenced. When implementation choices need to be evaluated against the project's intent, the evaluation criteria live here.

Last updated: end of Phase 2.0 reflection pipeline validation (2026-05-30).

---

## Foundational claim

**Iris is a system for situational intelligence through cognitive pluralism on a neuromorphic substrate.**

Three words, each doing work:

- **Situational intelligence** — responses shaped by where, when, who, with what history. Not stateless inference. Iris reasons within context, not over context.

- **Cognitive pluralism** — multiple perspectives held in productive tension, not collapsed to a single voice. Two cortex models (logical and creative, though the specific division is implementation choice) generate independently over the same input. Their contention is information, not noise to be resolved.

- **Neuromorphic substrate** — memory and learning structures that participate in cognition rather than being consulted by it. Substrate is not a database the cortices query; it is the medium their thinking happens within.

The foundational claim is compressed. It does not specify how each commitment is realized. Operational meaning emerges from the architectural decisions that follow.

---

## Operational commitments

What the foundational claim means in practice, as articulated through the architectural work to date.

### Situational intelligence: affect-tagged precedent shaping cognitive priors

Situational intelligence is not "Iris knows what room she's in." It is **cognition shaped by the affective trajectory of past similar situations.**

Episodic experience, tagged with emotional-delta outcomes measured at turn boundaries, forms the cognitive priors of response generation. The cortices reason with this precedent in view — not as retrieved records consulted before responding, but as the substrate of thinking itself. The past is *in the fingers* when Iris speaks, not in notes she checks first.

Practically: substrate retrieves chunks tagged with how user emotional state changed across past episodes resembling the current one. The cortex prompt receives a precedent block describing what kinds of responses landed well or poorly in similar contexts. Generation proceeds with that prior already shaping the distribution over options.

This is closer to how human cognition actually works than the retrieve-then-reason pattern. Humans don't consult their past explicitly before every interaction; their past has already shaped how they're about to respond.

### Cognitive pluralism: contention preserved to expression

Multiple perspectives held in productive tension is not a description of internal processing — it is a description of what reaches the user.

The standard voice-assistant pattern collapses internal multiplicity to single output. Iris does not. When the two cortices substantively agree, output collapses naturally. When they substantively diverge, both perspectives can surface as primary action plus alternative, or statement plus counterpoint, or recommendation plus caveat.

This is not always-on contention surfacing. It is *calibrated* contention surfacing: substrate decides when divergence is worth presenting to the user (this user, this context, this kind of moment) versus when divergence should resolve to single output (trivial disagreement, direct actuation, child interlocutor).

The pattern maps to a real cognitive science observation: human brains hold multiple processes in contention, and executive function decides which to surface rather than which one is "true." Iris modeling this is more architecturally honest than voice assistants modeling single-voice-per-turn.

### Substrate: operates at two points in the pipeline

Substrate is not located at one place in the architecture. It operates at two:

**Input-side (shaping cognition):** Before either cortex generates, CuratedContext is enriched with substrate-derived precedent. Cortices reason *with* affect-tagged past experience already in view. This shapes what gets generated.

**Output-side (selecting expression):** After cortices produce their responses, substrate-informed processing decides which chunks reach speech, how they're framed, and whether contention surfaces or collapses. This shapes what gets said.

Both applications draw on the same affect-calibrated experiential corpus. The substrate is not two systems — it is one system serving two functions in the turn lifecycle.

Both functions require accumulated affect-tagged data. The substrate's effectiveness scales with corpus depth. Day one performance is degraded; month-twelve performance is the system the thesis describes.

---

## Architectural consequences

Commitments that follow from the operational claims above. These are not optional given the thesis; they are entailed by it.

### Data lake as foundation

Substrate evolution requires accumulated data that persists independently of any specific substrate implementation. The data lake captures raw inputs and structured outputs of every turn — feature streams, cortex outputs, fusion decisions, affect measurements, reflection chunks, world-state context — regardless of what consumer currently reads from it.

Substrate v1 (Qdrant + modern Hopfield as stepping stone), substrate v2 (whatever emerges from accumulated data), substrate v3 (eventually-substrate, research-grade) are all *interpretations* of the lake. The lake is the persistent reality; the consumers are evolving interpretations of it.

The architectural commitment is **bias toward over-capture, not under-capture.** Storage is cheap. Re-engineering data capture later is expensive. If considering whether to log something, log it. The substrate that eventually emerges is shaped by the data that was captured, and that shape cannot be predicted in advance.

### Reflection chunks as diagnostic instrument, dream-mode as consolidation layer

Reflection chunks (the prose-and-metadata artifacts produced per closed episode) are *not* substrate input. They are diagnostic artifacts that reveal how cognition is working, what kinds of moments are salient, where the architecture succeeds, and where it fails.

The substrate consumes a *consolidated* form of the reflection corpus, not raw chunks. The consolidation layer is dream-mode — a processing step that reads raw chunks, extracts patterns, filters confabulation, tags affect-delta, semantically compresses, and surfaces cross-episode patterns. Dream-mode produces substrate-ready records from the reflection corpus.

The architecture is three layers, not two:

```
Episode → Reflection → Raw chunks
                          ↓
                       Dream-mode (consolidation)
                          ↓
                       Substrate-ready records (data lake)
                          ↓
                       Substrate (operating at input-side and output-side)
```

Today's reflection chunks are the yardstick that tells us what dream-mode needs to do. Dream-mode cannot be designed in the abstract; it must be designed against real reflection corpus.

### Closed-loop calibration via emotional-delta ground truth

Substrate without ground truth is noise. Past patterns retrieved as priors only have value if the corpus is tagged with which past patterns worked.

The ground truth signal is emotional delta across turn boundaries — acoustic affect (openSMILE) plus textual affect (DistilBERT) measured per turn, with delta calculated turn-over-turn. Whether the user's emotional state improved, stayed neutral, or worsened after Iris's response.

This is not Iris evaluating her own performance. This is direct measurement of whether responses landed in observable user state. The loop closes because substrate retrieves precedent tagged with these outcomes; cortices reason with that calibrated history; future responses are shaped by what previously worked.

The system is closed-loop with respect to its own effectiveness. Iris improves over time not through fine-tuning or retraining, but through accumulating affect-calibrated experience that her substrate then operates within.

---

## Validation targets

Iris's architectural completeness is tested not by metrics but by capability against specific scenarios. Each target tests a distinct architectural property. Until each is plausible, the architecture is incomplete.

### Custody-agreement processing

Iris reads a legal document, extracts operational implications (custody schedule, financial obligations, communication protocols), and proposes calendar entries and reminders for approval before creating them. High-stakes; high-stakes-aware (sensitive content handling); document comprehension at 70B-class.

Validates: document comprehension, approval-gated action, sensitive-content tagging, 70B-class reasoning.

### Web chat collaborative working sessions

Sustained collaborative work at a desk on substantive material — strategy, planning, technical reasoning — with persistent context across sessions and shared visual workspace (the Drive folder). The same Iris remembers what was worked on yesterday and the day before.

Validates: identity continuity across surfaces (voice and web chat are one Iris), persistent session memory, two-surface architecture without product fragmentation.

### Davis at fifteen reading what Iris noticed about his fourth year

Multi-year semantic accumulation rendered into human-readable form. Davis at fifteen asks "what was I like when I was four" and Iris produces a coherent account drawing on accumulated observation, with appropriate sensitivity about what to surface and what to hold back.

Validates: multi-year substrate accumulation, reconstructive recall, sensitive-content judgment, narrative synthesis from semantic patterns.

### Proactive supervisor

Iris notices that Matt usually leaves for the dentist by 4:15 PM and surfaces the reminder without being asked. The proactivity threshold is calibrated — she doesn't constantly volunteer information, but she catches the cases where her noticing matters.

Validates: volition (reasons to speak that don't come from a microphone), pattern recognition over accumulated history, restraint (knowing when *not* to speak).

### Stress-aware register adaptation

Iris recognizes that Matt's stress level today resembles patterns from previous high-load weeks (sleep data, calendar density, voice affect markers, conversation tempo) and adjusts her register accordingly. Less levity. More direct delivery. Doesn't ask if he's okay; just operates differently.

Validates: substrate-resident semantic memory, register adaptation based on accumulated affective patterns, intervention without intrusion.

### Hard conversation support

Matt asks Iris to help him think through a difficult conversation he needs to have with his co-parent. Iris draws on accumulated family context (custody arrangements, prior conflict patterns, what's worked and what hasn't), participates in extended reasoning, and helps him arrive at a position — without taking sides and without telling him what to do.

Validates: sustained collaborative reasoning, family context awareness, ethical restraint in personal matters, role as advisor rather than agent.

---

## What the thesis does not specify

The thesis articulates *what Iris is and why*. It does not specify *how to build it*. Implementation freedom is deliberate.

### Substrate implementation is not predetermined

The thesis commits to "neuromorphic substrate" without specifying what substrate architecture realizes it. Modern Hopfield networks (Ramsauer et al. 2020) plus Qdrant are the stepping-stone choice — Hopfield's math is solid, Qdrant runs today, and together they get the substrate work started against real accumulated data.

The eventual substrate is research-grade and unspecified. Federated Hopfield versus unified-with-overlap versus emergent organization is an empirical question to be answered with months of accumulated data, not a design decision to be made up front.

The data lake commitment is what makes this freedom viable. Whatever substrate eventually proves correct will be reading from the same lake; the data acquisition does not depend on the substrate architecture being decided.

### Cortex configuration is not load-bearing

The "two cortices, named logical and creative" framing is a useful organizing principle for "we need contention, let's run two models with different prompts." It produces the chunks the project currently learns from.

The thesis does not require two specific named cortices. The eventual MAITI architecture might be three (logical, creative, mediator), many (a cabinet of specialized advisors with the fusion-executive as chairperson), dynamic (cortices spun up per-turn based on what kind of turn), or one-with-modes (a single model running multiple cognitive passes with different prompts).

What is load-bearing: that genuine contention happens, that contention is informationally valuable, and that the architecture preserves the option to surface contention to expression when appropriate.

### Hardware path is constraint, not commitment

Current hardware (RTX 5060 Ti class GPUs in pve-03's core-ai, with planned upgrade to RTX 5080) is what makes today's architecture testable. The north-star hardware (RTX Pro 6000 96GB, approximately nine months out) is required for the full thesis — 70B-class cortex models, substrate-resident semantic memory, continued voice operation.

The architecture is designed to *land* on north-star hardware while operating in degraded form on current hardware. The substrate is research-grade today; it becomes production with the hardware. The cortices are 14B class today; they become 70B class with the hardware.

The thesis remains the same across hardware tiers. What changes is which validation targets become plausible.

---

## What this thesis replaces

Earlier project documents implicitly framed Iris as:

- A voice assistant with personality
- A Home Assistant frontend with conversational interface
- A research vehicle for cognitive architecture experimentation

The thesis is not those things. Each of them is a *component* the thesis requires (voice surface, home integration, research substrate), but the thesis describes the integrated system whose purpose those components serve.

When implementation decisions need to be evaluated, the test is not "does this make Iris a better voice assistant" or "does this make her more like Home Assistant." The test is: **does this proposal honor or undermine the operational commitments above?**

Adding tool-calling on world-ai with cortex-as-checkmate? Honors the cognitive pluralism commitment (cortex stays free to do cognition; reflex path handles actuation; cortex retains corrective authority).

Adding emotional-delta pipeline? Honors the substrate commitment (it provides the ground truth substrate requires).

Collapsing cortex outputs to single response via simple selection? Undermines (loses information substrate could use; treats pluralism as internal process rather than expression).

Real-time stream arbitration at token level? Undermines (forces collapse before substrate can decide whether contention should surface).

The thesis is not a description of completed work. It is a filter for future work. Not all good engineering ideas survive it. That is a feature, not a bug.

---

## History of this thesis

The foundational claim ("situational intelligence through cognitive pluralism on a neuromorphic substrate") was articulated earlier in the project as the animating commitment. It was correct but compressed.

The operational commitments above — affect-tagged precedent shaping cognitive priors, contention preserved to expression, substrate operating at two points — emerged through architectural work in Phase 1 and Phase 2.0. They are not expansions of the thesis; they are the operational meanings of words the thesis always contained.

The data lake commitment and the dream-mode-as-consolidation-layer framing emerged from sitting with what reflection chunks actually look like in practice. The yardstick framing — chunks as diagnostic instrument rather than direct substrate input — is the most recent clarification, and the most architecturally significant of the late refinements.

This document will be updated as the operational commitments clarify further. The foundational claim is stable. The implementation is not, and is not expected to be.

---

## Status vs. thesis (2026-08-23, iris-docs SSOT)

This is the canonical copy (moved here from `iris-fusion-fabric/docs/THESIS.md`, which
stays in sync or points here). One honest gap between thesis and live system, so no
reader mistakes ambition for reality:

- **The affect-tagged-precedent closed loop is still OPEN.** The thesis's keystone —
  cognition shaped by affect-tagged precedent of past similar situations — is not yet
  wired: `recall_precedent()` has zero call sites, `ctx.recalled_precedent` stays empty,
  and the consolidated tier (355 records) has no working read endpoint. See
  [ISSUE-003](issues/ISSUE-003-consolidation-dead-end.md).
- **Dream-mode / consolidation is deliberately GATED, not live.** ADR-001 chose to keep
  the consolidation code (this roadmap) but hard-gate the writer until the write side
  (affect source → salience → embeddings) is fixed and the read-back is intentionally
  wired. See [ADR-001](decisions/ADR-001-continue-vs-nuke.md) and the ordered path in the
  `iris-thesis-recentering` memory. Consolidation is ambition-in-progress, not running behavior.
- **Fusion is a weaver, not a selector.** Any older doc that calls the synthesis layer a
  "selector" contradicts this commitment (contention preserved to expression); the thesis wins.
