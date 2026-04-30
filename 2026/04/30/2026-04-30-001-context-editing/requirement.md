# Context Editing — 需求

## 1. 背景与目标

### 1.1 业界标杆

`doc/design/session-context-solution.md` 第 4.3 节明确提出 P6 模式 ——
"为 `largemodel` 中间件链补一层 **`ContextEditor`**：在调用 LLM 前对 messages 做最后一道'擦除/折叠'"。

参考 Anthropic Claude Code 的 *context editing* 与 *Effective context engineering* 博客，业界对该模式的普遍约束如下：

1. **历史保持完整可追溯**。Anthropic 的方案是把老的 `tool_result` 替换为占位符，但对应的 `tool_use` 仍保留——LLM 从未"看见过 tool_use_id 但拿不到结果"。
2. **只擦"已消费完"的中间产物**，不擦最近若干轮——"近端原文"是 LLM 续推的关键。
3. **可观察、可关闭**——擦除是有损的，必须留审计 trail，并允许在故障排查或某些工具流程中关闭。
4. **不影响 prompt 缓存**——擦除位置应当在缓存断点之后，不要让一条变化的 tool_result 把整段缓存击穿（这点取决于 vage 当前 cache breakpoint 的位置；要么擦除发生在 breakpoint 之后，要么擦除整段都不缓存）。
5. **协议无关**。兼容 OpenAI（`role:tool`，`tool_call_id` 显式）与 Anthropic（assistant `tool_use` + user `tool_result` 配对）两种结构。vage 把 OpenAI shape 作为规范化形式（`aimodel.Message.Role == RoleTool` + `ToolCallID`），Anthropic translator 在 `aimodel/anthropic.go` 内做翻译——所以**编辑动作直接作用在规范化 `aimodel.Message` 上**即可，不必触碰协议层。

### 1.2 vage 现状与差距

- 已具备：三层 memory（P1）、五种 compressor（P2）、Session 实体化（P1 加固）、ContextBuilder + Source（P2/P3 雏形）、Plan Workspace（P4 子集）、迭代级 Checkpoint（P8 雏形）。
- 现状的**痛点**：长任务里 ReAct 循环跑十几轮后，每一轮都把过去所有 tool_result 原文重新送回 LLM。一个文件读取 / 一段 bash 输出动辄 5–50 KiB，10 轮就 0.5 MiB tokens，**绝大多数都是 LLM 已经消费完、再送回去的纯成本**。
- 现有 ContextBuilder 处理的是 **session 级** 历史，不处理 **本次 Run 内部** 的 ReAct 中间消息——后者就是 Context Editing 的目标。
- TaskAgent 已经在 `largemodel` 注入中间件链；这是把 ContextEditor 装上去的天然位置，且与协议无关、与 ContextBuilder 解耦。

### 1.3 目标

新增 `largemodel.ContextEditorMiddleware`，在 `ChatCompletion` / `ChatCompletionStream` 调用前对 `req.Messages` 做最后一道"擦除/折叠"，**不修改原始消息切片**（caller 持有的 ReAct messages 必须保持完整可重放），**只对送给 LLM 的副本生效**。

---

## 2. 用户故事 & 验收标准

### US-1（开发者使用）：长任务里把老的工具结果折叠掉

**As** vage / vv 的下游接入方，
**I want** 在 LLM 中间件链里挂上一层 ContextEditor，让 ReAct 循环里跑了 N 轮之后，老的 `tool_result` 自动折叠成短占位符，
**so that** 长任务的 prompt 不会随着轮次线性膨胀。

**验收标准**

