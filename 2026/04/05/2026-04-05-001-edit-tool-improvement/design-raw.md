# Design: Edit Tool Safety and Compliance Improvements

## 1. Architecture Overview

This change touches three packages within the `vage` module, adding safety checks at two layers:

1. **Shared layer** (`tool/toolkit/`): UNC path blocking added to `ValidatePath`, plus a new `ReadTracker` interface and deny-rule matching utility.
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
  os.Stat  ->  write-permission check  (new)
    |
  size check  (improved error message, int64)
    |
  ReadFile -> string match -> replace -> AtomicWriteFile
    |
  result with snippet
```

## 2. Component Design

### 2.1 `tool/toolkit/path.go` -- UNC path blocking

Add a UNC check at the top of `ValidatePath`, before the `filepath.IsAbs` check. Reject any path starting with `\\` or `//`.

```
ValidatePath(toolName, path, allowedDirs):
  if path starts with "\\\\" or "//":
    return error "<tool> tool: UNC paths are not allowed: <path>"
  ... existing checks ...
```

The `\\` prefix must be checked as a raw string (two backslashes). On non-Windows systems `filepath.IsAbs("\\\\server\\share")` returns false and would already fail, but the explicit check gives a targeted error message and protects against edge cases.

### 2.2 `tool/toolkit/path_test.go` -- new file

Create unit tests for `ValidatePath` covering:

- UNC path `\\server\share\file.txt` rejected.
- UNC path `//server/share/file.txt` rejected.
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

Provide `NewMapReadTracker() ReadTracker` returning a `sync.Mutex`-guarded `map[string]bool` implementation. This keeps the interface minimal and the default implementation trivial.

### 2.4 `tool/toolkit/readtracker_test.go` -- new file

Unit tests for `MapReadTracker`:

- `HasRead` returns false for unrecorded path.
- `HasRead` returns true after `RecordRead`.
- Concurrent `RecordRead`/`HasRead` safety.

### 2.5 `tool/toolkit/deny.go` -- new file

Provide a utility function for deny-rule matching:

```go
// MatchDenyRule checks path against a list of glob patterns.
// Returns the first matching pattern, or "" if none match.
func MatchDenyRule(path string, patterns []string) string
```

Uses `filepath.Match` against `filepath.Base(path)` for simple patterns (no `/`), and against the full path for patterns containing `/`. This keeps the logic centralized so other tools can reuse it later.

### 2.6 `tool/toolkit/deny_test.go` -- new file

Unit tests:

- Exact basename match (`*.env` matches `/home/user/.env`).
- Glob wildcard match (`*.lock` matches `/repo/go.lock`).
- Pattern with path separator (`**/credentials.json` matches nested paths).
- No-match pass-through returns `""`.

### 2.7 `tool/edit/edit_tool.go` -- modifications

**Struct changes:**

```go
type EditTool struct {
    allowedDirs  []string
    maxFileBytes int64          // changed from int
    denyRules    []string       // new
    readTracker  toolkit.ReadTracker  // new, optional
}
```

**Constant changes:**

```go
defaultMaxEditFileBytes int64 = 1 * 1024 * 1024 * 1024  // 1 GiB
```

**New functional options:**

```go
func WithMaxFileBytes(n int64) Option       // signature change: int -> int64
func WithDenyRules(patterns ...string) Option
func WithReadTracker(tracker toolkit.ReadTracker) Option
```

**Handler changes (in order of execution):**

1. After `ValidatePath` succeeds, check deny rules:
   ```
   if matched := toolkit.MatchDenyRule(cleaned, et.denyRules); matched != "" {
       return error "edit tool: file is protected by deny rule %q: %s"
   }
   ```

2. After deny-rule check, check read prerequisite:
   ```
   if et.readTracker != nil && !et.readTracker.HasRead(cleaned) {
       return error "edit tool: file must be read before editing; use the read tool first: <path>"
   }
   ```

3. After `os.Stat` succeeds, check write permission:
   ```
   if info.Mode().Perm()&0o200 == 0 {
       return error "edit tool: file is not writable: <path>"
   }
   ```
   Note: This checks the owner write bit. A more thorough check could use `unix.Access`, but the simple mode-bit check is portable and sufficient for the stated requirement.

4. Improve the file-size error message:
   ```
   "edit tool: file size (%d bytes) exceeds maximum allowed (%d bytes): %s"
   ```

5. Improve the not-found error message:
   ```
   "edit tool: old_string not found in file. Possible causes: whitespace/indentation mismatch, or the file may have changed since last read."
   ```

### 2.8 `tool/read/read_tool.go` -- modifications

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

### 2.9 `tool/edit/edit_tool_test.go` -- modifications

Add or update the following tests:

