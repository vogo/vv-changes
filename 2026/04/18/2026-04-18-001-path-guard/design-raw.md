# Design — 工作目录白名单 + 路径逃逸拦截

## 1. 顶层思路

引入统一的"路径守门员" `toolkit.PathGuard`：

- 对于单文件 I/O 工具（`read` / `write` / `edit`），用 Go 1.24+ 的 `os.Root` 做原子 TOCTOU-safe 访问。
- 对于子进程类工具（`glob` / `grep`），继续使用 `ValidatePath` 校验，但加强符号链接逃逸防护。
- 对于 `bash`，新增 `bash.PathGuardian`：在分类器之外做一次路径规则评估，输出 `Classification`（复用同一 Tier 枚举）。BashTool 内置硬阻（Blocked）、CLI PermissionState 二次消费（Dangerous/Caution 触发审批）。
- 在 vv 层增加 `tools.allowed_dirs` YAML 配置并在 setup 阶段计算好默认值（WorkingDir + OS Temp），统一通过 `registries/tool_access.go` 注入全部 6 个工具。

不引入外部依赖；不改现有工具 API；`WithAllowedDirs` 从"可选"变为"事实必须"——未注入时守门员为空策略（open，允许一切），向后兼容 vage 单元测试。

## 2. 模块变更清单

### 2.1 `vage/tool/toolkit/pathguard.go`（新增）

```go
// PathGuard binds a set of canonicalized allowed directories and exposes
// root-based access primitives plus a strict Validate for non-os.Root paths.
type PathGuard struct {
    roots []*rootEntry // canonical-path + pre-opened os.Root
}

type rootEntry struct {
    canonical string    // filepath.Abs + EvalSymlinks of the dir
    root      *os.Root  // opened once at construction
}

// NewPathGuard canonicalizes each dir (must exist, absolute, resolved),
// opens an os.Root for each, returns an error on the first failure.
// An empty PathGuard (no dirs) means "no restriction" — same semantics as today.
func NewPathGuard(allowedDirs []string) (*PathGuard, error) { ... }

// Allowed reports whether the guard enforces any restriction.
func (g *PathGuard) Allowed() bool { return g != nil && len(g.roots) > 0 }

// Dirs returns the canonical allowed directories (stable slice for logging).
func (g *PathGuard) Dirs() []string { ... }

// Check validates a path string, returning the canonical absolute path.
// Rules: non-empty, absolute, no UNC, filepath.Clean, must canonicalize into
// one of the roots; symlink components resolving outside are rejected.
func (g *PathGuard) Check(toolName, p string) (string, error) { ... }

// OpenForRead / OpenForWrite / Stat / Create — return *os.File (or FileInfo)
// bound to the matching root. All use root.Open* (=> openat2 on Linux).
// relWithin returns the (root, relPath) pair if p is inside a root.
func (g *PathGuard) OpenForRead(p string) (*os.File, string, error)
func (g *PathGuard) OpenForWrite(p string, flag int, perm os.FileMode) (*os.File, string, error)
func (g *PathGuard) Stat(p string) (os.FileInfo, string, error)

// Close closes each pre-opened os.Root; call on shutdown (best-effort).
func (g *PathGuard) Close() error
```

Internals:

- `Check` does: reject empty / UNC / relative; `filepath.Clean`; then for each root attempt `filepath.Rel(root.canonical, absClean)`; if `rel` starts with `..` or is absolute → skip; the first matching root wins. On match, call `root.Stat(rel)` to atomically confirm containment (closes TOCTOU). On `ENOENT`, fall back to walking ancestors via `ResolveExistingPath` and rechecking containment, so "write a new file" still works.
- Platform note: `os.Root` on macOS/Linux uses `openat2(RESOLVE_BENEATH | RESOLVE_NO_MAGICLINKS)` where supported; on Windows it uses emulation. This is exactly the primitive the industry research recommends.
- Pre-opening `os.Root` at construction is cheap and keeps the fd alive; closing it releases the fd.

### 2.2 `vage/tool/toolkit/path.go`（改动）

