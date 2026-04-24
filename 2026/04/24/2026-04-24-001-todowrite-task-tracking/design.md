# P1-9 · TodoWrite 技术设计（v2，已并入评审意见）

> 对应需求文档：`requirement.md`。评审记录：`design-review.md`。原始版本：`design-raw.md`。
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
│  2) 解析 args、校验（非空、status 合法、≤1 in_progress、≤100）│
│  3) Store.Apply(sessionID, items) → 返回新快照 + 版本号      │
│  4) ctx 里取 Emitter.Emit(EventTodoUpdate,{items,version})   │
│  5) 返回 ToolResult（极简确认文本："ok (v3, 4 items)"）       │
└───────────────┬──────────────────────────────────────────────┘
                │ event
                ▼
┌──────────────────────────────────────────────────────────────┐
│ TaskAgent.executeToolBatch 顶部注入的 Emitter                │
│   → RunStream → CLI scrollback / HTTP SSE                    │
│  CLI: renderTodoList() 打印带色勾号列表                      │
│  HTTP: SSE `event: todo_update`                              │
└──────────────────────────────────────────────────────────────┘
```

核心约束：
- **只在 `vage/tool/todo`、`vage/schema`、`vage/agent/taskagent` 与 `vv` 层改**；不动 aimodel。
- **不新增抽象接口、不新增 DI 容器**；通过 `context.Context` 注入 sessionID 与事件发射器。
- **全量替换语义**；不支持 diff/patch 写入。
- ctx 注入在 TaskAgent **执行 tool 的唯一 choke point**（`executeToolBatch` 顶部），不在 Run / RunStream 两处重复。

---

## 2. 包布局

```
vage/
  schema/
    event.go                       # (改) 新增 EventTodoUpdate + TodoUpdateData + TodoItem
    context.go                     # (新) ctx 注入工具：WithSessionID / WithEmitter
  agent/taskagent/
    task.go                        # (改) executeToolBatch 顶部注入 ctx
  tool/
    todo/                          # (新)
      todo.go                      # ToolDef, Handler, Register
      store.go                     # 会话级 Store（sync.Mutex map）
      store_test.go
      todo_test.go
      prompt.go                    # 导出"TodoWrite 使用指引"文案常量，便于 vv 拼装 prompt
vv/
  tools/
    tools.go                       # 不改动；仍是"挂 vage 内置工具"的薄封装
  registries/
    registry.go                    # 不改动；FactoryOptions 保持不变
  agents/
    coder.go                       # (改) prompt 追加 TodoWrite 段
    researcher.go                  # (改) 同上（弱化一行）
    reviewer.go                    # (改) 同上（弱化一行）
  cli/
    render.go                      # (改) 新增 renderTodoList；renderToolCallResult 对 "todo_write" 返回 ""
    cli.go                         # (改) handleStreamEvent 新增 case EventTodoUpdate
  setup/
    setup.go                       # (改) 构造 todo.Store、按 profile 对三个 registry 调用 todo.Register
