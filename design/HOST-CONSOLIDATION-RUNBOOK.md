# Host consolidation — execution runbook

Companion to [HOST-CONSOLIDATION.md](HOST-CONSOLIDATION.md). Ordered steps to migrate all pve-01 VMs
onto pve-02/pve-03 and retire pve-01. **No shared storage → every move is offline** (stop → vzdump →
restore + re-passthrough). Physical steps are Matt's (⚙); VM-software steps Claude can run via root (▶).
Verify (✓) after each. Stage all backups on pve-02's `memory-data` pool (7 T free).

Legend: ⚙ = physical/Matt · ▶ = software (Claude or Matt) · ✓ = verify

## Pre-flight (do FIRST — gates the whole plan)
- ⚙ **Confirm pve-02 RAM path**: how many DIMM slots, max capacity, and does pve-01's DDR match pve-02's?
  voice-ai + infra need pve-02 at **≥48G** (it's 15G today). If pve-01's sticks don't fit pve-02, the
  fallback is voice-ai as a **CPU build** on pve-02 (slower STT/TTS) — decide before teardown.
- ▶ Snapshot current state: `qm config <vmid>` for 200/205/210/110/111/112 saved off-node (so we can
  rebuild net/disk/passthrough exactly). Claude can dump these now.
- ✓ Confirm audio-ai (202) + core-ai (201) untouched on pve-03 the whole time (they don't move).

## Phase 1 — world-ai off its GPU, onto pve-03 (frees pve-02's 5060 Ti)
1. ⚙/▶ **Retire world-ai's reflex GPU**: stop `ollama` on world-ai; `qm set 210 --delete hostpci0`
   (removes the 5060 Ti). Reflex now errors → curator falls through to cortex-logical (Iris still
   actuates, ~2-3s, silent). *No fabric code change needed.*
2. ▶ **Move world-ai (210) pve-02 → pve-03** (offline): `qm stop 210` → `vzdump 210 --storage
   <pve02-backup> --mode stop` → copy the dump to pve-03 → `qm restore` on pve-03 as GPU-less → boot.
3. ✓ world-ai: sensors/Meridian/HA-proxy up; DDS domain-42 rejoin (curator sees it); `/spotify` + HA
   proxy reachable; curator→world-ai `/fastpath` will 5xx (expected — reflex degraded).
4. ✓ Iris end-to-end: a light command still actuates (via logical now), speech-output voices normally.

## Phase 2 — physical: retire-in-place prep on pve-01 + RAM to pve-02
5. ▶ **Back up all remaining pve-01 VMs** to pve-02's `memory-data`: vzdump 205 (voice-ai), 200
   (bastion), 111 (infra-01), 112 (monitor-01), 110 (net-tools, stopped). Do while running
   (`--mode snapshot`) to stage without downtime, OR `--mode stop` at cutover.
6. ⚙ Schedule a maintenance window (bastion + monitor-01 moves = external-access + monitoring outage).
7. ⚙ Power down pve-01 **and** pve-02; move RAM DIMMs pve-01 → pve-02 (→ ≥48G). Leave pve-02's 5060 Ti
   in place. Boot pve-02.
8. ✓ pve-02 posts with the new RAM; `free -g` shows the upgrade; existing VMs (memory-ai, world-ai
   already moved off) healthy.

## Phase 3 — voice-ai onto pve-02 with the freed GPU
9. ⚙ Confirm pve-02's 5060 Ti (01:00) is free (world-ai released it in Phase 1) and IOMMU-isolated.
10. ▶ **Restore voice-ai (205) onto pve-02**; attach the 5060 Ti: `qm set 205 --hostpci0
    0000:01:00,pcie=1,x-vga=1`; boot.
11. ✓ voice-ai: `nvidia-smi` sees the 5060 Ti; Kokoro TTS + Canary/ASR run on GPU; speech-output emits
    audio on a real turn; STT finalizes.

## Phase 4 — infra onto pve-02
12. ▶ Restore bastion (200), infra-01 (111), monitor-01 (112), net-tools (110) onto pve-02.
13. ✓ bastion: Cloudflare tunnel/ztna back (iris.mattstaley.com reachable). ✓ monitor-01: Loki ingest
    + Grafana resume (do this move LAST; monitoring is blind during it — keep a scratch `journalctl` tail).
    ✓ infra-01 / net-tools: their services (internal NTP, etc.) back.

## Phase 5 — retire pve-01
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
