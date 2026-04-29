# Plan 工作区 — 技术设计

> 目标：交付能让 `plan_update`/`notes_write`/`notes_read` 正确工作 + `WorkspaceSource` 自动注入 prompt + vv 端默认拉起 + HTTP 暴露 + PRD 同步的最小可工作方案。

## 1. 总览

```
┌─────────────────────────────────────────────────────────────────┐
│ vv setup.Init                                                   │
│   ├── 已有: FileSessionStore + SessionHook(2026-04-29-002/004)  │
│   └── 新增: NewFileWorkspace(<session-root>/<project>) ──┐      │
└──────────────────────────────────────────────────────────┼──────┘
                                                          │
        ┌─────────────────────────────────────────────────┘
        │ workspace 单例(per process,session 路径在调用时拼)
        ▼
┌──────────────────────┐         ┌──────────────────────────────┐
│ vage/workspace/      │         │ vage/tool/workspace/         │
│   Workspace iface    │◀───────▶│  plan_update                 │
│   FileWorkspace impl │         │  notes_write                 │
│   ErrInvalidName     │         │  notes_read                  │
│   ReadPlan/WritePlan │         └──────────────────────────────┘
│   ReadNote/WriteNote │                       │
│   ListNotes/Delete   │                       │ 注册到 Primary 的 Tool Registry
│                      │                       ▼
└──────────────┬───────┘             Primary Assistant ReAct loop
               │
               │ 读取 plan + notes index
               ▼
┌──────────────────────────────────┐
│ vage/context/                    │
│   sources_workspace.go           │   ← 注入到 ContextBuilder 中
│     WorkspaceSource{Workspace}   │
└──────────────────────────────────┘
               ▲
               │ TaskAgent.WithExtraSources(...)
               │
        所有 dispatchable agent + Primary
```

## 2. 包结构

```
vage/
├── workspace/                          (新增)
│   ├── workspace.go        # Workspace iface, errors, name validation
│   ├── workspace_test.go
│   ├── filestore.go        # FileWorkspace impl (filesystem backend)
│   └── filestore_test.go
├── tool/
│   └── workspace/                      (新增)
│       ├── plan.go         # plan_update tool
│       ├── plan_test.go
│       ├── notes.go        # notes_write / notes_read tools
│       └── notes_test.go
├── context/
│   ├── sources_workspace.go            (新增)
│   └── sources_workspace_test.go       (新增)
├── schema/
│   └── events.go           # 新增 EventWorkspacePlanUpdated / EventWorkspaceNoteWritten
├── agent/taskagent/
│   └── options.go          # 新增 WithExtraSources(...vctx.Source) option
└── .doc/
    └── workspace.md                    (新增)

vv/
├── setup/
│   └── setup.go            # 构造 Workspace, 注册工具, 注入 source
├── apis/                   # HTTP routes
│   └── sessions.go         # 新增 /v1/sessions/{id}/workspace/plan / notes / notes/{name}
└── CLAUDE.md               # 新增 Plan Workspace 段
```

## 3. 关键 API（vage/workspace）

### 3.1 Workspace 接口

