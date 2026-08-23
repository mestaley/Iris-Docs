# ISSUE-001: Speaker gate threshold (0.35) cuts into the genuine speaker → first-try silence
- Status: FIXED + DEPLOYED LIVE 2026-08-23 — `SPEAKER_THRESHOLD` 0.35→0.28 on audio-ai asr-consumer (iris-conditioning commit b086e59); container recreated, `loaded voiceprints: ['matt'] (threshold=0.28)`, all 3 nodes reconnected. Re-measure recall after soak.
- Severity: high
- Area: audio/recall
- Discovered: 2026-08-22
- Evidence: `intent-consumer` env `SPEAKER_THRESHOLD=0.35` (live); `speaker.py:44-45`; 13-day intent-consumer log sweep

## Symptom
Matt: "she used to answer almost every time, now frequently silent the first time; I repeat and it works." Worst in single-mic rooms (bedroom vg-03).

## Root cause
The mic→reply chain is **series (logical AND)**: VAD → burst-length → ASR → address-gate (G4) → **speaker-gate (G5)** → router. Nothing downstream rescues a turn G5 drops.

- G5 rejects a genuine address unless ECAPA cosine vs enrolled "matt" ≥ `SPEAKER_THRESHOLD` (0.35).
- Live cosine for genuine Matt **straddles the cutoff** (histogram mass at 0.30–0.50). The 0.35 floor cuts into his own distribution; VoxCeleb EER is ~0.25, so 0.35 is set conservatively high.
- Compounded recall: early Iris (no G4/G5) ≈ 96% first-try (~4% miss). Now ≈ 78% (~22% miss). The gap is the two precision gates multiplied in series (G4×G5 ≈ 0.90×0.90).

## Live evidence
- Of turns that passed the address gate, **35/157 (22%)** were then dropped by the speaker gate (13-day log).
- Smoking gun (2026-08-08 14:25:51): genuine "turn the kitchen lights on" heard by two mics — vg-01 cos=0.474 **accepted**, vg-02 cos=0.344 **rejected** (0.006 under the line), same utterance. Multi-mic rooms mask the loss; single-mic rooms eat it.

## Nuance — G5 is not fully redundant
33 of 35 rejects were a virus/vaccine **TV documentary** hitting "virus" (a legit wake-alias) that the address gate can't stop. Deleting G5 re-admits that. Keep the gate; loosen it.

## Fix direction (cheapest, TV-safe)
1. Lower `SPEAKER_THRESHOLD` 0.35 → ~0.25–0.30 (env on asr-consumer/intent-consumer, no redeploy). Recovers the 0.30–0.35 genuine band; TV bursts scored well below 0.30.
2. Keep `GATE_TRUSTS_WINDOW=true` — the only thing keeping follow-ups/await-answers alive.
3. Do NOT delete G5. Do consider deleting the duplicate standalone `vad_consumer.py` (asr-consumer runs its own Silero VAD) — that removes complexity without touching recall or TV precision.
