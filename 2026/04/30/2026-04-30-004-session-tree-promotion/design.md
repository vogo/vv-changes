# 技术设计：Session Tree Promotion + 折叠（vage 框架层）

## 0. 设计原则

1. **复用 MVP 形成的契约**：`SessionTreeStore` 接口已经定型，不破坏；新增方法、新增字段，旧调用站不受影响。
2. **同步操作 + 异步触发分离**：`PromoteNode` 是同步的（调用方要等结果），自动触发器是异步的（不阻塞写路径）。
3. **fail-open 渲染**：`SessionTreeSource` 在不认识新字段时退化为旧行为；任何新错误路径都返回 fail-open（保持现有 Source 风格）。
4. **不引入循环依赖**：`vage/session/tree` 不依赖 `vage/memory`、`vage/largemodel`；通过窄接口（`aimodel.ChatCompleter`、`memory.ContextCompressor`）反向接入。

---

## 1. 数据模型（A1）

### 1.1 TreeNode 字段扩展

```go
type TreeNode struct {
    // ... existing fields ...

    Pinned    bool      `json:"pinned,omitempty"`
    Promoted  bool      `json:"promoted,omitempty"`     // NEW
    PromotedAt time.Time `json:"promoted_at,omitempty"` // NEW

    // ... existing fields ...
}
```

- `Promoted`：标记该节点已被折叠到父节点 summary 里，渲染层默认跳过（路径上的节点除外）；
- `PromotedAt`：标记发生折叠的时间点；fail-open 容忍 zero value。

`Metadata["summary_source"]`（约定，不在结构体里强制）取值：
- `"user"` 或不存在 → caller 写入的 summary；
- `"promotion"` → 由 PromoteNode 路径写入的 summary（父节点）或被折叠子节点的标识。

### 1.2 触发阈值常量

```go
const (
    DefaultPromotionMinChildren     = 8
    DefaultPromotionMinSubtreeBytes = 8 * 1024
)
```

不在常量里硬编码 `AllChildrenDone bool` 这种"开关"——它是 decider 字段。

### 1.3 cloneNode 适配

`Promoted` 是 bool、`PromotedAt` 是 time.Time，都是值类型；现有 `cloneNode` 的 `out := *n` 直接拷贝，无需改动。

---

## 2. Store 接口扩展（A2）

### 2.1 接口新增方法

```go
// store.go

type ViewOptions struct {
    // IncludePromoted controls whether nodes with Promoted=true (and their
    // descendants) are kept in the returned tree. Default false hides them
    // so callers can render a "folded" view without filtering themselves.
    IncludePromoted bool
}

type SessionTreeStore interface {
    // ... existing methods ...

    // PromoteNode aggregates the eligible children of nodeID into nodeID's
    // summary using the configured Promoter, then marks each folded child as
    // Promoted=true. Eligible = !Pinned && !Promoted. Returns the updated
    // parent node (deep copy) on success, or:
    //   - ErrTreeMissing / ErrNotFound when target absent;
    //   - whatever Promoter.Summarize returns (transparent);
    //   - nil when there is nothing eligible to fold (parent is unchanged).
    PromoteNode(ctx context.Context, sessionID, nodeID string) (*TreeNode, error)

    // GetTreeView returns the tree filtered by opts. opts.IncludePromoted=true
    // is equivalent to GetTree.
    GetTreeView(ctx context.Context, sessionID string, opts ViewOptions) (*SessionTree, error)
}
```

### 2.2 PromoteNode 实现要点（同时适用 Map / File store）

伪代码：

