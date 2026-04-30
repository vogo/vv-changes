# 设计：迭代级 Checkpoint

> 对应需求文档：`requirement.md`。

## 1. 总体定位

本期在 vage 框架内新增 `vage/checkpoint/` 包并把 TaskAgent 的 ReAct 循环每轮迭代尾部接到该包，让 `(session_id, iteration)` 这条链上的中间状态可以持久化、可恢复。设计原则一句话：**checkpoint 是"恢复用的快照"——独立于 Session 的"事实流（events）"和 Memory 的"prompt 用缓存"，不试图包打**。

## 2. 包边界与依赖

```
vage/checkpoint/   (新增)
  ├── 定义 Checkpoint / CheckpointMeta / CheckpointStore / 错误
  ├── 内置 MapCheckpointStore (in-memory)
  └── 内置 FileCheckpointStore (filesystem, 复用 session 根目录)

vage/agent/taskagent/   (修改)
  ├── 新增 WithCheckpointStore option
  ├── 在 Run / runStreamLoop 末尾接 saveCheckpoint
  └── 新增 Resume 方法

vage/schema/   (微改)
  └── 新增 EventCheckpointWritten + CheckpointWrittenData
```

依赖方向：`taskagent → checkpoint → schema/aimodel`。**checkpoint 包不依赖 session 包**——避免循环也避免给未来的"per-agent checkpoint without session"留死结。FileCheckpointStore 只是"恰好把目录放在 session 根下"，不引用 session 的类型。

## 3. 核心类型

### 3.1 Checkpoint

```go
package checkpoint

// Checkpoint is a complete, restorable snapshot of one ReAct iteration.
type Checkpoint struct {
    // Identity (assigned by the store on Save when zero-valued).
    ID       string `json:"id"`
    Sequence int    `json:"sequence"` // 1-based, monotonic per session

    // Addressing.
    SessionID string `json:"session_id"`
    AgentID   string `json:"agent_id,omitempty"`

    // Position in the ReAct loop.
    Iteration  int                `json:"iteration"`           // 0-based, the iter that just finished
    Final      bool               `json:"final,omitempty"`     // run terminated at this checkpoint
    StopReason schema.StopReason  `json:"stop_reason,omitempty"` // valid only when Final

    // Restorable state.
    Messages        []aimodel.Message `json:"messages"`         // full snapshot sent to LLM next iter
    RequestMessages []schema.Message  `json:"request_messages"` // original user input of this Run
    SessionMsgCount int               `json:"session_msg_count"`// ContextBuilder offset for memory keys
    Usage           aimodel.Usage     `json:"usage"`            // accumulated up to and including this iter

    // Audit.
    CreatedAt time.Time      `json:"created_at"`
    Metadata  map[string]any `json:"metadata,omitempty"`
}

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
    TotalTokens   int                `json:"total_tokens"`
    CreatedAt     time.Time          `json:"created_at"`
}
```

**为什么 Messages 用 `aimodel.Message` 而非 `schema.Message`**：与 TaskAgent 内部 ReAct 循环里 `messages` 切片的类型严格一致，恢复时直接喂给下一轮 ChatCompletion，无需转换。`RequestMessages` 用 `schema.Message` 是因为 storeAndPromoteMessages 接收的就是该类型，恢复后写入 working memory 时直接传入。

### 3.2 接口

```go
type CheckpointStore interface {
    // Save persists cp. The store assigns Sequence and ID when the input
    // values are zero/empty; otherwise the caller-chosen values are honored
    // (used when restoring a snapshot from another store). Sequence is
    // strictly monotonic per SessionID — the store guards this under its
    // own lock so concurrent Save calls on the same session are serialized.
    //
    // After Save returns nil, cp.Sequence / cp.ID / cp.CreatedAt are set.
    Save(ctx context.Context, cp *Checkpoint) error

    // Load returns a checkpoint by id. id == "" means "the latest by
    // Sequence" — equivalent to "most recently written". Returns
    // ErrCheckpointNotFound when nothing matches.
    Load(ctx context.Context, sessionID, id string) (*Checkpoint, error)

    // List returns CheckpointMeta in ascending Sequence order. An empty
    // session returns ([]) without error (consistent with "no checkpoints
    // yet" being a normal state).
    List(ctx context.Context, sessionID string) ([]*CheckpointMeta, error)

    // Delete removes every checkpoint for sessionID. Idempotent on
    // unknown id (consistent with FileSessionStore.Delete).
    Delete(ctx context.Context, sessionID string) error
}
```

