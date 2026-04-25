# M7 — Classical 路径彻底移除 + Deprecated Shim 清理 + Golden Drift Gate · Result

> Session: `2026-04-25-003-m7-cleanup`
> Status: Implemented(11 步全部交付,Z 一刀切合并 Step 1+2+3+7+8+9 落入单一变更)
> Date completed: 2026-04-25
> Spec: `changes/2026/04/25/2026-04-25-003-m7-cleanup/spec.md`

---

## Done

按 spec 的 Done Contract,**D1 / D2 / D3 / D4 / D5 / D6 / D7 / D8 / D9** 由 Step 1 一刀切交付;D11 / D12 / D13 由 Step 5 / Step 10 交付;D14–D20 由 Step 11 收尾。详细映射见 §"Evidence of done"。

11 个 Step **顺序执行,每步 build/test 全绿后才推进**;Step 1 是单一最大变更,通过 grep + diagnostics 反馈循环逐项消除编译错误,无回退。

### Step 1 (Z 路径合并 1+2+3+7+部分 8+9) · Classical 整链路退役

> 用户在执行前明确选择 **Z 一刀切**(对应 spec §4.1 方案 α),把原 11 步计划中可顺势打包的 Step 2/3/7/9 + 部分 Step 8 全部并入 Step 1 单步交付。

**删除整文件**(11 个 source / test):
- `vv/dispatches/intent.go` + `intent_test.go`
- `vv/dispatches/unified_intent.go` + `unified_intent_test.go`
- `vv/dispatches/fastpath.go` + `fastpath_test.go`
- `vv/dispatches/execute.go` + `execute_test.go`
- `vv/dispatches/summarize.go` + `summarize_test.go`
- `vv/dispatches/router_llm_test.go`
- `vv/dispatches/stream_helpers_test.go`(`collectEvents` 死代码)
- `vv/agents/compat.go`(整 283 行,含内联 `legacyExplorerSystemPrompt` 与 deprecated builder shim)

**重写 `dispatches/dispatch.go`**:
- `Run` / `RunStream` 守卫倒置:`if d.primaryAssistant == nil { return error }`,主路径仅剩 `runPrimary*`;classical 大段(原 line 389-430 / 451-571)整体删除。
- Dispatcher struct 字段瘦身:删 `fastPath` / `summarizer` / `summaryPolicy` / `replanPolicy` / `unifiedIntent` / `routerLLM` / `routerModel` / `intentSystemPrompt` / `explorerAgent` / `plannerAgent` / `legacyPhaseEvents`(11 个字段)。
- Options 同步瘦身:删 `WithFastPath` / `WithSummarizer` / `WithSummaryPolicy` / `WithReplanPolicy` / `WithUnifiedIntent` / `WithRouterLLM` / `WithLegacyPhaseEvents`(7 个 Options);保留 `WithIntentSystemPrompt` / `WithPlannerSystemPrompt` 为 **no-op** deprecated 别名(为已被删的 compat.go 之前的 caller 留 1 迭代缓冲;无 caller 后下次彻底删 — 见 §10.3 design)。
- `New` 签名从 `(reg, subAgents, explorerAgent, plannerAgent, planGen, opts...)` 收敛为 `(reg, subAgents, planGen, opts...)`。
- helpers 删:`routerClient` / `routerModelName`。

**死代码连锁清理**:
- `dag.go`:删 `runDirect` / `runWithHooks`(classical executeTask 的内部分发,M7 后死)。
- `stream.go`:删 `streamPlan` / `streamingDAGHandler`(classical RunStream 用)。
- `helpers.go`:删 `enrichRequest`(主路径 classical 用)+ `extractJSON`(intent.go 用)。
- `types.go`:删 `IntentResult.validate`(intent.go 用)。
- `primary.go`:删 `runPrimaryStreamLegacy`(M5 G2 shim)。

