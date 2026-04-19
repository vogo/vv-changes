# Code Review — 2026-04-18-001 path-guard

Reviewer: Claude Opus 4.7 (1M context)
Scope: implementation pass of the path-guard feature per `design.md`.

## Files Reviewed

### vage

- `vage/tool/toolkit/pathguard.go` (new)
- `vage/tool/toolkit/pathguard_test.go` (new)
- `vage/tool/toolkit/atomicwrite.go` (+`AtomicWriteInRoot`)
- `vage/tool/toolkit/path.go` (ValidatePath symlink-escape + CanonicalizeDirs — already split into pathguard.go)
- `vage/tool/read/read_tool.go` (+`WithPathGuard`, dual-mode handler)
- `vage/tool/write/write_tool.go` (+`WithPathGuard`, `writeViaGuard`)
- `vage/tool/edit/edit_tool.go` (+`WithPathGuard`, root-branch logic)
- `vage/tool/glob/glob_tool.go` (+`WithPathGuard`)
- `vage/tool/grep/grep_tool.go` (+`WithPathGuard`)
- `vage/tool/bash/pathguardian.go` (new)
- `vage/tool/bash/pathguardian_test.go` (new)
- `vage/tool/bash/bash_tool.go` (+`WithPathGuardian`, hard-block at execute)

### vv

- `vv/configs/config.go` (+`AllowedDirs *[]string`)
- `vv/setup/setup.go` (+`buildPathEnforcement` / `buildAllowedDirs`)
- `vv/registries/tool_access.go` (variadic `RegistryOption`, guard/guardian injection)
- `vv/cli/permission.go` (+`SetPathGuardian`, +`SetNonInteractive`, +merge logic)
- `vv/main.go` (wire non-interactive + guardian)

## Findings

### Blockers
None.

### Recommended

**R1. `PathGuard.matchRoot` returns non-canonical `abs`.**
`matchRoot` returns `p` (the input to matchRoot) as the `abs` result. For a literal match this is `cleaned`; for the `ResolveExistingPath` fallback this is `resolved`. The two can differ for the same underlying file (macOS `/var` vs `/private/var`, Linux bind-mounts via symlink). Callers downstream use `abs` as the display path AND as the lock key (edit tool's `toolkit.LockPath(cleaned)`), so two edits to the same underlying file via different ancestor forms would lock independently and could race. Fix: always return `filepath.Join(r.canonical, rel)` as `abs` — the canonical form rooted at the matched root.

**R2. Missing unit test for `buildAllowedDirs` nil-vs-empty semantics.**
Design §2.11 explicitly distinguishes nil (YAML key absent → defaults) from empty (`allowed_dirs: []` → startup error). Currently only covered implicitly via `New` tests with unset `Tools.AllowedDirs`. Adds an explicit `setup.buildAllowedDirs` test exercising nil → merge, empty → error, non-empty → verbatim.

**R3. `AtomicWriteInRoot` lacks a unit test.**
No coverage of rename-over-existing, symlink-target semantics, or tmp cleanup on write failure. Low-risk for now because write-tool and edit-tool tests exercise it end-to-end, but direct coverage would harden it.

### Nits

**N1. Design doc says "Go 1.26+" in §2.1 but `os.Root` landed in 1.24.** Doc-only; leaving as-is (not in scope for this code review — would be a design doc fix).

**N2. PathGuardian tokenizer treats every whitespace-separated bare token as a candidate for absolute-path classification.** This is the intended "universal" rule, but stderr-redirect tokens like `2` in `2> file` are also classified (they're not absolute, so Caution — harmless). No action needed.

**N3. `PathGuard.OpenForWrite` doesn't ensure the parent directory exists.** Write tool does this at the tool level via `MkdirAll`, so the helper stays minimal. Acceptable as-is.

### Pressure-test resolutions (confirmed OK)

- **Non-existent writes with normal parent**: `guard.Check` takes the literal path (not via ResolveExistingPath since the literal is inside allowed), then `root.Stat` returns ENOENT, then MkdirAll(parent), then AtomicWriteInRoot. ✓
- **Non-existent writes under symlinked ancestor going outside**: literal matchRoot passes (symlink path literally looks inside allowed), `root.Stat` / `root.MkdirAll` / `root.OpenFile` all refuse via `os.Root` because the canonical root opened at construction is distinct. `os.Root.OpenFile` returns a path-escape error; we surface it. ✓
- **macOS `/var` → `/private/var` false positives**: `ResolveExistingPath` fallback in `matchRoot` handles this. Test `TestPathGuard_Check_Accepts` implicitly exercises it on macOS because `t.TempDir()` yields `/var/folders/...`. Verified locally. ✓
- **Classifier + guardian merge order**: `tryClassifyBashMerged` picks max tier; on Blocked the earlier dialog branch hard-rejects. Dangerous in non-interactive fails; interactive prompts per-invocation without allow-always. Caution falls through to mode-dependent dialog. ✓
- **`SetPathGuardian` call-after-Init**: Not a race because program is still single-threaded at that point; registry path uses the guardian from `buildPathEnforcement` via `RegistryOption`, PermissionState uses the same instance returned on `setup.Result.PathGuardian`. Both observers see the same pointer. ✓
- **Read tool refactor double-close**: In guard path, `f` is unconditionally assigned; both IsDir and file branches defer-close or invoke close via helper. No double-close. In fallback path, only the file branch opens `f`; IsDir path calls the `listDirectory(cleaned)` helper that does its own ReadDir. Defer runs only if `f != nil`. ✓

## Applied Improvements

- **R1** applied: `matchRoot` now returns `filepath.Join(r.canonical, rel)` as `abs`, with a special case for `rel == "."`. Updated docstrings on `matchRoot` and `Check`.
- **R2** applied: added `setup.TestBuildAllowedDirs_*` — nil (merge defaults), empty (startup error), non-empty (verbatim), non-existent (error), filesystem root (rejected), tilde expansion, containment-dedupe behavior documented via `TestBuildAllowedDirs_NilContainmentDedupe`.
- **R3** applied: added `toolkit.TestAtomicWriteInRoot_*` for happy path (create), overwrite-existing, nested subdir, and permission preservation.

## Verification

- `go test $(go list ./... | grep -v integrations)` in both `vage/` and `vv/`: all pass.
- `make lint` in both `vage/` and `vv/`: 0 issues.
