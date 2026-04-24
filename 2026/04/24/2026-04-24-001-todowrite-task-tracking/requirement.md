# P1-9 · TodoWrite 内置任务追踪工具 — 需求文档

> 需求来源：`doc/prd/feature-todo.md` 第一条未实现项 🆕 P1-9。
> Session：`changes/2026/04/24/2026-04-24-001-todowrite-task-tracking`。
> 原始条目：Claude Code 实测最能提升"多步任务完成率"的单一工具；与现有 memory 正交。

---

## 1. 背景与目标

### 1.1 背景

vv/vage 当前缺少一个 Agent 自己使用的 **内置任务追踪工具**。
在多步任务（重构、调试、文档编写、迁移等）中，LLM 没有把"分解出的子步骤"显式化、也没有渠道把"正在做第几步"实时吐给用户，导致：

- Agent 在长任务中容易漏步、跳步、或重复尝试已完成的步骤（无结构化"已完成"证据）。
- 用户在 CLI 里只能从滚动的工具调用流中反推"还剩几步"，无固定进度区域。
- 当 Agent 并行调用多个工具时，用户无法区分"这是哪一步的工具调用"。

2026 Q1 行业共识：具备此类工具的 Agent（Claude Code TodoWrite、vscode-agent-todos）实测显著提高多步任务完成率（参见研究报告"Convergent patterns"）。

### 1.2 目标

- 为 `coder / researcher / reviewer` 三个带工具的可分派 Agent 提供一个名为 `todo_write` 的内置工具。
- 维护 `pending → in_progress → completed` 三态任务列表；每次调用提交**整张列表**（全量替换语义）。
- CLI 每次列表变更后，把当前快照打印到 scrollback（与其他事件同渠道，天然融入交互节奏）。
- HTTP 模式以独立事件 `todo_update` 向客户端推送全量快照。
- 状态**仅存活于当前会话**，不落盘、不污染 `vv/memories` 持久层。

### 1.3 成功标准（可验证）

| # | 标准 | 验证方式 |
|---|------|---------|
| S1 | 工具能被已注册 Agent 正确调用；schema 符合 OpenAI / Anthropic 协议 | 单元测试 + 集成测试（启动 coder Agent → LLM 调用 todo_write） |
| S2 | 列表是会话级的：同一 session 多轮调用共享；不同 session 完全隔离 | 单元测试覆盖：同 session 并发两次 RunStream 可见对方写入；跨 session 读取为空 |
| S3 | "最多一个 in_progress"被 handler **强制执行**（不是靠 prompt） | 单元测试：提交 2 个 in_progress → 返回结构化错误，list 保持上一版 |
| S4 | CLI 收到事件后在 scrollback 新印一块渲染后的 todo 列表（绿 ✔ / 蓝 ● / 灰 ☐） | 渲染单元测试 + 手动目视（CLI 流程内） |
| S5 | HTTP SSE 客户端能收到 `todo_update` 事件，payload 为全量 `{todos:[...]}` | 集成测试（httpapis）：驱动一次 tool call，读一条 `todo_update` 事件 |
| S6 | 会话结束不遗留状态（不写文件、无 goroutine 泄漏、无 memory store 写入） | 单元测试 + `go test -race` 干净 |
| S7 | 非目标 Agent（chat/planner/explorer）**看不到**此工具 | 工具注册测试断言 profile 过滤 |

---

## 2. 用户故事与验收条件

### US-1：开发者在 CLI 发起一个多步任务

- **场景**：用户说"帮我把 foo 模块改成 ctx 透传，再更新对应单测"。
- **预期**：
  1. Coder Agent 第一次输出后，应当调用 `todo_write` 提交 3~5 条 `pending` 条目。
  2. 开始第一步前把对应条目改 `in_progress`，完成后立即改 `completed`，一次只能一条 `in_progress`。
  3. CLI scrollback 每次工具调用后显示最新列表块。
  4. 任务完成后，最后一次调用所有条目为 `completed`。
- **拒绝条件**：
  - Agent 一次性把所有条目标 `completed` 再写代码 → S3 覆盖（至少有一条 in_progress 在活跃期内）。
  - Agent 并行置多条 in_progress → handler 返回错误。

### US-2：HTTP 客户端消费进度

- **场景**：SRE 写一个 dashboard 消费 `/v1/agents/.../stream` SSE。
- **预期**：每次 tool call 结束 + 列表发生变化时，SSE 产生一条 `event: todo_update` 的记录；`data` 是 JSON `{todos:[{content, active_form, status, id}], version}`。
- **拒绝条件**：`data` 只给 diff → 不符合"全量快照"约定。

### US-3：并发会话互不干扰

- **场景**：两个 HTTP 客户端并发连接，各自携带不同 `session_id`。
- **预期**：两个会话维护各自 todo 列表；彼此看不到对方的条目；一个会话完成不影响另一个。

---

## 3. 范围

### 3.1 In-scope

