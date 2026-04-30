# 需求：迭代级 Checkpoint

## 1. 背景与目标

### 1.1 来源
`doc/design/session-context-solution.md` §4.5 提出在 vage 框架内补齐"迭代级 Checkpoint"能力，是该路线图上"持久长任务"主线的下一站（Session 实体 → Context Builder → vv wiring → Plan Workspace 之后）。

### 1.2 现状
| 已有 | 状态 |
|---|---|
| `vage/orchestrate/checkpoint.go` | DAG 级别 checkpoint：以 `(dagID, nodeID)` 为键保存 `RunResponse`，支持 in-memory；orchestrate 引擎在节点完成后写入，在重启时加载 |
| `vage/session/` 包 | 一等 Session 实体 + 三段 Store + 文件后端；**未提供**任何 checkpoint 接口 |
| TaskAgent ReAct 循环 | 每轮迭代：LLM 调用 → tool batch → append messages → 下一轮；**没有任何中间持久化**——崩溃即丢 |

### 1.3 目标
让任意一次 `TaskAgent.Run / RunStream` 在每完成一次 ReAct 迭代后，把"截至本轮的累积 messages + 关键运行时状态"写入到一个新的 checkpoint，支撑：

1. **进程重启 / 显式中断后恢复**：拿 `(session_id, checkpoint_id)`（缺省 = latest）即可在新进程内重续。
2. **只观察、不恢复**：列出某 session 的所有 checkpoint，作为审计与"任务进度"产品视图。
3. **消除"崩溃即丢"语义**：哪怕第 N 轮工具调用失败，也能回到第 N-1 轮重试，而不必从头跑。

### 1.4 非目标（明示 out-of-scope）
- ❌ **Fork**（从历史 checkpoint 分叉出独立任务）：实现复杂、产品形态未稳，留作后续。
- ❌ **Interrupt / HITL（人在回路审批）**：与 fork 同批，需要 stream pause / resume 配合。
- ❌ **DAG 级别下沉**：orchestrate 已有自己的 checkpoint store，本次不动。
- ❌ **跨进程文件锁、SQLite/Postgres 后端**：复用 session 包的现状（per-process 文件锁 + 单写者），出现需求再加。
- ❌ **vv 端 wiring**（CLI `--resume`、HTTP `/v1/sessions/{id}/resume`）：本期只交付框架层 + 集成测试；vv wiring 留下一迭代。
- ❌ **prompt cache 的 checkpoint 续接**：恢复时从 messages 重建即可，cache breakpoint 标记重新打。
- ❌ **memory 系统的回放**：working memory 是单次 Run 的临时态，恢复后由 ContextBuilder 重新装配；session memory 的现有写入路径不变。

## 2. 用户故事与验收标准

### US-1：长任务进程重启后续行
**作为** 使用 vage 跑一个 30 分钟级别 ReAct 任务的开发者，  
**当** 程序在第 5 轮迭代中途因为外部原因（OOM / SIGTERM / panic）退出后，  
**我希望** 重启程序时拿到 session_id 即可从已经完成的最后一轮（第 4 轮）状态续行，而不是从第 1 轮重新开始。

**AC-1.1** 给定一个完成了 N（≥1）轮迭代的 session，调用 `TaskAgent.Resume(ctx, session_id)` 后：
- 返回的 `*RunResponse` 中 messages 顺序与"完整跑完一次 ReAct 直到 stop"一致。
- 不重复执行已完成迭代里的 tool call。
- 后续迭代的 LLM 调用看到的初始 messages 等于"系统提示 + session 史 + 截至最后一个 checkpoint 的本次 Run 累积 messages"。

**AC-1.2** 给定一个零迭代的 session（仅创建未跑），调用 Resume 等价于普通 `Run`。

**AC-1.3** 给定一个找不到 checkpoint 的 session_id（不存在 / 已删），Resume 返回明确错误（`ErrCheckpointNotFound` 包装），不静默退化。

### US-2：迭代级 checkpoint 自动写入
**作为** TaskAgent 的集成方，  
**当** 我配置了 checkpoint store，  
**我希望** 每次 ReAct 迭代结束（一次 LLM 调用 + 一批 tool call 完成、并把 tool 结果 append 进 messages）就自动产生一个 checkpoint，**不需要** 我手动调任何 API。

**AC-2.1** 跑一次包含 K 轮迭代的 Run（最终 finish_reason 为 stop）后，store 中针对该 session 应该有 **K 个 checkpoint**（每轮一个），按顺序递增 sequence。

