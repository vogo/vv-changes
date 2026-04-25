# M7 — Classical 路径彻底移除 + Deprecated Shim 清理 + Golden Drift Gate · Spec

> Session: `2026-04-25-003-m7-cleanup`
> Status: Draft (awaiting approval)
> Date: 2026-04-25
> Mode: sdd-lite `deep`
> Spec is Truth · No Approval, No Execute

---

## 1. 任务复述

来源:`changes/2026/04/25/2026-04-25-002-m6-implementation/result.md` §"M7 衔接建议" 的 5 项总览表。

合并为 1 个 dev session 一次性交付,本期形态由用户在前置对话中拍板:

| ID | 工作包 | 形态 | 来源/决策 |
|----|--------|------|-----------|
| **M7-1** | 删 classical 兜底路径 + `OrchestrateModeClassical` 常量 + `intent.go` | 全删 | result.md §M7-1 |
| **M7-2** | 删 `LegacyPhaseEvents` 字段 / Option / shim / env / Warn / 测试 | 全删 | result.md §M7-2 |
| **M7-3** | 真实 LLM golden drift gate(`±50%` 宽窗 + 单点 baseline) | **Option b** —— 单点 baseline + ±50% | 前置对话 |
| **M7-4** | 退化 Primary fallback 路径加**静态** summarize 占位 phase | **Option d** —— 推翻 G4 决策,加 0-LLM 占位 | 前置对话 |
| **M7-5** | 删 `agents/compat.go` + 拆分处理 35 个 caller | **Option Y(细化)** —— 删 24 tests,port 7 callsites | 前置对话 + 侦察细化 |

## 2. 核心目标(Loop Anchor)

vv 的派发路径在代码层只剩**一条**:Primary Assistant + delegate_to/plan_task 工具集 + 退化 Primary 兜底。任何 classical 残留(intent recognize、explorer/planner 旧路径、`OrchestrateModeClassical` 常量、`LegacyPhaseEvents` shim、`agents.Create` / `NewOrchestratorAgent` 这两个 deprecated builder)在主仓代码与测试中**全部消失**。同时:

- HTTP 消费者订阅 fallback 路径时,SSE 事件流形态与主路径一致(M7-4 静态占位 phase)。
- 真实 LLM golden 有 drift 闸门(M7-3 ±50% 宽窗,后续收紧)。

判断完成的唯一锚点:`grep -rn 'OrchestrateModeClassical\|LegacyPhaseEvents\|legacyPhaseEvents\|legacy_phase_events\|VV_ORCHESTRATE_LEGACY_PHASE_EVENTS\|agents\.Create\|agents\.NewOrchestratorAgent\|legacyExplorerSystemPrompt\|BuildIntentSystemPrompt\|recognizeIntent\b' vv/` **零命中**(release notes / changelog 除外),且 `cd vv && make test && make lint` 全绿。

## 3. 边界与硬约束

### 3.1 范围内
- `vv/dispatches/`:删 `intent.go`、`intent_test.go`,改 `primary.go` 两处 nil-fallback,删 `dispatch.go` 中 `WithLegacyPhaseEvents` Option / `legacyPhaseEvents` 字段,改 `forwardSubAgentStream` 的 depth-exceed 路径(M7-4 占位 phase)。
- `vv/configs/config.go`:删 `OrchestrateModeClassical` 常量、`LegacyPhaseEvents` 字段、env override,简化 `ValidateOrchestrateMode`。
- `vv/setup/setup.go`:删 `dispatches.WithLegacyPhaseEvents(...)` 调用 + `//nolint:staticcheck` 标记。
- `vv/agents/compat.go`:整文件删除(包含 `legacyExplorerSystemPrompt`)。
- `vv/integrations/agents_tests/agents_tests/`:删 3 个 builder-API 测试文件(orchestrator / orchestrator_advanced / dynamic);port `agents_tool_profile_test.go` 到 `setup.New`(并删除 `ChatHasNoTools` 死用例)。
- `vv/integrations/cli_tests/cli_tests/cli_permission_test.go` / `cli_wiring_test.go` / `setup_tests/wiring_tests/wiring_test.go`:port 4 个 `agents.Create` callsite 到 `setup.New`。
- `vv/integrations/golden_tests/real_llm_tests/`:加 drift gate 逻辑 + checked-in `baseline_committed.json`。
- `vv/CLAUDE.md`:刷新章节(chat/explorer/intent.go classical pipeline 等过期描述)。
- `doc/design/enhance-prompt-understand-and-solve.md`:加 §10 "M7 Post-landing notes",标 M6 §9 章节中相应"M7+ 才彻底清"的项已兑现;§9.4 改"已补静态占位 phase,见 commit XXX"。