- AC-1.1 当 messages 里 `Role == RoleTool` 的消息 ≥ 阈值（默认保留近 K 个）时，**靠前的 tool_result 内容被替换为占位符**（如 `[context_edited: tool_result <tool_call_id> elided, N bytes]`），但消息本身（含 `Role`、`ToolCallID`）保留——LLM 看到的"对话结构"不变。
- AC-1.2 K 可配置；默认 `K = 5`（最近 5 个 tool_result 完整保留）。
- AC-1.3 caller 传入的 `*aimodel.ChatRequest` **不被原地修改**——拷贝副本编辑后送下游 `ChatCompleter`；caller 二次调用同一 request 时看到的依然是原始内容。
- AC-1.4 只对 `Role == RoleTool` 的内容做擦除；`assistant` 的 `tool_calls`（`ToolCall` 数组）保留——保持 `tool_use → tool_result` 的配对完整性。
- AC-1.5 中间件对流式（`ChatCompletionStream`）与非流式（`ChatCompletion`）行为一致：编辑发生在请求出站前，不影响响应处理。

### US-2（用户/平台）：可观察 & 可关闭

**As** 平台运维 / 排查问题的开发者，
**I want** 看到本轮 LLM 调用一共擦了几条、释放了多少 token，并且能在排障时一键关闭擦除，
**so that** 擦除是"明牌"的——不会出现"LLM 行为变了但说不清是哪条 tool_result 被折叠了"。

**验收标准**

- AC-2.1 当至少擦除了 1 条 tool_result 时，中间件通过注入的 `DispatchFunc`（与 `MetricsMiddleware` 同一接口）派发一条 `EventContextEdited` 事件，payload 包含：擦除条数、释放字节数（去掉的原文长度之和）、保留条数、消息总数。
- AC-2.2 一条 tool_result 都没擦时**不派发**事件（与 `GuardCheck` 的"silent pass 不发事件"一致）。
- AC-2.3 `DispatchFunc` 为 nil 时降级为静默执行，不 panic。
- AC-2.4 未挂中间件时行为与本期之前完全等价（零回归）。
- AC-2.5 同一 Run 内多次 LLM 调用时，每次调用各自独立判断、独立派发事件——中间件本身无状态。

### US-3（开发者集成）：TaskAgent option 暴露

**As** TaskAgent 用户，
**I want** 通过 functional option 直接把 ContextEditor 挂进 TaskAgent 内部的中间件链，无需自己重新组装 `largemodel.Chain`，
**so that** 接入成本与 `WithIterationStore` / `WithExtraSources` 等其他 option 持平。

**验收标准**

- AC-3.1 TaskAgent 至少暴露一种姿态把 `ContextEditorMiddleware` 装到内部 `chatCompleter` 外层（具体形式由设计阶段定）。
- AC-3.2 当 TaskAgent 已经被构造时无需重新组装即可启用；不要求改变现有 `WithChatCompleter` 的语义。
- AC-3.3 现有 TaskAgent 单测 / 集测全部通过（无回归）。

---

## 3. In-scope / Out-of-scope

### In-scope（本期交付）

- `largemodel/context_editor.go` —— 新增 `ContextEditorMiddleware`，提供 functional options（保留近 K 条、最小擦除阈值、占位符模板、DispatchFunc）。
- 单元测试：覆盖每条 AC，含 nil DispatchFunc / 全是 user assistant / 工具结果不足阈值 / 流式调用 / 不修改原 request 等边界。
- TaskAgent 暴露集成姿态（option 或 helper），含针对该姿态的测试。
- schema 新增 `EventContextEdited` + `ContextEditedData`。
- 文档更新：`vage/.doc/largemodel.md` 段落补 ContextEditor；`doc/design/session-context-solution.md` 第 4.3 / 第 8 节 / 末尾路线段标注 ✅；PRD 同步（如适用）。

### Out-of-scope（明示留作后续，避免本期范围蔓延）

