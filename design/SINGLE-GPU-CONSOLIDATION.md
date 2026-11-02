# Single-GPU consolidation — full assessment (proposed)

- Status: **direction chosen 2026-10-15** (variant of Option B) — pending the STT-trim validation gate
- **Chosen plan (Matt, 2026-10-15):** 2 GPUs for Iris + audio, cloud for the rest.
  - **Keep:** 5060 Ti 16GB (front-door 8B LLM + embed) · 3060 Ti 8GB ex-world-ai (STT + TTS) · A400 (Plex, NVENC confirmed, with the 8TB storage).
  - **Sell:** both RTX 5080s + the RTX 4080 (~$2.6–3k ballpark).
  - **Rent cloud GPU** ad-hoc for large-model experimentation (not production).
  - **Deep/hard-turn tier → cloud API** (Claude/ChatGPT), escalation-only; **retire memory-ai → local NVMe Qdrant**.
  - **Consequences:** local LLM ceiling is now **8B** (no 5080) → front door must be an 8B (Hermes-3-8B or
    qwen3-8B); **qwen2.5:14b retired** from the plan. The audio card is 8GB → **STT MUST be trimmed**
    (distil/medium whisper) to fit alongside kokoro — this is the one open gate before executing.
  - Delete locally once off: qwen3-deep (cloud now), gemma3 (creative retired).
- Proposal (Matt): collapse to a **single GPU, single local model**; keep the **audio fabric** local;
  **fail over to Claude/ChatGPT** for hard turns; **sell the surplus GPUs**; **retire memory-ai** and put a
  dedicated **1TB NVMe** in for the vector DB.
- Verdict up front: **feasible and well-motivated, with two load-bearing caveats** — (1) it crosses the
  thesis's "100% local" line (cloud failover), a deliberate philosophy change; (2) STT/TTS and the LLM
  would share one card, introducing GPU contention that doesn't exist today. A **2-GPU variant** removes
  caveat (2) while keeping most of the win — see §6.

## 1. Current state (measured 2026-10-15)
| Component | Where | VRAM | Notes |
|---|---|---|---|
| Front-door LLM (hermes3:8b) | core-ai GPU0 | 5.2 GB | qwen2.5:14b (9 GB) is the committed default; Hermes is a live env trial |
| Embed (nomic-embed) | core-ai GPU0 | 0.6 GB | mappings + tool retrieval |
| Thalamus (phi3.5) | core-ai GPU0 | 4.0 GB | routing classifier, critical path |
| Deep tier (qwen3-deep MoE) | core-ai GPU1 | ~9 GB | Q3, completion-only (no tools), fires on punts |
| STT (whisper, pid 1708) | voice-ai 5060Ti | ~8.2 GB | the big audio consumer — likely large-v3 |
| TTS (kokoro, pid 1962) | voice-ai 5060Ti | ~1.2 GB | |
| Qdrant (iris_memory/tools/mappings) | memory-ai LXC | — | **1.4 GB disk used total** — tiny |

GPUs owned: **2× RTX 5080 16GB + 1× RTX 5060 Ti 16GB** (world-ai is CPU-only). Hosts with GPUs: core-ai, voice-ai.

## 2. Can ONE 16GB GPU hold the local stack? — VRAM math
With the deep tier moved to cloud (−9 GB) and routing folded out of phi3.5 (−4 GB):

| Resident | GB |
|---|---|
| Front door (hermes3:8b) | 5.2 |
| Embed (nomic) | 0.6 |
| STT (whisper large) | 8.2 |
| TTS (kokoro) | 1.2 |
| **Total** | **15.2 / 16** |

**Fits — but with ~0.8 GB headroom, which is fragile** (KV-cache growth, fragmentation). Levers that create real room:
- **Trim STT**: distil-whisper-large-v3 (~1.5 GB) or medium (~3 GB) drops the total to ~9–11 GB — comfortable. The quality delta for near-field English commands is small; worth A/B.
- **Keep qwen2.5:14b instead of Hermes** costs +4 GB (9.2 vs 5.2) → 19 GB, does NOT fit alongside large STT. Single-GPU effectively **requires the 8B-class front door** unless STT is trimmed hard.
- **phi3.5 must go** (4 GB): fold routing into the single model as a tool-call (ADR-003 step 3, never done) or a cheap heuristic. CPU phi3.5 is likely too slow for the routing critical path.

## 3. The four moves, assessed

### 3a. Single GPU + single model — FEASIBLE (see §2)
Net removes: the second/third GPU, the local deep tier, and phi3.5. Requires folding routing and (ideally)
trimming STT. The front door becomes genuinely single-purpose, which also makes the **tool-retrieval work
(ADR-005 follow-on) more important, not less** — a single model with no deep-tier backstop must get the
turn right, and scoped tools are what keep its adherence high.

### 3b. Cloud failover (Claude / ChatGPT) — the biggest *philosophy* change
Replaces the local qwen3-deep MoE as the escalation/deep tier.
- **For:** vastly more capable than any local model here; zero VRAM; qwen3-deep was 3-bit **and
  completion-only (no tools)** — it was never a good deep tier anyway; aligns with the already-written
  RECALL-QUEUE "future cloud model" and ADR-003's "rented big model" idea.
- **Against / cost:** **crosses the THESIS "100% local" boundary** (ADR-003 states production stays
  local) — hard turns (and their context) leave the house to a third party. Network dependency (internet
  down → no deep tier, but the local front door still serves the common path → graceful degradation).
  Per-call API cost (trivial if **escalation-only**; material if it becomes the primary brain).