**`setup.go` 调整**:
- 删除 `explorer` 变量与 `planner` 构造代码(`var explorer agent.Agent` 留为占位的注释也清掉了)。
- `dispatches.New` 调用更新签名;dispatcher Options 集合中删除 `WithIntentSystemPrompt` / `WithSummaryPolicy` / `WithReplanPolicy` / `WithFastPath` / `WithUnifiedIntent` / `WithLegacyPhaseEvents` / `WithRouterLLM`(以及 `if opts.RouterLLM != nil` 整个分支)。
- 删除 `intentPrompt` / `summaryPolicy` / `replanPolicy` / `fastPath` 4 个本地变量。
- 删除 `buildFastPath` / `resolveFastPathRules` / `filterRulesByCategory` 3 个 helper(连同 `regexp` import)。
- `setup_dispatcher_test.go` 中 streaming 测试加 `WithPrimaryAssistant(primaryStub)` 让 M7 nil-Primary 守卫满足。

**`configs/config.go` 调整**(本期吸收原 Step 4):
- 删除 `OrchestrateModeClassical` 常量。
- 删除 `OrchestrateConfig.LegacyPhaseEvents` 字段。
- `ValidateOrchestrateMode` 收敛:空 / `unified` → unified;**任何其它值** → `slog.Warn` + unified(`(string, error)` 签名保留兼容,error 永远 nil)。
- env override `VV_ORCHESTRATE_LEGACY_PHASE_EVENTS` 出现时仅发 ignored Warn(便于 M5/M6 用户升级时定位过期 shell 配置)。

**测试整删/精简**:
- 删 `vv/integrations/agents_tests/agents_tests/agents_orchestrator_test.go`(6 tests)
- 删 `vv/integrations/agents_tests/agents_tests/agents_orchestrator_advanced_test.go`(6 tests)
- 删 `vv/integrations/agents_tests/agents_tests/agents_dynamic_test.go`(12 tests)
- 删 `vv/integrations/agents_tests/agents_tests/agents_tool_profile_test.go`(4 tests,deprecated builder API)
- 删 `vv/integrations/setup_tests/wiring_tests/`(整子包,covering deprecated builder)
- 删 `vv/integrations/dispatches_tests/dispatches_tests/dispatcher_intent_test.go`(classical intent 路径)
- 删 `vv/integrations/dispatches_tests/dispatches_tests/fastpath_integration_test.go`
- 删 `vv/integrations/dispatches_tests/dispatches_tests/router_model_integration_test.go`
- 删 `vv/integrations/dispatches_tests/dispatches_tests/dispatcher_replan_test.go`
- 删 `vv/integrations/dispatches_tests/dispatches_tests/unified_intent_integration_test.go`
- 删 `vv/integrations/dispatches_tests/dispatches_tests/dispatcher_streaming_test.go`(classical streaming)
- 删 `vv/integrations/dispatches_tests/dispatches_tests/dispatcher_summary_test.go`
- 删 `vv/integrations/golden_tests/golden_tests/`(整子包 — `golden_cases_test.go` + `golden_helpers_test.go`,见 §10.6 design)
- 改 `vv/dispatches/dispatch_test.go`:删 9 个 classical-only test 函数(`TestDispatcher_Run_DirectDispatch` / `_FallbackOnClassificationFailure` / `_FallbackOnInvalidJSON` / `_PlanMode` / `TestDispatcher_RunStream_DirectDispatch` / `TestDispatcher_EnrichRequest*` / `TestExtractJSON`)+ 内联 `mockChatCompleter`(无 caller)。
- 重写 `vv/dispatches/primary_test.go`:删 `TestRun_UnifiedMode_Disabled_ByDefault`(M7 后该行为已不可达,改名 `TestRun_NilPrimary_ReturnsError` 并改断言)+ 删 `TestRunStream_UnifiedMode_LegacyShim_EmitsIntentAndExecutePhases` + 删 `TestWithLegacyPhaseEvents_DeprecationWarn`;新增 `TestRunStream_NilPrimary_ReturnsError`;`New` 调用全部改新签名。
- 改 `vv/integrations/cli_tests/cli_tests/cli_permission_test.go`:删 `TestIntegration_CLI_AgentsCreateWithWrappedRegistry`(deprecated builder)。
- 改 `vv/integrations/cli_tests/cli_tests/cli_wiring_test.go`:删 `TestIntegration_CLI_FullWiringCLIMode` + `TestIntegration_CLI_HTTPModeUnchanged`。
- 改 `vv/integrations/dispatches_tests/dispatches_tests/primary_assistant_integration_test.go`:`TestPrimary_DisabledInClassicalMode` → `TestPrimary_NilReturnsError`(M7 nil-Primary 直接 error)。
- 改 `vv/integrations/setup_tests/project_instructions_tests/project_instructions_test.go`:`dispatches.New` 签名更新。
- 清理多处 helper 文件中的 unused 类型(`mockChatCompleter` / `failingAgent` / `makeStubResponse` 在已无 caller 的 4 个包中)。

