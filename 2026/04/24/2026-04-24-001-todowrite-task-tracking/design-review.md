# P1-9 · TodoWrite 设计评审

> 评审对象：`design-raw.md`（原 `design.md`）。
> 评审角色：架构/代码评审员（MetalClaw improver 阶段）。
> 准则：简单胜过复杂；不引入无收益的抽象；对已有代码的侵入点最小化。

以下每一项明确标注 **ACCEPT / REJECT / ADJUST**，并给出一行理由。被 ACCEPT / ADJUST 的条目会反映在新的 `design.md`。

---

## 1. `schema.WithEmitter` 抽象的必要性

**结论：ACCEPT（保留 ctx 注入 Emitter 的方案）。**

读过 `vage/agent/taskagent/task.go` 与 `vage/schema/stream.go`：

- `RunStream` 的事件到 CLI 的唯一通路是 `schema.NewRunStream` 内部的 `send` 闭包 → `rs.ch` → `stream.Recv()`。
- CLI 端 `handleStreamEvent` 从 `stream.Recv()` 读事件（不是从 `hook.Manager`）。
- `hook.Manager.Dispatch` 是侧信道（JSONL trace、budget 中间件），它**不会**把事件塞进 `RunStream` 的 channel。
- 因此 "hook-only" 路径无法让 CLI 看到 `todo_update`。原设计论证成立。

备选方案 "ToolResult 侧信道字段"（在 `schema.ToolResult` 加 `Events []Event`）被 REJECT：会污染跨所有 tool 的核心返回类型，且每个消费者都要学新的"从 result 里再次分发事件"规则；不值当。

> 对新设计的影响：保留 `vage/schema/context.go` + `WithSessionID` + `WithEmitter`。

---

## 2. `taskagent/task.go` 的 ctx 注入点

**结论：ADJUST — 注入点下移到单一 choke point。**

原设计说"在 `callTools` 的并行/串行两条路径都要注入"。实际代码中注入的唯一合理位置是 `executeToolBatch` 的**开头**——两条路径（`n==1` 串行、`n>1` 并行）都走这个函数，并且都从 `ctx` 派生 goroutine。注入一次 `ctx = toolCallCtx(ctx, rc, eventSink)`，下游两条路径天然继承。

另外：Run（非 stream）路径的 `callTools`（`task.go:924` 附近）已构造了 `sink := func(ev) { a.dispatch(ctx, ev); return nil }`，同样走 `executeToolBatch`。只要 `executeToolBatch` 顶部统一注入 ctx，两条入口都被覆盖，不需要两处 patch。

进一步：`toolCallCtx` 的实现可以完全省掉 `SessionID/Timestamp` 补齐逻辑——`schema.NewEvent` 已经做了；handler 调 `NewEvent` 时传 sessionID + 当时时间戳即可。删掉 15 行冗余。

> 对新设计的影响：把"两处注入"改为 **只在 `executeToolBatch` 顶部注入一次**；删除冗余 sessionID/timestamp 补齐。

---

## 3. `schema.TodoItem` 是否与 `todo.Item` 重复

**结论：ACCEPT（保留重复，但说明原因）。**

`vage/schema` 处于依赖底层，不能反向 import `vage/tool/todo`（否则会形成 schema → tool → schema 循环）。`todo.Item` 与 `schema.TodoItem` 形状几乎一致，但只是 6 个字段的投影函数 `toEventData(snap)`——代码量 < 20 行，换来了 layering 干净。不改。

> 对新设计的影响：无。

---

## 4. "store 进程单例 per-session" 在多 Agent（coder + researcher）同会话的语义

**结论：ACCEPT（进程单例 + sessionID 键是正确选择），但增加一条注解。**

需求 US-1 预期 coder 在多步任务中交给 reviewer 看代码时，reviewer 能看到同一张 todo 列表；dispatcher plan 模式甚至会在一次运行里跨多个子 agent。在这个语义下：

- **共享**是功能需求，不是 bug。
- 每个 Agent 独立 store → 列表会分裂、HTTP SSE 出现重复 version 不单调——不可接受。

**但要显式写下两条不变式**，developer 阶段需要测试覆盖：
1. 多 Agent 同 sessionID 写入时，version 单调（store mutex 已保证）。
2. sessionID 为空时直接 Error，绝不走 "global/default" fallback——否则串台风险。

> 对新设计的影响：§10 "关键权衡"里把"进程单例 per Session"的描述补上"版本号单调 + sessionID 必需"两条不变式。

---

