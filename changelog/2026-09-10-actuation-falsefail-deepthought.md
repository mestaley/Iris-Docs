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

---
## Fix: "got away from me" on responsive actuations (2026-09-10)
turn-9b7f6fab ("basement lights off"): the reflex turned them off (success=True) but the HA call
took ~1.4s and the result arrived AFTER fusion decided (~3s), so actuated_ok=False -> the empty
front door fell to the "Sorry - that one got away from me. Say it again?" fallback though the lights
DID go off. Fix: a reflex device/media actuation whose result is still PENDING counts as actuated
(optimistic; the reflex reliably dispatches, the async supervisor backstops - it logged "reflex
CORRECT"). A reflex result that ARRIVED failed is NOT pending, so the genuine-failure correction
still voices. fusion `feat/punt-escalation`, +1 test, 56 green. **Needs iris-fusion-executive restart.**

## Consolidated restarts owed (Matt sudo)
- **iris-fusion-executive** — actuation false-fail suppression + "the that device" grammar +
  media-play wording + the no-wake-window `?`-fallback arming.
- **iris-deep-mode** — the 15s deferred delay (verify_failed-skip already restarted earlier).
- (audio-ai speech-output + intent-consumer already deployed+restarted by automation. world-ai
  config cleanup needs no restart.)

---
## Spotify RESOLVED (2026-09-10) — MA re-auth worked; fixed the slow-resolve false-fail
Matt re-authed Music Assistant's Spotify provider (with his OWN Spotify developer API key — MA's
new requirement; cross-VLAN Connect pairing was the pain). **Confirmed playing** live:
media_player.dining_room = "Frank Sinatra - Dancing In The Dark". Remaining bug: MA resolves a
Spotify artist (search + queue) in >5s — longer than the fabric's ~5s tool budget — but plays ASYNC,
so the fabric reported a false "couldn't get that playing" WHILE it played. Fixed world-ai
`spotify_service.play_on_speaker`: 4s httpx timeout, treat a timeout as OPTIMISTIC SUCCESS (accepted;
playback follows); a fast non-200 (auth/bad entity) still a real failure. Committed on world-ai
`fix/actuation-settle-verify`. **Needs world-ai restart.**

Doc note for future-me: MA's Spotify provider now needs YOUR OWN Spotify developer app key + a
one-time Spotify Connect pairing (zeroconf/mDNS, L2-only — won't cross VLANs; pair once on the same
network, credentials cache). Not a world-ai token (that path was nuked 2026-05-29).

## Consolidated restarts owed (Matt sudo)
- **iris-fusion-executive** (core-ai) — actuation false-fail suppression + "the that device" grammar
  + media-play wording + no-wake-window `?`-fallback arming.
- **iris-deep-mode** (core-ai) — 15s deferred delay.
- **world-ai service** — Spotify slow-resolve optimistic-success + (already-applied) dead-config removal.
- (audio-ai speech-output + intent-consumer already restarted by automation.)
