# 设计评审：迭代级 Checkpoint

> 评审对象：`design.md`（v1，将归档为 `design-raw.md`）。本文档列出可落地的改进项，按"接受 / 部分接受 / 拒绝"分组；接受项已合入新版 `design.md`。

## 0. 总评

候选设计整体方向正确（独立包、独立接口、文件后端复用 session 根目录、warn-and-drop、Final 字段独立）。需要削减的是**过早抽象**和**冗余字段**，需要明确的是**与 orchestrate.CheckpointStore 的命名区分**和**Resume 的边界条件**。所有提议保持单 dev pass 可实现。

---

## 1. 命名 / 包边界

### 1.1 与 `orchestrate.CheckpointStore` 同名冲突 — 接受

**发现**：仓库已存在 `orchestrate.CheckpointStore`（DAG-level，键为 `(dagID, nodeID)`，值为 `RunResponse`）。本期再引入一个同名 `checkpoint.CheckpointStore` 会让读者在调用点 `xxx.CheckpointStore` 时无法立即判断语义；尤其是当 vv 同时配置两个 store 时（未来一定会发生），import alias 几乎不可避免。

**建议**：
- 类型重命名为 `checkpoint.IterationStore`（与字段语义"迭代快照"对齐），实体类型保留 `Checkpoint`。
- 错误保留 `ErrCheckpointNotFound` / `ErrInvalidArgument`（错误名跟"checkpoint"概念绑定，不跟接口名绑定）。
- 内置实现相应改为 `MapIterationStore` / `FileIterationStore`。
- TaskAgent option 改为 `WithIterationStore(s checkpoint.IterationStore)`。

**代价**：一次性命名调整，几乎零实现增量；阻止未来一年的"哪个 checkpoint store"歧义。

---

## 2. `Checkpoint` 数据形状

### 2.1 删除 `RequestMessages` 字段 — 接受

**发现**：`Checkpoint.RequestMessages` 在恢复路径上**没有真实消费者**：
- ReAct 循环里 `messages` 已经包含 user-as-message（由 `RequestMessagesSource` 在 build 时拼进 `cp.Messages`），下一轮 LLM 不需要单独的 `reqMsgs`。
- `storeAndPromoteMessages` 只在 `finalizeRun / finalizeStream` 末尾被调用一次，恢复后 finalize 时拿到的"req → resp"配对，按"恢复后这段 Run 不再 owns 原始 user 消息"语义处理就够：要么不写入 working memory（已经在初次 Run 时写过），要么用空切片。
- 序列化/反序列化的成本不大但语义混淆代价高（`Messages` 与 `RequestMessages` 的关系不变量必须维护）。

**建议**：删除 `RequestMessages`。Resume 在重建 runContext 时 `reqMsgs = nil`。`storeAndPromoteMessages` 在 reqMsgs 为 nil 时跳过 request 段写入（已实现：`for _, msg := range reqMsgs` 自动空循环），response 段照常 promote — 这是恢复路径上 idempotent 的（同一条 response 只会在 finalize 那一次被写入）。

### 2.2 删除 `Metadata map[string]any` — 接受

**发现**：v1 没有任何调用方写入 metadata；hook payload 已独立。开放一个 `map[string]any` 是典型的"为未来留口子"反模式 —— 真到了需要的时候再加，零迁移成本（旧 cp 的 nil map 与新版兼容）。

### 2.3 `AgentID` 保留 — 接受

恢复时验证 cp.AgentID == a.ID() 可以阻止"用 A agent 的 checkpoint 喂给 B agent"，是廉价的安全网。保留。

### 2.4 `Estimated` 字段补充 — 接受

**发现**：原 design 在 Resume 重建 runContext 时把 `tracker = newBudgetTracker(0)`、`totalUsage = cp.Usage`，但**没有恢复 `rc.estimated`**。stream 路径会把 estimated 重置为 true，sync 路径设为 false（默认值）—— 这本身没问题，但 hook 事件和 finalizeRun 的 `TokenBudgetExhaustedData.Estimated` 字段会因恢复路径而抖动。

