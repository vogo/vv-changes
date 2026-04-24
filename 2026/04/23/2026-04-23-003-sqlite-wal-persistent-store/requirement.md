# Requirement — P1-6 · SQLite + WAL persistent memory store

## 1. Background & objective

The `vv` agent application persists cross-session knowledge (project conventions, user preferences, skill outputs, agent-written notes) through a `memory.Store` backed by `vv/memories/FileStore` — one JSON file per entry under `~/.vv/memory/`. That works for a handful of entries and a single writer. It does not:
- tolerate concurrent writers without torn `read-modify-write`,
- answer prefix / namespace queries without walking the whole tree,
- provide a handle for the full-text search (P2-3), historical cost log (P2-2), session checkpoint (P2-14), or vector recall (P3-10) features queued behind P1-6.

**Objective:** ship a `SQLiteStore` that satisfies the same `memory.Store` interface, preserves the session-binding semantics FileStore landed in P1-4, and lays the foundation (single DB file, schema version, WAL mode) for the downstream search/recall work to attach to without another rewrite.

## 2. Scope

### In-scope

1. A new `vv/memories/sqlitestore.go` implementing `memory.Store` with the **exact same externally observable behavior** as `FileStore`, including session isolation, legacy fallback, and TTL-on-read.
2. A pure-Go SQLite driver dependency — `modernc.org/sqlite` (so `vv` continues to build without CGO).
3. WAL-mode initialization at open: `journal_mode=WAL`, `synchronous=NORMAL`, `busy_timeout=5000`, `foreign_keys=ON` (defensive, unused today), `PRAGMA user_version` tracked in Go code.
4. A single `entries` table with compound primary key `(namespace, name, session_id)`; indexes on `(namespace)`, `(session_id)`, `(updated_at)`.
5. Backend selector `cfg.Memory.Backend` (`"file"` default, `"sqlite"` opt-in) wired into `setup.Init`; `"file"` remains the default so existing users see no behaviour change.
6. Unit tests that mirror `filestore_test.go` 1-to-1 using the same assertions (same test names, just running against `SQLiteStore`).
7. Config validation: reject unknown backends with a clear error at load time.

### Explicitly out-of-scope (documented as follow-ups)

- **Auto-migration from FileStore → SQLiteStore.** Surveyed projects avoid it; users keep FileStore until they opt in, and a dedicated `vv memory migrate` command is left for a future session.
- **FTS5 / vector-search tables.** P2-3 / P3-10 add their own tables layered on the same DB file. Keeping this PR minimal avoids schema churn.
- **Active TTL sweeper.** Lazy-on-read is retained for parity with FileStore.
- **Backup / vacuum automation.** `VACUUM` is not called here; operator concern.
- **Changing the default backend** to SQLite. That decision is deferred until M4 when downstream features land.
- **Schema migration mechanism beyond `PRAGMA user_version`.** We store `user_version = 1` this round; future schema additions are free to bump to 2+ with whatever idempotent migration they like.

## 3. User stories & acceptance criteria

> Personas: **Operator** (deploys vv), **Developer integrator** (builds on top of vv's store), **Agent user** (CLI/HTTP/MCP end-user who doesn't pick a backend).

### US-1 · Opt into the SQLite backend

**As** an operator, **I want** to set `memory.backend: sqlite` in `~/.vv/vv.yaml` (or `VV_MEMORY_BACKEND=sqlite` env) **so that** persistent memory is stored in one SQLite DB file with WAL instead of many JSON files.

**AC-1.1** — With `memory.backend: sqlite` and an empty `cfg.Memory.Dir`, startup creates `<dir>/memory.db` + `<dir>/memory.db-wal` + `<dir>/memory.db-shm`, all mode `0o600`, and the parent dir at `0o700`.
**AC-1.2** — `PRAGMA journal_mode` returns `wal`, `PRAGMA synchronous` returns `1` (NORMAL), `PRAGMA user_version` returns `1`, `PRAGMA busy_timeout` returns `5000`.
**AC-1.3** — With `memory.backend` unset or `file`, no DB file is created; the legacy FileStore path is unchanged (verified by the existing FileStore tests still passing).
**AC-1.4** — `memory.backend: <garbage>` is rejected at config-load time with `unknown memory backend "<garbage>" (expected "file" or "sqlite")`.

### US-2 · Interface parity with FileStore

**As** a developer integrating vv, **I want** `SQLiteStore` to satisfy `memory.Store` with the same behaviour as `FileStore` **so that** switching backends is a config flip, not a code change.