```go
// Package workspace provides a per-session, persistent plan + notes scratchpad.
// It is intentionally narrow: plan.md is a single human-readable string,
// notes are a flat directory of <name>.md files. Anything richer (artifacts,
// scratch subtasks) is out of scope for this MVP.
package workspace

import (
    "context"
    "errors"
    "time"
)

// MaxPlanBytes caps the plan.md size to prevent prompt explosion.
const MaxPlanBytes = 64 * 1024

// MaxNoteBytes caps a single note size.
const MaxNoteBytes = 32 * 1024

// MaxNoteCount caps how many notes a session may keep.
const MaxNoteCount = 200

// Errors.
var (
    ErrInvalidName    = errors.New("workspace: invalid note name")
    ErrInvalidSession = errors.New("workspace: invalid session id")
    ErrTooLarge       = errors.New("workspace: payload exceeds limit")
    ErrTooManyNotes   = errors.New("workspace: note count exceeds limit")
)

// NoteInfo is the index entry returned by ListNotes.
type NoteInfo struct {
    Name      string
    Bytes     int
    UpdatedAt time.Time
}

// Workspace is the per-session plan + notes scratchpad.
//
// All methods take SessionID as their first arg; the Workspace impl owns
// path construction, so callers cannot escape the session sandbox.
//
// All methods are safe for concurrent use across distinct sessions.
// Writes against the same session are serialized internally.
type Workspace interface {
    // ReadPlan returns plan.md content. Empty string + nil error when the
    // session has no plan yet (the file does not exist). Errors are limited
    // to genuine IO failures.
    ReadPlan(ctx context.Context, sessionID string) (string, error)

    // WritePlan replaces plan.md with content. content must be ≤ MaxPlanBytes.
    // Empty content clears the plan (deletes the file).
    WritePlan(ctx context.Context, sessionID string, content string) error

    // ReadNote returns notes/<name>.md content. Empty string + nil error when
    // the note does not exist. Returns ErrInvalidName for malformed names.
    ReadNote(ctx context.Context, sessionID, name string) (string, error)

    // WriteNote writes notes/<name>.md. content must be ≤ MaxNoteBytes;
    // total notes must remain ≤ MaxNoteCount. Empty content deletes the note.
    WriteNote(ctx context.Context, sessionID, name, content string) error

    // ListNotes returns notes ordered by UpdatedAt DESC.
    ListNotes(ctx context.Context, sessionID string) ([]NoteInfo, error)

    // Delete removes the entire workspace for a session. Idempotent.
    Delete(ctx context.Context, sessionID string) error

    // PathOf returns the on-disk root for a session (advisory; primarily
    // for logging). Returns "" when the impl is not file-backed.
    PathOf(sessionID string) string
}
```

### 3.2 名称校验

```go
// notes 名称模式（与 session ID 同一族，但更短，因为 notes 数量更多）
var noteNamePattern = regexp.MustCompile(`^[A-Za-z0-9._-]{1,64}$`)

func validateNoteName(name string) error {
    if name == "" {
        return fmt.Errorf("%w: empty", ErrInvalidName)
    }
    if name == "." || name == ".." {
        return fmt.Errorf("%w: %q is reserved", ErrInvalidName, name)
    }
    if !noteNamePattern.MatchString(name) {
        return fmt.Errorf("%w: %q does not match %s", ErrInvalidName, name, noteNamePattern.String())
    }
    return nil
}
```

### 3.3 FileWorkspace

```go
type FileWorkspace struct {
    root string                       // <session-root>/<project>/
    mu   sync.Mutex                   // process-wide; per-session granularity not worth the bookkeeping for MVP
}

func NewFileWorkspace(root string) (*FileWorkspace, error) { ... }
```

布局：

```
<root>/<sessionID>/workspace/
├── plan.md                  # WritePlan 整体覆盖
└── notes/
    ├── <name>.md            # WriteNote 整体覆盖
    └── <name>.md
```

**关键决策**：复用 `vage/session.FileSessionStore` 的根（`<session-root>/<project>/<id>/`），workspace 目录是 sessionID 目录的子目录。这样 `session.Delete` 时整个 sessionID 目录被删，workspace 自然清理（无需在 SessionStore 接口加 hook）。**但**—— `FileSessionStore.Delete` 当前只删它自己创建的文件吗？让我们 check：

读 `filestore.go` 删除路径，发现 `os.RemoveAll(sessDir)` —— 已经会递归删整个目录。✅ 不需要改。

写文件用 atomic rename（`os.WriteFile` 到 `<file>.tmp` + `os.Rename`），权限 `0o600`，目录 `0o700` —— 与 SessionStore 一致。

**FileWorkspace.root 与 SessionStore.root 共用同一目录**：
- vv setup 已经计算好 `sessionRoot := filepath.Join(cfg.Session.EffectiveDir(), SessionProjectName(cfg.Tools.BashWorkingDir))`
- workspace 用同一个 root
- 在 FileSessionStore 创建后再创建 FileWorkspace；二者并存共享根目录

## 4. Tool API（vage/tool/workspace）

### 4.1 plan_update