## 5. "最多 1 个 in_progress" 与 "100 条硬上限" 的放置位置

**结论：ACCEPT，**但 100 条上限稍微宽松一点，**改成 ADJUST**：

- "≤ 1 in_progress" 放在 `Store.Apply` 内部：正确——唯一真相源，不信任 prompt。
- 100 条：这个上限主要是防 prompt 轰炸和 CLI 渲染雪崩。阅读 Claude Code 观察：正常任务 3~8 条，极端复杂迁移 15~30 条。100 已经非常松。但"不支持 10k 条"是显然的。结论：**保留 100**，代码中用命名常量 `maxItemsPerList = 100`，方便后续调参。

> 对新设计的影响：§3.2 补充 `maxItemsPerList` 常量命名。

---

## 6. Prompt 引导段的精简

**结论：ADJUST — 维持"三 Agent 都追加"，但**把文案从 "maintain a live plan for multi-step work..." 压缩到 **≤ 2 行**，reviewer 的段落换成一行提示（"If the review spans 3+ files, use todo_write to track progress."），符合 reviewer 本身的工作特点。

理由：`researcher` 做多文件阅读 + 综述的任务仍能受益；`reviewer` 偶尔会跨文件走查，但不强需要——一行软提示就够了。避免 coder prompt 中堆砌重复。

> 对新设计的影响：§6 prompt 段落改为三 Agent 差异化文案。

---

## 7. `researcher` profile 是否挂 `todo_write`

**结论：ACCEPT（挂）。**

`researcher` 是 read-only，但"多步研究"同样是 >3 步任务的典型场景（比如"梳理 A/B/C 三个模块的调用关系并输出 markdown"）。todo_write 本身不是"写工作空间"——它写的是会话级内存状态，与 read-only 语义不冲突。

tool.ToolDef.ReadOnly 的语义是"不修改工作区文件系统 / 外部副作用"，我们明确标 `ReadOnly: true` 更准确（`todo_write` 不碰磁盘，也不影响工作区），让 researcher 的"read-only claim"保持一致。

> 对新设计的影响：§5 把 `ToolDef.ReadOnly` 从 `false` 改成 **`true`**；并在 "关键权衡" 新增一条说明。

---

## 8. YAGNI 审计

### 8.1 `TodoItem.ID` 是否必需

**结论：ACCEPT（保留 ID，服务器分配）。**

- 不带 ID，客户端做 diff 只能靠位置 + content 猜，脆弱。
- LLM 输入里 `id` 可选，匹配到现有 id 复用；没有就 server 分配。新写 LOC 很少（<10 行）。
- 未来扩展到 CRUD/patch 语义、或要在 CLI 做 "spotlight focus on id=todo_3" 的交互，不用 schema breaking。
- 收益/成本比值得。

### 8.2 `Snapshot.Version` 是否必需

**结论：ACCEPT（保留）。**

HTTP SSE 不保证"绝对不重复"——代理/重连场景下客户端会看到同一 version 的 snapshot 两次，用 version 做幂等去重是行业习惯。内存里维护 int64 自增几乎零成本。

### 8.3 Handler 里 "formatForLLM" 的 ASCII 文案

**结论：ADJUST — 化简。**

原设计里给 LLM 返回 4 行文本 + 前缀 `Todo list updated (v3, 4 items):`。LLM 本来就看得到它自己刚提交的 items；重述不划算。改成：

```
ok (v3, 4 items)
```

节省 tokens，同样能让 LLM 确认写入成功。CLI 渲染不受影响（它拿的是 `TodoUpdateData`，不是 tool_result text）。

### 8.4 `configs.Agents.TodoEnabled` 配置开关

**结论：REJECT（删除这个开关，但保留 env 兜底）。**

- 默认开启、和"注册 6 个内置工具"是一起的，从来没人想"单独关掉 todo"而留着其他 5 工具。YAGNI。
- 但为了方便排障（debug `todo_write` 被错误 prompt 调用的场景），保留一个**单一**的隐式 env 开关 `VV_DISABLE_TODO`（默认 false）——代码量 4 行，不进 yaml schema、不进默认配置文件。

> 对新设计的影响：删 `AgentsConfig.TodoEnabled`、删 `EffectiveTodoEnabled`；只在 `setup.Init` 里做一次 `os.Getenv("VV_DISABLE_TODO")` 判断。

### 8.5 `Store.Clear` 暴露

**结论：ACCEPT（保留，标注 "P1-9 不调用"）。**

