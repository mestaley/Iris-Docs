# Iris — Principles & Working Agreement

_Ported into iris-docs 2026-08-23 from the earlier `Architecture/README.md`. The durable
"how this project is built" content — vision, methodological commitments, what Iris is NOT,
and how to work on it. Stale topology prose and the old nine-doc reading order were dropped
(see `ARCHITECTURE.md` for the live system, this repo's `README.md` for the layout)._

## The target is Jarvis

The eventual target is not "a better voice assistant." It's a **persistent household
intelligence** with the properties of Jarvis as fictional reference — a presence that lives in
the home, accumulates understanding over years, knows the family, and is accessible through
whatever surface fits the moment. Voice is the first surface; web chat at the desk is the
eventual second. The same Iris across both, with shared accumulated memory and consistent
character.

Voice was the right first step because it is the hardest interface to fake. A chat wrapper
around an API is shallow. Voice forces the full stack — wake-word, noise-robust ASR, fast
inference, character-preserving TTS, conversational flow, identity continuity. Solve voice
first and everything else inherits the foundations.

> Note on framing: the two cortices do not feed a synthesis layer that "picks which to speak."
> Per [THESIS.md](THESIS.md), fusion is a **weaver, not a selector** — contention is preserved
> to expression, not collapsed by selection. Any older doc using "selector" phrasing predates
> that commitment.

## What Iris is NOT

- A general-purpose AI assistant — scoped to family voice, smart home, music, and eventually collaborative work with Matt.
- A SaaS product — she lives in Matt's basement, on Matt's hardware.
- An always-improving system — hand-tuned static configuration and explicit decisions, NOT continuous fine-tuning.
- A toy — she controls real lights, plays real music, knows the family's actual schedule, and is used daily.
- A Home Assistant voice skin or Rhasspy frontend — she's a cognitive architecture (see ARCHITECTURE.md).

## Safety invariants (never violate)

Hard constraints on what Iris may touch, independent of any model/prompt/mapping. Enforced in
code, not by prompt instruction — a model can be wrong, so the wall is below the model.

- **The `lock` domain is off-limits — never actuate, never read.** The only HA lock is the z-wave
  deadbolt, present solely for the UniFi doorbell fingerprint→unlock automation (HA-internal, HA's
  own token). Iris must never unlock the door nor read the lock's state. Enforced at world-ai's
  `/ha/service` proxy (`config.blocked_domains`, default `lock`) — the single holder of the HA
  token and the chokepoint every actuation funnels through, so nothing in the fabric can bypass it.
  Corollary: **never seed a `lock.*` entity into `iris_mappings` or `iris_tools`.** A broad reseed
  from HA states must exclude the blocked domains, or Iris regains a path to the door.

## Working principles

Methodological commitments that shape every decision. Not features — how the project is built.

**The architecture is sovereign.** Components that don't yet exist have their place in the
design; build the system around their eventual presence and fill missing pieces when their time
comes. Implementation conforms to architecture. _(Bounded by [ADR-001](decisions/ADR-001-continue-vs-nuke.md):
don't leave dead code wired into live paths — the diagnosis surface is the cost.)_

**Data logging is a precondition for integration, not a follow-up.** Every component logs from
day one — operational logs (Loki, severity-based) for health, cognitive event logs (structured,
category-based) for what Iris is doing in cognitive terms. A component that doesn't log isn't
integrated.

**Cognitive and operational logs are architecturally separate.** Operators answering "is the
service healthy" vs. memory consolidation / cognitive dashboards / pluralism research. Different
transport, consumers, retention. Mixing them defeats both.

**Daydream-driven design with real-stakes validation.** Architectural commitments are tested
against concrete, real-stakes use cases that matter to specific humans, even years from
implementable. Custody-agreement processing is the canonical case — not on the roadmap, but a
test the architecture must eventually pass. (See the validation targets in THESIS.md.)

**Biology as engineering reference, not metaphor.** Sleep-driven consolidation, episodic/
semantic split, perception as active filtering, executive function as synthesis rather than
arbitration — adopted where they fit our engineering constraints, departed where they don't. The
default disposition is to take biology seriously as a teacher.

**Local-first is not ideology.** Iris is Matt's. A persistent household intelligence that knows
the family and accumulates years of understanding cannot be a service rented from someone else.

## Working agreement

Matt's working style: technical, direct, no hedging. Treat him as a capable adult who wants
things done. Dry humor and honest pushback welcome. Long debugging sessions are normal;
three-message exchanges about typography are not.

When something breaks, the diagnostic order is always:
1. Check what's actually happening (logs, ps, nvidia-smi, ss, curl).
2. Reproduce minimally.
3. Read the existing artifact (this repo, `decisions/`, prior code) before guessing.
4. Propose the fix in plain language, then patch.

Never propose architectural changes without reading the decisions first. Most of what feels
missing has been considered and either deferred or rejected — the reasons are written down
(here, in `decisions/`, and in the memory files).
