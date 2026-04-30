# 设计：A6 LLM 工具包 + B1–B8 vv 端 wiring

## 0. 目标速记

把 SessionTree（vage/session/tree/）的能力暴露到 LLM（A6）与 vv 应用层（B1-B8），保持现有 `Workspace + Plan` 同款风格：
- 工具按 ctx 取 sessionID，无 path 参数
- HTTP 路由仅在子系统启用时挂载
- Primary 写、sub-agent 只读

## 1. A6 — `vage/tool/sessiontree/` 包

### 1.1 文件布局

```
vage/tool/sessiontree/
├── tools.go         # 共用 helpers（参数解析、错误转 schema.ToolResult、事件 emit）
├── add.go           # tree_add
├── update.go        # tree_update
├── cursor.go        # tree_cursor
├── promote.go       # tree_promote
├── zoomin.go        # tree_zoom_in
├── register.go      # Register(reg, store) — 一次性注册全部 5 个工具
└── *_test.go        # 每文件配套单测
```

每个工具 ≤200 行；`tools.go` 集中放：

```go
type sessionIDProvider interface {
    SessionIDFromContext(ctx context.Context) string  // schema.SessionIDFromContext 直接复用
}

func parseArgs[T any](args string) (T, *schema.ToolResult)  // 不可用 generics return-pointer，简化为各工具内联

func emitTreeEvent(ctx, sessionID, op string, n *tree.TreeNode, count int)  // 复用 schema.EventSessionTreeUpdated
```

### 1.2 工具 schema 设计

#### tree_add

参数:
```json
{
  "parent_id": "tn-...",        // optional: 空时若树不存在则创建 root；否则用 cursor
  "type": "subtask|fact|...",   // optional: 默认 "subtask"
  "title": "string (≤200B)",    // required
  "summary": "string (≤2KiB)",  // optional
  "status": "pending|active|done|...",  // optional, 默认 "pending"
  "pinned": false               // optional
}
```

行为：
- `parent_id == ""` 且无 tree → 调用 `CreateTree(sessionID, root)` 创建 root
- `parent_id == ""` 且有 tree → 视为追加到 root
- 否则 → `AddNode(sessionID, parent_id, n)`
- 返回 `{"node_id": "...", "depth": N}`

#### tree_update

参数:
```json
{
  "node_id": "tn-...",        // required
  "title": "...",              // required (复刻 store 校验：title 永不为空)
  "summary": "...",            // optional
  "status": "...",             // optional
  "pinned": false,             // optional
  "metadata": {}               // optional
}
```
行为：直接调用 `UpdateNode`。
返回：`{"node_id": "...", "updated_at": "..."}`

#### tree_cursor

参数:
```json
{ "node_id": "tn-... or empty to clear" }  // required (允许空字符串)
```
行为：调用 `SetCursor`。
返回：`{"cursor": "..."}`

#### tree_promote

参数:
```json
{ "node_id": "tn-..." }  // required
```
行为：调用 `PromoteNode`。
返回：
- 成功：`{"node_id": "...", "folded_count": N, "summary_bytes": M}`（folded_count 由调用前后 GetTree 比对推断；或 store 返回值已经携带）
- 实际：`PromoteNode` 返回的是更新后的 parent，可以 `len(parent.Summary) - originalSummary` 给出 newSummaryBytes，folded_count 从 children 状态推断
- 简化：返回 `{"summary": "<new summary>"}` 即可，event 已经携带详细数据

#### tree_zoom_in

参数:
```json
{ "node_id": "tn-..." }  // optional: 空时返回当前 cursor 子树
```
行为：调用 `GetTreeView(opts={IncludePromoted: true})`，提取以 nodeID 为根的子树（含 promoted），按 SessionTreeSource defaultTreeRender 风格渲染并返回文本。
- 这是 LLM 主动 "zoom in" 一个折叠节点的入口。
返回：tool result 的 text content 即子树渲染。

### 1.3 注册函数

```go
// Register registers all five tools on reg, bound to store.
// Returns an error on duplicate name; nil store rejected.
func Register(reg *tool.Registry, store tree.SessionTreeStore) error
```

### 1.4 事件发射

