# Requirement — 工作目录白名单 + 路径逃逸拦截 (P0-2)

## 1. 背景 & 目标

vv 的文件类工具（`read` / `write` / `edit` / `glob` / `grep`）和 `bash` 工具目前以用户进程权限运行，未对访问路径做硬约束：

- `read` / `write` / `edit` 在 vage 层已有 `WithAllowedDirs` 选项，但 **vv 从未注入**，等同于无限制。
- `glob` / `grep` 注入了 `WithWorkingDir` 作为默认值，但只做 fallback，不作为边界。
- `bash` 仅通过 `cmd.Dir` 设置初始 cwd，shell 命令内部的 `cd /etc`、`$(cat /secret)`、绝对路径参数均不受约束。
- 所有工具共同依赖的 `toolkit.ValidatePath` 使用 `filepath.Clean` + `HasPrefix(resolved, dir+sep)` 做前缀匹配，符号链接被 `EvalSymlinks` 解析后才比对，存在 TOCTOU 竞争窗口。
- 配置层 (`configs.ToolsConfig`) 没有 `allowed_dirs` / `workdir_whitelist` YAML 键。

随着自主性增强（计划中的 MCP 接入、后台子 Agent、自学习闭环），文件系统成为放大攻击面最直接的通道。本需求落地的是 **最后一道确定性防线**：在用户进程边界内对所有工具调用执行统一的路径准入检查。

**目标**：在不改变现有工具 API 的前提下，让每个文件类 / bash 工具调用都经过一致的 "工作目录白名单" 准入判定，拒绝任何指向白名单外的文件访问与命令执行。

## 2. 用户故事 & 验收条件

### US-1 作为开发者，启动 vv 时我期望工具默认只能访问我的工作目录和系统临时目录

**AC**:
- 启动时，若用户未显式配置 `tools.allowed_dirs`，vv 自动将 `{ BashWorkingDir, os.TempDir() }` 装配为默认白名单。
- `read` / `write` / `edit` / `glob` / `grep` / `bash` 六个工具在注册时统一被注入该白名单。
- CLI 启动日志 / debug 模式可见当前生效的 allowed_dirs 列表。

### US-2 作为开发者，我可以通过 YAML 扩展白名单

**AC**:
- `vv.yaml` 新增 `tools.allowed_dirs: [path1, path2]` 键；支持 `~` 展开和相对路径解析（相对工作目录）。
- 用户配置的值 **追加** 到默认白名单上，而非替换（除非显式传入空数组）。
- 启动时对每条路径做 canonicalize（`filepath.EvalSymlinks` + `filepath.Abs`），解析失败直接报错退出。

### US-3 作为用户，工具尝试访问白名单外路径必须被拒绝，无论以何种形式

**AC**（`read`/`write`/`edit`）:
- 输入 `/etc/passwd` → 拒绝。
- 输入 `<workdir>/../../etc/passwd`（字面形式） → 拒绝（`filepath.Clean` 后落在白名单外）。
- 白名单内存在指向外部的符号链接 `<workdir>/link -> /etc`，访问 `<workdir>/link/passwd` → 拒绝。
- 在允许写入路径上创建符号链接指向外部（`edit` / `write` 写出的内容本身是 symlink 语义不直接支持，但若底层 `os.Root` 拒绝，则自然覆盖）。
- TOCTOU：在 `read` 过程中被替换为指向外部的 symlink，`os.Root` 通过 `openat2(RESOLVE_BENEATH)` 原子拒绝。

**AC**（`glob`/`grep`）:
- `path` 参数不在白名单 → 拒绝。
- 允许在白名单目录内 glob / grep，结果不需过滤（已经受限于路径参数）。

