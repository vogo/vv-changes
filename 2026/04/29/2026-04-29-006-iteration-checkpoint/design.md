# 设计：迭代级 Checkpoint

> 对应需求文档：`requirement.md`。  
> 本文为 v2，整合了 `design-review.md` 的改进项；v1 原稿保留为 `design-raw.md`。  
> 主要差异：接口/类型重命名为 `IterationStore` 以避免与 `orchestrate.CheckpointStore` 撞名；`Checkpoint` 删去 `RequestMessages` / `Metadata`、补 `Estimated`；Resume 在 Final cp 上返回 `ErrAlreadyFinal` 而非重组 RunResponse；删除 ResumeOption 占位符。

## 1. 总体定位

本期在 vage 框架内新增 `vage/checkpoint/` 包并把 TaskAgent 的 ReAct 循环每轮迭代尾部接到该包，让 `(session_id, iteration)` 这条链上的中间状态可以持久化、可恢复。

设计原则一句话：**checkpoint 是"恢复用的快照" —— 独立于 Session 的"事实流（events）"和 Memory 的"prompt 用缓存"，不试图包打**。

## 2. 包边界与依赖

```
vage/checkpoint/   (新增)
  ├── 定义 Checkpoint / CheckpointMeta / IterationStore / 错误
  ├── 内置 MapIterationStore (in-memory)
  └── 内置 FileIterationStore (filesystem, 复用 session 根目录)

vage/agent/taskagent/   (修改)
  ├── 新增 WithIterationStore option
  ├── 在 Run / runStreamLoop 末尾接 saveCheckpoint
  └── 新增 Resume 方法

vage/schema/   (微改)
  └── 新增 EventCheckpointWritten + CheckpointWrittenData
```

依赖方向：`taskagent → checkpoint → schema/aimodel`。**checkpoint 包不依赖 session 包** —— 避免循环也避免给未来的"per-agent checkpoint without session"留死结。FileIterationStore 只是"恰好把目录放在 session 根下"，不引用 session 的类型。

### 2.1 命名约定

仓库内已有 `orchestrate.CheckpointStore`（DAG-level，键为 `(dagID, nodeID)`，值为 `RunResponse`）。本包使用 **`IterationStore`** 作为接口名以消除歧义；实体类型保留 `Checkpoint` 名（"checkpoint"是行业通用术语，不可让出）。错误名 `ErrCheckpointNotFound` / `ErrInvalidArgument` 跟"checkpoint"概念绑定。内置实现命名 `MapIterationStore` / `FileIterationStore`。

## 3. 核心类型

### 3.1 Checkpoint

```go
package checkpoint

// Checkpoint is a complete, restorable snapshot of one ReAct iteration.
type Checkpoint struct {
    // Identity (assigned by the store on Save).
    ID       string `json:"id"`
    Sequence int    `json:"sequence"` // 1-based, monotonic per session

    // Addressing.
    SessionID string `json:"session_id"`
    AgentID   string `json:"agent_id,omitempty"`

    // Position in the ReAct loop.
    Iteration  int                `json:"iteration"`             // 0-based, the iter that just finished
    Final      bool               `json:"final,omitempty"`       // true ⇒ run terminated at this checkpoint
    StopReason schema.StopReason  `json:"stop_reason,omitempty"` // non-empty iff Final

    // Restorable state.
    Messages        []aimodel.Message `json:"messages"`            // full snapshot the next iter would consume
    SessionMsgCount int               `json:"session_msg_count"`   // ContextBuilder offset for memory keys
    Usage           aimodel.Usage     `json:"usage"`               // accumulated up to and including this iter
    Estimated       bool              `json:"estimated,omitempty"` // mirrors runContext.estimated (stream path)

    // Audit.
    CreatedAt time.Time `json:"created_at"`
}
```

**与 v1 的差异（来自 design-review）**：
- 删除 `RequestMessages`：恢复路径上无消费者；Resume 时 reqMsgs 重置为 nil 即可，`storeAndPromoteMessages` 自然跳过 request 段写入（已经在初次 Run 时写过）。
- 删除 `Metadata`：v1 没有写入方，是典型的"为未来留口子"；真有需要时再加，零迁移成本。
- 新增 `Estimated`：与 `runContext.estimated` 对齐，避免 Resume 后 `TokenBudgetExhaustedData.Estimated` 在事件层抖动。
- `Final` 与 `StopReason` 的不变量：`Final == false ⇒ StopReason == ""`；`Final == true ⇒ StopReason != ""`。在字段文档里明示。

