# Design: Edit Tool Safety and Compliance Improvements

## 1. Architecture Overview

This change touches three packages within the `vage` module, adding safety checks at two layers:

1. **Shared layer** (`tool/toolkit/`): UNC path blocking added to `ValidatePath`, plus a new `ReadTracker` interface.
2. **Edit tool** (`tool/edit/`): Wires new checks (deny rules, read prerequisite, write permission) into the handler pipeline, widens the file-size type, and improves error messages.
3. **Read tool** (`tool/read/`): Records successful reads into a `ReadTracker` when one is configured.

No new packages are introduced. No changes to the tool registry, schema, or agent layers.

### Handler pipeline (after changes)

```
  JSON parse
    |
  ValidatePath  (now includes UNC check)
    |
  deny-rule check  (new)
    |
  read-prerequisite check  (new, optional)
    |
  LockPath
    |
  os.Stat  ->  write-permission pre-check  (new)
    |
  size check  (improved error message, int64)
    |
  ReadFile -> string match -> replace -> AtomicWriteFile
    |
  result with snippet
```

## 2. Component Design

### 2.1 `tool/toolkit/path.go` -- UNC path blocking

Add a UNC check at the top of `ValidatePath`, before the `filepath.IsAbs` check. Reject paths that look like UNC network paths.

```
ValidatePath(toolName, path, allowedDirs):
  if path starts with "\\\\":
    return error "<tool> tool: UNC paths are not allowed: <path>"
  if path starts with "//" and len(path) > 2 and path[2] != '/':
    return error "<tool> tool: UNC paths are not allowed: <path>"
  ... existing checks ...
```

The `\\` prefix must be checked as a raw string (two backslashes). The `//` check is narrowed to avoid false positives on paths like `///foo` which `filepath.Clean` handles correctly. Only `//server/...` style paths are rejected. On non-Windows systems `filepath.IsAbs("\\\\server\\share")` returns false and would already fail, but the explicit check gives a targeted error message and protects against edge cases.

This check benefits all tools that call `ValidatePath` (read, write, edit) automatically.

### 2.2 `tool/toolkit/path_test.go` -- new file

Create unit tests for `ValidatePath` covering:

- UNC path `\\server\share\file.txt` rejected.
- UNC path `//server/share/file.txt` rejected.
- Triple-slash path `///foo/bar` not rejected (handled by `filepath.Clean`).
- Normal absolute path still accepted.
- Empty path rejected.
- Relative path rejected.
- Allowed-dirs enforcement (existing behavior, but not previously tested at this layer).

### 2.3 `tool/toolkit/readtracker.go` -- new file

Define the `ReadTracker` interface and a default in-memory implementation:

```go
// ReadTracker tracks which file paths have been read.
type ReadTracker interface {
    HasRead(path string) bool
    RecordRead(path string)
}
```

Provide `NewMemoryReadTracker(maxEntries int) ReadTracker` returning a `sync.Mutex`-guarded `map[string]bool` implementation. When `maxEntries` is reached, the map is cleared (simple eviction) to bound memory in long-running sessions. A `maxEntries` of 0 means unlimited (for short-lived or test scenarios).

This keeps the interface minimal and the default implementation trivial. Both the read tool and edit tool must be wired with the same `ReadTracker` instance for the feature to work. When no tracker is configured, both tools behave as before (backward compatible).

### 2.4 `tool/toolkit/readtracker_test.go` -- new file

Unit tests for `MemoryReadTracker`:

- `HasRead` returns false for unrecorded path.
- `HasRead` returns true after `RecordRead`.
- Concurrent `RecordRead`/`HasRead` safety (parallel goroutines).
- Eviction: after recording `maxEntries` paths, recording one more clears the map and only the new entry remains.

### 2.5 `tool/edit/edit_tool.go` -- modifications

**Struct changes:**

```go
type EditTool struct {
    allowedDirs  []string
    maxFileBytes int64                // changed from int
    denyRules    []string             // new
    readTracker  toolkit.ReadTracker  // new, optional
}
```

**Constant changes:**

```go
defaultMaxEditFileBytes int64 = 10 * 1024 * 1024  // 10 MB
```

