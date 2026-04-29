# Context Builder — 集成测试报告 v1

测试者：Claude（tester pass）
测试日期：2026-04-29
被测特性：`vage/context/`（vctx）Builder + Source 抽象、`schema.EventContextBuilt`、TaskAgent 集成。

## 1. 范围与方法

- 新增集成测试目录 `vage/integrations/context_tests/`，共 7 个 `.go` 文件、12 个 `Test*` 用例。
- 测试遵循集成测试原则：使用真实 `memory.Manager`、`hook.Manager`、`session.MapSessionStore`、真实 `vctx.DefaultBuilder` 与全部内置 Source；只对 LLM 进行 mock（`fakeChatCompleter` 捕获 ChatRequest）。
- 每个测试用例顶端有简注，说明覆盖的验收点。

## 2. 测试用例与验收点映射

| 测试用例 | 文件 | 验收点 |
|---|---|---|
| `TestTaskAgent_ContextAssembly_BehaviorCompat` | `taskagent_compat_test.go` | AC-3.1：System + SessionMemory + Request 三 Source 的输出顺序与旧实现一致（system → 历史 → 本轮）。 |
| `TestTaskAgent_PromptCacheBreakpointPreserved` | `taskagent_compat_test.go` | AC-3.4：Builder 之后 `markPromptCacheBreakpoints` 仍能在 system message 上打标。 |
| `TestTaskAgent_EmitsEventContextBuilt` | `event_test.go` | AC-2.1、AC-2.2：TaskAgent 触发 `EventContextBuilt`，Payload 类型 `schema.ContextBuiltData`，字段（Builder/Strategy/OutputCount/Sources…）非空。 |
| `TestBuilder_PluggableSource_AppearsInOutput` | `pluggable_source_test.go` | AC-1.3、AC-4.1：自定义 Source 的输出按声明顺序进入 prompt，且报告进入 `BuildReport.Sources`。 |
| `TestBuilder_Budget_OldestDropped` | `budget_test.go` | AC-5.1、AC-5.2：超 Budget 时 oldest history message 被丢弃；`DroppedN > 0`、`Status="truncated"`、`OutputTokens <= Budget`。 |
| `TestBuilder_BudgetZero_Unlimited` | `budget_test.go` | AC-5.3：Budget=0 时不触发裁剪，所有历史保留，`DroppedCount == 0`。 |
| `TestBuilder_SystemPromptRenderError_FailClosed` | `system_prompt_failclosed_test.go` | 设计 §7 + AC-4.2 例外：`SystemPromptSource` 渲染失败导致 Build 返 error，不静默恢复。 |
| `TestBuilder_OptionalSourceError_FailOpen` | `system_prompt_failclosed_test.go` | AC-4.2：可选 Source 报错走 fail-open 路径，后续 Source 仍执行；失败 Source 报告 `Status="error"` 且 `Error` 非空。 |
| `TestBuilder_SessionMemory_WithSlidingWindowCompressor` | `compressor_test.go` | AC-3.1、AC-5.4：与真实 `SlidingWindowCompressor` 协作；`OriginalCount` 是压缩前数量、`OutputN` 是压缩后；保留最新若干条。 |
| `TestSessionStateSource_EndToEnd` | `session_state_test.go` | AC-4.1：`SessionStateSource` 端到端对接 `session.MapSessionStore`，渲染选定 key 为 system message。 |
| `TestSessionStateSource_MissingKeysAndNoSession` | `session_state_test.go` | AC-4.1 / AC-4.2：空 SessionID + 缺失 key 时 fail-open 跳过。 |
| `TestBuildReport_JSONRoundTrip` | `json_test.go` | AC-2.3：实际 Build 产生的 BuildReport（含 Sources）与转出的 ContextBuiltData 都能 marshal+unmarshal 完整往返。 |

## 3. 运行结果

### 3.1 上下文集成测试

```
$ cd vage && go test ./integrations/context_tests/... -v
=== RUN   TestBuilder_Budget_OldestDropped
--- PASS: TestBuilder_Budget_OldestDropped (0.00s)
=== RUN   TestBuilder_BudgetZero_Unlimited
--- PASS: TestBuilder_BudgetZero_Unlimited (0.00s)
=== RUN   TestBuilder_SessionMemory_WithSlidingWindowCompressor
--- PASS: TestBuilder_SessionMemory_WithSlidingWindowCompressor (0.00s)
=== RUN   TestTaskAgent_EmitsEventContextBuilt
--- PASS: TestTaskAgent_EmitsEventContextBuilt (0.00s)
=== RUN   TestBuildReport_JSONRoundTrip
--- PASS: TestBuildReport_JSONRoundTrip (0.00s)
=== RUN   TestBuilder_PluggableSource_AppearsInOutput
--- PASS: TestBuilder_PluggableSource_AppearsInOutput (0.00s)
=== RUN   TestSessionStateSource_EndToEnd
--- PASS: TestSessionStateSource_EndToEnd (0.00s)
=== RUN   TestSessionStateSource_MissingKeysAndNoSession
--- PASS: TestSessionStateSource_MissingKeysAndNoSession (0.00s)
=== RUN   TestBuilder_SystemPromptRenderError_FailClosed
--- PASS: TestBuilder_SystemPromptRenderError_FailClosed (0.00s)
=== RUN   TestBuilder_OptionalSourceError_FailOpen
--- PASS: TestBuilder_OptionalSourceError_FailOpen (0.00s)
=== RUN   TestTaskAgent_ContextAssembly_BehaviorCompat
--- PASS: TestTaskAgent_ContextAssembly_BehaviorCompat (0.00s)
=== RUN   TestTaskAgent_PromptCacheBreakpointPreserved
--- PASS: TestTaskAgent_PromptCacheBreakpointPreserved (0.00s)
PASS
ok  github.com/vogo/vage/integrations/context_tests   1.261s
```

