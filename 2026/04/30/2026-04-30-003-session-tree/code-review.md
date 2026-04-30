# Code Review — Session Tree (P10) MVP

Reviewer: Claude (Opus 4.7)
Date: 2026-04-30

Scope: every file the developer created or modified in this dev session, listed in the orchestrator brief. The implementation closely follows the design and reuses well-established patterns from `vage/session`, `vage/workspace`, and `vage/context/sources_*`.

Overall the diff is sound. Two issues warranted code changes; the rest are tracked as accepted-as-is or documented for future iterations.

---

## Findings

### 1. Concurrency / Correctness — `FileTreeStore` dispatched hooks under the per-session lock (FIXED)

**Severity**: medium.

**Location**: `vage/session/tree/filestore.go` — every mutation method (`CreateTree`, `AddNode`, `UpdateNode`, `DeleteNode`, `SetCursor`, `DeleteTree`).

**Problem**: Every method opened with `mu.Lock(); defer mu.Unlock()` and then called `s.dispatch(ctx, ...)` *while still holding* the per-session lock. `hook.Manager.Dispatch` invokes synchronous hooks inline (see `vage/hook/manager.go:74`). A sync hook that calls back into the store on the same session — for example a tracelog hook that wanted to update the cursor in response to a `SessionTreeOpAdd` event — would deadlock on its own session's mutex.

The `MapTreeStore` already used the safer pattern: assemble snapshot under the lock, `mu.Unlock()`, then dispatch. That asymmetry was the reviewer's specific concern in the brief.

**Fix applied**: Replaced `defer mu.Unlock()` with explicit `mu.Unlock()` calls placed before each `s.dispatch(...)` (and before each early-return). For `SetCursor` I additionally clone the cursor node into `snapshot` *under* the lock, because the in-store node pointer would otherwise be exposed to dispatch after concurrent writers could mutate it. Updated the type-level docstring to record the new invariant: "Hook dispatch happens AFTER the per-session lock is released."

**Test added**: `TestFileStoreSyncHookReentrant` in `filestore_test.go` registers a sync hook that calls `SetCursor` back on the originating store, then runs `CreateTree`. The test guards against regressions: pre-fix it would deadlock on the per-session mutex and trip the 5-second test timeout; post-fix it completes in milliseconds.

### 2. Contract — `SessionTreeSource.MaxPathDepth` was not plumbed to `defaultTreeRender` (FIXED)

**Severity**: medium (silent contract break).

**Location**: `vage/context/sources_tree.go`.

**Problem**: `SessionTreeSource` exposes `MaxPathDepth` and the field is documented as "0 -> default 6". `defaultTreeRender` had:

```go
maxPathDepth := defaultTreeMaxPathDepth
if v := pathDepthFromContext(view); v > 0 {
    maxPathDepth = v
}
```

…where `pathDepthFromContext` was a stub that always returned 0. So `MaxPathDepth` had no effect on the default renderer — only custom renderers that read the field directly would see it.

**Fix applied**: Added a `MaxPathDepth int` field to `TreeView`, set it in `buildView` from `s.MaxPathDepth`, and read it in `defaultTreeRender`. Removed the `pathDepthFromContext` stub.

**Test added**: `TestTreeSource_MaxPathDepth` constructs a 5-deep path and sets `MaxPathDepth=1`, then asserts the cursor's summary survives but the next-to-last summary is degraded out of the path block. Pre-fix this test fails because the default 6 wins.

---

## Findings — accepted as-is

### 3. Concurrency — node id cannot escape the on-disk sandbox

**Severity**: none (verified safe).

`FileTreeStore` uses node IDs only as keys inside `tree.json` — never in `filepath.Join`. The only path component derived from user input is `sessionID`, which is filtered through `validateSessionID` (regex `^[A-Za-z0-9._-]{1,128}$`, rejects "..", "."). No traversal vector. Confirmed by grepping every `filepath.Join` site.

### 4. Concurrency — readers seeing partial state in FileTreeStore