```json
{
  "name": "plan_update",
  "description": "Replace the entire plan.md for the current session. Pass the FULL new plan; this is a complete rewrite, not a patch...",
  "parameters": {
    "type": "object",
    "properties": {
      "content": {"type": "string", "description": "Full plan markdown. Pass empty string to clear."}
    },
    "required": ["content"]
  }
}
```

行为：从 ctx 取 `sessionID`，调 `Workspace.WritePlan(ctx, sessionID, content)`，发出 `EventWorkspacePlanUpdated`。返回 `"ok (N bytes)"` 或 error 文本。

### 4.2 notes_write / notes_read

```json
{"name": "notes_write", "parameters": {"type":"object",
  "properties": {
    "name":   {"type":"string","description":"Note name; alphanumerics + . _ - only, ≤ 64 chars"},
    "content":{"type":"string","description":"Note markdown content. Pass empty to delete."}
  },
  "required":["name","content"]}}

{"name": "notes_read", "parameters": {"type":"object",
  "properties": {
    "name": {"type":"string"}
  },
  "required":["name"]}}
```

`notes_read` 返回内容；不存在返回 `"<empty>"`。`notes_write` 发 `EventWorkspaceNoteWritten`。

### 4.3 工具的描述说明（写在 `Description` 中给 LLM 看）

```
plan_update:
  WHEN to use:
    - User gives a multi-step task that may span multiple turns/sessions.
    - You complete a step and want to record progress before continuing.
    - You revise the plan based on new info.
  HOW to use:
    - Pass the FULL plan as `content`. The previous plan is overwritten.
    - Use Markdown checkboxes (- [x] / - [ ]) for granular steps.
    - Keep it terse; this is a strategy doc, not a journal. The journal is
      events.jsonl — you do not write to it.
  DO NOT:
    - Use this for short-lived TODO items within a single ReAct loop — that's
      what `todo_write` is for. plan.md persists across sessions; todo_write
      does not.
    - Treat plan.md as a notepad for facts. Use `notes_write` for facts.

notes_write:
  WHEN: Long-lived facts you want to recall later (decisions, file paths,
        config snippets, learned conventions).
  HOW:  One file per topic. Pass an empty content to delete a note.

notes_read:
  WHEN: When you need to recall a specific note. The current plan and the
        notes index are already in your prompt; this is for reading the
        full body of a specific note.
```

## 5. ContextSource（vage/context/sources_workspace.go）

```go
type WorkspaceSource struct {
    Workspace workspace.Workspace
    // Optional: cap on how many bytes of plan.md to inject. 0 = use
    // workspace.MaxPlanBytes. Used to keep prompt small even when plan is large.
    MaxBytes int
}

// Compile-time conformance
var _ Source = (*WorkspaceSource)(nil)

func (s *WorkspaceSource) Name() string { return SourceNameWorkspace }
```

`Fetch` 行为：

1. 没 Workspace 或 SessionID 空 → `Status="skipped"`。
2. 读 `ReadPlan(ctx, sid)`、`ListNotes(ctx, sid)`。
3. 二者都失败 → `Status="error"`，但是 fail-open（vctx Builder 默认）。
4. 二者都为空 → `Status="skipped"`。
5. 否则拼出单一 system 消息：

```
## Plan Workspace

<plan.md content if any, prefixed with "### Plan" header>

### Notes
- name1.md (243 bytes, updated 2m ago)
- name2.md (1.2 KB, updated 1h ago)

(Use `notes_read` to view a note's full content.)
```

6. 报告：`OriginalCount = len(plan) + len(notes index)`，`OutputN = 1`，`Status="ok"`。

源代码 ≤ 150 行（含注释）。

## 6. TaskAgent option

新增 **一个** option，`vage/agent/taskagent/options.go`：

```go
// WithExtraSources appends Sources to the ContextBuilder used by every
// Run/RunStream call. They are inserted **after** the built-in
// SessionMemorySource and **before** RequestMessagesSource, so the resulting
// order is [system, session_memory, ...extras, request].
//
// Use this to plug in cross-cutting context like a Plan Workspace, a vector
// recall layer, or a session tree without rewriting the whole Builder.
func WithExtraSources(srcs ...vctx.Source) Option {
    return func(a *Agent) { a.extraSources = append(a.extraSources, srcs...) }
}
```