**为什么不用 `(*Checkpoint, error)` 返回新指针**：避免一次拷贝 + 让 caller 一目了然 "in-place mutation"，与 `vage/session` 的 Save/Update 风格保持一致。

**为什么 Load 用 `sessionID, id` 双参**：双重校验——`id` 在跨 session 之间不保证唯一（uuid 短 token），加 sessionID 既能加速 file 后端的查找（不必扫整个 root）也能在 caller 误传 ID 时立刻报错。

### 3.3 错误

```go
var (
    ErrCheckpointNotFound = errors.New("checkpoint: not found")
    ErrInvalidArgument    = errors.New("checkpoint: invalid argument")
)
```

不引入 `ErrCheckpointExists` —— 同一 session 只允许 append，sequence 由 store 自分配，写冲突在并发层就解决了。

## 4. 内置实现

### 4.1 MapCheckpointStore

单一 `sync.RWMutex` 保护整张表（与 `MapSessionStore` 相同的"读 / 写都走同一个 lock"策略，业务量级在会话节奏，不需要 per-session lock）。

```go
type MapCheckpointStore struct {
    mu    sync.RWMutex
    data  map[string][]*Checkpoint   // sessionID -> ordered by Sequence
    seq   map[string]int             // sessionID -> last assigned Sequence
}
```

写：拷贝入参后 append，写返回的 cp 也是 deep copy（避免外部 mutation 污染）。

### 4.2 FileCheckpointStore

目录布局（与 `FileSessionStore` 共用根但不互相引用）：

```
<root>/<session_id>/checkpoints/000001-<8byte_hex>.json
<root>/<session_id>/checkpoints/000002-<8byte_hex>.json
...
```

- `<root>` 由构造函数接收，**不**默认推导为 session 根 —— 调用方可以共用也可以分根。
- 文件名前缀 `000001` 是 6 位零填充 sequence，`ls` 直接按时间序；后缀 `<8byte_hex>` 即 Checkpoint.ID（确保碰撞概率可忽略，并让 `Load(id)` 有 O(1) 命中——找到第一个以 `*-id.json` 结尾的文件即可，但实现里直接全名拼接更快）。
- 写：原子重写（temp file + rename），与 `vage/session/filestore.go` 的 `writeJSONAtomic` 完全一致；文件权限 `0o600`、目录 `0o700`。
- 并发：per-session `sync.Mutex`（`sync.Map` 懒分配），与 FileSessionStore 同模式。
- Sequence 分配：在 lock 内 `os.ReadDir(checkpoints/)` 取文件名前缀 max + 1。这一步代价 O(N) 在文件数千以下完全可接受（一个 session 撑到 1000 iter 已经异常）；如果未来真的有压力，可以缓存"上次分配的 sequence"在 lock 内递增。

### 4.3 共用契约测试

`vage/checkpoint/store_conformance_test.go` 用一组 helper 把 MapStore / FileStore 都跑一遍：
- 空 session List 返回 `([], nil)`；
- 连续 Save 三次，sequence 单调；
- Load(`""`) 返回最后一个；Load(指定 id) 返回该条；Load(不存在) 返回 `ErrCheckpointNotFound`；
- Delete 清空，再 List 返回空；Delete 不存在的 session 不报错。

与 `vage/session/store_conformance_test.go` 的写法一致。

## 5. TaskAgent 集成

### 5.1 新增 option

