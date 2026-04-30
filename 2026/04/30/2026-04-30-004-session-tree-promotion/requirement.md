# Session Tree Promotion + 折叠（vage 框架层）

## 1. 背景与目标

### 1.1 上文回顾

`doc/design/session-context-solution.md` §4.8 提出 P10 层级树记忆模式，作为长任务（多日跨 session、强目标、多子任务）场景下"结构本身就是一种压缩"的核心抽象。

§4.8.6 的渐进路线分 5 步：
1. ✅ MVP：手工节点（无 promotion）—— 已在 2026-04-30-003 落地（`vage/session/tree/`）。
2. ❌ 加 promotion：异步反思，复用 compressor —— **本期范围**。
3. ❌ 加双索引：summary 入向量库 —— 留作后续。
4. ❌ 加 LLM 工具：暴露 `zoom/promote/pin` 工具 —— 留作后续。
5. ❌ 跨 session 树森林 —— 留作后续。

### 1.2 待解决问题

当 SessionTree MVP 在长任务里跑久了，会遇到三个问题：

1. **节点上限硬撞墙**：`MaxNodes = 1024` 是没有 promotion 时的硬约束，超出 → `ErrTreeFull`，调用方除了人工归档没有出路。
2. **Prompt 噪声放大**：`SessionTreeSource` 默认渲染 cursor 的全部子节点 + 路径节点的 summary。当 cursor 父节点累计 50+ 子节点时，即使有 `MaxSiblingTitles = 8` 截断，调用方也丢失了"此前完成的多个子任务的语义"。
3. **结构无法收缩**：兄弟子节点全部 `done` 后，只剩一份扁平列表，没有"父节点已经汇总了上述结论"的上抛动作。

### 1.3 本期目标

实现 §4.8.2 描述的 **promotion（向上提取层级）**：把若干兄弟子节点的 (title, summary) 聚合，更新到父节点的 summary，子节点降权（标 `Promoted=true`，仍可检索但默认不进 prompt）；并在 §4.8.3 渲染算法中**折叠**已 promoted 的子节点，仅显示 `(folded: N children, M done)` 占位。

### 1.4 范围（本次只做 vage 框架层 = §A1–A5）

| # | 范围 | 文件 |
|---|---|---|
| A1 | 数据模型扩展（`Promoted bool` + `PromotedAt` + 触发阈值常量） | `vage/session/tree/tree.go` |
| A2 | Store 接口扩展（`PromoteNode` + `GetTreeView` + `ViewOptions`） | `vage/session/tree/store.go` + 两个内置实现 |
| A3 | Promoter 接口与 3 个内置实现（LLM / Compressor / Noop） | `vage/session/tree/promoter.go`（新文件） |
| A4 | 触发器、异步执行、singleflight、3 个新事件 | `vage/session/tree/triggers.go`（新文件） + `vage/schema/event.go` |
| A5 | 渲染层折叠 + `WithIncludePromoted` option | `vage/context/sources_tree.go` |

### 1.5 明示 out-of-scope

- A6 LLM 工具包 `vage/tool/sessiontree/`
- B1–B7 vv 应用层 wiring（配置、setup、CLI、HTTP、agent 注册、prompt）
- 双索引（summary 入向量库）
- 跨 session 树森林
- MoveNode / Reparent
- SQLite/Postgres 后端、跨进程文件锁

---

## 2. 用户故事与验收标准

### 2.1 US-1：作为 vage 框架使用方，我希望节点带 Promoted 标记并被渲染层折叠

**验收标准**：
- AC-1.1 `TreeNode` 新增字段 `Promoted bool`（json `"promoted,omitempty"`）+ `PromotedAt time.Time`（json `"promoted_at,omitempty"`）；
- AC-1.2 `Promoted=true` 的节点在 `SessionTreeSource` 默认渲染时不出现在 `Cursor's children` / `Recently completed` 段；
- AC-1.3 父节点的"折叠"指示文本格式为 `(folded: N children, M done)`，N=被折叠的子节点总数，M=其中 status==done 的数量；折叠数 == 0 时不输出该行；
- AC-1.4 路径上的节点（root → cursor 链）即使 `Promoted=true` 也仍然渲染（保证不丢失"我们在哪一步"的导航信息）。

### 2.2 US-2：作为框架使用方，我希望显式调用 `PromoteNode` 把若干子节点聚合到父节点

**验收标准**：
- AC-2.1 `SessionTreeStore` 新增 `PromoteNode(ctx, sessionID, nodeID) (*TreeNode, error)`；
- AC-2.2 行为：用配置的 `Promoter.Summarize(ctx, parent, eligibleChildren)` 生成新 summary；写回 `parent.Summary`，每个被折叠的子节点 `Promoted=true` + `PromotedAt=now`，并在子节点 `Metadata["summary_source"] = "promotion"`；父节点 `Metadata["summary_source"] = "promotion"`（如果是 promotion 路径写回的）；
- AC-2.3 安全护栏：
  - `Pinned=true` 的子节点**永不**被折叠；
  - `Promoted=true` 的子节点跳过（幂等）；
  - 没有可折叠子节点（即 eligible == 0）时不修改任何字段，返回当前父节点 + nil 错；
