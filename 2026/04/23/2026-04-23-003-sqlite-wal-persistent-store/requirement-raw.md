# Raw requirement (P1-6: SQLite + WAL 持久化 Store)

Source: `doc/prd/feature-todo.md` · row **P1-6** — first unimplemented row in the priority table.

> **P1-6 · SQLite + WAL 持久化 Store**
> - Category: 记忆 (memory)
> - Dependency: P1-5 (structured JSONL trace — ✅ already implemented)
> - Difficulty: 中 (medium)
> - Reuse: `vage/memory/Store` interface, `vv/memories/filestore.go`
> - Rationale (from the roadmap): *FileStore（JSON 文件）无事务/无索引/线性扫描；实现 `SQLiteStore` 保持接口兼容；FTS5 的前提。*

## Why now

The current `vv/memories/FileStore`:
1. One JSON file per entry with full-tree `os.ReadDir` traversal on every `List` → O(N) disk I/O per listing.
2. No transactions → two concurrent writers to the same namespace can interleave `read-modify-write` and silently corrupt the preserved `created_at`.
3. No indexes on key prefix, namespace, or `updated_at` → future `session_search` (P2-3 FTS5) has nothing to attach to.
4. `Clear()` is an `os.RemoveAll` over the root — no granularity, no audit.

A SQLite-backed store fixes all four, and **unlocks** the P1-6 downstream lineage already documented in the roadmap:
- P2-2 historical cost log → needs a queryable store.
- P2-3 FTS5 full-text search → needs SQLite + FTS5 virtual table.
- P2-7 dual-profile snapshot → benefits from key-prefix index.
- P2-14 session checkpoint → needs transactional updates.
- P3-10 vector recall → `sqlite-vec` extension rides on the same DB file.

## Market & industry survey

Reviewed how comparable Go-based agent / memory frameworks ship their persistent stores (as of 2026-Q2):

| Product | Backend | Driver | WAL? | Notes |
|---------|---------|--------|------|-------|
| Claude Code (`~/.claude/projects/`) | JSONL + JSON files | N/A | N/A | Pure file-per-session; similar to current `FileStore` — they compensate with in-memory caches and rotation. |
| Cursor / Cline | SQLite (main.db) | `better-sqlite3` (Node) | WAL | Uses WAL + `synchronous=NORMAL` for fast tool-heavy sessions. |
| Aider | SQLite (`~/.aider/chat_history.db`) | `sqlite3` stdlib (Python) | Default (no WAL) | Relies on single-writer lock; documents concurrent-use limits. |
| Continue.dev | SQLite (per workspace) | `better-sqlite3` | WAL | Adds schema-migration table; `PRAGMA user_version`. |
| Mem0 | SQLite + vector | `sqlite3` + `sqlite-vss` | WAL | Hybrid BM25 + vector queries; schema migrations. |
| LangChain Memory | Pluggable (SQLite, Postgres, Redis) | n/a | WAL (SQLite default) | Driver-agnostic — `SQLChatMessageHistory` is the reference impl. |
| LlamaIndex | SQLite + JSON on disk | `sqlite3` | WAL | Similar hybrid. |
| Letta (MemGPT) | PostgreSQL or SQLite | `sqlite3`/psql | WAL | `created_at`, `updated_at` columns; `delete_at` soft-delete. |
| Zep Memory Server | Postgres | pgx | n/a | Server-side; overkill for a CLI/HTTP agent. |
| Ollama | SQLite for models DB | `mattn/go-sqlite3` (CGO) | WAL | Go ecosystem precedent for `WAL + synchronous=NORMAL`. |

Common patterns that emerged:
1. **Single DB file per store root**, not per namespace — makes transactions and backup trivial.
2. **WAL mode is default** (`PRAGMA journal_mode=WAL`), paired with `synchronous=NORMAL` and a `busy_timeout` ≥ 5 s. Two readers + one writer can proceed concurrently; checkpointing is automatic.
3. **Single `entries` table** keyed by `(namespace, name, session_id)` — use `session_id = ''` to represent the shared-scope record; indexes on `namespace`, `session_id`, `updated_at`, and a compound `(session_id, namespace)` cover every existing `List` / `Get` path.
4. **Pure-Go drivers are becoming the norm.** `modernc.org/sqlite` (pure Go, no CGO) is now production-grade and used by Bluesky / SourceHut etc. No cross-compilation pain.
5. **Schema versioning is tracked via `PRAGMA user_version`** — lets us add FTS5 / vector tables later without hand-editing existing files.
6. **Legacy/file-based data is not auto-migrated.** Most projects either (a) start fresh per install or (b) ship a small one-shot migrator. Bulk auto-migration on startup is a source of slow first-run and silent corruption — surveyed projects avoid it.

