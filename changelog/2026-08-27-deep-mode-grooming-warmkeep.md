# 2026-08-27 — Deep-mode: speech grooming + warm-keep (ISSUE-010)

## Executive summary
Closed ISSUE-010 (from the turn-3ca38ed3 review): the deep-tier barge-in leaked raw entity_ids and
chain-of-thought into TTS, and cold-started ~8.6s (landing ~10s late). Built both fixes on
`feat/deep-mode`: a spoken-output prompt contract + internal-key filter + an entity-id post-groom,
and warm-keep (ollama keep_alive + a startup warm-up). Tested (deep-mode suite 10 green), STAGED to
core-ai — needs `sudo systemctl restart iris-deep-mode`.

## Changes by repo
### iris-fusion-fabric — branch `feat/deep-mode` (wt-deep-mode) — commit (deep-mode)
- `deep-mode/src/deep_mode/service.py` — `build_prompts` rewritten: answer OUT LOUD in character, no
  thinking-aloud/step narration, never speak identifiers; `_INTERNAL_AMBIENT_KEYS` filters
  entity_mappings/current_hour_24/recent_conversation out of the facts dump; new `groom_for_speech`
  humanizes leaked HA entity_ids (light.kitchen_lights → "kitchen lights"), applied before the
  barge-in publish; warm-up thread in `main` when enabled.
- `deep-mode/src/deep_mode/deep_client.py` — `keep_alive` on every request + a `warm()` method.
- `deep-mode/src/deep_mode/config.py` — `keep_alive_s` (default -1), `warm_on_start` (default True).
- `deep-mode/tests/test_deep_mode.py` — +4 tests (groom, multi-domain/no-op, prompt filter+contract,
  end-to-end groomed barge-in). 10 green.

## Deploy / live state
- STAGED to core-ai (files synced). Needs `sudo systemctl restart iris-deep-mode`. DEEP_MODE_ENABLED
  is already live, so on restart the warm-keep engages (model resident, first punt no longer cold).

## Follow-ups / owed
- **Matt sudo:** restart iris-deep-mode.
- **Origin push:** `feat/deep-mode`.
- **Still open (deferred):** whether the deep tier belongs on the voice path at all vs. web/research.

## Validation (live 2026-08-28)
- **Warm-keep: confirmed.** Startup log `warming deep model qwen3-deep (keep_alive=-1s)… warm-up OK`
  (cold-start paid at boot 14:42:28→37); ollama `/api/ps` shows qwen3-deep resident (expires 2318 =
  forever). A live complexity punt (turn-web-276be787) deliberated → **barged in ~1s** (14:45:20→21),
  vs the ~8.6s cold-start that opened the issue.
- **Grooming: works.** The deep answer to a hard advice question came out as a clean spoken reply, no
  entity_ids, no device-planning monologue. It opened "Right then," — which the first prompt banned;
  that ban was misjudged (dry-British Iris persona; the real leak was reasoning/action narration, not
  a conversational opener). Prompt reworded to forbid play-by-play-of-thinking while allowing a
  natural opener (commit on `feat/deep-mode`). STAGED — takes effect on the next iris-deep-mode restart.
