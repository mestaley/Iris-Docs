# 2026-11-02 — Music actually plays: Music Assistant search→URI→play

## Executive summary
Diagnosed (with Matt) why "play jazz" never worked and fixed it end to end. Root causes, in order
uncovered: (1) `select_content`/world-ai passed the **raw query as `media_id`** to
`music_assistant.play_media`, but MA needs a resolved **URI** — a plain name silently no-ops; (2) search
needs `library_only=false` to hit the Spotify catalog (default is library-only → empty); (3) Iris was
aiming at the **Sonos-integration** entities, not the **MA-owned** players. After Matt rebuilt Sonos+MA in
HA, the MA players now own the clean IDs (`media_player.basement` etc.) and the stale bare→MA map is wrong.
Fix: world-ai now does `music_assistant.search` (library_only=false, return_response) → pick best URI
(requested type, else playlist>artist>album>track) → `play_media` URI with `enqueue=play`; config_entry_id
discovered dynamically (changes per MA restart); stale entity map dropped. `select_content` routes through
world-ai `/spotify/play` again. **Verified live** (jazz → "Great Pumpkin Waltz", Vince Guaraldi, on
`media_player.basement`). Code committed on branches; **pending deploy** + a mappings fix.

## Changes by repo
### world-ai — branch `feat/ma-music-search` (commit e5c7c70, off deployed `fix/actuation-settle-verify`)
- `app/services/spotify_service.py` — **rewritten**: `play_on_speaker` now searches for a URI then plays it
  (two-step), discovers the MA config_entry_id, picks playlist-first for genres, `enqueue=play`, drops the
  stale `SONOS_TO_MA_ENTITY` map (entities pass through — clean IDs post-rebuild).
- `tests/test_spotify_service.py` — **added**: 5 tests (genre→playlist pick, artist pick, no-match clean
  fail, play-failure surfaced, entity passthrough).

### iris-fusion-fabric (via wt-integration) — branch `feat/music-via-worldai` (commit 0a80739)
- `tool-registry/src/tool_registry/backends/ha.py` — **changed**: `select_content` routes to world-ai
  `/spotify/play` (one call per target for multi-room) instead of calling `play_media` directly; dropped
  the raw-name + media_play-nudge approach.
- `tool-registry/tests/test_music_assistant.py` — **changed**: now tests the `/spotify/play` path. 5 green.

## Deploy / live state
- **Built + live-validated against HA, NOT deployed.** Both branches unpushed.
- MA integration is `loaded`; entities (MA-owned, clean): `media_player.basement` (Basement),
  `dining_room` (Dining Room), `master_bedroom` (Master Bedroom), `family_room` (**Living Room**). No
  kitchen audio player. Stale `media_player.spotify` + `media_player.living_room` no longer exist.

## Follow-ups / owed — deploy (Matt)
1. **world-ai**: merge/deploy `feat/ma-music-search` to the world-ai VM (`/opt/world-ai`, currently on
   `fix/actuation-settle-verify` @ 93ca541) + `sudo systemctl restart world-ai`. No new deps.
   (NB: world-ai branch sprawl — the lock-domain block is on a *different* branch (`fix/block-lock-domain`,
   pushed, still NOT deployed to the running world-ai). Worth consolidating.)
2. **tool-registry**: merge `feat/music-via-worldai` → deployed integration branch; bundle to core-ai;
   `sudo systemctl restart iris-tool-registry`.
3. **Mappings** (core-ai Qdrant — SSH was down at session end): `media_player.living_room` → `media_player.family_room`
   and delete the `media_player.spotify` entry. Script:
   ```python
   import urllib.request, json
   B="http://127.0.0.1:6333/collections/iris_mappings"
   def post(p,b): return json.load(urllib.request.urlopen(urllib.request.Request(B+p,data=json.dumps(b).encode(),headers={"Content-Type":"application/json"},method="POST")))
   pts=post("/points/scroll",{"limit":300,"with_payload":True,"with_vector":False})["result"]["points"]
   liv=[p["id"] for p in pts if p["payload"].get("entity_id")=="media_player.living_room"]
   spot=[p["id"] for p in pts if p["payload"].get("entity_id")=="media_player.spotify"]
   if liv: post("/points/payload?wait=true",{"payload":{"entity_id":"media_player.family_room"},"points":liv})
   if spot: post("/points/delete?wait=true",{"points":spot})
   print("living->family:",liv,"deleted spotify:",spot)
   ```
- After all three: "play some jazz in the basement" / "play Pearl Jam in the living room" should work.
- Known gap: multi-room plays each room independently (no MA sync-group); no kitchen player.
