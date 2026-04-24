# P1-9 · TodoWrite 技术设计

> 对应需求文档：`requirement.md`。
> 原则：最小改动、复用现有模式、不引入新的抽象层。

---

## 1. 总览

```
┌──────────────────────────────────────────────────────────────┐
│ LLM Tool Call                                                │
│   name=todo_write, args={"todos":[{content,active_form,id?,  │
│                                    status}]}                 │
└───────────────┬──────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────────────────┐
│ vage/tool/todo.Handler (新增)                                │
│  1) 从 ctx 取 sessionID                                      │
│  2) 解析 args、校验（非空、status 合法、≤1 in_progress）      │
│  3) Store.Apply(sessionID, items) → 返回新快照 + 版本号      │
│  4) ctx 里取 EventEmitter.Emit(EventTodoUpdate,{todos,version})│
│  5) 返回 ToolResult（ASCII 勾号列表 + 版本号，给 LLM 回显）   │
└───────────────┬──────────────────────────────────────────────┘
                │ event
                ▼
┌──────────────────────────────────────────────────────────────┐
│ TaskAgent RunStream eventSink → CLI scrollback / HTTP SSE   │
│  CLI: renderTodoList() 打印带色勾号列表                      │
│  HTTP: SSE event: todo_update                                │
└──────────────────────────────────────────────────────────────┘
```

核心约束：
- **只在 vage/tool/todo 与 vv 两层改**；不动 aimodel。
- **不新增抽象接口、不新增 DI 容器**；通过 `context.Context` 注入 sessionID 与事件发射器。
- **全量替换语义**；不支持 diff/patch 写入。

---

## 2. 包布局

```
vage/
  schema/
    event.go                       # (改) 新增 EventTodoUpdate + TodoUpdateData
    context.go                     # (新) ctx 注入工具：WithSessionID / WithEventEmitter
  agent/taskagent/
    task.go                        # (改) 在 executeToolCall 前注入 ctx
  tool/
    todo/                          # (新)
      todo.go                      # ToolDef, Handler, Register
      store.go                     # 会话级 Store（sync.Mutex map）
      store_test.go
      todo_test.go
      prompt.go                    # 导出一段"TodoWrite 使用指引"给 Agent prompt 拼装
vv/
  tools/
    tools.go                       # (改) 三个 Register* 都挂 todo_write（read-only 分支不挂）
  registries/
    registry.go                    # (改) FactoryOptions 增加 TodoStore *todo.Store
  agents/
    coder.go                       # (改) prompt 追加 TodoWrite 段
    researcher.go                  # (改) 同上
    reviewer.go                    # (改) 同上
  cli/
    render.go                      # (改) 新增 renderTodoList
    cli.go                         # (改) handleStreamEvent 新增 case EventTodoUpdate
  configs/
    config.go                      # (改) Agents.TodoEnabled（默认 true）
  setup/
    setup.go                       # (改) 构造 todo.Store 并传入 FactoryOptions
```

---

## 3. 数据模型

### 3.1 Tool input (JSON Schema)

```jsonc
{
  "type": "object",
  "properties": {
    "todos": {
      "type": "array",
      "description": "Full list of todos (replaces the current list entirely).",
      "items": {
        "type": "object",
        "properties": {
          "id":          { "type": "string", "description": "optional stable id; server will assign if empty" },
          "content":     { "type": "string", "description": "imperative form, e.g., 'Refactor foo module'" },
          "active_form": { "type": "string", "description": "present-continuous form, e.g., 'Refactoring foo module'" },
          "status":      { "type": "string", "enum": ["pending","in_progress","completed"] }
        },
        "required": ["content", "active_form", "status"]
      }
    }
  },
  "required": ["todos"]
}
```

### 3.2 In-memory types

```go
// vage/tool/todo/store.go
type Status string
const (
    StatusPending    Status = "pending"
    StatusInProgress Status = "in_progress"
    StatusCompleted  Status = "completed"
)

type Item struct {
    ID         string `json:"id"`
    Content    string `json:"content"`
    ActiveForm string `json:"active_form"`
    Status     Status `json:"status"`
}

type Snapshot struct {
    Version int64  `json:"version"`
    Items   []Item `json:"items"`
}

type Store struct {
    mu       sync.Mutex
    sessions map[string]*Snapshot // sessionID -> latest snapshot
    idCounter uint64              // monotonic; assigned to items without id
}

func NewStore() *Store
func (s *Store) Apply(sessionID string, items []Item) (Snapshot, error) // validates, deep-copies, bumps version
func (s *Store) Get(sessionID string) Snapshot                          // safe copy; empty snapshot for unknown session
func (s *Store) Clear(sessionID string)                                 // optional housekeeping; not called by P1-9
```

