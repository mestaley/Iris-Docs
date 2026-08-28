# 2026-08-28 — Iris voice: strip reflexive openers ("Right then,")

## Executive summary
Matt: Iris "leads with 'Right' wayyy too much" — Jarvis just answers. Added a leading-opener
strip to `denag` (the shared post-filter that already peels service-desk closers), so every voiced
say drops a reflexive opener ("Right then," / "Well," / "So," / "Okay,") and answers directly.
Post-filter, not prompt (the deep model ignored a prompt ban). Punctuation-gated so "Right away,"
and opener-only utterances are preserved. STAGED to core-ai venvs — needs restart of
iris-fusion-executive + iris-deep-mode (cortex-logical optional; fusion re-denags the final say).

## Changes by repo — iris-fusion-fabric (contracts shared across branches)
- `contracts/src/iris_fusion_contracts/denag.py` — new `strip_leading_opener()` + called inside
  `denag()`. Committed on `feat/web-tasks-tools`, `feat/punt-escalation`, `feat/deep-mode` (shared file).
- `contracts/tests/test_denag_openers.py` — added (strips openers, preserves meaningful leads +
  opener-only, composes with closer strip). Passes in every service venv.
- `deep-mode/src/deep_mode/service.py` (`feat/deep-mode`) — `groom_for_speech` also calls
  `strip_leading_opener`.

## Deploy / live state
- denag.py + branch-correct `__init__.py` deployed into the fusion-executive / cortex-logical /
  deep-mode venvs (contracts is a per-venv COPY, not editable). Suites green (fusion 40, deep-mode 10).
- **Owed (Matt sudo):** `sudo systemctl restart iris-fusion-executive iris-deep-mode`.
- Gotcha logged: contracts `__init__.py` differs per branch (PuntRequest export) — never blanket-copy
  it across venvs; only denag.py is safe to share. (Briefly broke it mid-change; repaired.)

## Follow-ups / owed
- **Decision for Matt:** how Iris addresses him — "Mr Staley" (Jarvis-style) vs just "Matt"
  (current). Not baked in pending his call.
- Origin pushes still owed (this change rides the three feature branches).