- AC-2.4 错误处理：
  - 节点不存在 → `ErrNotFound`；
  - 该 session 没树 → `ErrTreeMissing`；
  - 没有可折叠子节点 → 返回当前父节点 + nil；
  - Promoter 返回错 → 不修改任何字段，向上抛错（透明）。
- AC-2.5 串行性：store 内部对同一 session 的 PromoteNode 调用与其他写操作互斥（Map 用 RWMutex，File 用 per-session Mutex）；
- AC-2.6 事件：成功后派发 `EventSessionTreePromotionCompleted`（payload：`SessionID / ParentID / FoldedCount / NewSummaryBytes`）。

### 2.3 US-3：作为框架使用方，我希望在读取树时按需过滤已 promoted 的节点

**验收标准**：
- AC-3.1 `SessionTreeStore` 新增 `GetTreeView(ctx, sessionID, opts ViewOptions) (*SessionTree, error)`；
- AC-3.2 `ViewOptions{IncludePromoted bool}`：默认 `false`（即过滤掉 promoted 节点的子树，只保留路径上的 promoted 节点）；
- AC-3.3 当 `IncludePromoted=true` 时返回完整树（与 `GetTree` 等价）；
- AC-3.4 `IncludePromoted=false` 时：从结果的 `Nodes` 里移除 `Promoted=true` 节点（路径节点除外，由调用方判断），同时父节点的 `Children` 列表对应剔除被 promoted 的子节点 ID。
- AC-3.5 `GetTreeView` 返回值仍为深拷贝（caller 可安全修改）。

> 注：根据原始草案 A2 的描述，"把过滤下沉到 store" 是为了避免每个调用方各自再过滤；但**路径过滤**因为依赖 `cursor`，最适合放在 Source 渲染层做。Store 端 `GetTreeView` 只做"统一剔除 promoted 节点"，Source 自己决定如何处理路径上的特殊情况。

### 2.4 US-4：作为框架使用方，我希望 promotion 摘要器可插拔（LLM/无 LLM/无操作）

**验收标准**：
- AC-4.1 `vage/session/tree.Promoter` 接口：`Summarize(ctx context.Context, parent *TreeNode, children []*TreeNode) (string, error)`；
- AC-4.2 `LLMPromoter`：注入 `aimodel.ChatCompleter` + 可选 `Model string`；prompt 模板拼接父节点 + 子节点的 (title, summary)；返回模型生成的新 summary；
- AC-4.3 `CompressorPromoter`：注入 `memory.ContextCompressor`；把每个子节点格式化为 `schema.Message`，调用 Compressor 压缩到 `SummaryMaxBytes` 字节预算（用 token 估算近似），返回拼接结果；
- AC-4.4 `NoopPromoter`：永远返回 `(parent.Summary, nil)`，子节点信息丢失但仍触发 `Promoted=true`（用于纯标记降权场景）；
- AC-4.5 三个实现都安全处理 `len(children) == 0` 的 corner case。

### 2.5 US-5：作为框架使用方，我希望 store 在写入后能按阈值自动决定是否触发 promotion

**验收标准**：
- AC-5.1 `PromotionDecider` 接口：`ShouldPromote(parent *TreeNode, children []*TreeNode) bool`；
- AC-5.2 内置阈值：
  - `MinChildren int`（默认 8）—— 子节点数量超过该阈值；
  - `AllChildrenDone bool`（默认 true）—— 所有非 promoted 子节点都 done；
  - `MinSubtreeBytes int`（默认 8 KiB）—— 子节点 (title+summary) 字节累计超过阈值；
- AC-5.3 提供组合器：`AnyOf(deciders...)` / `AllOf(deciders...)`；
- AC-5.4 触发时机：`AddNode` / `UpdateNode` 写成功并释放锁后，**同步判断**该节点的 parent 是否满足条件，满足时**异步**调用 `PromoteNode`；
- AC-5.5 防重入：per-(sessionID, parentID) singleflight；同一 (session, parent) 已有 in-flight promotion 时跳过新触发；
- AC-5.6 异步事件：触发瞬间派发 `EventSessionTreePromotionStarted`，完成后派发 `EventSessionTreePromotionCompleted`，失败时派发 `EventSessionTreePromotionFailed`（包含错误信息）；
- AC-5.7 Decider == nil（默认状态）→ Store 不做任何自动触发，仅暴露同步 `PromoteNode` 给手动调用。

### 2.6 US-6：作为渲染层使用方，我希望可以显式打开"包含 promoted 节点"模式（zoom-in）

