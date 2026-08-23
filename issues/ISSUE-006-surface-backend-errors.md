# ISSUE-006: Iris should say a backend ERRORED, not vaguely deflect
- Status: open (captured 2026-08-23, Matt: "it'd be nice if Iris actually told me there was a 503 error")
- Severity: medium
- Area: cognition / honesty
- Evidence: live turn ~15:05 2026-08-23 — Meridian weather 502'd; she said "Unavailable. I'll check." / "I can't check the current conditions since…" (vague, not "the weather service is erroring")

## Symptom
When a prefetch tool fails (Meridian 502/503, HA down, etc.), Iris deflects vaguely
("unavailable, I'll check") instead of telling Matt there was an actual error. He wants the
honest signal: "the weather service is down / I got a 503 reaching it."

## Root cause
Curator only puts SUCCESSFUL tool results into ambient:
`if tr.success and tr.result is not None: ambient[tr.tool_name] = tr.result`
(curator/service.py ~268). A FAILED prefetch is silently dropped — the cortex sees *absence*
of data, indistinguishable from "no data was requested." The creative prompt then (correctly,
for the absence case) says "if AMBIENT DATA shows no weather, say you'll check rather than
guess." So she can't report the error because she was never told one happened.

## Fix direction (deliberate — do with an eval case, not a hot-patch)
1. **Curator:** on a failed prefetch, add a structured error entry the cortex can see — e.g.
   `ambient["_service_errors"] = [{tool, code, message}]` (or a dedicated ctx field). Include
   the HTTP status when available (502/503) so she can name it if useful.
2. **Cortex prompts (creative + logical):** add a rule — "If AMBIENT DATA shows a service
   error for what was asked, say so honestly and briefly ('the weather service is erroring
   right now, I couldn't reach it') — distinct from having nothing to say. Don't recite a raw
   stack trace; a plain 'X is down / erroring' is right." Keep it short and in-character.
3. Distinguish transient (5xx/timeout → "erroring right now, try again") from not-configured
   (404/unsupported → "I can't do that") if cheap; otherwise treat all as "couldn't reach it."

## Notes
- Same honesty family as [[ISSUE-005]] (don't fabricate; here: don't hide a failure behind a
  vague deflection). The actuation path already voices failures (fusion correction); this
  extends honest failure-reporting to READ/prefetch failures.
- Contract check: adding an ambient error entry is additive (ambient is a dict); a dedicated
  CuratedContext field would touch the (frozen) contract — prefer the ambient-dict route.