`buildInitialMessages` 修改后：

```go
func (a *Agent) buildInitialMessages(ctx context.Context, req *schema.RunRequest) (buildResult, error) {
    opts := []vctx.BuilderOption{
        vctx.WithSource(&vctx.SystemPromptSource{Template: a.systemPrompt}),
        vctx.WithSource(&vctx.SessionMemorySource{Manager: a.memoryManager}),
    }
    for _, s := range a.extraSources {
        opts = append(opts, vctx.WithSource(s))
    }
    opts = append(opts,
        vctx.WithSource(&vctx.RequestMessagesSource{}),
        vctx.WithHookManager(a.hookManager),
    )
    builder := vctx.NewDefaultBuilder(opts...)
    // ... rest unchanged
}
```

**行为兼容**：当 `a.extraSources == nil` 时，输出与今日逐字节一致；现有所有 TaskAgent 单测无回归。

## 7. vv setup wiring

`vv/setup/setup.go` 增量改动（伪代码）：

```go
// 在 buildHookManagerAndSession 之后，构造 workspace（可复用同一 root）
var ws workspace.Workspace
if cfg.Session.IsEnabled() && sessionStore != nil {
    fw, err := workspace.NewFileWorkspace(sessionRoot)
    if err != nil {
        return nil, fmt.Errorf("workspace: %w", err)
    }
    ws = fw
    slog.Info("vv: plan workspace enabled", "dir", sessionRoot)
}

// New 函数中，给 Primary Assistant 注册三个工具 + 给所有 dispatchable agent 装 source

// 1. 给 Primary 的 toolReg 注册三个工具
if ws != nil {
    if err := wsworkspace.RegisterPlan(toolReg, ws); err != nil { ... }
    if err := wsworkspace.RegisterNotes(toolReg, ws); err != nil { ... }
}

// 2. 在每个 desc.Factory 调用前,把 WorkspaceSource 注入 FactoryOptions
factoryOpts.ExtraContextSources = []vctx.Source{
    &vctx.WorkspaceSource{Workspace: ws},
}
```

`registries.FactoryOptions` 加字段：

```go
type FactoryOptions struct {
    // ... existing fields
    ExtraContextSources []vctx.Source
}
```

每个 agent factory（`vv/agents/coder.go` 等）把 `opts.ExtraContextSources` 传给 `taskagent.WithExtraSources(...)`。

**关闭路径**：当 `cfg.Session.IsEnabled()=false` 时 `ws=nil`，工具不注册，ExtraContextSources 为空切片，零开销。

`InitResult` 新增 `Workspace workspace.Workspace`。

## 8. HTTP routes

新增三个 GET + 一个隐式 DELETE（已有 `DELETE /v1/sessions/{id}` 调用 `SessionStore.Delete`，需要并发调 `Workspace.Delete`）。

```
GET    /v1/sessions/{id}/workspace/plan         → 200 text/markdown {plan.md} | 404
GET    /v1/sessions/{id}/workspace/notes        → 200 application/json [{name,bytes,updated_at}, ...]
GET    /v1/sessions/{id}/workspace/notes/{name} → 200 text/markdown | 400 invalid name | 404
```

`vv/apis/sessions.go`（或 `vv/apis/workspace.go` 新文件）：单纯把请求里的 sessionID + name 校验后转发给 `workspace.Workspace`。

`DELETE /v1/sessions/{id}` 现有 handler 末尾追加 `_ = workspace.Delete(ctx, id)`（fail-soft，session 已经删了就好）。

## 9. 事件 schema

`vage/schema/events.go` 增 2 个常量 + 2 个 data 结构：

```go
const (
    EventWorkspacePlanUpdated EventType = "workspace.plan_updated"
    EventWorkspaceNoteWritten EventType = "workspace.note_written"
)

type WorkspacePlanUpdatedData struct {
    SessionID string `json:"session_id"`
    Bytes     int    `json:"bytes"`
    Cleared   bool   `json:"cleared,omitempty"` // true when content == ""
}
func (WorkspacePlanUpdatedData) eventDataMarker() {}

type WorkspaceNoteWrittenData struct {
    SessionID string `json:"session_id"`
    Name      string `json:"name"`
    Bytes     int    `json:"bytes"`
    Cleared   bool   `json:"cleared,omitempty"`
}
func (WorkspaceNoteWrittenData) eventDataMarker() {}
```