`Apply` 里强制校验：
- `items` 可以为空（允许清空）。
- 每个 item：`content`/`active_form` 非空；`status` 必须是三值之一。
- `in_progress` 状态的 item 数 ≤ 1（否则 `ErrTooManyInProgress`）。
- 保留旧 id：如果 input 的 item 带与现有 id 匹配的 `id`，沿用；否则分配 `todo_<N>`。
- 不支持 10k 条以上列表（硬上限 100，超过 `ErrTooManyItems`，防 prompt 轰炸）。

### 3.3 Event payload

```go
// vage/schema/event.go
const EventTodoUpdate = "todo_update"

type TodoUpdateData struct {
    Version int64      `json:"version"`
    Items   []TodoItem `json:"items"`
}

type TodoItem struct {
    ID         string `json:"id"`
    Content    string `json:"content"`
    ActiveForm string `json:"active_form"`
    Status     string `json:"status"`
}

func (TodoUpdateData) eventData() {}
```

注意：`TodoItem` 是 schema 包的独立类型（不直接暴露 `todo.Item`），避免 schema 反向依赖 tool。tool 内部在发射事件前做一次投影即可。

---

## 4. 上下文注入

### 4.1 新增 `vage/schema/context.go`

```go
package schema

import "context"

type sessionIDCtxKey struct{}
type emitterCtxKey struct{}

// Emitter sends a single event into the active stream.
type Emitter func(Event)

func WithSessionID(ctx context.Context, sessionID string) context.Context {
    if sessionID == "" { return ctx }
    return context.WithValue(ctx, sessionIDCtxKey{}, sessionID)
}
func SessionIDFromContext(ctx context.Context) string {
    v, _ := ctx.Value(sessionIDCtxKey{}).(string)
    return v
}

func WithEmitter(ctx context.Context, e Emitter) context.Context {
    if e == nil { return ctx }
    return context.WithValue(ctx, emitterCtxKey{}, e)
}
func EmitterFromContext(ctx context.Context) Emitter {
    v, _ := ctx.Value(emitterCtxKey{}).(Emitter)
    return v
}
```

**为什么放在 schema**：
- 已经被 agent/tool/service 都依赖（反向依赖风险最低）。
- 与现有 `EventData` 等类型紧邻，语义上属于"执行上下文的调度数据"。
- vv/memories/session.go 继续保留其自己的 ctx key（作用是 memory store 访问控制，不冲突）。

### 4.2 TaskAgent 的注入点

`vage/agent/taskagent/task.go` 中所有调用 `a.executeToolCall(ctx, tc)` / `a.toolRegistry.Execute(...)` 的地方，都需要在调用前注入：

```go
func (a *Agent) toolCallCtx(parent context.Context, rc *runContext, sink func(schema.Event) error) context.Context {
    ctx := schema.WithSessionID(parent, rc.sessionID)
    if sink != nil {
        emitter := schema.Emitter(func(e schema.Event) {
            if e.SessionID == "" { e.SessionID = rc.sessionID }
            if e.Timestamp.IsZero() { e.Timestamp = time.Now() }
            _ = sink(e) // 已有的 sink 语义：错误不阻塞主流程
        })
        ctx = schema.WithEmitter(ctx, emitter)
    }
    return ctx
}
```

在 `callTools`（并行/串行两条路径都要）的每次迭代前，把 `ctx` 替换为 `toolCallCtx(ctx, rc, eventSink)`。

**幂等性**：注入只增加 value，不破坏已有 ctx；对其他 tool（read/write/bash…）完全透明（它们不读这两个 key）。

**Run（非 stream）路径**：沿用同样注入，但 emitter 指向 `a.dispatch`（hook manager）。对 todo_update 的作用是：hook 订阅者（如 JSONL trace）依然能收到；但 CLI 场景不走 Run 路径，无副作用。

---

## 5. Handler 关键流程

