# 2026-09-10 — Multi-room synced speaker playback (speaker grouping)

## Executive summary
Built end-to-end support for playing music across multiple Sonos speakers **in sync**,
fixing Matt's long-standing "all the speakers play, but not in sync" problem. Root cause:
each speaker was started as an independent stream. Fix: group the speakers (Sonos hardware
sync) and play once on the group leader. World-ai gained the grouped-playback capability
(LIVE — world-ai restarted); the fabric voice path (`select_content` tool → tool-registry →
world-ai) is wired and deployed but **pending a fabric restart** (iris-curator,
iris-cortex-logical, iris-tool-registry). Verified the join/unjoin mechanics live on idle
speakers before building.

## Key finding (verified live)
`media_player.join` on the Music Assistant player entities (`*_2`/`family_room`/`main_bedroom`)
groups the underlying **Sonos hardware** — the bare Sonos twins (`media_player.dining_room`,
`living_room`, …) reflected the same `group_members`. So a single MA `play_media` on the leader
plays the whole group in true hardware sync. `unjoin` cleanly dissolves it. `SUPPORT_GROUPING`
feature-bit was unreliable in states; presence of the `group_members` attribute is the real signal.

## Changes by repo

### world-ai — branch `fix/actuation-settle-verify` (commit 9ab4bfa)
- `app/services/spotify_service.py` — **changed**: added `play_on_speakers(query, content_type,
  ha_entity_ids: list)`. For >1 target: best-effort `unjoin` each (dissolve stale groups) →
  `media_player.join` members onto the first as leader → `GROUP_SETTLE_S=1.5s` settle →
  `music_assistant.play_media` once on the leader. Grouping failure degrades to leader-only;
  slow MA resolve still returns optimistic success. `play_on_speaker` (singular) kept as a
  back-compat wrapper. Added `_join`/`_unjoin`/`_headers`/`_ha_service` helpers + order-preserving dedup.
- `app/routes/spotify.py` — **changed**: `PlayRequest` now accepts `entity_ids: list[str]` as
  well as `entity_id` (back-compat), coalesced leader-first via a model validator; `PlayResponse`
  gained `group`.

### iris-fusion-fabric — branch `feat/punt-escalation` (commit 48edfb0)
- `curator/src/curator/tool_catalog.py` — **changed**: `select_content` advertises an optional
  `entity_ids` array param alongside `entity_id` (now `required=False`); description tells the
  cortex one room → entity_id, multiple → entity_ids (grouped, synced).
- `cortex-logical/src/cortex_logical/tool_schema.py` — **changed**: array params now emit
  `items:{type:string}` in the Ollama tools schema (qwen needs it or emits a bare array).
- `tool-registry/src/tool_registry/dispatch.py` — **changed**: pass `entity_id` + `entity_ids`
  (both `.get`, optional) through to `select_content`.
- `tool-registry/src/tool_registry/backends/ha.py` — **changed**: `select_content` coalesces
  both forms into an ordered target list, keeps only `media_player.*` entities, posts
  `entity_ids` to `/spotify/play`; returns `group`.

## Deploy / live state
- **LIVE**: world-ai grouped-playback capability (service restarted). Testable now:
  `curl -XPOST localhost:8001/spotify/play -d '{"query":"Frank Sinatra","content_type":"artist",
  "entity_ids":["media_player.dining_room","media_player.living_room"]}'`.
- **STAGED (pending fabric restart)**: the voice path. Needs `sudo systemctl restart
  iris-curator iris-cortex-logical iris-tool-registry`. Until then, voice "play X in the dining
  room and living room" won't reach the new grouping.

## Correction (same day) — order bug: play-THEN-join
First cut did join-then-play; symptom "music only plays out of the first speaker."
Root cause found live: `music_assistant.play_media` returns **HTTP 500 when its target is
already in a native Sonos group**, and even when it doesn't 500, only the leader plays.
Working order is **play-then-join**: play on the leader alone, then join the members onto the
LIVE leader — each inherits the leader's playing stream in sync (verified: member went
idle → playing the leader's exact track, both grouped). Rewrote `play_on_speakers` (world-ai
commit on `fix/actuation-settle-verify`): dissolve stale groups → play leader (self-heals a
500 from a leftover group by unjoin+retry once, `PLAY_TIMEOUT_S=10s`) → settle → join members.
**world-ai needs another restart** to pick this up.

## Correction 2 (same day) — group on BARE entities + poll-for-playing
After the play-then-join fix, live testing surfaced two more: (a) `media_player.unjoin` on the
**MA twin** returns 200 but does NOT dissolve the Sonos group (a wedged group stayed grouped →
next `play_media` 500'd on the still-grouped target); grouping must target the **bare Sonos
entities** (`media_player.dining_room`), MA twin only for the play. (b) A slow MA resolve made
the code join a not-yet-live leader and the join timed out (bumped `HA_TIMEOUT_S` 4→8s), leaving
the group unformed ("plays only on the first speaker") — added `_wait_playing` to poll the leader
to `state==playing` before joining. Added `_bare_entity`/`MA_TO_SONOS_ENTITY`. Verified live via
combo test AND the real `play_on_speakers` module call: returns success with the full group, both
rooms play the same track in sync. world-ai commit 93ca541. **world-ai restart still owed.**

**Observed, unrelated:** MA/Spotify **playback dropped out ~15s after starting** on several runs
(plays + syncs correctly, then goes idle on its own) — an MA/Spotify resolve-layer flakiness, not
grouping. Possibly the personal Spotify dev key / artist-radio queue. Worth watching.

## Follow-ups / owed
- Origin pushes owed (automation can't push): world-ai `fix/actuation-settle-verify`,
  iris-fusion-fabric `feat/punt-escalation`.
- Still-owed unrelated restart: iris-deep-mode (15s deferred-thought delay).
- Possible future: a "play everywhere" shortcut (all Sonos entities); volume normalization
  across a group; the reflex fastpath still can't do multi-room (deliberate-only, fine for now).