复用 `schema.EmitterFromContext(ctx)`，工具不直接调用 store 之外的事件——store 自身已经在所有写路径上 dispatch `EventSessionTreeUpdated`，工具只需把成功或错误结果转成 `schema.ToolResult`。

`tree_promote` 也不需要工具层补事件——store 内部已 dispatch `Started/Completed/Failed`。

### 1.5 错误处理约定

复用 store 的 sentinel：`tree.ErrInvalidArgument` / `tree.ErrNotFound` / `tree.ErrAlreadyExists` / 等等。映射到 `schema.ErrorResult` 文本，IsError=true：

```go
if errors.Is(err, tree.ErrInvalidArgument) { return schema.ErrorResult(...), nil }  // 不返 Go error
return schema.ErrorResult(toolName + ": " + err.Error()), nil
```

只有 ctx canceled 或 panic recover 例外——保持与 plan_update / notes_write 同款 fail-soft 行为。

## 2. B1 — vv 配置（`vv/configs/config.go`）

### 2.1 新增 SessionTreeConfig

```go
type SessionTreeConfig struct {
    Enabled   *bool                       `yaml:"enabled,omitempty"`   // default false（与设计文档一致）
    Promotion SessionTreePromotionConfig  `yaml:"promotion,omitempty"`
}

type SessionTreePromotionConfig struct {
    Enabled               *bool   `yaml:"enabled,omitempty"`                 // default false
    Promoter              string  `yaml:"promoter,omitempty"`                // "llm" | "compressor" | "noop"; default "compressor"
    Model                 string  `yaml:"model,omitempty"`                   // 空 = 复用 LLM.Model
    ChildrenThreshold     int     `yaml:"children_threshold,omitempty"`      // default 8
    SubtreeBytesThreshold int     `yaml:"subtree_bytes_threshold,omitempty"` // default 8192
    AllChildrenDone       *bool   `yaml:"all_children_done,omitempty"`       // default true (开启 AllChildrenDoneDecider)
}

func (s SessionTreeConfig) IsEnabled() bool { return s.Enabled != nil && *s.Enabled }
func (s SessionTreePromotionConfig) IsEnabled() bool { return s.Enabled != nil && *s.Enabled }
func (s SessionTreePromotionConfig) PromoterKind() string {
    if s.Promoter == "" { return "compressor" }
    return s.Promoter
}
```

放在 `Config.SessionTree`：

```go
SessionTree SessionTreeConfig `yaml:"session_tree,omitempty"`
```

环境变量覆盖（`applyEnvOverrides`）：
- `VV_TREE_ENABLED` → bool → `cfg.SessionTree.Enabled`
- `VV_TREE_PROMOTION_ENABLED` → bool → `cfg.SessionTree.Promotion.Enabled`
- `VV_TREE_PROMOTER` → string → `cfg.SessionTree.Promotion.Promoter`

**默认行为**：默认 `Enabled=nil → false`。与设计文档 §4.8.5 "默认关闭，仅长任务场景显式启用" 对齐。

## 3. B2 — vv setup 初始化

### 3.1 setup.Result 字段扩展

```go
type Result struct {
    ...
    TreeStore   tree.SessionTreeStore  // nil when SessionTree disabled
}
```

### 3.2 InitResult 扩展

```go
type InitResult struct {
    ...
    TreeStore   tree.SessionTreeStore  // nil when SessionTree disabled
}
```

### 3.3 buildHookManagerAndSession 扩展

把 tree store 构造也下放到这里（已经在管 sessionStore + planWorkspace + hookManager），统一返回多一个返回值：

```go
func buildHookManagerAndSession(cfg *configs.Config, llm aimodel.ChatCompleter) (
    *hook.Manager, session.SessionStore, workspace.Workspace, tree.SessionTreeStore,
    func(context.Context), error,
)
```

`llm` 参数仅在 SessionTree.Promotion.Promoter == "llm" 时使用。

逻辑（仅在 cfg.Session.Enabled && cfg.SessionTree.Enabled 时构造）：

