# Design — P1-6 · SQLite + WAL persistent memory store

## 1. Approach

Add `vv/memories/SQLiteStore`, a second implementation of `memory.Store` that lives beside `FileStore`. A config flag `memory.backend` (default `"file"`) selects which one `setup.Init` wires into the persistent memory. Nothing in the memory manager above the `Store` boundary needs to change.

Design priorities (from the requirement's "simplicity first"):
- **One file, one table, one driver.** Scope creep on day 1 (FTS5 tables, vector columns, background sweeper) is explicitly deferred — those will ride this DB in later PRs.
- **Mirror FileStore's semantics exactly at the Go API level.** Tests are copy-modified so parity regressions show up immediately.
- **Fail fast on open.** A probe statement runs during `NewSQLiteStore` — the surveyed bug class where SQLite reports open errors only on first query is avoided.

## 2. Dependency

Add `modernc.org/sqlite` to `vv/go.mod` — pure-Go SQLite driver, production-grade, used by Bluesky / Fossil / SourceHut as of 2026. Keeps `CGO_ENABLED=0` builds working (a stated constraint — all existing `vv` targets are CGO-free).

Registered driver name: `"sqlite"`.

## 3. Schema (v1)

Single table, WITHOUT ROWID for tighter row layout since the compound PK is always present:

```sql
CREATE TABLE IF NOT EXISTS entries (
    namespace  TEXT    NOT NULL,
    name       TEXT    NOT NULL,
    session_id TEXT    NOT NULL DEFAULT '',
    key        TEXT    NOT NULL,
    value      TEXT    NOT NULL,
    created_at INTEGER NOT NULL,  -- unix nanos (time.UnixNano)
    updated_at INTEGER NOT NULL,  -- unix nanos
    ttl        INTEGER NOT NULL DEFAULT 0,  -- seconds; 0 = no expiry
    PRIMARY KEY (namespace, name, session_id)
) WITHOUT ROWID;
```

- The PK gives us fast point lookups (`Get`, `Delete`) and prefix scans (`List` with `namespace`-prefixed keys walk PK order).
- `key` stores the original caller-supplied key so `List` can roundtrip it verbatim (covering the `"default:foo"` vs `"foo"` collision edge case without reconstruction drift).
- `session_id = ''` marks shared records **and** legacy records in a namespace that is not in the shared allowlist; the Go-layer namespace allowlist disambiguates the two at read time, identical to FileStore.
- `created_at` / `updated_at` as INTEGER nanos: smaller, faster, trivial `time.Unix(0, n)` roundtrip. SQLite CLI debugging loses the pretty datetime but gains consistency.

No secondary indexes in v1 — the PK index handles every query path we need. FTS5 (P2-3) will add its own virtual table.

`PRAGMA user_version = 1` stamps the schema version. Future migrations bump and branch.

## 4. Connection setup

Open the DB with PRAGMAs via DSN (modernc.org/sqlite parses `_pragma` params):

```
file:<path>?_pragma=journal_mode(wal)&_pragma=synchronous(normal)&_pragma=busy_timeout(5000)&_pragma=foreign_keys(on)
```

- `journal_mode=WAL` — multi-reader / single-writer without blocking readers.
- `synchronous=NORMAL` — fsync on checkpoint only, safe with WAL for persistent-memory durability (losing the last uncheckpointed write on power loss is acceptable for this tier — consistent with FileStore which isn't fsynced either).
- `busy_timeout=5000` — retries SQLITE_BUSY up to 5 s before erroring. Applied at connection-open via DSN so every pooled conn inherits it.
- `foreign_keys=ON` — defensive; no FKs today, future-proof.

Connection pool: `db.SetMaxOpenConns(8)` + `db.SetMaxIdleConns(4)`. Writers queue through `busy_timeout`; readers proceed concurrently under WAL.

After the DB is created by the first open, chmod the DB, `-wal`, and `-shm` files to `0o600`. The parent dir is already `0o700`-created by the caller (same as FileStore).

**Fail-fast probe:** after `sql.Open`, the constructor runs `db.Ping()` → `ensureSchema()` → a small probe `SELECT 1 FROM entries LIMIT 0`. If any step errors, return without exposing a half-initialized store.

## 5. API

```go
// Option matches FileStore's option type.
type Option func(*SQLiteStore)

// WithSharedNamespaces — identical signature / semantics to FileStore's.
func WithSharedNamespaces(names ...string) Option

// NewSQLiteStore opens (or creates) the DB at <dir>/memory.db with WAL mode,
// applies the schema if missing, and returns a store ready for use.
func NewSQLiteStore(dir string, opts ...Option) (*SQLiteStore, error)

// Close closes the underlying *sql.DB. Safe to call multiple times.
func (s *SQLiteStore) Close() error
```

Implements `memory.Store` (Get / Set / Delete / List / Clear) with the session-binding semantics defined in `WithSessionID` / `WithUserPath` / `isShared` from the existing `memories` package. Those helpers are package-scope and reused as-is — SQLiteStore is in the same Go package.

**Context usage:** every method calls the `ctx`-aware variants (`ExecContext`, `QueryRowContext`, `QueryContext`) so caller cancellation propagates to in-flight queries.

## 6. Method-by-method mapping

### 6.1 Get

```sql
SELECT value, session_id, created_at, updated_at, ttl
FROM entries
WHERE namespace = ? AND name = ? AND session_id = ?
LIMIT 1;
```

Path:
1. Parse key → `(ns, name)`. Check `shared = isShared(ns, extra)`.
2. If `shared` → query with `session_id = ''`.
3. Else (private namespace):
   a. Query with `session_id = sid`. If found and (row `session_id == sid`) → return value.
   b. Else fall back to `session_id = ''` (legacy). Same visibility rules as FileStore: a legacy row in a private namespace is readable from any session.
4. Apply TTL check: `if ttl>0 && now - updated_at > ttl*1e9` → `DELETE` the row, return `(nil, false, nil)`.

### 6.2 Set

```sql
INSERT INTO entries (namespace, name, session_id, key, value, created_at, updated_at, ttl)
VALUES (?, ?, ?, ?, ?, ?, ?, ?)
ON CONFLICT(namespace, name, session_id) DO UPDATE SET
    value      = excluded.value,
    key        = excluded.key,
    updated_at = excluded.updated_at,
    ttl        = excluded.ttl;
-- created_at NOT updated → preserved by the DO UPDATE clause omitting it.
```

Guards (before UPSERT, in Go):
1. If `!shared && isUser` → `ErrSessionForbidden`.
2. If `!shared && sid == ""` (no user-path, no session) → `ErrSessionForbidden`.
3. If `!shared`:
   - Check `SELECT 1 FROM entries WHERE namespace=? AND name=? AND session_id='' LIMIT 1` — if a legacy row exists, reject with `ErrSessionForbidden`. (Mirrors FileStore's "private namespace, legacy entry cannot be overwritten" guard.)

The per-row cross-session tampering test that FileStore exercises (`TestFileStore_AgentPath_DefenseInDepth_OverwriteBlocked`) is **structurally unreachable** in SQL — PK `(namespace, name, session_id)` physically separates session-A and session-B slots. The SQLiteStore test suite replaces that case with `TestSQLiteStore_AgentPath_StructuralIsolation` documenting and asserting the stronger invariant.

### 6.3 Delete

Path:
1. Parse key, determine shared/private.
2. If `!shared && isUser` → `ErrSessionForbidden`.
3. If `!shared && sid == ""` → `ErrSessionForbidden`.
4. If `shared`:
   - `DELETE FROM entries WHERE namespace=? AND name=? AND session_id=''`. Swallow not-found.
5. Else (private):
   - Load the row via `(ns, name, sid)`; if missing, fall back to legacy `(ns, name, '')` lookup.
   - If loaded row's `session_id != ''` and `session_id != sid` → `ErrSessionForbidden`. (Cannot happen via API with PK schema but kept for defense-in-depth / test parity.)
   - `DELETE FROM entries WHERE namespace=? AND name=? AND session_id=?` with the resolved PK.

### 6.4 List

```sql
-- Agent path (session sid, not user-path):
SELECT key, value, created_at, ttl, session_id, namespace, updated_at
FROM entries
WHERE (session_id = '' OR session_id = ?)
ORDER BY key;

-- User path:
SELECT key, value, created_at, ttl, session_id, namespace, updated_at
FROM entries
WHERE session_id = ''
ORDER BY key;
```

Go-side filtering for visibility + TTL:
1. For each row, reconstruct visibility rule identical to `readSharedDir` / `readPrivateDir` in FileStore:
   - **Agent path:** if `session_id == sid` → always visible (private match). If `session_id == ''`: visible when ns is in the shared allowlist **or** when it's a legacy row in a private namespace (the FileStore "legacy shared" case).
   - **User path:** only `session_id == ''` rows are fetched; among those, show shared namespaces and legacy-private rows (matching FileStore's current behavior).
2. Apply lazy TTL: if expired, `DELETE` and skip.
3. Apply the `prefix` filter against `key` (in Go — the dataset is small enough; keeping the SQL simple wins over a dynamic `WHERE key LIKE` that risks ordering / escaping bugs).

### 6.5 Clear

```sql
DELETE FROM entries;
```

- Guarded by `IsUserPath(ctx)` exactly like FileStore; non-user-path callers get `ErrSessionForbidden`.
- Does **not** run `VACUUM`. Documented: operator concern, `vv/memory/memory.db` will retain page slack until vacuumed externally. Keeps `Clear()` fast and non-blocking.

## 7. Configuration wiring

### 7.1 `MemoryConfig` addition

```go
type MemoryConfig struct {
    Dir            string `yaml:"dir"`
    SessionWindow  int    `yaml:"session_window"`
    PersistentLoad bool   `yaml:"persistent_load"`
    MaxConcurrency int    `yaml:"max_concurrency"`
    Backend        string `yaml:"backend"` // "file" (default) | "sqlite"
}
```

Validation added to `Validate()`:
- Empty → default to `"file"`.
- Set to a value other than `"file"` / `"sqlite"` → return `fmt.Errorf(`unknown memory backend %q (expected "file" or "sqlite")`, v)`.
- Env override: `VV_MEMORY_BACKEND`.

### 7.2 `setup.go` branch

```go
var store memory.Store
switch cfg.Memory.Backend {
case "", "file":
    fs, err := memories.NewFileStore(cfg.Memory.Dir)
    if err != nil { return nil, fmt.Errorf("create file store: %w", err) }
    store = fs
case "sqlite":
    ss, err := memories.NewSQLiteStore(cfg.Memory.Dir)
    if err != nil { return nil, fmt.Errorf("create sqlite store: %w", err) }
    store = ss
    // registered for shutdown so the DB closes cleanly
    result.Shutdown = chainShutdown(result.Shutdown, func(ctx context.Context) error { return ss.Close() })
default:
    // Defensive — validation already rejected; keeps the switch total.
    return nil, fmt.Errorf("unsupported memory backend %q", cfg.Memory.Backend)
}
persistentMem := memory.NewPersistentMemoryWithStore(store)
```

`chainShutdown` composes the existing `InitResult.Shutdown` with the SQLite close. If the trace shutdown already holds one, this adds SQLite on top. Errors from either are joined (use `errors.Join`).

### 7.3 CLI feedback

No new CLI command. The startup banner already prints the memory directory; an extra line `memory backend: sqlite (wal)` is printed when `cfg.Memory.Backend == "sqlite"` for operator awareness. Silent when `file` (default) to avoid banner noise.

## 8. Testing plan

### 8.1 Unit — `vv/memories/sqlitestore_test.go`

Ported test-by-test from `filestore_test.go`:

| FileStore test | SQLiteStore test | Notes |
|----------------|------------------|-------|
| `TestFileStore_SetAndGet` | `TestSQLiteStore_SetAndGet` | File-existence check becomes `SELECT COUNT(*)`. |
| `TestFileStore_GetNotFound` | `TestSQLiteStore_GetNotFound` | verbatim |
| `TestFileStore_Delete` | `TestSQLiteStore_Delete` | verbatim |
| `TestFileStore_DeleteNonExistent` | `TestSQLiteStore_DeleteNonExistent` | verbatim |
| `TestFileStore_List` | `TestSQLiteStore_List` | verbatim |
| `TestFileStore_Clear` | `TestSQLiteStore_Clear` | `SELECT COUNT(*)` to verify empty. |
| `TestFileStore_Clear_NonUserPathForbidden` | `TestSQLiteStore_Clear_NonUserPathForbidden` | verbatim |
| `TestFileStore_UpdatePreservesCreatedAt` | `TestSQLiteStore_UpdatePreservesCreatedAt` | relies on UPSERT not touching created_at. |
| `TestFileStore_DefaultNamespace` | `TestSQLiteStore_DefaultNamespace` | verbatim |
| `TestFileStore_WithSharedNamespaces_Extends` | `TestSQLiteStore_WithSharedNamespaces_Extends` | verbatim |
| `TestFileStore_UserPath_PrivateWrite_Forbidden` | `TestSQLiteStore_UserPath_PrivateWrite_Forbidden` | verbatim |
| `TestFileStore_AgentPath_PrivateRoundTrip` | `TestSQLiteStore_AgentPath_PrivateRoundTrip` | file-path check replaced by `SELECT ... WHERE session_id='session-A'`. |
| `TestFileStore_AgentPath_CrossSessionRead_NotFound` | `TestSQLiteStore_AgentPath_CrossSessionRead_NotFound` | verbatim |
| `TestFileStore_AgentPath_CrossSessionIsolation_Overwrite` | `TestSQLiteStore_AgentPath_CrossSessionIsolation_Overwrite` | verbatim |
| `TestFileStore_AgentPath_CrossSessionIsolation_Delete` | `TestSQLiteStore_AgentPath_CrossSessionIsolation_Delete` | verbatim |
| `TestFileStore_AgentPath_DefenseInDepth_OverwriteBlocked` | `TestSQLiteStore_AgentPath_StructuralIsolation` | replaced: assert PK separation — can't write `session_id='session-A'` at B's PK by construction. |
| `TestFileStore_Delete_NoSessionOnPrivate_Forbidden` | `TestSQLiteStore_Delete_NoSessionOnPrivate_Forbidden` | verbatim |
| `TestFileStore_List_FiltersBySession` | `TestSQLiteStore_List_FiltersBySession` | verbatim |
| `TestFileStore_Set_NoSessionOnPrivate_Forbidden` | `TestSQLiteStore_Set_NoSessionOnPrivate_Forbidden` | verbatim |
| `TestFileStore_Legacy_PrivateNs_Readable` | `TestSQLiteStore_Legacy_PrivateNs_Readable` | legacy seeded by direct `INSERT` with `session_id=''`. |
| `TestFileStore_Legacy_PrivateNs_OverwriteBlocked` | `TestSQLiteStore_Legacy_PrivateNs_OverwriteBlocked` | verbatim |
| `TestFileStore_Shared_UserPath_WriteAndSessionFree` | `TestSQLiteStore_Shared_UserPath_WriteAndSessionFree` | file-check replaced by `SELECT session_id`. |

Additional SQLite-specific tests:

- `TestSQLiteStore_WALEnabled` — opens store, `SELECT` `PRAGMA journal_mode`, asserts `"wal"`.
- `TestSQLiteStore_UserVersion` — asserts `PRAGMA user_version == 1`.
- `TestSQLiteStore_ConcurrentWrites` — spawn N goroutines, each performing 50 Sets under different session_ids; assert final row count == N * 50 and zero errors.
- `TestSQLiteStore_FailFast_OnCorruptDB` — seed the path with `[]byte("garbage")`, assert `NewSQLiteStore` returns non-nil err (not a nil-pointer on first Get).
- `TestSQLiteStore_TTLExpiry_DeletesRow` — Set with TTL=1, Sleep 1.2s, Get returns `found=false`, then `SELECT COUNT(*)` to verify the row was physically deleted.

### 8.2 Config validation test

- Add one case to `vv/configs/config_test.go` (or the nearest equivalent) asserting that `memory: { backend: sqlite }` validates, `backend: bogus` errors, empty defaults to `"file"`.

### 8.3 Integration

- Existing `vv/integrations/**` tests continue to use FileStore (default). No regression expected.
- Add a single integration test `vv/integrations/setup_tests/setup_tests/setup_sqlite_backend_test.go` that runs `configs.Load` with a YAML that selects `backend: sqlite`, calls `setup.Init`, Sets a key via `persistentMem`, and reads it back.

## 9. Non-functional notes

- **CGO:** `modernc.org/sqlite` is pure Go; `go.mod` update + `go mod tidy`. `CGO_ENABLED=0` build still works.
- **Binary size:** `modernc.org/sqlite` adds ~5 MB to the binary. Acceptable for a CLI tool; documented.
- **Fork safety:** N/A — vv doesn't fork.
- **Cross-platform:** modernc.org/sqlite ships AMD64 / ARM64 Linux / macOS / Windows / FreeBSD — same matrix vv already targets.
- **Licensing:** modernc.org/sqlite is BSD-3-Clause (vendor-permissive); Apache 2.0 license header check in `make build` already passes through go.sum only.

## 10. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| `SQLITE_BUSY` under concurrent writers | `busy_timeout=5000`; `TestSQLiteStore_ConcurrentWrites` exercises. |
| Slow first open (page allocation) | Constructor runs a probe statement; typical cost <20 ms on SSD. |
| Schema evolution trap | `PRAGMA user_version` check refuses to open a DB written by a newer vv. |
| Binary-size shock to CGO-less users | Documented in requirement/PRD; the trade is "WAL + transactions" which is the whole point. |
| `Clear()` leaks pages | Accepted; documented. Adding `VACUUM` here stalls other ops. Left for operator / a future `vv memory vacuum` command. |
| Accidental data loss on `backend: file → sqlite` flip | Documented: no auto-migration; the SQLite store starts empty. Users keep their JSON tree on disk if they flip back. |

## 11. Follow-ups explicitly deferred

- Migrator (FileStore → SQLiteStore) — one-shot CLI command. Ship when a user asks; the upstream data is still on disk as JSON, so late migration is non-destructive.
- FTS5 virtual table + `session_search` tool — P2-3, attaches to this DB.
- Daily budget counter persistence — P1-3 follow-up, one new table keyed by UTC date.
- Session checkpoints — P2-14, a `sessions` table.
- Vector recall — P3-10, `sqlite-vec` extension or separate file.
- Default backend flip to SQLite — candidate for M4 when downstream features make SQLite the obvious choice.
