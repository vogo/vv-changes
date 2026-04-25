# M4 — Primary Assistant 原型(Unified Mode)· Result

> Session: 2026-04-24-005-m4-primary-assistant
> Status: Implemented
> Date completed: 2026-04-25

## Done

按 spec 的 Done Contract 全部兑现:

1. **Config**:`OrchestrateConfig.Mode` 新增 + `""/"classical"/"unified"` 合法值 + `VV_ORCHESTRATE_MODE` env + `ValidateOrchestrateMode` 校验(未知值在 `Load` 时 fail-fast)。
2. **Primary Assistant**:`vv/agents/primary.go` —— `RegisterPrimary` 注册 ID=`primary`、`ProfileReadOnly`、非 dispatchable 的 descriptor;Factory 产出 TaskAgent,支持外部注入的 ToolRegistry。
3. **委派/计划工具**:`vv/dispatches/primary_tools.go` ——
   - `PlanExecutor` 接口 + `DelegateToolName()` + `RegisterDelegateTools()` + `RegisterPlanTaskTool()`。
   - `delegate_to_<agent>` handler 在调 sub-agent 前 `IncrementDepth(ctx)`。
   - `plan_task` handler 同样递增深度,并在调 `PlanExecutor.RunPlan` 前做 goal/steps 空值校验。
4. **Dispatcher**:
   - 新增 `primaryAssistant` 字段、`WithPrimaryAssistant` Option、`SetPrimaryAssistant` post-construction setter。
   - `Run`/`RunStream` 在 `depth >= maxRecursionDepth` 之后、`fastPathClassify` 之前插入 `if d.primaryAssistant != nil` 分支 → 走 `runPrimary` / `runPrimaryStream`。
   - `runPrimaryStream` 包一对 `EventPhaseStart{Phase:"unified_primary"} / EventPhaseEnd{...}`,`phaseTracker` 统计工具调用和 token。
   - `RunPlan(ctx, *Plan, *schema.RunRequest)` public 方法满足 `PlanExecutor`(薄包装 `runPlan`);编译期断言 `_ PlanExecutor = (*Dispatcher)(nil)` 防签名漂移。
5. **Setup 接线**:`vv/setup/setup.go:New` 末尾新增 `if cfg.Orchestrate.Mode == OrchestrateModeUnified` 分支 → `buildPrimaryAssistant(...)` 构造 Primary 的 tool registry(read-only + ask_user + todo_write + 3 个 `delegate_to_*` + `plan_task`)→ 经 Permission/Truncate/Debug 包装 → Factory 生成 agent → `dispatcher.SetPrimaryAssistant(primary)`。classical 分支零侵入。
6. **向后兼容**:pre-M4 的所有测试不改动,`make test` 全绿。Primary 仅在显式 `mode=unified` 时构造,不出现在 `reg.Dispatchable()` 列表里,HTTP 层不自动暴露。
7. **事件形态**:unified 路径只发射单对 `unified_primary` phase 事件,不再走 `intent/execute/summarize`(HTTP 订阅者兼容层按设计留给 M5)。

## 改动清单

| 文件 | 行数 | 说明 |
|------|------|------|
| `vv/configs/config.go` | +30 | `OrchestrateConfig.Mode` + 常量 + `ValidateOrchestrateMode` + env override + Load 校验 |
| `vv/configs/config_test.go` | +94 | 5 个新用例:`ValidateOrchestrateMode` 表格测 + YAML + env + 未知值报错 + default |
| `vv/agents/primary.go` | 95 | 新文件 — `PrimaryAgentID`、`PrimarySystemPrompt`、`RegisterPrimary` |
| `vv/agents/primary_test.go` | 97 | 新文件 — 3 个用例 descriptor/Factory/prompt-drift guard |
| `vv/dispatches/primary_tools.go` | 278 | 新文件 — `PlanExecutor`、`DelegateToolName`、`RegisterDelegateTools`、`RegisterPlanTaskTool`,含深度递增与参数校验 |
| `vv/dispatches/primary_tools_test.go` | 387 | 新文件 — 10 个用例覆盖 schema/handler/错误路径/深度 |
| `vv/dispatches/primary.go` | 119 | 新文件 — `runPrimary`、`runPrimaryStream`、`RunPlan`、`PrimaryPhase`、`PrimaryAgentName` + 接口断言 |
| `vv/dispatches/primary_test.go` | 217 | 新文件 — 5 个 dispatcher 级用例(Run/Stream/Setter/PlanExecutor 合约) |
| `vv/dispatches/dispatch.go` | +43 | `primaryAssistant` 字段、`WithPrimaryAssistant`、`SetPrimaryAssistant`、`Run`/`RunStream` 早返回分支 |
| `vv/setup/setup.go` | +95 | unified-mode 分支 + `buildPrimaryAssistant` 辅助函数 |
| `vv/setup/setup_test.go` | +53 | 2 个 smoke 测 `mode=unified` / `mode=classical` |
| `vv/integrations/dispatches_tests/dispatches_tests/primary_assistant_integration_test.go` | 335 | 新文件 — 4 个端到端:AnswersDirectly / DelegatesToCoder / PlanTask / DisabledInClassicalMode |

**合计新增代码约 1830 行**(约 60% 是测试)。

## Evidence of done

