# Plan 工作区 — 结构化需求

## 1. 背景与目标

### 1.1 上下文

`doc/design/session-context-solution.md` §4.4 规划了 P7 模式 "Plan/Scratchpad 工作区"。在 §8 差距汇总中，按优先级排序为：~~Session 实体化~~ → ~~Context Builder~~ → ~~vv Session wiring~~ → **Plan 工作区** → 迭代级 Checkpoint。

依赖现状：
- ✅ `vage/session/` —— Session 实体 + 三类 Store + SessionHook（2026-04-29-002）
- ✅ `vage/context/`（包名 vctx）—— Builder + Source 抽象，已内置 SystemPrompt/SessionMemory/SessionState/RequestMessages 四个 Source（2026-04-29-003）
- ✅ vv 端 wiring —— `setup.Init` 默认拉起 FileSessionStore + SessionHook，CLI `--session`，HTTP `/v1/sessions/*`（2026-04-29-004）
- ❌ Plan 工作区 —— 本次实现

### 1.2 核心目标

> 让 LLM 在跨进程、跨多步任务时，**把进度写在文件里、不写在 prompt 里**。

具体说：
1. 每个 Session 拥有持久化的 `plan.md`，记录任务大纲与进度。
2. LLM 通过工具（不靠 raw filesystem write）安全地读写 `plan.md`，避免 path traversal。
3. 每次 LLM 推理前，`plan.md` 自动注入 prompt 顶端（通过 `vctx.Source`）。
4. 工作区随 Session 生命周期自动创建/删除。
5. 完整可审计：write/read 触发 hook 事件，便于 tracelog 落盘。

### 1.3 业界对照（设计完整性评估）

| 系统 | 做法 | 借鉴点 |
|---|---|---|
| **Claude Code Memory tool** | `/memories/` 目录注入为工具，LLM 自行 read/write | 工具化访问、目录树即记忆 |
| **Devin / SWE-Agent** | `plan.md` 持久落盘，每步前后回写 | 简单、人类可读、无 schema |
| **Aider** | `.aider.tags.cache` + chat history 文件 | 文件即状态，不依赖 prompt |
| **MemGPT recall/archival** | LLM 主动调 `archival_insert/search` | 工具收口，避免 raw FS 暴露 |
| **vv 现有 todo_write** | 进程内、易失、为本轮 ReAct 服务 | **正交**：plan.md 是跨 run 持久化的高层任务大纲，todo_write 是当轮的细粒度步骤 |

**对原始设计的补充与收紧**：
- 原设计提到 `plan.md / scratch/ / notes/ / artifacts/` 四类。本期 **MVP 仅交付 `plan.md` + `notes/`**，scratch/artifacts 留作后续——避免接口签名为未实现需求过早承诺。
- 原设计未规定 LLM 如何写：本期决定**用专用工具** `plan_update` / `notes_write` / `notes_read`，**不**让 LLM 通过 `write`/`edit` 直接操作工作区目录，这样：
  - 不污染 `tools.allowed_dirs`
  - path traversal 由工具包内自洽校验
  - hook 事件结构化（不是 generic `tool_call`，可单独审计）
- 原设计未规定 ContextSource 形态：本期定义 `WorkspaceSource`，把 plan + notes index 拼成单一 system 消息注入 prompt。

## 2. 用户故事与验收标准

### US-1：开发者首次使用 vv 即拥有 plan 工作区

**作为** 一个 vv 用户  
**我希望** 我提交的多步任务能被自动记录在 `plan.md` 里  
**以便** 关闭终端后再回来时，LLM 立刻知道上次进展到哪一步

**验收**：
- 当 `session.enabled=true`（默认）且 LLM 调用 `plan_update` 时，工作区目录自动创建于 `<session-root>/<project>/<id>/workspace/`，权限 `0o700`，文件 `0o600`。
- 关闭并重启 vv 后用 `--session <id>` 续接，新一轮第一次推理 prompt 中能看到上次写入的 `plan.md` 内容。
- 关闭 `session.enabled=false` 时，`plan_update` 工具不注册，`WorkspaceSource` 不挂载——零成本。

### US-2：LLM 安全地维护 plan.md

**作为** 框架维护者  
**我希望** LLM 不能通过 `plan_update` 越界写入工作区之外  
**以便** 不引入新的 path traversal 风险

**验收**：
- `plan_update` 仅接受单一字段 `content: string`（整体覆盖），无法指定文件名或子路径——目标始终是工作区根的 `plan.md`。
- `notes_write` 接受 `name` + `content`，`name` 必须匹配 `^[A-Za-z0-9._-]{1,64}$`（拒绝 `..`、`/`、`\` 等），写入 `workspace/notes/<name>.md`。
- `notes_read` 同样校验 `name`。
- Workspace 包内单元测试覆盖 5 个 path traversal 攻击向量并断言返回 `ErrInvalidName`。

### US-3：plan.md 自动注入 prompt 顶端

**作为** Primary Assistant  
**我希望** 我每次推理前都能看到当前 plan.md  
**以便** 我决定下一步要做什么、不重复已完成的步骤

**验收**：
- `vctx.WorkspaceSource` 在 `[SystemPromptSource, WorkspaceSource, SessionMemorySource, RequestMessagesSource]` 顺序中，输出一条 `system` 消息：
  - 第一行: `## Plan Workspace`
  - 第二行: workspace 的 `plan.md` 路径或省略提示
  - 第三行起: plan.md 的全文内容（如有）
  - 末尾: `notes/` 目录中文件名清单（仅文件名，不是内容；要看内容需要 `notes_read`）