```

> 和 raw 版对比：`vv/tools/tools.go` 与 `vv/configs/config.go` 都**不再改动**。`todo.Register` 挪到 `setup.Init`，不透过 `FactoryOptions` 或 `Register*` 函数。理由：`vv/tools` 是"把 vage 工具挂上来"的薄层，不该知道 todo store 实例；`setup` 本来就是 wiring 层。

---

## 3. 数据模型

### 3.1 Tool input (JSON Schema)

```jsonc
{
  "type": "object",
  "properties": {
    "todos": {
      "type": "array",
      "description": "Full list of todos (replaces the current list entirely). Pass [] or null to clear.",
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

const maxItemsPerList = 100  // prompt-bombing guard; adjustable.

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
    mu        sync.Mutex
    sessions  map[string]*Snapshot // sessionID -> latest snapshot
    idCounter uint64               // monotonic; assigned to items without id
}

func NewStore() *Store
func (s *Store) Apply(sessionID string, items []Item) (Snapshot, error) // validates, deep-copies, bumps version
func (s *Store) Get(sessionID string) Snapshot                          // safe copy; empty snapshot for unknown session
func (s *Store) Clear(sessionID string)                                 // reserved for P2-14 checkpoint; not called by P1-9
```

`Apply` 校验规则：
- `items` 可为 nil 或 []（语义：清空；version 仍 +1）。
- 每个 item：`content`/`active_form` 非空；`status` 必须是三值之一。
- `in_progress` 状态的 item 数 ≤ 1（否则 `ErrTooManyInProgress`）——**handler 强制，不信 prompt**。
- 保留旧 id：如果 input item 带与现有 id 匹配的 `id`，沿用；否则分配 `todo_<N>`。
- 总数 ≤ `maxItemsPerList`（否则 `ErrTooManyItems`）。

**Store 不变式（developer 阶段测试覆盖）：**
1. 多个 Agent（coder → reviewer → coder）在**同一 sessionID** 写入时，`Snapshot.Version` 严格单调递增——因为同一个 mutex 串行化所有写入。
2. `sessionID == ""` 的调用被 handler 拦截为 `ErrorResult`，**绝不**走 "global/default" fallback（防止跨会话串台）。

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

`TodoItem` 是 schema 包的独立类型（不直接暴露 `todo.Item`），避免 schema 反向依赖 tool 形成 import 循环。tool handler 在发射事件前做一次 ~20 行的投影（`toEventData(snap)`）。

---

## 4. 上下文注入

### 4.1 新增 `vage/schema/context.go`

```go
package schema

import "context"

type sessionIDCtxKey struct{}
type emitterCtxKey struct{}

// Emitter sends a single event into the active stream. Returning an error
// normally means the stream is shutting down; callers ignore it (the run is
// terminating anyway).
type Emitter func(Event) error

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
- 已经被 agent / tool / service 都依赖（反向依赖风险最低）。
- 与现有 `EventData` 等类型紧邻，语义上属于"执行上下文的调度数据"。
- `vv/memories/session.go` 继续保留其自己的 ctx key（作用是 memory store 访问控制，**不冲突**，因为 key 类型不同且相互不感知）。

### 4.2 TaskAgent 注入点（单一 choke point）

**只在一个地方注入**：`vage/agent/taskagent/task.go` 的 `executeToolBatch` 顶部——这是 Run 与 RunStream 两条路径执行工具调用的共同入口（见 `task.go:928` 与 `task.go:1293`）。

```go
func (a *Agent) executeToolBatch(
    ctx context.Context,
    rc *runContext,
    agentID string,
    toolCalls []aimodel.ToolCall,
    emitResultEvent bool,
    eventSink func(schema.Event) error,
) ([]aimodel.Message, error) {
    // --- NEW: inject per-call context for tool handlers ---
    ctx = schema.WithSessionID(ctx, rc.sessionID)
    if eventSink != nil {
        ctx = schema.WithEmitter(ctx, schema.Emitter(eventSink))
    }
    // --- end new ---

    // ... existing code unchanged ...
}
```

**幂等性**：注入只增加 value，不改动既有 ctx；对其他 tool（read/write/bash/…）完全透明（它们不读这两个 key）。

**错误处理**：`eventSink` 返回错误只发生在 `RunStream` 被关闭时；此时整个 run 都在终止，Emitter 闭包直接把错误当作无操作（`_ = sink(e)`）即可。

**冗余消除**：不再由 Emitter 补齐 `Event.SessionID` / `Event.Timestamp`——handler 调 `schema.NewEvent(...)` 时自己就传齐了（`NewEvent` 内部已经填 timestamp）。原设计的 `toolCallCtx` helper 因此省掉。

**关于 Run（非 stream）路径**：`callTools` 在 `task.go:924` 构造 `sink := func(ev){a.dispatch(ctx,ev);return nil}` 并传给 `executeToolBatch`。注入发生在同一个 `executeToolBatch`，因此 Run 路径也自然收到 sessionID 和 Emitter；Emitter 会把事件喂回 `a.dispatch`，供 hook.Manager 订阅者（JSONL trace 等）消费。CLI 场景不走这条路径，无副作用。

---

## 5. Handler 关键流程

```go
// vage/tool/todo/todo.go

type Tool struct{ store *Store }

func (t *Tool) ToolDef() schema.ToolDef {
    return schema.ToolDef{
        Name:        "todo_write",
        Description: toolDescription, // 见 §6
        Source:      schema.ToolSourceLocal,
        ReadOnly:    true,             // 见说明
        Parameters:  parametersSchema,
    }
}

func (t *Tool) Handler() tool.ToolHandler {
    return func(ctx context.Context, _ string, args string) (schema.ToolResult, error) {
        sessionID := schema.SessionIDFromContext(ctx)
        if sessionID == "" {
            return schema.ErrorResult("", "todo_write: session id missing from context"), nil
        }

        var in struct{ Todos []Item `json:"todos"` }
        if err := json.Unmarshal([]byte(args), &in); err != nil {
            return schema.ErrorResult("", "todo_write: invalid JSON: "+err.Error()), nil
        }
        // nil and [] are both treated as "clear the list".

        snap, err := t.store.Apply(sessionID, in.Todos)
        if err != nil {
            return schema.ErrorResult("", "todo_write: "+err.Error()), nil
        }

        if em := schema.EmitterFromContext(ctx); em != nil {
            _ = em(schema.NewEvent(
                schema.EventTodoUpdate, "", sessionID,
                toEventData(snap), // projects Item → schema.TodoItem
            ))
        }

        return schema.TextResult("", fmt.Sprintf("ok (v%d, %d items)", snap.Version, len(snap.Items))), nil
    }
}

func Register(reg tool.ToolRegistry, store *Store) error {
    if store == nil {
        return errors.New("todo.Register: store is nil")
    }
    t := &Tool{store: store}
    if r, ok := reg.(*tool.Registry); ok {
        return r.RegisterIfAbsent(t.ToolDef(), t.Handler())
    }
    return reg.Register(t.ToolDef(), t.Handler())
}
```

### 5.1 `ReadOnly = true` 的含义

`vage/tool` 中 `ToolDef.ReadOnly` 语义是"不会对**工作区（文件系统 / 外部副作用）**产生影响"。`todo_write` 只写**会话级内存状态**，不碰磁盘，不修改用户文件。因此：
- 标 `ReadOnly: true`。
- researcher（read-only profile）挂载 `todo_write` 与其"只读"声明保持一致——不违反用户合同。

### 5.2 返回给 LLM 的文本

返回极简确认串 `ok (v3, 4 items)`，而不是把整张列表再列一遍：
- LLM 在同一轮已经看到自己提交的 tool_call arguments，没必要让 tool_result 再重述。
- 节省 tokens；同时让 CLI 不需要"抑制 tool_result 打印"的特殊逻辑（但我们仍然做 §7 的抑制，双保险）。

---

## 6. 工具描述文案（todo_write）

集中在 `vage/tool/todo/todo.go` 顶部常量：

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
  - If you abandon an item, remove it from the list.

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

### 6.1 Agent prompt 引导段（差异化、精简）

**coder**（多步任务高发）：

```
## Task Tracking
- **todo_write**: maintain a live plan for multi-step work (≥3 steps).
  Follow the tool's description exactly — prefer this over free-form
  planning text.
```

**researcher**（可能多步，但场景窄一些）：

```
## Task Tracking
- **todo_write** is available. Use it when your investigation spans
  3+ files or modules; skip it for short lookups.
```

**reviewer**（短反馈居多）：

```
## Task Tracking
- If the review spans 3+ files, use **todo_write** to track progress.
```

保持每段 ≤ 2 行，避免吃系统 prompt 预算。

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

`vv/cli/cli.go:handleStreamEvent` 新增 case：

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

**抑制 tool_result 双重打印**：`renderToolCallResult` 在 `toolName == "todo_write"` 时返回空字符串（复用现有"空串即不打印"约定）。即使 §5.2 的返回文本已经非常短，这一层保险仍然保留，确保 scrollback 只有"结构化渲染块"一份。

---

## 8. HTTP 集成

`vv/httpapis/` 通过 `vage/service/` 的 SSE 通路自动推送 `EventTodoUpdate`。**不需要改 service 层**：事件会被 JSON 编码 + `type: todo_update` 送到客户端。

约定：客户端若不识别 `todo_update` 应静默忽略（SSE 默认行为）。

---

## 9. 配置与注册

### 9.1 配置

**不新增 yaml 字段。** 原设计的 `AgentsConfig.TodoEnabled` 被删除——默认开启、和"挂载 6 个内置工具"一起打包，不存在"单独关掉 todo 但保留其他 5 工具"的用户故事（YAGNI）。

保留一个**隐式 env 开关**供排障使用：`VV_DISABLE_TODO`（默认 false）。

```go
// vv/setup/setup.go（摘录）
if os.Getenv("VV_DISABLE_TODO") != "true" {
    // register todo_write on eligible registries (see §9.2)
}
```

### 9.2 setup.Init 连接

```go
// vv/setup/setup.go（摘要）
store := todo.NewStore()

if os.Getenv("VV_DISABLE_TODO") != "true" {
    // fullReg: coder (full tools)
    // reviewReg: reviewer (read + bash)
    // readOnlyReg: researcher (read-only)
    for _, reg := range []*tool.Registry{fullReg, reviewReg, readOnlyReg} {
        if err := todo.Register(reg, store); err != nil {
            return err
        }
    }
    // explicitly NOT registered on: chat (no tools), planner, explorer (short tasks).
}
```

### 9.3 registries.FactoryOptions

**不改动。** store 通过 tool registry 的 Register 闭包已经 capture，Agent factory 不需要直接感知 store。

---

## 10. 关键权衡与备选方案

| 决策 | 选择 | 被放弃的方案 | 理由 |
|---|---|---|---|
| sessionID 如何到 tool | ctx 注入（新增 `schema.WithSessionID`） | 让 tool 自己维护最近一次 sessionID / 传 RunRequest 指针 | 前者破坏并发语义；后者污染 tool 接口 |
| event 如何到 stream | ctx 注入 Emitter 闭包 | TaskAgent 扫描 ToolResult 的侧信道字段 | 闭包侵入小、无需改 `schema.ToolResult`；类型安全 |
| 注入点 | **只在 `executeToolBatch` 顶部一次** | 在 Run/RunStream 两个路径各注入 | 单一 choke point，减少漂移 |
| store 作用域 | 进程单例 per Session | 每 Agent 一个 store | 多 Agent（coder → reviewer）在同 session 需共享列表；version 单调只有单 store 能保证 |
| `sessionID == ""` | 直接 ErrorResult | 走 "global/default" fallback | 防止跨会话串台 |
| 存储介质 | 纯内存 | SQLite（复用 P1-6） | 与 memory 正交；会话结束即丢弃；未来可通过 P2-14 检查点持久化 |
| 状态转移约束 | handler 强制"最多一个 in_progress" | 仅 prompt 引导 | Claude issue #6528 直接动因；低成本高收益 |
| `ToolDef.ReadOnly` | **true** | false | `todo_write` 不碰磁盘/工作区；与 researcher 的只读声明一致 |
| LLM 回显文本 | `ok (v3, 4 items)` | 整张列表重述 | tokens 更省，LLM 自己就能看到它刚提交的 items |
| 注册挂载位置 | `setup.Init` | `vv/tools/tools.go` | `vv/tools` 是"挂 vage 工具"薄层，不该知道 todo store 实例 |
| 配置开关 | **无 yaml；保留 `VV_DISABLE_TODO` env** | `AgentsConfig.TodoEnabled *bool` + env | 没有用户故事需要单独关 todo；env 够用 |
| 消息回显符号 | `[x] [>] [ ]` ASCII | emoji（✅🔧❌） | 零依赖、终端兼容性更广；CLI 渲染仍用 lipgloss 着色 |
| 增量 vs 全量 | 全量替换 | CRUD + id | 对齐 Claude TodoWrite 既有模式；id 由服务器分配提供未来拓展空间 |

---

## 11. 错误与降级

| 条件 | 行为 |
|---|---|
| sessionID 缺失 | `ToolResult` Error，列表不变；LLM 可继续 |
| JSON 解析失败 | Error，列表不变 |
| 多于 1 个 in_progress | Error（`todo_write: only one in_progress item allowed`），列表不变 |
| 超过 `maxItemsPerList` (=100) 条 | Error（`todo_write: too many items (max 100)`），列表不变 |
| emitter 为 nil（Run 非 stream 且无 hook manager） | 静默跳过事件发射，`ToolResult` 仍正常返回 |
| `items == nil` 或 `items == []` | 合法；列表清空；version +1（不是 reset 到 0） |
| Store 并发写同一 session | Mutex 串行化；每次都拿到一个新的 version |

**非 fatal 原则**：任何错误都返回 `ErrorResult`（`error` 返回值始终为 nil），让 Agent 在下一轮 LLM call 修复输入。参考 askuser / write 等既有 tool 的做法。

---

## 12. 测试计划（单测 / 覆盖率目标）

### 12.1 vage/tool/todo

- `store_test.go`
  - Apply 空列表（`nil` / `[]` 两路径）→ version 递增，items=nil
  - Apply 合法列表 → version=1，id 自动分配 `todo_1..N`
  - 第二次 Apply 相同 id → 保留 id；version=2
  - Apply 两个 in_progress → `ErrTooManyInProgress`，状态不变
  - Apply 非法 status → 错误
  - 空 content / 空 active_form → 错误
  - 两个 session 并发 Apply 100 次 → race-free（`go test -race`）；各自 version 单调
  - >`maxItemsPerList` items → `ErrTooManyItems`
  - Get 未知 session → 空快照（version=0）
  - **新增（来自评审 §4）**：同一 session、不同 "agent 身份" 顺序写入 → version 全局单调递增
- `todo_test.go`
  - Handler: ctx 无 sessionID → ErrorResult
  - Handler: 非法 JSON → ErrorResult
  - Handler: 成功路径 → 触发 emitter 一次，payload 匹配新快照
  - Handler: emitter 为 nil 时不 panic
  - Handler: 返回文本匹配正则 `^ok \(v\d+, \d+ items\)$`

### 12.2 vage/schema/context

- `WithSessionID("")` 不写入
- `WithEmitter(nil)` 不写入
- 读取缺失值返回零值

### 12.3 vage/agent/taskagent

- 新增表驱动 mock 工具，用 `schema.SessionIDFromContext` / `EmitterFromContext` 断言 ctx 已注入（stream 与非 stream 各一个用例；两个用例都走 `executeToolBatch`，验证"只在此处注入"的预期）。

### 12.4 vv/cli/render_test.go

- `renderTodoList` 三种状态的渲染字符串（去色后）包含对应 marker 与文本。
- 空列表：header + 0 items。
- **新增（来自评审 §9）**：`renderToolCallResult` 对 `toolName == "todo_write"` 的 `ToolResultData` 返回空串——防止未来误删双重打印抑制。

### 12.5 vv/tools/tools_test.go（现有）

- 更新断言："todo_write" 存在于 coder (full)、reviewer (review tools)、researcher (read-only) 注册出的三个 registry。
- "todo_write" 不存在于 chat / planner / explorer 对应的工具挂载点（这些 agent 要么没工具、要么非 dispatchable）。

### 12.6 集成测试 `vv/integrations/cli_tests`

- 启动 vv（stream mode），驱动 coder agent 调用 todo_write 两轮；读取事件流，断言收到两条 `EventTodoUpdate`，version 1→2。

### 12.7 集成测试 `vv/integrations/httpapis_tests`

- 走 `/v1/agents/coder/stream`，驱动一次 todo_write，SSE 解析到 `event: todo_update` 一条；payload.version=1 + items 三条。

---

## 13. 里程碑与代码量估算（评审后）

| 块 | 预估 |
|---|---|
| vage/schema/context.go + event.go 改动 | 70 LOC |
| vage/tool/todo 包（store + todo + 测试） | 370 LOC |
| vage/agent/taskagent 注入 ctx（单点） | 10 LOC + 单测 60 LOC |
| vv/cli/render.go + cli.go | 50 LOC + 单测 70 LOC |
| vv/agents coder/researcher/reviewer prompt 追加 | 15 LOC |
| vv/setup wiring（+ env gate） | 20 LOC |
| 集成测试（cli + http） | 120 LOC |
| **合计** | **~785 LOC**，其中测试占 ~500 LOC。比 raw 版 ~875 LOC 少约 90 行（来源：删除 `TodoEnabled` 配置开关 ~30 LOC、注入点合并 ~20 LOC、formatForLLM 化简 ~5 LOC、删除 tools.go 三分支改动 ~30 LOC）。 |

---

## 14. 已知未覆盖（下游需求接续）

- P2-14 会话检查点：上线后可把 Snapshot 序列化进 checkpoint；TodoWrite 本身无需改动。`Store.Clear(sessionID)` 已预留。
- P4-13 对话分支：如需支持"从任意历史点 fork 后继承 todo 列表"，Store 需要按 (sessionID, branchID) 扩展 key——届时再做。
- UI 侧"专用面板"：如后续 CLI 把 todo 提到固定区域（类 Claude Tasks `Ctrl+T`），只需重用 `renderTodoList`+新增 View 区域，无需改协议。

---

## 15. 变更摘要（相对 design-raw.md）

1. **注入点收敛**：从"Run / RunStream 两处"改为"`executeToolBatch` 顶部一次"；删除 `toolCallCtx` helper 中冗余的 sessionID/timestamp 补齐。
2. **ReadOnly 从 false 改成 true**：语义"不动工作区"，使 researcher 的 read-only profile 合约一致。
3. **LLM 回显文本**：从"ASCII 勾号列表 4+ 行"化简为 `ok (v3, 4 items)`。
4. **删除 `AgentsConfig.TodoEnabled`**：YAGNI；仅保留 `VV_DISABLE_TODO` env。
5. **注册挂载位置**：从 `vv/tools/tools.go` 三分支改为 `vv/setup/setup.go` 统一；`vv/tools` 与 `vv/registries` 零改动。
6. **prompt 引导段差异化**：coder 两行；researcher 一行（带场景门限）；reviewer 一行（带场景门限）。
7. **`items=nil` / `items=[]` 明确同语义**（清空，version +1）。
8. **`maxItemsPerList = 100` 作为命名常量**，方便后续调参。
9. **显式两条 Store 不变式**：多 Agent 同 session 下 version 单调；`sessionID == ""` 直接错误，绝不 fallback。
10. **CLI 新增一条测试**：`renderToolCallResult` 对 `todo_write` 返回空串——防止未来误删双重打印抑制。
