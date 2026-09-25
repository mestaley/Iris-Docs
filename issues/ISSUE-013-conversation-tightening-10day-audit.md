# ISSUE-013: Conversation tightening — 10-day Loki audit (2026-09-19)

- Status: **audit done; two fixes landed this session, three open**
- Area: cross-cutting — cortex-logical latency, conditioning gate/intent, volition proactive voicing
- Source: 171 turns w/ decisions over 10 days pulled from Loki (`monitor-01:3100`), 124 with user text.
  Method: fusion `decision after` lines (latency/strategy/say/conf) joined by turn_id with curator
  `FS arrived text=`, plus signal greps (got_away / correction / actuation_failed / punt / silent).

## Headline numbers
- **Latency (fusion decision): p50 4.6s, p90 6.9s, p95 9.2s, max 13s. 70% of turns >4s.**
  cortex-logical `elapsed` is ~70% of it: p50 3.2s, p90 4.3s, p95 5.7s.
- Strategy mix: logical_dominant 66%, neither_confident 33%. Empty-say 37% (fine on actuation → "Done.").
- Signals over 10d: got_away 18, correction 29, actuation_failed 9, proactive_punt 15, shadow_punt 9,
  **silent_clause 0** (the kokoro fix held).
- Degenerate ASR turns (≤1 content word): 16/124 (13%) — "Yeah.", "six", "Player.", "?", plus
  mis-hears ("The great fuck cool play.", "Will you take my Bible?"). 5 of them triggered a deep punt.

## FIXED this session
1. **Proactive briefs read aloud as prompts (ISSUE-011 regression).** Since creative retired, fusion
   voices `proactive_brief` verbatim, but the briefing offer/reoffer + welcome briefs were written as
   model instructions/facts ("Ask Matt, warmly and in one short line…", "The household just got home
   after…"). Rewrote to final speech. Fixes "she reads her prompt out loud for the morning briefing" +
   "thinking out loud." fusion `feat/punt-escalation` (volition briefing.py, originate.py). **Needs
   iris-volition restart.**
2. **Commands mis-routed into reminders → fired "out of nowhere."** The cortex called set_reminder on
   immediate commands (`text='turn on the music…' when='now'`, plus empty `text='' when=''` junk);
   `dateparser` resolves "now" to the current instant, so the "reminder" fired straight back out as
   proactive speech. Guard: `resolve_when` rejects immediate phrases (now/right now/asap/…), intake
   rejects empty text (both soft-fail, nothing queued); prompt: set_reminder description forbids
   immediate commands. `feat/punt-escalation` (volition reminders.py/intake.py, curator tool_catalog.py),
   +2 tests, 26 green. **Needs iris-volition + iris-curator restart.**

## OPEN — ranked by impact

### A. Latency (the dominant felt problem) — RECOMMENDED NEXT
cortex-logical's two-pass native-tools flow (chat_with_tools → wrap_final) on qwen2.5:14b is ~70% of
the ~4.6s p50. The reflex actuates in ~800ms, but Iris's SPOKEN "Done." waits the full fusion decision.
Proposed fixes (scope, not yet built):
- **Reflex-confirmation short-circuit:** on a reflex-actuated turn where logical will come back empty
  (37% do), let fusion confirm "Done." as soon as the reflex result lands instead of waiting the
  deadline. Contained change in fusion `_fire`/ready-to-fire; guard against talking over a logical
  reply that DOES carry prose.
- **Single-pass path for trivial/actuation turns** (the Phase-2.5 CLAUDE.md escape hatch): skip
  wrap_final when tool_calls present and no prose is needed.

### B. ASR noise admitting junk turns (13%)
TV bleed / partial captures ("Yeah.", "six", "?") reach the pipeline as full turns. Cheap guards:
a min-content gate (drop ≤1-content-word turns unless in an open follow-up window) + the banked
speaker-gate re-enroll (ISSUE-001). Ties to the perceived-quality problem.

### C. Deep-punt on garbage
5 proactive punts fired on degenerate input ("Yeah.", "six") and mis-hears. Gate deep escalation on
input quality (length/ASR confidence) — don't spend the deep tier on noise. Cheap salience pre-filter.

### D. Backyard light actuation_failed (spot-check)
09-12, `logical_dominant` + actuation_failed on "turn the backyard light on" — smells like an
entity-mapping miss. Verify the backyard light entity id/mapping.

## Good news confirmed
- got_away by day: 09-10: 7 → 09-11: 1 → 09-12+: 0. The got-away + double-fire fixes held in prod;
  the only stragglers were multi-room music, since fixed by speaker grouping.
- silent_clause = 0 over 10 days (kokoro 0-byte fallback fix holding).