```go
var treeStore tree.SessionTreeStore
if sessionEnabled && cfg.SessionTree.IsEnabled() {
    fileOpts := []tree.FileOption{
        tree.WithFileHookManager(mgr),
    }
    if cfg.SessionTree.Promotion.IsEnabled() {
        promoter, perr := buildTreePromoter(cfg, llm, ...)
        if perr != nil { return ..., perr }
        fileOpts = append(fileOpts, tree.WithFilePromoter(promoter))
        decider := buildTreeDecider(cfg)
        fileOpts = append(fileOpts, tree.WithFilePromotionDecider(decider))
    }
    fts, err := tree.NewFileTreeStore(sessionRoot, fileOpts...)
    if err != nil { ... }
    treeStore = fts
}
```

`buildTreePromoter`：

```go
func buildTreePromoter(cfg *configs.Config, llm aimodel.ChatCompleter, mem *memory.Manager) (tree.Promoter, error) {
    switch cfg.SessionTree.Promotion.PromoterKind() {
    case "noop":      return tree.NoopPromoter{}, nil
    case "llm":
        if llm == nil { return nil, errors.New("llm promoter requires LLM client") }
        return &tree.LLMPromoter{
            Client: llm,
            Model:  effectiveTreePromoterModel(cfg),
        }, nil
    case "compressor":
        // 复用 memory.NewSlidingWindowCompressor 即可——同款简单摘要
        compressor := memory.NewSlidingWindowCompressor(cfg.Memory.SessionWindow)
        return &tree.CompressorPromoter{Compressor: compressor}, nil
    default:
        return nil, fmt.Errorf("unknown promoter kind %q", cfg.SessionTree.Promotion.Promoter)
    }
}

func buildTreeDecider(cfg *configs.Config) tree.PromotionDecider {
    children := tree.ChildrenCountDecider{Min: cfg.SessionTree.Promotion.ChildrenThreshold}
    bytes := tree.SubtreeBytesDecider{Min: cfg.SessionTree.Promotion.SubtreeBytesThreshold}
    if cfg.SessionTree.Promotion.AllChildrenDone == nil || *cfg.SessionTree.Promotion.AllChildrenDone {
        return tree.AnyOf(children, bytes, tree.AllChildrenDoneDecider{})
    }
    return tree.AnyOf(children, bytes)
}
```

### 3.4 setup.Options 扩展

```go
type Options struct {
    ...
    TreeStore tree.SessionTreeStore  // optional
}
```

### 3.5 buildExtraContextSources 扩展

```go
func buildExtraContextSources(opts *Options) []vctx.Source {
    var srcs []vctx.Source
    if opts != nil && opts.Workspace != nil {
        srcs = append(srcs, &vctx.WorkspaceSource{Workspace: opts.Workspace})
    }
    if opts != nil && opts.TreeStore != nil {
        srcs = append(srcs, &vctx.SessionTreeSource{Store: opts.TreeStore})
    }
    return srcs
}
```

### 3.6 buildPrimaryAssistant 扩展

注册 5 个 tree 工具（仅 Primary）：

```go
if opts != nil && opts.TreeStore != nil {
    if err := sessiontreetool.Register(toolReg, opts.TreeStore); err != nil {
        return nil, fmt.Errorf("primary: register tree tools: %w", err)
    }
}
```

sub-agent (`buildSubAgents` 等价的 for 循环已存在) 不动——它们通过 ExtraContextSources 自动获得 SessionTreeSource。

## 4. B3 — vv CLI

### 4.1 文件 `vv/cli/tree.go`

```go
// PrintTree writes the SessionTreeSource-formatted tree for sessionID to w.
// includePromoted=true mirrors the HTTP ?include_promoted=1 flag.
// Returns ErrTreeMissing when the tree does not exist.
func PrintTree(ctx context.Context, store tree.SessionTreeStore, sessionID string, includePromoted bool, w io.Writer) error
```

实现：调用 `vctx.SessionTreeSource{Store: store, IncludePromoted: includePromoted}.Fetch(...)` 然后 `Print` 第一条 Message 的 text。

### 4.2 main.go 的 `--tree` flag 处理

新增 `treeFlag := flag.String("tree", "", "tree subcommand: 'show <session-id>' to print session tree, 'show <session-id> --all' to include promoted nodes")`。

flag 的解析比较麻烦——为了不打破现有 flag 结构，**改用 dual-flag**：
- `--tree show` (bool-trigger)
- `--tree-id <session-id>`
- `--tree-all` (bool, includePromoted)

