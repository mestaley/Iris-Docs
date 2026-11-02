# ADR-006: Scenes are Iris-owned (YAML + a deterministic tool), not Home Assistant scenes

- Status: accepted (2026-11-02) — implementation pending (thin slice; see "Implementation sketch")
- Date: 2026-11-02
- Relates to: [ADR-005](ADR-005-name-based-actuation-contract.md) (name-based actuation — scenes build directly on it),
  [`iris-proactive-lighting-scope`], Phase-4 Volition. Reuses the `control_device` dispatch + `iris_mappings`
  resolution + settle-verify already in place.

## Context
We want "activity" behaviors — "movie time" → living-room lamps to 10%, kitchen off, TV backlight on;
"good night" → everything off but the hallway; etc. The question is **where the scene abstraction
lives**: a Home Assistant `scene.*` entity that Iris merely fires, or a definition Iris owns and
expands herself.

Two framing facts:
1. Matt does not want to author/maintain scenes in HA.
2. Volition (proactive Iris) is not close to being useful, so a scene must be valuable **user-invoked**
   first ("Iris, movie time"), independent of any proactive trigger.

## Decision
**A scene is an Iris-owned, named compound of [ADR-005](ADR-005-name-based-actuation-contract.md)
actions, defined in version-controlled YAML and invoked through one deterministic tool
`activate_scene(name)`.** Not an HA scene.

Device *actuation* still flows through the existing path — `control_device → world-ai `/ha/service`` →
HA, with settle-verify. We are **not** reinventing device control; we own only the **grouping/semantics
layer**, and that layer belongs in Iris, not in HA.

### Why Iris-owned, not HA scenes
- **Composition / adjustment.** "movie time, but leave the hallway on" or "a bit brighter" is only
  possible if Iris knows the scene's constituent devices. An HA scene is opaque to her.
- **Explanation.** Iris can narrate *"dimmed the lamps, killed the kitchen, left the TV backlight"* —
  the grounded, narratable behavior the thesis-grounding work targets — instead of a blind "ok".
- **Thesis line.** PRINCIPLES: Iris is *not* a Home Assistant voice skin; household semantics live in
  her substrate, not HA config. A scene-as-HA-entity is exactly the skin we said she isn't.
- **Ownership.** Matt doesn't want HA scene maintenance; a small YAML file he edits is the opposite of
  an HA UI/YAML scene registry he has to keep in sync.
- HA keeps doing the one thing it's best at — moving the devices — behind `control_device`.

### Shape
```yaml
# scenes.yaml  — Iris-owned, version-controlled, hand-tuned (PRINCIPLES: explicit config, not learned)
movie_time:
  description: "watching something in the living room"      # also the semantic-match text
  actions:
    - { target: "living room lamps", action: "set", brightness_pct: 10 }
    - { target: "kitchen lights",    action: "off" }
    - { target: "tv backlight",      action: "on" }
normal:                                                      # the return-to-baseline scene
  description: "back to normal lighting"
  actions:
    - { target: "living room lamps", action: "on" }
    - { target: "kitchen lights",    action: "on" }
```

- **`activate_scene(name)`** is the only model-facing surface. The model picks a *name*; it does **not**
  emit N precise device calls per turn — more reliable (it can't fumble the compound shape, cf.
  ADR-005's whole motivation) and lower-latency.
- A deterministic **expander** turns the scene into the existing `control_device` actions. `target`
  stays a **name** (ADR-005) resolved to `entity_id` through `iris_mappings` — so scenes inherit all
  current grounding, ambiguity-handling, and settle-verify for free.
- **Loose-phrase matching for free.** Seed each scene's name+description into the `iris_tools` Qdrant
  collection, so "it's time for a movie" / "movie night" / "let's watch something" all retrieve
  `activate_scene(movie_time)` without enumerating phrasings. YAML defines *what*; Qdrant (existing
  tool-retrieval) handles *how it's said*.
- **Return-to-baseline** is just a `normal`/`reset` scene — no need to replicate HA's snapshot/restore.

### Decouple the mechanism from the trigger (key design point)
The **mechanism** (scenes.yaml + `activate_scene` + expander) is useful immediately, user-invoked, and
has nothing to do with Volition — ship it now. **Proactive triggering** ("living room dim + someone
sat down at 8pm → offer the movie scene") is a Volition concern and lands whenever Volition matures;
it just calls the same `activate_scene`, no rework. We get ~90% of the value (named scenes on command)
without touching the hard proactive problem.

## Consequences
- **Positive:** composable/explainable scenes; zero HA scene maintenance; reuses control_device +
  mappings + settle-verify (small new surface); fits "hand-tuned static config, not continuous
  fine-tuning"; forward-compatible with Volition.
- **Cost / risk:** Iris orchestrates multiple actuations per scene (already supported — compound
  actuation exists). If a constituent device fails, the scene is partial — the expander must report
  per-action success (reuse settle-verify) and Iris should say what didn't take (consistent with the
  actuation-verification barge-in). If Iris/core-ai is down there is no scene (an HA scene would still
  fire from a wall button — acceptable; voice is the entry point anyway).
- **Not precluded:** Iris can still fire a pre-existing HA scene via `control_device(domain=scene)` if
  one ever exists — the two models aren't mutually exclusive — but the *primary* model is Iris-owned.

## Alternatives considered
- **HA scenes (Iris fires `scene.turn_on`).** Rejected: opaque to Iris (no composition/explanation),
  requires HA-side maintenance Matt doesn't want, and is the "HA skin" the thesis rejects. One call and
  HA's state capture/restore are the only real draws, and neither outweighs the above.
- **Scenes in the Qdrant substrate (learned/adaptive).** Rejected for the *definitions*: scenes are a
  small hand-curated set Matt wants explicit control over (PRINCIPLES: not an always-improving system).
  Qdrant is still used for the *trigger-phrase matching*, which is the right split.
- **LLM emits the compound every turn (no scene tool).** Rejected: re-exposes exactly the shape-fumbling
  ADR-005 moved away from, and is slower; the name-only tool is the reliable surface.

## Implementation sketch (thin slice, when picked up)
1. `scenes.yaml` + a loader (fabric-side; lives with the fabric config or iris-docs-referenced).
2. `activate_scene` ToolDefinition in the catalog (`{name: str}`), with `side_effects=True`.
3. Expander in tool-registry dispatch: load scene → emit `control_device` actions (names, ADR-005) →
   dispatch compound → aggregate per-action settle-verify → return a result Iris can narrate.
4. Seed scene name+description into `iris_tools` (Qdrant) so curator's tool-retrieval surfaces it.
5. Verify live: "Iris, movie time" fires the compound, single grounded ack naming what changed.