### 3.2 范围外(本次不做)
- M7-3 的"4-8 周数据采集"——本期只交付**单点** baseline + ±50% 宽窗,后续手动收紧到 ±20% 不在本 session。
- M7-4 不再保留任何"按需补 LLM 调用"的钩子——本期决策**永远**走静态占位,不引入 summarizer。
- 重构 dispatcher 的递归深度模型(`maxRecursionDepth`)——保持现状。
- vage / aimodel 层零改动。
- HTTP API 公共契约的破坏性变更——SSE 事件 phase 名只新增/收紧含义,不删除。
- 引入新 agent 类型 / 重写 fastpath。

### 3.3 硬约束
- **公共 API 收紧 = 破坏**:删 `WithLegacyPhaseEvents`、`OrchestrateModeClassical`、`agents.Create`、`agents.NewOrchestratorAgent`、`agents.BuildIntentSystemPrompt` 都属于公共 API 移除。M6 已经发了 deprecation runtime Warn(本期已是第 2 个迭代),M7 删除符合 "deprecation 1 周期 → 删除" 节奏。
- **YAML 字段下线静默处理**:`legacy_phase_events` / `orchestrate.mode=classical` 这些过期 YAML 键,在 `Load` 中**不能**报错,只能 `slog.Warn` "已忽略,M7 起被永久移除"——避免存量 vv.yaml 启动崩溃。
- **测试净覆盖不退化**:删 24 个 builder-API 测试前,必须验证以下能力在保留测试中已有等价覆盖(详见 §5.5),若发现空缺必须先补。
- **真实 LLM golden 在缺 API key 时仍 Skip**:M7-3 增加的 drift gate **不能让无 key 环境下的 CI 红**。
- **classical 残留扫描需零命中**:Loop Anchor 中的 grep 命令是"完成"的硬证据,任何遗漏(包括注释、godoc、changelog 之外的文档)都视为未完成。

## 4. 行业方案对比与取舍(deep 必备)

### 4.1 M7-1 删除节奏

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| α | 单 commit 全删:primary.go nil-fallback、intent.go、常量、`fallbackRun` 中的 classical-only 路径一次拿掉 | review 一目了然,语义切换原子 | diff 巨大,任一处 build 失败需整体回退 |
| β | 三步走:(1) primary.go nil-fallback 改 error;(2) 删 intent.go;(3) 删常量 | 每步可独立绿测,回滚颗粒细 | commit 数多,但同 dev session 内可顺序提交 |
| γ | 仅改 nil-fallback 为 error,intent.go 保留(再 deprecation 1 周期) | 风险最低 | 没真正完成 M7-1,且 intent.go 没有 caller 了不删等于死代码 |

**取**: β。同 dev session 内顺序提交,Step 1/2/3 各自一次 `cd vv && go test ./... && make lint`。

### 4.2 M7-3 baseline 数据策略

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| I | **取真实单点**(本机跑一次 5 case,把 latency / total_tokens 直接写 `baseline_committed.json`)+ ±50% 宽窗 | 数据真实,gate 能 catch 50% 以上的 regression | 单点方差大,某些 case 可能首次跑就贴近上限 |
| II | **保守静态值**(查 OpenAI / Anthropic 公开 benchmark 估一个保守 token / latency 范围,写死) | 不依赖本地环境 | 不真实,gate 无意义 |
| III | **mock baseline**(用 mock LLM 跑,假装真实) | 永远绿 | 等于没 gate |

**取**: I。前置对话已选 b。执行时本机跑一次 `make` 等价的 real_llm 测试(需 `VV_LLM_API_KEY`),写 baseline 后 commit;若用户当下不便提供 API key,**降级为先写代码框架 + 占位 0 值 baseline,gate 在 baseline 全 0 时跳过**(自我兜底,不阻塞 spec 推进)。

### 4.3 M7-4 占位 phase 形态

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| A | 静态字符串 `"fallback: no summarization"` | 0 LLM 调用,与"廉价兜底"一致 | Summary 内容空洞 |
| B | 截取 fallback agent 最后一段 stream 输出作为 Summary | 内容相对真实 | 实现复杂,需要 stream 缓冲;若 agent 输出本身已是 markdown 长文,截取规则要定 |
| C | 调 summarizer agent(主路径行为) | 与主路径完全一致 | **本期已被用户拒绝**(违反 G4 决策中"廉价兜底"原则)—— Option d 明确选 0-LLM |

