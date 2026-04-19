# Design — 工作目录白名单 + 路径逃逸拦截

> Revision note: superseded `design-raw.md`. Changes incorporate the design-review improvements: HTTP-mode hard-block for Dangerous, explicit `allowed_dirs: []` semantics, `/`-root rejection, containment-based dedup, error messages that expose allowed_dirs, bash `cd` edge-case matrix, universal absolute-path check in the guardian, Windows rejection rules, explicit WithDenyRules × PathGuard ordering, and byte-compatible `WithAllowedDirs` fallback.

## 1. 顶层思路

引入统一的"路径守门员" `toolkit.PathGuard`：

- 单文件 I/O 工具（`read` / `write` / `edit`）走 Go 1.26+ 的 `os.Root` 做原子 TOCTOU-safe 访问。
- 子进程类工具（`glob` / `grep`）继续走 `ValidatePath`，并加强符号链接逃逸防护（subprocess 无法被 `os.Root` 覆盖）。
- `bash` 工具新增 `bash.PathGuardian`：在分类器之外做一次路径规则评估，输出 `Classification`（复用同一 Tier 枚举）。BashTool 内置硬阻（Blocked）。`PermissionState` 取 classifier 和 guardian 的**最高** Tier，并根据运行模式应用策略（CLI 提示，HTTP 硬阻）。
- vv 层在 `tools.allowed_dirs` YAML 上计算默认值（WorkingDir + OS Temp），通过 `registries/tool_access.go` 注入全部 6 个工具。

不引入外部依赖；不改现有工具外观 API；`WithAllowedDirs` 的既有语义（在 vage 独立使用场景下）保持不变以保证 vage 自身测试零回归。

## 2. 模块变更清单

### 2.1 `vage/tool/toolkit/pathguard.go`（新增）

```go
// PathGuard binds a set of canonicalized allowed directories and exposes
// root-based access primitives.
type PathGuard struct {
    roots []*rootEntry // canonicalized path + pre-opened *os.Root
}

type rootEntry struct {
    canonical string   // filepath.Abs + EvalSymlinks of the dir
    root      *os.Root // opened once at construction via os.OpenRoot
}

// NewPathGuard canonicalizes each dir (must exist, absolute, resolved), opens
// an os.Root for each, deduplicates by containment, and rejects filesystem-root
// entries (Unix "/" or Windows drive roots). Returns an error on the first
// failure. An empty `dirs` yields a guard with Allowed()==false, meaning
// "no restriction" — preserved for vage library use cases.
func NewPathGuard(dirs []string) (*PathGuard, error) { ... }

// Allowed reports whether the guard enforces any restriction.
func (g *PathGuard) Allowed() bool { return g != nil && len(g.roots) > 0 }

// Dirs returns the canonical allowed directories (stable, sorted) for logging
// and error messages.
func (g *PathGuard) Dirs() []string { ... }

// Check validates `p`, returning the canonical absolute path and the matching
// root. Rejections:
//   - empty / non-absolute / UNC / Windows extended-path forms (\\?\, \\.\)
//   - short-name (8.3) markers on Windows
//   - drive-relative "C:foo" (no separator after drive letter)
//   - cleaned path that escapes every root
//   - symlink component whose realpath escapes the containing root
// On success the matching root is returned so callers can invoke Root.Open*.
func (g *PathGuard) Check(toolName, p string) (absClean, relInRoot string, root *os.Root, err error) { ... }

// OpenForRead / OpenForWrite / Stat / Lstat: wrappers around Check + Root.Open*.
// All use Root.* methods (openat2(RESOLVE_BENEATH) on Linux; emulation elsewhere).
func (g *PathGuard) OpenForRead(p string) (*os.File, string, error)
func (g *PathGuard) OpenForWrite(p string, flag int, perm os.FileMode) (*os.File, string, error)
func (g *PathGuard) Stat(p string) (os.FileInfo, string, error)
func (g *PathGuard) Lstat(p string) (os.FileInfo, string, error)

// MkdirAll creates the directory tree for relPath inside the matching root;
// used by the write path for "create new file under missing subdir" flows.
func (g *PathGuard) MkdirAll(p string, perm fs.FileMode) error

// Close closes all pre-opened roots; call on shutdown (best-effort).
func (g *PathGuard) Close() error
```

