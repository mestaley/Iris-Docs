# Host consolidation — execution runbook

Companion to [HOST-CONSOLIDATION.md](HOST-CONSOLIDATION.md). Ordered steps to migrate all pve-01 VMs
onto pve-02/pve-03 and retire pve-01. Physical steps are Matt's (⚙); VM-software steps Claude can run
via root (▶). Verify (✓) after each.

**Migration mechanism (corrected 2026-09-28):** it's a cluster and every node has the same-named
`local-zfs` (rpool/data), so **`qm migrate <vmid> <target>` transfers the disk via ZFS send** — no
manual vzdump/restore. Caveats: (a) **GPU-passthrough VMs migrate OFFLINE** (hostpci blocks live
migration) — brief downtime, and remove `hostpci` before migrate, re-add on the target; (b) **world-ai
(210) disk is on `rpool-zfs` (pve-02-pinned)** → add `--targetstorage local-zfs`; (c) infra
(net-tools/infra-01/monitor-01) are likely **LXC** → `pct migrate` (also ZFS-based, offline for
running CTs unless `--restart`). The physical RAM move + GPU re-passthrough are still the real work.

Legend: ⚙ = physical/Matt · ▶ = software (Claude or Matt) · ✓ = verify

## Pre-flight (do FIRST — gates the whole plan)
- ⚙ **Confirm pve-02 RAM path**: how many DIMM slots, max capacity, and does pve-01's DDR match pve-02's?
  voice-ai + infra need pve-02 at **≥48G** (it's 15G today). If pve-01's sticks don't fit pve-02, the
  fallback is voice-ai as a **CPU build** on pve-02 (slower STT/TTS) — decide before teardown.
- ▶ Snapshot current state: `qm config <vmid>` for 200/205/210/110/111/112 saved off-node (so we can
  rebuild net/disk/passthrough exactly). Claude can dump these now.
- ✓ Confirm audio-ai (202) + core-ai (201) untouched on pve-03 the whole time (they don't move).

## Phase 1 — world-ai off its GPU, onto pve-03 (frees pve-02's 5060 Ti)  ✅ DONE 2026-09-28
> world-ai migrated pve-02→pve-03 (offline, `--targetstorage local-zfs`), hostpci0 removed, running GPU-less on pve-03 (IP 172.16.10.21, HA proxy/sensors/API verified). pve-02's 5060 Ti freed for voice-ai. ollama runs on CPU ~<1s (reflex still functional). Iris fully operational.
1. ⚙/▶ **Retire world-ai's reflex GPU**: stop `ollama` on world-ai; `qm set 210 --delete hostpci0`
   (removes the 5060 Ti). Reflex now errors → curator falls through to cortex-logical (Iris still
   actuates, ~2-3s, silent). *No fabric code change needed.*
2. ▶ **Move world-ai (210) pve-02 → pve-03** (offline; hostpci0 already removed in step 1):
   `qm migrate 210 pve-03 --targetstorage local-zfs` (ZFS-sends the disk off the pve-02-pinned
   `rpool-zfs` onto pve-03's `local-zfs`) → `qm start 210` on pve-03.
3. ✓ world-ai: sensors/Meridian/HA-proxy up; DDS domain-42 rejoin (curator sees it); `/spotify` + HA
   proxy reachable; curator→world-ai `/fastpath` will 5xx (expected — reflex degraded).
4. ✓ Iris end-to-end: a light command still actuates (via logical now), speech-output voices normally.

> **Ordering gotcha (the RAM chicken-and-egg):** voice-ai + infra must vacate pve-01 *before* you
> power it down to pull its RAM — but pre-upgrade pve-02 (15G) can't hold everything at once. After
> world-ai leaves pve-02 (Phase 1), pve-02 has ~12G free: **voice-ai fits, the infra doesn't.** So
> **park the light infra (bastion + the LXCs) on pve-03 temporarily** (it has ~19G free) to empty
> pve-01, do the RAM swap, then land them on pve-02 in Phase 4. (Or just leave them on pve-03 — it has
> the room — if you're fine relaxing "infra on pve-02.")

## Phase 2 — physical: retire-in-place prep on pve-01 + RAM to pve-02
5. ▶ **Optional insurance**: a `vzdump --mode snapshot` of 205/200/111/112 to pve-02's `memory-data`
   (7 T free) before touching pve-01 — `qm migrate` doesn't need it, but it's a cheap rollback if a
   transfer is interrupted mid-move.
6. ⚙ Schedule a maintenance window (bastion + monitor-01 moves = external-access + monitoring outage).
7. ⚙ Power down pve-01 **and** pve-02; move RAM DIMMs pve-01 → pve-02 (→ ≥48G). Leave pve-02's 5060 Ti
   in place. Boot pve-02.
8. ✓ pve-02 posts with the new RAM; `free -g` shows the upgrade; existing VMs (memory-ai, world-ai
   already moved off) healthy.

## Phase 3 — voice-ai onto pve-02 with the freed GPU  ✅ DONE 2026-09-28
> voice-ai migrated pve-01→pve-02 (offline, 71.5G ZFS send), re-attached the freed 5060 Ti (@01:00), running: GPU bound (~6.9G models loaded), IP unchanged 172.16.10.15, Kokoro TTS :8880=200 reachable from audio-ai, STT :10300 up. **No RAM upgrade was needed** (world-ai vacating pve-02 freed the 8G).
9. ⚙ Confirm pve-02's 5060 Ti (01:00) is free (world-ai released it in Phase 1) and IOMMU-isolated.
10. ▶ **Move voice-ai (205) → pve-02**: `qm set 205 --delete hostpci0` (drop pve-01's GPU ref) →
    `qm migrate 205 pve-02` (disk already on `local-zfs`, ZFS-sends) → on pve-02 attach the freed
    5060 Ti: `qm set 205 --hostpci0 0000:01:00,pcie=1,x-vga=1` → `qm start 205`.
11. ✓ voice-ai: `nvidia-smi` sees the 5060 Ti; Kokoro TTS + Canary/ASR run on GPU; speech-output emits
    audio on a real turn; STT finalizes.

## Phase 4 — infra onto pve-02
12. ▶ Move bastion (200, `qm migrate`) + infra-01/monitor-01/net-tools (likely LXC → `pct migrate
    <ctid> pve-02 --restart`) onto pve-02.
13. ✓ bastion: Cloudflare tunnel/ztna back (iris.mattstaley.com reachable). ✓ monitor-01: Loki ingest
    + Grafana resume (do this move LAST; monitoring is blind during it — keep a scratch `journalctl` tail).
    ✓ infra-01 / net-tools: their services (internal NTP, etc.) back.

> **bastion (200) goes LAST — it hosts the live Claude Code session driving this migration.** Migrating
> bastion (or powering down pve-01 for the RAM pull) kills the session, and it does not auto-resume. So
> Claude drives Phases 1–4 (world-ai, voice-ai, infra LXCs) while alive, verifying each; then Matt does
> the bastion cutover + RAM swap + pve-01 retirement (Phase 5) solo, and **starts a fresh Claude session
> on the relocated bastion** — it reads this runbook + HOST-CONSOLIDATION.md to verify the end state and
> resume the code work (reflex cleanup, Path 2).

## Phase 5 — retire pve-01 (Matt solo; Claude session is down)
14. ⚙ Confirm every VM accounted for on pve-02/pve-03 and healthy for a full turn cycle + a proactive
    (welcome/briefing) + a light command + a status query.
15. ⚙ Remove pve-01 from the cluster: `pvecm delnode pve-01` (from a surviving node), clean
    `/etc/pve/nodes/pve-01`. Power off; pull it for the Z2 Frankenstein. 5060 Ti → cold spare.

## Post-migration target (verify matches)
- **pve-02** (≥48G, 1× 5060 Ti): memory-ai, voice-ai(GPU), world-ai?*, bastion, infra-01, monitor-01,
  net-tools. *(world-ai went to pve-03 in Phase 1; if you'd rather keep it on pve-02, skip step 2 —
  but pve-02 is tighter, so pve-03 is preferred.)*
- **pve-03** (62G, 2× 5080): core-ai (2×5080), audio-ai, world-ai (no GPU).
- **pve-01**: gone.

## Then — resume code work (the "continue" phase)
- Formal reflex retirement cleanup (remove the fast-path call path / world-ai LLM service + `actuation_tools`
  trim), now that it's already degraded.
- Path 2: cortex-logical as sole actuator + live-state query loop (the light-status grounding fix).
- Owed origin pushes (fusion `feat/punt-escalation`, world-ai, iris-docs) + staged restarts.

---
## ⚠ Phase 1 fallout — the reflex was load-bearing (2026-09-28)
Removing world-ai's GPU killed the reflex (CPU tool-gen > 5s curator budget → timeout),
which **exposed that cortex-logical's `control_device` is malformed** (`{entity_id, state}`,
no domain/service) and has always failed 0ms — the reflex was masking it. Actuation broke
("can't reach the device", though HA is reachable). So the migration **forced the
sole-actuator transition** (it was NOT a no-op as first assumed). Fixed same day:
- tool-registry: normalize logical's `state→service`. fusion: silence on any completed
  actuation (not just reflex), voice failures synchronously. curator `.env`:
  `CURATOR_FASTPATH_ENABLED=false` (disable the dead reflex).
- **Restarts required: iris-tool-registry, iris-fusion-executive, iris-curator.**
- Lesson for the rest of the migration: the reflex is gone; logical is the sole actuator.

---
## Phase 4 progress + Phase 5 bastion commands (Matt runs these) — 2026-09-28
Migrated to pve-03 (parked; rebalance to pve-02 after the RAM upgrade): infra-01 (111, Pi-hole
DNS + chrony NTP — ~15s DNS blip), monitor-01 (112, ~1m43s). **net-tools (110) is NOT migrated —
abandon it with pve-01.** pve-01 now holds only **bastion (200)** + net-tools.

### Bastion (200 → pve-03) — run NOT through bastion (use the Proxmox web UI, or a shell on pve-01/pve-03 directly; bastion is the SSH jump host)
bastion has no GPU → LIVE migration works and should keep the Claude session alive:
```
qm migrate 200 pve-03 --online --with-local-disks      # live; sub-second pause, session survives
```
Fallback if online is refused (kills the Claude session — fine, work is done):
```
qm stop 200 && qm migrate 200 pve-03 && qm start 200
```
(Web UI: select VM 200 → Migrate → target pve-03 → it picks online+local-disks automatically.)

### Retire pve-01
```
qm list                     # on pve-01: should show only 110 (net-tools, stopped)
# power off pve-01; move its RAM DIMMs into pve-02 (target >=48G); pull the 5060 Ti (cold spare)
pvecm delnode pve-01        # from pve-02 or pve-03
# (optional) rm -rf /etc/pve/nodes/pve-01   # only after delnode, from a surviving node
```

### After pve-01 gone + pve-02 RAM upgraded — rebalance infra to pve-02 (fresh session, optional)
```
qm migrate 200 pve-02                  # bastion
pct migrate 111 pve-02 --restart 1     # infra-01 (brief DNS blip)
pct migrate 112 pve-02 --restart 1     # monitor-01 (brief monitoring blip)
```

---
## ⛔ BLOCKER (2026-09-28): bastion can't move — pve-03 QEMU behind
Migrated OK: world-ai→pve-03, voice-ai→pve-02, infra-01→pve-03, monitor-01→pve-03. **bastion (200)
blocked**: live-migrate failed — `[pve-03] QEMU 10.1.2 too old for machine type pc-q35-11.0`.
Versions: **pve-01 = pve-manager 9.1.11 / QEMU 11.0.0** (kernel 7.0.2); **pve-03 = 9.1.9 / QEMU 10.1.2**
(kernel 6.17.2). bastion (born on pve-01) is pinned pc-q35-11.0 → won't run on pve-03 (live OR offline).
world-ai/voice-ai migrated fine (older/compatible machine types). Bastion is still running on pve-01.
**Fix options:** (a) update pve-03's `pve-qemu-kvm` to 11 (no reboot needed — target starts fresh
QEMU for the incoming VM), then live-migrate bastion (session survives); full update incl. kernel →
schedule a reboot window (whole-Iris outage, everything's on pve-03). (b) `qm set 200 --machine
pc-q35-10.0` + offline migrate (session dies). (c) leave bastion on pve-01, retire later.
**pve-01 retirement is blocked until bastion moves.** net-tools (110) is abandoned with pve-01.

---
## ✅ MIGRATION COMPLETE — 2026-09-28 (pve-01 drained)
After Matt updated pve-03 (→ pve-manager 9.2.20 / QEMU 11.0.3), **bastion LIVE-migrated to pve-03 —
the driving Claude session survived the switchover.** Final layout, Iris healthy (fabric target
active, 0 failed units; voice TTS 200; DNS ok; pve-03 54G/67G used):
- **pve-02**: memory-ai (105), voice-ai (205, 5060 Ti)
- **pve-03**: core-ai1 (201, 2×5080), audio-ai (202), world-ai (210, no GPU), + parked infra:
  infra-01 (111, DNS/NTP), monitor-01 (112), bastion (200)
- **pve-01**: only net-tools (110, stopped) — abandon it.

### Remaining (Matt, physical):
1. Power down pve-01; move its RAM DIMMs → pve-02 (→ ≥48G); pull the 5060 Ti (cold spare).
2. `pvecm delnode pve-01` (from pve-02/pve-03); optionally `rm -rf /etc/pve/nodes/pve-01`.
3. AFTER pve-02 RAM upgrade — rebalance parked infra pve-03→pve-02 (lightens pve-03):
   `qm migrate 200 pve-02; pct migrate 111 pve-02 --restart 1; pct migrate 112 pve-02 --restart 1`
   (bastion 200 is now QEMU-11 machine-typed; pve-02 must be on QEMU 11 too — verify before live-migrate.)