实际更简单的方式：复用现有 `--session list` 风格，把 `--tree show <id>` 拆成单独的子命令解析路径。

**采纳实现**：新增 string flag `--tree`，值 = `<session-id>`；若额外的 bool flag `--tree-all`，则带 promoted。语义：
- `vv --tree <id>` → 打印当前 tree（默认隐藏 promoted）
- `vv --tree <id> --tree-all` → 打印含 promoted 的 tree

如果 `--tree=list`：列出有 tree 的 session id（按需，但暂可与 `--session list` 复用）。

最简实现：仅 `--tree <id>`，不支持 `list`（已经有 `--session list` 涵盖大部分需求）；`--tree-all` 控制 include_promoted。

### 4.3 mode 限制

仅 CLI 模式可用——HTTP 模式提示 "use /v1/sessions/{id}/tree"。

## 5. B4 — vv HTTP 路由

### 5.1 文件 `vv/httpapis/tree.go`

```go
// JSON 响应类型
type treeResponse struct {
    SessionID string                  `json:"session_id"`
    RootID    string                  `json:"root_id,omitempty"`
    Cursor    string                  `json:"cursor,omitempty"`
    Nodes     []*treeNodeResponse     `json:"nodes"`
    UpdatedAt string                  `json:"updated_at"`
}

type treeNodeResponse struct {
    ID         string         `json:"id"`
    Type       string         `json:"type"`
    Status     string         `json:"status"`
    Title      string         `json:"title"`
    Summary    string         `json:"summary,omitempty"`
    Parent     string         `json:"parent,omitempty"`
    Children   []string       `json:"children,omitempty"`
    Pinned     bool           `json:"pinned,omitempty"`
    Promoted   bool           `json:"promoted,omitempty"`
    PromotedAt string         `json:"promoted_at,omitempty"`
    Depth      int            `json:"depth"`
    CreatedAt  string         `json:"created_at"`
    UpdatedAt  string         `json:"updated_at"`
    Metadata   map[string]any `json:"metadata,omitempty"`
}

// Body for POST root creation
type createTreeRequest struct {
    Type    string         `json:"type,omitempty"`     // 默认 "goal"
    Title   string         `json:"title"`              // required
    Summary string         `json:"summary,omitempty"`
    Status  string         `json:"status,omitempty"`
    Metadata map[string]any `json:"metadata,omitempty"`
    Pinned  bool           `json:"pinned,omitempty"`
}

type addNodeRequest struct {
    ParentID string `json:"parent_id"`  // required
    Node     createTreeRequest `json:"node"`
}

type updateNodeRequest struct {
    Title      *string         `json:"title,omitempty"`
    Summary    *string         `json:"summary,omitempty"`
    Status     *string         `json:"status,omitempty"`
    Pinned     *bool           `json:"pinned,omitempty"`
    Metadata   *map[string]any `json:"metadata,omitempty"`
}

type setCursorRequest struct {
    NodeID string `json:"node_id"`  // empty 允许清除
}
```

### 5.2 路由与 handler

```go
mux.HandleFunc("GET /v1/sessions/{id}/tree", handleGetTree(store))
mux.HandleFunc("POST /v1/sessions/{id}/tree", handleCreateTree(store))
mux.HandleFunc("DELETE /v1/sessions/{id}/tree", handleDeleteTree(store))
mux.HandleFunc("POST /v1/sessions/{id}/tree/nodes", handleAddNode(store))
mux.HandleFunc("PATCH /v1/sessions/{id}/tree/nodes/{nid}", handleUpdateNode(store))
mux.HandleFunc("DELETE /v1/sessions/{id}/tree/nodes/{nid}", handleDeleteNode(store))
mux.HandleFunc("POST /v1/sessions/{id}/tree/cursor", handleSetCursor(store))
mux.HandleFunc("POST /v1/sessions/{id}/tree/promote/{nid}", handlePromote(store))
```

### 5.3 错误映射

