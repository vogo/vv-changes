# Path-Guard Integration Test Report — v1

**Suite:** `vage/integrations/tool_tests/pathguard_tests/pathguard_test.go`
**Run command:** `go test ./integrations/tool_tests/pathguard_tests/... -v` (from `vage/`)
**Platform covered:** Unix (`//go:build !windows`; mirrors `bash_tests` convention)

## Matrix overview

Per design §5.4, the suite covers 6 tools × 4 escape vectors (24 rejection cases) plus 24 matching legal/zero-regression cases.

| Tool  | Rejection vectors                                                                 | Legal counterparts                                                                    |
|-------|-----------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| read  | literal `..`; absolute outside; symlink outside; empty path                        | absolute inside; directory listing; offset+limit; nested subdir                        |
| write | literal `..`; absolute outside; symlink outside; empty path                        | create new; overwrite; nested MkdirAll; `create_only=true`                             |
| edit  | literal `..`; absolute outside; symlink outside; empty path                        | single replacement; `replace_all`; nested subdir; delete-via-empty-new-string          |
| glob  | literal `..`; absolute outside; symlink outside; `..` in pattern                   | `*.txt` at root; `**/*.go` recursive; default workdir (no path); prefix pattern        |
| grep  | literal `..`; absolute outside; symlink outside; empty path (no workdir)           | match; no-match-is-success; default workdir; include filter                            |
| bash  | `..` token (Dangerous, via `guardian.Classify`); `cat /etc/passwd` (Blocked); `cd /etc` (Blocked); `echo $(cat /etc/passwd)` (Blocked via substitution) | `echo hello`; `cat <inside>`; `cd <inside>`; relative arg                              |

Note on bash: per task guidance, only TierBlocked classifications are asserted via the registered tool's hard-block (the BashTool itself rejects only Blocked; Dangerous is left to PermissionState which is absent from this test wiring). TierDangerous cases are asserted by calling `guardian.Classify(cmd)` directly.

## Result summary

| Bucket      | Count | PASS | FAIL |
|-------------|-------|------|------|
| Rejection   | 24    | 22   | 2    |
| Legal       | 24    | 24   | 0    |
| **Total**   | 48    | 46   | 2    |

### Failing cases

1. `TestGlob_Reject/symlink_outside` — glob accepts an in-allowlist symlink whose target is outside; subprocess transparently follows it.
2. `TestGrep_Reject/symlink_outside` — same class of escape on grep.

Both are caused by the same implementation gap in `toolkit.PathGuard.Check`: when the literal cleaned path lands inside an allowed root, `matchRoot` reports success before any symlink resolution runs, so a symlink-to-outside inside the allow-list is never rejected. `os.Root` masks this gap for read/write/edit (file-open is atomically refused); glob/grep execute in a subprocess and have no such secondary layer.

Details, design-AC impact, and three candidate fix approaches are in `integrations-error-1.md`. Suggested fix is the minimal extension of `Check` to resolve symlinks on success and re-verify containment.

## AC coverage map (requirement §2)

| AC | Coverage |
|----|----------|
| US-1 default allowed dirs, 6-tool uniform injection | Suite injects `PathGuard` into all 6 registered tools; every rejection case asserts enforcement from the registry boundary. |
| US-2 YAML extension, canonicalize | Canonicalization exercised indirectly via `NewPathGuard` (which calls `CanonicalizeDirs`); unit coverage for YAML parsing stays in `vv/configs`. |
| US-3 read/write/edit — reject `/etc/passwd`, `../etc/passwd`, symlink escape | All three passing on read/write/edit; symlink-escape for glob/grep failing (see error-1). |
| US-3 glob/grep — reject path outside allow-list | Passing for literal `..`, absolute outside, and `..`-in-pattern. Failing for symlink-inside-pointing-outside. |
| US-3 bash — `cd`, command substitution, absolute-outside, `..` | All four passing. `cd /etc` Blocked; `cat /etc/passwd` Blocked; `echo $(cat /etc/passwd)` Blocked via inner-token extraction; `ls ../../etc` reports rule `path-traversal-dots` (Dangerous). |
| US-4 error messages include tool name + reason + allow-list | Every rejection assertion checks a descriptive substring (`"path not allowed"`, `"must not be empty"`, rule names, etc.). Allow-list truncation itself is covered by unit tests in `toolkit/pathguard_test.go`. |
| US-5 zero regression | All 24 legal cases pass — creating, overwriting, reading, editing, listing, grepping, globbing, and bash-executing inside the allow-list all succeed end-to-end. |

## Edge cases surfaced

- **Implementation gap for glob/grep symlink rejection** — the `os.Root` rail that protects read/write/edit does not extend to glob/grep, and `PathGuard.Check` does not resolve symlinks when the literal path is inside an allowed dir. See `integrations-error-1.md`.
- **Bash `cd <inside absolute>` passes without bump** — confirmed not to fire `cd-outside-allowed`; matches the design row "cd <literal> inside allowed" (no bump).
- **Bash command substitution with an inside path** was not added to the legal suite because the design classifies inner tokens recursively; the existing `cat /etc/passwd` extraction check is sufficient evidence of the mechanism.
- **`grep` with no path and no workingDir** returns a distinct "no search path provided" error — used as grep's tool-specific 4th vector, matching requirement US-3 implied behavior.
- **Windows coverage** intentionally skipped via `//go:build !windows` (matches `bash_tests`); guard surface checks for UNC / `\\?\` / short-name are covered in `toolkit/pathguard_test.go` unit tests.

## Files delivered

- `/Users/hk/workspaces/github/vogo/vagents/vage/integrations/tool_tests/pathguard_tests/pathguard_test.go` — 48-case integration suite (this report's subject).
- `/Users/hk/workspaces/github/vogo/vagents/changes/2026/04/18/2026-04-18-001-path-guard/integrations-error-1.md` — structured error report for the two failing cases.
- `/Users/hk/workspaces/github/vogo/vagents/changes/2026/04/18/2026-04-18-001-path-guard/test-report-v1.md` — this report.

## Recommendation

Ship the fix proposed as approach (1) in `integrations-error-1.md` — resolve symlinks inside `PathGuard.Check` after a successful literal match and re-verify containment against the matching root. Once applied, the two failing cases should flip to PASS with no test changes (error substring already accepts the generic "path not allowed" phrasing).