**Severity**: none (verified safe).

`writeTree` writes to a `.tmp` file, fsyncs, then `os.Rename`s onto `tree.json`. POSIX rename is atomic, and `os.ReadFile` is a single syscall that gets either the old or the new inode. Lock-free reads are safe.

### 5. Aliasing in `applyUpdate`

**Severity**: none.

`applyUpdate` rebuilds `Evidence`, `Supersedes`, and `Metadata` via `append([]string(nil), ...)` / `make(map[...]any, ...)`. The caller's post-call mutation of its own `n` cannot leak into the store, and the store's pointer cannot leak into the caller. `CreateTree` / `AddNode` already pre-clone via `cloneNode(&n)` before mutating. Confirmed by reading both call sites.

### 6. Validation duplication between `tree` and `vage/session`

**Severity**: very low (intentional).

`validateSessionID` in `tree/tree.go` re-implements the regex from `vage/session`. The duplication is deliberate — the design decision (§1.2 of design.md) is to avoid a circular import between `vage/session/tree` and `vage/session`. Both regexes are simple, tested, and stable. No fix.

### 7. `SessionTreeSource.Fetch` fail-open contract

**Severity**: none (verified).

Walked every code path: `nil store / empty session` → skipped, `ErrTreeMissing` → skipped, other store error → error (no propagate), no-root → error (no propagate), renderer panic → recoveringTreeRenderer turns it into "" → skipped, byte cap exceeded → truncated. No path returns a non-nil error to the Builder. Matches design §4.1.

### 8. `removeID` mutates underlying array

**Severity**: very low.

`removeID` does `append(ids[:i], ids[i+1:]...)`, which is `O(n)` but mutates the backing array. Since `parent.Children` is the in-store slice (under lock), and readers receive deep copies via `cloneNode`, no reader can observe the mid-mutation state. Same approach used elsewhere in the repo. No fix.

### 9. `FileTreeStore.lockFor` lifecycle on DeleteTree

**Severity**: very low.

`DeleteTree` does `s.locks.Delete(sessionID)` *while holding* the mutex. A goroutine that called `lockFor` concurrently could end up holding a stale `*sync.Mutex` no longer in the map — but its operations all key off `sessionDir(sessionID)`, which `os.RemoveAll`-ed; subsequent `os.Stat` returns `ErrNotExist`, which surfaces naturally (`ErrAlreadyExists` cannot trigger; `readTree` returns `ErrTreeMissing`). Identical pattern to `vage/workspace.FileWorkspace.Delete`. No fix.

### 10. `defaultTreeRender` uses leading dash on the goal node

**Severity**: nit.

The design example showed `[goal] [active] add OAuth login (root)` (no dash, "(root)" suffix). The implementation always uses `writeNodeLine` which prepends "- ". Tests assert markers but not the exact prefix, so this is consistent rather than wrong. No fix — design example was illustrative.

### 11. `MapTreeStore` dispatch ordering with concurrent mutations

**Severity**: very low.

`MapTreeStore` releases the lock before dispatch, so two concurrent mutations may dispatch their events in an order that does not match the in-memory write order. The documented invariant ("event count == successful writes") still holds. Strong ordering would require dispatch under the lock, which conflicts with finding #1. Acceptable trade-off.

---

## Files changed by this review

- `vage/session/tree/filestore.go` — refactored to release per-session lock before hook dispatch (5 mutations + DeleteTree). Added invariant docstring.
- `vage/session/tree/filestore_test.go` — new `TestFileStoreSyncHookReentrant` plus a small `reentrantHookManager` helper.
- `vage/context/sources_tree.go` — `TreeView.MaxPathDepth` plumbing; removed stub `pathDepthFromContext`.
- `vage/context/sources_tree_test.go` — new `TestTreeSource_MaxPathDepth`.

## Verification

- `go test ./...` from the `vage/` module: all packages green, including the new tests.
- `make lint`: 0 issues.