`Messages` 用 `aimodel.Message`（而非 `schema.Message`）的理由：与 TaskAgent 内部 ReAct 循环里 `messages` 切片的类型严格一致，恢复时直接喂给下一轮 ChatCompletion，无需转换。

### 3.2 CheckpointMeta

```go
// CheckpointMeta is the slim metadata returned by List — never embeds messages.
type CheckpointMeta struct {
    ID            string             `json:"id"`
    Sequence      int                `json:"sequence"`
    SessionID     string             `json:"session_id"`
    AgentID       string             `json:"agent_id,omitempty"`
    Iteration     int                `json:"iteration"`
    Final         bool               `json:"final,omitempty"`
    StopReason    schema.StopReason  `json:"stop_reason,omitempty"`
    MessagesCount int                `json:"messages_count"`
    Usage         aimodel.Usage      `json:"usage"`
    CreatedAt     time.Time          `json:"created_at"`
}
```

**改动**：把原 `TotalTokens int` 替换为完整的 `Usage`。理由：(1) 命名与主类型对齐（无新词典）；(2) List 视图的产品消费者通常想看 prompt/completion 拆分，否则要再 Load 一次；(3) `aimodel.Usage` 体积小（4-5 个 int），meta 列表整体不会因此爆炸。

### 3.3 接口

```go
type IterationStore interface {
    // Save persists cp. The store assigns Sequence and ID; any caller-supplied
    // values for these fields are overwritten. Sequence is strictly monotonic
    // per SessionID — the store guards this under its own lock so concurrent
    // Save calls on the same session are serialized.
    //
    // After Save returns nil, cp.Sequence / cp.ID / cp.CreatedAt are populated.
    Save(ctx context.Context, cp *Checkpoint) error

    // Load returns a checkpoint by id. id == "" means "the latest by Sequence"
    // — equivalent to "most recently written". The sessionID parameter is
    // required (double-keys help file backends locate the directory directly
    // and catch caller bugs where an id is mixed up across sessions).
    // Returns ErrCheckpointNotFound when nothing matches.
    Load(ctx context.Context, sessionID, id string) (*Checkpoint, error)

    // List returns CheckpointMeta in ascending Sequence order. An empty
    // session returns ([]) without error.
    List(ctx context.Context, sessionID string) ([]*CheckpointMeta, error)

    // Delete removes every checkpoint for sessionID. Idempotent on
    // unknown id.
    Delete(ctx context.Context, sessionID string) error
}
```

**为什么不用 `(*Checkpoint, error)` 返回新指针**：避免一次拷贝；与 `vage/session` 的 Save/Update 风格保持一致（in-place mutation 写回 ID/Sequence/CreatedAt）。

**Save 的 store-assigned-only 语义**：原 v1 暗示"caller-supplied values are honored when restoring across stores"。本期没有跨 store 迁移的消费者，开放这一口子会把"sequence per session 严格单调"的不变量从"store 维护"降级为"caller + store 共谋"。如果将来真的有跨 store 迁移需求，新增 `Import(ctx, *Checkpoint) error` 方法即可，零向后兼容代价。

### 3.4 错误

```go
var (
    ErrCheckpointNotFound = errors.New("checkpoint: not found")
    ErrInvalidArgument    = errors.New("checkpoint: invalid argument")
    ErrAlreadyFinal       = errors.New("checkpoint: session already finalized")
)
```

`ErrAlreadyFinal` 由 `TaskAgent.Resume` 在最新 cp 是 Final 时返回（详见 §5.5）。

不引入 `ErrCheckpointExists`：同一 session 只允许 append，sequence 由 store 自分配，写冲突在并发层就解决了。

## 4. 内置实现

### 4.1 MapIterationStore

单一 `sync.RWMutex` 保护整张表（与 `MapSessionStore` 相同）：

```go
type MapIterationStore struct {
    mu   sync.RWMutex
    data map[string][]*Checkpoint   // sessionID -> ordered by Sequence
    seq  map[string]int             // sessionID -> last assigned Sequence
}
```

写：拷贝入参后 append；读返回的 cp 是 deep copy（防外部 mutation 污染内部状态，主要发生在测试场景）。

### 4.2 FileIterationStore

目录布局（与 `FileSessionStore` 共用根但不互相引用）：

