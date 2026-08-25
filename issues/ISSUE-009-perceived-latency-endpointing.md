# ISSUE-009: Perceived latency is governed by VAD endpointing, not cortex first-audio
- Status: open (Matt's insight, 2026-08-25 — after retiring creative)
- Severity: medium (perceived quality — the recurring "feels slow/regressed" thread)
- Area: audio / conditioning (VAD endpointing)

## The insight
Retiring creative moved first-audio from ~0.2s (creative streaming) to ~2.5-3s (logical, no
stream) — yet Matt reports it **feels faster**. His read: the perceived delay is dominated by
**how many audio chunks are buffered before the system decides "this is the complete command"**
— i.e. VAD endpointing / silence-hangover — NOT by when Iris starts speaking.

Two mechanisms, both real:
1. **Coherence > fast-start.** Creative's fast start was often a wrong/confabulated first draft
   that got corrected/barged (a stutter). One clean grounded answer *reads* as faster than a
   fast-but-revised one. (Also why the ISSUE-005 fix "feels" better.)
2. **Endpointing is a fixed per-turn delay BEFORE cognition.** The silence-hangover that finalizes
   a burst adds the same wait to every turn, ahead of the cortex. It was masking the cortex
   first-audio all along — which is why 0.2s→2.5s didn't hurt. Lines up with the earlier finding
   (VAD mic-open hangover; ~5s bursts for ~2s of speech — see `iris-regression-diagnosis-2026-08`).

## Why it matters
Tuning the endpointing (fewer silence-chunks / shorter hangover before "command complete") is a
**per-turn** perceived-latency win on EVERY turn — likely bigger than adding cortex streaming.
The tension: too aggressive cuts Matt off mid-sentence; too conservative adds the felt lag. There's
a sweet spot to find.

## To investigate / do
- **Measure it:** from asr/vad-consumer logs, the delay from last-speech to burst-finalized
  (the hangover), and the burst-length vs actual-speech ratio. Confirm it's the dominant term.
- Find the endpointing knobs (VAD silence-duration / min-silence-chunks / MIN_BURST / hangover in
  `iris-conditioning` asr-consumer) and tune down, watching for mid-sentence cutoffs.
- Relates to [ISSUE-001] (speaker-gate recall) and the perceived-latency notes in
  `iris-regression-diagnosis-2026-08` / `iris-latency-and-gate-2026-07`.

## Note
This reframes "cortex streaming" as a *smaller* lever than assumed — endpointing first.
