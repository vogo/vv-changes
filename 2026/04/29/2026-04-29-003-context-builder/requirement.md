# 需求：Context Builder 抽象

## 背景与目标

vage 当前已具备：
- 三层 memory（Working / Session / Store）—— 见 `vage/memory/`
- 五种 compressor —— 见 `vage/memory/compressor*.go`
- Session 一等公民实体（2026-04-29-002 已落地）—— 见 `vage/session/`

但是从 **Session 全集 → 实际送入 LLM 的 message 序列**这条链路目前是**隐式硬编码在 `taskagent.Agent.buildInitialMessages` / `loadAndCompressSessionHistory` / `injectSkillInstructions` 三个私有方法中**的：

1. **不可插拔**：无法替换或追加 ContextSource（向量召回、状态投影、文件记忆、子代理结论…）。
2. **无审计**：拼装过程没有 BuildReport，事后无法回答"为什么这次 prompt 是这样"。
3. **无 token 预算**：当前先全量 load → 再 compress，没有"按 budget 选择性装箱"的机制。每个 source 也无法声明自己的 budget 占比。
4. **强耦合 memory.Manager**：dispatcher / planner / explorer 等不依赖 memory 的 agent 也只能复用 TaskAgent 这套硬编码流程，或绕开重写。
5. **难以演进**：未来 §4.3 Context Editing、§4.4 Plan/Scratchpad、§4.6 Subagent SessionView、§4.8 Session Tree 都需要在"Build prompt 之前"插入逻辑。当前没有插入点。

`doc/design/session-context-solution.md` §4.2 给出了 Builder/Source 的初步草图；本次需求是把它落到代码中，并优先级排在 §8 表格的 **Session 实体化（已完成）→ Context Builder（本次）→ vv wiring → ...**。

## 与方案完整性评估

> 在分析阶段同步评估 `doc/design/session-context-solution.md` §4.2 草图的完整性，识别需要在 Designer 阶段补齐的点。

草图确认覆盖：
- ✅ Builder + Source 双层接口
- ✅ BuildReport 用于审计
- ✅ 内置 source 列表（System / RecentTurns / Summary / VectorRecall / StateProjector / FileMemory / SubagentSummary）
- ✅ 装箱（greedy + budget）

草图**未明示**、需在 Design 阶段决断的：
- ⚠️ **Intent 类型**未定义。是 `string` 标签？还是结构体？OpenAI/LangChain 都用 string，建议简单。
- ⚠️ **Budget 数据结构**未定义。是 total int？还是分配比例？建议先 total + 优先级排序，比例分配作为后续 enhancement。
- ⚠️ **BuildReport.SummarizedSpan** 仅一个 `[2]int` 是不够的——多个 source 各自做了不同处理，需要逐 source 报告。
- ⚠️ **错误路径**：source 失败如何处理？fail-open（warn + skip）还是 fail-closed（return err）？业界普遍 fail-open。
- ⚠️ **Source 的执行顺序与 budget 切片机制**需明示——是按 source 优先级顺序拿 budget 余额，还是预分配？
- ⚠️ **TaskAgent 集成路径**：Builder 是替换 `buildInitialMessages` 的全部逻辑，还是只替换 session-history 部分？skill 注入、prompt cache 标记等如何衔接？
- ⚠️ **与 hook 体系集成**：BuildReport 应该作为 EventContextBuilt 事件流到 hook 体系，便于被 SessionHook / TraceLog 自动落盘。

## 业界优秀实践调研要点（已在 §2 完成）

| 框架 | 关键启示 |
|---|---|
| LangChain Memory | Memory 抽象按"取什么"分类，对应 ContextSource |
| LangGraph | State 与 LLM context 解耦，每个节点独立读取 → 印证 Source 模式 |
| LlamaIndex Composable Memory | 多 source 合成单一 context view |
| MemGPT | LLM 主动 page-in/out → 暴露工具的方向（本次仅做静态，留扩展位） |
| Semantic Kernel ChatHistoryReducer | 声明式 reducer chain → 与 compressor chain 概念对齐 |
| Anthropic Effective Context Engineering | "上下文是稀缺资源" → BuildReport + budget 是必备 |

## 用户故事 & 验收标准

### US-1（核心）：作为 vage 用户，我希望能用声明式的方式把 session 装配为 prompt

**Acceptance：**
- AC-1.1 提供 `context.Builder` 接口：`Build(ctx, sess, intent) ([]schema.Message, BuildReport, error)`。
- AC-1.2 提供 `context.Source` 接口：`Fetch(ctx, sess, intent, budget) ([]schema.Message, FetchReport, error)`。
- AC-1.3 提供 `DefaultBuilder`，可通过 `WithSource(...)` 追加任意数量 source；source 顺序即优先级。
- AC-1.4 `Builder` 在 `vage/context/` 新包下；不引入循环依赖（不能依赖 `vage/agent/`）。

### US-2：作为 vage 集成方，我能拿到 BuildReport 用于审计