**取**: A。简洁、零额外开销、消费者拿到的是一个"形态一致但内容明确标识 fallback"的 phase 事件,UI 侧逻辑自然分流。Summary 文本 = `"fallback path: no summarization performed"`。

### 4.4 M7-5 删 vs port 的细分判据

侦察发现 35 个 callsite 跨 7 文件,**整删整 port 都不是最优**。判据:

| 判据 | 整删 | Port |
|------|------|------|
| 测的核心是"deprecated builder API 自身行为"(orchestrator 内部 routing、plan execution、dynamic agent injection、fallback-on-LLM-error 这些 dispatcher 已有等价测试的能力) | ✅ | ❌ |
| 测的是"产品行为"且无等价覆盖(coder/researcher/reviewer 各自的 tool profile assignment) | ❌ | ✅ |
| 测的是 CLI permission 链路(`WrapRegistryWithPermission` × `agents.Create`) | ❌ | ✅ |

**应用结果**:
- **DELETE 3 文件 / 24 tests**:`agents_orchestrator_test.go`(6)/`_advanced_test.go`(6)/`agents_dynamic_test.go`(12)。
- **PORT 4 文件 / 7 callsites**:`agents_tool_profile_test.go`(原 4 tests,删 ChatHasNoTools,余 3 ports);`cli_permission_test.go`(1);`cli_wiring_test.go`(2);`wiring_test.go`(1)。

§5.5 给详细 port 模板。

## 5. 设计决定

### 5.1 M7-1 · 删 classical 路径 + 常量

#### 5.1.1 primary.go nil-fallback 改 error

侦察确认,**真正的 classical 兜底**在 `primary.go`:

```go
// primary.go:24 (Run)
if d.primaryAssistant == nil {
    return d.fallbackRun(ctx, req, nil)
}
// primary.go:60 (RunStream)
if d.primaryAssistant == nil {
    // ... fallback path
}
```

改为:

```go
if d.primaryAssistant == nil {
    return nil, fmt.Errorf("vv: dispatcher requires a Primary Assistant (call SetPrimaryAssistant or WithPrimaryAssistant)")
}
```

stream 版本同形(send 一个 error event 后 return)。

`dispatch.go:385/447` 的 `if d.primaryAssistant != nil` 守卫**保留** —— 它们包的是主路径优先尝试 Primary 的逻辑,在 Primary 必然存在(已 nil-check)的世界里这条 if 永远进 true 分支,后续清理工作可在 M7+ 再做(本期不强求,避免 diff 蔓延);若清理简单(只是去掉一层 indent),则在 Step 1 末尾顺手做。

#### 5.1.2 intent.go 整文件删除

intent.go 200 行(M6 已瘦身),**当前唯一外部引用**是单元测试 `intent_test.go`。primary.go nil-fallback 改 error 后,`fallbackRun` 不再走 intent recognize 路径,intent.go 与 intent_test.go 一并删除。

具体删除符号清单(grep 确认零外部引用):
- `recognizeIntent`、`recognizeIntentStream`、`recognizeIntentDirect`
- `useUnifiedIntent`、`fallbackIntent`、`buildIntentSummary`
- `BuildIntentSystemPrompt`(**公共导出,需在 release notes 标记 breaking**)
- `intentSystemPromptTemplate`(常量)

#### 5.1.3 常量与 ValidateOrchestrateMode

```go
// configs/config.go (BEFORE M7)
const OrchestrateModeUnified   = "unified"
const OrchestrateModeClassical = "classical" // Deprecated: M6+ → unified

func ValidateOrchestrateMode(mode string) (string, error) {
    switch mode {
    case "":                       return OrchestrateModeUnified, nil
    case OrchestrateModeUnified:   return OrchestrateModeUnified, nil
    case OrchestrateModeClassical: slog.Warn(...); return OrchestrateModeUnified, nil
    default:                       slog.Warn(...); return OrchestrateModeUnified, nil
    }
}

// AFTER M7
const OrchestrateModeUnified = "unified"

func ValidateOrchestrateMode(mode string) (string, error) {
    switch mode {
    case "", OrchestrateModeUnified:
        return OrchestrateModeUnified, nil
    default:
        slog.Warn("vv: orchestrate.mode is deprecated as of M7; the field is ignored and the unified pipeline is always used", "received", mode)
        return OrchestrateModeUnified, nil
    }
}
```