- 当 plan.md 不存在或为空，且 notes 目录为空，Source 标记 `Status="skipped"`，不输出消息。
- 当读取失败（IO 错误），Source 标记 `Status="error"`，但 builder fail-open 继续后续 source。
- BuildReport 中能查到 `WorkspaceSource` 报告的 `OriginalCount`（plan + notes 字符总数）。

### US-4：HTTP / CLI 可见

**作为** 运维人员或调试人员  
**我希望** 我能通过 HTTP/CLI 查看任意 session 的 plan.md  
**以便** 排查问题或回顾进度

**验收**：
- HTTP 新增 `GET /v1/sessions/{id}/workspace/plan` 返回 plan.md 全文（404 if 未创建）。
- HTTP 新增 `GET /v1/sessions/{id}/workspace/notes` 返回 notes 文件清单，`GET /v1/sessions/{id}/workspace/notes/{name}` 返回 note 内容。
- HTTP `DELETE /v1/sessions/{id}` 同时清理 workspace 目录（与现有的 meta.json/events.jsonl/state.json 一并删除）。

### US-5：可观测

**作为** 框架维护者  
**我希望** plan_update / notes_write 触发结构化事件  
**以便** tracelog 与 SessionHook 能落盘审计

**验收**：
- 工具调用成功后通过 ctx 中的 `Emitter` 发出新事件 `EventWorkspacePlanUpdated`（payload `WorkspacePlanUpdatedData{ SessionID, Bytes, Truncated }`）和 `EventWorkspaceNoteWritten`（payload `WorkspaceNoteWrittenData{ SessionID, Name, Bytes, Truncated }`）。
- 现有 `tracelog.JSONLHook` / `session.SessionHook` 不需要改动——事件透过 hook.Manager 自然落盘。

## 3. 范围

### 3.1 In Scope（本期交付）

- 新增 Go 包 `vage/workspace/`：
  - `Workspace` 接口 + `FileWorkspace` 实现（基于 filesystem，持久化）
  - 名称校验、path traversal 防护
  - 单元测试（≥ 80% 覆盖）
- 新增工具 `vage/tool/workspace/`：
  - `plan_update`、`notes_write`、`notes_read`
  - 使用与 `todo` 包相同的 ToolDef + Handler 模式
- 新增 `vctx.WorkspaceSource`（`vage/context/sources_workspace.go`）
- 扩展 `vage/agent/taskagent/`：暴露 `WithContextBuilder(Builder)` option（之前 §4.2 留作 TODO；本期顺带交付，因为 WorkspaceSource 必须挂上才能工作）
- vv 端：
  - `setup.Init` 默认构造 `FileWorkspace`（root = `<session-root>/<project>/<id>/workspace`，懒创建）
  - 把 `plan_update`/`notes_write`/`notes_read` 注册到 **Primary Assistant** 的 tool registry
  - 把 `WorkspaceSource` 添加到所有 dispatchable agent 与 Primary 的 ContextBuilder
  - HTTP routes: `GET /v1/sessions/{id}/workspace/plan` / `notes` / `notes/{name}`
  - HTTP `DELETE /v1/sessions/{id}` 增删除 workspace
- 文档：
  - `vage/.doc/workspace.md`（新文件）
  - 更新 `vage/.doc/context.md`（WorkspaceSource 入表）
  - 更新 `vv/CLAUDE.md`（Plan Workspace 段）
  - PRD：新增 `doc/prd/models/core/workspace/model-plan-workspace.md`、`doc/prd/applications/api/pages/core/workspace/`
  - 在 `doc/design/session-context-solution.md` §4.4、§8 标记 Plan 工作区已落地

### 3.2 Out of Scope（留作后续迭代）

- `scratch/` 子任务草稿（无具体使用场景；P10 树记忆落地时再补）
- `artifacts/` 产出物管理（同上；与 §4.5 checkpoint 配套交付更合理）
- LLM 主动 paging（§4.9 LLM-driven Paging）
- 跨进程并发写入（依赖 §4.5 锁机制）
- SQLite/Postgres 后端（仅 file 后端 MVP）
- Workspace 与 memory.Store 的双写同步（设计中提到 notes 与 memory 同步，但本期 notes 仅文件，不同步）
- TaskAgent option `WithContextBuilder` 公共化以外的 Builder 配置 API（仅暴露最小必需）