**Acceptance：**
- AC-2.1 `BuildReport` 至少包含：策略名（builder name）、输入 budget、各 source 的 FetchReport、最终 message 数 / 估算 token 数、被丢弃的 message 数。
- AC-2.2 Builder 完成后通过 hook 体系发出 `EventContextBuilt`（新事件类型），data 即 BuildReport。
- AC-2.3 BuildReport 可序列化为 JSON（无 unsupported types）。

### US-3：能复刻当前 TaskAgent 的行为

**Acceptance：**
- AC-3.1 提供能完全复现当前行为的内置 source 组合：`SystemPromptSource` + `SessionMemorySource`（载入 + 压缩）。
- AC-3.2 现有 TaskAgent 测试在切到 Builder 后全部通过（行为兼容）。
- AC-3.3 Skill instructions 注入路径仍然工作（通过新增 `SkillSource` 或保留 `injectSkillInstructions` post-step，二选一在 Design 决定）。
- AC-3.4 Prompt cache breakpoint 标记仍然被打上（在 Builder 之后）。

### US-4：可扩展为 §4.2 列出的其他 source

**Acceptance：**
- AC-4.1 Source 接口签名足够稳定，能在不修改 Builder 的前提下接入：`SessionStateSource`（投影 SessionStateStore）、`SubagentSummarySource`（拼接子代理结论）。本次至少落实 1 个非平凡 source 作为示例（`SessionStateSource`）。
- AC-4.2 Source 失败 fail-open：报 warning 并继续；最终 BuildReport 中标记 source 状态。

### US-5：Token budget 装箱

**Acceptance：**
- AC-5.1 Builder 接受 `Budget{ Total: int, Reserve: int }`，按 source 顺序贪心分配剩余 budget。
- AC-5.2 当 source 返回的 message 累加超出 budget 时，Builder 截断（drop oldest 优先）；BuildReport 记录被丢弃的数量。
- AC-5.3 Budget 可以为 0（"无限"），此时不裁剪。
- AC-5.4 Token 估算复用 `memory.DefaultTokenEstimator` / `memory.EstimateTextTokens`，不引入新估算器。

## 范围

### In-scope

- 新包 `vage/context/`：Builder、Source、BuildReport、Budget、DefaultBuilder。
- 内置 source：`SystemPromptSource`, `SessionMemorySource`, `SessionStateSource`, `RecentMessagesSource`（来自 RunRequest 的本轮消息）。
- TaskAgent 集成：在 `taskagent` 内部用 Builder 替换 `buildInitialMessages` 的等价路径；保持 100% 行为兼容。
- 新 Event 类型：`EventContextBuilt`，data 类型 `ContextBuiltData = BuildReport`。
- 单元测试：context 包 ≥ 90% 覆盖率；taskagent 现有测试无回归。
- 文档：`vage/.doc/context.md`。

### Out-of-scope（留作后续迭代）

- VectorRecallSource（依赖向量库）
- FileMemorySource（依赖 §4.4 工作目录）
- SubagentSummarySource（依赖 §4.6 SessionView）
- Context Editing 中间件（§4.3，largemodel 层）
- LLM-driven paging 工具（§4.9）
- Builder 的 budget 比例分配策略（仅做"顺序贪心 + 总 budget"）

## 影响范围

| 模块 | 影响 |
|---|---|
| 新包 `vage/context/` | 新增 |
| `vage/schema/event.go` | 新增 `EventContextBuilt` 常量与 `ContextBuiltData` |
| `vage/agent/taskagent/task.go` | `buildInitialMessages` 改写为调用 Builder（保持外部签名兼容） |
| `vage/.doc/context.md` | 新增 |
| `doc/design/session-context-solution.md` | §4.2 与 §8 的 "Context Builder" 行标注完成 |

## 假设与权衡

- **假设**：当前 vage 用户均通过 TaskAgent 间接使用 context 拼装；没有第三方直接调用 `loadAndCompressSessionHistory`（私有方法，验证过）。因此重构是"隐式 → 显式"，不破坏外部 API。
- **权衡**：是把 Builder 暴露成 `WithContextBuilder(b)` 的 TaskAgent option，还是固定使用 DefaultBuilder？
  - **选项 A**：暴露 option，灵活但增加配置复杂度。
  - **选项 B**：固定 DefaultBuilder，但 DefaultBuilder 内部按现有字段（memoryManager/skillManager/systemPrompt）自动组装 source。
  - **建议 A**：让用户能 plug 自定义 source；TaskAgent 默认用 DefaultBuilder（基于现有字段构造），保持向后兼容。
- **权衡**：BuildReport 通过 hook 还是直接挂 RunResponse？
  - **建议**：通过 hook（EventContextBuilt）。RunResponse 已有 Usage/Duration，再加 BuildReport 会让接口臃肿。

## 与现有文档不一致点

- `doc/design/session-context-solution.md` §4.2 写的是 `Build(ctx, sess *session.Session, intent Intent)`，但 vage 当前的 `RunRequest` 不传 `*session.Session`，只传 `SessionID string`。Builder 必须能两种形态都支持：直接传 sess 对象 / 传 SessionID 通过 store 拉取。Design 阶段决定。