The default is set to 10 MB rather than 1 GiB. The edit tool reads the entire file into memory and performs string operations that create multiple copies, so a 1 GiB default would risk multi-gigabyte memory allocations. Callers who need to edit larger files can use `WithMaxFileBytes` to raise the limit and accept the memory cost.

**New functional options:**

```go
func WithMaxFileBytes(n int64) Option       // signature change: int -> int64
func WithDenyRules(patterns ...string) Option
func WithReadTracker(tracker toolkit.ReadTracker) Option
```

**Private deny-rule helper:**

```go
// matchDenyRule checks path against a list of glob patterns.
// Patterns are matched against filepath.Base(path) using filepath.Match.
// Returns the first matching pattern, or "" if none match.
func matchDenyRule(path string, patterns []string) string
```

This is a private function within the `edit` package. It uses `filepath.Match` against `filepath.Base(path)` for all patterns. Recursive-directory globs (`**`) are not supported by `filepath.Match` and are not included in scope. Patterns should be basename-level globs (e.g., `*.env`, `*.lock`, `credentials.json`).

**Handler changes (in order of execution):**

1. After `ValidatePath` succeeds, check deny rules:
   ```
   if matched := matchDenyRule(cleaned, et.denyRules); matched != "" {
       return error "edit tool: file is protected by deny rule %q: %s", matched, cleaned
   }
   ```

2. After deny-rule check, check read prerequisite:
   ```
   if et.readTracker != nil && !et.readTracker.HasRead(cleaned) {
       return error "edit tool: file must be read before editing; use the read tool first: <path>"
   }
   ```

3. After `os.Stat` succeeds, check write permission (owner write bit):
   ```
   if info.Mode().Perm()&0o200 == 0 {
       return error "edit tool: file appears to be read-only (mode %s): %s", info.Mode().Perm(), cleaned
   }
   ```
   Note: This checks the owner write bit only. It is a best-effort hint, not a guarantee. The actual write will fail with a clear OS error if the process lacks permission. Including the file mode in the error message helps the agent diagnose the issue.

4. Improve the file-size error message:
   ```
   "edit tool: file size (%d bytes) exceeds maximum allowed (%d bytes): %s"
   ```

5. Improve the not-found error message:
   ```
   "edit tool: old_string not found in file. Possible causes: whitespace/indentation mismatch, or the file may have changed since last read. File: %s"
   ```

### 2.6 `tool/edit/deny_test.go` -- new file

Unit tests for the private `matchDenyRule` helper (tested indirectly through the handler, or via an exported test helper if needed):

- Exact basename match (`*.env` matches `/home/user/.env`).
- Glob wildcard match (`*.lock` matches `/repo/go.lock`).
- No-match pass-through returns `""`.
- Multiple patterns: first match wins.
- Invalid pattern does not panic (graceful skip).

Since `matchDenyRule` is private, these tests live in the `edit` package and test the function directly.

### 2.7 `tool/read/read_tool.go` -- modifications

**Struct changes:**

```go
type ReadTool struct {
    allowedDirs  []string
    maxReadBytes int
    readTracker  toolkit.ReadTracker  // new, optional
}
```

**New functional option:**

```go
func WithReadTracker(tracker toolkit.ReadTracker) Option
```

**Handler change:**

After a successful file read (just before returning the `TextResult` for a file -- not for directory listings), record the read:

```go
if rt.readTracker != nil {
    rt.readTracker.RecordRead(cleaned)
}
```

### 2.8 `tool/edit/edit_tool_test.go` -- modifications

Add or update the following tests:

| Test | Purpose |
|------|---------|
| `TestEditTool_DenyRule_ExactMatch` | `*.env` blocks `/tmp/dir/.env` |
| `TestEditTool_DenyRule_GlobMatch` | `*.lock` blocks `/tmp/dir/go.lock` |
| `TestEditTool_DenyRule_NoMatch` | Edit succeeds when path does not match any deny rule |
| `TestEditTool_ReadPrerequisite_Blocked` | Edit rejected when tracker configured and file not read |
| `TestEditTool_ReadPrerequisite_Allowed` | Edit succeeds after `RecordRead` called |
| `TestEditTool_ReadPrerequisite_NoTracker` | Edit succeeds when no tracker configured (backward compat) |
| `TestEditTool_ReadOnlyFile` | Read-only file returns `"file appears to be read-only"` error with mode |
| `TestEditTool_ExceedsMaxFileBytes` | Update to use `int64` argument and check for file size and limit in error message |
| `TestEditTool_NotFound` | Update assertion to check for actionable guidance in error message |
| `TestEditTool_UNCPath` | `\\server\share\file.txt` rejected (via `ValidatePath`) |
| `TestEditTool_UNCPathSlash` | `//server/share/file.txt` rejected (via `ValidatePath`) |

