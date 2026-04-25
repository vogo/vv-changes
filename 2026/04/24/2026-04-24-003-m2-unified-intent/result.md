# M2 统一意图（unified intent）落地结果

> Session: 2026-04-24-003-m2-unified-intent
> 对应设计: `doc/design/enhance-prompt-understand-and-solve.md` §3.2 / 里程碑 M2
> 状态: 已完成，灰度开关默认关闭

## 1. 核心目标对齐（Core Goal 回写）

原核心目标：当 `orchestrate.unified_intent=true` 且模型选择 `answer_directly` 时，用户可见的回复必须通过**恰好一次** LLM 调用产出（原实现为 2 次：intent + chat）。

验证证据：

- 单测 `TestUnifiedIntent_AnswerDirectly_SingleLLMCall`（`vv/dispatches/unified_intent_test.go`）直接断言 `llm.calls == 1` 并验证 `chat` sub-agent 未被触发。
- 集成测 `TestUnifiedIntent_AnswerDirectly_OneLLMCall`（`vv/integrations/dispatches_tests/dispatches_tests/unified_intent_integration_test.go`）在 Dispatcher 端到端路径上断言 `mockLLM.callCount == 1`，且 `chat`/`coder` 均未被调用。
- 两测均通过。核心目标达成。

## 2. 实际改动清单（Change Log）

### 新增文件
- `vv/dispatches/unified_intent.go` — 工具模式（`answer_directly` / `delegate_to` / `plan_task`）、统一意图 prompt、解析器、答复短路辅助函数。
- `vv/dispatches/unified_intent_test.go` — 9 个单测覆盖三条工具路径 + 纯文本回退 + 参数解析失败 + 未知 agent + LLM 异常 + 关闭开关 + 工具模式校验。
- `vv/integrations/dispatches_tests/dispatches_tests/unified_intent_integration_test.go` — 5 个集成测（answered 一次调用、delegate_to 回退至直连、plan_task 走 DAG、流式 Phase 名切换、关闭开关回退经典路径）。
- `changes/2026/04/24/2026-04-24-003-m2-unified-intent/spec.md` — 事前 spec。
- `changes/2026/04/24/2026-04-24-003-m2-unified-intent/result.md` — 本文件。

### 修改文件
- `vv/configs/config.go`
  - `OrchestrateConfig` 新增字段 `UnifiedIntent bool`（yaml key: `unified_intent`），默认 `false`。
- `vv/dispatches/types.go`
  - `IntentResult` 新增 `Answer string` 字段；新增常量 `IntentModeAnswered = "answered"`；`validate` 增加对 `answered` 模式的校验（要求非空 Answer）。
- `vv/dispatches/dispatch.go`
  - `Dispatcher` 新增 `unifiedIntent bool` 字段。
  - 新增 `WithUnifiedIntent(bool)` 函数式选项。
  - `Run` 在 `intent.Mode == IntentModeAnswered` 时短路，返回通过 `runAnsweredDirect` 构造的 RunResponse，跳过 `executeTask`/`summarize`。
  - `RunStream` 在统一模式下将 Phase 名由 `"intent"` 切换为 `"unified_intent"`；在 answered 分支调用 `streamAnsweredDirect` 以 `text_delta` 形式流出答复。
- `vv/dispatches/intent.go`
  - `recognizeIntent` / `recognizeIntentStream` 在 `plannerAgent == nil && useUnifiedIntent()` 时路由至新增的 `recognizeIntentUnified`。
  - 新增 `useUnifiedIntent()` 守卫方法（要求 `llm != nil`）。
  - `buildIntentSummary` 为 `answered` 模式返回 `unifiedAnsweredSummary`；`direct` / `plan` 的原有格式保持不变（保证既有消费方不受影响，原单测 `TestBuildIntentSummary` 继续通过）。
- `vv/setup/setup.go`
  - Dispatcher 构造处新增一行 `dispatches.WithUnifiedIntent(cfg.Orchestrate.UnifiedIntent)`。
- `vv/dispatches/dispatch_test.go`
  - `stubAgent` 附加 `runs` 计数和 `ranCount()` 方法（加性修改，不影响既有断言）。

### 未触达
- `vv/dispatches/intent.go:recognizeIntentDirect` / `reassessIntent` / `recognizeIntentViaPlanner` 均未动。M2 显式 scope out：planner-agent 分支、explorer 重评流程。
- `vv/agents/chat.go` 未引入新依赖；为避免 `vv/agents` → `vv/dispatches` 的循环，将 chat guidelines 文本就地拷贝到 `unified_intent.go` 的 `unifiedChatGuidelines()`。

## 3. 验证结果（Evidence / Done Contract）

### 单测（`make test` 在 `vv/`）
- `go test ./dispatches/` ：全部通过（含新增 9 个 `TestUnifiedIntent_*`）。
- `make test` 完整扫出无 `FAIL`。