P2-14 会话检查点可能会用到；保留不会多花时间，删了将来补更贵。

---

## 9. 事件双重打印防御

**结论：ACCEPT（保留原设计），但** 补充一条测试：当 `tool_result` 的 `ToolName == "todo_write"` 时，`renderToolCallResult` 必须返回空字符串——测试文件 `vv/cli/render_test.go` 里加一条断言，防止未来有人改渲染代码时丢掉这个规则。

> 对新设计的影响：§12 测试计划补一条断言。

---

## 10. 其他杂项

### 10.1 "空列表即允许清空" 的语义

**结论：ACCEPT**，但补一条说明："清空"后的下一次调用 version 仍然 +1（不是 reset 到 0），确保 HTTP 客户端 version 单调可做幂等键。

### 10.2 Handler 对 `items=nil` vs `items=[]` 的处理

**结论：ADJUST** — 统一视为清空。JSON 里 `"todos": null` 也走 `Apply(sessionID, nil)`（合法）。这样 LLM 如果 "I'll clear the list" 之后写 `{todos: []}`，行为与预期一致。增加一条单测即可。

### 10.3 Stream 路径下 Emitter 错误处理

原设计 `_ = sink(e)` —— 忽略错误（注释 "已有的 sink 语义：错误不阻塞主流程"）。

**结论：ACCEPT**。`sink` 返回错误仅在 `RunStream` 被关闭时出现，这时整个 run 也在终止，再处理 todo_update 的发送错误没有意义。但**删掉多余 sessionID/timestamp 补齐**（见第 2 条），注入闭包缩到 3 行。

### 10.4 `vv/tools/tools.go` 三个 Register 入口的改动范围

**结论：ADJUST** — 不在 `tools.go` 里直接 `todo.Register(reg, store)`，而是在 `setup.Init` 里对已经构造好的三个 registry 做一次 `todo.Register`。理由：

- `vv/tools/` 当前是"把 vage 内置工具挂进来"的薄封装，不应该知道 store 实例。
- `setup.Init` 本来就是"把 store / registry / agents 串起来"的地方，职责合适。
- 避免给 `Register*` 一堆参数变体。

> 对新设计的影响：§2 包布局里 `vv/tools/tools.go` 标注 "无改动"；改动挪到 `vv/setup/setup.go`。

---

## 11. 未被修改的设计点（ACCEPT as-is）

- 总览图、数据模型、JSON Schema、三态枚举。
- Event 类型命名 `EventTodoUpdate = "todo_update"`。
- CLI 渲染颜色 / marker 方案（`[x] [>] [ ]`）。
- HTTP SSE 免改 service 层。
- 测试计划骨架（补一条 tool_result 不双重打印的断言即可）。
- LOC 估算量级（~850）—— 小幅下调约 40 LOC 因为删了 `TodoEnabled` 开关和两处冗余注入补齐。

---

## 12. 摘要：本次评审引入的净改动

| # | 评审项 | 动作 | 实际落到新 design.md 的位置 |
|---|---|---|---|
| 1 | `schema.WithEmitter` 保留 | ACCEPT | §4 不变 |
| 2 | ctx 注入点收敛到 `executeToolBatch` 顶部 | ADJUST | §4.2 |
| 3 | `TodoItem / Item` 保持分离 | ACCEPT | §3 原文微调说明 |
| 4 | Store 不变式显式化 | ADJUST | §10、§11 |
| 5 | 100 条硬上限 → 命名常量 | ADJUST | §3.2 |
| 6 | Prompt 引导段差异化 | ADJUST | §6 末段 |
| 7 | `ToolDef.ReadOnly = true` | ADJUST | §5 |
| 8.1/.2 | 保留 id、version | ACCEPT | §3 原文 |
| 8.3 | formatForLLM 化简 | ADJUST | §5 |
| 8.4 | 删 TodoEnabled 配置，保留 env | ADJUST | §2、§9 |
| 8.5 | `Store.Clear` 保留 | ACCEPT | §3 原文 |
| 9 | tool_result 双重打印断言 | ADJUST | §12.4 |
| 10.2 | `items=null` 合法清空 | ADJUST | §11 |
| 10.4 | 注册逻辑挪到 setup | ADJUST | §2、§9.2 |

评审后预估实际 LOC 降到 **~830**，比原 ~875 少约 40 行，主要靠删 TodoEnabled 配置开关、合并注入点、formatForLLM 化简得到。

没有发现需要回到"调研或需求"阶段的结构性缺陷，可以直接进入 developer 阶段。