- `cd vv && make test` → 所有 34 个包全绿(含 23 个 integration test package),coverage 维持不变。
- `cd vv && make lint` → `0 issues`。
- `cd vv/integrations/dispatches_tests && golangci-lint run` → `0 issues`。
- `go test ./dispatches/ -run "Primary|TestRun_UnifiedMode|TestSetPrimaryAssistant|TestRunStream_UnifiedMode|TestRegisterDelegateTools|TestRegisterPlanTaskTool|TestDelegateToolName" -v` → 20 个用例全通过。
- `go test ./integrations/dispatches_tests/dispatches_tests/ -run TestPrimary_ -v` → 4 个 E2E 用例全通过。

关键断言:
- `TestPrimary_AnswersDirectly`:mockLLM.callCount == 1,所有 sub-agent ranCount == 0 → "1 LLM call answered" 承诺成立。
- `TestPrimary_DelegatesToCoder`:LLM 2 次(tool call + 最终文本),coder.called == true,其余 sub-agent == false → depth 递增 + tool 注册链路正确。
- `TestPrimary_PlanTask`:researcher + coder 均运行,reviewer 不跑 → plan_task → `Dispatcher.RunPlan` → DAG 整链路打通。
- `TestPrimary_DisabledInClassicalMode`:未 `SetPrimaryAssistant` 时走 M2 unified_intent 路径 → 向后兼容证明。
- `TestRunStream_UnifiedMode_EmitsUnifiedPrimaryPhase`:只发出 `[start:unified_primary, end:unified_primary]` 两个 phase 事件,无 intent/execute/summarize 残留。

## 与计划的偏差

1. **`buildPrimaryAssistant` 的 registry 复用策略**:spec 里说用 `agents.RegisterPrimary(reg)` 往主 registry 注册。实际实现用了**一个一次性的 `registries.New()`** 只为取 descriptor 的 Factory,原因:主 reg 在 `setup.New` 开头就封闭了(`Dispatchable()` 列表已被 sub-agent 循环消费),再往里加非 dispatchable 的 primary 纯属脏数据,还会让主 reg 的 `All()` 返回不稳定。一次性 reg 获取 Factory 之后扔掉,语义最干净。未来若需要把 Primary 暴露给 HTTP 层,再显式注册到主 reg 也行。
2. **`todoStore` + `todoDisabled` 传入签名**:spec 未明说。实际实现把这两者作为参数传进 `buildPrimaryAssistant`,保证 Primary 的 todo_write 与 sub-agent 共享同一 store。
3. **`setup_test.go` 的 `TestNew_UnifiedMode_AttachesPrimary` 断言深度**:因为 `Dispatcher.primaryAssistant` 是 unexported 字段,setup 包无法直接读取。改为 smoke 测 + 完整行为断言挪到 integration 测试(后者能通过 mockLLM.callCount 间接验证 primary 已接入)。
4. **`vv/dispatches/dag.go` 未改**:spec 提到可能要抽取 `runPlan` 公共逻辑。实际发现 `runPlan` 已经是私有方法 + 接受所有需要的参数,直接在 `primary.go` 里加一个 thin public `RunPlan` 包装即可,零重构。
5. **未处理 `depth >= maxRecursionDepth` 下的 delegate 退化**:spec 的 Risks 一节提过。实现里 **`delegate_to_*` handler 不做 depth check**,而是让 `IncrementDepth(ctx)` 后 sub-agent 内部的 Dispatcher 去触发 fallback(`Run` 起点就有 `if depth >= maxRecursionDepth` 保护)。这等价行为,更节省一次判断;测试 `TestRegisterDelegateTools_HandlerIncrementsDepth` 覆盖到这一点。

## M5 衔接建议

1. **默认切 unified**:当前 `mode=""` 默认走 classical。M5 可把默认切到 unified 并保留 `mode=classical` 兜底。需要在 YAML migration 脚本里把未设 `mode` 的用户保留一次 `explicit classical` 保底。
2. **HTTP legacy event shim**:已有 HTTP 消费者在 `EventPhaseStart{Phase:"intent"}` 上挂可视化。加一个可选 shim:当 unified 路径生效时,把 `unified_primary` 映射为 legacy 的 `intent + execute` 事件对,便于无痛迁移。
3. **Chat agent 下线**:M4 的 Primary 已经吞下 chat 的职责。classical 下线后 chat sub-agent 可以从 `RegisterChat` 去掉(但写代码时要和 `subAgents["chat"]` 的 fallbackAgent 配合好——先替换 fallback 为 primary 或 coder 再删)。
4. **Explorer phase 下线**:同上。Primary 自带 read/glob/grep,Explorer sub-agent 的独立 phase 在 unified 下不存在,可合并。
5. **Primary 的 bash 写工具**:本期严禁写工具;M5 可以加可选开关 `primary.allow_bash` 让高级用户把 Primary 升级为"轻度 coder",但默认仍关。
6. **基准集**:spec 提到的 token/latency 基准测集在 `vv/integrations/` 下新增 `golden_tests/` 子包。M5 前把 greeting/simple-math/simple-read/simple-edit/multi-step-refactor 五条 golden case 跑一轮,作为 mode=unified 切默认的回归闸门。

## Resume Anchor

所有工件已落盘:
- Spec:`changes/2026/04/24/2026-04-24-005-m4-primary-assistant/spec.md`
- Result:`changes/2026/04/24/2026-04-24-005-m4-primary-assistant/result.md`(本文件)
- 新增代码:`vv/{agents,dispatches,setup,configs,integrations/dispatches_tests/dispatches_tests}/` 各文件

下一轮若继续推进:直接切 M5 或补 golden 基准集。