Construction details:

- `NewPathGuard` first calls `CanonicalizeDirs` (abs + EvalSymlinks; each dir must exist).
- Rejects filesystem roots: on Unix the canonical form `/`; on Windows any path that is exactly a drive root (`C:\`, `D:\`) or a UNC root.
- **Containment dedupe**: sort canonical dirs by length ascending; drop any dir that has an ancestor already in the result (via `filepath.Rel` not returning `..`).
- Opens `os.OpenRoot(canonical)` for each surviving dir.
- On any error closes already-opened roots to avoid fd leak.

`Check` details:

- Rejects: empty, not absolute, UNC (`\\server\share`, Windows extended `\\?\`, `\\.\`), drive-relative (`C:foo` without separator), `..` escape in the cleaned form.
- Uses `filepath.Clean` then `ResolveExistingPath` (handles non-existent leaf for writes).
- For each root, try `filepath.Rel(canonical, absClean)`; if rel does not start with `..` and is not absolute, use that root.
- After matching, call `root.Lstat(rel)` (for existing paths) to confirm containment atomically — if rel traverses a symlink pointing outside, `Root.Lstat` returns an error and the guard rejects.

`os.Root` is goroutine-safe per Go docs; `PathGuard` is read-only after construction, so itself is also safe for concurrent use.

Platform note: `os.Root` on Linux uses `openat2(RESOLVE_BENEATH | RESOLVE_NO_MAGICLINKS)` where supported; on macOS/Windows it uses emulation. Windows emulation does not provide atomic TOCTOU rejection the same way; the extra `Check` rejections listed above compensate.

Known design limitation (documented in §7 risks): symlinks inside the allowlist pointing outside are rejected. Users who need e.g. `<proj>/vendor -> /usr/share/...` must add the target to `allowed_dirs`.

### 2.2 `vage/tool/toolkit/path.go`（改动）

- Keep `ValidatePath` / `ResolveExistingPath` / `CleanAllowedDirs` so glob/grep and existing vage tests retain byte-level behavior.
- Strengthen `ValidatePath`: after symlink resolution, assert the resolved path is still under one of `allowedDirs`. Introduce an internal `underAny(resolved, allowed)` helper and produce the explicit error `symlink resolves outside allowed directories: <path>`.
- Add `CanonicalizeDirs(dirs []string) ([]string, error)`: for each dir, `filepath.Abs` + `EvalSymlinks`; error if the dir does not exist or resolves to filesystem root.

### 2.3 `vage/tool/read/read_tool.go`（改动）

- `WithAllowedDirs(dirs ...string)` keeps its current semantics (ValidatePath fallback). **No behavioral change for existing vage consumers.**
- New `WithPathGuard(g *PathGuard) Option`. If both are passed, PathGuard wins.
- Handler flow:
  1. If guard installed: `cleaned, _, root, err := guard.Check("read", parsed.FilePath)`; open via `root.Open(rel)`; `ReadDir` on the `*os.File` for directory mode (`listDirectory` signature becomes `listDirectory(f *os.File, displayPath string)`).
  2. Else: today's path (`ValidatePath` + `os.Open`).
- Errors include the allowed_dirs list (see §3).

### 2.4 `vage/tool/write/write_tool.go`（改动）

- Options same shape as read.
- When guard is installed: `guard.Check` → `guard.Stat` (ENOENT tolerated for new files) → `guard.MkdirAll(filepath.Dir(rel))` → atomic write via a new `toolkit.AtomicWriteInRoot(root, rel, data, perm)`. The helper:
  - Creates a temp file in the same directory via `root.OpenFile(tmp, O_CREATE|O_EXCL|O_WRONLY, 0o600)`.
  - Writes, Close, `root.Chmod(tmp, perm)` (may no-op on platforms where `Root.Chmod` is racy — documented), `root.Rename(tmp, rel)`.
- Without guard: `toolkit.AtomicWriteFile` as today.
- Reject `create_only=true` if target exists via `root.Stat` (not `os.Stat`).

### 2.5 `vage/tool/edit/edit_tool.go`（改动）

- Same option shape. Flow inside handler:
  1. `guard.Check` (if installed) else `ValidatePath`.
  2. `matchDenyRule(cleaned)` — unchanged.
  3. `readTracker.HasRead(cleaned)` — unchanged.
  4. `LockPath(cleaned)` — unchanged.
  5. Load content via `root.ReadFile(rel)` or `os.ReadFile(cleaned)`.
  6. Write-back via `AtomicWriteInRoot` or `AtomicWriteFile`.

Evaluation order is intentionally: **(1) PathGuard → (2) matchDenyRule → (3) readTracker**. PathGuard is the cheapest hard-reject; deny-rules is a basename filter (complementary, not redundant); readTracker is the usability rail.

### 2.6 `vage/tool/glob/glob_tool.go`（微改）

- Add `WithPathGuard(g *PathGuard) Option`; cache it on the tool.
- On execute, resolve the search dir → if guard installed, use `guard.Check("glob", dir)` (which also rejects symlink escape); else fall back to `ValidatePath`.
- Pattern validation (`containsDotDot`, absolute-pattern rejection) unchanged — already sufficient because `..` is scrubbed on the pattern side; combined with bounded `cmd.Dir`, subprocess escape is blocked.
- `git ls-files` path: also run inside `cmd.Dir=dir` where `dir` has passed the guard check.

### 2.7 `vage/tool/grep/grep_tool.go`（微改）

- Same as glob. If grep grows `include` / `exclude` filters later, apply the same `containsDotDot` rule to them.

### 2.8 `vage/tool/bash/pathguardian.go`（新增）

```go
type PathGuardian struct {
    allowedDirs []string // canonicalized
    workingDir  string   // for resolving relative paths in tokens
}

func NewPathGuardian(allowedDirs []string, workingDir string) *PathGuardian

// Classify inspects each sub-command (via splitSubCommands / readBackticks /
// readParen already in this package) and returns the worst-case Classification.
func (g *PathGuardian) Classify(command string) Classification
```

**Rules (priority order):**

| Pattern | Tier | Rule name |
|---|---|---|
| `cd <literal>` outside allowed | Blocked | `cd-outside-allowed` |
| `cd` / `cd ~` / `cd $HOME` | Dangerous | `cd-home-ambiguous` |
| `cd -` | Dangerous | `cd-prev-unknown` |
| `cd $VAR` / `cd "${...}"` / `cd $(...)` | Dangerous | `cd-variable` |
| `cd <glob>` | Dangerous | `cd-glob` |
| `cd <literal>` inside allowed | (no bump) | — |
| Any absolute path token (bare arg or redirect target) outside allowed | Blocked | `path-outside-allowed` |
| Any absolute path token under `/proc`, `/sys`, `/dev`, regardless of allowlist | Blocked | `system-sensitive-path` |
| Any bare token containing `..` | Dangerous | `path-traversal-dots` |
| Otherwise | Caution | (no match) |

**Tokenizer requirements.** The tokenizer is command-agnostic (rule #7 applies universally, not only to a cat/less allow-list):

- Splits sub-commands on top-level separators (reusing `splitSubCommands`).
- For each sub-command, tokenizes on whitespace while tracking token role: `cmd | arg | redirect-target`. Redirect operators recognized: `>`, `>>`, `<`, `2>`, `&>`, `<>`, `|&`. The target is the next whitespace-separated token.
- Single-quoted content is dropped (as today). Double-quoted content is dropped **but embedded `$(...)` / backticks are still extracted** — the inner content is classified recursively.
- Variable-expansion strings (`$VAR`, `${...}`) in `cd` → Dangerous; in other args → treated as opaque (not matched by the absolute-path rule).

**Not attempted (documented as future work):** full AST via `mvdan.cc/sh/v3/syntax`; variable-value tracking (`X=rm; $X -rf /`); globs that actually expand to outside paths. These are already out of scope per requirement §3.

### 2.9 `vage/tool/bash/bash_tool.go`（改动）

- New `WithPathGuardian(g *PathGuardian) Option`; field `BashTool.pathGuardian`.
- At the start of `execute`: if guardian is installed and `guardian.Classify(command).Tier == TierBlocked`, return `ErrorResult` with rule name and reason. This is a hard-block ("belt-and-suspenders") that applies regardless of whether PermissionState is in the chain — covers HTTP and library callers.
- Dangerous / Caution classifications are **not** handled inside BashTool; they are for PermissionState to apply mode-dependent policy.

### 2.10 `vv/cli/permission.go`（改动）

- `PermissionState` gains:
  - `pathGuardian *bash.PathGuardian` + `SetPathGuardian(g)`.
  - `isNonInteractive bool` set at construction (HTTP / non-interactive CLI).
- Bash classification replaced by a **merge** step in `Execute`:
  ```go
  cls := maxTier(classifier.Classify(cmd), guardian.Classify(cmd))
  ```
  `maxTier` picks the higher Tier; on ties the guardian's decision wins (path-aware is more specific). The merged classification is attached to ctx as today.
- Policy by Tier:
  - `TierBlocked` → always hard-reject.
  - `TierDangerous` → if `isNonInteractive`, hard-reject with message `bash command classified as dangerous and non-interactive mode is active; rejected`; else prompt per-invocation, no allow-always.
  - `TierCaution` → mode-dependent prompt (as today).
  - `TierSafe` → bypass.
- `NewPermissionState` signature grows to `NewPermissionState(mode, isNonInteractive bool)`.

### 2.11 `vv/configs/config.go`（改动）

```go
type ToolsConfig struct {
    BashTimeout    int             `yaml:"bash_timeout"`
    BashWorkingDir string          `yaml:"bash_working_dir"`
    // AllowedDirs semantics:
    //   nil  (YAML key absent) -> merge defaults [BashWorkingDir, os.TempDir()]
    //   []   (YAML "allowed_dirs: []" or empty list) -> explicit opt-out of defaults;
    //         startup fails because the guard would be unenforceable.
    //         To disable the guard entirely, the user must also set a feature flag
    //         (not in scope — the guard is on by default).
    //   [...] (non-empty) -> use exactly these; no auto-merge of WorkingDir/TempDir
    //         unless the user included them explicitly.
    AllowedDirs    *[]string       `yaml:"allowed_dirs,omitempty"` // NEW: pointer distinguishes unset from empty
    BashRules      BashRulesConfig `yaml:"bash_rules,omitempty"`
}
```

Rationale for the pointer: requirement AC US-2 explicitly says "except when user passes an empty array" — we need to tell `key absent` from `key present but empty`. YAML unmarshal into `*[]string` gives us that.

### 2.12 `vv/setup/setup.go`（改动）

Add a `buildAllowedDirs` step after capturing `BashWorkingDir`:

```go
func buildAllowedDirs(cfg configs.ToolsConfig) ([]string, error) {
    var dirs []string
    switch {
    case cfg.AllowedDirs == nil:
        // YAML key absent -> merge defaults.
        dirs = []string{cfg.BashWorkingDir, os.TempDir()}
    case len(*cfg.AllowedDirs) == 0:
        // Explicit empty -> user opted out of defaults; treat as config error.
        return nil, fmt.Errorf("tools.allowed_dirs is explicitly empty; at least one directory is required")
    default:
        dirs = *cfg.AllowedDirs
        for i, d := range dirs {
            expanded, err := expandUserPath(d) // ~ -> HOME, relative -> abs(wrt WorkingDir)
            if err != nil {
                return nil, fmt.Errorf("allowed_dirs[%d]=%q: %w", i, d, err)
            }
            dirs[i] = expanded
        }
    }
    // CanonicalizeDirs: filepath.Abs + EvalSymlinks + existence check + reject "/"/drive roots.
    // Also collapses containment (subdir of another entry is dropped).
    canonical, err := toolkit.CanonicalizeDirs(dirs)
    if err != nil {
        return nil, err
    }
    return canonical, nil
}
```

After building `canonicalDirs`:

- `pathGuard, err := toolkit.NewPathGuard(canonicalDirs)` (errors fail startup).
- Assert `pathGuard.Allowed() == true`; internal error if not.
- `pathGuardian := bash.NewPathGuardian(canonicalDirs, cfg.Tools.BashWorkingDir)`.
- Store both on `setup.Result` (new fields `PathGuard`, `PathGuardian`); consumed by `tool_access.go` and `main.go`.
- Log at **INFO** (not DEBUG): `slog.Info("vv: allowed_dirs active", "dirs", canonicalDirs)` — AC US-1 wants this visible.

### 2.13 `vv/registries/tool_access.go`（改动）

```go
func (p ToolProfile) BuildRegistry(
    toolsCfg configs.ToolsConfig,
    guard *toolkit.PathGuard,
    guardian *bash.PathGuardian,
) (*tool.Registry, error) { ... }
```

In `registerCapabilityTools`:

- `CapRead`: `read.Register(reg, read.WithPathGuard(guard))`
- `CapWrite`: `write.Register(reg, write.WithPathGuard(guard))` + `edit.Register(reg, edit.WithPathGuard(guard), edit.WithDenyRules(...), edit.WithReadTracker(tracker))`
- `CapExecute`: `bash.Register(reg, bash.WithPathGuardian(guardian), bash.WithWorkingDir(...), bash.WithTimeout(...))`
- `CapSearch`: `glob.Register(reg, glob.WithPathGuard(guard), ...)` and `grep.Register(reg, grep.WithPathGuard(guard), ...)`

`vv/tools/tools.go`'s `Register` / `RegisterReadOnly` / `RegisterReviewTools` get an extra variadic option for guard/guardian so callers outside of the setup path (tests) can either opt in or keep current behavior.

### 2.14 `vv/main.go`（改动）

- Construct `permissionState` with `cli.NewPermissionState(permissionMode, cfg.Mode == "http")`.
- After `setup.Init`, call `permissionState.SetClassifier(...)` (existing) and `permissionState.SetPathGuardian(initResult.SetupResult.PathGuardian)` (new).

### 2.15 `vv/configs/bashrules.go`（不改）

Classifier rule library stays independent. Guardian is additive.

## 3. 错误信息规范

Format: `<tool>: <reason>: <path> (allowed: [d1, d2, ...])`.

Rationale: the agent (LLM) is the consumer of these error strings; it needs the allow-list to self-correct. The allow-list is not sensitive — it's the user's own working directory and `/tmp`. The list is truncated to 3 entries with `... +N more` if longer.

Examples:

- `read tool: path not allowed: /etc/passwd (allowed: [/home/me/proj, /tmp])`
- `write tool: symlink resolves outside allowed directories: /workdir/link (allowed: [/home/me/proj, /tmp])`
- `bash tool: blocked by rule "cd-outside-allowed": cd target '/etc' outside allowed directories`
- `bash tool: rejected (non-interactive mode): dangerous command pattern curl-to-shell`
- `edit tool: failed to resolve path: <inner err> (allowed: [...])`

Startup logs contain the full list once; runtime errors contain the clipped list.

## 4. 数据流总览

```
vv.yaml -> ToolsConfig.AllowedDirs (*[]string) + BashWorkingDir
         -> setup.buildAllowedDirs
              - nil  -> merge [WorkingDir, TempDir()]
              - []   -> fail startup
              - set  -> expand + canonicalize + containment-dedupe
         -> canonicalDirs
             ├─ toolkit.NewPathGuard(canonicalDirs)
             │    ├─ reject "/" / drive roots
             │    ├─ open one *os.Root per canonical dir
             │    └─ read / write / edit / glob / grep (via WithPathGuard)
             └─ bash.NewPathGuardian(canonicalDirs, BashWorkingDir)
                  ├─ BashTool: TierBlocked → hard-reject in execute()
                  └─ PermissionState: max(classifier, guardian)
                      ├─ Blocked → hard-reject (all modes)
                      ├─ Dangerous → hard-reject if HTTP, else prompt no-allow-always
                      ├─ Caution  → mode-dependent prompt
                      └─ Safe     → bypass
```

## 5. 测试计划

### 5.1 Unit (`vage/tool/toolkit/pathguard_test.go`)

- Construction failure: dir does not exist, relative path, root `/`, Windows drive root.
- `Check`: empty, relative, UNC, Windows extended path, drive-relative, `..` escape, symlink pointing outside an allowed dir.
- `OpenForRead` / `Stat`: verify handles are `*os.File` scoped to the matching root.
- `MkdirAll` + `OpenForWrite`: create-new-file under nested non-existent subdir — success; same path but ancestor is dangling symlink outside — rejected.
- Containment dedupe: `["/a", "/a/b"]` → `["/a"]`.
- Empty-guard: `Allowed() == false`.
- Concurrency: 100 goroutines opening/reading — no data race (go test -race).
- Platform-specific: `pathguard_windows_test.go` covers `\\?\`, `\\.\`, short-name, drive-relative.

### 5.2 Unit (`vage/tool/bash/pathguardian_test.go`)

Cover every row of the cd edge-case table and the absolute-path-universal rule (using unknown commands like `perl /etc/passwd`). Cover redirection target rule (`echo hi > /etc/motd` → Blocked). Cover `$(...)` and backtick extraction.

### 5.3 Unit (`vage/tool/{read,write,edit,glob,grep}`)

- Each tool gets new tests for the `WithPathGuard` path: rejection cases and happy path.
- Existing `WithAllowedDirs`-only tests MUST continue to pass unchanged (verifies byte-level compatibility — see design principle in §2.3).

### 5.4 Integration (`vage/integrations/tool_tests/pathguard_tests/`)

Single integration suite covering the 6×4 matrix (6 tools × 4 escape vectors: literal `..`, absolute outside, symlink outside, `$(...)` injection) = 24 rejection cases plus 24 matching legal cases to prove zero regression. This is an intentional exception to the per-tool naming convention because the value is cross-cutting.

### 5.5 Unit (`vv/configs/`, `vv/setup/`)

- `allowed_dirs` YAML: nil → merged defaults; explicit empty → startup error; non-empty → used verbatim with `~` expansion.
- Non-existent user-supplied dir → startup error with path in message.
- `/` in allowed_dirs → rejected.

### 5.6 Unit (`vv/cli/permission_test.go`)

- Add `isNonInteractive=true` cases: Dangerous → hard-reject; existing cases with `false` still prompt.
- classifier + guardian merge: when classifier says Caution and guardian says Blocked, result is Blocked.

### 5.7 Regression

- `vage/tool/{read,write,edit,glob,grep}` existing unit + integration tests: unchanged (guard not injected, falls back to `ValidatePath` which is unchanged except for the stronger symlink-escape rejection).
- `vv/cli/permission_test.go` existing classifier-only cases: still pass after signature change (test helpers updated to pass `false` for `isNonInteractive`).

## 6. 实施顺序

1. `vage/tool/toolkit/pathguard.go` + `CanonicalizeDirs` + tests.
2. `vage/tool/toolkit/path.go`: strengthen `ValidatePath` symlink-escape rejection + tests.
3. `read` / `write` / `edit`: add `WithPathGuard`, preserve `WithAllowedDirs` fallback; `AtomicWriteInRoot` helper; tests.
4. `glob` / `grep`: add `WithPathGuard` option; tests.
5. `vage/tool/bash/pathguardian.go` + tests (including the cd edge-case table).
6. `vage/tool/bash/bash_tool.go`: inject guardian + hard-block on Blocked; tests.
7. `vv/configs/config.go`: pointer-typed `AllowedDirs` + `Load` support; tests.
8. `vv/setup/setup.go`: `buildAllowedDirs` + INFO log + propagate PathGuard/PathGuardian on `Result`; tests.
9. `vv/registries/tool_access.go`: new signature; pass guard/guardian into each Register call; `vv/tools/tools.go` mirrors.
10. `vv/cli/permission.go`: `SetPathGuardian`, `isNonInteractive`, merge logic; tests.
11. `vv/main.go`: wire `isNonInteractive` (cfg.Mode=="http") and `SetPathGuardian`.
12. `vage/integrations/tool_tests/pathguard_tests/`.
13. `make lint && make test` for all three modules.

## 7. 风险 / 决策记录

| 决策 | 备选 | 理由 |
|---|---|---|
| Guard 构造失败 → 启动退出 | Fallback to "open guard" | Silent degradation violates the whole point of the feature. |
| `os.Root` 只用于 read/write/edit | Try to bound glob/grep too | glob/grep shell out — `os.Root` cannot bound a subprocess. `ValidatePath` + bounded `cmd.Dir` + pattern validation is the correct layer. |
| 默认白名单 = WorkingDir + TempDir | Only WorkingDir | Requirement explicitly asks for both; `go build` / `go test` routinely touch TempDir. |
| `allowed_dirs: []` → startup error | Silently allow zero roots | Zero-roots means "no enforcement"; user likely mis-configured; fail loud. |
| Bash 采用模式 + 分类器规则，而非 AST | AST via `mvdan.cc/sh/v3/syntax` | AST would catch more, but adds a dep and is significantly more code; shipped today is a meaningful step up; AST is a listed future enhancement. |
| 错误信息暴露 allowed_dirs | 仅显示被拒路径 | LLM caller needs the list to self-correct; paths are not secret — they're the user's own CWD and `/tmp`. |
| 既有 `WithAllowedDirs` 保持 ValidatePath 语义 | 内部升级到 os.Root | Preserves vage library consumer compatibility; os.Root behavior is opt-in via `WithPathGuard`. |
| HTTP 模式下 Dangerous 强阻 | 静默放行 | HTTP runs headless — cannot prompt; silently allowing contradicts AC US-3. |
| 符号链接指向外部被拒 | 跟随外部符号链接 | Defeats the point; documented as known limitation with workaround (add target to allowed_dirs). |
| Windows 采用 os.Root 仿真 + 额外 Check 拒绝 | 重新实现 RESOLVE_BENEATH equivalent | Go 的仿真实现是上游维护的正确做法；加上 `\\?\` / 短名 / drive-relative 拒绝收敛剩余面。 |

## 8. 文档后置联动（documenter 阶段）

- `doc/prd/overview.md`: "Covers" 新增 "Working-directory allow-list enforced on file tools and bash command path arguments".
- `doc/prd/architecture/architecture.md`: 说明 PathGuard / PathGuardian 选型与 `os.Root` 使用。
- `doc/prd/feature-implement.md`: 安全一节新增条目，引用新增文件路径。
- `doc/prd/feature-todo.md`: 移除 P0-2.
