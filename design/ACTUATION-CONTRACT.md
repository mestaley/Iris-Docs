# Name-based actuation contract — design (fleshes out ADR-005)

- ADR: [`decisions/ADR-005-name-based-actuation-contract.md`](../decisions/ADR-005-name-based-actuation-contract.md) (proposed)
- Status: design draft, 2026-09-30 — buildable, incremental, reversible
- Reacts to: `changelog/2026-09-29-repair-control-device-punctuation.md` (the four-commit coercion stopgap + live verification)

## 0. Current state (live-verified 2026-09-30)
- `control_device` (also `control_media`, `select_content`) advertise `entity_id` (+ `domain`,
  `service`) as **required** params. `qwen2.5:14b` under Ollama native tools ignores this: it emits a
  natural-language `{device/command/action, ...}` shape and **almost never produces an `entity_id`**.
- **Where the mappings live:** `curator.mappings.retrieve_entity_mappings()` semantic-searches Qdrant
  `iris_mappings` (collection: 56 pts today) on the user's text each turn, min_score 0.5, top-K 10,
  filtered `user_id=matt`, room-locality sorted. Result → `ctx.ambient["entity_mappings"]` as
  `[{phrase, entity_id, domain, room, ...}]`. **Only curator touches Qdrant; tool-registry has no
  mapping access.** cortex sees the mappings via `ctx.ambient`.
