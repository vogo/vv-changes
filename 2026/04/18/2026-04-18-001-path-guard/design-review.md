# Design Review — Working-directory Allow-list + Path-Escape Guard

Pressure-tested against `os.Root` docs (Go 1.26), the existing tool code in `vage/tool/*`, the CLI/HTTP wiring in `vv/main.go` + `vv/cli/permission.go`, and the requirement acceptance criteria.

Severity legend: **[B]** Blocker (must change), **[R]** Recommended (strongly advised), **[N]** Nit.

---

## [B1] HTTP mode silently allows Dangerous bash commands

**Concern.** `vv/main.go` constructs one `permissionState` and shares it across CLI and HTTP. In HTTP mode, `interactor = askuser.NonInteractiveInteractor{}` and `confirmFn` stays at its default (returns `PermissionAllow`). The raw design makes BashTool hard-block `TierBlocked`, but defers `TierDangerous` to `PermissionState`. Net result in HTTP mode: `cat /workdir/../../etc/passwd` → Dangerous → `confirmFn` → allow. That contradicts AC-3 ("must be rejected, whatever the form").

**Proposed change.** Add an HTTP-mode policy: when `cfg.Mode == "http"`, treat `TierDangerous` classifications (from classifier OR guardian) as hard-reject in `permissionExecutor.Execute`. Either:
- `PermissionState` exposes `WithDangerousPolicy(DangerousHardBlock | DangerousConfirm)`, set to `DangerousHardBlock` in HTTP mode; or
- simpler: pass a mode flag into `NewPermissionState(mode, isNonInteractive bool)` and short-circuit Dangerous to `ErrorResult` when `isNonInteractive`.

**Rationale.** Without this, the "symbolic path escape via `..`" class is classified Dangerous by the guardian, but HTTP users receive zero protection — they get a weaker guarantee than CLI for no good reason.

---

## [B2] Explicit empty `allowed_dirs: []` must disable defaults (requirement says so)

**Concern.** Requirement AC US-2 says "用户配置的值 **追加** 到默认白名单上，而非替换（**除非显式传入空数组**）". The raw design's `buildAllowedDirs` unconditionally prepends `{WorkingDir, TempDir}`, so `allowed_dirs: []` becomes `[WorkingDir, TempDir]` — the "explicit empty disables defaults" contract is violated.

**Proposed change.** Use a pointer (`*[]string`) or a sentinel in YAML so "unset" vs "explicit empty" are distinguishable, then:
- `cfg.Tools.AllowedDirs == nil` → merge defaults.
- `cfg.Tools.AllowedDirs != nil && len(...) == 0` → user opted out of defaults → error out at startup (because guard with zero roots == "open policy" would be surprising; require at least one root to enable enforcement).
- `cfg.Tools.AllowedDirs != nil && len(...) > 0` → use exactly those (no auto-merge of TempDir / WorkingDir unless user included them).

**Rationale.** Matches the requirement contract; avoids the silent "user thinks they restricted, but we quietly added their home dir back" surprise.

---

## [B3] Reject `/` (or any root-of-filesystem) allowed-dir as a config error

**Concern.** If a user writes `allowed_dirs: ["/"]`, `os.OpenRoot("/")` succeeds and every path trivially falls under it — the guard is now a no-op without any warning. Same issue for `$HOME` on a single-user box.