```
<root>/<session_id>/checkpoints/000001-<8byte_hex>.json
<root>/<session_id>/checkpoints/000002-<8byte_hex>.json
...
```

- `<root>` 由构造函数接收，不默认推导为 session 根 —— 调用方可以共用也可以分根。
- 文件名前缀 `000001` 是 6 位零填充 sequence；`ls` 直接按时间序；后缀 `<8byte_hex>` 即 Checkpoint.ID。
- 写：原子重写（temp file `*.json.tmp` + rename），与 `vage/session/filestore.go` 的 `writeJSONAtomic` 完全一致；文件权限 `0o600`、目录 `0o700`。
- 并发：per-session `sync.Mutex`（`sync.Map` 懒分配），与 FileSessionStore 同模式。
- **Sequence 分配**：在 lock 内 `os.ReadDir(checkpoints/)`、**仅识别 `.json` 后缀（跳过 `.json.tmp`）** 取前缀 max + 1。这一步代价 O(N) 在文件数千以下完全可接受（一个 session 撑到 1000 iter 已经异常）；如果未来真有压力，可以缓存"上次分配的 sequence"在 lock 内递增。

#### 4.2.1 SQLite 后端的演进路径（备注，不在本期实现）

`IterationStore` 是抽象层，FileStore / MapStore / 未来的 SQLiteStore 平行实现。建议 SQLite schema：

```sql
CREATE TABLE iteration_checkpoints (
    session_id TEXT NOT NULL,
    sequence   INTEGER NOT NULL,
    id         TEXT NOT NULL,
    payload    TEXT NOT NULL,         -- json
    created_at TIMESTAMP NOT NULL,
    PRIMARY KEY (session_id, sequence)
);
CREATE INDEX idx_session_id ON iteration_checkpoints(session_id);
```

这意味着我们的接口设计不需要为本期所未做的事预留任何字段。

### 4.3 共用契约测试

`vage/checkpoint/store_conformance_test.go` 用一组 helper 把 MapStore / FileStore 都跑一遍：

- 空 session List 返回 `([], nil)`。
- 连续 Save 三次，sequence 单调；`cp.Sequence > 0 && cp.ID != ""`（store-assigned 字段确实回填）。
- Save 不同 sessionID 的 sequence 各自从 1 开始（per-session 单调，跨 session 不共享）。
- Load(`""`) 返回最后一个；Load(指定 id) 返回该条；Load(不存在) 返回 `ErrCheckpointNotFound`；**Load(sessionA, idFromSessionB) 返回 `ErrCheckpointNotFound`**（双参校验）。
- Delete 清空，再 List 返回空；Delete 不存在的 session 不报错。

与 `vage/session/store_conformance_test.go` 的写法一致。

## 5. TaskAgent 集成

### 5.1 新增 option

```go
// taskagent/task.go
type Agent struct {
    ...
    iterationStore checkpoint.IterationStore
}

func WithIterationStore(s checkpoint.IterationStore) Option {
    return func(a *Agent) { a.iterationStore = s }
}
```

未配置时（zero value），所有 checkpoint 路径整体短路 —— 与"不开 memoryManager 时存储跳过"同一个不变量。

### 5.2 写入时机

**严格在每轮迭代尾部写一次**，无论该轮是不是终态：

```text
for iter in 0..maxIter:
    [pre-budget check; chat completion; append assistant msg]
    if no_tool_calls or budget_exhausted_after:
        save_checkpoint(Final=true, StopReason=...)
        finalizeRun(...)
        return
    [post-budget check]
    [execute tool batch; append tool msgs]
    save_checkpoint(Final=false)
# fall-out:
save_checkpoint(Final=true, StopReason=MaxIterations)
finalizeRun(...)
```

- "Iteration ends with assistant-only message"（无工具调用）→ checkpoint Final=true，StopReason=Complete。
- "Iteration ends with tool batch"（还要再一轮）→ checkpoint Final=false。
- Budget exhausted / max iterations → checkpoint Final=true。

**为什么写 Final checkpoint**：(1) 让 List 返回的最后一条就是任务结束态，作为产品视图直接可读；(2) 让 Resume 在已完成 session 上能立即识别（返回 `ErrAlreadyFinal`，调用方明确处理而不是误以为"resume 成功"）；(3) 让"finalize 之前崩溃"和"finalize 之后崩溃"在恢复路径上不需要分叉（都按最后一条 cp 即可）。