```go
func writeTreeErr(w http.ResponseWriter, err error) bool {
    case errors.Is(err, tree.ErrInvalidArgument): 400
    case errors.Is(err, tree.ErrNotFound): 404
    case errors.Is(err, tree.ErrTreeMissing): 404
    case errors.Is(err, tree.ErrAlreadyExists): 409
    case errors.Is(err, tree.ErrTreeFull): 422
    case errors.Is(err, tree.ErrHasChildren): 409
    case errors.Is(err, tree.ErrImmutableField): 400
    default: 500
}
```

### 5.4 GET ?include_promoted=1

调用 `GetTreeView(ctx, id, ViewOptions{IncludePromoted: parseBool(q.Get("include_promoted"))})`。

## 5b. B6 — Dispatcher 自动写树

### 5b.1 范围

当 `dispatcher.write_tree=true` 且 `session_tree.enabled=true` 时：当 `Dispatcher.RunPlan` 被调用（即 plan_task 工具触发 DAG 执行），自动把 plan 的每一个 step 写为 SessionTree 子节点。

### 5b.2 配置

新增到 `OrchestrateConfig`：

```go
type OrchestrateConfig struct {
    ...
    WriteTree *bool `yaml:"write_tree,omitempty"` // default false
}
```

环境变量 `VV_DISPATCHER_WRITE_TREE`。

### 5b.3 实现

`Dispatcher` 增加可选 `treeStore tree.SessionTreeStore` 字段 + `WithTreeStore(store)` Option + `WithWriteTreeEnabled(bool)` Option。

`RunPlan` 在 plan 执行**前**写入 tree（仅当 store 非 nil 且 writeTree=true 且 ctx 有 sessionID）：

```go
func (d *Dispatcher) RunPlan(ctx, plan, req) (*RunResponse, error) {
    if d.shouldWriteTree(ctx, req) {
        d.writePlanToTree(ctx, plan, req)  // best-effort: 错误 slog.Warn 不阻断
    }
    return d.runPlan(ctx, req, plan, nil, "")
}

func (d *Dispatcher) writePlanToTree(ctx context.Context, plan *Plan, req *schema.RunRequest) {
    sessionID := req.SessionID  // session id 来源
    if sessionID == "" { return }

    tr, err := d.treeStore.GetTree(ctx, sessionID)
    if errors.Is(err, tree.ErrTreeMissing) {
        // 创建 root
        tr2, _ := d.treeStore.CreateTree(ctx, sessionID, tree.TreeNode{
            Type:    tree.NodeGoal,
            Status:  tree.StatusActive,
            Title:   truncateTitle(plan.Goal),
            Summary: plan.Goal,
            Metadata: map[string]any{"source": "dispatcher"},
        })
        if tr2 == nil { return }
        // 重新读
        tr, _ = d.treeStore.GetTree(ctx, sessionID)
    }
    if err != nil && !errors.Is(err, tree.ErrTreeMissing) {
        slog.Warn(...)
        return
    }

    parentID := tr.RootID
    for _, step := range plan.Steps {
        title := truncateTitle(step.Goal)
        if title == "" { title = step.AgentID }
        _, addErr := d.treeStore.AddNode(ctx, sessionID, parentID, tree.TreeNode{
            Type:    tree.NodeSubtask,
            Status:  tree.StatusPending,
            Title:   title,
            Summary: step.Goal,
            Metadata: map[string]any{
                "source":   "dispatcher",
                "agent":    step.AgentID,
                "step_id":  step.ID,
            },
        })
        if addErr != nil { slog.Warn("dispatcher: tree write step", ...); }
    }
}
```

**简化**：仅"写入"，不在 step 完成后翻状态——状态翻转留给 LLM/手动。这是首次集成，先看产品反馈。

### 5b.4 setup wiring

在 `setup.New` 里把 treeStore + writeTree flag 通过 `dispatches.WithTreeStore` / `dispatches.WithWriteTreeEnabled` 注入：

```go
if treeStore != nil && cfg.Orchestrate.WriteTree != nil && *cfg.Orchestrate.WriteTree {
    dispatcherOpts = append(dispatcherOpts,
        dispatches.WithTreeStore(treeStore),
        dispatches.WithWriteTreeEnabled(true),
    )
}
```

### 5b.5 边界