注意 `(string, error)` 签名保留以避免破坏 caller —— `error` 字段永远 `nil`。godoc 加 deprecation 备注:"orchestrate.mode is retained for YAML backward compatibility only; the field is silently normalized."

#### 5.1.4 测试影响
- 删 `intent_test.go` 整文件
- `configs/config_test.go`:`TestLoad_OrchestrateModeExplicitClassicalRoutesToUnified` / `TestValidateOrchestrateMode` 等保留 classical 引用的用例改写或删除;改用 `"unknown_mode"` 作为 normalize-to-unified 的代表用例。
- `setup_tests` 中任何 `Mode: configs.OrchestrateModeClassical` 字面引用删除(M6 应已清,但执行时再 grep 确认)。

### 5.2 M7-2 · 删 `LegacyPhaseEvents` 整条 shim

侦察行号:

| 位置 | 操作 |
|------|------|
| `vv/dispatches/dispatch.go:94-99` | 删 `legacyPhaseEvents` 字段 |
| `vv/dispatches/dispatch.go:291-307` | 删 `WithLegacyPhaseEvents` Option(连同 deprecation Warn closure) |
| `vv/dispatches/primary.go:49-56` 注释 + `66-...` shim 分支 | 删 `if d.legacyPhaseEvents { return d.runPrimaryStreamLegacy(...) }`,然后删 `runPrimaryStreamLegacy` 函数本身 |
| `vv/configs/config.go:84-92` | 删 `LegacyPhaseEvents` 字段 + comment + yaml tag |
| `vv/configs/config.go:657-661` | 删 env override block(`VV_ORCHESTRATE_LEGACY_PHASE_EVENTS`) |
| `vv/setup/setup.go:298` | 删 `dispatches.WithLegacyPhaseEvents(...)` 调用 + `//nolint:staticcheck` 标记 |
| `vv/dispatches/primary_test.go:187-300` | 删 `TestUnifiedStream_RewritesPhasesUnderLegacyShim` + `TestWithLegacyPhaseEvents_DeprecationWarn` 整段 |

`Load` 中针对 `legacy_phase_events` YAML 键的兼容兜底:加一段最小代码,在 unmarshal 前先检测原始 YAML 树中是否有该键,有则 `slog.Warn("vv: orchestrate.legacy_phase_events is no longer supported as of M7; the field is ignored")`。

### 5.3 M7-3 · 真实 LLM Golden Drift Gate

#### 5.3.1 baseline_committed.json schema

```json
{
  "version": 1,
  "generated_at": "2026-04-25T...",
  "model": "claude-sonnet-4-6",
  "cases": {
    "Greeting_Hello":        {"latency_ms_p50": 1234, "total_tokens_p50": 500},
    "SimpleMath_Calc":       {"latency_ms_p50": 1500, "total_tokens_p50": 600},
    "SimpleRead_ExplainFile":{"latency_ms_p50": 4000, "total_tokens_p50": 1500},
    "SimpleEdit_DelegateToCoder": {"latency_ms_p50": 6000, "total_tokens_p50": 2200},
    "MultiStepRefactor_Plan":{"latency_ms_p50": 12000, "total_tokens_p50": 5000}
  },
  "tolerance_pct": 50
}
```

#### 5.3.2 loadBaseline + drift 比对

`real_llm_baseline_test.go` 加 helper:

```go
func loadBaseline(t *testing.T) (Baseline, bool) {
    data, err := os.ReadFile("baseline_committed.json")
    if err != nil { return Baseline{}, false }
    // ... unmarshal
}

// 每个 case 在 sanity check 后:
if base, ok := loadBaseline(t); ok {
    if c, found := base.Cases[caseName]; found && c.LatencyMsP50 > 0 {
        lower := float64(c.LatencyMsP50) * 0.5
        upper := float64(c.LatencyMsP50) * 1.5
        if float64(actual) < lower || float64(actual) > upper {
            t.Errorf("latency drift: case=%s actual=%dms baseline_p50=%dms (±50%% window=[%.0f, %.0f])",
                caseName, actual, c.LatencyMsP50, lower, upper)
        }
    }
    // total_tokens 同形
}
```

**关键**:`c.LatencyMsP50 > 0` 守卫确保占位 0 值 baseline(无数据时)不会假阳性失败。

