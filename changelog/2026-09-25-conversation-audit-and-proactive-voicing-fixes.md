# 2026-09-25 — 10-day conversation audit + proactive-voicing / reminder-misroute fixes

## Executive summary
Audited 10 days of real turns from Loki (171 turns) to find where to tighten Iris, then fixed the two
concrete bugs Matt flagged from listening: (1) she read her own prompt out loud for the morning
briefing / "thought out loud", and (2) she'd say commands "out of nowhere" (a fired reminder). Both
trace to the same place — proactive turns are voiced VERBATIM since creative retired (ISSUE-011), and
some briefs were prompt-seeds, plus the cortex was mis-routing immediate commands into reminders. Both
fixed + committed on `feat/punt-escalation`; **staged pending iris-volition + iris-curator restarts.**
Audit findings filed as ISSUE-013 (latency is the top open item).

## Changes by repo

### iris-fusion-fabric — branch `feat/punt-escalation`
- `volition/src/volition/briefing.py` — **changed**: briefing offer/reoffer rewritten from model
  instructions ("Ask Matt, warmly and in one short line…") to final spoken lines. They're voiced
  verbatim now, so the instruction was being read aloud. (commit e74c813)
- `volition/src/volition/originate.py` — **changed**: `welcome_brief` rewritten from a third-person
  situational note ("The household just got home after…") to natural welcome speech. (e74c813)
- `volition/src/volition/events/reminders.py` — **changed**: `resolve_when` now rejects immediate
  phrases (now/right now/asap/at once/…) as None — an immediate "when" is a mis-routed command, not a
  future reminder; `dateparser` had resolved "now" to the current instant and it fired back out as
  proactive speech. `_IMMEDIATE_WHENS` set added.
- `volition/src/volition/events/intake.py` — **changed**: `/reminders` rejects empty-text reminders
  (`text=''` junk the cortex spammed) with `ok=False, reason=empty_text`.
- `volition/tests/test_reminders.py` — **added**: `test_immediate_when_rejected`. 26 green.
- `curator/src/curator/tool_catalog.py` — **changed**: `set_reminder` description now forbids
  immediate commands ("turn on the lights" → actuate via control_device/media, never a reminder;
  `when` is never "now"). (reminder-misroute commit)

### iris-docs
- `issues/ISSUE-013-conversation-tightening-10day-audit.md` — **added**: the audit + ranked open items.
- `changelog/2026-09-25-…md` — this entry.

## Deploy / live state
- **STAGED (pending restarts, Matt sudo):** `sudo systemctl restart iris-volition iris-curator`.
  Until then the briefing still reads its prompt and reminders can still mis-fire.
- Nothing else went live this session.

## Audit headline (see ISSUE-013)
- Latency p50 4.6s / p90 6.9s (cortex-logical ~70% of it) — the top OPEN item; recommended next fix is
  the reflex-confirmation short-circuit so a reflex-actuated "Done." doesn't wait the full decision.
- ASR noise: 13% degenerate turns (TV bleed / partial captures); 5 triggered deep punts.
- Confirmed holding: got_away 7→1→0 across the Sept-10/11 fix; silent_clause 0 over 10 days.

## Follow-ups / owed
- Restarts above.
- Origin pushes still owed (automation can't push): fusion `feat/punt-escalation`, world-ai
  `fix/actuation-settle-verify`, iris-docs. `/opt/repos/_push-staging/` has bundles + push-all.sh.
- Open ISSUE-013 items: latency short-circuit (A), ASR min-content guard (B), deep-punt input gate (C),
  backyard-light entity mapping (D).
