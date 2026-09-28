# 2026-09-28 — Host consolidation (retire pve-01) + forced sole-actuator transition

## Executive summary
Consolidated the 3-node cluster toward 2 active nodes: migrated every VM off **pve-01** so it can be
retired (its parts → Matt's Z2 workstation). world-ai → pve-03 (GPU dropped), voice-ai → pve-02 (took
world-ai's freed 5060 Ti), infra-01 (DNS/NTP) + monitor-01 → pve-03, and **bastion live-migrated to
pve-03 with the driving Claude session surviving the switchover**. pve-01 now holds only net-tools
(stopped, abandoned); its physical retirement (RAM → pve-02, GPU → spare, `pvecm delnode`) is Matt's.
Iris healthy throughout on the new layout. Removing world-ai's GPU **forced the sole-actuator code
transition early** (the reflex was masking cortex-logical's always-broken actuation) — a cluster of
fabric fixes landed to make cortex-logical the sole actuator. All committed; **staged fabric restarts
were applied during the session** (tool-registry, fusion, curator). LIVE.

## Changes by repo

### Infrastructure (Proxmox cluster — no repo; see design/HOST-CONSOLIDATION*.md)
- **world-ai (210)** pve-02 → pve-03, GPU passthrough removed (offline `qm migrate --targetstorage local-zfs`).
- **voice-ai (205)** pve-01 → pve-02, re-attached pve-02's freed 5060 Ti (@01:00) — no CPU build, no RAM upgrade needed (world-ai vacated the room).
- **infra-01 (111, Pi-hole DNS + chrony NTP)** pve-01 → pve-03 (`pct migrate --restart`, ~15s DNS blip).
- **monitor-01 (112)** pve-01 → pve-03 (`pct migrate --restart`, ~1m43s Loki/Grafana blip).
- **bastion (200)** pve-01 → pve-03 (**live** `qm migrate --online --with-local-disks`, session survived). Required Matt updating pve-03 first (QEMU 10.1.2 → 11.0.3; bastion pinned pc-q35-11.0).
- **curator `.env` (core-ai, host)** — `CURATOR_FASTPATH_ENABLED=false`: reflex disabled (world-ai GPU-less → reflex timed out, 5s hang/turn).

### iris-fusion-fabric — branch `feat/punt-escalation`
- `tool-registry/src/tool_registry/dispatch.py` — **changed**: normalize logical's HA-idiom `{state:on/off}` → `{service:turn_on/turn_off}` (domain already derived from entity prefix); reject a call with neither.
- `fusion-executive/src/fusion_executive/service.py` — **changed**: `_repair_control_device` fills a dropped on/off intent from `user_text` before dispatch; `_silent_reflex_actuation`→`_silent_completed_actuation` keyed on `actuated_ok` (silence ANY completed actuation, not just reflex); failed-actuation correction voices synchronously (no reflex-supervisor to defer to); creative ignored when `enable_creative` off (authoritative retirement, from earlier today).
- `fusion-executive/tests/{test_proactive_punt,test_reflex_dedup}.py` — **changed**: predicates renamed/rekeyed, control_device-repair + failure-voices-correction tests. 57 green.

### iris-docs
- `design/HOST-CONSOLIDATION.md` + `design/HOST-CONSOLIDATION-RUNBOOK.md` — **added/updated**: plan, executable runbook, per-phase completion, the QEMU blocker + resolution, migration-complete state.
- `changelog/2026-09-28-*.md` (this + the earlier silent-actuation/punt-noise entry).

## Deploy / live state
- **LIVE**: full migration (pve-01 drained); sole-actuator fabric (tool-registry + fusion + curator all restarted this session). Iris healthy: fabric target active, 0 failed units, voice TTS 200, DNS ok.
- **Pending (Matt, physical)**: power down pve-01, RAM → pve-02 (≥48G), 5060 Ti → cold spare, `pvecm delnode pve-01`; then rebalance parked infra (bastion/infra-01/monitor-01) pve-03 → pve-02.

## Follow-ups / owed
- Origin pushes owed (automation can't push): fusion `feat/punt-escalation` (large: silent-actuation, punt floor, reflex-status-gate, sole-actuator), world-ai `fix/actuation-settle-verify`, iris-docs. `/opt/repos/_push-staging/`.
- **Code work Matt deferred to "after stable":** Path 2 (cortex-logical live-state query loop — the light-status grounding, per `iris-prefetch-only-slow-external`); formal reflex-code removal (it's disabled, not deleted); cortex-logical tool-call reliability (the dropped-`service` root cause the fusion repair papers over); rotate the pve root password (shared in-channel).