#### 5.3.3 baseline 生成路径
- **理想**:本期执行 Step 9 时,本机有 `VV_LLM_API_KEY`,跑一次 `vv/scripts/run-golden-real-llm.sh`,把 5 case 的 latency / total_tokens 中位数(单点)写入 `baseline_committed.json`,commit。
- **降级**:如执行时无 key,写一个全 0 占位 baseline + Step 10 文档明确标注"M7 落地时未采到真实 baseline,首次跑 cron 后由维护者填入"。**降级路径下 gate 等效不生效,但代码结构已就位**。

#### 5.3.4 README 增量
`vv/integrations/golden_tests/README.md` 加 "Updating baseline_committed.json" 一节:何时更新(模型升级、prompt 大改、置信下降时)、如何更新(本机跑 → P50 计算 → commit)、宽窗调整 timeline(±50% → ±30% → ±20%,随数据稳定逐步收紧)。

### 5.4 M7-4 · Fallback 路径静态占位 summarize phase

#### 5.4.1 落点
`vv/dispatches/dispatch.go:441/492` 两处 depth-exceed 兜底:

```go
// BEFORE
if depth >= d.maxRecursionDepth {
    return d.forwardSubAgentStream(ctx, send, d.fallbackAgent, req, d.fallbackAgentName(), "", sessionID)
}

// AFTER
if depth >= d.maxRecursionDepth {
    if err := d.forwardSubAgentStream(ctx, send, d.fallbackAgent, req, d.fallbackAgentName(), "", sessionID); err != nil {
        return err
    }
    // M7 G4: 静态占位 summarize phase,与主路径事件流形态对齐;不调 summarizer agent。
    send(agent.StreamEvent{
        Type:    agent.StreamEventTypePhase,
        Phase:   "summarize",
        Summary: "fallback path: no summarization performed",
    })
    return nil
}
```

类似改动在 `Run` 路径(line 377-400 `fallbackRun`)的对应位置——需确认 fallbackRun 是否也需要 phase 事件(Run 是非流式,通常无 phase 事件,所以可能只在 RunStream 路径补)。**Step 4 执行时再次确认 Run vs RunStream 的 phase 事件契约**。

#### 5.4.2 测试
新增 `dispatch_test.go` 中:
- `TestRunStream_DepthExceeded_EmitsStaticSummarizePhase`:构造 depth >= maxRecursionDepth 的递归调用,断言事件流末尾有 `Phase: "summarize", Summary: "fallback path: no summarization performed"`。

#### 5.4.3 design 文档同步
`doc/design/enhance-prompt-understand-and-solve.md §9.4` 当前文本:"退化 Primary 路径不发 `summarize` phase ... 若 HTTP 消费者具体投诉再补"——M7 改为:"退化 Primary 路径在 M7 起发**静态** summarize phase(`Summary: \"fallback path: no summarization performed\"`,零 LLM 调用),消费者订阅时与主路径事件流同形。"

### 5.5 M7-5 · 删 compat.go + 测试拆分处理

#### 5.5.1 整文件删除清单(24 tests / 3 文件)

删除前**必须验证**以下能力在保留测试中已有等价覆盖。覆盖矩阵(执行时 grep 验证,任一空缺先补再删):

| 待删测试 | 等价保留覆盖 |
|---------|------------|
| `OrchestratorDirectDispatch` / `PlanExecution` / `FallbackOnInvalidJSON` / `FallbackOnEmptyPlan` / `FallbackOnInvalidAgent` / `ImplementsStreamAgent` | `dispatches/dispatch_test.go` 的 `TestRun_DirectDispatch` / `TestRunStream_DirectDispatch` / `TestFallbackOnInvalidJSON` 等 + `golden_tests` 端到端 case |
| `WorkingDirectoryCaptureAndPropagation` / `OrchestratorParallelStepExecution` / `PlanStepFailure` / `TokenUsageAggregation` / `ChatInPlanSteps` / `FallbackOnLLMError` | `vage/orchestrate/...` 已有 DAG 测试覆盖 parallel / failure / aggregation;`ChatInPlanSteps` 覆盖的 chat agent 已删,**无等价** —— 用例随 chat 一起死亡,可删 |
| 12 个 `DynamicAgents_*`(StaticOnlyPlanRegression / DynamicOnlyPlan / Mixed / ValidationFailures / ToolAccessLevel / DefaultFallback / CustomSystemPrompt / ORCH19 / BackwardCompat / NilToolRegistries / CreateWires) | dynamic agent injection 覆盖在 `vage` 层 + `vv/registries/` 层有单测;**ORCH19PrecedenceRule** 是产品规则需特别确认:若 vage 层无对应,补一个最小用例到 `dispatches/dynamic_test.go`(新建小文件);**ToolAccessLevelCorrectness** 同样需独立验证 |