- **Stale tool_result 标记**（同一文件被改动后旧读返回失效）。需要语义层信息，不在本期纯结构层擦除范围内，未来可作 ContextEditor 的"语义子模块"。
- **单条消息最大 tokens / 转引用**（草案里另一条建议）。需要外部存储 + 引用回指机制，留作后续与 P3 检索 / Workspace artifacts 配套交付。
- **跨 Run 的 session 级擦除**。session 历史已经由 `vage/context` 的 `SessionMemorySource` + compressor 处理，本期 ContextEditor 只针对**本次 Run 的 ReAct 内部**消息（即 caller 传入 `ChatRequest.Messages` 的全集）。
- **TaskAgent 的"擦除策略热切换"** 等高级配置。本期默认中间件挂上即生效，关闭靠"不挂"。

---

## 4. 受影响范围

| 维度 | 受影响 |
|---|---|
| 角色 | TaskAgent 用户（开发者） / 平台运维（hook 消费方） |
| 模型 | 不引入新业务模型；新增 `schema.ContextEditedData` |
| 流程 | 长任务 LLM 调用前增加一次"擦除/折叠"步骤（中间件链内） |
| 应用 | vv 间接受益（默认 setup 中是否启用，本期默认**不启用**，避免立刻改变现网行为；由用户显式挂入） |

---

## 5. 假设 & 关键判断

- **A1**：`aimodel.Message` 是规范化形式，OpenAI/Anthropic translator 不读 `Role==RoleTool` 之外的字段做擦除判定——所以擦除直接作用在规范化 message 上是协议无关的。✓ 已验证：`aimodel/anthropic.go` 在 `Role==RoleTool` 时把 `Content.Text()` 包装成 anthropic 的 `tool_result` block，因此**擦后的占位文本**会自然进入 anthropic 的 tool_result content 字段，不破坏协议。
- **A2**：caller 在调用 `ChatCompleter.ChatCompletion(ctx, req)` 之后**仍然依赖 `req.Messages` 保持原样**（TaskAgent 把 messages 当作 ReAct 累积态写回）——所以中间件**必须**拷贝副本，不能原地编辑。
- **A3**：擦除策略本期最简：**最近 K 条完整保留，更早的 tool_result 内容替换为占位符**。这就是 Anthropic 文档里的最简形态，且对长任务收益最大、行为可解释、易测试。复杂策略（按 token 阈值、按工具名白名单、按错误标记）留作后续扩展。
- **A4**：擦除发生在 **最外层**（与 BudgetMiddleware 同一档次），但**位置上挂在 cache 之外、metrics 之内即可**——具体顺序在设计阶段确认。本需求不绑定具体顺序。
- **A5**：擦除事件**不**纳入 `LogMiddleware` 的常规日志主体，避免日志噪声。审计走 hook 体系（Event）。

---

## 6. 强可验证成功标准（汇总）

实现完成后必须**全部**满足：

1. ✅ 新增 `largemodel.ContextEditorMiddleware` + functional options，单测覆盖率 ≥ 现有 largemodel 包均值。
2. ✅ TaskAgent 提供集成姿态，附测试证明 option 不破坏 ReAct 主路径。
3. ✅ 全量 `make test` 在 `aimodel/`、`vage/`（含 `vage/integrations/` 不依赖外网的部分）、`vv/` 三个 module 通过，无回归。
4. ✅ 全量 `make lint` 通过。
5. ✅ `doc/design/session-context-solution.md` 第 4.3 节末尾标注 ✅，第 8 节 *Context Editing* 行从 ❌ 改为 ✅，末尾路线段把 "Context Editing" 项标注完成。
6. ✅ `vage/.doc/largemodel.md` 在 §3 内建中间件 + §4 流式行为表 + 必要处提及 ContextEditor。

---

## 7. 已发现的不一致 / 风险

- 设计文档 §4.3 列了三条建议（折叠老 tool_result / 标记 stale / 限制单条最大 tokens 转引用）。本期只交付第一条，第二、三条作为 out-of-scope 显式标注，避免读者误以为已全部实现。
- 设计文档没说"挂在中间件链的哪一档"——本期由 designer 决策并写入 design.md。
