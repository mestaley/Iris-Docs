# ISSUE-010: Deep-mode barge-in leaks internals + is cold-start slow

- Status: **RESOLVED (built) 2026-08-27** — grooming + warm-keep implemented on `feat/deep-mode`
  (commit on wt-deep-mode); STAGED to core-ai, needs `sudo systemctl restart iris-deep-mode`. See
  "Resolution" at the bottom. (Originally found 2026-08-26 reviewing turn-3ca38ed3; the two
  spurious-punt bugs were fixed then.)
- Severity: medium (voice quality when the deep tier does speak; not a correctness bug)
- Area: cognition / iris-deep-mode (+ ollama warmth)

## Origin
Diagnosing the "interesting" voice output on **turn-3ca38ed3** (2026-08-26 12:39, "turn the
kitchen lights to 20%"). Root cause of the misfire was two-fold and is now **fixed**:
1. `cortex-logical` returned an empty pass1 on the reflex-satisfied turn → fusion voiced the canned
   filler and self_uncertain-**punted**. Fixed: cortex-logical now confirms ("Done.") on a
   fastpath-satisfied empty pass1 (commit on `feat/web-tasks-tools`); fusion suppresses the
   proactive punt when a device actuation succeeded this turn (commit on `feat/punt-escalation`).

That kills the *spurious* escalation. This issue tracks the **two residual deep-mode problems** that
were exposed but are real whenever the deep tier legitimately barges in:

## The residual problems
2. **Ungroomed output — leaks internals into speech.** The deep answer that barged in voiced the raw
   HA entity_id and chain-of-thought: *"Right then. Kitchen lights — that's `light.kitchen_lights`,
   confirmed. Twenty percent brightness. That's a dim setting, not full on, not off. Got it. I'll
   adjust… Done. The kitchen lights are now at twenty percent."* Iris should never speak entity_ids
   or think-aloud. `iris-deep-mode` must sanitize its `say` before publishing the SpeechUtterance —
   strip entity_ids, drop meta/planning prose, and keep it in Iris's register (terse, dry). The
   30B instruct model is verbose by default; needs an output contract / post-filter like the cortex
   wrappers have.
3. **Cold-start latency.** The barge-in landed ~10s after the front-door reply, worsened by an
   **ollama qwen3-deep cold start of 8.61s** (the llama runner had to load). If the deep tier stays
   a live path, keep it warm (`KEEP_ALIVE=-1` on the :11435 instance, or a periodic ping) so a real
   escalation barges in promptly instead of ~10s late.

## Recommended work
- **deep-mode output grooming** (primary): an output filter / prompt contract on `deep_client` so
  the `say` is speech-ready — no entity_ids, no chain-of-thought, Iris's voice. Mirror the
  cortex-logical pass-2 wrapper discipline.
- **Warm-keep qwen3-deep**: `KEEP_ALIVE=-1` (or heartbeat) on GPU1's ollama for the deep model.
- Revisit whether the deep tier belongs on the *voice* path at all vs. the web/research surface
  (STATE.md already flags voice as utilitarian — almost nothing needs a 30B deliberator aloud).

## Evidence
turn-3ca38ed3, deep-mode log 12:39:14→12:39:24 ("deliberating (qwen3-deep)", runner load 8.61s,
"barged in deep answer"); speech-output synth'd it cleanly (the audio wasn't the problem, the text
was). Full trace in the 2026-08-26 changelog.

## Resolution (2026-08-27) — deep-mode `feat/deep-mode`
Both parts built + tested (deep-mode suite 10 green), STAGED to core-ai:
- **Grooming — prompt contract** (`service.py build_prompts`): system now says answer OUT LOUD in
  Iris's voice, no thinking-aloud / step narration ("let me…", "I'll…", "Right then", "Got it"),
  and never speak technical identifiers/entity_ids (with the `light.kitchen_lights` example).
- **Grooming — facts filter**: `_INTERNAL_AMBIENT_KEYS` keeps `entity_mappings` (the id leak
  source), `current_hour_24`, and `recent_conversation` OUT of the facts dump, so the model can't
  parrot them.
- **Grooming — post-filter** (`groom_for_speech`): humanizes any HA entity_id that still slips
  through (`light.kitchen_lights` → "kitchen lights") before the utterance is published. Applied in
  `_deliberate_and_barge`.
- **Warm-keep** (`config.py` + `deep_client.py` + `service.py main`): pass `keep_alive` to ollama
  (`DEEP_MODE_KEEP_ALIVE_S`, default -1 = resident) and fire one tiny warm-up call at startup
  (`DEEP_MODE_WARM_ON_START`, only when enabled) so the first real punt doesn't pay the ~8.6s cold
  start.

**Owed:** `sudo systemctl restart iris-deep-mode`; origin push of `feat/deep-mode`.
**Still open (the standing question, NOT built):** whether the deep tier belongs on the *voice*
path at all vs. the web/research surface — deferred.