#### 5.5.2 Port 模板(7 callsites)

`agents.Create(cfg, mock, reg, readOnlyReg, reviewReg, nil, nil)` →

```go
opts, err := setup.New(setup.Inputs{
    Config:           cfg,
    Completer:        mock,
    ToolRegistry:     reg,
    ReadOnlyRegistry: readOnlyReg,
    ReviewRegistry:   reviewReg,
})
require.NoError(t, err)
allAgents := opts.Agents
```

(`setup.Inputs` 等结构体名称按当前 setup.go 实际签名调整,执行时核对。)

`agents.NewOrchestratorAgent(...)` 调用点全在被整删的 3 个文件中,无需 port。

#### 5.5.3 `agents/compat.go` 删除
整文件删除,包括内联的 `legacyExplorerSystemPrompt`(283 行全部走人)。

#### 5.5.4 `agents_tool_profile_test.go` port 细节
- **DELETE** `TestIntegration_Agents_ChatHasNoTools`(chat agent 已删)
- **PORT** `CoderHasTools` / `ResearcherHasReadOnlyTools` / `ReviewerHasCorrectTools`:换 setup 入口,断言点不变(从 `allAgents["coder"]` 取出 agent → 检查其 tool registry → 工具集与 ProfileFull / ProfileReadOnly / ProfileReview 一致)。

### 5.6 文档收尾

- `doc/design/enhance-prompt-understand-and-solve.md`:加 §10 "M7 Post-landing notes",每个 G 项写 1 段结论。M6 §9 中相应"务实保留(M7+ 才彻底清)"段落改"已于 M7 兑现,见 §10"。
- `vv/CLAUDE.md`:刷新 §"Dispatch Pipeline" / §"Agent Registry" 章节,删除 chat / explorer / classical pipeline 描述;`intent.go` 提及全删。
- result.md 落盘(标 D1..DN evidence)。

## 6. Done Contract

| # | 检查 | 证据 |
|---|------|------|
| D1 | `vv/dispatches/intent.go` 与 `intent_test.go` 不存在 | `ls vv/dispatches/intent*.go` 空 |
| D2 | `primary.go` 的 nil-Primary 路径返回 error | `TestRun_PrimaryRequired_ReturnsError` + `TestRunStream_PrimaryRequired_ReturnsError` 通过 |
| D3 | `OrchestrateModeClassical` 常量不存在;`ValidateOrchestrateMode` 收敛为 unified-only + Warn | `grep -n 'OrchestrateModeClassical' vv/` 零;新单测 `TestValidateOrchestrateMode_NormalizesUnknown` |
| D4 | `LegacyPhaseEvents` 字段、`WithLegacyPhaseEvents` Option、`runPrimaryStreamLegacy`、env override 全删 | grep 零命中 |
| D5 | `setup.go:298` 的 `//nolint:staticcheck` 与 `WithLegacyPhaseEvents` 调用全删 | `grep -n 'staticcheck\|LegacyPhaseEvents' vv/setup/setup.go` 零 |
| D6 | YAML 包含 `legacy_phase_events: true` 或 `mode: classical` 时 Load 成功并发 Warn | 新单测 `TestLoad_StaleYAMLKeysAreWarned` |
| D7 | `agents/compat.go` 不存在 | `ls vv/agents/compat.go` 空 |
| D8 | `grep -rn 'agents\.Create\|agents\.NewOrchestratorAgent\|legacyExplorerSystemPrompt' vv/` 零(release notes 除外) | grep 零 |
| D9 | 24 个 builder-API 测试已删,等价覆盖经核查 | 删除前的核查清单回写到 result.md "M7-5 等价覆盖审计" 段 |
| D10 | `agents_tool_profile_test.go` port 完成,3 个 tool profile case 仍绿 | 新文件名 `setup_tests/profile_tests/profile_test.go`(或 inplace 改) |
| D11 | `forwardSubAgentStream` depth-exceed 路径发静态 summarize phase | 新单测 `TestRunStream_DepthExceeded_EmitsStaticSummarizePhase` |
| D12 | `real_llm_baseline_test.go` 加 drift gate;`baseline_committed.json` 存在(可为占位 0 值) | `ls vv/integrations/golden_tests/real_llm_tests/baseline_committed.json` |
| D13 | drift gate 在无 API key 时仍 Skip,在 baseline 全 0 时不报失败 | 本地 `VV_LLM_API_KEY=` unset 跑测试,Skip;有 key + 真实 baseline 时 ±50% 内通过 |
| D14 | `cd vv && make test` 全绿 | terminal |
| D15 | `cd vv && make lint` 0 issues | terminal |
| D16 | `cd vage && make test` + `cd aimodel && make test` 全绿(零回归) | terminal |
| D17 | `vv/CLAUDE.md` 不再描述 chat / explorer / intent.go classical pipeline | grep `chat\|explorer\|intent\.go` 零相关命中(代码注释除外) |
| D18 | design 文档加 §10 M7 Post-landing notes;§9 中"务实保留"段标已兑现 | 文件存在 §10 |
| D19 | `result.md` 完整记录改动清单、evidence、与 spec 偏差;Loop Anchor grep 零命中复核回写 | 文件存在 |
| D20 | Loop Anchor grep `OrchestrateModeClassical\|LegacyPhaseEvents\|...` 全表零命中 | terminal 输出粘贴到 result.md |

