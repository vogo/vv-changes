# Design Review: Edit Tool Safety and Compliance Improvements

## Overall Assessment

The design is well-structured, follows existing codebase conventions, and correctly identifies the handler pipeline ordering. The following items are improvement suggestions ranging from correctness issues to simplification opportunities.

## Issue 1: 1 GiB Default is Excessive and Introduces Memory Risk

**Severity:** High

The requirement asks to change the default max file size from 1 MB to 1 GiB. The current edit tool reads the entire file into memory as a `string`, performs `strings.Count` and `strings.Replace` on it, then writes the result back. For a 1 GiB file, this means holding at least 2-3 GiB in memory (original + replacement + intermediate copies from string operations).

The reference specification's 1 GiB constant is a hard ceiling, not necessarily a sensible default. The codebase is an agent framework where the edit tool is used by LLM agents performing small, targeted edits. Files approaching 1 GiB are not realistic targets for string-replacement-based editing.

**Recommendation:** Keep the default at a reasonable value such as 10 MB (`10 * 1024 * 1024`). Change the field type to `int64` as planned so that callers can opt into larger sizes via `WithMaxFileBytes`. Document that callers bear the memory cost when raising the limit.

## Issue 2: Deny Rules Belong on the Edit Tool, Not in toolkit

**Severity:** Medium

The design proposes a standalone `toolkit/deny.go` with `MatchDenyRule`. Currently no other tool (read, write) needs deny-rule matching. Creating a shared utility for a single consumer is premature abstraction.

The `filepath.Match` function has limited glob semantics (no `**` support). The design's test case for `**/credentials.json` will not work with `filepath.Match` -- that function does not support recursive-directory globs. This is a correctness bug in the design.

**Recommendation:** Keep deny-rule matching as a private helper inside `tool/edit/`. Use `filepath.Match` against `filepath.Base(path)` for all patterns. Drop `**` prefix support from the initial scope; document that patterns match against the filename only. If full-path glob matching is needed later, use `doublestar` or a similar library. This avoids a broken feature.

## Issue 3: ReadTracker Creates Tight Cross-Tool Coupling

**Severity:** Medium

The `ReadTracker` interface and `MapReadTracker` are placed in `toolkit`, and both the read tool and edit tool must be wired with the same instance. This creates implicit coupling: the two tools must be constructed together with a shared object, yet there is no compile-time or structural enforcement of this. Forgetting to wire the tracker into the read tool silently disables the safety feature while the edit tool happily rejects everything.

Additionally, `MapReadTracker` uses a simple `map[string]bool` with no eviction. In long-running agent sessions, this map grows unboundedly.

**Recommendation:**
- The `ReadTracker` interface definition in `toolkit` is fine (it is the shared layer). The `MapReadTracker` implementation is also fine there.
- Add a `maxEntries` parameter or use an LRU-style eviction (even a simple "clear when full" strategy) to bound memory. Alternatively, document that the tracker is scoped to a single agent session and should not be reused across sessions.
- Consider renaming `MapReadTracker` to `MemoryReadTracker` for clarity.
- The design should note clearly that the tracker is optional and both tools must share the same instance for the feature to work. This is already implied but should be explicit.

## Issue 4: Write Permission Check is Incomplete on Unix

**Severity:** Low

The design checks `info.Mode().Perm()&0o200 == 0` (owner write bit). This is incorrect when the process runs as a non-owner user who has write access via group or other bits, or when ACLs grant access. Conversely, it will pass for a file owned by another user where the owner write bit is set but the current process has no access.

The design acknowledges this ("A more thorough check could use `unix.Access`") but dismisses it for portability. However, `unix.Access` is not needed. Go's `os.OpenFile` with `O_WRONLY` is the canonical portable check, but it has side effects (modifying atime, potentially creating the file).

**Recommendation:** Since the edit tool already attempts to write (via `AtomicWriteFile`), and a permission failure there will produce a clear error from `os.CreateTemp` in the same directory, the pre-check adds marginal value. Keep it simple: check the owner write bit as proposed, but frame the error message as a hint rather than a guarantee. Change the message to: `"edit tool: file appears to be read-only (mode %s): %s"` -- including the mode helps the agent understand why.

## Issue 5: UNC Path Check Placement

**Severity:** Low

The design adds the UNC check to `ValidatePath` before `filepath.IsAbs`. This is correct. However, `//` is a valid path prefix on some Unix systems (POSIX allows implementation-defined behavior for paths starting with `//`). The check should be narrowed: only reject if the path starts with `//` followed by a non-`/` character (i.e., `//server/...` but not `///foo`).

Actually, looking more carefully at the existing `ValidatePath`: on Unix, `//server/share` passes `filepath.IsAbs` (it starts with `/`), and `filepath.Clean` collapses `//` to `/`, which would bypass the allowed-dirs check. So the UNC check does provide genuine value on Unix as well, not just Windows.

**Recommendation:** Check for `strings.HasPrefix(path, "\\\\")` and for paths matching `//` followed by a non-`/` character. Use a simple condition: `strings.HasPrefix(path, "//") && len(path) > 2 && path[2] != '/'`. This avoids false positives on paths like `///absolute/path` which `filepath.Clean` handles correctly.

## Issue 6: New Files in toolkit May Be Overkill

**Severity:** Low

The design creates three new files in `toolkit/`: `readtracker.go`, `readtracker_test.go`, `deny.go`, `deny_test.go`, plus a `path_test.go`. Five new files for what amounts to a small interface, a thin map wrapper, and a one-function utility.

**Recommendation:**
- `readtracker.go` in `toolkit` is justified -- the interface is shared between read and edit.
- Move deny-rule logic into `tool/edit/` as a private function (per Issue 2).
- `path_test.go` in `toolkit` is justified -- `ValidatePath` is shared and deserves its own tests.
- Net result: two new files in `toolkit` (`readtracker.go`, `path_test.go`) plus their tests, and a private helper in `edit`. This is simpler.

## Issue 7: Missing Consideration for the Write Tool

**Severity:** Low

The requirement explicitly scopes out the write tool, but the write tool has the same `maxWriteBytes` type (`int`) and lacks UNC path protection. The UNC check is added to `ValidatePath` which the write tool already calls, so it gets UNC protection for free. But the design should note this as a positive side-effect.

The write tool could also benefit from deny rules and the read-prerequisite check (enforcing that a file is read before overwriting). This is out of scope but worth noting as future work.

## Issue 8: Test Plan Section 4 is Redundant

**Severity:** Low

Section 4 ("Integration Test Plan") describes scenarios that are all unit-testable and are already covered by Section 2.9/2.10. The section then concludes "these can all be run as standard Go tests (no LLM keys needed) and should be placed as unit tests colocated with source." This contradicts the section title and adds no value beyond what is already specified.

**Recommendation:** Remove Section 4 or fold it into the test specifications in Section 2.

## Summary of Recommended Changes

| # | Issue | Action |
|---|-------|--------|
| 1 | 1 GiB default is dangerous | Default to 10 MB, keep int64 type for opt-in |
| 2 | Deny rules in toolkit premature | Move to edit package, drop `**` support |
| 3 | ReadTracker coupling and unbounded growth | Add size bound or session-scope guidance |
| 4 | Write permission check incomplete | Keep simple check, improve error message with mode |
| 5 | UNC `//` false positive | Narrow the `//` check to exclude `///...` |
| 6 | Too many new toolkit files | Consolidate deny logic into edit package |
| 7 | Write tool gets UNC for free | Note as positive side-effect |
| 8 | Redundant integration test section | Remove or fold into unit test section |
