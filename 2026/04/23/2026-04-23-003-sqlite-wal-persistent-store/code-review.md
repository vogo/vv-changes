# Code review — P1-6 · SQLite + WAL persistent memory store

Reviewer: Opus 4.7 (1M)
Scope: the diff listed in the task prompt. All changes applied live in `vv/memories/`, `vv/configs/`, `vv/setup/`, `vv/integrations/setup_tests/setup_tests/`, and `vv/go.{mod,sum}`.

## Summary

Design fidelity is high: the schema, DSN PRAGMAs, UPSERT semantics, visibility rules, and the fail-fast probe match the design doc one-for-one. FileStore parity tests were ported verbatim plus the specifically-promoted `TestSQLiteStore_AgentPath_StructuralIsolation`. The `Option` → interface refactor is clean because (a) both impls share `sharedNamespacesOpt` and (b) no external caller in this module constructs `Option` values directly.

Two real correctness wrinkles surfaced during review, plus one minor resource leak in `setup.Init`. All three have been fixed in the developer's tree. No passing tests regressed; lint remained at zero issues.

## Blocking issues (all fixed)

### B1. TTL lazy-delete could clobber a concurrently-refreshed row

Both `getRow` (Get) and the List lazy-sweep issued an unconditional `DELETE WHERE (ns, name, session_id)` after seeing an expired row. Under WAL, a writer may Set the same row between the SELECT and the DELETE — leaving a brand-new entry with a fresh `updated_at`. The old code would still delete it, silently destroying a just-written value.

Mitigation: carry `updated_at` forward from the SELECT and include it in the DELETE predicate, turning the race into a benign no-op when a writer slipped in.

- `vv/memories/sqlitestore.go` `getRow`: DELETE now adds `AND updated_at = ?`.
- `vv/memories/sqlitestore.go` `List`: `sqlitePK` gained an `updatedAt` field; the post-iteration sweep includes it in the predicate.

Tests still pass (`TestSQLiteStore_TTLExpiry_DeletesRow`, `TestSQLiteStore_ConcurrentWrites`) and the fix is invisible to single-threaded callers.

### B2. `setup.Init` leaked the SQLite DB if `buildHookManager` failed

`openMemoryStore` succeeded → `buildHookManager` returned an error → `Init` returned early without calling `closeStore()`. The later success path correctly delegates to `chainShutdown`, but the hook-build failure path didn't.

Fix: the `buildHookManager` error branch now calls `closeStore()` before returning. Parallel to the existing `New(...)` failure branch that already closed the store.

File: `vv/setup/setup.go` around line 485.

## Non-blocking improvements applied

### N1. `List` returns the stored `name` column instead of reconstructing it

The original List SELECT omitted `name` and reconstructed it for the lazy-delete using `keyToName(ns, key)`, a helper that applied `parseKey`'s inverse rules. It was correct today but quietly coupled to the exact storage convention — any future change to how the `key` column is stored would silently break the delete.

Swapped to `SELECT namespace, name, key, ...` and deleted `keyToName`. The DELETE now uses the authoritative `name` from the row. Net: ~18 lines removed, zero reconstruction drift risk, no test changes needed.

File: `vv/memories/sqlitestore.go`.

## Non-blocking improvements not applied

### R1. Batching the List lazy-sweep into a single DELETE with an `IN` clause

`List`'s lazy-expiry loop issues N single-row DELETEs. Rejected: the sweep set is typically empty or tiny, the design doc explicitly called this out ("single-row deletes here keep the SQL simple; batching would need a prepared statement + IN-clause builder"), and grouping by `(ns, name, sid, updated_at)` would lose the per-row `updated_at` guard from B1. Not worth the complexity.

### R2. Use `PRAGMA user_version` as the fail-fast probe on pool connections

The probe `SELECT 1 FROM entries LIMIT 0` only runs on the initial connection. Pool conns that open later re-apply DSN PRAGMAs but don't re-probe. Rejected: every subsequent real query acts as a de-facto probe, and `db.Ping()` in the constructor already establishes one live connection. A dedicated ping-per-conn hook would need a custom driver.Connector — overkill for P1-6.

### R3. Verify DSN PRAGMAs fire on every pooled connection in the test suite

Reviewed focus area 4. `modernc.org/sqlite` applies `_pragma=foo(bar)` per-connection at conn-open time (confirmed by source inspection — `libc.Xsqlite3_exec` is invoked in each `(*Driver).Open`). `journal_mode=WAL` is a database-level, persistent setting so only one conn need set it. `synchronous`, `busy_timeout`, `foreign_keys` are per-conn but re-applied on every open. `TestSQLiteStore_ConcurrentWrites` (8 conns × contended Sets with the 5-second busy timeout) already exercises this and would fail fast if any pooled conn missed `busy_timeout`. Not adding another test.

### R4. Stamp `user_version` inside a transaction alongside the schema apply

`ensureSQLiteSchema` runs CREATE TABLE then PRAGMA user_version separately. If the process dies between them, the table exists without a user_version stamp — next open would re-apply the CREATE TABLE (idempotent via `IF NOT EXISTS`) and then stamp. Harmless; not a real risk. Rejected.

## Questions / follow-ups for the designer

- **Config validation nit.** `ValidateMemoryBackend` is called in `configs.Load` but the default branch of `openMemoryStore` still returns an error for unknown values. That's defensive (the comment even says so) but `configs.Load` normalizes empty → `"file"`, so `cfg.Memory.Backend` is always one of `"file"` or `"sqlite"` by the time `setup.Init` runs. The defensive branch is unreachable by the public API, worth a note if anyone ever starts constructing `configs.Config` programmatically without calling `Load`.

- **Concurrent `Set` race with a legacy-row plant.** `Set` for a private ns does a SELECT-for-legacy-row then UPSERT, non-atomically. In the current codebase, legacy rows can only be created by direct SQL (no API path writes `session_id=''` to a private namespace). If a future migrator or a `DB.Exec` caller adds rows directly, this gap widens. A `BEGIN IMMEDIATE; SELECT; UPSERT; COMMIT;` wrap would close it; deferred for now as the attack surface is "someone with direct DB access", which already defeats the whole model.

- **`keyToName` removal** trimmed the only place that called `parseKey`'s inverse. If P2-3 FTS5 lands and needs to reconstruct keys for its virtual-table indexer, consider exposing a formal `formatKey(ns, name)` helper rather than re-introducing a reconstruction function.

## Re-run results

```text
$ cd vv && go test ./...
[37/37 packages ok]

$ cd vv && make lint
golangci-lint run
0 issues.
```

No regressions introduced by the three changes above.