### 5.3 写入失败的取舍

`saveCheckpoint` 失败 **不阻断主路径** —— `slog.Warn` 一次后丢弃。理由：
1. 一致性优先级低于活性。一次 ReAct 任务的最大价值是它正在产生 token；checkpoint 的价值是"如果崩溃，能恢复"。崩溃是低概率事件，让低概率事件的兜底机制把高概率主路径打挂是反直觉的。
2. 与 `SessionHook` 的失败语义一致（也是 warn-and-drop）。

代价：极端情况（store 持续写不进），可能恢复时拿到的是几轮以前的快照。**外部观测路径**：调用方对比"`EventCheckpointWritten` 计数 ≠ 实际迭代次数（可由 `EventIterationStart` 计数）"即可发现。我们**不**在本期引入 `WithCheckpointStrictMode` 这类 opt-in 严格模式（YAGNI；调用方真有"不能丢"诉求时更合理的方式是 store 实现自身做 retry/replication）。

### 5.4 数据填充

每轮结束时构造 cp 的伪码：

```go
cp := &checkpoint.Checkpoint{
    SessionID:       rc.sessionID,
    AgentID:         a.ID(),
    Iteration:       iter,
    Final:           final,
    StopReason:      stopReason,                // empty when !final
    Messages:        cloneMessages(messages),   // see §5.7
    SessionMsgCount: rc.br.sessionMsgCount,
    Usage:           rc.totalUsage,
    Estimated:       rc.estimated,
}
if err := a.iterationStore.Save(ctx, cp); err != nil {
    slog.Warn("vage: save iteration checkpoint", "error", err,
        "session_id", rc.sessionID, "iteration", iter)
    return
}
a.dispatch(ctx, schema.NewEvent(schema.EventCheckpointWritten, a.ID(), rc.sessionID,
    schema.CheckpointWrittenData{
        CheckpointID:  cp.ID,
        Sequence:      cp.Sequence,
        Iteration:     cp.Iteration,
        Final:         cp.Final,
        StopReason:    cp.StopReason,
        MessagesCount: len(cp.Messages),
        TotalTokens:   cp.Usage.TotalTokens,
    }))
```

写入失败时**不**派发 hook event（保留 hook 计数 = 成功写入 cp 数 这个不变量；外部用 hook count vs. iter count 的差值发现异常）。

### 5.5 Resume API

```go
// Resume re-enters the ReAct loop for sessionID using the latest stored
// checkpoint.
//
// Errors:
//   - ErrInvalidArgument when no IterationStore is configured.
//   - ErrCheckpointNotFound when the session has no checkpoints.
//   - ErrAlreadyFinal when the latest checkpoint is Final == true (the run
//     has already terminated; the caller decides whether to surface the
//     stored RunResponse via List/Load or treat as a no-op).
//
// Resume bypasses input guards (the input was vetted in the original Run).
// Output guards still run on the final response of the resumed run. Tool
// result guards continue to run on every fresh tool execution.
func (a *Agent) Resume(ctx context.Context, sessionID string) (*schema.RunResponse, error)
```

**与 v1 的差异**：
- 删除 `ResumeOption` / `resumeConfig` 占位符。Go 的 variadic option 增加是非破坏性的，等真有需求再加。
- Final cp 不再"重组 RunResponse 返回"。理由：那条路径会复制 finalizeRun 的一半（output guards 不能再跑、AgentEnd 事件可能重复、duration 字段语义模糊），调用方把"已终结"误解为"resume 成功"反而埋雷。返回明确错误 `ErrAlreadyFinal`，让调用方自己用 `Load(sessionID, "")` 拿到最后一条 cp 来读 Messages / StopReason。

### 5.6 Resume 的执行步骤

```text
1. require a.iterationStore != nil; otherwise return ErrInvalidArgument
2. cp, err := store.Load(ctx, sessionID, "")
3. if err == ErrCheckpointNotFound: return same error to caller
4. if cp.AgentID != "" && cp.AgentID != a.ID():
       return ErrInvalidArgument (cross-agent resume disallowed)
5. if cp.Final: return ErrAlreadyFinal
6. rebuild runContext with
        sessionID         = cp.SessionID
        tracker           = newBudgetTracker(0)        // fresh budget per Resume
        totalUsage        = cp.Usage                   // restore accumulated
        estimated         = cp.Estimated
        br.messages       = cp.Messages
        br.sessionMsgCount= cp.SessionMsgCount
        reqMsgs           = nil                        // see §3.1 RequestMessages removal
        iteration         = cp.Iteration + 1
   then enter the ReAct loop at that iteration count for the remaining budget
   of MaxIterations (a.maxIterations - cp.Iteration - 1).
```