**AC-2.2** Run 在第 i 轮中途（LLM 调用失败 / context 超时）异常返回时，已经成功完成的 i-1 个 checkpoint 仍然在 store 中可读。

**AC-2.3** 未配置 checkpoint store 时，Run 行为与配置前**逐字节一致**（无副作用、无 panic）。

### US-3：观察 checkpoint 链
**作为** 调试 long-running 任务的工程师，  
**当** 怀疑"为什么走到了第 N 轮"，  
**我希望** 列出某 session 的所有 checkpoint 元数据（sequence、created_at、iteration index、累计 token），快速对照 trace。

**AC-3.1** 提供 `CheckpointStore.List(ctx, session_id)` 返回按 sequence 升序的元数据切片（不含完整 messages，避免 O(N×M)）。

**AC-3.2** 提供 `CheckpointStore.Load(ctx, session_id, checkpoint_id)` 加载完整 checkpoint（含 messages）。`""` 作为 checkpoint_id 表示加载最新。

**AC-3.3** Hook 体系收到 `EventCheckpointWritten` 事件，payload 含 sequence / iteration / total_tokens / messages_count，便于 vv tracelog 落盘。

### US-4：可插拔后端
**作为** 部署 vage 的运维，  
**当** 我希望迭代级 checkpoint 与 Session events 落到同一个目录便于备份，  
**我希望** 内置一个 `FileCheckpointStore`，目录布局贴近 `FileSessionStore`；同时保留 `MapCheckpointStore` 用于测试。

**AC-4.1** `FileCheckpointStore` 与 `FileSessionStore` 共用 `<root>/<session_id>/` 目录，新增子目录 `checkpoints/`；checkpoint 文件名为 `<sequence>-<id>.json`，便于 `ls` 直接看到顺序。

**AC-4.2** `MapCheckpointStore` 单 mutex；写入返回的 checkpoint 是 deep copy，避免外部 mutation 污染内部状态。

**AC-4.3** 两个内置实现共用一份黑盒契约测试（与 `vage/session/store_conformance_test.go` 同 pattern）。

## 3. 涉及的角色 / 模型 / 流程 / 应用

### 3.1 角色（roles）
- 框架使用方（业务工程师）：通过 TaskAgent option 注入 store；通过 `Run` / `Resume` 触发。
- 运维：选择内存 / 文件后端，配置目录。

### 3.2 模型（vage 内的实体）
- 新增 `vage/checkpoint/` 包：
  - `Checkpoint`（元数据 + 载荷）
  - `CheckpointMeta`（精简元数据，List 返回值）
  - `CheckpointStore`（接口）
  - `MapCheckpointStore` / `FileCheckpointStore`（实现）
  - `ErrCheckpointNotFound` / `ErrInvalidArgument`
- 新增 schema event：`EventCheckpointWritten` + `CheckpointWrittenData`
- TaskAgent 新增 option `WithCheckpointStore` + 公开方法 `Resume`

### 3.3 流程
- **写**：`TaskAgent.Run / RunStream` 内部，每轮迭代尾部（tool batch 完成、messages 已 append）→ 构造 Checkpoint → 调 Store.Save → 派发 hook 事件。
- **读**：`TaskAgent.Resume(ctx, session_id, options...)` → `Store.Load` 取 latest（或指定 id）→ 用 checkpoint 中的 messages 重建 runContext → 进入 ReAct 循环（从下一轮开始）。

### 3.4 应用层
本次仅交付框架层；vv（CLI / HTTP）的 wiring 不在本期范围。

## 4. 假设与权衡

### 4.1 关键假设
- **A1**：iteration 是天然的 checkpoint 粒度。子粒度（每个 tool call 后写一次）会让 store I/O 暴涨，超出最小可用价值；恢复时再跑一遍同一轮的剩余工具，对绝大多数无副作用工具可接受。  
  **风险**：对**有副作用工具**（写文件、调外部 API）会重复执行——这是 LangGraph、Temporal 等业界方案在"无幂等键"前提下的普遍取舍，由调用方在工具层自行做幂等。
- **A2**：checkpoint 的 messages 是"完整快照"而不是 diff。messages 体量在 ReAct 任务里通常 O(数十到数百 KiB)，全量快照实现简单、读路径 O(1)；diff 模式优化幅度有限但实现复杂度跳跃。
- **A3**：恢复后从下一轮的"LLM 调用"开始，而不是"从最后一个工具调用结果"。已完成的 tool batch 已经 append 到 messages 里，下一轮 LLM 的输入和"未崩溃情况下 LLM 看到的"完全一致。
- **A4**：恢复**不**触发 input/output guards 重跑（避免对已审过的内容重复 block）；保留 tool result guards 的实时审查（恢复后第一个工具调用仍被审）。