```go
// taskagent/task.go
type Agent struct {
    ...
    checkpointStore checkpoint.CheckpointStore
}

func WithCheckpointStore(s checkpoint.CheckpointStore) Option {
    return func(a *Agent) { a.checkpointStore = s }
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

要点：
- "Iteration ends with assistant-only message"（无工具调用）→ checkpoint Final=true，StopReason=Complete。
- "Iteration ends with tool batch"（还要再一轮）→ checkpoint Final=false。
- Budget exhausted / max iterations → checkpoint Final=true。

**为什么写终态 checkpoint**：(1) 让 Resume 在已经完成的 session 上能识别"无事可做"；(2) 让 List 返回的最后一条就是任务结束态，作为产品视图直接可读；(3) 让"finalize 之前崩溃"和"finalize 之后崩溃"在恢复路径上不需要分叉（都按最后一条 checkpoint 即可）。

### 5.3 写入失败的取舍

`saveCheckpoint` 失败 **不阻断主路径** —— `slog.Warn` 一次后丢弃。理由：
1. 一致性优先级低于活性。一次 ReAct 任务的最大价值是它正在产生 token；checkpoint 的价值是"如果崩溃，能恢复"。崩溃是低概率事件，让低概率事件的兜底机制把高概率主路径打挂是反直觉的。
2. 与 `SessionHook` 的失败语义一致（也是"warn-and-drop"）。

代价：极端情况（store 持续写不进），可能恢复时拿到的是几轮以前的快照。验收侧通过"checkpoint 写入事件计数 ≠ 迭代计数"的 hook 对比即可发现，不需要引擎层兜底。

### 5.4 数据填充

每轮结束时 build 新 checkpoint 的代码片段（伪码）：

```go
cp := &checkpoint.Checkpoint{
    SessionID:       rc.sessionID,
    AgentID:         a.ID(),
    Iteration:       iter,
    Final:           final,
    StopReason:      stopReason,            // empty when !final
    Messages:        cloneMessages(messages), // see §5.7
    RequestMessages: rc.reqMsgs,
    SessionMsgCount: rc.br.sessionMsgCount,
    Usage:           rc.totalUsage,
}
_ = a.checkpointStore.Save(ctx, cp)
a.dispatch(ctx, schema.NewEvent(schema.EventCheckpointWritten, a.ID(), rc.sessionID,
    schema.CheckpointWrittenData{
        CheckpointID:  cp.ID,
        Sequence:      cp.Sequence,
        Iteration:     cp.Iteration,
        Final:         cp.Final,
        MessagesCount: len(cp.Messages),
        TotalTokens:   cp.Usage.TotalTokens,
    }))
```

### 5.5 Resume API

```go
// Resume re-enters the ReAct loop for sessionID using the latest stored
// checkpoint. Returns ErrCheckpointNotFound when no checkpoint exists for
// the session. Returns the assembled response immediately when the latest
// checkpoint is Final == true (no LLM call is made).
//
// Resume bypasses input guards (the input was already vetted in the
// original Run). Output guards still run on the final response. Tool
// result guards continue to run on every fresh tool execution.
func (a *Agent) Resume(ctx context.Context, sessionID string, opts ...ResumeOption) (*schema.RunResponse, error)
```

**为什么是 `Resume` 而不是 `Run` 的一个 RunOption**：Resume 的入参形状本质不同（没有 user prompt，没有 messages 输入），如果硬塞进 RunOption 会把 RunRequest 的语义弄脏。独立方法读者一眼能判断意图。

`ResumeOption` v1 留 placeholder，本期不开放任何参数，避免接口为未来需求过早承诺：

```go
type ResumeOption func(*resumeConfig)
type resumeConfig struct{}  // intentionally empty, future-proofing only
```

### 5.6 Resume 的执行步骤

```text
1. require a.checkpointStore != nil; otherwise return ErrInvalidArgument
2. cp, err := store.Load(ctx, sessionID, "")
3. if err == ErrCheckpointNotFound: return same error to caller
4. if cp.Final: assemble RunResponse from last assistant message in cp.Messages
                + cp.Usage + cp.StopReason; emit AgentStart + AgentEnd events;
                return without calling LLM
5. else: rebuild runContext with
        sessionID = cp.SessionID
        tracker   = newBudgetTracker(0)               // fresh budget per Resume
        totalUsage = cp.Usage                         // restore accumulated
        br.messages         = cp.Messages
        br.sessionMsgCount  = cp.SessionMsgCount
        reqMsgs             = cp.RequestMessages
        iteration           = cp.Iteration + 1
   then enter the ReAct loop at that iteration count for the remaining budget
   of MaxIterations (a.maxIterations - cp.Iteration - 1).
