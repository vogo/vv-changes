# Integration Error Report 1 — glob/grep symlink escape not rejected

## Summary

Two integration cases in `vage/integrations/tool_tests/pathguard_tests/pathguard_test.go` fail because the `PathGuard.Check` method does not detect an in-allowlist symlink that points outside the allow-list. For `read` / `write` / `edit` the escape is still caught atomically at file-open time by `os.Root`; for `glob` / `grep` the guard's Check is the only gate, so the escape slips through.

## Failing cases

| # | Test | Scenario |
|---|------|----------|
| 1 | `TestGlob_Reject/symlink_outside` | `glob` with `path = <allowed>/escape` where `escape -> <outside>` |
| 2 | `TestGrep_Reject/symlink_outside` | `grep` with `path = <allowed>/escape` where `escape -> <outside>` |

### Observed output (from `go test -v ./integrations/tool_tests/pathguard_tests/...`)

```
=== RUN   TestGlob_Reject/symlink_outside
    pathguard_test.go:621: [glob/symlink_outside] expected IsError=true, got success with text: ""
--- FAIL: TestGlob_Reject/symlink_outside

=== RUN   TestGrep_Reject/symlink_outside
    pathguard_test.go:731: [grep/symlink_outside] expected IsError=true, got success with text: "No matches found."
--- FAIL: TestGrep_Reject/symlink_outside
```

Both tools accept the symlink path as if it were inside the allow-list, and the underlying subprocess (`find` / `grep`) then transparently follows the symlink and operates on the outside directory.

## Root cause

`toolkit.PathGuard.Check` (vage/tool/toolkit/pathguard.go:232) performs the allow-list containment check on the **literal cleaned path** first (via `matchRoot`). For `<allowed>/escape`, `filepath.Rel(allowed, <allowed>/escape)` returns `"escape"` — a non-`..` relative path — so `matchRoot` reports success before any symlink resolution runs.

The `ResolveExistingPath` fallback on lines 251–256 is only invoked when the literal path **does not match any root**. For an in-allowlist symlink, the literal path always matches, so the fallback branch never runs and the symlink's target is never compared against the allow-list.

For `read` / `write` / `edit` this gap is masked because those tools subsequently call `root.Open(rel)` / `root.Stat(rel)`, and Go's `os.Root` methods reject symlinks that resolve outside the root atomically (openat2(RESOLVE_BENEATH) on Linux; emulation elsewhere — still refuses). For `glob` / `grep` the guard is the **only** check: the tool resolves `dir = filepath.Clean(path)`, passes the guard, and then executes a subprocess in that directory. The subprocess follows the symlink and returns whatever lives outside.

This contradicts the design text in §2.6 which claims `guard.Check("glob", dir)` "also rejects symlink escape".

## Affected design ACs

- US-3 (glob/grep): "`path` 参数不在白名单 → 拒绝". A symlink whose target is outside the allow-list is semantically "not in whitelist" even though its literal path sits inside.
- Design §5.4 matrix explicitly calls out "symlink inside allowed pointing outside" as a mandatory rejection vector.

## Suggested fix approaches

Pick one — listed in increasing cost:

1. **(Smallest change, recommended.)** In `PathGuard.Check`, always invoke `ResolveExistingPath(cleaned)` and, when the resolved form differs from the cleaned literal **and is not under any root**, reject with the existing `symlink resolves outside allowed directories` error message used by `toolkit.ValidatePath`. Concretely: after the `matchRoot(cleaned)` success branch, also call `ResolveExistingPath(cleaned)` and verify its result still lands inside the matched root; if not, reject. This costs one extra stat per call only when a symlink is present along the path — for non-symlink paths `EvalSymlinks` short-circuits on the same inode.

2. Alternatively, perform the stricter check only inside the glob/grep handlers: after `guard.Check("glob", dir)` succeeds, call `filepath.EvalSymlinks(dir)` and re-run containment. Keeps read/write/edit on their current fast path (os.Root already covers them). Down-side: duplicates logic across glob/grep; easy to forget.

3. Extend `PathGuard` with a dedicated `CheckStrict(toolName, p string)` variant that always resolves symlinks. Wire glob/grep to `CheckStrict`; leave read/write/edit on `Check`. Explicit at the call-site, but grows the API surface.

Approach (1) is the smallest source change and preserves a single code path. Expected error string (matching existing convention in `ValidatePath`): `"<tool> tool: symlink resolves outside allowed directories: <path> (allowed: [...])"`.

## Test status after fix

With approach (1) the two failing cases should flip to PASS without any test file change — the assertion substring `"path not allowed"` matches the generic "outside" branch; if the fix uses the distinct `"symlink resolves outside"` message, loosen the test substring assertion to `"tool"` (already used for the read/write/edit symlink cases in this same file).

## Repro

```
cd vage
go test ./integrations/tool_tests/pathguard_tests/... -run 'TestGlob_Reject/symlink_outside|TestGrep_Reject/symlink_outside' -v
```
