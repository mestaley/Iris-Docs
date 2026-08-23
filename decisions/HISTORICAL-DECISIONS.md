# Historical Decisions (ported from the legacy log)

_Ported into iris-docs 2026-08-23 from the earlier `Architecture/DECISIONS.md` (981 lines, ~45
decisions), triaged against the live async-streaming fabric + ADR-001. This one file holds the
**still-in-force** decisions worth keeping; NEW decisions get their own `ADR-NNN-slug.md`
(see README). Section B lists decisions the live system has since CONTRADICTED — kept as a
warning, not as guidance. Everything else in the legacy log was done, superseded, or belongs to
the reflection/consolidation plane that ADR-001 hard-gated (see Section C)._

---

## A. Still in force

### HD-1 — world-ai is the integration broker / anti-corruption boundary (2026-04-30, 2026-05-26)
- **Context:** Iris's reasoning layers must not know external-API shapes (HA REST, Met.no, Google, RSS, Spotify).
- **Decision:** All external integration (read + write) lives on world-ai/Meridian, which translates provider vocabulary into Iris vocabulary at the boundary; tool-registry stays pure transport. The cortex never sees `cache_age_seconds`, compass-degree wind, Google event resources, etc.
- **Consequence:** Provider churn is absorbed in one place; `HA_TOKEN` lives once; the cortex prompt stays shape-agnostic. Still the live topology (`tool-registry → world-ai/meridian → HA`). See [[PATTERNS]] "Proxy through world-ai."

### HD-2 — Reflex/cognition split: fast tool-calling on world-ai, cortex as supervisor (2026-05-30, Phase 2.5)
- **Context:** A cortex doing both cognition and actuation forced one wrong latency profile and root-caused the "articulate lie."
- **Decision:** A fast model on world-ai (qwen2.5:7b) handles direct actuation via curator's fast-path; the cortex reasons about outcomes and keeps authority to correct. Cortex-logical keeps native tool-calling for the ambient/reasoning path.
- **Consequence:** LIVE and load-bearing (curator FAST-PATH → world-ai `/fastpath/decide`). **Open caveat:** the grounding half — cortex speech not grounded in `tool_results` — is [ISSUE-005](../issues/ISSUE-005-cortex-blind-to-tool-results.md); multi-intent is [ISSUE-002](../issues/ISSUE-002-multi-intent-collapse.md).

### HD-3 — Structured cortex→fusion signals over surface-text inspection (2026-05-26/27; reverses the 2026-05-25 "?" heuristic)
- **Context:** Fusion inspecting `"?" in proposed_say` to suppress actuation caught rhetorical wit and silently dropped tool calls.
- **Decision:** Creative sets an explicit `needs_clarification: bool`; fusion suppresses actuation only on that flag. Generalize: cortex→fusion signals (uncertainty, salience, tone) are structured fields, never inferred from prose.
- **Consequence:** LIVE. Reliable clarify-vs-act. Explicitly reversed the "?" heuristic. See [[PATTERNS]] "Clarifying questions block actuation."