```

**关于 RunOptions（model / temperature / maxIterations）**：v1 不接收 RunOptions，全部使用 agent 默认值。理由：原 Run 的 RunOptions 没有持久化在 checkpoint 里，硬要恢复语义要么撒谎要么全量序列化（包括 `*float64` / 切片），收益小、复杂度大。需求场景"工具调用打挂、跑同一 agent 续命"用默认值就够。下一期可以加 `WithResumeOptions` option 显式传入。

### 5.7 Messages 的浅拷贝

`cloneMessages` 不做深拷贝 `Content` / `ToolCalls` 内部指针，只复制顶层切片结构：

```go
func cloneMessages(in []aimodel.Message) []aimodel.Message {
    out := make([]aimodel.Message, len(in))
    copy(out, in)  // ToolCalls slice header copied; tool calls themselves immutable post-creation
    return out
}
```

`aimodel.Message.Content` 持有的内部 parts 在生成后不再 mutation；`ToolCalls` 也是如此。深拷贝纯属浪费。但 store 的 in-memory 实现仍然在 Load 时深拷贝（防外部代码改了 messages 后污染），主要发生在测试环节，性能不敏感。

### 5.8 Stream 路径

`runStreamLoop` 路径同样接 saveCheckpoint，只是 send 改用 `send(...)` 走 stream emitter，hook 由 `buildSend` 统一派发。Final checkpoint 的写入紧接 `finalizeStream` 之前。

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
- `vage/checkpoint/store_conformance_test.go`：黑盒契约（Save 单调 / Load latest / Load by id / 错误路径 / Delete 幂等）。
- `vage/agent/taskagent/checkpoint_test.go`（新文件）：用 fake `aimodel.ChatCompleter`：
  - 跑 3 轮（finish_reason: tool_calls, tool_calls, stop）→ List 返回 3 条 meta，最后一条 Final=true。
  - 第 2 轮 LLM 报错 → Run 返回错误，store 中仍有 1 条 Final=false 的 cp。
  - Resume 在错误后调用 → 取最新 cp → 继续第 3 轮 → 完成；总 Save 次数 = 2 + 1 = 3。
  - 已经 Final 的 session 调用 Resume → 不发起 LLM 调用，直接返回最后一条 assistant 消息。
  - `WithCheckpointStore` 为 nil 时与不开等价（行为对照）。

### 7.2 集成（`vage/integrations/taskagent_tests/`）

- `checkpoint_resume_test.go`：用真实 `taskagent.New` + `MapCheckpointStore`，在 fake LLM + fake bash tool 组合下验证：跑两轮、模拟"中断"（直接停 ctx 或人为 panic）、新 ctx 重建 agent + Resume → 最终 Final 消息文本与"完整跑一次的对照组"等价。

### 7.3 不测的事

- 不测 `FileCheckpointStore` 在跨进程下的并发（不在范围内）。
- 不测 token cache 命中（aimodel 层的责任）。
- 不在 `largemodel` 中间件链中构造特殊场景测试（与本期无关）。

## 8. 文档落地

- 新增 `vage/.doc/checkpoint.md`：包结构、API、写入时机、Resume 语义、与 orchestrate checkpoint 的区别。
- 修改 `vage/.doc/index.md` 与 `vage/.doc/architecture.md`：把"iteration-level"作为新条目纳入，避免读者把它跟 orchestrate 那个混淆。
- 修改 `doc/design/session-context-solution.md`：§4.5 标记 ✅ 已落地（部分），列出 fork / interrupt 留作后续；§8 差距汇总更新对应行；§4.5 补 "实际 API 与原始草案的差异" 表（与 §4.1 / §4.2 / §4.4 一致的格式）。

## 9. 决策清单（一行一条，留作 reviewer 复核）

| 决策 | 选择 | 备选 |
|---|---|---|
| 是否单独建包 | 是（`vage/checkpoint/`） | 挂在 `session/` 下 → 接口耦合到 session lifecycle |
| Messages 类型 | `aimodel.Message` | `schema.Message` → 每次恢复要转换 |
| Sequence 来源 | store 内分配 | caller / unix-nanos |
| Final 检测 | 独立字段 | 用 StopReason != "" 推断 → 不严格 |
| 失败语义 | warn-and-drop | 中断 Run → 主路径敏感 |
| Resume 是否带 RunOptions | 不带 | 持久化 RunOptions → 复杂度跳跃 |
| HTTP wiring | 不做 | 留下一迭代 |

## 10. 实施顺序（developer 入手指南）

1. 先建 `vage/checkpoint/` 包：types → store interface → MapStore → FileStore → 共用契约测试。
2. 在 `vage/schema/event.go` 加 EventCheckpointWritten + Data。
3. TaskAgent：拆出新文件 `vage/agent/taskagent/checkpoint.go`，里面放 saveCheckpoint helper / Resume 主体；在 `task.go` 加 option + 在 Run / runStreamLoop 写入点调用 helper。
4. 单测先行（fake completer 的最小测试），再写集成测试。
5. 跑 `make build && make lint && make test`。
6. documenter 阶段补 `vage/.doc/checkpoint.md` + 更新 architecture.md / index.md / 主设计文档。
