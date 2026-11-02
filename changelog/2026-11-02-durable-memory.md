# 2026-11-02 — Durable memory (the first make-it-useful build)

## Executive summary
Iris was amnesic: the Loop-1 memory store was an in-RAM dict with token-overlap recall (lost on every
restart), and the elaborate pgvector "consolidation" tier meant to be the durable layer was hollow and
preference-corrupting (retired in the reliability pass). Built the deliberate, boring replacement: a
durable SQLite backend for `memory-substrate` that honors the existing `MemoryBackend` Protocol and the
`/write`+`/recall` HTTP contract, so the fusion/curator seams are unchanged. Additive (never supersedes),
semantic (Ollama nomic-embed + pure-Python cosine, token-overlap fallback), stdlib-only (sqlite3 +
urllib — no new deps, no sqlite-vec extension). Built + tested on bastion; **not yet deployed** (needs
Matt's deploy + a data dir on core-ai). This replaces the hollow stack rather than adding to it.

## Changes by repo
### iris-fusion-fabric (via wt-integration worktree) — branch `feat/durable-sqlite-memory` (commit ab634d5)
- `memory-substrate/src/memory_substrate/sqlite_store.py` — **added**: `SqliteVecBackend` — durable
  SQLite (WAL), embeddings as BLOB, cosine recall in pure Python with the same similarity+salience+recency
  ranking blend as the in-memory backend; `write` is `INSERT OR IGNORE` (additive, replay-safe); SQL
  filters for session/speaker/time + tier floor; embed via stdlib urllib → None-safe token fallback.
- `memory-substrate/src/memory_substrate/backend.py` — **changed**: `BACKEND` now selected by config —
  durable SQLite by default, in-memory for tests, with a logged fallback to in-memory if the store can't
  initialise (service must still start).
- `memory-substrate/src/memory_substrate/config.py` — **changed**: `backend` default `in_memory` →
  `sqlite`; added `sqlite_db_path` (default `/var/lib/iris-memory/memory.db`), `embed_url`, `embed_model`.
- `memory-substrate/tests/test_sqlite_store.py` — **added**: 6 tests — durability across reopen, additive
  no-overwrite, session filter, salience weighting, cosine path (fake embedder), write/recall. All green;
  existing `test_app.py` still green (no regression).

## Deploy / live state
- **Built + tested on bastion, NOT deployed.** Branch `feat/durable-sqlite-memory` (ab634d5), unpushed.
- Deploy is coupled to the reliability-pass teardown: of the renamed-aside units, **only
  `iris-memory-substrate-http` comes back** (rebuilt durable); `iris-memory-substrate` (DDS variant) and
  the consolidation machinery (dream-mode, reflection-coordinator, episode-buffer, chunk-store, dream-digest)
  stay dead.

## Follow-ups / owed — deploy (Matt)
Service runs from `/opt/iris-fusion-fabric/memory-substrate` as user `matt`, `.venv`, `-m memory_substrate.app`.
1. Review + merge `feat/durable-sqlite-memory` → the deployed branch; deploy to core-ai `/opt/iris-fusion-fabric` (git-bundle flow). No `uv sync` needed (stdlib-only change), though harmless.
2. `sudo mkdir -p /var/lib/iris-memory && sudo chown matt:matt /var/lib/iris-memory` (durable DB dir; service runs as matt).
3. `cd /etc/systemd/system && sudo mv iris-memory-substrate-http.service.disabled iris-memory-substrate-http.service`
4. `sudo systemctl daemon-reload && sudo systemctl start iris-memory-substrate-http`  (optionally `enable`).
5. Verify: `curl -s 127.0.0.1:8013/health` and `/count`; take a voice turn, then `/count` should climb and a later related turn should recall (watch curator `recalled=N>0`).
- Leave `iris-memory-substrate` (DDS) + the consolidation units renamed-aside. Push owed (bastion).