- **Design constraint:** scope failover to **escalation-only** — the punt mechanism (ADR-003/004) pointed
  at a cloud endpoint instead of qwen3-deep. The front door + audio stay local; only genuinely hard/low-
  confidence turns go out. This keeps the common path private and offline-tolerant.
- **Decision for Matt:** are you willing to retire the all-local principle for the deep tier? If yes,
  this is a clean upgrade. If it must stay local, the deep tier stays a (better) local model and the
  single-GPU math tightens.

### 3c. Retire memory-ai → local 1TB NVMe vector DB — EASY WIN
- Active vector data is **tiny** (memory-ai LXC: 1.4 GB used total; 3 small Qdrant collections). A 1TB NVMe
  is ~700× the current need — capacity is a non-issue; even a 256 GB would do, 1TB is cheap future-proofing.
- **Benefits:** one fewer host, recall becomes a local socket call (no network hop → faster), cheap (~$80–100).
- **Care:** export/reimport `iris_memory` (episodic/consolidated) + mappings + tools into the local Qdrant —
  small, scriptable, the seed script already exists for tools. Verify the pgvector/memory-substrate tier
  and the "60-day raw lake" — per earlier findings the consolidated store is hollow/corrupting
  ([[iris-thesis-recentering-2026-08]]), so this is a good moment to **drop or rebuild** it rather than migrate.
- **Risk:** low. Back up the Qdrant snapshot before decommissioning memory-ai.

### 3d. Sell the surplus GPUs — the cash
Keep 1, sell 2. Rough resale: 2× RTX 5080 (~$900–1000 ea used) + 1× RTX 5060 Ti 16GB (~$350–430).
- Keep one **5080**, sell the other 5080 + the 5060 Ti → recoup ~**$1,250–1,430**, keep the faster card.
- Keep the **5060 Ti**, sell both 5080s → recoup ~**$1,800–2,000**, but a 5060 Ti is slower (bandwidth/cores)
  and would feel the STT+LLM contention more. Only advisable with trimmed STT and the 8B front door.
- Recommendation: **keep one 5080** (headroom for concurrency + a 14B option later), sell the rest.

## 4. Risks, ranked
1. **GPU contention (single-GPU only):** STT decode and LLM inference on one card compete; a long transcription
   can stall a reply and vice-versa. Today they're on separate cards → no contention. This is the #1 technical
   risk and the main reason to consider the 2-GPU variant. Mitigation: trimmed STT, request scheduling, or §6.
2. **Single point of failure:** one box runs LLM+embed+STT+TTS+Qdrant. If it dies, Iris is fully down (today
   audio/world/memory are distributed). Acceptable for a homelab **with backups** (Qdrant snapshot, config in git).
3. **Thesis boundary (cloud):** §3b — deliberate, not accidental; decide consciously.
4. **Routing fold:** losing phi3.5 means routing must move into the front door (real work, ADR-003 step 3) or a
   heuristic; a bad router now also mis-scopes tools and mis-prefetches.
5. **Cloud cost/latency/availability** for the deep tier — bounded if escalation-only.

## 5. What it buys
Cash (~$1.3–2k) · one GPU instead of three (power/heat/noise) · fewer hosts (retire memory-ai, possibly
collapse to a single Proxmox node) · faster local recall (NVMe, no network) · a far more capable deep tier
(cloud) than the local 3-bit MoE · much less operational surface (no deep-model management, no storage node).

## 6. Options — pick the risk posture
- **A. Full single-GPU (aggressive):** one 5080 does LLM+embed+STT+TTS; cloud deep tier; NVMe Qdrant; sell 2
  cards. Max cash + simplicity. Carries risk #1 (contention) — needs trimmed STT + folded routing.
- **B. Two-GPU (lower-risk middle, RECOMMENDED to weigh):** keep the **5060 Ti for audio** (STT+TTS, exactly as
  today — zero contention) and **one 5080 for the LLM+embed**; sell **one 5080**. Still: cloud deep tier, NVMe
  Qdrant, memory-ai retired, ~$900–1000 recouped, routing fold optional (phi3.5 fits with headroom on the LLM
  card). You lose one GPU's worth of cash vs A but erase the biggest technical risk and keep audio/LLM isolated.
- **C. Status quo minus deep tier:** just move the deep tier to cloud + retire memory-ai, keep the GPUs. Smallest
  change, no cash — only pursue if the cash/power isn't the driver.

**Recommendation:** if the driver is cash + simplicity and you accept the cloud boundary, **B is the sweet
spot** — nearly all the simplification and cash of A without the single-card contention gamble. Go to **A**
only if squeezing to one box matters more than audio/LLM isolation, and commit to the STT trim. Either way:
cloud failover escalation-only, NVMe Qdrant, memory-ai retired, and keep tool-retrieval on (more important
with a single model, not less).

## 7. Open decisions for Matt
1. **Cloud boundary:** retire all-local for the deep tier? (gates 3b)
2. **One GPU (A) or two (B)?** — risk posture.
3. **Front-door model** for the slim build: commit to 8B-class (Hermes/qwen3-8B) — the Hermes trial decision
   folds into this, since a single GPU with large STT can't also hold a 14B.
4. **Memory:** drop the hollow consolidated/pgvector tier, or migrate it? (recommend drop + rebuild later)
5. **STT trim** (distil/medium) — do the A/B before committing to single-GPU.

If a direction is chosen, this becomes an ADR + a migration runbook (mirroring HOST-CONSOLIDATION-RUNBOOK).