12 PASS / 0 FAIL。

### 3.2 TaskAgent 集成回归

```
$ cd vage && go test ./integrations/agent_tests/taskagent_tests/... -v
... (Budget* 等 10+ 用例) ...
PASS
ok  github.com/vogo/vage/integrations/agent_tests/taskagent_tests   0.548s
```

无回归。`TestTaskAgentIntegration` 因没有 LLM API key 在环境中，按既有逻辑提前返回（PASS 但只是 skip 的占位）；其余 mock 化的预算/迭代用例全部通过。

### 3.3 Lint & License

```
$ make lint
golangci-lint run
0 issues.

$ make license-check
... license_ok: 300 files
```

## 4. 验收点覆盖核对

| AC | 状态 | 由谁覆盖 |
|---|---|---|
| AC-1.1 Builder 接口 | 已覆盖（间接） | 单元测试 `vage/context/builder_test.go` + 集成测试均通过 `vctx.NewDefaultBuilder` 调用 `Build` 验证。 |
| AC-1.2 Source 接口 | 已覆盖 | `TestBuilder_PluggableSource_AppearsInOutput` 通过实现自定义 Source 验证。 |
| AC-1.3 DefaultBuilder + WithSource(...) | 已覆盖 | 多个测试均使用 `WithSource` 追加 Source 链。 |
| AC-1.4 包路径 / 无循环依赖 | 已覆盖（编译） | `go build` 和 `go test` 均通过即说明 vctx 包路径与依赖正确。 |
| AC-2.1 BuildReport 字段齐全 | 已覆盖 | `TestTaskAgent_EmitsEventContextBuilt`、`TestBuilder_Budget_OldestDropped`、`TestBuildReport_JSONRoundTrip` 均断言 builder/strategy/budget/sources/dropped/output_tokens 等字段。 |
| AC-2.2 EventContextBuilt 触发 | 已覆盖 | `TestTaskAgent_EmitsEventContextBuilt` 通过 hook 抓到事件并断言 `schema.ContextBuiltData`。 |
| AC-2.3 BuildReport JSON 序列化 | 已覆盖 | `TestBuildReport_JSONRoundTrip` 双向 marshal/unmarshal 加字段名断言。 |
| AC-3.1 复刻旧 TaskAgent 行为 | 已覆盖 | `TestTaskAgent_ContextAssembly_BehaviorCompat` 按字节顺序断言 `[system, history, request]` 排列。 |
| AC-3.2 现有 TaskAgent 测试无回归 | 已覆盖 | `integrations/agent_tests/taskagent_tests` 全套通过。 |
| AC-3.3 Skill instructions 仍注入 | 部分覆盖 | 未在本次集成层新增专用用例（需要 SKILL.md 测试夹具）。`taskagent` 既有的 `task_test.go` 中 skill 相关单元用例覆盖该路径；本次 Builder 改造未触及 `injectSkillInstructions`，逻辑保持原样。 |
| AC-3.4 Prompt cache breakpoint | 已覆盖 | `TestTaskAgent_PromptCacheBreakpointPreserved` 断言 system 消息 `CacheBreakpoint == true`。 |
| AC-4.1 Source 接口稳定 / SessionStateSource 示例 | 已覆盖 | `TestSessionStateSource_EndToEnd`、`TestSessionStateSource_MissingKeysAndNoSession` 用真实 `MapSessionStore` 跑通。 |
| AC-4.2 Source 失败 fail-open | 已覆盖 | `TestBuilder_OptionalSourceError_FailOpen` 验证 fail-open；`TestBuilder_SystemPromptRenderError_FailClosed` 验证文档化的例外。 |
| AC-5.1 Budget 顺序贪心 | 已覆盖 | `TestBuilder_Budget_OldestDropped` 输出 token 总量受 Budget 约束。 |
| AC-5.2 oldest dropped 优先 | 已覆盖 | `TestBuilder_Budget_OldestDropped` 断言 `00:` 前缀消息消失。 |
| AC-5.3 Budget=0 视为无限 | 已覆盖 | `TestBuilder_BudgetZero_Unlimited`。 |
| AC-5.4 复用 memory 估算器 | 已覆盖（间接） | budget 测试中实际 token 数量与 `memory.EstimateTextTokens` 启发式（4 字符≈1 token）吻合，否则 budget 行为不会与预期一致。 |

## 5. 未覆盖项 / 已知局限

- **AC-3.3 Skill instructions 注入** 仅靠 taskagent 既有单元测试覆盖。本次 Builder 改造未修改 `injectSkillInstructions`，改造点之后无回归即等价于行为保持；如需端到端集成层断言，需引入 SKILL.md 测试夹具，建议下一轮再加。
- 真实 LLM 的端到端跑测（`TestTaskAgentIntegration` 调用真实 OpenAI/Anthropic）依赖环境变量；本次未提供 key，已按既有约定 skip。

## 6. 结论

所有可在无 LLM key 环境下验证的验收点已通过自动化集成测试覆盖，被测特性可进入下一阶段（documenter 更新 PRD）。