## 7. 计划(执行顺序)

按 4.1 β 节奏分 11 步,每步要求 `cd vv && go test ./... && make lint` 全绿;Step 4 / 5 / 7 是高风险段(主路径 / fallback 路径 / 测试整删),完成后追加 `cd vage && make test && cd ../aimodel && make test`。

| Step | 内容 | 主要文件 | 风险 |
|------|------|----------|------|
| **1** | primary.go nil-Primary 改 error;新增 `TestRun_PrimaryRequired_ReturnsError` / stream 版 | `primary.go`、`primary_test.go` | 中 |
| **2** | 删 intent.go + intent_test.go(grep 确认无外部引用) | `intent.go`、`intent_test.go` | 低 |
| **3** | 删 `LegacyPhaseEvents` 整条 shim(字段 / Option / shim 函数 / env / setup `//nolint`) + 删 `primary_test.go` 中 shim 测试 | `dispatch.go` / `primary.go` / `configs/config.go` / `setup.go` / `primary_test.go` | 中 |
| **4** | 删 `OrchestrateModeClassical` 常量;`ValidateOrchestrateMode` 收敛;Load 加 stale-key Warn;改写测试 | `configs/config.go` / `configs/config_test.go` | 中 |
| **5** | M7-4 fallback 静态 summarize phase + 单测 | `dispatch.go` / `dispatch_test.go` | 中(契约改动) |
| **6** | M7-5 整删 3 个 builder-API 测试文件**前置**审计:grep 等价覆盖,空缺补到 dispatches_tests / vage 层 | 见 §5.5.1 矩阵 | **高** |
| **7** | 整删 `agents_orchestrator_test.go` / `_advanced_test.go` / `agents_dynamic_test.go`(24 tests) | 上述 3 文件 | 高 |
| **8** | port 4 文件 / 7 callsites 到 `setup.New`(包括 `agents_tool_profile_test.go` + 3 个 cli/wiring 测试) | 见 §5.5.2 | 中 |
| **9** | 删 `agents/compat.go`(整文件) | `agents/compat.go` | 低(经 8 后无 caller) |
| **10** | M7-3 drift gate:加 loadBaseline / 比对逻辑 / `baseline_committed.json`(本机有 key 跑一次写真实数据,无 key 写 0 占位) + README | `real_llm_baseline_test.go` / `baseline_committed.json` / `README.md` | 低 |
| **11** | 文档收尾:`doc/design/...md §10` 新增 + §9 标兑现;`vv/CLAUDE.md` 刷新;result.md 落盘 + Loop Anchor grep 零命中粘贴 | 文档若干 | 低 |

每 Step 单独 commit;Step 6/7 之间不可跨夜——审计结果是 Step 7 的执行依据,中间漂移会导致空缺补不齐。

## 8. 风险与缓解

