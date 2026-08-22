# Iris — Architecture

_Stub. To be filled from the pipeline/dead-code map (investigation agent 4, 2026-08-22)._

## Intended contents
- Full service inventory (one line each).
- The per-turn **hot path** sequence: FeatureStream → thalamus → curator →
  cortex-logical / cortex-creative → fusion-executive → tool-registry → world-ai/HA →
  speech-output, marking sync-blocking vs async-streaming hops and latency per hop.
- Dead / vestigial paths with confidence (confirmed-dead / probably-dead / needs-runtime-check).
- "Diagnosis surface": for a symptom (no-response, wrong-action), how many services could
  independently cause it.
- Then-vs-now contrast with the early synchronous `fabric_brain` request/reply path.

See STATE.md for the live subsystem table and issues/ for the specific defects.
