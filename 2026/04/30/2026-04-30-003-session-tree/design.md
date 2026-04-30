# Session Tree —— 技术设计

> 实现 `doc/design/session-context-solution.md` §4.8 P10 的 MVP(渐进路线第 1 步:**手工节点 + ContextSource 渲染**)。
> 参考已落地的 §4.4 Workspace + §4.9 Vector + §4.2 vctx 的命名/分包/接口风格。

---

## 1. 总体方案

### 1.1 包布局

```
vage/session/tree/
├── tree.go                       # 节点类型常量、TreeNode、SessionTree、错误、ID 校验
├── tree_test.go
├── store.go                      # SessionTreeStore 接口
├── mapstore.go                   # MapTreeStore (in-memory + sync.RWMutex)
├── mapstore_test.go
├── filestore.go                  # FileTreeStore (filesystem,单文件 tree.json + atomic rename)
├── filestore_test.go
└── store_conformance_test.go     # MapStore / FileStore 共享黑盒测试

vage/context/
├── sources_tree.go               # vctx.SessionTreeSource (实现 vctx.Source)
└── sources_tree_test.go

vage/schema/event.go               # 新增 EventSessionTreeUpdated 常量 + SessionTreeUpdatedData 类型
```

**包名选择 `tree`**(在 `vage/session/tree/`):

- 避免与 `vage/session` 顶层污染 — `tree` 是 session 体系的子能力,放子包符合 `vage/session/...` 子集语义;
- 与 §4.8.1 草案 `vage/session/tree/` 一致;
- 类型是 `tree.SessionTree` / `tree.TreeNode`,调用方读起来自然。

> **不**放在 `vage/session/` 内是为了避免污染:Session 现有面是 meta + events + state;Tree 是更高级的面。共用根目录 + 共用 sessionID 校验,但接口分离。

### 1.2 与现有模块的关系

| 现有模块 | 关系 |
|---|---|
| `vage/session.SessionStore` | 共用 sessionID,**不**通过 store 接口调用;FileTreeStore 在同一根目录下另开 `tree/` 子目录 |
| `vage/workspace.Workspace` | 同级:都是"per-session 持久化结构化数据";互不依赖 |
| `vage/context.Builder` | TreeSource 通过 `vctx.Source` 接口接入,无侵入 |
| `vage/schema` | 新增事件常量 + payload type;不改现有 |
| `vage/hook` | TreeStore 写入路径派发事件(可选);MapStore / FileStore 都支持 `WithHookManager` option |
| `vage/memory` | 不依赖 |
| `vage/vector` | 不依赖 |

### 1.3 模块依赖方向(无环)

```
schema  ←  hook
   ↑       ↑
   │       │
session/tree (本包)  ←  context (新增 sources_tree.go)
```

---

## 2. 类型定义

### 2.1 NodeType / NodeStatus

```go
// vage/session/tree/tree.go

package tree

type NodeType string

const (
    NodeGoal        NodeType = "goal"          // root,顶层目标
    NodeSubtask     NodeType = "subtask"       // 可分解的任务
    NodeFact        NodeType = "fact"          // 已确认的事实/结论
    NodeObservation NodeType = "observation"   // 工具调用产物的浓缩
    NodeArtifactRef NodeType = "artifact_ref"  // 指向 artifacts/ / notes/ 的引用
)

func (t NodeType) Valid() bool { /* enum check */ }

type NodeStatus string

const (
    StatusPending    NodeStatus = "pending"
    StatusActive     NodeStatus = "active"
    StatusDone       NodeStatus = "done"
    StatusBlocked    NodeStatus = "blocked"
    StatusSuperseded NodeStatus = "superseded"
)

func (s NodeStatus) Valid() bool { /* enum check */ }
```

### 2.2 TreeNode