## 10. 测试策略

### 10.1 单元测试（位于源码同目录）

- `workspace/workspace_test.go` ——
  - happy path: WritePlan + ReadPlan / WriteNote + ReadNote / ListNotes 排序
  - 边界: 超 MaxPlanBytes / MaxNoteBytes / MaxNoteCount → ErrTooLarge / ErrTooManyNotes
  - 安全: 5 个 path traversal 攻击向量（`../etc/passwd`, `foo/bar`, `..\\bar`, `\x00name`, 空名）→ ErrInvalidName
  - 删除: WritePlan("") 删 plan.md；Delete 整个 session 子树
  - 并发: 同 session 并发 WriteNote 不死锁、不 corrupt
- `tool/workspace/plan_test.go` / `notes_test.go` ——
  - 工具 args 解码、ctx 缺 sessionID、调 Workspace、emit Event
- `context/sources_workspace_test.go` ——
  - skipped / ok / error 三档；plan 与 notes 内容在输出 system 消息中正确出现
- `agent/taskagent/task_test.go` 增量 ——
  - 加一个测试: WithExtraSources 注入一个 fake source，断言它出现在 [system, session, EXTRA, request] 顺序中（行为兼容性）

### 10.2 集成测试（暂不引入；MVP 范围内）

仅靠单测 + `make build` 全绿验证。集成测试留作后续。

## 11. 验证清单

- [ ] `cd vage && make build`（含 lint + format + test）全绿
- [ ] `cd vv && make build` 全绿
- [ ] 设计文档 §4.4 / §8 标记 "Plan 工作区 ✅ 已落地"
- [ ] 新增 PRD：模型 `Plan Workspace`，API page workspace
- [ ] `vage/.doc/workspace.md` 创建；`vage/.doc/context.md` 表格新增 WorkspaceSource 行
- [ ] `vv/CLAUDE.md` 增 "Plan Workspace" 段，明确 plan_update vs todo_write 区别
- [ ] 各文件 ≤ 800 行（CLAUDE.md 规则）

## 12. 边界与不实现

- **scratch/ artifacts/**：原设计提到，本期不交付。理由：当前 dispatcher 不产出 artifacts；scratch 没有具体使用场景；接口先窄、后扩。
- **跨进程并发**：进程内 mutex 串行写，跨进程未定义（与 FileSessionStore 一致）。
- **notes ↔ memory.Store 同步**：原设计提到，本期不实现。MVP 用户先体会 plan.md 价值；后续若有同步需求加 `WorkspaceMemoryAdapter`。
- **TaskAgent `WithContextBuilder`（完全替换 builder）**：本期只加 `WithExtraSources`（增量追加）。完全替换的 option 留作后续，避免本期范围扩张。
- **plan.md 的 schema/lint**：不约束格式。LLM 自由用 Markdown；做 schema 约束等于跟 LLM 内卷，得不偿失。
- **CLI 暴露**：本期不改 CLI flags。`--session list` 输出已经够用；查 plan 走 HTTP。

## 13. 实现顺序

1. `vage/workspace/`（含完整单测）
2. `vage/schema/events.go` 加事件常量
3. `vage/tool/workspace/`（含单测，依赖 1+2）
4. `vage/context/sources_workspace.go`（含单测，依赖 1）
5. `vage/agent/taskagent/options.go` 加 `WithExtraSources` + 修改 `buildInitialMessages`（含单测）
6. `vv/registries/` `FactoryOptions.ExtraContextSources` 字段 + 各 agent factory 透传
7. `vv/setup/setup.go` 拉起 workspace + 注册工具 + 注入 source
8. `vv/apis/` 新 routes
9. 文档（vage/.doc/workspace.md, .doc/context.md, vv/CLAUDE.md）
10. PRD
11. 设计文档 §4.4 §8 标记落地