```text
acquire write lock for sessionID
load tree
parent := tree.Nodes[nodeID]; require non-nil → ErrNotFound
collect eligible children: !Promoted && !Pinned
if len(eligible) == 0:
    release lock; return clone(parent), nil    // no-op, no event
release lock        // Promoter may take long (LLM call)
newSummary, err := promoter.Summarize(ctx, parent, eligible)
if err != nil:
    return nil, err

acquire write lock again
re-load tree (could have changed)
re-fetch parent; if missing → ErrNotFound (race lost; consider this a failure)
re-evaluate eligible: only fold those still {!Promoted && !Pinned} & still children
parent.Summary = clamp(newSummary, SummaryMaxBytes)
parent.Metadata["summary_source"] = "promotion"
parent.UpdatedAt = now
foldedCount := 0
for each eligible child still present:
    child.Promoted = true
    child.PromotedAt = now
    child.Metadata["summary_source"] = "promotion"
    child.UpdatedAt = now
    foldedCount++
tree.UpdatedAt = now
write tree back
release lock
dispatch EventSessionTreePromotionCompleted{ParentID, FoldedCount, NewSummaryBytes}
return clone(parent), nil
```

**两阶段加锁**的原因：Promoter（LLM/compressor）可能耗时秒级；不能持有 store 的写锁。第二次加锁后重新校验状态防止 race。

**第一次"无可折叠"短路**：避免无意义的 Promoter 调用 + 无事件干扰审计。

### 2.3 GetTreeView 实现

```text
tr := readTree(sessionID)        // 已 deep-clone
if opts.IncludePromoted: return tr
walk tr.Nodes, build delete set:
    for each node n where n.Promoted: include n.ID and all descendants
for each id in delete set: delete tr.Nodes[id]
for each remaining node: rewrite n.Children, removing IDs in delete set
return tr
```

**注意**：路径节点（root → cursor）即使 Promoted=true 也由调用方决定是否保留——`GetTreeView` 不知道 cursor 语义，所以 store 端**统一剔除**所有 Promoted 节点。Source 渲染层自己拿 `GetTree`（含 promoted）来构造路径，避免路径上 promoted 节点被错误剔除。

> 这是与原始草案 A2 略微不同：草案让 caller 一次拿到"过滤后的视图"；但路径过滤需要 cursor 语义，所以 Source 不能直接用 `GetTreeView` 的过滤结果。Source 改用 `GetTree`（含 promoted）+ 自己在渲染时跳过非路径 promoted 节点。`GetTreeView` 仍然有用——给将来非路径相关的查询（如 `IncludePromoted=true` 的 zoom-in、HTTP `GET /tree?include_promoted=1`）提供统一入口。

---

## 3. Promoter（A3）

新文件 `vage/session/tree/promoter.go`。

### 3.1 接口

```go
type Promoter interface {
    // Summarize returns a new summary text for parent that aggregates the
    // information in children. Implementations MUST tolerate len(children)==0
    // by returning (parent.Summary, nil). The store will clamp the returned
    // string to SummaryMaxBytes; implementations need not clamp themselves.
    Summarize(ctx context.Context, parent *TreeNode, children []*TreeNode) (string, error)
}
```

### 3.2 LLMPromoter

```go
type LLMPromoter struct {
    Client aimodel.ChatCompleter   // required
    Model  string                  // optional; "" → caller-default
    SystemPrompt string            // optional override
    MaxOutputTokens int            // optional; 0 → 512
}
```

实现：拼接 prompt：

```
[system] You are a senior assistant compressing sub-task notes.
Output a concise summary (≤ 200 字 / ~600 bytes) of the parent node's progress
based on its children. Reply with a single paragraph; no headings, no lists.

[user]
PARENT
- Title: <parent.Title>
- Status: <parent.Status>
- Existing summary: <parent.Summary>

CHILDREN (<n> total)
1. [type] [status] title — summary
2. ...
```

调用 `aimodel.ChatCompleter.ChatCompletion`，取第一个 choice 的 text。

### 3.3 CompressorPromoter

```go
type CompressorPromoter struct {
    Compressor memory.ContextCompressor  // required
    MaxBytes   int                       // optional; 0 → SummaryMaxBytes
}
```