**建议**：
- 在 `Checkpoint` 增加 `Estimated bool` 字段（与 `runContext.estimated` 对齐），随 Usage 一起序列化。
- Resume 时 `rc.estimated = cp.Estimated`。
- 不是"必须"修复但属于"正确性补丁"，单 dev pass 可顺手落地。

### 2.5 `CheckpointMeta.TotalTokens` 命名 — 接受

**发现**：`CheckpointMeta` 同时含 `Sequence / Iteration / TotalTokens`，但 TotalTokens 在 Checkpoint 主类型里叫 `Usage.TotalTokens`。命名不一致，且 list 视图想看 prompt/completion 拆分时只能再 Load 一次。

**建议**：把 `CheckpointMeta.TotalTokens int` 替换为 `CheckpointMeta.Usage aimodel.Usage`（或浅拷贝的子集）。Usage 类型本身不大（4-5 个 int），meta 列表也不会因此爆炸。如果非要 slim，至少把字段名换成 `UsageTotal` 或保留 Usage 完整结构 —— 选后者，命名一致性比 5 字节的 List 体积更重要。

---

## 3. `IterationStore` 接口

### 3.1 Save 入参/返回语义 — 接受（小修）

**发现**：原 design `Save(ctx, *Checkpoint) error` + "store 在 zero-value 时分配 Sequence/ID" 是合理的，但"caller-chosen values are honored (used when restoring a snapshot from another store)"这一句是为未来留口子，本期没有任何调用方使用 caller-supplied sequence；且打开"caller 可指定 sequence"会把"sequence per session 严格单调"这个不变量从"store 维护"降级为"caller + store 共谋"。

**建议**：
- 文档收紧为"store always assigns Sequence and ID; caller-supplied values are ignored"。删除 cross-store snapshot 迁移的暗示。
- 真要迁移，加专门的 `Import(ctx, *Checkpoint) error` 方法即可，不在本期范围。

### 3.2 删除 `Delete` — 拒绝

**发现**：原 design 提供 `Delete(ctx, sessionID) error` 清空整 session。乍看没消费者，但：
- 测试里需要清理（conformance test 需要它做 fixture teardown）。
- session 删除时 cascade（即便本期不接 session.Delete 钩子，留个 Delete 让 ops 手动清理是合理的）。

保留。注释明确"删除整 session 的所有 checkpoint，幂等"。

### 3.3 Load 的双参 sessionID + id — 接受

合理，保留。文件后端可以由 sessionID 直接定位目录，不必扫 root。

---

## 4. TaskAgent 集成

### 4.1 Final checkpoint 是否需要写入 — 接受（保留写入，但简化 Resume 分支）

**发现**：原 design "Final=true 时 Resume 不调 LLM，直接组装 RunResponse 返回"会引入一条 Resume 专属的奇怪路径：
- 它复制了 `finalizeRun` 的一半（output guards 不能再跑，因为初次 finalize 已经跑过；event 要不要 dispatch 也成问题 —— 重复 AgentEnd？）。
- 调用方拿到的 RunResponse 是"重新装配的"，duration 字段语义模糊。

**讨论**：两个方案：
- **A**: 保留 Final 写入；Resume 在 Final cp 上返回明确的 `ErrAlreadyFinal`，由调用方决定怎么展示。
- **B**: 不写 Final checkpoint —— 终态信息只通过 hook event 或 RunResponse 暴露，恢复路径上"找不到 cp" 与"已 Final" 都视为"无事可做"。

**建议**：采用 **A**。理由：
- Final cp 仍然有用（List 给产品视图的最后一行，写到磁盘后做审计、看 stop_reason）。
- Resume 上"已 Final" 是错误而非成功路径 —— 调用方需要显式 handle，不该静默返回空 RunResponse。
- 错误名 `ErrAlreadyFinal`（包装 ErrInvalidArgument 或独立 sentinel 都可，倾向独立），含义直白。