- 保留 `ValidatePath` / `ResolveExistingPath` / `CleanAllowedDirs`（向后兼容 glob/grep 单元测试）。
- `ValidatePath` 里增加一步：解析后，若解析前后指向不同目录且解析后的路径不再位于任何 allowedDirs 之下 → 拒绝（关闭 symlink-escape）。具体：对结果调用新的私有 `underAny(resolved, allowed)`，返回 `false` 时显式用错误"symlink resolves outside allowed directories"。
- 新增辅助 `CanonicalizeDirs(dirs) ([]string, error)`：对每个目录做 `filepath.Abs` + `EvalSymlinks`，不存在则报错。PathGuard 构造时使用。

### 2.3 `vage/tool/read/read_tool.go`（改动）

- 保留 `WithAllowedDirs(dirs ...string)`，内部转为构造 `*PathGuard` 并保存；保留 `allowedDirs` 字段供回退路径使用。
- 新增 `WithPathGuard(g *PathGuard)` 作为更直接的注入口（vv 从注册侧使用）。
- Handler 流程：
  1. `guard.Check("read", parsed.FilePath)` → `cleaned`
  2. 若 guard 生效：`f, rel, err := guard.OpenForRead(cleaned)`；用 `f.Stat()` 替代 `os.Stat(cleaned)`；目录列表改走 `os.File.ReadDir` on `f`
  3. 若 guard 不生效（空策略，兼容旧调用点）：回退到原实现
- 关键：目录列表模式改为基于 `*os.File` 的 `ReadDir`，`listDirectory(dir string)` 签名变为 `listDirectory(f *os.File, displayPath string)`。

### 2.4 `vage/tool/write/write_tool.go`（改动）

- `WithAllowedDirs` / `WithPathGuard` 同上。
- 写入流程：
  1. `guard.Check("write", parsed.FilePath)` → `cleaned`
  2. `guard.Stat(cleaned)` 判断是否为目录、是否存在（允许 ENOENT）
  3. 写入：通过 `guard.OpenForWrite(cleaned, O_RDWR|O_CREATE|O_TRUNC, 0o644)` 打开，再走与 `toolkit.AtomicWriteFile` 等价的"写到临时文件 + rename"，但临时文件也在 root 内用 `Root.OpenFile` 创建、用 `Root.Rename` 替换。为支持这点，给 `toolkit` 增加一个 `AtomicWriteInRoot(root *os.Root, rel string, data []byte, perm fs.FileMode)` helper。
  4. 非 guard 情况回退到原 `toolkit.AtomicWriteFile`。

### 2.5 `vage/tool/edit/edit_tool.go`（改动）

- 和 write 同型：Check → guard Stat → `Root.OpenFile` 读 → `AtomicWriteInRoot` 写。
- 现有 `matchDenyRule`、`readTracker` 逻辑保留不动。

### 2.6 `vage/tool/glob/glob_tool.go`（微改）

- 继续使用 `ValidatePath`（不引入 `os.Root`），但调用方式改为通过新增的 `guard.ValidateWith(toolName, path)`（如果 guard 非 nil，调用 guard.Check；否则回退到 `ValidatePath(toolName, path, gt.allowedDirs)`），便于后续统一。为实现此回退，`WithAllowedDirs` 继续写入 `gt.allowedDirs`；新增 `WithPathGuard` 同样缓存 guard，注册时二选一（vv 走 PathGuard，测试走 allowedDirs）。
- `pattern` 校验不变（拒绝绝对路径和 `..`）。

### 2.7 `vage/tool/grep/grep_tool.go`（微改）

- 与 glob 同。

### 2.8 `vage/tool/bash/pathguardian.go`（新增）

```go
// PathGuardian inspects each sub-command of a shell invocation and returns a
// Classification, parallel to Classifier. It is path-aware and needs the
// allowed-dir set to judge.
type PathGuardian struct {
    allowedDirs []string // canonicalized
    workingDir  string   // base for "./..", "../..", and relative args
}

func NewPathGuardian(allowedDirs []string, workingDir string) *PathGuardian { ... }

// Classify returns the worst-case classification across sub-commands.
// Uses the same splitSubCommands / readBackticks / readParen helpers already
// in this package so subshell content is caught.
func (g *PathGuardian) Classify(command string) Classification { ... }
```