```go
type TreeNode struct {
    ID     string     `json:"id"`
    Type   NodeType   `json:"type"`
    Status NodeStatus `json:"status"`

    // 三段式负载
    Title   string `json:"title"`              // <= TitleMaxBytes,永远进 prompt
    Summary string `json:"summary,omitempty"`  // <= SummaryMaxBytes,按 budget 决定是否进 prompt

    // 引用类字段——本期保留位,Source 不渲染、Store 不解释
    ContentRef  string   `json:"content_ref,omitempty"`
    EmbeddingID string   `json:"embedding_id,omitempty"`
    Evidence    []string `json:"evidence,omitempty"`
    Supersedes  []string `json:"supersedes,omitempty"`
    Pinned      bool     `json:"pinned,omitempty"`

    // 树结构
    Parent   string   `json:"parent,omitempty"`   // root 的 Parent == ""
    Children []string `json:"children,omitempty"` // 顺序保留追加序

    // 派生字段(Store 计算并维护)
    Depth     int       `json:"depth"`        // root.Depth == 0
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`

    Metadata map[string]any `json:"metadata,omitempty"`
}
```

**与 §4.8.1 草案的差异**:

| 草案 | 本期 | 理由 |
|---|---|---|
| `TokenCost int` | 移除 | 由 Source 渲染时按 estimator 计算,Store 不维护静态字段(否则 estimator 切换会让字段失效) |
| `Author / Confidence` 等 | 留给 `Metadata map[string]any` | metadata bag 足够,不为零碎字段炒接口 |
| `pinned bool` 单独字段 | 保留 | promotion 阶段需要一等公民判断,提前预留;本期 Source 不读但 Store 接受写入 |

### 2.3 SessionTree

```go
type SessionTree struct {
    SessionID string               `json:"session_id"`
    RootID    string               `json:"root_id"`             // 创建后即不可变;空树时为 ""
    Cursor    string               `json:"cursor,omitempty"`    // 当前 active 节点;允许 == RootID
    Nodes     map[string]*TreeNode `json:"nodes"`                // 所有节点,key == ID
    UpdatedAt time.Time            `json:"updated_at"`
}
```

### 2.4 容量与校验常量

```go
const (
    TitleMaxBytes   = 200          // 中文约 65-80 字
    SummaryMaxBytes = 2 * 1024     // 2 KiB
    MaxNodes        = 1024         // 单棵树节点上限
    NodeIDPrefix    = "tn-"
)

var nodeIDPattern = regexp.MustCompile(`^tn-[A-Za-z0-9._-]{1,128}$`)
```

### 2.5 错误

```go
var (
    ErrInvalidArgument = errors.New("tree: invalid argument")
    ErrNotFound        = errors.New("tree: node not found")
    ErrAlreadyExists   = errors.New("tree: tree already exists")
    ErrTreeFull        = errors.New("tree: node count exceeds MaxNodes")
    ErrTreeMissing     = errors.New("tree: tree does not exist")
    ErrHasChildren     = errors.New("tree: node has children, cannot delete")
    ErrCycle           = errors.New("tree: operation would create a cycle")  // 预留 reparent 用
)
```

---

## 3. SessionTreeStore 接口

```go
// store.go

type SessionTreeStore interface {
    // CreateTree initializes an empty tree for sessionID with a root node of
    // the given type/title/summary. Returns ErrAlreadyExists if a tree
    // already exists for that session, ErrInvalidArgument on malformed
    // input.
    CreateTree(ctx context.Context, sessionID string, root TreeNode) (*TreeNode, error)

    // GetTree returns the full tree for sessionID. Returns ErrTreeMissing
    // when no tree exists. The returned SessionTree is a deep-enough copy:
    // callers may mutate it without affecting the store.
    GetTree(ctx context.Context, sessionID string) (*SessionTree, error)

    // AddNode appends a new child under parentID with the values from n
    // (excluding ID/Parent/Children/Depth/CreatedAt/UpdatedAt which are
    // assigned by the store). Returns the materialised node with its
    // generated ID.
    AddNode(ctx context.Context, sessionID, parentID string, n TreeNode) (*TreeNode, error)

    // UpdateNode replaces a subset of mutable fields on the node identified
    // by n.ID. Mutable fields: Title, Summary, Status, ContentRef,
    // EmbeddingID, Evidence, Supersedes, Pinned, Metadata. Type and Parent
    // are immutable in this MVP — re-shape requires Delete + Add.
    UpdateNode(ctx context.Context, sessionID string, n TreeNode) (*TreeNode, error)

    // DeleteNode removes the leaf node with the given ID. Returns
    // ErrHasChildren if it has children; ErrNotFound if absent.
    // Deleting the root is rejected (use DeleteTree instead).
    DeleteNode(ctx context.Context, sessionID, nodeID string) error

    // SetCursor moves the tree's cursor to nodeID. nodeID must exist; "" is
    // accepted to clear the cursor.
    SetCursor(ctx context.Context, sessionID, nodeID string) error

    // DeleteTree removes the entire tree for sessionID. Idempotent.
    DeleteTree(ctx context.Context, sessionID string) error
}
```

**接口设计要点**:

1. **没有"开放节点 mutation"**:返回值是拷贝;调用方修改后回写靠 `UpdateNode`。否则跨进程 / 并发场景容易出 race。
2. **不暴露 Reparent / Move**:本期不需要;接口未来增量加 `MoveNode(sid, nodeID, newParent)` 即可。
3. **CreateTree 的 root 参数采用 `TreeNode`**:复用 Title/Summary/Type/Metadata 字段,避免再造一对 `RootInput`。Type 默认接受任何值但通常应为 `NodeGoal`。

### 3.1 存储后端

#### 3.1.1 MapTreeStore

```go
type MapTreeStore struct {
    mu    sync.RWMutex
    trees map[string]*SessionTree

    hooks *hook.Manager       // optional,nil-safe
    nowFn func() time.Time    // 测试可注入
}