`storeAndPromoteMessages` 自然处理 `reqMsgs == nil`（`for _, msg := range reqMsgs` 即空循环），response 段照常 promote —— 这与初次 Run 的 finalize 在恢复后产生的"response 段"是同一条 message，因此整体语义是 idempotent 的（同一条 response 只会在 finalize 那一次被写入 working memory）。

**关于 RunOptions（model / temperature / maxIterations）**：v1 不接收 RunOptions，全部使用 agent 默认值。理由：原 Run 的 RunOptions 没有持久化在 cp 里，硬要恢复语义要么撒谎要么全量序列化（包括 `*float64` / 切片），收益小、复杂度大。需求场景"工具调用打挂、跑同一 agent 续命"用默认值就够。下一期可以加 `Resume(ctx, sid, opts...ResumeOption)` 显式传入。

### 5.7 Messages 的浅拷贝

`cloneMessages` 不深拷贝 `Content` / `ToolCalls` 内部指针，只复制顶层切片结构：

```go
func cloneMessages(in []aimodel.Message) []aimodel.Message {
    out := make([]aimodel.Message, len(in))
    copy(out, in)
    return out
}
```

**安全前提**：`Content` 与 `ToolCalls` 在 ReAct 循环里只被**追加新元素**，既有元素不被原地 mutation。`CacheBreakpoint` 字段确实在 `markPromptCacheBreakpoints` 中 in-place 设置在 system 消息上 —— 但 system 消息位于切片首位，每次 Resume 重建 `messages` 切片时 backing array 是新分配的，cp 之间不会互相污染。

MapIterationStore 在 Save / Load 时仍然深拷贝整个 cp（防外部代码改了 cp 后污染），主要发生在测试环节，性能不敏感。

### 5.8 Stream 路径

`runStreamLoop` 路径同样接 saveCheckpoint，只是 send 改用 stream emitter；hook 由 `buildSend` 统一派发。Final checkpoint 的写入紧接 `finalizeStream` 之前。`rc.estimated` 在 stream 路径下默认 true，因此 stream 写入的 cp 会带 `Estimated=true`。

### 5.9 与 prompt cache 的相处

`markPromptCacheBreakpoints` 在 Resume 重建 messages 后照常调用（`a.promptCaching` 控制），与初次 Run 路径一致。Anthropic 的 cache 命中由 prefix hash 决定 —— 我们只是给同样的 system 消息打上同样的 boundary 标记，命中率与不中断时一致。

## 6. Schema 改动

`vage/schema/event.go` 新增：

```go
const EventCheckpointWritten = "checkpoint_written"

type CheckpointWrittenData struct {
    CheckpointID  string             `json:"checkpoint_id"`
    Sequence      int                `json:"sequence"`
    Iteration     int                `json:"iteration"`
    Final         bool               `json:"final,omitempty"`
    StopReason    StopReason         `json:"stop_reason,omitempty"`
    MessagesCount int                `json:"messages_count"`
    TotalTokens   int                `json:"total_tokens"`
}

func (CheckpointWrittenData) eventData() {}
```

事件以 hook 形式向外发，不进 stream（stream 是 LLM 文本/工具事件流，checkpoint 是后台基础设施，不该污染 SSE）。

## 7. 测试矩阵

### 7.1 单元（colocated）

- `vage/checkpoint/checkpoint_test.go`：JSON 序列化 round-trip、Sequence 比较辅助。
- `vage/checkpoint/mapstore_test.go` / `filestore_test.go`：实现自身的小测试。
- `vage/checkpoint/store_conformance_test.go`：黑盒契约（见 §4.3 的 case 列表）。
- `vage/agent/taskagent/checkpoint_test.go`（新文件）：用 fake `aimodel.ChatCompleter`：
  - 跑 3 轮（finish_reason: tool_calls, tool_calls, stop）→ List 返回 3 条 meta，最后一条 Final=true。
  - 第 2 轮 LLM 报错 → Run 返回错误，store 中仍有 1 条 Final=false 的 cp。
  - Resume 在错误后调用 → 取最新 cp → 继续第 3 轮 → 完成；总 Save 次数 = 2 + 1 = 3。
  - **已经 Final 的 session 调用 Resume → 返回 `ErrAlreadyFinal`**（不发起 LLM 调用）。
  - **找不到 cp 的 session → 返回 `ErrCheckpointNotFound`**。
  - **`WithIterationStore` 为 nil 时调用 Resume → 返回 `ErrInvalidArgument`**；普通 Run 行为与不开等价。
  - **跨 agent 校验**：cp.AgentID 与 a.ID() 不一致时 Resume 返回 `ErrInvalidArgument`。

