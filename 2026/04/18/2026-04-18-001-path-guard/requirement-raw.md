# Raw Requirement

Source: `doc/prd/feature-todo.md` (first unimplemented item, P0-2)

## 原文

| 优先级 | 需求名称 | 所属类别 | 依赖项 | 难度 | 可复用的现有能力 | 排序理由 |
|--------|---------|---------|--------|------|-----------------|---------|
| P0-2 | 工作目录白名单 + 路径逃逸拦截 | 安全 | 无 | 低 | `BashTool.WithWorkingDir`、`tool/read,write,edit` | 防 `../` 逃逸与绝对路径跳出；所有文件类工具统一加 `pathGuard` 中间件；不做会被自主 Agent 放大风险 |

## 用户补充要求

- 做一个详细的需求调研，了解类似产品，做这个需求都有些哪些实现？注意些什么问题？
- 调研相关行业，这些功能的实现都有什么具体的方案？
- 在设计方案的时候做相关的参考
- 实现完了以后，将这个功能，从 todo 文件中移除放到 implemented 文件中

## 澄清问答（analyst 阶段补充）

### Q1 — Bash 工具路径强制等级
**A**: Medium — 通过模式 + 分类器识别 `cd`/子 shell/绝对路径参数；符合现有分类器架构，覆盖大多数 LLM 逃逸尝试。

### Q2 — 默认 AllowedDirs 包含范围
**A**: **工作目录 + OS 临时目录**（用户明确指出）。注意：不包含 `~/.vv`（`vv/memories/` 的持久化内存通过独立 API 访问，不走 read/write/edit，故不需要加入）。

### Q3 — 是否迁移到 `os.Root`（Go 1.24+）
**A**: Partial — read/write/edit 迁移到 `os.Root`（单文件操作，完美闭合 TOCTOU）；glob/grep 保持 `ValidatePath`（内部 shell out 到 find/rg/grep，os.Root 不适用）。

### Q4 — 符号链接逃逸策略
**A**: 严格 — 任何 symlink 解析后指向 allowed_dirs 之外均拒绝；read 和 create/link 均覆盖。

