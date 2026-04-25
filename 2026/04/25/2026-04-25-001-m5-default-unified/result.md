# M5 — 默认切 Unified + Legacy Shim + Golden 基准集 · Result

> Session: `2026-04-25-001-m5-default-unified`
> Status: Implemented
> Date completed: 2026-04-25
> Spec: `changes/2026/04/25/2026-04-25-001-m5-default-unified/spec.md`

---

## Done

按 spec 的 Done Contract 全部兑现。

### G1 · 默认 `mode` 从 classical 切到 unified

- `ValidateOrchestrateMode("")` 现返回 `OrchestrateModeUnified`(M4 及以前是 `classical`)。常量、注释、测试全部同步更新。
- `Load` 在探测到 "YAML 文件存在 + 没显式写 `orchestrate.mode` + 没设 `VV_ORCHESTRATE_MODE`" 时,发一条 `slog.Info` 迁移提示,引导用户把 `orchestrate.mode: classical` 写进 `~/.vv/vv.yaml` 以锁回 pre-M5 行为。
- 新增辅助函数 `yamlHasExplicitOrchestrateMode` — 用 `map[string]any` 二次解析判断,兼容注释、anchor、任意缩进。
- 环境变量 `VV_ORCHESTRATE_MODE=classical` 继续覆盖 YAML,对容器/CI 场景零改配置回退。
- 显式 `orchestrate.mode: classical` 配置**逐字节**与 M4 行为等价,不触发迁移日志。

### G2 · HTTP legacy phase-events shim

- 新增 `configs.OrchestrateConfig.LegacyPhaseEvents bool`(默认 false) + env `VV_ORCHESTRATE_LEGACY_PHASE_EVENTS`。
- 新增 `dispatches.WithLegacyPhaseEvents(bool) Option` + Dispatcher 字段 `legacyPhaseEvents`。
- `runPrimaryStream` 在开关打开时走新函数 `runPrimaryStreamLegacy`,发射事件序列为 `[start:intent, end:intent, start:execute, end:execute]`,**完全不发** `unified_primary`(避免双写)。关闭时(默认)与 M4 行为逐字节等价。
- `intent` phase 瞬时结束(duration ~0,summary 明示 "unified mode: intent absorbed into Primary"),`execute` phase 包住真实的 Primary 流并聚合 tracker 的 tool-call / token 计数 — HTTP 成本面板只需要订阅 `execute` phase 就能拿到与 classical 同形的指标。

### G3 · Unified 下 fallbackAgent 切 Primary(含 R2 缓解)

- 新增 `Dispatcher.SetFallbackAgent(agent.Agent)` 公有 setter,与 `SetPrimaryAssistant` 对称。
- `setup.go` unified 分支新增 `buildFallbackPrimary(...)`,构造一个**无工具**的退化 Primary(同系统提示 + `MaxIterations=1` + `ToolRegistry=nil`),通过 `dispatcher.SetFallbackAgent(fallbackPrimary)` 注入。
  - 关键安全属性:退化 Primary 没有 `delegate_to_*` / `plan_task`,`depth >= maxRecursionDepth` 早返回路径上再进入 Primary 也不会触发新一轮递归 — R2 风险彻底堵死。
- Classical 路径分支**未改动**,`WithFallbackAgent(subAgents["chat"])` 依旧。
- 新增测试:
  - `TestSetFallbackAgent_PostConstruction_AttachesAgent`
  - `TestRun_UnifiedMode_DepthExceeded_UsesPrimaryFallback`(构造 `depth=maxDepth`,断言 `chat.ranCount == 0`、`primary.ranCount == 0`、`fallback-primary.ranCount == 1`)。

### G4 · Post-landing 文档

- 在 `doc/design/enhance-prompt-understand-and-solve.md` 追加 `§8 M5 Post-landing notes`,含四小节:
  - 8.1 默认行为与迁移
  - 8.2 HTTP legacy event shim
  - 8.3 unified 模式下"不再被调用"的组件(chat / explorer / intent / summarize — 带"彻底删除的前置条件"说明)
  - 8.4 下一个里程碑(M6 草案)
- 代码层零改动 — 仅文档。

### G5 · Golden 基准集(`vv/integrations/golden_tests/golden_tests/`)

- 新建 package + 2 个测试文件。
- Harness 设计:`countedAgent` 包装器可包任意 `agent.Agent`(stub 或 real taskagent),统一按调用次数计数;`runClassical` / `runUnified` 分别构造两种模式的 dispatcher,共享 mock LLM。
- 5 条 case,每条同时跑 classical + unified:

| Case | Classical LLM calls | Unified LLM calls | 断言 |
|------|--------------------|--------------------|------|
| `Greeting_Hello` | 2 (intent + real chat taskagent) | 1 (Primary inline) | **unified < classical ✅** |
| `SimpleMath_Calc` | 2 (intent + real chat taskagent) | 1 (Primary inline) | **unified < classical ✅** |
| `SimpleRead_ExplainFile` | 1 (intent + stub researcher) | 1 (Primary inline) | 结构化:sub-agent 被调用次数正确 |
| `SimpleEdit_DelegateToCoder` | 1 (intent + stub coder) | 2 (Primary→delegate→final) | 结构化:coder 被调 1 次,其余不跑 |
| `MultiStepRefactor_Plan` | 1 intent + 2 stub sub-agents | 2 (Primary→plan_task→final) | 结构化:researcher + coder 各 1 次,reviewer 不跑 |