func NewMapTreeStore(opts ...MapOption) *MapTreeStore { ... }
func WithMapHookManager(m *hook.Manager) MapOption { ... }
func WithMapClock(fn func() time.Time) MapOption { ... }
```

- 单 RWMutex 保护整张表;
- 所有读返回**深拷贝**(节点 / 子节点切片 / metadata map 浅拷贝,与 `session.cloneSession` 风格一致);
- 节点 ID 用 `tn-<unix-nanos>-<8 hex>` 生成,与 session.GenerateID 同构。

#### 3.1.2 FileTreeStore

```go
type FileTreeStore struct {
    root  string
    locks sync.Map  // map[sessionID]*sync.Mutex

    hooks *hook.Manager
    nowFn func() time.Time
}

func NewFileTreeStore(root string, opts ...FileOption) (*FileTreeStore, error) { ... }
```

**目录布局**:

```
<root>/<sessionID>/tree/tree.json
```

- 与 `vage/session.FileSessionStore` / `vage/workspace.FileWorkspace` 共用 `<root>/<sessionID>/`;
- `SessionStore.Delete(sessionID)` 已经 `os.RemoveAll(<root>/<sessionID>)`,会一并清理树目录;
- 单文件 JSON,不分片(MVP 节点上限 1024,文件期望 < 1 MiB);
- 写路径:每次 mutation 全量 marshal → atomic rename(复用与 workspace 同构的 `writeFileAtomic` 逻辑);
- 读路径无锁(atomic rename 保证读到完整旧版或新版,永不部分);
- 写路径 per-session mutex(进程内串行化);
- 跨进程不保证(与 workspace / session 一致)。

### 3.2 事件

```go
// schema/event.go (新增)

const EventSessionTreeUpdated = "session_tree.updated"

type SessionTreeUpdatedData struct {
    SessionID string `json:"session_id"`
    Operation string `json:"operation"`            // "create" | "add" | "update" | "delete" | "cursor" | "delete_tree"
    NodeID    string `json:"node_id,omitempty"`
    NodeType  string `json:"node_type,omitempty"`
    Status    string `json:"status,omitempty"`
    NodeCount int    `json:"node_count"`           // 写后整棵树的节点数
}

func (SessionTreeUpdatedData) eventData() {}
```

事件由 Store 在 successful write 后派发(`hooks != nil` 时);失败路径不发事件,保持"hook 计数 = 成功写次数"不变量,与 SessionHook / CheckpointWritten / WorkspacePlanUpdated 一致。

---

## 4. SessionTreeSource

```go
// vage/context/sources_tree.go

package vctx

const SourceNameSessionTree = "session_tree"

type SessionTreeSource struct {
    Store tree.SessionTreeStore  // 必需;nil -> skipped

    // MaxPathDepth 限制路径渲染时从 root 开始的最大深度。0 -> 默认 6。
    // 超出深度时,头部(靠近 root)节点降级为 title-only,尾部(靠近 cursor)
    // 节点保留 summary。
    MaxPathDepth int

    // MaxSiblingTitles 限制 cursor 节点的兄弟标题列表条数。0 -> 默认 8。
    MaxSiblingTitles int

    // MaxBytes 限制单条 system 消息的字节数。0 -> 默认 8 KiB。
    // 超出时尾部截断(保留前导,因为前导是 root → cursor 的导航上下文)。
    MaxBytes int

    // Render 自定义渲染器;nil -> defaultTreeRender。
    Render TreeRenderer
}

type TreeRenderer func(in FetchInput, view TreeView) string

// TreeView 是渲染器看到的视图;Source 内部组装好后传给 Render
type TreeView struct {
    Tree              *tree.SessionTree
    Path              []*tree.TreeNode  // root → cursor
    CursorChildren    []*tree.TreeNode  // cursor 的直接子节点
    RecentDoneSibling *tree.TreeNode    // cursor 的最近完成兄弟(可空)
}