**AC**（`bash`）:
- 命令含 `cd <path>` 且 path 在白名单外 → 分类为 `blocked`。
- 命令含 `$(...)` 命令替换、反引号 `` ` `` 命令替换、`eval` → 分类为 `dangerous`（触发审批），理由："命令替换可绕过路径约束"。
- 命令参数中包含绝对路径且该路径不在白名单 → 分类为 `blocked`（例如 `cat /etc/passwd`）。
- 命令参数中含 `..` 向上逃逸路径 → 分类为 `dangerous`（触发审批）。
- 初始 cwd 仍通过 `cmd.Dir = BashWorkingDir` 设置；命令本身的 `cd` 向白名单内路径不阻止。

### US-4 作为开发者，误操作产生清晰错误信息以便排查

**AC**:
- 所有拒绝场景返回的错误字符串包含：工具名、被拒绝的路径、拒绝原因（"not in allowed_dirs" / "symlink escapes" / "shell command substitution detected"）、当前 allowed_dirs 列表（首条，便于提示用户）。
- 错误信息不泄露系统敏感路径内容，只说拒绝事实。

### US-5 作为开发者，希望现有功能零回归

**AC**:
- 所有现有 vage `tool/*` 单元测试、vage `integrations/tool_tests/` 集成测试在开启 allowed_dirs=工作目录 的前提下全部通过。
- vv 的 `integrations/` 现有用例（若存在）通过。

## 3. 范围

### In Scope
- `vage/tool/toolkit/path.go`：新增基于 `os.Root` 的安全文件访问抽象 `SafeFS`，保留 `ValidatePath` 作为非 `os.Root` 场景（glob/grep）的守门员，强化其 canonicalize + 符号链接校验。
- `vage/tool/read` `vage/tool/write` `vage/tool/edit`：迁移 I/O 到 `os.Root`；`WithAllowedDirs` 作为 root 集合的输入。
- `vage/tool/glob` `vage/tool/grep`：`ValidatePath` 路径强化（拒绝路径中含 symlink 逃逸的路径）。
- `vage/tool/bash`：新增 `PathGuard` 配置（`WithAllowedPaths`），在 classifier 前置一层路径规则（`cd` / 绝对路径 / `..` / 子 shell 检测）。
- `vv/configs/config.go`：新增 `ToolsConfig.AllowedDirs []string`；`~` 展开 + Abs 解析。
- `vv/registries/tool_access.go`、`vv/tools/tools.go`：注册所有六个工具时传入 `WithAllowedDirs` / `WithAllowedPaths`。
- `vv/setup/setup.go`：启动时构建默认白名单（`BashWorkingDir + os.TempDir()`），合并用户配置，做 canonicalize。
- 单元 & 集成测试。

### Out of Scope（列入后续需求）
- OS 级沙箱（bubblewrap / sandbox-exec） → 归入已登记的 P4-4。
- 基于 `mvdan.cc/sh/v3/syntax` 的完整 bash AST 解析 → 后续增强。
- MCP 层路径守卫（独立的 P0-4 MCP 凭据过滤覆盖）。
- 跨会话 namespace 强隔离 → P1-4（依赖本项完成）。
- 白名单的运行时动态更新 / 工具审批类目录扩展 UI。

## 4. 受影响的模块 / 组件

| 类别 | 模块 | 说明 |
|------|------|------|
| Role | 开发者 | 使用 vv 的开发者；无新增角色 |
| 模型 | 无 | 不新增产品模型实体 |
| 流程 | 工具调用流程 | 在工具 pre-exec 位置新增"路径准入"检查点；成败路径受影响 |
| 应用 | vv CLI / HTTP | 启动流程多一步 allowed_dirs 构建；错误面显示拒绝理由 |
| 字典 | 无 | 无新增业务字典项 |
| 架构 | vage/tool/* | 引入 SafeFS；Bash 新增 PathGuard 路径规则层 |

## 5. 业界参考（调研摘要）

完整报告见附录链接（research 子代理输出），关键结论：

- **os.Root（Go 1.24+）** 是 Go 生态下正确的 TOCTOU 防御；对应 Linux `openat2(RESOLVE_BENEATH)`。现有 vage 代码 `filepath.Clean + HasPrefix` 方案的已知不足被该 API 直接闭合。[go.dev/blog/osroot](https://go.dev/blog/osroot)
- **OpenHands** 用容器化作为最强隔离边界；**Claude Code** 新近加入 `bwrap` / `sandbox-exec`；**Cursor / Windsurf** 仅有 ignore-file 机制，已多次被绕过（CVE-2025-54135）。本需求为 OS 级沙箱到位前的"用户态硬兜底"。
- **Node `--allow-fs-read`** 与 **Deno `--allow-read`** 的历史 bypass（case-insensitive、realpath 竞争、已有 fd 绕过）提醒：**只在工具边界做检查不够**，必须用原子 API（openat2）+ canonicalize after open（`GetFinalPathNameByHandle`-等价）；Go 的 `os.Root` 内置两者。
- **Bash 子 shell / 反引号 / `eval`**：Claude Code 最终从正则放弃切到 AST 解析，证实正则过滤会被绕过；本需求采用中等强度（识别显式模式 + 分类器），AST 解析列为未来增强。

## 6. 成功衡量

- 六工具 × 4 类逃逸向量（字面 `..`、绝对路径、symlink 外指、`$(...)` 注入）的 24 组拒绝场景，在新增 integration test 下 100% 命中拒绝。
- 现有测试集 0 回归。
- 启动时 allowed_dirs 构建失败（路径不存在 / 解析失败） → 立即退出、报错包含失败路径，不静默降级。
- 新增代码行数预期 < 600（含测试）；不引入新的外部依赖。

## 7. 一致性备忘（不在本次解决）

- `feature-implement.md` 记录 P0-1 "危险命令规则库" 已实现；本需求在 bash 侧新增 `PathGuard` 分类规则，与 P0-1 的 `classifier.go` 是协作关系，不冲突。
- `overview.md` 的 "Does Not Cover" 区块未列出"工作目录白名单"；完成后需由 documenter 阶段同步到 "Covers" 并在 `feature-implement.md` 新增条目。