```go
// vage/tool/todo/todo.go

type Tool struct{ store *Store }

func (t *Tool) ToolDef() schema.ToolDef {
    return schema.ToolDef{
        Name:        "todo_write",
        Description: toolDescription, // 见下一节
        Source:      schema.ToolSourceLocal,
        ReadOnly:    false,           // 写入共享会话状态
        Parameters:  parametersSchema,
    }
}

func (t *Tool) Handler() tool.ToolHandler {
    return func(ctx context.Context, _ string, args string) (schema.ToolResult, error) {
        sessionID := schema.SessionIDFromContext(ctx)
        if sessionID == "" {
            return schema.ErrorResult("", "todo_write: session id missing from context"), nil
        }

        var in struct { Todos []Item `json:"todos"` }
        if err := json.Unmarshal([]byte(args), &in); err != nil {
            return schema.ErrorResult("", "todo_write: invalid JSON: "+err.Error()), nil
        }

        snap, err := t.store.Apply(sessionID, in.Todos)
        if err != nil {
            // user-visible, non-fatal for the agent — it can retry with corrected list
            return schema.ErrorResult("", "todo_write: "+err.Error()), nil
        }

        if em := schema.EmitterFromContext(ctx); em != nil {
            em(schema.NewEvent(
                schema.EventTodoUpdate, "", sessionID,
                toEventData(snap), // projects Item → schema.TodoItem
            ))
        }

        return schema.TextResult("", formatForLLM(snap)), nil
    }
}

func Register(reg *tool.Registry, store *Store) error {
    if store == nil {
        return errors.New("todo.Register: store is nil")
    }
    t := &Tool{store: store}
    return reg.RegisterIfAbsent(t.ToolDef(), t.Handler())
}
```

`formatForLLM(snap)` 生成让 LLM 方便回显/对齐的简短文本：

```
Todo list updated (v3, 4 items):
[x] Read existing code
[>] Refactoring foo module
[ ] Update unit tests
[ ] Run lint
```

这是返回给模型的"工具结果"，不是用户看到的 CLI 渲染；CLI 自己有彩色版本（见 §7）。

---

## 6. 工具描述文案（todo_write）

集中在 `vage/tool/todo/todo.go` 顶部常量，方便未来调整：

```
Use this tool to plan and track multi-step work. The list you submit
REPLACES the previous list entirely — pass all items every call.

WHEN to use (proactively, without being asked):
  - the task has 3 or more distinct steps
  - the task is non-trivial and would benefit from a written plan
  - the user provides multiple tasks or asks for planning

WHEN NOT to use:
  - a single trivial change (read one file, tweak one line)
  - purely informational answers
  - tasks that finish in one tool call

STATUS LIFECYCLE (must follow):
  pending → in_progress → completed
  - Mark an item in_progress BEFORE you start working on it.
  - Mark it completed IMMEDIATELY when it's done. Do not batch.
  - Only ONE item may be in_progress at a time. The tool will reject
    calls that violate this invariant.
  - If you abandon an item, remove it from the list (do not keep
    completed/in_progress items that no longer reflect reality).

FIELDS:
  content:     imperative present tense, e.g. "Fix the auth bug"
  active_form: present continuous, e.g. "Fixing the auth bug"
  status:      one of {pending, in_progress, completed}
  id:          optional; if you keep the list stable across calls,
               the server-assigned id lets the UI diff cleanly

DO NOT:
  - create TODO.md files or other side-channel tracking files;
    use only this tool
  - announce "I'll update the todo list" without actually calling
    this tool
  - flip multiple items to completed in one call without having
    executed them
```

Prompt 引导段（追加到 coder/researcher/reviewer）：

```
## Task Tracking
- **todo_write**: maintain a live plan for multi-step work. Follow the
  tool's description exactly. Prefer this over free-form planning text.
```

保持最短，避免吃系统 prompt 预算。

---

## 7. CLI 渲染

`vv/cli/render.go` 新增：

```go
var (
    todoHeaderStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("11")).Bold(true) // yellow bold
    todoDoneStyle   = lipgloss.NewStyle().Foreground(lipgloss.Color("10"))            // green
    todoActiveStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("14")).Bold(true) // cyan bold
    todoPendStyle   = lipgloss.NewStyle().Foreground(lipgloss.Color("8"))             // dim
)

func renderTodoList(data schema.TodoUpdateData, depth int) string {
    var b strings.Builder
    header := fmt.Sprintf("Todos (v%d, %d items)", data.Version, len(data.Items))
    b.WriteString(todoHeaderStyle.Render(header))
    b.WriteString("\n")
    for _, it := range data.Items {
        var marker, text string
        switch it.Status {
        case "completed":
            marker, text = todoDoneStyle.Render("[x]"), todoDoneStyle.Render(it.Content)
        case "in_progress":
            marker, text = todoActiveStyle.Render("[>]"), todoActiveStyle.Render(it.ActiveForm)
        default:
            marker, text = todoPendStyle.Render("[ ]"), todoPendStyle.Render(it.Content)
        }
        fmt.Fprintf(&b, "  %s %s\n", marker, text)
    }
    return indentBlock(strings.TrimRight(b.String(), "\n"), depth)
}
```