- 赚 LLM 调用数的场景是 greeting / simple-math(1 次 vs 2 次);委派/规划路径上 unified 多付 1 次 Primary 决策成本 — 这是 design doc §3.4 的已知权衡,honest-by-construction。

---

## 改动清单

| 文件 | 行数 | 说明 |
|------|------|------|
| `vv/configs/config.go` | +63 | `LegacyPhaseEvents` 字段 + env override + M5 默认切换 + `yamlHasExplicitOrchestrateMode` + `slog.Info` 迁移提示 |
| `vv/configs/config_test.go` | +61 −4 | 更新空值默认断言 + 新增 `TestLoad_OrchestrateModeDefaultsToUnified` + `TestLoad_OrchestrateModeExplicitClassicalPreserved` + `TestYAMLHasExplicitOrchestrateMode` |
| `vv/dispatches/dispatch.go` | +29 | `legacyPhaseEvents` 字段 + `WithLegacyPhaseEvents` Option + `SetFallbackAgent` setter |
| `vv/dispatches/primary.go` | +79 | `runPrimaryStream` 按 shim 开关分派 + 新函数 `runPrimaryStreamLegacy` |
| `vv/dispatches/primary_test.go` | +165 | 3 个新用例:`TestRunStream_UnifiedMode_LegacyShim_EmitsIntentAndExecutePhases` + `TestSetFallbackAgent_PostConstruction_AttachesAgent` + `TestRun_UnifiedMode_DepthExceeded_UsesPrimaryFallback` |
| `vv/setup/setup.go` | +55 | unified 分支末尾加 `buildFallbackPrimary` 构造 + `SetFallbackAgent` 注入;`dispatcherOpts` 加 `WithLegacyPhaseEvents` |
| `vv/integrations/golden_tests/golden_tests/golden_helpers_test.go` | 143 | 新文件 — `sequentialMockLLM` / `callTrackingAgent` / `newGoldenRegistry` / 3 个 response 构造器 |
| `vv/integrations/golden_tests/golden_tests/golden_cases_test.go` | 494 | 新文件 — `countedAgent` 包装 + harness + 5 个 E2E case |
| `doc/design/enhance-prompt-understand-and-solve.md` | +40 | §8 M5 Post-landing notes |

**合计新增代码约 1029 行**(约 60% 是测试 + 文档)。

---

## Evidence of done

### 单元 + 集成测试

```
cd vv && make test
... (全部 36 个 test package 全绿)
ok  	github.com/vogo/vv/integrations/golden_tests/golden_tests	3.014s
```

### Lint

```
cd vv && make lint
golangci-lint run
0 issues.

cd vv/integrations/golden_tests && golangci-lint run ./...
0 issues.
```

### 关联模块无回归

```
cd vage && make test  # ✅ 全绿
cd aimodel && make test  # ✅ 全绿
```

### Golden 基准输出(关键数据)

```
greeting:     classical llmCalls=2, unified llmCalls=1   # 1:2 ✅
simple-math:  classical llmCalls=2, unified llmCalls=1   # 1:2 ✅
simple-read:  classical llmCalls=1 researcher=1; unified llmCalls=1
simple-edit:  classical coder=1 llmCalls=1; unified coder=1 llmCalls=2
multi-step:   classical R=1 C=1 Rev=0; unified R=1 C=1 Rev=0
```

greeting / simple-math 展示了 M5 切默认的量化收益 — 1 次 LLM 调用对比 classical 的 2 次。

### 关键断言

- `TestValidateOrchestrateMode` 空值输入期望 `unified`(M4 是 `classical`) — 断言了 M5 默认翻转。
- `TestLoad_OrchestrateModeDefaultsToUnified` — YAML 中没写 mode 的用户启动后 mode 为 unified。
- `TestLoad_OrchestrateModeExplicitClassicalPreserved` — 显式 classical 不被默认翻转覆盖。
- `TestRunStream_UnifiedMode_EmitsUnifiedPrimaryPhase`(M4 遗留) — 关闭 shim 时事件序列与 M4 一致,零回归。
- `TestRunStream_UnifiedMode_LegacyShim_EmitsIntentAndExecutePhases` — 开启 shim 时事件序列恰为 `[start:intent, end:intent, start:execute, end:execute]`,且不含 `unified_primary`。
- `TestRun_UnifiedMode_DepthExceeded_UsesPrimaryFallback` — depth 爆表时走 fallback primary(无工具),不走 chat、不走有工具的 primary — R2 缓解成立。
- `TestPrimary_DisabledInClassicalMode`(M4 遗留) — 未 `SetPrimaryAssistant` 时走 M2 unified_intent 路径 — classical 回退兼容。

---

## 与计划的偏差