实现：把 `parent.Summary + 每个 child` 渲染成 `[]schema.Message`（user/assistant 角色），调用 `Compressor.Compress(ctx, msgs, tokenBudget)`，把返回的 messages 拼成单一字符串。

`tokenBudget := bytesToTokens(MaxBytes)` —— 简单按 4 字节 ≈ 1 token 估算。

### 3.4 NoopPromoter

```go
type NoopPromoter struct{}

func (NoopPromoter) Summarize(_ context.Context, parent *TreeNode, _ []*TreeNode) (string, error) {
    if parent == nil {
        return "", nil
    }
    return parent.Summary, nil
}
```

用于"我只想标记降权，不要 LLM 调用"的场景。

---

## 4. 触发器与异步执行（A4）

新文件 `vage/session/tree/triggers.go`。

### 4.1 PromotionDecider

```go
type PromotionDecider interface {
    ShouldPromote(parent *TreeNode, children []*TreeNode) bool
}

type DeciderFunc func(parent *TreeNode, children []*TreeNode) bool
func (f DeciderFunc) ShouldPromote(p *TreeNode, c []*TreeNode) bool { return f(p, c) }
```

### 4.2 内置 deciders

```go
type ChildrenCountDecider struct {
    Min int  // 0 → DefaultPromotionMinChildren
}

type AllChildrenDoneDecider struct{}

type SubtreeBytesDecider struct {
    Min int  // 0 → DefaultPromotionMinSubtreeBytes
}

func AnyOf(deciders ...PromotionDecider) PromotionDecider
func AllOf(deciders ...PromotionDecider) PromotionDecider
```

ChildrenCountDecider / SubtreeBytesDecider 都按"非 promoted、非 pinned"的子节点过滤后计数。AllChildrenDoneDecider 也只考察非 promoted 子节点（避免把"全部已 done 但已 promoted 的子节点"再次触发）。

### 4.3 Store 集成（不破坏接口的可插拔点）

不把 `Decider / Promoter / async runner` 加到 `SessionTreeStore` 接口；只加到 **MapTreeStore / FileTreeStore 的 option** 上：

```go
// 公共 option 位（map 与 file 各自暴露同名 option，行为一致）
WithMapPromoter(p Promoter) MapOption
WithMapPromotionDecider(d PromotionDecider) MapOption
WithMapPromotionAsync(fn func(func())) MapOption  // 默认 go func() { fn() }()

WithFilePromoter(p Promoter) FileOption
WithFilePromotionDecider(d PromotionDecider) FileOption
WithFilePromotionAsync(fn func(func())) FileOption
```

> Async 注入器（`func(func())`）让测试可以注入"同步执行"，无需 sleep 等待 goroutine。

### 4.4 单流（singleflight）防重入

每个 store 持一个 `sync.Map[promotionKey]struct{}`（promotionKey = sessionID + "|" + parentID），CAS 占位。已有占位时跳过新触发；占位释放在 `PromoteNode` 返回（成功/失败/skip）后的 defer 中。

不引入 `golang.org/x/sync/singleflight` —— 它的语义是"合并并发调用为同一结果"，而我们想要的是"已有 in-flight 时**丢弃**新触发"。手写一个 set 即可，避免新依赖。

### 4.5 触发位置

- `AddNode`：写成功且释放主锁后，对**新节点的 parent** 跑 decider；命中则异步执行 `PromoteNode(parent.ID)`；
- `UpdateNode`：写成功且释放主锁后，对**被更新节点本身**跑 decider（典型场景：把某 child 状态改为 done 后，触发 parent 折叠）。
  - 实现：取 `n.Parent`，对父节点跑 decider。如果 `n.Parent == ""`（即 n 是 root），跳过。
- `PromoteNode` 自身**不**触发递归 promotion（避免循环）。

### 4.6 三个新事件

`vage/schema/event.go`：

