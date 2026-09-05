# Design — The Recall Queue (offer-based surfacing + a unified "things Iris is holding")

_Direction from Matt, 2026-09-06, after the ADR-004 deferred deep-dive landed live. Refines
[ADR-004](../decisions/ADR-004-automaticity-and-attention.md) (deferred surfacing) and unifies it
with tasks and future research requests. Design note, not yet built._

## The shift: offer, don't dump
The deferred deep-dive works, but it **auto-delivers** the full ~30s analysis when the arbiter fires.
Better: Iris **offers** it and waits for a yes.

> "I've got a deeper read on that holiday-weekend weather — want to hear it?" → *(Matt: "yeah")* → the
> full day-by-day analysis.

The considered result is **held**; only a short **offer** surfaces first; delivery happens **on
confirmation**. This mirrors the conservative Volition offer pattern (ask-&-confirm) and respects
singular attention — a minute-long analysis shouldn't barge in unbidden, even deferred.

## The unifying idea: a Recall Queue
In Matt's framing, the same durable queue holds **everything Iris is carrying to surface or act on
later** — one substrate, several item kinds with different lifecycles:

| Kind | Example | Lifecycle |
|---|---|---|
| **deep-thought** | the deferred deliberation ("been chewing on that…") | **ages out** — TTL, drops if stale/undelivered |
| **task / list item** | "add Swiffer dusters", "call the vet" | **persists** until done/removed |
| **research request** | "look into X for me" → a bigger model, returns later | pending until the result lands, then becomes an offer |

Each entry carries the **offer text + turn_id + context** so that, when surfaced, a confirmation
resolves back to the held payload (the full analysis, the task detail, the research result).

This is not a new store from scratch — it is the **generalization of the events queue**
(`volition/events`: durable WAL sqlite `PendingEvent` with `kind`, `priority`, `expiry`/TTL,
`holdable`, the delivery arbiter). Tasks live in a sibling table today; the recall queue is the
conceptual umbrella over both, plus the new research kind.

## Two items on the critical path
1. **True conversational follow-up without a wake word.** For "want to hear it?" → "yeah" to work,
   the offer must open a **no-wake-word answer window** (the await-answer / continuation window from
   the conversational-turn-taking work). Without it, confirming an offer would need "Iris" again,
   which kills the naturalness. **This is prerequisite #1.**
2. **Store the offer + turn_id + context in the recall queue.** The offer surfaces, but the payload
   (deep result / task / research answer) is held keyed by turn_id/context, so the confirm — arriving
   in the no-wake-word window — resolves to it and delivers. **This is prerequisite #2.**

## Research requests → an escalation tier beyond the local deep tier
A "research this for me" item dispatches **asynchronously to a bigger model** — a cloud-hosted ~70B,
or the Claude / GPT APIs — and lands its result back in the recall queue as an offer when ready
("I looked into X — got a few minutes?"). This is the natural next rung above the local deep tier
(ADR-004's parallel deliberation): local for the quick considered take, cloud for the genuinely
heavy research. Async + offer-based, so latency never touches a live turn. (Local-first still holds:
the cloud tier is opt-in per request, for depth the local silicon can't reach.)

## How it composes with what exists
- **ADR-004 deferred surfacing** — change deep-mode's delivered event from the full brief to a short
  **offer**; hold the full analysis keyed by turn_id; deliver on confirm.
- **Events queue / delivery arbiter** — the substrate; add an "offer" delivery shape + a
  confirmation-resolves-payload path. `deep-thought` uses TTL/aging (already `expiry`).
- **Conversational turn-taking (no-wake-word window)** — restore/extend it to gate offer confirmations.
- **Tasks** — federate under the recall-queue umbrella (or keep the table, unify the read/surface view).
- **Curator `recent_conversation` / recall** — the recall queue is the durable, actionable sibling to
  the semantic memory recall.

## Open questions
- One physical store (evolve `PendingEvent`) vs. a queue that federates events + tasks + research.
- Confirmation grammar: yes/no/"later"/"remind me" — and how "later" re-queues.
- Offer batching: several held items at once → one "a few things I'm holding for you…"?
- Aging policy per kind (deep-thoughts expire; tasks don't; research results expire slower).
- The cloud-research tier: provider abstraction, cost/consent, where the request originates (voice
  "look into…" vs. the web surface), and how the result's provenance is voiced.

## Sequencing (rough)
1. **No-wake-word follow-up** working reliably (prereq #1) — the biggest single unlock.
2. **Offer-before-deliver** for the deep-thought (deep-mode emits an offer; hold payload; confirm →
   deliver) — smallest end-to-end slice of the recall queue on the existing events substrate.
3. **Generalize** the queue (kinds, aging, the surface/offer view) to absorb tasks.
4. **Research kind** + the async cloud/bigger-model tier.