| Test | Purpose |
|------|---------|
| `TestEditTool_DenyRule_ExactMatch` | `*.env` blocks `/tmp/dir/.env` |
| `TestEditTool_DenyRule_GlobMatch` | `*.lock` blocks `/tmp/dir/go.lock` |
| `TestEditTool_DenyRule_NoMatch` | Edit succeeds when path does not match any deny rule |
| `TestEditTool_ReadPrerequisite_Blocked` | Edit rejected when tracker configured and file not read |
| `TestEditTool_ReadPrerequisite_Allowed` | Edit succeeds after `RecordRead` called |
| `TestEditTool_ReadPrerequisite_NoTracker` | Edit succeeds when no tracker configured (backward compat) |
| `TestEditTool_ReadOnlyFile` | Read-only file returns `"file is not writable"` error |
| `TestEditTool_ExceedsMaxFileBytes` | Update assertion to check for file size and limit in error message |
| `TestEditTool_NotFound` | Update assertion to check for actionable guidance in error message |
| `TestEditTool_UNCPath` | `\\server\share\file.txt` rejected |
| `TestEditTool_UNCPathSlash` | `//server/share/file.txt` rejected |

Update `TestEditTool_ExceedsMaxFileBytes` to use `WithMaxFileBytes` with `int64` argument.

### 2.10 `tool/read/read_tool_test.go` -- modifications

Add:

| Test | Purpose |
|------|---------|
| `TestReadTool_ReadTrackerRecords` | After a successful read, `HasRead` returns true |
| `TestReadTool_ReadTrackerNotCalledForDir` | Directory listing does not call `RecordRead` |

## 3. Implementation Plan

Tasks are ordered by dependency. Steps within a group can be done in parallel.

### Step 1: Shared utilities (toolkit)

1. Add UNC path check to `ValidatePath` in `tool/toolkit/path.go`.
2. Create `tool/toolkit/readtracker.go` with `ReadTracker` interface and `MapReadTracker`.
3. Create `tool/toolkit/deny.go` with `MatchDenyRule`.
4. Create `tool/toolkit/path_test.go` with `ValidatePath` unit tests (including UNC).
5. Create `tool/toolkit/readtracker_test.go`.
6. Create `tool/toolkit/deny_test.go`.

### Step 2: Edit tool changes

1. Change `maxFileBytes` field type to `int64` and update `defaultMaxEditFileBytes` constant to 1 GiB.
2. Update `WithMaxFileBytes` signature from `int` to `int64`.
3. Add `denyRules` field and `WithDenyRules` option.
4. Add `readTracker` field and `WithReadTracker` option.
5. Add deny-rule check in handler (after path validation, before file lock).
6. Add read-prerequisite check in handler (after deny-rule check, before file lock).
7. Add write-permission pre-check in handler (after `os.Stat`, before size check).
8. Improve error messages for not-found, size-exceeded, and deny-rule cases.
9. Update and add unit tests in `tool/edit/edit_tool_test.go`.

### Step 3: Read tool changes

1. Add `readTracker` field and `WithReadTracker` option to `ReadTool`.
2. Call `RecordRead` after successful file reads.
3. Add unit tests in `tool/read/read_tool_test.go`.

### Step 4: Verify

1. Run `make build` from `vage/` to confirm license-check, format, lint, and all tests pass.

## 4. Integration Test Plan

Per the requirement, integration tests requiring LLM API keys are out of scope. The following manual/scripted integration scenarios should be verified post-implementation:

| Scenario | Steps | Expected |
|----------|-------|----------|
| End-to-end deny rule | Create an `EditTool` with `WithDenyRules("*.env")`. Attempt to edit a `.env` file via the handler. | Error returned naming the `*.env` rule. |
| End-to-end read prerequisite | Create a shared `MapReadTracker`. Create `ReadTool` and `EditTool` both with `WithReadTracker(tracker)`. Attempt edit without prior read. Then read, then edit. | First edit rejected. Second edit succeeds. |
| UNC path across tools | Call `ValidatePath` from read, edit, and write tools with `//server/share/file.txt`. | All three reject with UNC error. |
| Large file edit | Create a file slightly under 1 GiB. Edit it. Then create one slightly over. Edit it. | Under-limit succeeds. Over-limit fails with both sizes in the error message. |
| Read-only file | Create a file, `chmod 0444`. Attempt edit. | Error: `"file is not writable"`. |

These can all be run as standard Go tests (no LLM keys needed) and should be placed as unit tests colocated with source. No `integrations/` tests are needed for this change.

## 5. Files Summary

| File | Action | Description |
|------|--------|-------------|
| `vage/tool/toolkit/path.go` | Modify | Add UNC path blocking to `ValidatePath` |
| `vage/tool/toolkit/path_test.go` | Create | Unit tests for `ValidatePath` |
| `vage/tool/toolkit/readtracker.go` | Create | `ReadTracker` interface and `MapReadTracker` |
| `vage/tool/toolkit/readtracker_test.go` | Create | Unit tests for `MapReadTracker` |
| `vage/tool/toolkit/deny.go` | Create | `MatchDenyRule` utility |
| `vage/tool/toolkit/deny_test.go` | Create | Unit tests for `MatchDenyRule` |
| `vage/tool/edit/edit_tool.go` | Modify | All edit tool changes (type, options, checks, error messages) |
| `vage/tool/edit/edit_tool_test.go` | Modify | New and updated unit tests |
| `vage/tool/read/read_tool.go` | Modify | Add `ReadTracker` recording |
| `vage/tool/read/read_tool_test.go` | Modify | Tests for tracker recording |