```go
const (
    EventSessionTreePromotionStarted   = "session_tree.promotion.started"
    EventSessionTreePromotionCompleted = "session_tree.promotion.completed"
    EventSessionTreePromotionFailed    = "session_tree.promotion.failed"
)

type SessionTreePromotionStartedData struct {
    SessionID string `json:"session_id"`
    ParentID  string `json:"parent_id"`
    Eligible  int    `json:"eligible"`        // pre-flight count
}

type SessionTreePromotionCompletedData struct {
    SessionID       string `json:"session_id"`
    ParentID        string `json:"parent_id"`
    FoldedCount     int    `json:"folded_count"`
    NewSummaryBytes int    `json:"new_summary_bytes"`
}

type SessionTreePromotionFailedData struct {
    SessionID string `json:"session_id"`
    ParentID  string `json:"parent_id"`
    Error     string `json:"error"`
}
```

事件由 store 在 PromoteNode 路径**异步**派发；同步调用 `PromoteNode` 也派发（统一）。

> 同步路径会派发 `*Started` 事件吗？答：**会**。事件语义跟"路径"无关——只跟"实际执行"有关。caller 直接调用 PromoteNode 时也会看到 Started/Completed 对。

---

## 5. 渲染层折叠（A5）

`vage/context/sources_tree.go` + `sources_tree_test.go`。

### 5.1 字段新增

```go
type SessionTreeSource struct {
    // ... existing fields ...

    // IncludePromoted disables folding: when true, nodes with Promoted=true
    // are rendered alongside the rest. Default false (the typical "folded"
    // view used in the prompt).
    IncludePromoted bool
}
```

### 5.2 buildView 调整

`buildView` 已经收集 `CursorChildren / RecentDoneSibling`。改动：

- `CursorChildrenN` 改为只统计**非 promoted** 的子节点数（默认行为）；新增 `CursorChildrenPromotedN`、`CursorChildrenPromotedDone` 表示折叠掉的子节点统计：
  ```go
  type TreeView struct {
      // ... existing fields ...
      CursorChildrenPromotedN    int  // count of skipped (promoted) children
      CursorChildrenPromotedDone int  // among those, count with Status==done
  }
  ```
- 当 `s.IncludePromoted == true` 时，按完整子节点列表填充 CursorChildren，PromotedN/PromotedDone 仍然填（信息更全），但 renderer 不打印 folded 行。
- `RecentDoneSibling`：跳过 promoted 兄弟（除非 `IncludePromoted=true`）。

### 5.3 defaultTreeRender 调整

在 `### Cursor's children` 段尾部追加：

```
(folded: <PromotedN> children, <PromotedDone> done)
```

仅当 `PromotedN > 0` 且 `IncludePromoted == false` 时输出。

路径节点的 promoted 处理：路径上的节点**始终渲染**（保留导航信息，符合 AC-1.4）。`pathFromRoot` 不需要改动。

### 5.4 失效路径

- `IncludePromoted=true` + 树中没有 promoted 节点 → 行为与现状完全一致；
- `IncludePromoted=false` + 树中没有 promoted 节点 → CursorChildrenPromotedN=0，不输出 folded 行，与现状一致。

测试覆盖这两种"无 promoted 节点"路径以保证不回归。

---

## 6. 文件清单与改动量估算

| 文件 | 改动类型 | 估算 LOC |
|---|---|---|
| `vage/session/tree/tree.go` | 字段 + 阈值常量 | +20 |
| `vage/session/tree/store.go` | 接口扩展 + ViewOptions 类型 | +30 |
| `vage/session/tree/mapstore.go` | PromoteNode + GetTreeView + decider 触发 + option | +220 |
| `vage/session/tree/filestore.go` | 同上（结构对称） | +220 |
| `vage/session/tree/promoter.go` | 新文件：3 个 promoter | +180 |
| `vage/session/tree/triggers.go` | 新文件：deciders + AnyOf/AllOf + 共享 helpers | +160 |
| `vage/session/tree/promoter_test.go` | 单测 | +220 |
| `vage/session/tree/triggers_test.go` | 单测 | +180 |
| `vage/session/tree/store_conformance_test.go` | conformance 新增 PromoteNode/GetTreeView 用例 | +180 |
| `vage/session/tree/mapstore_test.go` | 异步触发 + singleflight 测试 | +120 |
| `vage/context/sources_tree.go` | 字段 + 渲染分支 | +60 |
| `vage/context/sources_tree_test.go` | 折叠 / IncludePromoted / 路径节点 | +180 |
| `vage/schema/event.go` | 3 个事件常量 + 3 个 payload | +60 |
| **合计** | — | **~1830** |