`vv/cli/cli.go:handleStreamEvent` 加一条 case：

```go
case schema.EventTodoUpdate:
    if data, ok := event.Data.(schema.TodoUpdateData); ok {
        rendered := renderTodoList(data, m.toolDepth())
        m.app.messages = append(m.app.messages, DisplayMessage{
            Role:      RoleTool,
            Content:   rendered,
            Timestamp: time.Now(),
            Rendered:  true,
        })
        return m, tea.Println(rendered)
    }
```

**为什么和 tool-result 独立**：
- `ToolResultData` 承载的是"LLM 看到的文本"（`formatForLLM`）；`TodoUpdateData` 承载的是"UI 结构化数据"。分离是为了客户端（包括 HTTP SSE）能直接消费结构，不必解析文本。
- 分离后，CLI 对 tool_result 打印 `todo_write: (n items)` 一类的占位，避免双重打印——在 renderer 里做：如果 `toolName == "todo_write"`，`renderToolCallResult` 直接返回 `""`（已有机制："若渲染返回空字符串就不打印"）。

---

## 8. HTTP 集成

`vv/httpapis/` 通过 `vage/service/` 的 SSE 通路自动推送 `EventTodoUpdate`。不需要改 service 层：事件会被 JSON 编码 + `type: todo_update` 送到客户端。

约定：客户端若不识别 `todo_update` 应静默忽略（SSE 默认行为）。

---

## 9. 配置与注册

### 9.1 配置

`vv/configs/config.go` 在 `Agents` 下新增：

```go
type AgentsConfig struct {
    ...
    // TodoEnabled 控制是否在有工具的 Agent 上注册 todo_write。
    // 默认 true（nil pointer 视作 true）。设 false 可关闭。
    TodoEnabled *bool `yaml:"todo_enabled"`
}
```

Env 覆盖：`VV_AGENTS_TODO_ENABLED`（类比 PromptCaching，使用 `EffectiveTodoEnabled()` 解析器）。

### 9.2 setup.Init 连接

```go
// vv/setup/setup.go（摘要）
store := todo.NewStore()
// 注入到三个工具 registry
for _, reg := range []*tool.Registry{fullReg, reviewReg} {
    if EffectiveTodoEnabled(cfg.Agents.TodoEnabled) {
        _ = todo.Register(reg, store)
    }
}
// read-only（researcher）的处理
// → researcher 也受益于 todo_write，所以 readOnlyReg 也挂。
// 唯一不挂的场景：chat（工具注册集为空）、planner/explorer（非 dispatchable、短任务）。
```

### 9.3 registries.FactoryOptions

不新增字段。store 通过 tool registry 的 Register 闭包已经 capture，Agent 不需要直接感知 store。

---

## 10. 关键权衡与备选方案

| 决策 | 选择 | 被放弃的方案 | 理由 |
|---|---|---|---|
| sessionID 如何到 tool | ctx 注入（新增 `schema.WithSessionID`） | 让 tool 自己维护最近一次 sessionID / 传 RunRequest 指针 | 前者破坏并发语义；后者污染 tool 接口 |
| event 如何到 stream | ctx 注入 Emitter 闭包 | TaskAgent 扫描 ToolResult 的侧信道字段 | 闭包侵入小、无需改 `schema.ToolResult`；类型安全 |
| store 作用域 | 进程单例 per Session | 每 Agent 一个 store | 多 Agent（coder → reviewer）在同 session 需共享列表 |
| 存储介质 | 纯内存 | SQLite（复用 P1-6） | 与 memory 正交；会话结束即丢弃；未来可通过 P2-14 检查点持久化 |
| 状态转移约束 | handler 强制"最多一个 in_progress" | 仅 prompt 引导 | Claude issue #6528 直接动因；低成本高收益 |
| 消息回显格式 | `[x] [>] [ ]` ASCII | emoji（✅🔧❌） | 零依赖、终端兼容性更广；CLI 渲染仍用 lipgloss 着色 |
| 增量 vs 全量 | 全量替换 | CRUD + id | 对齐 Claude TodoWrite 既有模式；降低 LLM 调用学习成本；id 由服务器分配提供未来拓展空间 |

---

## 11. 错误与降级

