# 2026-10-21 — Hard-block the `lock` domain from Iris (deadbolt safety invariant)

## Executive summary
Matt flagged that the HA z-wave deadbolt (the only `lock` entity, present only for the UniFi
doorbell fingerprint→unlock automation) must **never** be exposed to Iris. Added a hard,
code-level block at world-ai's `/ha/service` proxy — the single holder of the HA token and the
chokepoint every actuation funnels through — refusing both actuation and state-read for any entity
in a configurable blocked-domains set (default `lock`). Verified with a focused test (3 pass). Also
verified earlier in the session that no `lock.*` entity currently exists in `iris_mappings` or
`iris_tools` (the weak "is the front door locked"→porch-light match that prompted this was *correct*
— there is no lock mapping to find). **Staged, not deployed** (needs Matt's push + restart).
Recorded as a never-violate SAFETY INVARIANT in SSOT + memory so no future mappings reseed or model
swap can re-expose the door. Context: this is during the slim-rig migration (Phase 5); the curator
was repointed to local Qdrant and mappings verified resolving at the top of the session.

## Changes by repo
### world-ai — branch `fix/block-lock-domain` (new, commit f848fe0)
- `app/config.py` — **changed**: added `blocked_domains: str = "lock"` setting (env
  `WORLD_AI_BLOCKED_DOMAINS`), documented as the deadbolt guardrail.
- `app/services/ha_service.py` — **changed**: `_BLOCKED_DOMAINS` (normalized from config) +
  `_is_blocked()` helper; `call_service` refuses blocked actuation before any HA call,
  `get_state` raises `PermissionError` on a blocked entity. Entity_id prefix checked too, so a
  spoofed `{domain:"light", entity_id:"lock.*"}` is still caught.
- `tests/test_ha_block.py` — **added**: proves lock actuation refused (no HA touch), lock read
  refused, and legitimate light/switch domains pass. 3 pass (`uv run --with pytest`).

### iris-docs — branch `main`
- `PRINCIPLES.md` — **changed**: new "Safety invariants (never violate)" section — `lock` domain
  off-limits (actuate + read), enforced in code below the model; corollary never to seed `lock.*`
  into mappings/tools.
- `design/SLIM-RIG-RUNBOOK.md` — **changed**: "Safety invariant — carry across the rebuild" note so
  the block is cherry-picked onto the live world-ai branch and no `lock.*` is reseeded during the
  Qdrant restore.
- `changelog/2026-10-21-block-lock-domain.md` — **added**: this entry.

### auto-memory
- `iris-lock-domain-never-expose.md` — **added** (+ MEMORY.md index line): the safety invariant,
  the enforcement point, and the never-reseed rule.

## Deploy / live state
- **STAGED, not live.** world-ai `fix/block-lock-domain` (f848fe0) is committed locally only.
  Deploy = Matt: push, cherry-pick f848fe0 onto whichever world-ai branch is actually running
  (`feat/presence-fusion` and `fix/actuation-settle-verify` have diverged 2/2; both carry the
  settle-verify `ha_service.py` the block sits on, so it applies cleanly either way), then
  `sudo systemctl restart iris-world-ai`.
- Verified pre-existing safe state: no `lock.*` in `iris_mappings`/`iris_tools` (confirmed via the
  session's mapping-retrieval test; the lock has no mapping, so target resolution can't reach it).
  The block is defense for the future-reseed and hallucination cases, not a fix for a live leak.
- **No effect on the doorbell fingerprint→unlock** — that's HA-internal on HA's own token, not this
  proxy.

## Follow-ups / owed
- **Matt:** origin push of world-ai `fix/block-lock-domain` (f848fe0) + cherry-pick onto the live
  branch + `sudo systemctl restart iris-world-ai`.
- Slim-rig Phase 5: when restoring Qdrant and (if ever) reseeding mappings/tools from HA states,
  exclude blocked domains. Linked in SLIM-RIG-RUNBOOK.md.
- iris-docs still ahead 50 of origin (pre-existing); this session added PRINCIPLES/runbook/changelog.