**AC-2.1** — `SQLiteStore` passes every assertion currently covered by `filestore_test.go` (full test parity including session isolation, legacy fallback, user-path write-to-private refusal, `Clear` user-path-only, `WithSharedNamespaces`).
**AC-2.2** — `BatchGet`/`BatchSet` are **not** required (FileStore doesn't implement `BatchStore` either); nothing in the memory manager relies on them today.
**AC-2.3** — `Get` on a TTL-expired entry returns `found=false` and physically removes the row (same lazy behaviour as FileStore — verified by a second `List` not returning the key).

### US-3 · Session-binding defense-in-depth

**As** an agent user, **I need** session-B writes to never overwrite or leak session-A's private entries, even if the DB file is tampered with or contains legacy records.

**AC-3.1** — A record owned by session A (`session_id='session-A'`) cannot be overwritten by a write from session B; the attempt returns `ErrSessionForbidden`.
**AC-3.2** — A legacy row (`session_id=''`) in a private namespace is readable from any session but **not** overwritable from any session — same as FileStore's legacy-slot guard.
**AC-3.3** — `List` from session B never returns session A's private rows; `List` from a user-path call never returns any private-namespace rows (only shared).
**AC-3.4** — `Delete` from session B on a key that exists only for session A is a silent no-op (returns `nil`), and session A's row is intact afterwards.

### US-4 · No surprises for existing users

**As** an agent user who has been running vv with the default `FileStore`, **I want** upgrading to the new version to change nothing about my data or my workflow unless I deliberately opt in.

**AC-4.1** — All existing CLI / HTTP / MCP integration tests that pass today continue to pass after this change with `memory.backend` unset.
**AC-4.2** — No automatic import of FileStore data into SQLite is performed; switching from `file` to `sqlite` starts with an empty store (documented behaviour).
**AC-4.3** — The new dependency on `modernc.org/sqlite` keeps `CGO_ENABLED=0` builds working (no CGO transitively).

### US-5 · Concurrent tolerance under WAL

**As** an operator running the CLI and a background `-mode mcp` process against the same memory dir, **I want** SQLite WAL mode to let both read/write without spurious `SQLITE_BUSY` errors.

**AC-5.1** — With `busy_timeout=5000`, a contention test (two goroutines each doing 100 concurrent writes) completes with zero `SQLITE_BUSY` errors and zero lost updates (final row count == 200 unique keys).

## 4. Success criteria (verifiable)

- `cd vv && make test` passes with `memory.backend: sqlite` wired through integration tests.
- `cd vv && make lint` passes.
- `cd vv && CGO_ENABLED=0 go build ./...` succeeds.
- All new tests are in `vv/memories/sqlitestore_test.go` and mirror `filestore_test.go` names.
- No regression in any of `vv/integrations/**` test runs.
- `doc/prd/feature-todo.md` row P1-6 is moved to `doc/prd/feature-implement.md` with the session directory path referenced.

## 5. Assumptions surfaced (no user answer sought — autonomous run)

| # | Decision | Rationale |
|---|----------|-----------|
| A1 | Default backend stays `file` | This PR lands SQLite as opt-in; flipping default is a separate decision bundled with M4's search features. Avoids first-run surprises. |
| A2 | No auto-migration of FileStore → SQLite | Industry norm; silent import risks data loss. A dedicated migrator is a future small task. |
| A3 | Single DB file at `<cfg.Memory.Dir>/memory.db` | Reuses existing config; no new path knob. |
| A4 | `modernc.org/sqlite` (pure Go) over `mattn/go-sqlite3` (CGO) | Keeps the repo CGO-free, easier release. Pure-Go driver is production-grade as of 2026. |
| A5 | `PRAGMA user_version=1` tracks schema version | Simplest migration anchor; future tables (FTS5, vector) bump it. |
| A6 | Lazy-on-read TTL, not a background sweeper | Parity with FileStore; cheaper; the working set fits in memory for any realistic vv install. |
| A7 | `Clear()` is `DELETE FROM entries` (no VACUUM) | VACUUM stalls the DB; operator can run it externally if disk matters. |

## 6. Inconsistencies noted

- `doc/prd/overview.md` line 75 says *"trimming old files is the operator's responsibility until P1-6 SQLite replaces the JSONL store"* — but P1-6 as written covers only persistent memory, not trace JSONL. The trace JSONL store is a separate output (P1-5 already landed as JSONL). Flag for the documenter phase: overview should not imply SQLite replaces the JSONL trace; it replaces the persistent-memory JSON-per-file store.
- `doc/prd/overview.md` line 80 mentions *"Restart persistence of the daily budget counter (P1-3 is in-memory only; ... SQLite-backed persistence is deferred to P1-6)"*. P1-6 as scoped here does NOT persist the daily budget counter; that's a follow-on task riding the same DB. Flag for the documenter phase.