### 4.2 多种可行解的取舍
| 方案 | 选择 | 理由 |
|---|---|---|
| 直接复用 `orchestrate/CheckpointStore` | ❌ | 接口签名以 `(dagID, nodeID)` 为键、值类型是 `RunResponse`，与 session/iteration 的语义不匹配；强行复用要么破坏 orchestrate 现有契约，要么在该接口外加一层 ad-hoc kv，得不偿失 |
| 把 checkpoint 数据挂到 `SessionStateStore` 的特定 key 里 | ❌ | List 困难（要扫所有 key）；并发写依赖 SetState 的覆盖语义不符合 append-only |
| 新增独立的 `CheckpointStore` 接口 + 独立后端 | ✅ | 语义清晰、与 Session/Memory 解耦、未来加 SQLite 后端无需绑死 |
| 全量 messages vs. diff（patch） | 全量 | A2 |
| sequence 用 unix-nanos 还是 store 内自增 | store 自增 | nanos 跨进程不严格唯一；自增由 store 持有锁内分配 |
| checkpoint id 用 sequence 字符串 vs. 独立 hex | 独立 hex 拼 sequence 前缀 | 前缀让目录 `ls` 自动按时间序，hex 后缀避免顺序冲突也方便错误信息中识别 |

## 5. 业界优秀实践对照（用于评估方案完整性）

| 框架 | 核心机制 | 我们对齐的做法 | 不照搬的部分 |
|---|---|---|---|
| **LangGraph Checkpointer** | 每个 graph step 写 checkpoint；`thread_id` + `checkpoint_id` 寻址；支持 SQLite/Postgres/Redis；`time travel` 由历史链支撑 | 每轮 iter 写一次；以 `(session_id, checkpoint_id)` 寻址 | time travel / fork 留作后续，避免一次性吃掉过大设计 |
| **Temporal Durable Execution** | 工作流代码 = 状态机；活动结果 checkpoint；崩溃后从最近成功 step 重放 | iteration 即"活动"；恢复时回放已 append 的 messages，跳过已完成 iter | 不引入工作流 DSL —— TaskAgent 的 ReAct 循环天然就是状态机 |
| **OpenAI Assistants Runs** | 服务端持久化 thread；`previous_response_id` 接力 | `session_id + latest checkpoint` 是同样的接力链 | 不是服务端模式，纯本地存储 |
| **Anthropic Claude Agent SDK** | `Auto-Compact` + 文件式 memory；状态由用户自己 file 管理 | 通过 SessionEvents + 本期 Checkpoint 双轨：events 是审计用，checkpoint 是恢复用 | 不引入 LLM 主动 page-in 工具 |
| **Devin / SWE-Agent / OpenDevin** | plan.md 落盘是任务进度的"单一事实"；进程崩溃靠 plan + 工作目录恢复 | 与 §4.4 已有的 Plan Workspace 互补：plan 是任务**意图**，checkpoint 是 ReAct **执行进度** | 我们已有 Plan Workspace，本期只补 checkpoint |

**结论**：本期 MVP 的最小方案与业界主流（LangGraph / Temporal）一致，差异仅在 fork / time travel / HITL 留待后续——这正是设计文档 §4.5 显式列出的范围；本次方案的边界判定是合理的。

## 6. 验证策略概要（细节交给 designer / tester）

- 单元：MapStore / FileStore 的契约测试 + checkpoint 元数据序列化往返。
- TaskAgent：`Run` 写入 N 个 checkpoint 后调 `Resume` 应当从第 N+1 轮 LLM 调用开始，messages 与不中断时一致。
- 集成（integrations/）：用 fake ChatCompleter 模拟一次 3 轮迭代，第二轮中途返回错误 → 重启 → Resume → 验证 messages 对齐。

## 7. 已发现的文档不一致 / 待协调

- `vage/.doc/architecture.md`、`vage/.doc/orchestrate.md` 都已经描述 orchestrate 级 checkpoint，本期需要在 `vage/.doc/` 下新增 `checkpoint.md` 描述 iteration 级别，并在 architecture / index 里加交叉引用，避免读者误以为框架只有 orchestrate 一种 checkpoint。
- `doc/design/session-context-solution.md` §4.5 的"interrupt / fork"等内容**本期不实现**，但要在 §8 差距汇总把"迭代级 Checkpoint" 行从❌改为部分✅，并明确尚未交付的是 fork / interrupt。
