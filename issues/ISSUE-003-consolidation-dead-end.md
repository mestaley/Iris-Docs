# ISSUE-003: Consolidation cluster is a write-only dead-end (complexity for zero payoff)
- Status: WRITE HARD-GATE STAGED 2026-08-23 — iris-fusion-fabric commit 13d66b5: `on_trigger` forces shadow (write-nothing) unless `DREAM_CONSOLIDATION_WRITE_ENABLED=true`. +3 tests, 7/7 green; service.py synced to core-ai (safe pre-restart: only makes dream-mode more conservative). Takes runtime effect on `sudo systemctl restart iris-dream-mode`. Operational belt (disable the nightly trigger timer) still needs Matt's sudo — see staged commands. Per ADR-001 the consolidation CODE is KEPT (roadmap: future read-back wiring), only the writer is gated.
- Severity: medium
- Area: memory
- Discovered: 2026-08-22
- Evidence: `fusion-executive/.../memory_client.py:51` (zero call sites); `dream-mode/.../consolidator.py:42,64`; live `:8013/consolidated/count` = 355; `:8013/consolidated/recall` → "no consolidated record recall"

## The important correction
The thesis-recentering note feared consolidation was "corrupting prefs LIVE." The code and live state say **it cannot** and **is not**:
- The live turn reads the **episodic** tier (`app.py:147 recall()`). The consolidator only POSTs to `/consolidate` + `/consolidate/judge` — the **consolidated** tier. It never calls `/write`. Two different stores.
- Read-back is **open**: `recall_precedent()` is defined but has **zero call sites** (grep). `ctx.recalled_precedent` stays the empty default.
- Live confirmation: `/consolidated/recall` returns "no consolidated record recall" — there is **no working read endpoint at all**. 355 records that nothing can consult.

So the hollow records (salience→0 from dead affect source, `extract.py:33`; neutral affect vector) are real garbage rotting in an **unread** store. Zero effect on responses.

## Live status
- `iris-dream-mode.service` active but **trigger-only** (nightly `iris-dream-mode-trigger.timer` waiting; trigger unit currently inactive). `dream-mode.env` absent → shadow defaults → writer dormant.
- `iris-reflection-coordinator.service` active but its unit is **`disabled`** (running now, won't survive a reboot).

## Decision needed (see ADR-001)
This is the cleanest "nuke half" candidate: three services + a whole memory tier + 355 dead records, provably read by nothing. Options: (a) hard-gate/stop the writer + leave inert; (b) delete the services and tier. Either is safe for responses. Do NOT touch the episodic tier, working-memory field, or prosodic affect — those are the live, value-earning pieces.