注：原始草案估"~700"。实际多出来的部分主要在测试 / 两个 store 的并行实现 / promoter 三个变体。

---

## 7. 关键风险与权衡

| 风险 | 缓解 |
|---|---|
| Promoter 调用慢（LLM 数秒）持有 store 写锁 → 阻塞写路径 | 两阶段加锁：Promoter 调用在锁外。 |
| 异步触发器导致写入 burst → 大量 goroutine | per-(session,parent) singleflight 跳过新触发；异步注入器允许调用方限流。 |
| 第二次加锁时父节点已删 / 子节点已变 | 重新校验 eligible；零余量时不修改并 skip-completed，不返回错（避免误报失败）。 |
| 用户依赖 `GetTree` 看完整树（含 promoted）→ 改 GetTree 行为会破坏兼容 | 不改 `GetTree`；新增 `GetTreeView` 提供过滤入口。 |
| Tree.json 老格式没有 promoted 字段 | omitempty + bool/time.Time 默认零值 → 反序列化天然兼容。 |
| LLMPromoter 输出 > SummaryMaxBytes | store 端按 SummaryMaxBytes 字节截断（utf-8 安全裁剪：在最近一个 ASCII / 完整 utf-8 边界处切）。 |

---

## 8. 验证计划

### 8.1 单测

- `tree_test.go`：新字段的 JSON round-trip（promoted/promoted_at omitempty）。
- `promoter_test.go`：
  - `LLMPromoter`：mock ChatCompleter happy path / error path / 空 children；
  - `CompressorPromoter`：mock Compressor / 空 children / Compressor 错；
  - `NoopPromoter`：返回 parent.Summary 不变。
- `triggers_test.go`：
  - 各 decider 阈值边界；
  - AnyOf/AllOf 短路 / 反例。
- `store_conformance_test.go`（Map/File 共享）：
  - `PromoteNode happy path`：父 summary 更新 + 子节点 Promoted=true + Metadata["summary_source"]="promotion"；
  - `PromoteNode 跳过 Pinned 子节点`；
  - `PromoteNode 子节点已 promoted 时跳过`；
  - `PromoteNode no eligible → no-op + no event`；
  - `PromoteNode promoter 错 → no fields changed`；
  - `GetTreeView IncludePromoted=false` 过滤掉 promoted 子树；
  - `GetTreeView IncludePromoted=true` == GetTree。
- `mapstore_test.go`（针对异步 trigger 的 Map-only 测试）：
  - 注入同步 async runner；AddNode 命中 decider → PromoteNode 被调用；
  - singleflight：连续 5 次 AddNode → PromoteNode 仅被调用 1 次（用 counter 验证）；
  - 三个事件按序派发（mock hook collector）。
- `sources_tree_test.go`：
  - 默认 IncludePromoted=false：promoted children 折叠为占位行；
  - IncludePromoted=true：promoted children 出现在列表；
  - 路径节点 Promoted=true 仍渲染；
  - 没有 promoted 节点时与基线行为一致；
  - RecentDoneSibling 跳过 promoted 兄弟。

### 8.2 集成测试

本期无新增 integration test（promotion 不需要真实 LLM —— 用 LLMPromoter 对接 mock ChatCompleter 已足够覆盖契约）。

### 8.3 验收用 sanity 命令

```bash
cd vage && go test ./session/tree/... ./context/... -v -count=1
cd vage && make lint
```