1. **`fallbackPrimary` 独立工厂 vs 复用 `buildPrimaryAssistant`**:spec 只提到"构造退化 Primary"。实际实现了一个独立函数 `buildFallbackPrimary`,原因是它的 FactoryOptions 差异点有 3 处(ToolRegistry=nil、MaxIterations=1、没有 todoStore/todoDisabled 参数),把条件分支塞到 `buildPrimaryAssistant` 里反而更晦涩。两个函数都走 `agents.RegisterPrimary + desc.Factory` 的同一个 descriptor,系统提示身份一致。
2. **`yamlHasExplicitOrchestrateMode` 选择 `map[string]any` 而非字符串匹配**:spec 里写了"简化方案:重新解析 YAML 到一个 `map[string]any`";最终按推荐路径落地,覆盖 `orchestrate.mode` 带注释 / 空字符串 / 任意缩进的所有变体 — `TestYAMLHasExplicitOrchestrateMode` 覆盖 7 个 variant。
3. **迁移 `slog.Info` 的位置**:spec 里说"在 `ValidateOrchestrateMode` 前 / 后",实际放在 `ValidateOrchestrateMode` 归一化**之后**,这样归一化 + 迁移判定解耦,`cfg.Orchestrate.Mode` 在 log line 里直接可读。
4. **G5 Golden harness 没按 spec 画的"统一 harness"结构写**:spec 里说"一个 `goldenHarness` struct 封装 mock LLM + registries + subAgents + dispatcher 构造器,支持 `.WithMode(...)` 切换"。实际拆成两个函数 `runClassical(...)` / `runUnified(...)` 加一个 `countedAgent` 包装器。原因:两种模式的构造差异太大(classical 需要 `WithIntentSystemPrompt`,unified 需要 `SetPrimaryAssistant` + `RegisterDelegateTools` + `RegisterPlanTaskTool` + `taskagent.New`),用一个链式 builder 会把 "mode switch" 变成两个完全不同的代码分支,不如两个函数清晰。
5. **未改 `setup/setup_test.go` 的默认 mode 断言**:走一遍 setup 测试发现 `TestNew_UnifiedMode_AttachesPrimary` / `TestNew_ClassicalMode_NoPrimary` 都显式设置了 `cfg.Orchestrate.Mode`,**没有**隐式依赖默认值 — 所以不需要改这两个测试。
6. **HTTP 集成测试 `httpapis_tests/http_tests` 没跑新 shim 场景**:spec 隐含建议这里加一个 shim 开启的端到端测试。实际发现 `dispatches/primary_test.go:TestRunStream_UnifiedMode_LegacyShim_*` 已经在 dispatcher 层覆盖了事件序列合约,HTTP 层本身只是透传 SSE — 再加一层 HTTP test 只是重复断言。若后续有 HTTP 可视化消费者上报集成 bug,再补。

---

## M6 衔接建议

1. **`primary.allow_bash` 开关**(衔接建议 #5 推迟项):在 Primary 的工具集加一个可选 bash 工具,让"一行命令就能做完的事"不用走 delegate_to_coder 再多一轮 LLM。默认仍关,通过 `configs.OrchestrateConfig.PrimaryAllowBash` 开。
2. **彻底移除 chat / explorer / intent / summarize 代码**:在 M5 经过一个迭代周期、且 issue tracker 没人要求 classical 兜底后动手。三个注意点:(a) `dispatches/intent.go` 里面有 `recognizeIntentUnified` 被 M2 路径复用,不是死代码,拆的时候要挑出来;(b) `fastpath.go:71` 的 `Agent:"chat"` 硬编码需要改成 `fallbackAgentName()`;(c) 先改 `fallbackAgent` 的接线代码能处理 `nil` chat 的场景,再删 chat 本体。
3. **Golden baseline 接真实 LLM 的 CI job**:当前 golden 用 mock LLM 做回归闸门。M6 可以加一个 weekly-cron CI 跑同一组 case 的**真实 LLM** 版,把实际 latency / token 数据落盘,作为"unified 比 classical 省多少"的真实性能基准。
4. **`legacy_phase_events` 的弃用路径**:M5 加的 shim 是过渡兼容层。在 HTTP 消费者都迁移到 unified 事件后,可以在 M6/M7 把这个字段 deprecate(加 slog.Warn),M7 再移除。
5. **fallback path 的 summarize 行为**:现在 unified 下 depth 爆表走退化 Primary,返回的是纯文本。classical 有 summarize phase,unified 没有 — 后续若发现 HTTP 消费者依赖 summarize 事件,需要在退化 Primary 出口补一个同形 phase。

---

## Resume Anchor

所有工件已落盘:
- Spec: `changes/2026/04/25/2026-04-25-001-m5-default-unified/spec.md`
- Result: `changes/2026/04/25/2026-04-25-001-m5-default-unified/result.md`(本文件)
- 新增代码: `vv/{configs,dispatches,setup,integrations/golden_tests/golden_tests}/` 各文件
- 文档更新: `doc/design/enhance-prompt-understand-and-solve.md` §8

下一轮若继续推进:直接切 M6 的任一子项(`primary.allow_bash` / 彻底删代码 / 真实 LLM golden CI)都是独立小任务,可各自单开一个 dev session。
