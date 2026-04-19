# Code Review: Edit Tool Safety and Compliance Improvements

**Reviewer**: Claude Opus 4.6 (1M context)  
**Date**: 2026-04-05  
**Status**: Approved with one minor fix applied

## Summary

The implementation correctly follows the design document. All new features (UNC path blocking, deny rules, read prerequisite check, write-permission pre-check, improved error messages, int64 max file bytes) are implemented as specified. Tests are comprehensive and all pass. Lint reports zero issues.

## Files Reviewed

| File | Verdict |
|------|---------|
| `vage/tool/toolkit/path.go` | Good |
| `vage/tool/toolkit/path_test.go` | Good |
| `vage/tool/toolkit/readtracker.go` | Minor fix applied (see below) |
| `vage/tool/toolkit/readtracker_test.go` | Good |
| `vage/tool/edit/edit_tool.go` | Good |
| `vage/tool/edit/edit_tool_test.go` | Good |
| `vage/tool/read/read_tool.go` | Good |
| `vage/tool/read/read_tool_test.go` | Good |

## Fix Applied

### `readtracker.go`: Use `sync.RWMutex` instead of `sync.Mutex`

`HasRead` is a read-only operation that can safely run concurrently with other `HasRead` calls. Using `sync.RWMutex` with `RLock`/`RUnlock` for `HasRead` and `Lock`/`Unlock` for `RecordRead` allows concurrent readers without contention. This matters in agent workloads where multiple tool calls may check the tracker simultaneously.

**Changed**: `sync.Mutex` to `sync.RWMutex`, `mu.Lock()/Unlock()` in `HasRead` to `mu.RLock()/RUnlock()`.

## Detailed Review

### Correctness

- **UNC path blocking** (`path.go` lines 37-43): Correctly rejects `\\server\share` and `//server/share` patterns while allowing `///foo` (triple-slash). The ordering -- UNC check before `filepath.IsAbs` -- is correct because `\\` paths are not absolute on Unix but should still produce a UNC-specific error.
- **Deny rules** (`edit_tool.go` lines 143-145, 232-248): Uses `filepath.Match` against `filepath.Base(path)`, which is the right approach for basename-level patterns. Invalid patterns are gracefully skipped (no panic).
- **Read prerequisite** (`edit_tool.go` lines 147-149): Correctly uses the cleaned path for tracker lookup, matching what the read tool records. The nil check for optional tracker preserves backward compatibility.
- **Write permission check** (`edit_tool.go` lines 177-179): Checks owner write bit (`0o200`). This is a best-effort hint as documented. The actual write will fail with an OS error if permission is truly denied.
- **Read tracker recording** (`read_tool.go` lines 218-220, 247-249): Records in two places -- after truncation and after normal completion. Directory listings correctly skip recording (line 169 returns before the recording code).
- **Eviction** (`readtracker.go` lines 60-61): Simple clear-on-full strategy works correctly. The `>=` comparison ensures the map never exceeds `maxEntries`.

### Security

- No security issues found. The UNC blocking, deny rules, and read prerequisite all add defense-in-depth. Path traversal is handled by the existing `ResolveExistingPath` + allowed-dirs logic.
- The write-permission check is advisory only (TOCTOU between `os.Stat` and write), but this is documented and acceptable -- the actual write will fail with a proper OS error if needed.

### Performance

- The `RWMutex` fix (applied above) is the only performance concern. No other issues.
- The `matchDenyRule` function iterates patterns linearly, which is fine for the expected small number of deny rules.

### Code Quality

- Apache 2.0 license headers present on all files.
- Functional options pattern used consistently (`WithDenyRules`, `WithReadTracker`, `WithMaxFileBytes`).
- `context.Context` checked at handler entry.
- Error messages are actionable and include file paths.
- `matchDenyRule` is correctly kept private.
- `NewMemoryReadTracker` returns `*MemoryReadTracker` (concrete type), which is idiomatic Go -- callers can use it as `ReadTracker` interface where needed.

### Test Coverage

- **toolkit/path_test.go**: 7 tests covering UNC (backslash, slash, triple-slash), normal path, empty, relative, allowed-dirs (reject and accept).
- **toolkit/readtracker_test.go**: 5 tests covering unrecorded, recorded, concurrent access, eviction, and unlimited entries.
- **edit/edit_tool_test.go**: 26 tests covering all new features (deny rules: exact/glob/no-match, read prerequisite: blocked/allowed/no-tracker, read-only file, UNC paths) plus existing functionality.
- **read/read_tool_test.go**: 22 tests covering tracker recording for files and non-recording for directories, plus existing functionality.
- Race condition coverage via `TestEditTool_Concurrent`, `TestEditTool_ConcurrentSameFile`, `TestReadTool_Concurrent`, and `TestMemoryReadTracker_ConcurrentAccess`.

### Design Consistency

All implementation matches the design document. No deviations noted. The design's decision to put deny-rule tests in `edit_tool_test.go` (rather than a separate `deny_test.go`) was followed -- this is reasonable since `matchDenyRule` is private and the tests exercise it through the handler.

## Conclusion

Clean implementation with good test coverage. One minor improvement applied (RWMutex). No bugs, security issues, or design deviations found.