**Proposed change.** In `CanonicalizeDirs`:
- Reject `/` (Unix) and drive roots (`C:\`, `D:\` on Windows) with an explicit error.
- On startup, log (WARN) if any dir is a user's home dir or an ancestor of it — not rejected, but visible.

**Rationale.** Fail-fast on a config value that silently disables the feature is required by the "no silent degradation" decision already recorded in §7 of the raw design.

---

## [B4] Overlapping allowed_dirs: deduplicate containment, not just exact strings

**Concern.** The raw design says "dedupe + canonicalize". If user gives `["/work", "/work/sub"]`, naive string-dedupe keeps both, but `/work/sub` is redundant (anything there matches `/work`). More importantly, `Check`'s first-match-wins walk has undefined order, and pre-opening two `os.Root`s for nested dirs doubles the fd and increases confusion in error messages.

**Proposed change.** After canonicalize, sort by length ascending and drop any dir that is a descendant of an earlier entry (via `filepath.Rel`). Document the rule: "allowed_dirs are automatically collapsed; the broadest ancestor wins."

**Rationale.** Deterministic behavior; fewer fds held; error messages reference the canonical owning root.

---

## [B5] Error messages must expose allowed_dirs for LLM self-correction

**Concern.** Raw design §3 says errors include the *rejected* path but not the allowed_dirs list — "不含 allowed_dirs 列表，避免刷屏". But the LLM caller is the tool consumer. If it gets `read tool: path not allowed: /etc/passwd`, it has no way to know where it *is* allowed to read and will probably retry with another variant of the same forbidden path. The paths in the allow-list are not secret — they are the user's project directory and `/tmp`.

**Proposed change.** Include the allowed dirs in the error. Format:

```
read tool: path not allowed: /etc/passwd (allowed: [/home/me/proj, /tmp])
```

Cap the list at e.g. 3 entries with `... +N more` if longer, to keep the line tractable. Keep the startup debug log with the full list.

**Rationale.** Usability for the agent loop; security cost is ~zero because the paths are already CWD + TempDir by default. Requirement AC US-4 even calls for clear error information for diagnostics.

---

## [B6] Bash `cd` edge cases must be specified

**Concern.** The raw design says "`cd <target>` outside → Blocked" but glosses over the cases:
- `cd` with no arg → HOME
- `cd -` → OLDPWD (not observable)
- `cd $VAR` / `cd "${FOO}"` → variable-expansion unknown
- `cd ~` / `cd ~user`
- `cd` followed by a glob

**Proposed change.** Specify per-case policy explicitly:
| Pattern | Tier | Rule |
|---|---|---|
| `cd` / `cd ~` / `cd $HOME` | Dangerous | `cd-home-ambiguous` (HOME may be outside allowlist) |
| `cd -` | Dangerous | `cd-prev-unknown` |
| `cd <literal-path>` inside allowed | Caution | passes |
| `cd <literal-path>` outside allowed | Blocked | `cd-outside-allowed` |
| `cd $VAR` / `cd "$(...)"` | Dangerous | `cd-variable` (cannot resolve statically) |
| `cd <glob>` | Dangerous | `cd-glob` |

**Rationale.** Makes test matrix complete and precludes "forgot an edge case" bugs.

---

## [B7] Glob pattern canonicalization is insufficient

**Concern.** Current `glob_tool.go` already rejects absolute patterns and `..` in `pattern`. But the *combination* is unchecked: `path=/allowed/inner/dir`, `pattern=*/../../etc/passwd`. After the literal `..` check, note the existing check also catches this in the pattern, so it does. BUT: the design only calls for "pattern 校验不变". We should also verify the resolved search directory (dir + pattern root components) stays inside the guard. Since glob's find/PowerShell subprocesses run with `cmd.Dir = dir`, actual escaping requires `..` *in the pattern*, which is already blocked. Verified OK.

However, grep has a similar issue (`include` filters, `path` arg): verify `grep_tool.go`'s include patterns are similarly sanitized. Also, `git ls-files` in `glob_tool.go` respects `.gitignore` but doesn't bound to `path` — verify git mode honors the bounded dir.

**Proposed change.** Add to §2.6/§2.7: explicitly note that `pattern` validation (no `..`, no absolute) already covers pattern-side escape; additionally, in grep, apply the same validation to `include` / `exclude` filters if they exist.

**Rationale.** Avoids future regressions; makes the existing safety explicit.

---

## [R8] Don't pre-open roots at construction — open-on-use with cache

**Concern.** Raw design §2.1 pre-opens all `os.Root`s at construction and keeps the fds for program lifetime. Fine on Linux/macOS, but:
- On a subagent-heavy workload, the guard is the same singleton; no per-request leak.
- On plan9/js the `os.Root` "references a directory name, not a file descriptor" per docs — that's a portability caveat, not a concurrency bug, but worth noting.
- If the allowed dir is deleted/renamed mid-session, pre-opened root keeps pointing at the (unlinked) inode on Linux. Subsequent `Open` inside silently succeeds on the ghost dir. Fresh-open-per-call would fail loudly.

**Proposed change.** Keep pre-open but document: "Allowed directories are assumed stable for the process lifetime. Deleting an allowed dir mid-session is undefined behavior. Guard does not detect rename after construction." Alternatively, open-on-use with an `sync.Map` of canonical-path → `*os.Root` (closed at shutdown); the open cost is a single `openat(O_DIRECTORY)` — microseconds.

**Rationale.** Pre-open is simpler and correct for the expected use case; just write down the assumption.

Also: confirm `os.Root` is safe for concurrent use — **yes, per docs: "Methods on Root are safe to be used from multiple goroutines simultaneously."** The design's `PathGuard` also needs to be safe (it's just a slice read; read-only after construction, so trivially safe).

---

## [R9] Acknowledge the legitimate-symlink limitation

**Concern.** `os.Root` rejects any symlink whose target is outside the root, per docs: "Methods on Root will follow symbolic links, but symbolic links may not reference a location outside the root. Symbolic links must not be absolute." Common real-world cases that break:
- `vendor -> /some/gopath/cache/...` (rare in Go these days but possible)
- `node_modules/.bin/*` on some platforms
- macOS: `/tmp -> /private/tmp` (this one is fine because `/tmp` is canonicalized before constructing the root)
- Symlinked project checkouts (e.g., dotfiles repos)

**Proposed change.** Add a §7 risk row: "Symlinks that cross the allowlist boundary are rejected by design. Users hitting legitimate cases should add the symlink target to `allowed_dirs` explicitly." Keep this behavior — don't add an opt-in to follow-outside. The one subtlety to document: macOS system dirs like `/tmp`, `/var` are themselves symlinks into `/private/...`; `CanonicalizeDirs` already `EvalSymlinks`es the allow-list entry, so that case works naturally. Add an integration test.

**Rationale.** Documents a known real-world gotcha rather than pretending it doesn't exist.

---

## [R10] PathGuardian needs tight scope on Absolute-Path rule

**Concern.** Raw design §2.8 rule #2 matches "subsequent tokens starting with `/`" for specific commands. Problems:
- `grep -rn 'foo' /allowed/path` — token `/allowed/path` IS absolute but IS inside allowed. Rule says check membership, so this passes. Good.
- `cat /etc/passwd 2>/dev/null` — token `2>/dev/null` — redirection target handling ambiguous; the tokenizer strips redirects but the rule should be explicit.
- `echo foo > /etc/motd` — redirection TARGET must be checked, not just command-allow-listed.
- Many commands not in the list (e.g., `perl`, `python`, `node`, `ruby`, `tar`, `zip`, `ssh`, `scp`, `rsync`) accept absolute paths and execute arbitrary code.

**Proposed change.**
1. Apply the absolute-path-outside-allowed check **universally** (not just for an allow-list of commands). The rule is "any bare argument or redirect target that is an absolute path outside the allowlist is Blocked." The command-allow-list is a distraction — `unknown-command /etc/passwd` should still be blocked.
2. Parse redirect operators (`>`, `>>`, `<`, `2>`, `&>`) as path-producing too. This is already a tokenizer pass, so the tokenizer should emit `(token, role)` where role is `cmd | arg | redirect-target`.
3. Keep `/proc`, `/sys`, `/dev` unconditional Blocked regardless of allowlist.

**Rationale.** An allow-list of 20 commands is always incomplete; inverting to "any absolute path must be in allowed_dirs" is simpler and safer.

---

## [R11] Windows support: explicit policy, not "emulation handles it"

**Concern.** `os.Root` on Windows uses emulation (no `openat2`). UNC paths (`\\server\share`), drive-relative paths (`C:foo` — which resolves relative to the drive's CWD), `\\?\` extended paths, and 8.3 short names are all potential bypasses. Current `ValidatePath` rejects `\\` prefix but not the others.

**Proposed change.** In `PathGuard.Check`:
- Reject on Windows: paths starting with `\\?\`, `\\.\`, `\\server\`, drive-relative forms (`C:` without a subsequent separator), and paths containing `~` short-name markers (detect via `syscall.GetLongPathName` canonicalization).
- Add a unit test file `pathguard_windows_test.go` with these vectors.
- Acknowledge in §7 that `RESOLVE_BENEATH` does not exist on Windows — the guard relies on Go's emulation, which is best-effort.

**Rationale.** P0 security feature shouldn't have "Windows is emulated" as the only defense.

---

## [R12] `WithDenyRules` × `PathGuard` composition — document the ordering

**Concern.** Edit tool has `WithDenyRules` for basename patterns (`*.env`, `credentials.json`). With PathGuard, both apply. If a file is inside allowed_dirs AND matches a deny rule, it must be rejected by the deny rule, not silently readable.

**Proposed change.** In §2.5, state explicitly: "Evaluation order is (1) PathGuard.Check → (2) matchDenyRule → (3) readTracker. PathGuard is the cheapest hard-reject; deny-rules are a secondary basename filter; readTracker is the usability rail. Both must pass for edit to succeed. If either fails, return its specific error."

Also: note the read tool does NOT have `WithDenyRules` today. Consider whether secrets-shaped files should be deny-listed at read too, or leave that to a future task. (Leave for now.)

**Rationale.** Readers of future code need the interaction documented; otherwise someone will "simplify" by removing deny rules believing the guard subsumes them.

---

## [R13] Fallback semantics — don't silently disable with empty guard

**Concern.** Raw design §1 says "未注入时守门员为空策略（open，允许一切），向后兼容 vage 单元测试". This is correct for the vage library itself (a third-party Go user may not want this feature). But in vv we should *assert* a guard is present so a future refactor that accidentally drops the registration doesn't silently lose protection.

**Proposed change.** Add to `vv/setup/setup.go` a post-registry assertion:
```go
if result.pathGuard == nil || !result.pathGuard.Allowed() {
    return nil, fmt.Errorf("internal error: path guard not installed")
}
```
Library-level (vage) behavior stays "no guard = open"; vv requires guard.

**Rationale.** Defense in depth against refactor regressions.

---

## [R14] Preserve the existing `WithAllowedDirs` public API bitwise

**Concern.** Tests in `vage/integrations/tool_tests/{read,write,edit,glob,grep}_tests` use `WithAllowedDirs`. Raw design §2.6 says vv switches to `WithPathGuard` but `WithAllowedDirs` remains "for tests". Make sure the behavior of `WithAllowedDirs(dirs ...)` alone (without a PathGuard) is **byte-identical** to today — currently it uses `ValidatePath`. If we silently upgrade it to use `os.Root` internally, tests relying on symlink behavior may flap.

**Proposed change.** Explicit in §2.3/§2.4/§2.5:
- `WithAllowedDirs(dirs)` on its own keeps the pre-feature behavior (uses `ValidatePath` + `os.Open` / `os.ReadFile`). No `os.Root` involvement.
- Only `WithPathGuard(g)` activates the `os.Root` path.
- If both are passed, `WithPathGuard` wins (guard is stricter).
- Library users not calling either retain no restrictions.

**Rationale.** Byte-level compatibility with existing test suite; zero regression cost for vage consumers outside vv.

---

## [R15] `ResolveExistingPath` for non-existent write targets — specify the contract

**Concern.** For `write` creating a new file at `/allowed/newfile.txt`, the file does not yet exist, so `EvalSymlinks` fails and we fall back to `ResolveExistingPath` (walking up to the parent). What if parent doesn't exist either (`MkdirAll` case)? What if an ancestor is a symlink pointing outside? Current code walks all the way up; raw design doesn't clarify the new PathGuard approach here.

**Proposed change.** Specify: for writes, `PathGuard.OpenForWrite` resolves the longest existing prefix, requires that prefix to canonicalize inside a root, and then uses `Root.MkdirAll` + `Root.OpenFile(O_CREATE)` for the relative remainder. If any component of the remainder would create a path outside via symlink, `os.Root` rejects atomically. Add test case: "create file in nonexistent subdir under allowed root → success; same but nonexistent subdir is a dangling symlink pointing outside → rejected."

**Rationale.** Closes ambiguity in the write-new-file flow; `os.Root.MkdirAll` is the right primitive and the docs confirm it honors root containment.

---

## [R16] Plan mode + search tools need the same guard

**Concern.** In Plan mode, read-only tools (read/glob/grep) are always permitted. With guard installed, they're bounded by allowed_dirs, which is the correct behavior. Sanity-check in permission.go is unaffected. Verified OK.

No change needed — flagging that it was checked.

---

## [N17] Minor: don't call it `SafeFS` if you're not building an `fs.FS`

**Concern.** Raw design §3 (scope section of requirement) alludes to `toolkit.SafeFS`, but the design uses `PathGuard` with explicit open methods. Pick one name and stay with it.

**Proposed change.** Use `PathGuard` throughout; drop the `SafeFS` term from the requirement or mark it as a renamed concept.

---

## [N18] Default: add `/tmp` even if `TMPDIR` is overridden

**Concern.** `os.TempDir()` respects `$TMPDIR`. If a user set `TMPDIR=/some/ramdisk` at launch, `TempDir()` returns that. Intent probably is "the system temp — whatever it may be". Fine. Just note it in the comment on `buildAllowedDirs`.

**Proposed change.** Comment: `// os.TempDir() resolves $TMPDIR or OS default; users can override via env before launch.`

---

## [N19] Startup log line

**Concern.** AC US-1 says the CLI startup log should show the allowed_dirs. Make this an INFO-level line, not DEBUG, so users actually see it on first run.

**Proposed change.** `slog.Info("vv: allowed_dirs active", "dirs", canonicalDirs)` in `setup.Init`.

---

## [N20] Test file naming

**Concern.** Raw design §5.4 says `vage/integrations/tool_tests/pathguard_tests/`. The existing pattern is `<tool>_tests/`. The cross-tool nature is useful, but consider splitting per-tool-plus-adding a `crosscutting_tests/` for the escape-vector matrix.

**Proposed change.** Keep `pathguard_tests/` as a single integration suite (the cross-cutting matrix belongs together). Note this is an exception to the per-tool naming convention.

---

## Things the raw design got right (verified, not just skipped)

- **`os.Root` method surface is correct.** Go 1.26 `os.Root` exposes `OpenRoot`, `Open`, `Create`, `OpenFile`, `Stat`, `Lstat`, `Rename`, `MkdirAll`, `Close`, `FS`, `ReadFile`, `WriteFile`, `Remove`, `RemoveAll`, `Readlink`, `Symlink`, `Link`, `Chmod`, `Chown`, `Chtimes`. All APIs the design plans to use exist.
- **`os.Root` is goroutine-safe.** Docs confirm: "Methods on Root are safe to be used from multiple goroutines simultaneously."
- **Keeping glob/grep on `ValidatePath`** — correct choice; they shell out to `find` / `rg`, where `os.Root` cannot bound the subprocess.
- **Fail-fast on guard construction error** (no silent degradation) — correct, keep.
- **Default whitelist = WorkingDir + TempDir** — matches requirement.
- **Splitting classifier and guardian** — right separation of concerns; classifier is command-pattern, guardian is path-aware.
- **BashTool hard-block at TierBlocked even without PermissionState** — correct defense-in-depth.
- **Classifier + guardian merge via max-Tier** — the design's rule is sensible.
- **`RESOLVE_BENEATH` flag** — Go 1.26 `os.Root` does use `openat2` with `RESOLVE_BENEATH` on Linux kernels that support it, per the Go blog post referenced in §5 of the requirement.
