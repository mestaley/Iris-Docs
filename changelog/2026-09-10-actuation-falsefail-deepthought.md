# 2026-09-10 — Failed-actuation cluster: false "can't reach it" + spurious deep-thought

## Executive summary
From the Frank Sinatra / lights turns Matt flagged: a cluster around failed/false-failed actuations.
Fixed three things. (1) deep-mode now only deliberates on proactive salience (affect/complexity),
never `verify_failed` — a failed command no longer spawns a nonsensical deferred "deep thought story".
(2) fusion no longer voices "couldn't reach it" when a device actuation actually succeeded this turn
(the reflex+logical double-fire made duplicate no-op calls report failure though the lights dimmed).
(3) grammar: "the that device" → "the device".

## Changes by repo
- **iris-fusion-fabric `feat/deep-mode`** — `deep-mode/.../service.py`: `_DELIBERATE_REASONS =
  {affect, complexity}`; on_punt skips other reasons (esp. verify_failed). +2 tests. RESTARTED (live).
- **iris-fusion-fabric `feat/punt-escalation`** — `fusion-executive/.../service.py`: gate the
  actuation-failure correction on `not actuated_ok` (suppress the false "can't reach it" when
  something did actuate); fallback name "that device"→"device". +1 test, 54 green. **Needs
  iris-fusion-executive restart.**

## Known deeper root cause (banked)
The reflex + cortex-logical **double-fire** the same actuation (dedup gap): the reflex succeeds, then
logical's duplicate control_device calls no-op to success=False in 0ms. The fusion fix suppresses the
false failure symptom; the double-fire itself (why dedup misses, e.g. group vs specific entities)
deserves its own fix — see [[iris-address-asr-actuation-verify-bugs]].

## Delay / deep-thought UX (Matt's feedback, not yet built)
Matt: the deferred deep-thought "takes too long to be useful, I don't understand the delay." Real fix
= offer-before-deliver (recall-queue Phase 3, ISSUE-012): Iris OFFERS ("deeper read — want it?") and
delivers on confirm, instead of a mysterious late monologue. Now feasible (Phase 1 arms the no-wake
window for offers). Sequenced next.

---
## Spotify "502" diagnosis + cleanups (2026-09-10)
turn-1924965e Frank Sinatra failure traced: world-ai `/spotify/play` → HA
`music_assistant/play_media` → **HA returns 500** → world-ai returns 502 (its own design)
→ fabric said "can't reach it". Reproduced live (HA 500). NOT a world-ai Spotify token — that
path was **nuked for Music Assistant 2026-05-29** (the `spotify_*` config vars were dead/misleading).
The real failure is in **HA/Music Assistant's play_media** (Matt to check: MA Spotify provider
re-auth, and there's a pending MA update). Matt confirmed the MA refactor.
- **world-ai** `app/config.py` (branch fix/actuation-settle-verify): removed dead `spotify_client_id/
  secret/refresh_token`. No restart needed (unused).
- **fusion** `feat/punt-escalation`: media-play failures say "couldn't get that playing", not "reach
  the device" (a control_media/select_content fail isn't a device being unreachable). +1 test.

## Consolidated restarts owed (Matt sudo)
- **iris-fusion-executive** — actuation false-fail suppression + "the that device" grammar +
  media-play wording + the no-wake-window `?`-fallback arming.
- **iris-deep-mode** — the 15s deferred delay (verify_failed-skip already restarted earlier).
- (audio-ai speech-output + intent-consumer already deployed+restarted by automation. world-ai
  config cleanup needs no restart.)