**验收标准**：
- AC-6.1 `SessionTreeSource` 新增字段 `IncludePromoted bool`（默认 false），等价于 `WithIncludePromoted(true)` 风格的 option（直接写字段更符合现有 Source 风格 —— `MaxBytes / MaxPathDepth` 都是字段）；
- AC-6.2 `IncludePromoted=true` 时不折叠任何节点，渲染所有 children/recent；
- AC-6.3 `IncludePromoted=false`（默认）时按 US-1 的折叠规则渲染。

---

## 3. 行业实践调研（关键启示）

| 来源 | 启示 |
|---|---|
| RAPTOR (ICLR 2024) | 递归聚类 + 摘要构建多层摘要树；本期采用"显式触发 + 单层向上"，不递归到根 —— 保持 MVP 简单。 |
| Generative Agents (UIST 2023) | Reflection 机制：累积观察后异步生成洞察 —— 与 §A4 "异步执行 + 阈值触发"对齐。 |
| MemGPT | Page-in/Page-out 由 LLM 自己发起 —— 本期保留了 PromoteNode 的同步入口给 §A6 的 LLM 工具用，但本期不暴露。 |
| LangGraph Reducer | reducer 链可声明组合 —— 与 `AnyOf/AllOf` decider 对齐。 |
| singleflight (golang.org/x/sync) | 同一 key 的并发调用合并为一次执行 —— 直接复用为防重入手段。 |
| Anthropic Claude Code | Auto-Compact 在临近 context 阈值时整体摘要老对话 —— 与本期"父节点超阈值时折叠子节点"同源。 |

**结论**：方案完整性评估 → §4.8 已经覆盖了节点模型、触发器、执行步骤、安全护栏（pinned）、与其他模块的关系，**完整性足够**。落地时只需要把"伪代码 + 阈值"翻译成 Go 接口与代码即可。

---

## 4. 受影响范围

### 4.1 包

- `vage/session/tree`：核心改动；新增 promoter / triggers，扩展 tree.go / store.go / mapstore.go / filestore.go；
- `vage/context`：渲染层改动 `sources_tree.go` + `sources_tree_test.go`；
- `vage/schema`：新增 3 个事件常量与 payload 类型。

### 4.2 不受影响

- `vage/session`：本包不依赖 session.SessionStore；
- `vage/workspace`：独立模块，互不关联；
- `vage/memory`：本包对其零依赖；CompressorPromoter 通过接口反向依赖 `memory.ContextCompressor`，不引入循环；
- `vage/tool/*`：本期不交付 LLM 工具；
- `vv/*`：本期不做应用层 wiring。

### 4.3 向后兼容

- `TreeNode` 新增字段都带 `omitempty`：旧的 tree.json 反序列化时 `Promoted=false / PromotedAt=zero`，渲染行为不变。
- `SessionTreeSource` 新增字段 `IncludePromoted` 默认 false：旧 callsite 不需要改。
- `SessionTreeStore` 新增方法 `PromoteNode / GetTreeView`：仅对实现了 `SessionTreeStore` 的**外部**类型构成 break；本仓库内的 Map / File store 一并升级。

---

## 5. 验证策略（提示给 designer/tester）

### 5.1 单测覆盖

- promoter 三个实现（含 children 为空 / Compressor 错误 / LLM ChatCompleter 错误）
- decider 阈值（MinChildren / AllChildrenDone / MinSubtreeBytes / AnyOf / AllOf）
- store.PromoteNode：happy path / Pinned 排除 / 已 Promoted 跳过 / Promoter 错回滚 / 没可折叠子节点 / 树/节点不存在
- store.GetTreeView：IncludePromoted=true vs false / 路径节点是否保留
- triggers：AddNode 后 decider 命中 → PromoteNode 异步调用 / singleflight 防重入 / 三个事件按序派发
- SessionTreeSource：折叠占位文本 / IncludePromoted=true 关闭折叠 / 路径上的 promoted 节点仍渲染

### 5.2 一致性测试

- 黑盒 conformance（Map vs File）覆盖 PromoteNode / GetTreeView 至少 2 例；保证两种 store 行为对齐。

---

## 6. 不会做的事（明示）

- **不做**节点的递归折叠（即 promoted 子节点的子节点也不会做二次 promotion）—— 本期保持单层。
- **不做**自动 unfold（即 promoted=true 后没有 API 可以变回 false）—— 调用方需要写一个 `UpdateNode` 显式重置 `Promoted=false`。MVP 假设 promotion 是单调的。
- **不做**同步触发器：所有触发都是异步的，store 写路径永不阻塞在 Promoter 上。
- **不做**跨 store 的事件去重：调用方自己的 hook subscriber 需要处理 PromotionCompleted 事件可能重复（理论上 singleflight 已防重入，但崩溃恢复语义没有承诺）。