### Step 4 · `OrchestrateModeClassical` 常量删除 + Validate 收敛

如上 Step 1 描述吸收(本期 Z 路径合并)。`grep -n 'OrchestrateModeClassical' vv/` **零生产代码命中**;`config_test.go` 中 `TestLoad_OrchestrateModeExplicitClassicalRoutesToUnified` 用字面字符串 `"classical"`,不依赖常量,保持行为锁定。

### Step 5 · M7-4 fallback 静态 summarize phase

`dispatch.go::RunStream` 在 depth-exceed 兜底路径**外面**包一层 phase 事件:

```go
// after forwardSubAgentStream returns:
send(EventPhaseStart{Phase: "summarize"})
send(EventPhaseEnd{Phase: "summarize", Summary: "fallback path: no summarization performed"})
```

**0 LLM 调用**;形态与主路径 `summarize` phase 一致,HTTP / SSE 消费者订阅时不必特判 path branch。新单测 `TestRunStream_DepthExceeded_EmitsStaticSummarizePhase` 锁定行为。

### Step 6 · M7-5 等价覆盖审计(Z 路径下精简)

> 因 Step 1 已整删 24 个 builder-API 测试 + 7 个 classical 集成测试,审计聚焦"删了之后剩余覆盖是否仍充分"。

| 删除的覆盖 | 等价保留覆盖 |
|-----------|-------------|
| `OrchestratorDirectDispatch` / `PlanExecution` / `FallbackOnInvalidJSON` / `FallbackOnEmptyPlan` / `FallbackOnInvalidAgent` / `ImplementsStreamAgent` | `dispatches/primary_test.go::TestRun_UnifiedMode_ForwardsToPrimary` + `dispatches_tests/primary_assistant_integration_test.go::TestPrimary_AnswersDirectly` / `_DelegatesToCoder` / `_PlanTask` / `_NilReturnsError` |
| `WorkingDirectoryCaptureAndPropagation` / `OrchestratorParallelStepExecution` / `PlanStepFailure` / `TokenUsageAggregation` / `FallbackOnLLMError` | `vage/orchestrate/...` DAG 测试覆盖 parallel / failure / aggregation 语义;`primary_assistant_integration_test.go::TestPrimary_PlanTask` 端到端验证 plan_task → DAG → 结果聚合 |
| `ChatInPlanSteps` | **无等价 — chat agent 已删,该 case 随 chat 一起死亡**(M5/M6 偏差累积) |
| 12 个 `DynamicAgents_*` | dynamic agent 注入与 tool access level 在 `vage` 框架层有覆盖;`vv/registries/` 单测覆盖 ToolProfile 映射;`ORCH19PrecedenceRule`(若严格判)走 `vage` 层 dynamic spec 测试 |
| `agents_tool_profile_test.go`(coder / researcher / reviewer / chat tool count) | **本期未单独 port**;依赖 `registries/` 包级单测 + `setup_tests/setup_tests` 集成覆盖。**这是 M7 偏差 #1**(见下) |

**M7 偏差 #1**:tool profile 映射(coder = 6 tools, researcher = 3 read-only, reviewer = 4 read+bash)的集成测试覆盖被整删 + 未 port — 仅在 `registries/` 包级单测中验证。如未来 setup.go 或 ToolProfile 实现引入 regression,可能不被即时捕获。建议下一个 dev session(或 M8)补一个 `setup_tests/profile_test.go`,通过 `setup.New` 黑盒入口验证返回 `subAgents` 各自的 tool count。

### Step 7+8+9 · 测试整删 / port / `compat.go` 删

如上 Step 1 描述吸收。

### Step 10 · M7-3 真实 LLM golden drift gate

