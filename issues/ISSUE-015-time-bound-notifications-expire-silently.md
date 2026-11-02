# ISSUE-015: Time-bound notifications expire silently when no presence is detected in their window

- Status: open
- Severity: low
- Area: volition / events (delivery arbiter — occupancy gating + expiry)
- Discovered: 2026-11-02 (reminders/timers pressure test)
- Evidence: `pending_events` lifetime counts delivered=320 (all attempts=1), expired=31 (all **attempts=0**), cancelled=5. Expired breakdown: 20 calendar, 6 weather_alert, 5 deliberation — zero of them the user-reminder kind. Sample: a calendar ping created 12:18, fire_at 12:43, expiry 12:50 (~7-min window) that never got an attempt. Related: the morning briefing "fired + lapsed" (STATE/ audit).

## Summary
The delivery arbiter is **occupancy-gated** ("never talk to an empty house — no occupied speaker
room → everything holds") and the queue expires held events at their `expiry`. When a **time-bound**
notification (kind=calendar / weather_alert / deliberation, which carry a short expiry) fires during a
window in which **no presence is detected in a tracked speaker room**, it holds, is never attempted
(`attempts=0`), and lapses at expiry — **with no trace to the user**. 31 events have vanished this way.

This is working as designed, and arguably correct (a meeting reminder is useless after the meeting
starts). Two real downsides:
1. **Silence** — the user never learns a ping was dropped. A dropped "leave for your 9:00" is invisible.
2. **Leans entirely on presence detection being right** — the dog-problem foundation. A missed presence
   read (user home but not detected in a *speaker* room) silently eats the notification.

## NOT affected: user-set reminders
`reminders.build_reminder` sets `expiry=None, holdable=False` ("a reminder is the one thing we must
not silently drop"), so "remind me to …" never expires — it waits and fires the moment presence is
detected. Verified live 2026-11-02 (evt-77c67620: set via NL "in 1 minute" → delivered multicast to
vg-01+vg-02 on the first attempt). The reminder capability is reliable; this issue is only the
time-bound notification path.

## Directions (not yet decided — deferred, low priority)
- **Leave a trace** when a time-bound notification expires undelivered: surface a terse "you missed:
  <brief>" on the user's next turn (or at least a log/metric + a NOC panel), so drops aren't invisible.
- **Widen or make configurable** the fire→expiry window for some kinds (a weather alert may be worth a
  longer tail than a 7-min calendar ping).
- **Presence robustness** — the deeper fix is better occupied-speaker-room detection (dog-problem
  foundation, out of scope here).
- Consider a fail-soft fallback channel (phone push) for genuinely urgent, non-holdable items — noted
  as "fail-quiet phone" elsewhere; not pursued.

## Notes
Filed to capture the finding and move on (Matt, 2026-11-02). Delivery itself is healthy (320/320 first
attempt); this is about what happens to the minority of time-bound items that can't reach a present
user in time.