| 风险 | 级别 | 缓解 |
|------|------|------|
| **R1**: primary.go nil-fallback 改 error 后,某些被遗忘的测试构造 dispatcher 时不传 Primary,运行时 panic 或意外报错 | 中 | Step 1 先 `grep -rn 'dispatches\.New\b' vv/` 列全部构造点,逐一确认 Primary 参数;无 Primary 的测试用 `primaryStub` 兜底 |
| **R2**: `BuildIntentSystemPrompt` 删除是公共 API breaking;若有 vv-外部消费者(罕见)会编译失败 | 低 | M6 已发 deprecation,本期 release notes 标 breaking;按 SemVer 视为 minor breaking |
| **R3**: M7-5 整删 24 tests 后,某条产品行为只在被删测试中覆盖 | **高** | Step 6 强制审计 + 空缺先补再删;审计结果回写 result.md 留存 |
| **R4**: 真实 LLM baseline 单点方差大,首次 cron 跑就触发 ±50% gate | 中 | tolerance 设 50%(不是 20%)+ baseline 用 P50 而非均值 + 数据成熟后再收紧;无 key 占位路径不阻塞 |
| **R5**: M7-4 静态占位 phase 推翻 G4 决策,若有 HTTP 消费者依赖"fallback 路径无 summarize phase"语义,反而被 break | **中** | 用户已明确选 d 推翻 G4,接受此风险;release notes 标"M7 起 fallback 路径会发 summarize phase" |
| **R6**: `legacy_phase_events` / `mode=classical` YAML 键在 `Load` 兜底逻辑遗漏导致解析失败 | 中 | Step 3/4 加 stale-key Warn 测试覆盖 |
| **R7**: vage / aimodel 因 vv 改动间接 break(replace directive) | 低 | 高风险 Step 后跑跨模块 make test |
| **R8**: `vv/CLAUDE.md` 刷新遗漏导致下一次 onboarding 误导 | 低 | Step 11 grep `chat\|explorer` 后人工 review 一次 |
| **R9**: M7-1 删 intent.go 后,fastpath / runPrimary 中可能仍间接 import(fallback agent 借 chat prompt 之类) | 中 | Step 2 编译失败立刻 grep 找剩余 import,补迁前驱 |
| **R10**: 一次 PR / 一个 dev session 改动面过大,review 困难 | 中 | 11 个 commit 顺序提交;PR 描述按 Step 分节;result.md 详细记录每步 evidence |

## 9. 决策点(需用户审批)

请用户对以下 4 项分别明示 OK / 修改:

1. **M7-3 baseline 生成方式**:Step 10 执行时若**无** `VV_LLM_API_KEY`,降级为占位 0 值 baseline + 文档标注"维护者首次跑 cron 后填入" —— 是否接受? 如不接受,需用户当场提供 key。
2. **M7-4 推翻 G4 决策的明确化**:M6 design §9.4 改写为"M7 起 fallback 路径发**静态** summarize phase",release notes 标 "**SSE 事件流变化**"。可能影响订阅 fallback 路径的 HTTP 消费者(若存在)—— 是否同意?
3. **`BuildIntentSystemPrompt` 公共 API 删除是 minor breaking**:release notes 标记后接受 —— 是否同意?
4. **Step 6 审计的输出形式**:计划是把"24 tests 等价覆盖矩阵 + 空缺补丁"作为 result.md 的一节回写。审计粒度 = 每个 test function 独立判断 —— 是否同意?如不同意,可改为粗粒度("3 个文件整体覆盖审计 = 通过/不通过")。

## 10. Resume Anchor(Step Checkpoint)

| Step | Status | Note |
|------|--------|------|
| 1 | pending | primary.go nil-Primary → error |
| 2 | pending | 删 intent.go + intent_test.go |
| 3 | pending | 删 LegacyPhaseEvents 整条 shim |
| 4 | pending | 删 OrchestrateModeClassical 常量 + Validate 收敛 |
| 5 | pending | M7-4 fallback 静态 summarize phase |
| 6 | pending | **M7-5 整删前置审计**(24 tests 等价覆盖) |
| 7 | pending | 整删 3 个 builder-API 测试文件 |
| 8 | pending | port 4 文件 / 7 callsites 到 setup.New |
| 9 | pending | 删 agents/compat.go |
| 10 | pending | M7-3 drift gate + baseline_committed.json |
| 11 | pending | 文档收尾(design §10、CLAUDE.md、result.md) |

中断时把状态填到这张表;下次 resume 直接接着干。

---

**等待批准 — `No Approval, No Execute`**

需要你回复:
- §9 四个决策点 → OK / 修改
- 整体方案是否通过 → 通过则进入 Step 1,不通过则修订 spec
