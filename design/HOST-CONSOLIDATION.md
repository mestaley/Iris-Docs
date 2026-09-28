# Host consolidation — retire pve-01, 3 nodes → 2 active roles

Status: **PROPOSAL 2026-09-28** (inventory complete; awaiting go). Owner: Matt.
Prereq: **retire the fast-path reflex + world-ai LLM** (frees world-ai's GPU and makes world-ai a
light, GPU-less VM) — see ISSUE-013 / the 2026-09-28 changelog.

## Goal (Matt)
Retire **pve-01**. Consolidate onto **pve-02** (storage + infra + memory) and **pve-03** (compute/GPU).
Redistribute pve-01's RAM to the survivors.

## Current inventory (2026-09-28, from cluster)
Cluster of 3 (pve-01/02/03). **Not a single shared pool**, but every node has the same-named `local-zfs` (rpool/data) and it's a
cluster ⇒ **`qm migrate` transfers disks via ZFS send** (straightforward, one command). GPU-passthrough
VMs migrate OFFLINE (hostpci blocks live) — brief downtime. `memory-data` 7.27T + `rpool-zfs` are
`nodes pve-02`-pinned, so world-ai (disk on `rpool-zfs`) needs `--targetstorage local-zfs` to move.

| node | cores | RAM | NVIDIA GPU | VMs (vmid, mem) |
|---|---|---|---|---|
| **pve-01** *(retire)* | 16 | 31G | 1× RTX 5060 Ti | bastion(200,4G), voice-ai(205,8G,**GPU**), net-tools(110,1G,stopped), infra-01(111,1G), monitor-01(112,2G) |
| **pve-02** *(storage)* | 16 | **15G** (~11 used) | 1× RTX 5060 Ti → world-ai | memory-ai(105,4G), world-ai(210,8G,**GPU**) + `memory-data` 7.27T pool |
| **pve-03** *(compute)* | 20 | 62G (~43 used) | 2× RTX 5080 + Intel iGPU | core-ai1(201,26G,**2×5080**), audio-ai(202,17G) |

GPU total = 4 NVIDIA: 2× 5080 (core-ai), 2× 5060 Ti (voice-ai on pve-01, world-ai on pve-02).

## Key decisions this enables
1. **Reflex/world-ai LLM retired** → world-ai's 5060 Ti (in pve-02) is freed. We drop to the 3 GPUs
   actually needed: 2× 5080 (cortex-logical + deep-mode) + 1× 5060 Ti (voice STT/TTS). **Losing one
   GPU is the consequence, not a compromise.**
2. **voice-ai keeps its GPU without a CPU build OR a card move:** it moves pve-01 → **pve-02** and
   inherits the 5060 Ti **already physically in pve-02** (world-ai vacates it). This resolves the
   "voice-ai CPU build or pve-02" fork — pve-02, with a GPU.

## Target topology
| node | role | VMs | GPUs |
|---|---|---|---|
| **pve-02** | storage + infra + memory + **voice** | memory-ai, **voice-ai** (+5060 Ti), bastion, infra-01, monitor-01, net-tools | 1× 5060 Ti → voice-ai |
| **pve-03** | compute | core-ai1 (2×5080), audio-ai, **world-ai** (no GPU) | 2× 5080 → core-ai |
| **pve-01** | RETIRED | — | 5060 Ti → cold spare |

## Capacity analysis
- **pve-03**: core-ai(26) + audio-ai(17) + world-ai(8) = **51G / 62G** ✓ (world-ai is GPU-less now, fits).
- **pve-02** is the bottleneck: memory-ai(4) + voice-ai(8) + bastion(4) + infra-01(1) + monitor-01(2) +
  net-tools(1) ≈ **20G guest**, but pve-02 has only **15G**. ⇒ **pve-02 needs a RAM upgrade** — move
  DIMMs from the retired pve-01 (31G) into pve-02 (target ≥ 48G for headroom). *Open: pve-02 board's
  DIMM slots / max capacity + DDR generation match with pve-01 — confirm before ordering the teardown.*

## Migration order (offline; each step reversible until pve-01 is wiped)
0. **Retire reflex** (code ready): set world-ai LLM off, stop `ollama` on world-ai, remove world-ai's
   `hostpci0` (the 5060 Ti) from its config. world-ai now GPU-less.
1. **world-ai → pve-03**: `qm stop 210` → `vzdump`/`zfs send` 210 to pve-03 → restore as GPU-less →
   verify sensors/Meridian/HA-proxy + DDS (domain 42) rejoin + curator→world-ai URL still resolves.
2. **Physical**: power down pve-01 + pve-02; move RAM pve-01→pve-02 (→≥48G); (optionally pull pve-02's
   5060 Ti stays put). Boot pve-02.
3. **Re-passthrough** pve-02's 5060 Ti (now free) to a fresh voice-ai config.
4. **voice-ai → pve-02**: offline move 205 to pve-02, attach the 5060 Ti (`hostpci0`), verify
   Kokoro/ASR on GPU + speech-output path.
5. **infra → pve-02**: move bastion(200), infra-01(111), monitor-01(112), net-tools(110). Verify:
   bastion = Cloudflare tunnel/ztna still up; monitor-01 = Loki/Grafana ingest resumes.
6. **Decommission pve-01**; 5060 Ti → labeled cold spare.

## Risks / watch-items
- **No shared storage** → real downtime per VM + manual backup staging (use pve-02's `memory-data` 7T
  free as the vzdump target). Sequence so only one service is down at a time.
- **GPU passthrough** must be re-established per VM on the new node (IOMMU groups, vfio bind, `x-vga`
  for voice-ai). The 5080s on pve-03 don't move — only VM configs.
- **Identity/networking**: hostnames, static IPs, the DDS domain-42 peer set, and cross-host URLs
  (curator→world-ai `/fastpath` goes away with the reflex; `/spotify`, HA proxy stay) must survive.
- **Self-monitoring blind spot**: moving monitor-01 blinds Loki/Grafana during its move — do it last,
  keep a scratch log tail.
- **bastion move = external-access risk**: the Cloudflare tunnel for iris.mattstaley.com rides bastion;
  plan a short outage window.

## Open items (need Matt / hardware)
- pve-02 DIMM slots + max RAM + DDR match with pve-01's sticks.
- Confirm world-ai → pve-03 (vs. leave on pve-02) — pve-03 has the room; pve-02 is tight, so pve-03 is
  preferred.
- Whether pve-03 could ever take a 3rd GPU (board slots) — not needed under this plan; noted for future.