### 2.9 `tool/read/read_tool_test.go` -- modifications

Add:

| Test | Purpose |
|------|---------|
| `TestReadTool_ReadTrackerRecords` | After a successful file read, `HasRead` returns true |
| `TestReadTool_ReadTrackerNotCalledForDir` | Directory listing does not call `RecordRead` |

## 3. Implementation Plan

Tasks are ordered by dependency. Steps within a group can be done in parallel.

### Step 1: Shared utilities (toolkit)

1. Add UNC path check to `ValidatePath` in `tool/toolkit/path.go`.
2. Create `tool/toolkit/readtracker.go` with `ReadTracker` interface and `MemoryReadTracker`.
3. Create `tool/toolkit/path_test.go` with `ValidatePath` unit tests (including UNC).
4. Create `tool/toolkit/readtracker_test.go`.

### Step 2: Edit tool changes

1. Change `maxFileBytes` field type to `int64` and update `defaultMaxEditFileBytes` constant to 10 MB.
2. Update `WithMaxFileBytes` signature from `int` to `int64`.
3. Add private `matchDenyRule` helper function.
4. Add `denyRules` field and `WithDenyRules` option.
5. Add `readTracker` field and `WithReadTracker` option.
6. Add deny-rule check in handler (after path validation, before file lock).
7. Add read-prerequisite check in handler (after deny-rule check, before file lock).
8. Add write-permission pre-check in handler (after `os.Stat`, before size check).
9. Improve error messages for not-found, size-exceeded, and deny-rule cases.
10. Update and add unit tests in `tool/edit/edit_tool_test.go`.

### Step 3: Read tool changes

1. Add `readTracker` field and `WithReadTracker` option to `ReadTool`.
2. Call `RecordRead` after successful file reads (not directory listings).
3. Add unit tests in `tool/read/read_tool_test.go`.

### Step 4: Verify

1. Run `make build` from `vage/` to confirm license-check, format, lint, and all tests pass.

## 4. Backward Compatibility

All changes are backward compatible:

- `WithMaxFileBytes` signature changes from `int` to `int64`. Go allows implicit conversion from untyped integer constants, so existing callers like `WithMaxFileBytes(100)` continue to compile. Callers passing a typed `int` variable will need a cast -- this is a minor breaking change. If this is unacceptable, add a new `WithMaxFileBytesInt64` option and deprecate the old one. Given this is a pre-1.0 framework, the signature change is acceptable.
- New options (`WithDenyRules`, `WithReadTracker`) are purely additive.
- Default behavior when no deny rules or tracker are configured is identical to current behavior.
- Error message text changes may affect tests that assert on exact error strings. These are internal tests and will be updated as part of this change.

## 5. Files Summary

| File | Action | Description |
|------|--------|-------------|
| `vage/tool/toolkit/path.go` | Modify | Add UNC path blocking to `ValidatePath` |
| `vage/tool/toolkit/path_test.go` | Create | Unit tests for `ValidatePath` |
| `vage/tool/toolkit/readtracker.go` | Create | `ReadTracker` interface and `MemoryReadTracker` |
| `vage/tool/toolkit/readtracker_test.go` | Create | Unit tests for `MemoryReadTracker` |
| `vage/tool/edit/edit_tool.go` | Modify | All edit tool changes (type, options, checks, error messages, private deny helper) |
| `vage/tool/edit/edit_tool_test.go` | Modify | New and updated unit tests (including deny-rule tests) |
| `vage/tool/read/read_tool.go` | Modify | Add `ReadTracker` recording |
| `vage/tool/read/read_tool_test.go` | Modify | Tests for tracker recording |