## What to watch out for

Issues called out in the bug trackers of the surveyed projects (not theoretical — real regressions):

1. **Concurrent CLI + MCP server on the same DB file.** Both processes lock independently; single `sqlite.Open` without `?cache=shared&mode=rwc` + `busy_timeout` → `SQLITE_BUSY` under any contention. Must set `busy_timeout` **at the connection level** (via DSN or `PRAGMA busy_timeout` on every connection), not once.
2. **`go-sqlite3` CGO vs `modernc.org/sqlite` pure Go.** Binaries must cross-compile easily (this repo already does). Pure Go chosen: no CGO in `vv/go.mod` today, easier release pipeline.
3. **`database/sql` connection pooling breaks WAL semantics** if `SetMaxOpenConns(1)` not set **for writes** OR if mixed-mode transactions aren't held on a single `*sql.Conn`. Known foot-gun.
4. **`Clear()` semantics**: `DELETE FROM entries` leaks pages inside the DB file; need `VACUUM` or `PRAGMA auto_vacuum=INCREMENTAL` + explicit `PRAGMA incremental_vacuum` on clear for tests that check on-disk size.
5. **TTL sweeping.** Current FileStore does lazy TTL (delete on read). SQLite allows batch `DELETE WHERE ttl > 0 AND updated_at + ttl < now()` but running it on every call is wasteful. Options: keep lazy-on-read like FileStore (keeps parity), or add a cheap sweep on `List` with a throttle. Industry answer is lazy-on-read + periodic compaction — we'll mirror.
6. **`created_at` preservation on UPDATE.** FileStore reads the old record before overwriting to preserve `CreatedAt`. In SQL, `INSERT ... ON CONFLICT(key_triple) DO UPDATE SET value=..., updated_at=... (created_at stays)` — UPSERT handles it with no round-trip.
7. **Session-binding defense-in-depth.** FileStore plants `session_id` on disk and rejects mismatched overwrite. SQLite must preserve the same semantics: `PRIMARY KEY` on `(namespace, name, session_id)` separates A/B physically like the `session/<sid>/` dir does today, **and** the write path must still read the existing row to catch cross-session collisions in the legacy (session_id='') slot.
8. **Context cancellation** — `database/sql` respects `ctx` only for Query/Exec once open; `Open` itself does not. We accept this; the store is long-lived.
9. **Disk full / permission errors on first open.** `sqlite3_open_v2` returns errors very late (after the first statement). We must run a probe statement during `NewSQLiteStore` to fail fast — surveyed projects that didn't do this had opaque nil-pointer panics from `manager.NewPersistentMemoryWithStore`.
10. **File permissions.** FileStore uses `0o700` dir / `0o600` files. Must set `0o600` on the DB file after open. Note: SQLite creates the DB + `-wal` + `-shm` files; `0o600` must be applied once the WAL pair exists.
11. **Legacy record handling.** FileStore supports legacy records (missing `session_id` in a now-private namespace). Our SQL schema must model the same — `session_id` column `NOT NULL DEFAULT ''`, legacy rows have `session_id=''` stored under a private namespace.
12. **Unicode / case sensitivity.** SQLite is case-sensitive by default for `LIKE`; we only use prefix match via `>=` / `<` bounds so this doesn't bite, but flagged.

## Questions worth surfacing (for analyst phase)

- **Migration from FileStore → SQLiteStore:** do we ship an automatic migrator, a one-shot `vv migrate memory` CLI, or nothing?
- **Default backend for new installs:** keep `FileStore` until M4, or flip default to SQLite now that we're landing it?
- **Downstream dependents (P2-3, P2-14, P3-10):** should we add extension points (migrations table, placeholder for FTS5 trigger) in this PR, or stay strictly minimal?
- **Single DB file location:** reuse `cfg.Memory.Dir + "/memory.db"`, or a separate `cfg.Memory.SQLitePath`?

## User's implicit asks

From the /loop prompt:
1. Research similar products → done above.
2. Research industry practices / implementation approaches → done above.
3. Note cautions → done above.
4. Reference these in the design.
5. After implementation, move item from `feature-todo.md` → `feature-implement.md`.
6. Run `/gitpush` to commit & push.