- **The stopgap** (`cortex_logical.service._coerce_control_device`, commits 3bd7654→e912596): after
  pass-1 parse, resolves `entity_id` from any device-name key against `ctx.ambient["entity_mappings"]`
  (`_norm_phrase`: lowercase, `[\W_]+`→space, singularize trailing `s`; `_resolve_entity_id`: exact
  normalized match then unambiguous substring, **deduped on entity_id** so alias-phrases→one device
  isn't "ambiguous"), and normalizes `service`. Verified live (turn-054f8cab: `success=True 153ms`).
  `fusion._repair_control_device` is a last-resort net. This works but is a normalizer chasing every
  loose shape the model invents.

## 1. The core change — and the insight that makes it small
There are **two** contracts, and only one changes:

- **Model-facing tool schema** (what curator's `tool_catalog` advertises → Ollama `tools`): **CHANGES.**
  Stop asking for `entity_id`; ask for a device **name** + a small **action** enum — what the model
  reliably produces.
- **Internal `ToolCall`** (cortex → fusion → tool-registry, `iris_fusion_contracts.tools.ToolCall`):
  **UNCHANGED.** cortex resolves `target`→`entity_id` and publishes the same
  `{entity_id, service, ...}` shape fusion/tool-registry already consume.

So the wire contract to fusion/tool-registry does not move. No `iris_fusion_contracts` **schema**
change is required (`ToolParameterSpec`/`ToolDefinition` already express string+enum params) — only
different tool *definitions*, a formalized cortex resolution step, and the prompt. This is materially
smaller than ADR-005's worst-case cost estimate; the ADR's "touches several repos" holds, but each
touch is small and the risky part (wire contract) is untouched.

### New model-facing shape (illustrative)
```
control_device(target: str,            # device in the user's words: "living room lamps"
                action: "on"|"off"|"set",
                brightness_pct?: int)   # required iff action="set"

control_media(target: str, action: "play"|"pause"|"resume"|"volume", volume_level?: number)

select_content(query: str, content_type: str,
               targets: [str])          # one or more room names; resolved+grouped
```
`target`/`targets` are natural-language names. No `entity_id`/`entity_ids`/`domain`/`service` in the
model-facing contract — all derived during resolution.

## 2. Where resolution lives (the real placement decision)
Resolution needs **the model's chosen `target`** (only known after pass-1) **plus the mappings**
(only curator fetches them, into `ctx.ambient`). Options:

- **A — Resolve in cortex-logical (RECOMMENDED for v1).** The `target` is right there in pass-1 output;
  `ctx.ambient["entity_mappings"]` is already present. This is the stopgap's resolver, *promoted from
  repair to sanctioned step* — minus the loose-key coercion, since the contract now matches the model.
  Reuses e912596's tested matching verbatim. Downstream (fusion/tool-registry) unchanged. Smallest,
  lowest-risk, reversible.
- **B — Resolve in tool-registry ("true server-side").** Cleanest cognition/execution separation, but
  tool-registry has no Qdrant/mappings today; it would need either its own Qdrant client (duplicates
  curator) or the mappings passed on every `ToolCall` (heavy). Defer; revisit if we later want
  execution-layer purity.

**Recommendation: A now.** It's the same code path that already works, made legitimate. Document B as
the future move. Resolution is fabric-internal either way — the point of the ADR (model names, fabric
resolves) is satisfied by A.

### Resolution algorithm (v1, reusing the shim)
1. If model emitted a valid `entity_id` (dual-accept, §6) → keep.
2. Else resolve `target` against `ctx.ambient["entity_mappings"]` via `_resolve_entity_id`
   (normalized exact, then unambiguous substring, deduped on distinct entity_id).
3. **Miss / >1 distinct entity → clarify, don't guess** (§3).
4. Fill `service` from `action` (`on`/`set`→`turn_on`, `off`→`turn_off`) + `brightness_pct`; `domain`
   is recoverable from the entity_id prefix (tool-registry already does this).
- **Enhancement (v1.1):** on a miss in `ctx.ambient`, a targeted `retrieve_entity_mappings(target)`
  Qdrant search (curator's function, reused) before clarifying — catches devices curator's top-K on
  the user text missed. Keep v1 to ctx-only for simplicity; add if miss-rate warrants.

## 3. Ambiguity → first-class clarification (an upgrade, not just parity)
Today an unresolvable target is an honest actuation failure ("couldn't reach the device"). Under the
new contract, 0 or >1 **distinct** entities should drive `needs_clarification` in cortex's pass-2
(the field already exists), voicing the candidate **names**: "the living room ceiling lights or the
lamps?" This turns a dead end into a normal conversational repair and structurally prevents actuating
the wrong room. Zero-candidate ("garage disco ball") → honest "I don't have a device called that."

## 4. The prompt — names, not slugs
- The `entity_mappings` block in `cortex_logical.prompt` exists only to (unsuccessfully) coach
  entity_id emission. Under name-based resolution the model never needs a slug. **Replace it with a
  device-NAME list** (the mapping `phrase`s, no entity_ids) so the model can align its `target` to a
  known canonical name — raising the exact-match rate and shrinking the prompt.
- `ACTUATION RULE` keeps the "one call per target" compound rule; the `REQUIRED FIELDS` line rewrites
  to "every control_device MUST include `target` and `action`."

## 5. Sibling tools
`control_media` and `select_content` share the disease (they also demand `entity_id`/`entity_ids`).
Bring them under the same name-based resolution in the same change: `target`(s) → resolved
media_player entity(ies). `select_content` multi-room grouping is unchanged downstream (still resolves
to `entity_ids` list internally; the existing group-then-play path in world-ai is untouched).

## 6. Migration (incremental, reversible — never break live)
1. **Resolver accepts BOTH shapes.** The current shim already does: valid `entity_id` kept, else
   resolve a name. It is the dual-accept bridge. Keep it accepting either during transition.
2. **Flip the advertised schema** (curator `tool_catalog`) to `target`/`action`. Update the prompt
   (name list). Cortex now resolves the model's `target`; if the model still emits a stray
   `entity_id`, the dual-accept path handles it. Soak.
3. **Add clarification** (§3) once resolution is confirmed clean in logs.
4. **Remove legacy handling**: once traces show the model emitting `target`/`action` consistently,
   drop the loose-key coercion and, optionally, retire `fusion._repair_control_device` and simplify
   tool-registry's `state→service`/`domain` recovery hacks (cortex now sends clean calls).
Each step is one deploy on the integration branch, independently revertible via the advertised-schema
flip (behavior is contract-driven).

## 7. Eval / test plan (reuse the fixtures from e912596)
Assert against **real** `iris_mappings` shape (alias phrases → one entity), not fabricated fixtures —
that discipline is what found the dedup bug. Cases:
- alias-to-same-entity ("living room lamp" + "living room lamps" → `light.living_room_lamps`) — not ambiguous
- underscored slug (`living_room_lamps`), plural/singular, punctuation
- genuinely ambiguous (ceiling vs lamps → clarification) ; unmapped (→ honest "no such device")
- brightness (`action=set` + `brightness_pct`) ; multi-room `select_content`
- dual-accept: model emits legacy `{entity_id, state}` → still works
- live smoke: trace a "turn off the living room lamps" turn → `coerced/resolved → entity=… service=…`,
  tool-registry `success=True` non-zero `duration_ms`, fusion silent.

## 8. Open decisions for Matt
1. **Resolution placement A vs B** — recommend A (cortex, reuse the working resolver). OK?
2. **v1.1 targeted-search fallback** — build now, or ctx-only until miss-rate warrants?
3. **Clarification wording/aggressiveness** — clarify on any >1, or auto-pick a same-room match when
   room-locality is unambiguous and only clarify across rooms?
4. **Retire the fusion net + tool-registry hacks** in step 4, or keep them as defense-in-depth?