- `req.SessionID == ""` → 跳过（无写入目标）
- `treeStore == nil` 或 flag 关 → 跳过
- 已有 tree 时不重建 root，直接挂到 root 下（有可能多次 plan 出现"plan-1, plan-2 …"——可以用 metadata.batch 区分）
- 失败 → slog.Warn，不返错（不阻塞 plan 执行）

## 6. B5 — agent 注册（已经覆盖在 B2）

- Primary：注册 5 个 tree 工具（写）
- 所有 dispatchable agent + Primary：通过 `buildExtraContextSources` 自动获得 `SessionTreeSource`

## 7. B7 — Primary system prompt 微调

在 `vv/agents/primary.go` 的 `PrimarySystemPrompt` 末尾追加一段：

```
## Session Tree (when enabled)
- 任务跨多步、涉及子任务分解时，使用 `tree_add` / `tree_update` 维护 SessionTree。
- 当某父节点的子节点数超过 ~8 或全部完成时，调用 `tree_promote` 将子树聚合为父节点摘要。
- 用 `tree_cursor` 标记当前焦点；`tree_zoom_in <node>` 查看折叠子树详情。
- 不强制开启——SessionTree 工具不存在时直接忽略本节。
```

不写硬规则，让 LLM 按情景判断。

## 8. B8 — 文档与 PRD

### 8.1 vv/CLAUDE.md

新增 "Session Tree Subsystem" 段，结构与 "Session Subsystem" 平行：
- 配置（key、env、默认）
- Storage layout
- Wiring summary
- LLM tools (Primary only) + 描述
- HTTP endpoints 一览
- Disabled path

### 8.2 doc/prd 新增 / 更新

- `doc/prd/applications/api/pages/core/sessions/tree.md` —— HTTP 路由清单
- `doc/prd/models/core/session/model-session-tree.md` —— SessionTree 模型概述
- `doc/prd/models/core/session/model-tree-node.md` —— TreeNode 模型概述

### 8.3 doc/design/session-context-solution.md

- §4.8.6 渐进路线表第 2 行（promotion + 折叠）保持 ✅；第 4 行（LLM 工具）打 ✅
- §8 差距汇总 "Session Tree" 行的注释更新为 "MVP + promotion + LLM 工具 + vv wiring"
- 末尾"促 + 折 + vv wiring"小节里 A6 与 B1-B8 全部打 ✅
- 更新差距 → 已落地汇总语，剩余 "双索引 / 跨 session 森林 / dispatcher 自动写树" 仍 ❌

## 9. 测试策略（B9 子集）

| 层 | 文件 | 内容 |
|---|---|---|
| unit | `vage/tool/sessiontree/*_test.go` | 每工具 happy/error path；MapTreeStore 驱动 |
| unit | `vv/configs/session_tree_test.go` | env 覆盖、默认值、IsEnabled 语义 |
| unit | `vv/setup/setup_test.go` 增加 case | tree.enabled=false 时 TreeStore=nil；enabled=true 时构造 FileTreeStore |
| integration | `vv/integrations/sessiontree_tests/http_tests/` | HTTP happy paths + 错误码 |
| 跳过 | LLM 驱动 e2e | 受限于 API key，留待后续 |

## 10. 风险与缓解

1. **promotion 默认 LLM 高成本** —— 默认 `promoter: compressor`（无 LLM 调用），用户显式选 `llm`。
2. **工具与 HTTP 双写源风险** —— LLM 工具与 HTTP 都写同一个 store，但内部锁/事件已经 idempotent；HTTP 路由"初期只 GET + DELETE"的设计建议仅作为产品观察阶段，本期完整提供 CRUD（与设计文档 B4 一致），便于运维诊断。
3. **session disabled 时 SessionTree 强依赖** —— Hard fail：`session_tree.enabled=true && session.enabled=false` 在 Init 期返错并退出。
4. **测试 sessionID 跨 vage/session.IDPattern 与 vage/session/tree.sessionIDPattern** —— 已对齐，无需额外。

## 11. 落地顺序

1. A6 工具包（`vage/tool/sessiontree/`）+ 单测
2. B1 配置 + 测试
3. B2 setup 改造 + 测试
4. B4 HTTP 路由 + 单测
5. B3 CLI `--tree`
6. B5/B7 Primary prompt + tool 注册 (B2 内已覆盖)
7. B8 文档