Rules (in priority order):

1. **`cd <target>` escapes allowedDirs** (resolved relative to workingDir): `Tier=Blocked`, `Rule=cd-outside-allowed`, `Reason="cd target '<t>' outside allowed directories"`.
2. **Absolute path argument outside allowedDirs**, for a small allow-listed set of file-reading commands (`cat`, `less`, `more`, `head`, `tail`, `vi`, `vim`, `nano`, `grep`, `rg`, `cp`, `mv`, `rm`, `tee`, `awk`, `sed`, `xargs`, `file`, `stat`, `ls`) — match first whitespace-separated token as the command, subsequent tokens starting with `/` are checked. If outside: `Tier=Blocked`, `Rule=path-outside-allowed`. Paths under `/proc` / `/sys` / `/dev` are always blocked regardless.
3. **Relative path with `..` component** in arguments: `Tier=Dangerous`, `Rule=path-traversal-dots`, `Reason="argument contains '..' escape"`. Caller must confirm per-invocation.
4. Otherwise: `Tier=Caution` (no match), consistent with classifier's default.

Implementation note: the tokenizer splits on whitespace after stripping single/double quotes' contents the way the classifier already does; it does NOT fully shell-parse (that's the AST future enhancement).

### 2.9 `vage/tool/bash/bash_tool.go`（改动）

- 新增 `WithPathGuardian(g *PathGuardian) Option`；`BashTool.pathGuardian` 字段。
- 在 `execute` 最前增加一道硬阻：若 guardian 非 nil 且 `guardian.Classify(command).Tier == TierBlocked`，直接返回 `schema.ErrorResult("", "bash tool: blocked: "+reason)`。这样即使 vv 的 CLI PermissionState 未装 guardian，HTTP 模式或库调用方也有兜底。
- Dangerous/Caution 不在 BashTool 内处理；那由 CLI PermissionState 消费。

### 2.10 `vv/cli/permission.go`（改动）

- `PermissionState` 增加 `SetPathGuardian(g *bash.PathGuardian)` 与 `pathGuardian` 字段。
- 在评估顺序上，bash 工具的判定为：`max(classifier.Classify(cmd), pathGuardian.Classify(cmd))`，取最高 Tier；若两侧都有 Rule/Reason，优先采用高 Tier 的那一侧。
- 这保留现有 classifier 的语义并叠加路径维度。

### 2.11 `vv/configs/config.go`（改动）

```go
type ToolsConfig struct {
    BashTimeout    int             `yaml:"bash_timeout"`
    BashWorkingDir string          `yaml:"bash_working_dir"`
    AllowedDirs    []string        `yaml:"allowed_dirs,omitempty"` // NEW
    BashRules      BashRulesConfig `yaml:"bash_rules,omitempty"`
}
```

- 无显式默认值；由 `setup.Init` 在启动时合并默认列表（WorkingDir + os.TempDir()）。
- `~` 展开和相对路径处理放在 `setup.Init` 而不是 config 加载层——保持 config 结构是纯数据。

### 2.12 `vv/setup/setup.go`（改动）

启动阶段在"捕获 cwd"之后新增 `buildAllowedDirs`：

```go
func buildAllowedDirs(cfg configs.ToolsConfig) ([]string, error) {
    dirs := []string{cfg.BashWorkingDir, os.TempDir()}
    for _, d := range cfg.AllowedDirs {
        expanded, err := expandUserPath(d) // ~ -> $HOME, relative -> abs(wrt WorkingDir)
        if err != nil { return nil, fmt.Errorf("allowed_dirs[%q]: %w", d, err) }
        dirs = append(dirs, expanded)
    }
    // dedupe + canonicalize via toolkit.CanonicalizeDirs
    return toolkit.CanonicalizeDirs(dirs)
}
```