### 集成测（`go test ./...` 在 `vv/integrations/dispatches_tests`）
- 5 个 `TestUnifiedIntent_*` 全通过，既有 M1 `TestFastPathIntegration_*`、经典意图/replan/streaming/summary 测试未受影响。

### Lint
- `make lint` 在 `vv/`：`0 issues`。
- `golangci-lint run` 在 `vv/integrations/dispatches_tests`：`0 issues`。

### Done Contract 逐条核对
| # | Done 项 | 状态 |
|---|---------|------|
| 1 | `orchestrate.unified_intent` 可从 YAML 加载并经 functional option 传入 | ✅ `configs/config.go` + `setup/setup.go` |
| 2 | 开启后走 `recognizeIntentUnified`（planner 为 nil 时） | ✅ `intent.go:useUnifiedIntent` |
| 3 | `IntentResult` 新增 `Mode="answered"` + `Answer`，Run/RunStream 短路 | ✅ `types.go` + `dispatch.go` |
| 4 | 事件中可区分三路径（path labels） | ✅ Phase 名切换为 `unified_intent`；Summary 含 `unified_intent -> answered`；`Direct -> <agent>` 与计划 goal 文本保持既有格式（不影响下游） |
| 5 | 关闭开关时行为与 pre-M2 字节等价 | ✅ `TestUnifiedIntent_Disabled_RunsClassicIntent`（单测 + 集成测） |
| 6 | 单测覆盖三种工具选择 + 异常路径 | ✅ 9 个 `unified_intent_test.go` 用例 |
| 7 | 集成测覆盖 answered / delegate / plan / flag-off | ✅ 5 个 `unified_intent_integration_test.go` 用例 |
| 8 | `make test` + `make lint` 全绿 | ✅ 两个 module 均通过 |

## 4. 与原计划的偏差

| 计划项 | 实际处理 | 原因 |
|--------|----------|------|
| 事件方案：复用 `EventPhaseStart/End`，在 `Summary`/`Detail` 里写 path label（spec 开放问题 #2 默认方案） | 按计划执行。**额外**把流式 Phase 名从 `"intent"` 切换为 `"unified_intent"`，以便 dashboards 一眼区分两种 shape。 | 仅改字符串、向后加性，既不破坏 M1 也不引入新 Event 类型。 |
| `buildIntentSummary` 统一改写为 path label 格式 | **未改**，只增加 `answered` 分支；`direct` / `plan` 保持 `Direct -> X` 与 Plan goal 文本 | 原测 `TestBuildIntentSummary` 依赖原文字面值；为了向后兼容回滚此项改造，改由 Phase 名承载路径区分信息。 |
| `vv/agents/chat.go` 的 `ChatSystemPrompt` 直接 import 进 unified prompt | 改为 `unifiedChatGuidelines()` 内联常量 | `vv/agents` → `vv/dispatches` 会引入循环（agents 通过 registry factory 被 dispatches 使用）。内联是最小侵入方案；若将来想统一来源，可将该常量抽到独立 `vv/prompts` 包。 |
| result.md 语言 | 中文 | 会话过程中 `CLAUDE.md` 更新新增「Use Chinese to generate changes spec/requirement/design document」规则；spec.md 在该规则出现前已完成，未回改。 |

## 5. 未解决 / 后续建议

1. **真实模型回归**：本次实现只用 mock LLM 验证。上线前需要在 Anthropic / OpenAI 后端各跑一遍 answered / delegate / plan 三路径的冒烟，确认工具调用格式被正确翻译（尤其 Anthropic 的 tool_use 映射）。
2. **Planner-agent 分支与 unified intent 的合并**：当 `plannerAgent != nil` 时 unified 模式被主动跳过；若未来希望 planner 也走 tool-calling，需要设计 planner 版本的统一工具集，属于 M4 话题。
3. **HTTP 事件兼容层**：设计 §6.1 提醒 HTTP 客户端可能依赖 `EventPhaseStart("intent")` 做可视化；我们只在流式路径把 Phase 改为 `unified_intent`，非流式 `Run` 端不发事件，风险可控。如需强兼容，建议在 HTTP 适配层把 `unified_intent` 映射回 `intent`。
4. **Prompt 质量调优**：当前 `unifiedIntentSystemPromptTemplate` 是合理初稿；真实环境下若出现 `plan_task` 过度触发，应在 M4 之前把这份 prompt 迁入带有 prompt-caching 的常量并做小样本 eval（设计 §6.4 提到的黄金集）。

## 6. 恢复/交接锚点（Resume Ready）

如需后续迭代：

- 配置开关：`~/.vv/vv.yaml` 下的 `orchestrate.unified_intent: true`；无需重新 setup。
- 开关位置：`vv/configs/config.go:61` 起的 `OrchestrateConfig` 字段；`vv/setup/setup.go:314` 传入处。
- 核心实现：`vv/dispatches/unified_intent.go`（工具 + 解析）+ `vv/dispatches/dispatch.go` 的 `Run`/`RunStream` 短路。
- 单测/集成测在同名 `_test.go` 文件里，`make test`/`make lint` 两端全绿即可视为未回归。