| 条件 | 行为 |
|---|---|
| sessionID 缺失 | `ToolResult` Error，列表不变；LLM 可继续 |
| JSON 解析失败 | Error，列表不变 |
| 多于 1 个 in_progress | Error（`todo_write: only one in_progress item allowed`），列表不变 |
| 超过 100 条 | Error（`todo_write: too many items (max 100)`） |
| emitter 为 nil（Run 非 stream 且无 hook manager） | 静默跳过事件发射，`ToolResult` 仍正常返回 |
| Store 并发写同一 session | Mutex 串行化；每次都拿到一个新的 version |

**非 fatal 原则**：任何错误都返回 `ErrorResult`（`error` 返回值始终为 nil），让 Agent 在下一轮 LLM call 修复输入。参考 askuser / write 等既有 tool 的做法。

---

## 12. 测试计划（单测 / 覆盖率目标）

### 12.1 vage/tool/todo

- `store_test.go`
  - Apply 空列表 → version=1，items=nil
  - Apply 合法列表 → version=1，id 自动分配 `todo_1..N`
  - 第二次 Apply 相同 id → 保留 id；version=2
  - Apply 两个 in_progress → `ErrTooManyInProgress`，状态不变
  - Apply 非法 status → 错误
  - 空 content / 空 active_form → 错误
  - 两个 session 并发 Apply 100 次 → race-free（`go test -race`）；各自 version 单调
  - >100 items → `ErrTooManyItems`
  - Get 未知 session → 空快照（version=0）
- `todo_test.go`
  - Handler: ctx 无 sessionID → ErrorResult
  - Handler: 非法 JSON → ErrorResult
  - Handler: 成功路径 → 触发 emitter 一次，payload 匹配新快照
  - Handler: emitter 为 nil 时不 panic
  - formatForLLM 对三种状态都正确渲染

### 12.2 vage/schema/context

- WithSessionID("") 不写入
- WithEmitter(nil) 不写入
- 读取缺失值返回零值

### 12.3 vage/agent/taskagent

- 新增一个表驱动的 mock 工具，用 `schema.SessionIDFromContext`/`EmitterFromContext` 断言 ctx 已注入（stream 和非 stream 各一个用例）。

### 12.4 vv/cli/render_test.go

- renderTodoList 三种状态的渲染字符串（去色后）包含对应 marker 与文本
- 空列表：header + 0 items

### 12.5 vv/tools/tools_test.go（现有）

- 更新："todo_write" 存在于 `Register` 和 `RegisterReviewTools` 和 `RegisterReadOnly` 的输出
- "todo_write" 不存在于……（如果我们决定 chat 不走 registry，则无需断言）

### 12.6 集成测试 `vv/integrations/cli_tests`

- 启动 vv（stream mode），驱动 coder agent 调用 todo_write 两轮；读取事件流，断言收到两条 `EventTodoUpdate`，version 1→2。

### 12.7 集成测试 `vv/integrations/httpapis_tests`

- 走 `/v1/agents/coder/stream`，驱动一次 todo_write，SSE 解析到 `event: todo_update` 一条；payload.version=1 + items 三条。

---

## 13. 里程碑与代码量估算

| 块 | 预估 |
|---|---|
| vage/schema/context.go + event.go 改动 | 80 LOC |
| vage/tool/todo 包（store + todo + 测试） | 380 LOC |
| vage/agent/taskagent 注入 ctx | 30 LOC + 单测 60 LOC |
| vv/tools/tools.go 三分支挂载 | 30 LOC |
| vv/cli/render.go + cli.go | 50 LOC + 单测 60 LOC |
| vv/agents coder/researcher/reviewer prompt 追加 | 15 LOC |
| vv/configs TodoEnabled | 30 LOC |
| vv/setup wiring | 20 LOC |
| 集成测试（cli + http） | 120 LOC |
| **合计** | **~875 LOC**，其中测试占 ~500 LOC；贴近需求表的"~400 LOC + 测试"预估，偏上。 |

---

## 14. 已知未覆盖（下游需求接续）

- P2-14 会话检查点：上线后可把 Snapshot 序列化进 checkpoint；TodoWrite 本身无需改动。
- P4-13 对话分支：如需支持"从任意历史点 fork 后继承 todo 列表"，Store 需要按 (sessionID, branchID) 扩展 key——届时再做。
- UI 侧"专用面板"：如后续 CLI 把 todo 提到固定区域（类 Claude Tasks `Ctrl+T`），只需重用 `renderTodoList`+新增 View 区域，无需改协议。