然后：

- 把 `canonicalDirs` 写回 `cfg.Tools.AllowedDirs`（以便下游所有消费点一致）。
- 构造 `pathGuard, err := toolkit.NewPathGuard(canonicalDirs)`；将其保存到 `setup.Result` 结构（新字段），供 `tool_access.go` 和 `main.go` 消费。
- 构造 `pathGuardian := bash.NewPathGuardian(canonicalDirs, cfg.Tools.BashWorkingDir)`；同样挂到 `Result`。
- 在 CLI setup 路径将 `pathGuardian` 通过 `permissionState.SetPathGuardian(pathGuardian)` 装配。
- 在启动日志（debug 级别）打印最终 allowed_dirs，便于排查。

### 2.13 `vv/registries/tool_access.go`（改动）

```go
func (p ToolProfile) BuildRegistry(
    toolsCfg configs.ToolsConfig,
    guard *toolkit.PathGuard,
    guardian *bash.PathGuardian,
) (*tool.Registry, error) { ... }
```

在 `registerCapabilityTools`：

- `CapRead`: `read.Register(reg, read.WithPathGuard(guard))`
- `CapWrite`: `write.Register(...)` + `edit.Register(reg, edit.WithPathGuard(guard), edit.WithDenyRules(...), edit.WithReadTracker(tracker))`
- `CapExecute`: bash 增加 `bash.WithPathGuardian(guardian)`
- `CapSearch`: glob/grep 加 `WithPathGuard(guard)`

`vv/tools/tools.go` 的 `Register` / `RegisterReadOnly` / `RegisterReviewTools` 同步加上签名变更（保持向后兼容：如果调用方不提供 guard/guardian，则内部也不注入）。三个函数增加可选 `Option` 参数形式 `WithGuard(guard)`、`WithGuardian(g)`。

### 2.14 `vv/main.go`（改动）

- 从 `setup.Result` 拿到 `pathGuardian`，替换现有 `permissionState.SetClassifier(...)` 调用处一并 `SetPathGuardian(...)`。

### 2.15 `vv/configs/bashrules.go`（不改）

classifier 规则库保持独立。PathGuardian 另立一职。

## 3. 错误信息规范

统一格式：`<tool>: <reason>: <path>`（不含 allowed_dirs 列表，避免刷屏；debug 日志里一次性打印 allowed_dirs）。

例：

- `read tool: path not allowed: /etc/passwd`
- `write tool: symlink resolves outside allowed directories: /workdir/link`
- `bash tool: blocked: cd target '/etc' outside allowed directories`
- `edit tool: failed to resolve path: <err>`

## 4. 数据流总览

```
vv.yaml -> ToolsConfig.AllowedDirs + BashWorkingDir
         -> setup.buildAllowedDirs (expand ~, abs, eval symlinks, dedupe)
         -> canonicalDirs
             ├─ toolkit.NewPathGuard(canonicalDirs)   -> *PathGuard
             │    -> read / write / edit / glob / grep (via WithPathGuard)
             └─ bash.NewPathGuardian(canonicalDirs, BashWorkingDir) -> *PathGuardian
                  -> BashTool (hard-block on Tier=Blocked)
                  -> PermissionState (max over classifier+guardian)
```

## 5. 测试计划

### 5.1 单元测试（vage/tool/toolkit/）

- `pathguard_test.go`：
  - 构造失败场景：非绝对、不存在、相对路径 → 错误。
  - `Check`：字面 `..` 逃逸、绝对路径内/外、UNC、空串。
  - Symlink：在 tmp 下建 `a -> /etc`；`Check(a/foo)` 必须拒绝。
  - `OpenForRead` / `Stat`：验证返回的 `os.Root`-安全句柄。
  - 空 guard（nil roots）：`Allowed() == false`。

### 5.2 单元测试（vage/tool/bash/）