### HD-4 — Curator prefetch dual-locus: emit AND gate v0 together (2026-05-26)
- **Context:** A prefetch flag that emits a tool call but isn't added to the v0 gating disjunction lets cortices fire on empty ambient before the prefetch lands → hallucination.
- **Decision:** Every prefetch flag moves in lockstep across three sites: `thalamus_flag ↔ curator_emit_call ↔ curator_gate_v0`. The v0/v1 curated-context staging is the enforcement point.
- **Consequence:** LIVE, non-obvious, a real land-mine for anyone adding a prefetch. _(Directly relevant to ISSUE-002's redo: any dual-rail change must respect this.)_

### HD-5 — Prompt discipline: structural prevention + abstract examples over prompt-force (2026-05-22, 2026-05-26)
- **Context:** Telling a model "don't do X" three times fails; concrete in-prompt examples get recited verbatim as data (gemma emitted "33°F snowing" on a 77°F day).
- **Decision:** Make X structurally impossible (curator removes already-satisfied tools from the catalog; identity anchor injected, not requested). Example inputs use abstract bracketed conditions, never recitable concrete values.
- **Consequence:** Durable doctrine for every prompt (cortex, thalamus, intent). See [[PATTERNS]] "Filter at the structural layer" and "Hard prompt rules." _(The 2026-08-23 multi-intent over-prefetch miss (ISSUE-002) is a fresh instance of the "prompt-force backfires on a small model" lesson.)_

### HD-6 — Eval is a peer participant on the production bus; reliability harness for nondeterministic actuation (2026-05-26, 2026-05-28)
- **Context:** A single-shot eval pass hid a ~45% partial-failure rate on compound actuation.
- **Decision:** The eval harness is a plain DDS participant (real turns, real tools, no mock seam); actuation fixes are gated by a reliability harness running N=20 at production temperature with a ≥19/20 bar.
- **Consequence:** Directly relevant NOW — ISSUE-002 says multi-intent + field-tuning must be redone against an eval harness. `iris-fusion-fabric/eval/` exists; rebuild cases against this standard.

### HD-7 — Per-node stateful model instances, never shared across node workers (2026-05-25)
- **Context:** One shared Silero VAD across per-node workers corrupted its RNN state → 100GB error-log disk-fill.
- **Decision:** Each per-node worker owns its own instance of any stateful model; model *files* shared via bind-mounted cache, model *objects* isolated. A `Lock` is not a fix (prevents races, not state interleaving).
- **Consequence:** Durable anti-pattern. See [[PATTERNS]]. _(ADR-001 separately flags a leftover duplicate `vad_consumer.py` to prune — the pattern stands.)_

### HD-8 — Jarvis is the explicit target (persistent household intelligence, not a voice assistant) (2026-05-25)
- **Decision:** Iris targets a years-accumulating household presence across surfaces (voice now, desk web-chat later), justifying the memory/pluralism/hardware roadmap.
- **Consequence:** Governing vision. See [[PRINCIPLES]] and [[THESIS]].

### HD-9 — Daydream-driven design with real-stakes validation targets (2026-05-25)
- **Decision:** Architectural completeness is tested against concrete real-stakes cases (custody-agreement processing is canonical) — validation targets, not roadmap items with dates.
- **Consequence:** Methodology anchor; keeps the architecture serious without over-committing. See THESIS validation targets.

### HD-10 — Architecture is sovereign: design around not-yet-built components (2026-05-24)
- **Decision:** Commit to contracts/interfaces for components that don't exist yet; implementation conforms to architecture.
- **Consequence:** In force **with a noted tension** — ADR-001 bounds it: don't keep dead code wired into live paths; the diagnosis surface is the cost. Sovereignty for *interfaces*, prune for *dead wiring*.

### HD-11 — Two-track long-term memory: episodic (conventional) + semantic (substrate-resident) (2026-05-24) — SUPERSEDED-IN-PRACTICE
- **Decision (as written):** Track 1 episodic = discrete consolidated chunks with match-and-reinforce dedup; Track 2 semantic = substrate-resident patterns the cortex thinks *with*; a neutral store decouples data accumulation from substrate experiments.
- **Status:** This is the long-term-memory ambition ADR-001 deliberately hard-gates (keeps the code, stops the writer). The semantic tier is currently the hollow, unread dead-end of [ISSUE-003](../issues/ISSUE-003-consolidation-dead-end.md). Kept here as the **preserved roadmap record**, NOT a description of live behavior.

Lower-priority still-valid: a "cognitive event stream separate from operational logs / logging-is-a-precondition" principle (2026-05-24) — partially live (Loki + turn-tracing); see [[PRINCIPLES]].

---

## B. Contradicted by the live system — do NOT treat as current

These describe behavior the system no longer has; porting them wholesale would mislead.

1. **"Reflection precedes memory write; short-term memory holds only reflected episodes" (2026-05-24).** Live writes episodic memory INLINE from fusion (`FUSION_DECISION → MEMORY_WRITE`), not gated on reflection. The reflection/consolidation plane this assumes is exactly what ADR-001 hard-gated (ISSUE-003).
2. **"When fabric and brain-era guidance disagree, brain wins" (2026-05-26).** The `fabric_brain` bridge is dormant/vestigial and slated to prune (ISSUE-004). The fabric prompts evolved far past the `/opt/iris-brain` baseline.
3. **"Clarifying '?' in say blocks actuation" (2026-05-25).** REVERSED by HD-3's structured `needs_clarification`.
4. **The reflection-plane cluster read as "in production"** (sequential dual-cortex, both-chain-orderings, creative_first chain, two-phase coordinator timer, reflection deferral, parse_failed, eval-traffic filter). Gated OFF. Roadmap-preserved, not live.
5. **Hardware "2× RTX 5080 for cortex" (2026-05-30).** The real path was 3060 Ti → 5060 Ti-16GB swaps (per memory). Don't port as current topology.

---

## C. Everything else (done / superseded / gated) — not ported

Bulk-discardable from the legacy log: reflection-pipeline mechanics (now gated), done operational
cleanups (Music Assistant install, Spotify creds removal, Docker log rotation, uv workspace-dep
declaration, Cranberry-vs-Gibsonia fact, GROUND-TRUTH prompt scoping, Meridian calendar TZ window,
dream-digest email decision, skip-pass-2 optimization), and process retrospectives. The original
`Architecture/DECISIONS.md` retains the full history if ever needed.