## 4. 受影响的角色 / 模型 / 流程 / 应用

| 维度 | 影响 |
|---|---|
| 角色 | Primary Assistant —— 新增 plan_update/notes_write/notes_read 三个工具；其他 dispatchable agent —— 仍只读 plan.md（通过 WorkspaceSource） |
| 模型 | 新增 PRD 模型 `Plan Workspace`（在 `doc/prd/models/core/workspace/`） |
| 流程 | TaskAgent 的 `buildInitialMessages` 现在多走一个 `WorkspaceSource`（顺序紧跟 SystemPromptSource 之后） |
| 应用 | API 新增 3 个 GET 路由 + 1 个 DELETE 路由扩展；CLI 无新参数（路径在 `--session list` 输出中可附带提示） |

## 5. 关键假设

下列决策不再向用户确认，记录于此以便审计：

| 决策 | 选择 | 备选 | 理由 |
|---|---|---|---|
| 工作区根路径 | `<session-root>/<project>/<id>/workspace/`（与 events.jsonl 同目录） | `~/.vv/workspaces/...` 独立根 | 与 session 同寿命，删 session = 删 workspace；运维直观 |
| LLM 写入方式 | 专用工具（plan_update/notes_write/notes_read） | 复用 write/edit + 把 workspace 加入 allowed_dirs | 收紧攻击面、事件结构化、无需污染 path guard |
| 写入语义 | 全量覆盖（plan_update 接 `content: string`） | 增量 patch | 与 todo_write 一致；diff 由 LLM 自己组织 |
| MVP 内容 | 仅 plan.md + notes | 全量 4 种（含 scratch/artifacts） | 收敛接口承诺；后两类无明确使用场景 |
| ContextSource 顺序 | `[System, Workspace, SessionMemory, Request]` | `[System, SessionMemory, Workspace, Request]` | Workspace 是高层任务结构、应紧贴 system；SessionMemory 是历史轮次细节 |
| Builder 注入方式 | 新增 `taskagent.WithContextBuilder(Builder)` option | 修改 setup 内部硬编码 | option 模式与现有 TaskAgent 风格一致；让 ContextBuilder MVP 真正"暴露" |
| notes 文件后缀 | `.md`（自动追加） | 让 LLM 指定 | 减一个出错点；Markdown 是 LLM 默认 |
| notes 名称模式 | `^[A-Za-z0-9._-]{1,64}$`，长度 ≤ 64 | 更宽松或完全自由 | 与 session id 同一族，复用 validateID 模式；64 比 128 短，因为 notes 数量更多但单个名字更短 |
| 单 plan.md 内容上限 | 64 KB | 256 KB / 1 MB | 防止 prompt 爆炸；一个 plan 不应该是文档；超过 = LLM 用法错了 |
| 单 note 内容上限 | 32 KB | 同上 | 同上 |
| 文件不存在的语义 | 读返回 `("", nil)`（空字符串 + nil err） | 返回 `os.ErrNotExist` | 工具/Source 用调用更顺畅；fs 错误仅在意 IO 失败 |

## 6. 不一致 / 遗留问题（供后续迭代或文档同步处理）

- `vage/CLAUDE.md` 提到的"四种 agent 类型"中的 TaskAgent 当前没有 `WithContextBuilder` option，本期顺带补齐。
- 设计文档 §4.4 提到 `notes/` 与 `memory.Store` 同步——本期不实现；后续若需要可加 `WorkspaceMemoryAdapter` 桥接（已记入 §3.2）。
- 现有 `todo_write` 是**会话内、不持久化**的细粒度任务列表，**与 plan.md 不同**：plan 是"跨 run 的任务大纲"，todo 是"本轮 ReAct 的进度条"。两者并存；文档中需明确二者关系（在 vv/CLAUDE.md 的 "Plan Workspace" 段落里写明）。

## 7. 成功标准（强可验证）

| # | 标准 | 验证方式 |
|---|---|---|
| S1 | `plan_update`/`notes_write`/`notes_read` 三工具单测覆盖 happy path + 5 种 path traversal 攻击 | `go test ./tool/workspace/... -v` 全绿 |
| S2 | `WorkspaceSource` 三档语义（ok / skipped / error）单测 | `go test ./context/... -run WorkspaceSource -v` |
| S3 | TaskAgent + WithContextBuilder 行为兼容现有 SystemPrompt + SessionMemory + RequestMessages 测试 | `go test ./agent/taskagent/...` 无回归 |
| S4 | vv setup 启动后 workspace 路径可见在日志，HTTP `GET /v1/sessions/{id}/workspace/plan` 返回写入内容 | 现网手测或集成测试 |
| S5 | 关闭 `session.enabled` 时不构造 workspace、不注册工具，零成本 | 单测断言 setup result 中 workspace 字段为 nil |
| S6 | `make build` (vage + vv) 全绿 | CI 通过 |
| S7 | 设计文档 §4.4 与 §8 已标记"已落地" | 文档审核 |