- 新增 Go 包 `vage/tool/todo/`：tool schema、handler、会话级 store、事件类型。
- 新增事件类型 `schema.EventTodoUpdate` + `schema.TodoUpdateData`。
- `vv/tools/` 注册入口新增 `RegisterTodo`；写入默认 6 工具之外作为一个可选扩展工具，默认开启。
- `vv/registries/` 把 `todo_write` 能力挂到 `coder/researcher/reviewer` 三个 profile。
- `vv/cli/render.go` + `vv/cli/cli.go` 的 `handleStreamEvent` 新增 `EventTodoUpdate` case，打印渲染块。
- 在 `coder/researcher/reviewer` 的 system prompt 里追加使用指引段落（3~5 行）。
- 单元测试 + 集成测试 + lint 全绿。

### 3.2 Out-of-scope

- 任务之间的依赖关系 DAG（Claude Tasks 风格的 `blockedBy`）。本轮仅扁平列表，避免过度工程。
- 跨会话持久化 / `/resume` 支持。P2-14（会话检查点）负责；此需求**正交**。
- CLI 固定顶端状态面板（类 Claude Tasks `Ctrl+T` 面板）。本轮复用 scrollback，不引入新布局。
- 增量 patch 语义（`todo_update` 单条更新）。本轮统一为 **全量替换**，降低 Agent 调用模式学习成本、贴合 Claude TodoWrite 约定。
- `todo_read` 工具（用于 Agent 自己读列表）。本轮不做——Agent 通过上次的 tool_result 已知列表；后续如果发现上下文被压缩后列表丢失，再补。
- 在 `vage` 已有的 `memory.Store` / `FileStore` 里落盘。本轮**明确不借道**，工具状态是纯内存 per-session map。

---

## 4. 受影响的角色 / 模型 / 流程 / 应用

| 维度 | 影响 |
|------|------|
| 角色（roles）| 无新角色；终端用户、Agent 的职责均不变。 |
| 模型（models）| 新增 `TodoItem{id, content, active_form, status}` 会话状态模型；不写入任何已有模型。 |
| 字典（dictionaries）| 新增枚举 `TodoStatus = pending \| in_progress \| completed`（与 Claude Code 对齐）。 |
| 流程（procedures）| 多步执行流程增加一个"Agent 先写 todo"的建议环节；不是强制门禁。 |
| 应用（applications）| `vv/cli`、`vv/httpapis`、`vv/agents` 三个子应用感知变更；MCP 模式同步（工具会出现在暴露列表里）。 |

---

## 5. 行业调研摘要（用于指导设计）

> 完整调研见 session 内 sub-agent 报告；此处摘关键决策点。

### 5.1 收敛的做法（我们跟随）

- **三态 `pending → in_progress → completed`**：Claude Code TodoWrite、vscode-agent-todos、OpenHands 1.5.0 一致采用。
- **`content` + `active_form` 双字段**：`content` 保存规范的祈使句（"Fix auth bug"）；`active_form` 现在分词（"Fixing auth bug"）供 UI 显示 spinner 文案。
- **"≥3 步任务才建议用"** 的 system prompt 引导，配合"开始某步前置 in_progress、完成后立即置 completed"。
- **side-panel / 专用渲染区**：Claude Tasks 面板、Cursor 2.0 sidebar、OpenHands task list panel。
- **vv 取舍**：本轮不做固定面板，改走"每次 tool call 结束时把列表打印到 scrollback"，避免 Bubble Tea 布局大改，同时保留后续加面板的可能。

### 5.2 分歧点（需要显式抉择）

| 分歧点 | Claude TodoWrite | Claude Tasks (v2.1.16+) | Aider/Cline/Cursor | **vv 选择** | 理由 |
|---|---|---|---|---|---|
| 写入语义 | 全量替换 | 增量 CRUD+ID | 自由 markdown | **全量替换** | Claude Code 模式 LLM 已大量熟悉；代码简单；"版本号单调递增"可直接充当幂等键。 |
| 任务标识 | 位置索引 | 稳定 ID | 无 | **稳定 ID（服务器分配）** | 全量替换不需要 ID 也能工作，但带 ID 后 HTTP 客户端可做 diff、未来接增量扩展不返工；"服务器分配"避免 LLM 随机给 ID 带来的乱序。 |
| 持久化 | 仅内存 | 用户目录 JSON | 仓库文件 | **仅内存（session-scoped）** | 与 P2-14 会话检查点解耦；memory 正交；默认零磁盘 I/O。 |
| 并发不变式 | 仅 prompt 提醒 | 仅 prompt 提醒 | 无 | **handler 强制执行** | 直接治理 Claude Code issue #6528 "fake completion" 的一个子症状（Agent 并行声称多条 in_progress）。 |
| 依赖关系 | 无 | 有 | 无 | **无** | 过度工程；P2-4 偏差重规划、P4-13 对话分支已覆盖更复杂的编排诉求。 |

### 5.3 已知失败模式（需要在测试和 prompt 里显式防御）