收益：删掉 Resume 里 "Final=true → 重组 RunResponse" 那 ~30 行；测试矩阵简化为"Final cp → Resume 返 err"。

### 4.2 Resume 不接收 `RunOptions` — 保留

合理。文档化"v1 用 agent 默认值"，下一期再加。

### 4.3 `ResumeOption` placeholder — 拒绝（删除）

**发现**：`type ResumeOption func(*resumeConfig); type resumeConfig struct{}` 是纯占位符，本期没有任何参数。Go 的"variadic 增加 option"是非破坏性的 —— 等真有需求时加签名 `Resume(ctx, sid, opts...ResumeOption)` 不破坏既有调用。

**建议**：v1 就 `Resume(ctx context.Context, sessionID string) (*schema.RunResponse, error)`。删除 ResumeOption / resumeConfig。新增参数留到下一期。

### 4.4 写入失败 warn-and-drop — 接受（保留），但加上 hook 反向通道

**发现**：`saveCheckpoint` 失败时 slog.Warn + 主路径继续是对的（与 SessionHook 一致）。但调用方目前**无法从外部观测到这次写入失败**：tracelog 看不到，metric 看不到。

**建议**：
- 失败时仍然 dispatch 一个 `EventCheckpointWritten` 的 *failure* 变体？过度复杂。
- 简化方案：失败时单独 dispatch `schema.NewEvent(schema.EventCheckpointWritten, ...)` 但 payload 中 `Final=false / Sequence=0 / Error="..."` —— 也很丑。
- **最简方案**：保留 warn-and-drop 不动；在 hook payload 里**不**派发 failure 事件；测试方通过"hook 计次 vs. 迭代计次"对比即可发现异常。这个方案是 design 原文的方案，保留。

不改。但在新版 design 里把"如何观测失败"明确写到运维注意事项一节。

### 4.5 失败语义"opt-in 严格模式" — 拒绝（YAGNI）

**讨论**：是否提供 `WithCheckpointStrictMode(true)` —— 写入失败即终止 Run？

**结论**：拒绝。本期没有需求，也没有可信的使用场景（如果调用方真的"不能丢 checkpoint"，更合理的方式是让 store 实现自身做 retry/replication，而不是把责任推到 agent loop）。等真出现需求再加。

### 4.6 Resume 跳过 input guards — 接受，并补 **bypass output guards on Final**（已经在 4.1 折叠）

设计里"Resume bypasses input guards" 合理（input 已经在初次 Run 中跑过，重新跑可能误伤）。Tool result guards 自然在新执行的 tool call 上跑（design 已说），保留。

### 4.7 `cloneMessages` 浅拷贝注释 — 接受（小修）

**发现**：原 design 注释"`aimodel.Message.Content` 持有的内部 parts 在生成后不再 mutation"，但 `Content` 在 `markPromptCacheBreakpoints` 处确实被 in-place 修改（`messages[i].CacheBreakpoint = true`）—— 这是字段而不是 Content parts，但读者会担心。

**建议**：注释精确化："`Content`/`ToolCalls` 在 ReAct 循环中只追加，不修改既有元素；`CacheBreakpoint` 在 system 消息上 in-place 设置，但 system 消息是切片首位，共用语义下不会出现 cp 之间互相污染（同一个 messages 切片 backing array 在 Resume 后会被新分配的切片覆盖）"。代码不变，只改注释。

---

## 5. 文件后端

### 5.1 Sequence 分配读 ReadDir — 接受（注释强化）

**发现**：原 design "lock 内 ReadDir 取 max + 1"实现简单；性能 O(N) 在 1000 cp 下完全可接受。但有一个风险点：**临时文件 `.tmp` 在并发恢复期间可能误算入 max**。