var _ Source = (*SessionTreeSource)(nil)

func (s *SessionTreeSource) Name() string { return SourceNameSessionTree }

// 不实现 MustIncludeSource —— SessionTree 是增强,不是前提。
```

### 4.1 Fetch 行为表

| 场景 | Status | 输出 |
|---|---|---|
| `Store == nil` 或 `SessionID == ""` | `skipped` | 0 messages |
| `GetTree` 返回 `ErrTreeMissing` | `skipped` | 0 messages |
| `GetTree` 返回其他 error | `error` | 0 messages,fail-open |
| 树存在但 root 缺失 (异常) | `error` | 0 messages,fail-open |
| 树存在,有 root 无 cursor | `ok` | 1 message,只渲染 root + 顶层子节点列表 |
| 树存在,有 root + cursor | `ok` | 1 message,完整渲染 |
| 渲染后字节数 > MaxBytes | `truncated` | 1 message(尾部截断),DroppedN=1 |

### 4.2 默认渲染格式

```
## Session Tree
(Persistent task structure. Use this as a navigation aid: where are we, how does this fit the overall goal.)

### Goal
[goal] [active] add OAuth login (root)
  Summary: 引入第三方登录,主要场景是企业 SSO,优先级高于自建账号体系。

### Path (root → cursor)
1. [goal] [active] add OAuth login
2. [subtask] [active] integrate Google provider  ← cursor
   Summary: callback handler + token store + scope=email,profile

### Cursor's children
- [subtask] [done] design schema
- [subtask] [active] implement callback     ← cursor
- [subtask] [pending] add e2e tests

### Recently completed (sibling)
- [subtask] [done] design schema
  Summary: chose oauth_token table with FK to users,JSON refresh_token