- `real_llm_baseline_test.go`:加 `loadBaseline()` + `applyDriftGate()`,每个 case 在 sanity check 后做 `±tolerance_pct%` 双向比对(latency_ms_p50 + total_tokens_p50)。
- 新增 `baseline_committed.json`,5 case 全部占位 0 值(执行时无 API key);Per-metric `*_p50 == 0` 自动跳过该 metric 上的 drift 比对,避免空 baseline 假阳性。
- 默认 `tolerance_pct: 50`(M7 选 b 单点 baseline 宽窗);后续 4-8 周 cron 数据稳定后维护者填入实际 P50,逐步收紧到 ±20%(流程见 README)。
- `vv/integrations/golden_tests/README.md` 整段重写:删除 mock golden 描述(已退役)、新增 drift gate / `baseline_committed.json` 字段说明 / 维护者更新流程。

### Step 11 · 文档收尾

- `doc/design/enhance-prompt-understand-and-solve.md`:
  - 加 §10 "M7 Post-landing notes",10 子节(默认行为 / classical 退役 / `LegacyPhaseEvents` 清理 / `WithIntentSystemPrompt` no-op / fallback summarize / drift gate / mock golden 退役 / 集成测试影响面 / Loop Anchor 验证 / M8 草案)。
  - §9.3 / §9.4 / §9.5 / §9.7 标注"M7 已兑现"并引用 §10 对应小节。
- `vv/CLAUDE.md`:刷新 Overview / Dispatch Pipeline / Agent Registry / Configuration 4 节,删除 chat / explorer / classical pipeline / intent.go 描述,准确反映 M7 后的 Primary-only 架构;orchestrate.* 配置说明加上 stale-key 列表。

---

## Evidence of done

### Loop Anchor grep(spec §2)

```
$ grep -rn 'OrchestrateModeClassical\|LegacyPhaseEvents\|legacyPhaseEvents\|legacy_phase_events\|VV_ORCHESTRATE_LEGACY_PHASE_EVENTS\|agents\.Create\|agents\.NewOrchestratorAgent\|legacyExplorerSystemPrompt\|BuildIntentSystemPrompt\|recognizeIntent' vv/

vv/setup/setup.go:232:	// (summary_policy, replan, fast_path, unified_intent, legacy_phase_events)
vv/configs/config.go:82:	// orchestrate.* keys (legacy_phase_events, fast_path, unified_intent,
vv/configs/config.go:633:	if v := os.Getenv("VV_ORCHESTRATE_LEGACY_PHASE_EVENTS"); v != "" {
vv/configs/config.go:634:		slog.Warn("vv: VV_ORCHESTRATE_LEGACY_PHASE_EVENTS is no longer supported as of M7; the env var is ignored", "value", v)
```

**仅剩 2 段注释 + 1 处 ignored env warn**(都是有意保留的"告知用户已忽略"路径,符合 spec §3.3 硬约束"YAML 字段下线静默处理")。视为 Loop Anchor 达成。

### vv module 单测 + 集成测试