| 失败模式 | 来源 | 我们的应对 |
|---|---|---|
| Fake completion（批量标完成但没实际执行） | Claude issue #6528 | prompt 引导 + 用户可见性（每次调用都在 CLI 打印 → 社会压力）+ 不设置自动校验（我们没有办法可靠校验，坦然接受） |
| Bulk updates（一次翻多条状态） | OpenHands issue #9970 | prompt 明确要求"开始前→in_progress、完成后→completed，逐条" |
| Announce-without-write（口头说"更新 todo"但没调用） | 通用观察 | 不在产品侧做强校验；在 prompt 里给明确 trigger 条件 |
| Silent additions（偷偷加条目） | 通用 | 事件必发，用户能看到；不屏蔽 |
| 重建整张列表每轮 | TodoWrite 全量替换的副作用 | 我们保留版本号 `version`，客户端可做 diff；LLM 模式会吞 tokens，但此为行业共性 |
| 并行侧信道（Agent 把列表写成 `.md`） | Claude 观察 | prompt 明确："不要在工作区创建 TODO.md 文件，使用 todo_write 工具" |
| 运行时可用性不一致（issue #23874） | Claude 观察 | 工具 per-Agent 能力 profile 显式定义，profile 测试覆盖 |

---

## 6. 前置已知/现存代码能力（减少重复工作）

| 能力 | 位置 | 如何利用 |
|------|------|---------|
| Tool 接口 / Registry | `vage/tool/tool.go`, `vage/tool/registry.go` | `TodoTool` 实现 `ToolDef()+Handler()`；用 `registry.Register` 挂载；Register 支持按 profile 过滤。 |
| Session context key | `vv/memories/session.go:SessionIDFrom(ctx)` | Handler 从 ctx 取 sessionID 作为 store 的 key。 |
| Event sink | `vage/agent/taskagent/task.go` 中的 `eventSink` 透传到 `tool.Handler` 是**没有**的——当前 tool handler 不能发事件 | 需要设计：tool handler 通过 ctx 注入的 "event emitter" 发自定义事件；或在 handler 返回 `ToolResult` 后由 TaskAgent 层主动 emit。后者更低侵入。我们走"ctx 注入 emitter"方案——见设计阶段。 |
| CLI 渲染 / scrollback | `vv/cli/cli.go:handleStreamEvent`、`vv/cli/render.go` | 参考 `EventToolResult` / `EventSubAgentEnd` 的模式：新增 `renderTodoList(todos, depth)` 函数、在 switch 加 `case schema.EventTodoUpdate:`。 |
| HTTP SSE | `vv/httpapis/http.go` → `vage/service/` | 事件由 `RunStream` 通路自动推送，新增事件类型即可，无需改 service 层。 |
| Agent factory options | `vv/registries/registry.go:FactoryOptions`、`vv/agents/coder.go` | 新增 `FactoryOptions.TodoStore *todo.Store`（或通过 `TodoEnabled bool`），在 Register 里按 profile 挂载。 |

---

## 7. 发现的不一致（留给 documenter 决定是否处理）

- `doc/prd/overview.md` "Does Not Cover" 段目前列 TodoWrite（若列入）需要同步更新——待 documenter 阶段核对。
- `doc/prd/architecture/architecture.md` 工具清单需要追加 `todo_write` 工具条目。
- `vv/CLAUDE.md` Tool Names 段目前枚举 "bash, read, write, edit, glob, grep" 六工具；documenter 阶段决定是否把 todo_write 合并进来（可能是第 7 个工具）。

---

## 8. 假设与权衡（显式登记）

- **假设 A**：`coder/researcher/reviewer` 的 prompt 有足够空间再塞 3~5 行 TodoWrite 说明而不超过任何现有 prompt 预算。若 prompt 拥挤，**降级方案**：仅把说明加到 coder（多步任务最常发生），其他两个 Agent 可见工具但不强推荐使用。
- **假设 B**：Bubble Tea TUI 当前"每次 tool call 结果 Println 到 scrollback"的模式对"多行渲染块 todo list"依然友好（不会把列表撕裂在终端 resize 之间）。若观察不佳，**降级方案**：改为极简单行摘要 `[todos: 2✔ 1●在做 3☐]`，详情下移到 debug 日志。
- **假设 C**：HTTP 客户端能处理未知事件类型（SSE 行为即是"不认识就忽略"）。已知 `vage/service/` SSE 实现走通用 JSON 编码 + type 字段——安全。

---

## 9. 集成测试计划要点（供 tester 阶段参考，如被纳入）

- ✅ coder Agent → LLM 调用 todo_write → 首次写入 3 条 pending → CLI 输出渲染块 → 验证条目数与状态。
- ✅ 连续两次调用，第二次含 `in_progress + completed` 混合 → version 单调递增。
- ✅ 提交两条 in_progress → handler 返回 `ErrorResult`，列表未变动。
- ✅ 两个独立 session 并发写入 → 各自读取自身列表，互不可见。
- ✅ HTTP SSE 客户端接收 `event: todo_update` → `data.todos` 全量 + `data.version` 正确。
- ✅ `chat` / `planner` / `explorer` Agent 的 tool registry 中无 `todo_write`（profile 过滤断言）。
- ✅ 无会话 ID 时调用 todo_write → 返回明确错误（不让工具在"无法定位 store"的情况下静默成功）。
