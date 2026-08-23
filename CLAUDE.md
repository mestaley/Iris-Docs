# Iris workspace — operating instructions

`/opt/repos` is a container of many Iris repos (iris-fusion-fabric, iris-conditioning, world-ai,
iris-volition, memory-substrate, iris-docs, …), not itself a git repo. This file is the
workspace-level guide; it is symlinked from `/opt/repos/CLAUDE.md` so it loads every session.

## Single source of truth
**`/opt/repos/iris-docs` is the SSOT.** Open `iris-docs/STATE.md` FIRST each session — it's the
live dashboard (hosts, running services, current focus). Then, as relevant: `THESIS.md` (intent),
`PRINCIPLES.md`, `ARCHITECTURE.md`, `PATTERNS.md`, `issues/`, `decisions/`. iris-docs wins over
auto-memory when they disagree; correct the memory to match.

## End-of-session changelog (REQUIRED)
At the end of any session that changed code, config, docs, or live deploy state in ANY repo,
write a changelog entry: **`iris-docs/changelog/YYYY-MM-DD-<slug>.md`**. This exists because git
messages are scattered across many repos and often unpushed on feature branches — the changelog
is the one cross-repo digest.

Follow `iris-docs/changelog/README.md`. Each entry has:
1. **Executive summary** — 2-5 plain sentences: what the session did, what's LIVE vs staged vs reverted.
2. **Changes by repo** — grouped by repo+branch, **one line per file** (added / changed / deleted, what + why, commit hash). Per-file granularity is the point — don't just restate commit subjects.
3. **Deploy / live state** — what went live where, what's staged pending sudo/restart, what was reverted.
4. **Follow-ups / owed** — owed origin pushes, banked tasks, decisions needed; link issues/ADRs.

Write it before wrapping up. A session that only answered questions and touched nothing needs no entry.

## Cross-repo / deploy notes (see the memory files for detail)
- core-ai = fabric (systemd `iris-*.service`, interactive sudo). audio-ai = conditioning (docker,
  sudoless). world-ai = sensors/Meridian/HA proxy (systemd). voice-ai = STT/TTS+GPU.
- This automation session can SSH the hosts (read + sudoless docker on audio-ai) but **cannot push
  to origin** and **cannot interactive-sudo restart** core-ai/world-ai services — those need Matt.
  Track owed pushes and staged restarts in the changelog + STATE.md.
- Commit messages: no Co-Authored-By / AI-attribution trailers (project rule).