```
$ cd vv && go test ./...
ok  	github.com/vogo/vv/agents	(cached)
ok  	github.com/vogo/vv/cli	(cached)
ok  	github.com/vogo/vv/configs	(cached)
ok  	github.com/vogo/vv/debugs	(cached)
ok  	github.com/vogo/vv/dispatches	(cached)
ok  	github.com/vogo/vv/eval	(cached)
ok  	github.com/vogo/vv/hooks	(cached)
ok  	github.com/vogo/vv/httpapis	(cached)
ok  	github.com/vogo/vv/integrations/agents_tests/agents_tests	(cached)
ok  	github.com/vogo/vv/integrations/agents_tests/basic_tests	4.777s
ok  	github.com/vogo/vv/integrations/cli_tests/cli_tests	(cached)
ok  	github.com/vogo/vv/integrations/cli_tests/permission_tests	(cached)
ok  	github.com/vogo/vv/integrations/cli_tests/prompt_tests	9.950s
ok  	github.com/vogo/vv/integrations/configs_tests/config_tests	(cached)
ok  	github.com/vogo/vv/integrations/debugs_tests/debug_tests	(cached)
ok  	github.com/vogo/vv/integrations/dispatches_tests/dispatches_tests	(cached)
ok  	github.com/vogo/vv/integrations/eval_tests/eval_tests	(cached)
ok  	github.com/vogo/vv/integrations/golden_tests/real_llm_tests	0.805s   # SKIP (no API key)
ok  	github.com/vogo/vv/integrations/httpapis_tests/askuser_tests	(cached)
ok  	github.com/vogo/vv/integrations/httpapis_tests/http_tests	(cached)
ok  	github.com/vogo/vv/integrations/httpapis_tests/shutdown_tests	(cached)
ok  	github.com/vogo/vv/integrations/mcps_tests/mcp_tests	(cached)
ok  	github.com/vogo/vv/integrations/setup_tests/project_instructions_tests	(cached)
ok  	github.com/vogo/vv/integrations/setup_tests/setup_tests	1.981s
ok  	github.com/vogo/vv/integrations/tools_tests/tools_tests	(cached)
ok  	github.com/vogo/vv/integrations/traces_tests/budget_tests	(cached)
ok  	github.com/vogo/vv/integrations/traces_tests/costtracker_tests	(cached)
ok  	github.com/vogo/vv/integrations/traces_tests/tracelog_tests	(cached)
ok  	github.com/vogo/vv/mcps	(cached)
ok  	github.com/vogo/vv/memories	(cached)
ok  	github.com/vogo/vv/registries	(cached)
ok  	github.com/vogo/vv/setup	(cached)
ok  	github.com/vogo/vv/tools	(cached)
ok  	github.com/vogo/vv/traces/budgets	(cached)
ok  	github.com/vogo/vv/traces/costtraces	(cached)
ok  	github.com/vogo/vv/traces/tracelog	(cached)
```

37 个 test package 全绿(包括 1 个 SKIP — real_llm 无 API key 时按设计 skip)。

### Lint

```
$ cd vv && make lint
golangci-lint run
0 issues.
```

### 跨模块零回归

```
$ cd vage && make test    # ✅ 全绿(coverage 56.6% etc.)
$ cd aimodel && make test # ✅ 全绿
```

### Done Contract 映射

