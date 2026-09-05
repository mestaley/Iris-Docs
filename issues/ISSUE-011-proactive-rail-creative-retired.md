# ISSUE-011: The proactive delivery rail is broken with creative retired

- Status: open (found 2026-09-05 while wiring ADR-004 deferred surfacing) — **blocks deferred
  deliberation delivery; also silently degrades welcome-home + briefing**
- Severity: high (proactive speech — a whole feature class — doesn't voice its content)
- Area: cognition / fusion-executive + curator + cortex (the proactive rail)

## The bug
Proactive (system-originated) turns — welcome-home, the events digest/notifications, the new
deferred deliberation (ADR-004) — voice their content through a **creative-only** path:
- The composed content rides `FeatureStream.trace.proactive_brief` → curator → `CuratedContext.
  proactive_brief`. **Only cortex-creative** read that brief and composed the spoken say (its
  "briefing mode"). `cortex-logical` has NO proactive handling (grep: zero refs).
- `fusion-executive` `ready_to_fire` for `origin=="system"` **waits for creative**
  (`service.py:182-183`): "Proactive turns are creative-only — logical is gated." Fusion never sees
  `proactive_brief` (it doesn't subscribe to the curated context).

**Creative was RETIRED 2026-08-25** (`FUSION_EXECUTIVE_ENABLE_CREATIVE=false`, service stopped). So
on a proactive turn: creative never responds → `ready_to_fire` never true → the turn stalls to the
`turn_deadline_seconds` (~25s) fallback → fires `neither_confident` → **voices the CANNED_FALLBACK,
not the brief.** The proactive content is never spoken.

This has been latent since creative retirement (welcome-home/briefing are dry-run or rare, so nobody
heard the regression). ADR-004 deferred surfacing surfaced it because it depends on this rail.

## Fix options
1. **cortex-logical handles the proactive rail** (most aligned): give logical a proactive branch
   that composes/voices `ctx.proactive_brief` in character; curator stops gating logical on proactive
   turns; fusion `ready_to_fire` fires on logical for system turns. A cortex voices it "in character."
2. **fusion voices `proactive_brief` verbatim** for system turns when creative is disabled: fusion
   subscribes to the curated context (or the brief rides through to fusion), and a system turn with a
   brief bypasses synthesis → `say = proactive_brief`. Simplest; the brief is already fully composed
   (volition for briefings, the deep model for deliberations — verbatim is arguably *better* for a
   deliberation you don't want paraphrased).

Recommend **(2)** for deliberations (don't paraphrase a considered take) and as the reliable rail;
optionally (1) later for briefings that want in-character phrasing.

## Related / dependencies
- Blocks: ADR-004 deferred deliberation *voicing* (the deep-mode POST side is DONE, `feat/deep-mode`).
- Also gated by `VOLITION_EVENTS_DRY_RUN=true` (delivery is dry-run) — separate flag, Matt's.
- Interim: deep-mode `deferred_delivery=true` already STOPS the jarring barge-in (net win) even
  before this rail fix; deliberations just queue silently until the rail voices them.