(Status legend: pending / active / done / blocked / superseded)
```

> 渲染顺序与 §4.8.3 算法一致:
> 1. Goal(root)title + summary;
> 2. Path 从 root 到 cursor;
> 3. Cursor 的 children titles + 状态(导航视野);
> 4. 最近完成的同级兄弟节点的 summary(LLM 看"刚才做了什么")。
>
> §4.8.3 中 step 3 的 `vector_recall(intent, scope=non_path_nodes, top_k=5)` 由独立的 `VectorRecallSource` 承担,不在本 Source 内做,避免 Source 间耦合。

### 4.3 Budget 处理

- `Budget == 0`:不裁剪,但仍受 `MaxBytes` 字节限。
- `Budget > 0`:渲染完成后估算 tokens(`memory.DefaultTokenEstimator`),若超出按字节比例截断尾部 + 标记 `Status="truncated"`;不像 VectorRecallSource 那样按"丢分数最低 hit"逻辑(树是结构化整体,丢中间会让 LLM 看到不连贯的列表)。

---

## 5. 验证策略(单元测试)

### 5.1 tree 包

| 测试文件 | 覆盖场景 |
|---|---|
| `tree_test.go` | NodeType.Valid / NodeStatus.Valid / nodeID 校验 / TreeNode 字段长度上限 |
| `mapstore_test.go` | CreateTree happy path、ErrAlreadyExists、空 root、ErrInvalidArgument(空 sessionID 等)、Get/Add/Update/Delete/SetCursor/DeleteTree、不可删 root、不可删非叶、不可改 Parent/Type、节点上限触发 ErrTreeFull、并发 Add 计数正确 |
| `filestore_test.go` | 读写路径、目录布局、原子重写、Delete 联动、跨实例可读、损坏 tree.json fail-open(GetTree → ErrTreeMissing)、validateSessionID 拦截 |
| `store_conformance_test.go` | 同一组黑盒用例对 Map / File 都跑 |

### 5.2 vctx.SessionTreeSource

`sources_tree_test.go`:

- nil store / empty session 的 skipped 路径;
- store error 的 fail-open;
- 空树(只 root) → ok,渲染 root 子节点列表;
- 含 cursor 的完整渲染;
- 兄弟节点超 `MaxSiblingTitles` → 截断 + 计数;
- 路径深度超 `MaxPathDepth` → 头部降级 title-only;
- 单消息超 `MaxBytes` → truncated,Tokens / DroppedN 报告正确;
- Budget > 0 时尾部截断;
- Custom Render panic → recoverer 兜底,emit skipped(参考 vector source 的 `recoveringRenderer`)。

### 5.3 schema 事件

`schema/event_test.go` 不必专门加(现有同模式新增 event 的测试经验:类型断言 + JSON round-trip)——而 SessionTreeUpdatedData 的覆盖在 mapstore_test.go 通过订阅 hook 验证。

### 5.4 不破坏现状

- 跑 `make test` 在 `vage/` 模块下全绿;
- 现有 `context/builder_test.go` / `context/sources_*` 全绿;
- 现有 `taskagent` / `agent` 集成测试全绿(本期不接入 TaskAgent default sources)。

---

## 6. 文档与 §4.8 标记

- 新建 `vage/.doc/session-tree.md`,与 `workspace.md` / `vector.md` 同结构(定位 / 类型 / 错误 / 实现 / Builder 集成 / 事件 / 文件结构 / out-of-scope / 与设计文档对齐)。
- `vage/.doc/index.md` 新增条目。
- `doc/design/session-context-solution.md`:
  - §4.8 标题尾部加 ✅ 已落地标记 + 引用 `vage/.doc/session-tree.md`;
  - §4.8.6 渐进路线第 1 步 ✅,第 2-5 步保持 pending;
  - §6 选型对照速查表"超长任务"行的 P10 仍保留(描述未变);
  - §8 差距汇总 "Session Tree(P10)" 行 ❌ → ✅,本期落地子集 + 后续清单;
  - §8 末尾按时间线追加本期项 (2026-04-30-003)。

---

## 7. 落地步骤(供 developer 阶段参考)

1. 写 `vage/session/tree/tree.go` —— 类型 / 常量 / 错误 / 校验。
2. 写 `vage/session/tree/store.go` —— 接口。
3. 写 `vage/session/tree/mapstore.go` —— 含事件派发 + ID 生成。
4. 写 `vage/session/tree/filestore.go` —— 单文件原子重写 + per-session lock。
5. 在 `vage/schema/event.go` 中加事件常量与 payload。
6. 写所有 store 单测(`mapstore_test.go` / `filestore_test.go` / `store_conformance_test.go` / `tree_test.go`)。
7. 写 `vage/context/sources_tree.go` 的 `SessionTreeSource` + `defaultTreeRender`。
8. 写 `vage/context/sources_tree_test.go`。
9. 跑 `make test` 在 `vage/` 模块下;
10. 跑 `make lint` 修 lint 警告;
11. 运行 documenter 阶段的文档更新。

---

## 8. 风险与决策记录

| 决策 | 取舍 |
|---|---|
| Tree 单文件 vs 节点级文件 | **单文件**:本期节点上限 1024,期望 < 1 MiB,加载/写入 O(N) 但 N 小;节点级会引入"软引用一致性"问题,出错时排查成本高 |
| 节点 Type/Parent 不可变 | **不可变**:任何 reparent 或 retype 等价于"新增节点 + 把旧节点 status=superseded"——更安全,审计链清晰。需要时再加 MoveNode |
| 派生字段 Depth 由 Store 维护 | 是:节省渲染时遍历;Update 时父子结构不可变,不会 stale |
| Source 是否做向量召回 | **不做**:与 VectorRecallSource 解耦;§4.8.3 step 3 由调用方组合两个 source 实现 |
| Source 默认开启吗 | **默认关闭**:与 §4.8.5 启用条件一致;调用方通过 `WithExtraSources(&vctx.SessionTreeSource{Store: store})` 显式开启 |
| Title/Summary 长度按字节 | **字节**:更严格,中文 80 字 ≈ 240 字节;200 字节让短中文也能用,长英文也够 |
| metadata 是否进 prompt | **不进**:与 WorkspaceSource 一致;metadata 给运维和 promotion 用 |
| MaxNodes = 1024 是不是太严 | 短期内不变:超过 1024 节点时 promotion 才有意义,本期没 promotion,严格上限反而是 LLM 的自我约束信号 |

---

## 9. 接口稳定性承诺

- `tree.NodeType` / `tree.NodeStatus`:本期五值 + 五值,枚举,公共 API,后续只扩不删。
- `tree.TreeNode` / `tree.SessionTree`:JSON 序列化兼容(添加字段需 omitempty);改名等于 break。
- `tree.SessionTreeStore` 接口:本期定型,后续只**新增**方法;签名不改。
- `vctx.SessionTreeSource`:struct 字段允许新增 Option;现有字段类型不改。
- `schema.EventSessionTreeUpdated` / `SessionTreeUpdatedData`:wire format 锁定;新增字段必须 omitempty。