**建议**：
- 写入流程必须先确定 sequence、构造文件名（`000001-<id>.json`），然后用"该最终路径 + .tmp"作为临时名（与 session/filestore.go 的 `writeJSONAtomic` 完全一致）。这保证 ReadDir 时遇到 `*.json.tmp` 不会被算进 sequence 计算 —— 加一行过滤即可。
- 在 design 里明示：sequence 解析必须"只看 `.json` 后缀，跳过 `.tmp`"。

### 5.2 文件名前缀宽度 — 接受

`%06d` 6 位零填充够用（一个 session 撑到 100 万 iter 已经远超合理上限）。保留。

### 5.3 SQLite 后端的演进 — 接受（无代码改动）

**发现**：要求"file 布局未来不阻塞 SQLite 后端"。

**建议**：
- 接口 `IterationStore` 已经是抽象层，FileStore / MapStore / SQLiteStore 平行实现，零阻塞。
- 在 design 里记一行"SQLite 后端可以以同样的接口实现，建议 schema：`(session_id TEXT, sequence INT, id TEXT, payload JSON, created_at TIMESTAMP, PRIMARY KEY (session_id, sequence))`，不在本期范围"。

---

## 6. Schema

### 6.1 `EventCheckpointWritten` payload — 接受（小调）

**发现**：原 payload 含 `MessagesCount` —— 让 tracelog 读者一眼看到"截至本轮 LLM 看到几条消息"。但缺 `SessionMsgCount`（恢复时关键的 ContextBuilder offset）；不过 hook 消费者通常不需要这个值。

**建议**：保留原字段集合不动。`SessionMsgCount` 是 cp 内部状态、不在事件 payload 中。

### 6.2 Final + StopReason 双字段 — 接受

合理。`Final == false` 时 StopReason 为零值；`Final == true` 时 StopReason 一定非空。在 `Checkpoint.Final` 字段注释里明确这个不变量。

---

## 7. 测试

### 7.1 共用契约测试 — 接受

按 `vage/session/store_conformance_test.go` 的形态做即可。新增 case：
- Save 后 `cp.Sequence > 0 && cp.ID != ""`（store-assigned 字段确实回填）。
- Save 不同 sessionID 的 sequence 各自从 1 开始（per-session 单调，跨 session 不共享）。
- Load(sessionA, idFromSessionB) 返回 ErrCheckpointNotFound（双参校验）。

### 7.2 fake completer 测试 — 接受

按原 design 的 5 个 case 落地。增加：
- "Resume on Final cp returns ErrAlreadyFinal" — 与 §4.1 决策对齐。

### 7.3 集成测试 — 接受

无需改动。

---

## 8. 实施顺序

原 design 第 10 节合理，无需调整。

---

## 9. 改动汇总

| 类别 | 接受 | 拒绝 |
|---|---|---|
| 命名 | `IterationStore` 替代 `CheckpointStore` | 保留 `CheckpointStore` 同名 |
| 字段 | 删除 `RequestMessages` / `Metadata`；新增 `Estimated`；`CheckpointMeta` 用 `Usage` 取代 `TotalTokens` | 保留 `RequestMessages` |
| 接口 | Save 文档收紧（store-assigned only） | 引入 `Import` 方法 |
| Resume | Final cp 返回 `ErrAlreadyFinal`；删除 `ResumeOption` placeholder | Final 时重组 RunResponse 返回；保留 ResumeOption 占位 |
| FileStore | sequence 计算跳过 `.tmp` | — |
| Schema | 维持现 payload 字段集 | — |
| 测试 | 增加 cross-session sequence + ErrAlreadyFinal | — |

合并后实现增量：基本与原 design 持平（删字段省的代码 ≈ 加 Estimated + 错误处理新增的代码）；删掉 Resume 的 Final 分支净减 ~30 行。仍然单 dev pass 可落地。