- `pathguardian_test.go`：
  - `cd /etc` (outside) → Blocked。
  - `cd ./subdir` (inside) → Caution。
  - `cat /etc/passwd` (outside) → Blocked。
  - `cat relative/../../etc/passwd` → Dangerous (traversal dots)。
  - `cat $(cat /etc/passwd)` → 子 shell 被提取，Blocked。
  - `echo hello` → Caution。
- `bash_tool_test.go` 追加：guardian 注入后 Blocked 命令直接 `ErrorResult`。

### 5.3 单元测试（read/write/edit/glob/grep）

- 每个工具追加：带 guard 时的路径拒绝；symlink-escape 拒绝；目录列表和目录遍历仍然正常。
- 现有 allowedDirs 测试仍须通过（通过内部转换为 guard 的路径）。

### 5.4 集成测试（vage/integrations/tool_tests/）

- 新增 `pathguard_tests/`：跨 read/write/edit/glob/grep 在真实临时工作区上验证完整闭环（24 组逃逸向量 → 拒绝；等量合法向量 → 成功）。

### 5.5 单元测试（vv/configs/ + vv/setup/）

- `allowed_dirs` YAML 加载/合并：默认注入 WorkingDir + TempDir；用户配置去重；`~` 展开；不存在的目录 → 启动失败。

### 5.6 现有测试回归

- `vage/tool/{read,write,edit,glob,grep}` 既有单元 + 集成测试：在 guard 不注入时路径仍走回退，全部通过。
- `vv/cli/permission_test.go` 既有 classifier 测试：保持通过（PathGuardian 额外挂点默认 nil）。

## 6. 实施顺序（开发阶段）

1. `vage/tool/toolkit/pathguard.go` + test。
2. `vage/tool/toolkit/path.go` 强化 `ValidatePath` 的 symlink-escape 检查 + test。
3. `read` / `write` / `edit` 改为优先走 PathGuard；回退路径保留；test。
4. `glob` / `grep` 增加 `WithPathGuard` 选项；test。
5. `vage/tool/bash/pathguardian.go` + test。
6. `vage/tool/bash/bash_tool.go` 注入 guardian + test。
7. `vv/configs/config.go` 新字段 + test。
8. `vv/setup/setup.go` 构建流程 + 启动自检 + test。
9. `vv/registries/tool_access.go` 注册注入 + `vv/tools/tools.go` 同步。
10. `vv/cli/permission.go` SetPathGuardian + 评估合并 + test。
11. `vv/main.go` 接线。
12. `vage/integrations/tool_tests/pathguard_tests/`。
13. lint + 三模块 `make test` 全绿。

## 7. 风险与决策记录

| 决策 | 备选 | 理由 |
|------|------|------|
| Guard 构造失败 → 启动退出 | Fallback 到"空 guard" | 静默降级会导致用户以为受保护但其实没有；fail-fast 更安全 |
| `os.Root` 仅用于 read/write/edit | 全工具统一 | glob/grep shell out 到外部命令，`os.Root` 不适用；强行抽象反而增加复杂度 |
| 默认白名单 = WorkingDir + TempDir | 仅 WorkingDir | 用户明确要求。许多 Go build / go mod 等流程依赖 `os.TempDir()` |
| Bash 采用模式/分类器级规则而非 AST | AST 解析 | 落地成本低、不引入新依赖；列为 P 级后续增强（不在本需求） |
| 不在 `~/.vv` 默认允许 | 加入默认 | 持久化内存（`vv/memories/`）通过独立 API 访问，不走 read/write/edit，故无需把 `~/.vv` 加入 |

## 8. 文档后置联动（documenter 阶段）

- `doc/prd/overview.md`：在 "Covers" 列表新增 "Working-directory allow-list enforced on file tools and bash command path arguments"；相应从 "Does Not Cover" 移除（该条目隐含在 P0-2 前）。
- `doc/prd/architecture/architecture.md`：说明 PathGuard / PathGuardian 结构与 `os.Root` 选型。
- `doc/prd/feature-implement.md`：在安全一节新增条目，引用关键文件路径。
- `doc/prd/feature-todo.md`：移除 P0-2 行。
