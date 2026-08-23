# Changelog

One file per Claude session that touches any Iris repo. Git commit messages are the ground
truth, but they're scattered across many repos (iris-fusion-fabric, iris-conditioning, world-ai,
iris-volition, memory-substrate, iris-docs, …) and often unpushed on feature branches. This
directory is the **cross-repo digest**: one place to see what a session changed everywhere.

## When to write one
At the end of any session that changed code, config, docs, or live deployment state in any repo.
Write it before wrapping up. If a session only answered questions and touched nothing, skip it.

## Naming
`YYYY-MM-DD-<short-slug>.md` — e.g. `2026-08-23-iris-review-and-docs-consolidation.md`.
Multiple sessions in one day: add a distinguishing slug (or `-2`). Newest sorts last by date;
that's fine — the entries are self-dating.

## Format
```markdown
# YYYY-MM-DD — <session title>

## Executive summary
2-5 sentences: what the session set out to do and what actually changed. Plain language,
the version you'd tell Matt in the hallway. Note anything DEPLOYED LIVE vs staged vs reverted.

## Changes by repo
### <repo> (<branch>)
- `path/to/file` — **added / changed / deleted** — one line on what and why. (commit `<hash>`)
- ...

## Deploy / live state
What went live where, what's staged pending a restart/sudo, what was reverted.

## Follow-ups / owed
Owed origin pushes, banked tasks, decisions needed. Cross-link iris-docs issues/ADRs.
```

## Rules
- **Per-file granularity** (added/changed/deleted), grouped by repo+branch — that's the whole
  point; don't just restate the commit subjects.
- Cite the commit hash so the changelog stays anchored to git.
- Record DEPLOY state and REVERTS honestly — a reverted change still belongs in the log.
- Don't duplicate the issue tracker; link to `issues/ISSUE-xxx` and `decisions/ADR-xxx` instead.
- This is a digest, not a diff — one line per file, not the code.