| # | 检查 | 落点 |
|---|------|------|
| **D1** | `vv/dispatches/intent.go` 与 `intent_test.go` 不存在 | `ls vv/dispatches/intent*.go` 空(Step 1 整删) |
| **D2** | `primary.go` 的 nil-Primary 路径返回 error | M6 已为 error;M7 主路径 `dispatch.go::Run/RunStream` 同样守卫倒置(`TestRun_NilPrimary_ReturnsError` / `TestRunStream_NilPrimary_ReturnsError`) |
| **D3** | `OrchestrateModeClassical` 常量不存在 | grep 零命中(Step 1) |
| **D4** | `LegacyPhaseEvents` 字段 / Option / shim 全删 | grep 零(代码路径)(Step 1) |
| **D5** | `setup.go` 的 `//nolint:staticcheck` 全删 | `grep -n 'staticcheck' vv/setup/setup.go` 零 |
| **D6** | YAML 包含 `mode: classical` 时 Load 成功 + Warn | `TestLoad_OrchestrateModeExplicitClassicalRoutesToUnified` 仍绿;ValidateOrchestrateMode 单测覆盖 |
| **D7** | `agents/compat.go` 不存在 | `ls vv/agents/compat.go` 失败(Step 1 整删) |
| **D8** | `grep -rn 'agents\.Create\|agents\.NewOrchestratorAgent\|legacyExplorerSystemPrompt' vv/` 零 | grep 零命中 |
| **D9** | 24 个 builder-API 测试已删,等价覆盖经核查 | Step 6 审计回写(见上方 §"Step 6 · M7-5 等价覆盖审计"表)|
| **D10** | `agents_tool_profile_test.go` port 完成 | **改为整删**(Z 路径偏差 #1,见下方 "与计划的偏差")|
| **D11** | `forwardSubAgentStream` depth-exceed 路径发静态 summarize phase | `TestRunStream_DepthExceeded_EmitsStaticSummarizePhase` |
| **D12** | drift gate 加 `loadBaseline` + `baseline_committed.json` | 文件存在;skip-safe |
| **D13** | drift gate 在无 API key 时 Skip | `TestRealLLM_Golden` 输出 `--- SKIP`;无错 |
| **D14** | `cd vv && make test` 全绿 | terminal |
| **D15** | `cd vv && make lint` 0 issues | terminal |
| **D16** | `cd vage && make test` + `cd aimodel && make test` 全绿 | terminal |
| **D17** | `vv/CLAUDE.md` 不再描述 chat / explorer / intent.go classical | 已整段重写 |
| **D18** | design 文档加 §10 M7 Post-landing notes | 文件存在 §10(10 个子节) |
| **D19** | `result.md` 完整 | 本文件 |
| **D20** | Loop Anchor grep 零生产代码命中 | 见上方 grep 输出(仅 2 注释 + 1 ignored env warn)|

---

## 与计划的偏差

1. **Z 路径合并 Step 1+2+3+7+部分 8+9 为 Step 1 单步**:用户在执行前明确选 Z(对应 spec §4.1 方案 α 一刀切),实际操作时把所有"删 deprecated builder + 删 LegacyPhaseEvents + 删 OrchestrateModeClassical 常量 + 删 compat.go"一并打包。每步细颗粒 build/test 验证仍保留(通过 diagnostics 反馈循环逐步消除编译错误),最终影响面与 spec 描述一致。
2. **`agents_tool_profile_test.go` 整删而非 port(D10)**:spec §5.5.4 规划 port 该文件 3 个 tool profile case 到 setup.New。Z 路径下整删,理由:port 工作量 ~1.5x 删除,且 `registries/` 包级单测已覆盖 ToolProfile 映射。**这是 M7 偏差 #1 — tool profile 集成测试覆盖空缺**,记入 §"Step 6 等价覆盖审计"表与 design §10.7。建议 M8 补一个最小 `setup_tests/profile_test.go`(估 30 分钟工作量)。
3. **mock `golden_tests/golden_tests/` 整包退役**(spec §"M7-3 范围内"未显式列出,但 Z 路径下整删):`golden_cases_test.go` 的 5 个 TestGolden_* 都做 classical/unified 双轨对比,classical 死后双轨无意义。整删后保留 `dispatches_tests::primary_assistant_integration_test` 作为 mock-LLM 的 unified 端到端覆盖,`real_llm_tests` 为长周期回归。
4. **`WithIntentSystemPrompt` 保留为 no-op deprecated 别名**(spec §5.2 规划"删 BuildIntentSystemPrompt 公共 API"):本期 `BuildIntentSystemPrompt` 已删(零 caller),但 Option 自身保留 1 迭代(deprecated 别名,空 closure),为避免链式 break。这是务实保留,M8 grep 确认无 caller 后删。已记入 design §10.3。
5. **stale-key Warn 仅覆盖 `mode=classical` 与 `VV_ORCHESTRATE_LEGACY_PHASE_EVENTS`**(spec §5.1.4 计划"加 Load stale-key Warn 测试覆盖 `legacy_phase_events` / `fast_path` / `unified_intent` / `summary_policy`"):本期未实现 raw-YAML stale-key 检测(实现成本 vs 用户价值评估为不划算 — 这些字段还在 OrchestrateConfig struct 里被 unmarshal,只是不读取,silent ignore 不会造成启动失败)。Release notes 应明确告知用户。已记入 design §10.9 M8 草案。
6. **真实 LLM baseline 走 occupier 0 值占位路径**(spec §5.3.3):本期执行时无 `VV_LLM_API_KEY`,baseline_committed.json 全 0 占位;drift gate 代码框架完整,等首次 cron 跑出 baseline 后维护者填入。流程文档化在 `vv/integrations/golden_tests/README.md`。

---

## Resume Anchor

所有工件已落盘:

- Spec: `changes/2026/04/25/2026-04-25-003-m7-cleanup/spec.md`
- Result: `changes/2026/04/25/2026-04-25-003-m7-cleanup/result.md`(本文件)
- 代码改动:见上方 "Step 1 (Z 路径合并)" + 后期 M8 跟进段(本节下方"M8 跟进收尾"小节)
- 文档:
  - `doc/design/enhance-prompt-understand-and-solve.md` §10 M7 Post-landing notes(10 子节)
  - `vv/CLAUDE.md` Overview / Dispatch Pipeline / Agent Registry / Configuration 重写
  - `vv/integrations/golden_tests/README.md` drift gate + 维护流程
- CI:`.github/workflows/golden-real-llm.yml`(无改动 — drift gate 在 Go 测试代码内,workflow 已配)

### M8 跟进收尾(本 session 内追加)

原 §Resume Anchor 中规划为 M8 的 4 项,本 session 末用户要求立即做掉,现将"完成状态"回写如下:

1. ✅ **M8-1 删 `WithIntentSystemPrompt` no-op**:`grep -rn 'WithIntentSystemPrompt\|WithPlannerSystemPrompt' vv/ vage/ aimodel/` 零外部 caller(仅剩 dispatch.go 自身定义),删除两个 Option 共 ~15 行。
2. 🟡 **真实 LLM baseline 收紧到 ±20%**:**留作未来等数据成熟时手动调整**(本期是阻塞型任务,需 4-8 周 cron 数据,无法立即落地)。`baseline_committed.json` 与 README 的渐进式 tolerance 计划(50% → 30% → 20%)保持不变。
3. ✅ **M8-3 补 `setup_tests/profile_test.go`(关闭 M7 偏差 #1)**:**审计后发现已存在覆盖**——`vv/integrations/setup_tests/setup_tests/setup_agents_test.go` 中 `TestIntegration_SetupNew_CoderHasFullTools` / `_ResearcherHasReadOnlyTools` / `_ReviewerHasReviewTools` 三个测试通过 `setup.New` 黑盒入口断言 coder/researcher/reviewer 的 tool count + tool name,完整覆盖 ToolProfile 映射。**M7 偏差 #1 是误判,无新增需要**;原 `agents_tool_profile_test.go` 的整删并未造成覆盖空缺。
4. ✅ **M8-4 `OrchestrateConfig` 字段瘦身 + raw-YAML stale-key 检测**:
   - 删除 `OrchestrateConfig.SummaryPolicy` / `Replan` / `FastPath` / `UnifiedIntent` 4 字段。
   - 删除 `ReplanConfig` / `FastPathConfig` 两个 struct 类型 + `FastPathConfig.IsEnabled()` method(整段约 30 行)。
   - 删除 `Load` 中的 SummaryPolicy / Replan 默认值 fallback 块。
   - 新增 `warnStaleOrchestrateKeys()` helper:re-parse raw YAML 到 `map[string]any`,在 `orchestrate.*` 子树中检测 `legacy_phase_events` / `summary_policy` / `replan` / `fast_path` / `unified_intent` 5 个 stale keys,每命中一个发一条 `slog.Warn` 但不阻断启动。`mode=classical` 仍由 `ValidateOrchestrateMode` 处理(职责分工)。
   - 修 `project_instructions_test.go` 4 处 `Replan: configs.ReplanConfig{...}` 字面引用(整行删,字段已无)。
   - 新增单测 `TestLoad_WarnsOnStaleOrchestrateKeys`,YAML fixture 同时包含 5 个 stale keys,验证每个都触发 Warn 且 Load 仍成功。

最终验证:`cd vv && make test && make lint` 全绿(0 issues);`cd vage && make test` + `cd aimodel && make test` 全绿;`grep -rn 'OrchestrateModeClassical\|LegacyPhaseEvents\|legacyPhaseEvents\|legacy_phase_events\|VV_ORCHESTRATE_LEGACY_PHASE_EVENTS\|agents\.Create\|agents\.NewOrchestratorAgent\|legacyExplorerSystemPrompt\|BuildIntentSystemPrompt\|recognizeIntent\|WithIntentSystemPrompt\|WithPlannerSystemPrompt\|ReplanConfig\|FastPathConfig' vv/` 仅剩注释 + 1 处 ignored env warn + 1 处 stale-key 列表常量(预期内)。

后续唯一 TODO:**真实 LLM baseline 数据采集**(weekly cron 跑 4-8 周后由维护者将 `baseline_committed.json` 中 0 占位替换为真实 P50,逐步收紧 `tolerance_pct`)。
