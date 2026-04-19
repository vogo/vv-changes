# Test Report v2 — post-fix

## Summary

After applying the fix for `integrations-error-1-DONE.md` (hardened `PathGuard.Check` to always resolve symlinks and verify the resolved path stays inside the matched root), the integration suite `vage/integrations/tool_tests/pathguard_tests/` now passes in full.

## Results

- **48 of 48 cases pass.** 0 fails, 0 errors.
- Matrix: 6 tools × (4 rejection vectors + 4 legal vectors) — all rejection vectors correctly rejected, all legal vectors correctly succeed.

### Tool-by-tool

| Tool | Rejection cases | Legal cases |
|------|-----------------|-------------|
| read | 4/4 | 4/4 |
| write | 4/4 | 4/4 |
| edit | 4/4 | 4/4 |
| glob | 4/4 | 4/4 |
| grep | 4/4 | 4/4 |
| bash | 4/4 | 4/4 |

### Key fix

Reshaped `PathGuard.Check` (`vage/tool/toolkit/pathguard.go:232`) to always resolve symlinks unconditionally before containment matching. Previously the literal `matchRoot` check short-circuited for in-allowlist symlinks whose targets lay outside — fine for `read`/`write`/`edit` (caught later by `os.Root`), but a real escape for `glob`/`grep` which shell out to subprocesses. After the fix the resolved form must still land inside an allowed root; if the literal was inside and the resolved form is outside, the error string distinguishes the symlink escape (`"symlink resolves outside allowed directories"`).

## Unit + lint

- `make lint` in both `vage/` and `vv/`: 0 issues.
- `go test $(go list ./... | grep -v integrations)` in both modules: all green.

## AC Coverage

All five requirement user stories exercised:

- **US-1** startup log + guard injection — verified via `buildPathEnforcement` unit tests and integration fixture setup.
- **US-2** YAML override semantics — covered by `TestBuildAllowedDirs_*` in `vv/setup/setup_test.go` (nil, empty, non-empty, `~` expansion, bad path).
- **US-3** rejection scenarios — full 6×4 matrix.
- **US-4** error message format — assertions check for allowed-dirs list and tool-name prefix.
- **US-5** zero regression on legitimate paths — 24 legal-path cases pass; pre-existing vage tool unit tests pass.

## Conclusion

No outstanding failures. Ready to advance to documenter.