### 7.2 集成（`vage/integrations/taskagent_tests/`）

- `checkpoint_resume_test.go`：用真实 `taskagent.New` + `MapIterationStore`，在 fake LLM + fake bash tool 组合下验证：跑两轮、模拟"中断"（直接停 ctx 或人为 panic）、新 ctx 重建 agent + Resume → 最终 Final 消息文本与"完整跑一次的对照组"等价。

### 7.3 不测的事

- 不测 `FileIterationStore` 在跨进程下的并发（不在范围内）。
- 不测 token cache 命中（aimodel 层的责任）。
- 不在 `largemodel` 中间件链中构造特殊场景测试（与本期无关）。

## 8. 文档落地

- 新增 `vage/.doc/checkpoint.md`：包结构、API、写入时机、Resume 语义、与 `orchestrate.CheckpointStore` 的区别。
- 修改 `vage/.doc/index.md` 与 `vage/.doc/architecture.md`：把"iteration-level"作为新条目纳入，避免读者把它跟 orchestrate 那个混淆。
- 修改 `doc/design/session-context-solution.md`：§4.5 标记 ✅ 已落地（部分），列出 fork / interrupt 留作后续；§8 差距汇总更新对应行；§4.5 补"实际 API 与原始草案的差异"表（与 §4.1 / §4.2 / §4.4 一致的格式）。

## 9. 决策清单（一行一条，留作 reviewer 复核）

| 决策 | 选择 | 备选 |
|---|---|---|
| 是否单独建包 | 是（`vage/checkpoint/`） | 挂在 `session/` 下 → 接口耦合到 session lifecycle |
| 接口名 | `IterationStore` | `CheckpointStore` → 与 orchestrate 撞名 |
| Messages 类型 | `aimodel.Message` | `schema.Message` → 每次恢复要转换 |
| Sequence 来源 | store 内分配，**忽略 caller 输入** | caller / unix-nanos / store-or-caller |
| Final 检测 | 独立字段 + 不变量 | 用 StopReason != "" 推断 → 不严格 |
| Final cp 的 Resume | 返回 `ErrAlreadyFinal` | 重组 RunResponse 返回 → 复制 finalize 半路、语义混淆 |
| `RequestMessages` 字段 | 删除 | 保留 → 没有真实消费者 |
| `Metadata` 字段 | 删除 | 保留 → YAGNI |
| `Estimated` 字段 | 新增 | 不存 → Resume 后事件层抖动 |
| `CheckpointMeta.TotalTokens` | 改为 `Usage` | 保留 int → List 看不到 prompt/completion 拆分 |
| `ResumeOption` placeholder | 删除 | 保留占位 → 纯噪声，需要时加签名是非破坏性的 |
| 失败语义 | warn-and-drop | 中断 Run → 主路径敏感 |
| 严格模式 opt-in | 不做 | YAGNI |
| Resume 是否带 RunOptions | 不带 | 持久化 RunOptions → 复杂度跳跃 |
| HTTP wiring | 不做 | 留下一迭代 |

## 10. 实施顺序（developer 入手指南）

1. 先建 `vage/checkpoint/` 包：types → IterationStore 接口 → MapIterationStore → FileIterationStore → 共用契约测试。
2. 在 `vage/schema/event.go` 加 EventCheckpointWritten + CheckpointWrittenData。
3. TaskAgent：拆出新文件 `vage/agent/taskagent/checkpoint.go`，里面放 saveCheckpoint helper / Resume 主体；在 `task.go` 加 option + 在 Run / runStreamLoop 写入点调用 helper。
4. 单测先行（fake completer 的最小测试），再写集成测试。
5. 跑 `make build && make lint && make test`。
6. documenter 阶段补 `vage/.doc/checkpoint.md` + 更新 architecture.md / index.md / 主设计文档。
