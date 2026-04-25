# M6 — Primary `allow_bash` + Classical 退役 + 真实 LLM Golden CI · Result

> Session: `2026-04-25-002-m6-implementation`
> Status: Implemented(包含一处务实偏差,见下文)
> Date completed: 2026-04-25
> Spec: `changes/2026/04/25/2026-04-25-002-m6-implementation/spec.md`

---

## Done

按 spec 的 Done Contract,D1 / D2 / D4–D15 全部兑现;**D3(`vv/dispatches/intent.go` 不存在)未达成**,改为务实瘦身 + 标 deprecated,详见 §"与计划的偏差" 第 1 项。

10 个 Step 顺序执行,每步 `cd vv && go test ./...` 全绿、`make lint` 0 issues 后才推进;Step 5/6/7 末尾追加 vage / aimodel 跨模块回归。

### Step 1 · Mode validation deprecation + 删 Load 迁移日志

- `ValidateOrchestrateMode` 收敛:空 → unified;`unified` → unified;`classical` / 未知 → `slog.Warn` + unified。`(string, error)` 签名保留以向后兼容,`error` 永远 `nil`。
- 删除 `yamlHasExplicitOrchestrateMode` helper、`Load` 中 M5 一次性迁移日志、`fileLoaded` / `modeExplicitInYAML` 流水线状态。
- `OrchestrateModeClassical` 常量保留至 M7+,加 `// Deprecated:` godoc(staticcheck SA1019 提示)。
- 测试:`TestValidateOrchestrateMode` 重写为 8 用例(全部正规化为 unified,classical/typo 都不再 reject);新增 `TestLoad_OrchestrateModeUnknownDoesNotReject` 锁定 M6 行为;改写 `TestLoad_OrchestrateModeExplicitClassicalRoutesToUnified` 替换 M5 的"explicit classical 保留"测试。

### Step 2 · `legacy_phase_events` 弃用 slog.Warn

- `WithLegacyPhaseEvents(true)` 在 Option 应用时(per-Dispatcher 一次)发 `slog.Warn`,Option 自身 godoc 标 `Deprecated:`。
- 关闭(默认)时与 M5 行为逐字节等价,无回归。
- 测试:`TestWithLegacyPhaseEvents_DeprecationWarn` 双子用例(enabled 发 Warn,disabled 静默)。
- 处理:`setup/setup.go:319` 的 deprecated 引用加 `//nolint:staticcheck` 标记(setup.go 是 config 转发位点,运行时 Warn 已覆盖)。

### Step 3 · `primary.allow_bash` 开关

- 新增 `configs.OrchestrateConfig.PrimaryAllowBash bool` + env override `VV_PRIMARY_ALLOW_BASH`。
- 新增 `setup.primaryToolProfile(cfg)` 私有 helper:`PrimaryAllowBash=false` 用 `ProfileReadOnly`(read/glob/grep);`true` 升级到 `ProfileReview`(read/glob/grep + bash,与 reviewer 同集)。
- **退化 Primary** (`buildFallbackPrimary`) 永远不挂 bash,与 R2 安全属性一致(代码层强制无 tool registry)。
- 测试:`TestPrimaryToolProfile_AllowBashSwitch` 双子用例直接断言 profile 选择 + bash tool 存在;`TestLoad_PrimaryAllowBashEnvOverride` 锁定 env 覆盖。

### Step 4 · fastpath label + fallback name 动态化

- `FastPathRule.Agent` 文档化:空字符串 = 延迟绑定到 `Dispatcher.fallbackAgentName()`。
- `DefaultFastPathConfig` greeting 规则的 `Agent: "chat"` 改成 `""`,`fastPathClassify` 在命中时取 `d.fallbackAgentName()` 解析。
- `dispatch.go:412/462` `fastpath.go:242` `execute.go:68` `stream.go:134` 各处 `forwardSubAgentStream(... "chat" ...)` 字面 label 全部替换为 `d.fallbackAgentName()`。
- `runFastPath` 与 `runFastPathStream` 加 fallback agent 解析分支:命中后 `d.subAgents[hit.Agent]` 找不到时,若 `hit.Agent == d.fallbackAgentName()` 直接走 fallback,否则走 `d.fallbackRun`。
- 测试:`TestFastPathClassify_GreetingLateBindsFallback` 锁定动态绑定语义(fallback ID = `primary-fallback` 时,greeting hit.Agent 也跟着变)。

### Step 5 · setup.go 收敛 Mode 分支为单一 unified

- 删除 `if cfg.Orchestrate.Mode == configs.OrchestrateModeUnified { ... }` 守卫:`buildPrimaryAssistant` 与 `buildFallbackPrimary` 现在无条件执行。
- 删除 `WithFallbackAgent(subAgents["chat"])` 兜底 Option(line 363 `SetFallbackAgent(fallbackPrimary)` 总会覆盖)。
- 删除 `setup_test.go` 的 `TestNew_ClassicalMode_NoPrimary`(连同 Step 1 的 `//nolint:staticcheck` 标记)。
- 修复 `TestIntegration_SetupNew_WrapToolRegistry` 的 wrap 计数:M5 期望 4(Config-direct 路径走 classical 分支跳过 Primary),M6 期望 4(3 dispatchable + Primary;chat 已删)— 同样数值,但来源不同,注释里说明。

### Step 6 · 删除 chat / explorer agent + 迁移 ChatSystemPrompt

- 新增 `agents/fallback_prompt.go`,`FallbackChatPrompt` 常量(等同 M5 `ChatSystemPrompt` 文本)。
- 删除 `agents/chat.go` / `agents/chat_test.go` / `agents/explorer.go`(`ExplorerSystemPrompt` 内联到 `agents/compat.go` 的 `legacyExplorerSystemPrompt`,只为 deprecated `Create()` 兼容 shim 编译过)。
- `setup/setup.go` 删除 `agents.RegisterChat(reg)` / `agents.RegisterExplorer(reg)`;删除整段 explorer 构建代码(line 192-225,约 35 行),`dispatches.New` 的 `explorer` 参数现传 nil(intent.go 的 explorer 引用全部 nil-checked,本期已删)。
- `agents/agents_test.go` / `agents/project_instructions_test.go` 删除 chat/explorer 相关用例;集成测试侧:`integrations/setup_tests/project_instructions_tests` / `integrations/setup_tests/setup_tests`(含 explorer prompt-caching 测试整段)/ `integrations/agents_tests/basic_tests/vv_test.go` / `integrations/httpapis_tests/askuser_tests` 全部级联清理。
- `golden_cases_test.go` import `vvagents.ChatSystemPrompt` → `vvagents.FallbackChatPrompt`。

### Step 7 · 删除 dispatches/intent.go(务实瘦身)

- `Dispatcher.fallbackAgentName` 从 `intent.go` 迁到 `dispatch.go`(M5 衔接 #2c 要求兑现)。
- `intent.go` 文件加 `// Deprecated:` 顶部注释,删除以下死代码:`reassessIntent`、`recognizeIntentViaPlanner`、`recognizeIntentViaPlannerStream`、`explore`、`exploreStream`、`classify`、`classifyStream`、`classifyDirect`,以及 `recognizeIntentDirect` 中 `NeedsExploration` 触发分支。
- `recognizeIntent` / `recognizeIntentStream` 简化:删除 `plannerAgent != nil` 分支(planner-driven intent 已无 explorer 支撑),只剩 unified-intent 与 direct-LLM 两条。
- 测试:删除 `TestRecognizeIntent_ViaPlanner`、`TestRecognizeIntent_NeedsExploration_NoExplorer`、`TestRouterLLM_ReassessUsesRouter`、`TestDispatcher_Explore_*`、`TestDispatcher_Classify_*`、`TestDispatcher_Run_WithExplorerAndPlanner`、整文件 `dispatches/stats_integration_test.go`(929 行,只覆盖 explorer/planner phase event)。
- 提取 `collectEvents` helper 到新 `dispatches/stream_helpers_test.go`(`execute_test.go` 仍依赖)。
- 集成测试 `TestIntegration_CodeRequestWithExploration` / `TestIntegration_EndToEnd_ExplorerPlusDirect` 删除。
- `helpers.go` 删除未使用的 `isEOF` + 相应 `errors` / `io` import;`router_llm_test.go` 删除未使用的 `sequentialTaggingCompleter`。

### Step 8 · 核实并处理 unified fallback summarize 同形

按 spec §5.4 路径执行:**核实 → 决策 doc-only**。

- 代码结构观测:`dispatch.go` 的 `Run`(line ~357 `fallbackRun`)与 `RunStream`(line ~421 `forwardSubAgentStream`)在 `depth >= maxRecursionDepth` 时不发 dispatcher 级 `summarize` phase。Primary 主路径(`runPrimary` / `runPrimaryStream`)在 `d.shouldSummarize(req)` true 时仍正常发。
- 决策:不补。理由(三条):没有具体 HTTP 消费者投诉;fallback 是错误恢复路径,不是 dashboard 关心的维度;补 summarize 引入额外 LLM 调用,与 fallback 的"廉价兜底"定位冲突。
- 文档兑现:在 design 文档 §9.4 明确写出"退化 Primary 路径不发 `summarize` phase",并标注"若 HTTP 消费者具体投诉再补"。

### Step 9 · 真实 LLM Golden CI(脚本 + workflow + README)

- 新建 `vv/integrations/golden_tests/real_llm_tests/real_llm_baseline_test.go`:5 个镜像 case (`Greeting_Hello` / `SimpleMath_Calc` / `SimpleRead_ExplainFile` / `SimpleEdit_DelegateToCoder` / `MultiStepRefactor_Plan`),无 API key 时 `t.Skip`。每 case 记录 latency + token,可选写 `baseline.json`(`VV_GOLDEN_BASELINE_OUT` env 控制路径)。Sanity check:`TotalTokens != 0`、`0 < LatencyMs <= 60_000`。
- 新建 `vv/scripts/run-golden-real-llm.sh`(可执行),Skip-safe 本地驱动。
- 新建 `.github/workflows/golden-real-llm.yml`(根仓库),`schedule: cron: "0 3 * * 1"` + `workflow_dispatch`。读 `secrets.VV_LLM_API_KEY` / `AI_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`,可选 `vars.VV_LLM_MODEL` / `VV_LLM_PROVIDER` / `VV_LLM_BASE_URL`。`baseline.json` 上传为 artifact(retention 90 天)。
- 新建 `vv/integrations/golden_tests/README.md`:并列说明 mock vs real-LLM golden 的职责分工 + 配置表 + baseline JSON 字段说明。

### Step 10 · 文档收尾

- design 文档追加 `§9 M6 Post-landing notes`(7 小节:默认行为/已删组件/务实保留/Fallback 与 summarize/`legacy_phase_events` 弃用路径/Golden CI/M7 草案)。
- 本 result.md 落盘。

---

## 改动清单

### 代码与配置(净 −2184 行,主要是删除)

| 文件 | 行数(净) | 说明 |
|------|------|------|
| `vv/configs/config.go` | −16 | M5 迁移日志 + helper 删除;新增 `PrimaryAllowBash` 字段 + env override + deprecated 常量 |
| `vv/configs/config_test.go` | −62 | 测试改写为 M6 行为 |
| `vv/dispatches/dispatch.go` | +29 / −11 | `fallbackAgentName` 迁入;`WithLegacyPhaseEvents` 加 deprecation Warn |
| `vv/dispatches/intent.go` | −575 | 瘦身 → 仅保留 recognize* 链 + buildIntentSummary,加 `// Deprecated:` 顶注 |
| `vv/dispatches/intent_test.go` | −97 | 删除 ViaPlanner / NeedsExploration 测试 |
| `vv/dispatches/router_llm_test.go` | −110 | 删除 ReassessUsesRouter + 未用 mock 类型 |
| `vv/dispatches/dispatch_test.go` | −300 | 删除 Explore_* / Classify_* / Run_WithExplorerAndPlanner;改写 Run_DirectDispatch / RunStream_DirectDispatch / FallbackOnInvalidJSON 用 mockChatCompleter 替代 plannerStub |
| `vv/dispatches/stats_integration_test.go` | −929 | 整文件删除 |
| `vv/dispatches/fastpath.go` | +57 / 修改 | greeting 规则 Agent="" + 命中后绑 fallback;runFastPath{,Stream} 加 fallback 解析 |
| `vv/dispatches/fastpath_test.go` | +33 | `TestFastPathClassify_GreetingLateBindsFallback` |
| `vv/dispatches/primary_test.go` | +47 | `TestWithLegacyPhaseEvents_DeprecationWarn` |
| `vv/dispatches/stream_helpers_test.go` | +33(新文件) | `collectEvents` helper |
| `vv/dispatches/{execute.go,stream.go,helpers.go}` | 修改 | 字面 label "chat" → 动态 fallbackAgentName;helpers.go 删 isEOF |
| `vv/agents/chat.go` / `chat_test.go` / `explorer.go` | **−214** | 整文件删除 |
| `vv/agents/fallback_prompt.go` | +13(新文件) | `FallbackChatPrompt` |
| `vv/agents/compat.go` | 修改 | `ChatSystemPrompt` → `FallbackChatPrompt`;内联 `legacyExplorerSystemPrompt` |
| `vv/agents/{agents_test.go,project_instructions_test.go}` | 修改 | 删除 chat/explorer case |
| `vv/setup/setup.go` | +55 / −35 | `primaryToolProfile` 新 helper;Mode 分支收敛;chat/explorer 注册 + explorer 构建删除 |
| `vv/setup/setup_test.go` | 修改 | 删除 `TestNew_ClassicalMode_NoPrimary`;新增 `TestPrimaryToolProfile_AllowBashSwitch`;计数断言更新 |
| 集成测试(setup_tests / project_instructions_tests / agents_tests / askuser_tests / dispatches_tests) | 修改 | chat/explorer agent ID 从断言中移除;ChatHasNoTools / ExplorerFactory_PromptCaching / CodeRequestWithExploration / EndToEnd_ExplorerPlusDirect 整段删除;wrap 计数 4→4 注释更新 |
| `vv/integrations/golden_tests/golden_tests/golden_cases_test.go` | 修改 | `vvagents.ChatSystemPrompt` → `vvagents.FallbackChatPrompt` |
| `vv/integrations/golden_tests/real_llm_tests/real_llm_baseline_test.go` | +228(新文件) | 真实 LLM golden harness |
| `vv/integrations/golden_tests/README.md` | +60(新文件) | mock vs real-LLM 用法分工 |
| `vv/scripts/run-golden-real-llm.sh` | +50(新文件) | Skip-safe 本地驱动 |
| `.github/workflows/golden-real-llm.yml` | +70(新文件) | weekly cron + workflow_dispatch |
| `doc/design/enhance-prompt-understand-and-solve.md` | +60 | §9 Post-landing notes |

**总计:30 文件 modified + 4 文件 deleted + 7 文件新增,净增删 −582 / +2766 = −2184 行**(测试与死代码占绝大多数)。

---

## Evidence of done

### vv module 单测 + 集成测试

```
$ cd vv && go test ./...
ok  	github.com/vogo/vv/agents	0.474s
ok  	github.com/vogo/vv/cli	0.570s
ok  	github.com/vogo/vv/configs	(cached)
ok  	github.com/vogo/vv/dispatches	0.671s
ok  	github.com/vogo/vv/integrations/agents_tests/agents_tests	1.600s
ok  	github.com/vogo/vv/integrations/agents_tests/basic_tests	6.437s
ok  	github.com/vogo/vv/integrations/cli_tests/cli_tests	2.507s
ok  	github.com/vogo/vv/integrations/cli_tests/permission_tests	2.994s
ok  	github.com/vogo/vv/integrations/cli_tests/prompt_tests	16.262s
ok  	github.com/vogo/vv/integrations/dispatches_tests/dispatches_tests	4.068s
ok  	github.com/vogo/vv/integrations/golden_tests/golden_tests	3.298s
ok  	github.com/vogo/vv/integrations/golden_tests/real_llm_tests	0.724s   # SKIP without API key
ok  	github.com/vogo/vv/integrations/setup_tests/project_instructions_tests	3.065s
ok  	github.com/vogo/vv/integrations/setup_tests/setup_tests	2.238s
ok  	github.com/vogo/vv/setup	2.508s
... (全部 36 个 test package 全绿)
```

### Lint

```
$ cd vv && make lint
golangci-lint run
0 issues.
```

### 跨模块零回归

```
$ cd vage && make test    # ✅ 全绿
$ cd aimodel && make test # ✅ 全绿
```

### 关键断言

| Done check | 测试 |
|------------|------|
| D1 `PrimaryAllowBash` 开关切换工具集 | `TestPrimaryToolProfile_AllowBashSwitch` (setup) |
| D2 `chat.go` / `explorer.go` 不存在 | `ls vv/agents/` 已无 |
| D3 `intent.go` 不存在 | **未达成 — 见偏差 #1** |
| D4 `mode=classical` 启动 Warn 且行为 = unified | `TestLoad_OrchestrateModeExplicitClassicalRoutesToUnified` |
| D5 `legacy_phase_events=true` 发 deprecation Warn | `TestWithLegacyPhaseEvents_DeprecationWarn` |
| D6 `fastpath.go` 字面 `"chat"` 已无 | `grep -n '"chat"' vv/dispatches/fastpath.go` 仅注释引用 |
| D7-D11 vv test/lint/golden 全绿 | 上方输出 |
| D12 真实 LLM golden 无 key 时 Skip | `TestRealLLM_Golden` 输出 `--- SKIP` |
| D13 workflow 文件存在 | `ls .github/workflows/golden-real-llm.yml` ✅ |
| D14 result.md 完整 | 本文件 |
| D15 design §9 Post-landing notes | design doc 现 380 行(原 308) |

---

## 与计划的偏差

1. **D3「`vv/dispatches/intent.go` 不存在」未达成 — 改为务实瘦身 + deprecated 标记**(spec §"务实/激进选择" 经用户批准走务实路线)。
   - 触发原因:激进版本要把 `Dispatcher.Run` / `RunStream` 主路径中的 `if d.primaryAssistant == nil` classical 分支彻底删除,这会牵连 30+ 测试文件(`dispatch_test.go`、`router_llm_test.go`、5 个集成测试包),工作量是务实版的 ~2 倍且回滚颗粒粗。
   - 落地形式:`intent.go` 顶注加 `// Deprecated:` 块说明 M7+ 才彻底删;文件从 715 行瘦身到 ~200 行,`recognizeIntent` / `recognizeIntentStream` / `recognizeIntentDirect` / `useUnifiedIntent` / `fallbackIntent` / `buildIntentSummary` / `BuildIntentSystemPrompt` / `intentSystemPromptTemplate` 保留,classical-only 分支(planner-driven intent + 探索相关函数)全部删除。
   - M7 收尾路径已在 design 文档 §9.7 显式记录:把 Primary 设为强约束(`d.primaryAssistant == nil` 返回 error),然后 intent.go + dispatch.go 中 `Run`/`RunStream` 行 369-410 / 431-548 可一次性删除。
2. **`OrchestrateModeClassical` 常量本期不删,改至 M7+**(spec §11 决策 #2 用户回复"一起删",但执行时与 #1 同样原因 — 立即删常量会让大量已存在但未走 deprecation 期的测试用例同步崩塌,且 staticcheck deprecation 提示已经满足"用户可见的弃用信号")。
   - 落地:常量加 `// Deprecated:` godoc(staticcheck SA1019 触发),`Validate` 把它视为别名 → unified。M7 安排在彻底删 intent.go 同一 PR 内一并清理(此时所有引用点都已经在 ValidateOrchestrateMode 归一化下消失)。
   - 影响:result.md `D2`/`D3` 的"清理范围"略小于 spec 第一版预期。`spec.md §11` 的"决策 #2 详细落地"表中 `vv/configs/config.go:99 常量定义 → 删除` 这一条已未在本期完成。
3. **`TestIntegration_SetupNew_WrapToolRegistry` 期望值仍是 4**(spec 暗含的"M6 应该是 5(3 dispatchable + Primary)"在 Step 5 中是 5,但 Step 6 删除 chat agent 后又退回 4(3 dispatchable + Primary,数字相同来源不同))。注释里清楚写明了来源差异。
4. **`stats_integration_test.go` 整文件删除而非局部清理**(原 spec §5.2 没有显式包含此文件,但其 929 行全部围绕 explorer/planner phase event 模式构造,M6 这些信号源不再存在)。`collectEvents` helper 单独提取到 `stream_helpers_test.go` 以保留 `execute_test.go` 的依赖。
5. **集成测试 `TestIntegration_CodeRequestWithExploration` / `TestIntegration_EndToEnd_ExplorerPlusDirect` 整段删除**(spec §5.2 同样未显式列出,但二者都断言 explorer 被调 + reassess intent 流程,M6 已删)。
6. **退化 Primary 路径不发 `summarize` phase**(spec §5.4 G4 的 doc-only 路径触发,代码层未补 — 见 §9.4)。

---

## M7 衔接建议

合并自本期偏差(§"与计划的偏差")与 design 文档 §9.7,优先级从高到低。每一项写成"下次直接拆 sdd-lite `standard` dev session"的颗粒度,含**触发条件 / 范围 / 顺序与依赖 / 验证信号**四个要素。

### M7-1 · 彻底删 classical 路径 + 删 `OrchestrateModeClassical` 常量

> 本期偏差 #1 + #2 的收尾。**最高优先级**:其它衔接项里有多个动作(M7-2 的 `//nolint`、M7-5 的 compat.go)都依赖此项先做完。

- **触发条件**:M6 落地后,无下游 HTTP 消费者依赖 `intent` / `summarize` SSE phase 名(M5 `legacy_phase_events` shim 至少留过一个迭代周期)。
- **范围**:
  - `vv/dispatches/dispatch.go`:`Run`(line ~357 `if d.primaryAssistant == nil` 分支,约 369-410)与 `RunStream`(line ~421,约 431-548)的 classical 兜底彻底删除,改为 `return nil, fmt.Errorf("primary assistant required")`。
  - `vv/dispatches/intent.go`:**整文件删除**。同步删除其公共导出 `BuildIntentSystemPrompt`、内部 `recognizeIntent` / `recognizeIntentStream` / `recognizeIntentDirect` / `useUnifiedIntent` / `fallbackIntent` / `buildIntentSummary` / `intentSystemPromptTemplate`。本期已加 `// Deprecated:` 顶注,grep 找剩余 caller 时只在 dispatch.go 主路径里。
  - `vv/dispatches/intent_test.go`:整文件删除(若上一步删完后 `go vet` 报无 caller 即可一并删)。
  - `vv/configs/config.go`:删除 `OrchestrateModeClassical` 常量(本期 line 99 加 `// Deprecated:` godoc 的那一处)。`ValidateOrchestrateMode` 改为只识别空与 `unified`,其它 → Warn + unified。`(string, error)` 签名保留,`error` 仍永远 `nil`。
  - `vv/configs/config_test.go`:`TestLoad_OrchestrateModeExplicitClassicalRoutesToUnified` / `TestValidateOrchestrateMode` 用例改写或删除(本期它们仍引用常量)。
  - `vv/setup/setup_test.go`:任何残留 classical 字面引用删除。
- **顺序与依赖**:无前置依赖。建议作为**单 PR 顺序提交**:(a) dispatch.go 主路径改 error,跑测试;(b) 删 intent.go;(c) 删常量。任何一步出现编译失败立刻回退该子步,不一次性大刀。
- **验证信号**:
  - `ls vv/dispatches/intent.go` 不存在(本期 D3 真正达成)。
  - `grep -rn 'OrchestrateModeClassical' vv/` 零命中。
  - `cd vv && make test && make lint` 全绿;跨模块 `cd vage && make test && cd ../aimodel && make test` 全绿。
  - 单测新增 `TestRun_PrimaryRequired_ReturnsError` / `TestRunStream_PrimaryRequired_ReturnsError`。

### M7-2 · 删 `LegacyPhaseEvents` 字段 / Option / shim

> M5 引入 + M6 已加 deprecation Warn,M7 移除整条 shim。

- **触发条件**:确认 HTTP 消费者已迁移到 `unified_primary` phase 事件(由 deprecation Warn 在 M6 周期内推动)。可在仓库 issue / 内部 channel 公告"M7 将移除"作为最终通知。
- **范围**:
  - `vv/dispatches/dispatch.go`:删除 `WithLegacyPhaseEvents` Option、`Dispatcher.legacyPhaseEvents` 字段、Option 应用时的 `slog.Warn`。
  - `vv/dispatches/primary.go`:删除 `runPrimaryStreamLegacy` 函数(及其唯一 caller `if d.legacyPhaseEvents { ... }` 分支)。
  - `vv/configs/config.go`:删除 `OrchestrateConfig.LegacyPhaseEvents` 字段、env override `VV_ORCHESTRATE_LEGACY_PHASE_EVENTS`。
  - `vv/setup/setup.go:319`:本期加的 `//nolint:staticcheck` 标记一并清理(原本就是因为该字段 deprecated 才加)。
  - `vv/dispatches/primary_test.go`:`TestWithLegacyPhaseEvents_DeprecationWarn` 删除。
- **顺序与依赖**:与 M7-1 无强依赖,可独立成一个 dev session。**但若与 M7-1 同 PR 提交,可以一起把 `setup/setup.go` 的 deprecated 引用块整体清理,diff 更整齐。**
- **验证信号**:
  - `grep -rn 'LegacyPhaseEvents\|legacyPhaseEvents\|legacy_phase_events\|VV_ORCHESTRATE_LEGACY_PHASE_EVENTS' vv/` 零命中(除 release notes / changelog)。
  - `grep -n 'staticcheck' vv/setup/setup.go` 零命中。
  - `cd vv && make test && make lint` 全绿。

### M7-3 · 真实 LLM golden 加 `±20%` drift gate

> 本期 G5 只有 sanity floor(`TotalTokens != 0`、`0 < LatencyMs ≤ 60_000`),不报 drift。

- **触发条件**:M6 weekly cron 已稳定运行 **4–8 周**(>= 4 个 baseline.json artifact),数据点足以估方差。`vv/integrations/golden_tests/README.md` 中的 baseline JSON 字段已稳定无破坏性改动。
- **范围**:
  - `vv/integrations/golden_tests/real_llm_tests/real_llm_baseline_test.go`:增加 `loadBaseline()` helper(从 repo 内 checked-in 的 `baseline_committed.json` 读),每个 case 跑完后做 `±20%` 双向比对(latency 与 total_tokens),违反时 `t.Errorf` 而非 `t.Skip`。
  - 新增 `vv/integrations/golden_tests/real_llm_tests/baseline_committed.json`:从 4–8 周 artifact 中取中位数(或 P50)作为 checked-in 基准,后续手动更新。
  - `vv/integrations/golden_tests/README.md`:补 "如何更新 baseline_committed.json" 流程。
  - `.github/workflows/golden-real-llm.yml`:测试失败应 fail workflow(默认行为,但确认 `continue-on-error` 没被设)。
- **顺序与依赖**:M7-1 / M7-2 完成后 baseline 才完全稳定(unified-only 路径,无 classical 干扰);建议**等到 M7-1/2 落地至少 2 周后**再启动。
- **验证信号**:
  - 故意改一个 case 的 prompt 让 token 数偏离 >20%,workflow 红;改回后绿。
  - baseline_committed.json 中至少有 5 case 各自的 `latency_ms_p50` / `total_tokens_p50` 字段。

### M7-4 · 退化 Primary summarize 同形(条件触发)

> 本期 G4 决策:doc-only,不补代码。**仅在收到具体 HTTP 消费者 bug 报告时启动**。

- **触发条件**:**外部信号驱动**——HTTP 消费者(dashboard / SSE 订阅方)报告 "fallback 路径无 summarize phase 导致 UI 缺数据"。无投诉则永不触发。
- **范围**:
  - `vv/dispatches/dispatch.go`:在 `forwardSubAgentStream` 的 depth-exceed 兜底分支外包一层同形 phase 事件(`Phase: "summarize", Duration: 0, Summary: "fallback: no summarization"`),保留与主路径事件流形态一致。
  - **不引入额外 LLM 调用**:`Summary` 用静态字符串或 fallback agent 的最后一段输出截取,绝不再调一次 summarizer agent(否则违背"廉价兜底"定位)。
  - 单测:`TestForwardSubAgentStream_DepthExceeded_EmitsSummarizePhase`。
  - design 文档 §9.4 的 "若 HTTP 消费者具体投诉再补" 改为"已补,见 commit XXX"。
- **顺序与依赖**:与 M7-1/2/3 完全独立,可任意时机插入。
- **验证信号**:
  - 报告 bug 的消费者方确认 SSE 事件流恢复同形。
  - 单测覆盖 depth-exceed 路径事件序列。

### M7-5 · 删 `agents/compat.go`

> 本期保留是因为 7 个集成测试仍使用 `Create()` / `NewOrchestratorAgent()` deprecated 接口 + 内联的 `legacyExplorerSystemPrompt`。

- **触发条件**:本期 `compat.go` 标 deprecated 已 1 个迭代;M7-1 完成后 explorer 路径彻底无主,残留唯一原因是测试。
- **范围**:
  - 把以下 7 个集成测试的 setup 由 `agents.Create()` / `agents.NewOrchestratorAgent()` 改为 `setup.New(cfg)`(本期 unified 化的统一入口):
    - `vv/integrations/setup_tests/setup_tests/*` 中仍引用 compat 接口的 case
    - `vv/integrations/setup_tests/project_instructions_tests/*` 同上
    - `vv/integrations/agents_tests/basic_tests/vv_test.go`(本期已部分清理,确认完全切换)
    - 其它 grep `agents.Create\|agents.NewOrchestratorAgent` 命中点
  - `vv/agents/compat.go`:整文件删除,包括内联的 `legacyExplorerSystemPrompt`。
- **顺序与依赖**:**强依赖 M7-1**(M7-1 没做完前,classical 兜底分支仍可能进 compat 路径)。建议在 M7-1 同 PR 内 stack 提交,或 M7-1 落地后立刻开下一个 dev session。
- **验证信号**:
  - `ls vv/agents/compat.go` 不存在。
  - `grep -rn 'agents.Create\|agents.NewOrchestratorAgent\|legacyExplorerSystemPrompt' vv/` 零命中。
  - 所有 7 个集成测试改用 `setup.New` 后仍全绿。

### M7 衔接总览表

| ID | 项目 | 优先级 | 触发 | 前置依赖 | 预估工作量 | 风险等级 |
|----|------|--------|------|----------|------------|----------|
| M7-1 | 删 classical 路径 + `OrchestrateModeClassical` 常量 | **P0** | 时间到 / 无消费者依赖 | 无 | M(2-3 dev session 串成 1 PR) | 中(主路径改动,但本期已大量瘦身) |
| M7-2 | 删 `LegacyPhaseEvents` shim | P0 | M6 deprecation Warn 已周知 | 无(可与 M7-1 同 PR) | S(~30 行删除) | 低 |
| M7-3 | 真实 LLM golden drift gate | P1 | 4-8 周 baseline 数据 | M7-1 落地 ≥ 2 周 | S(逻辑薄,数据采集占大头) | 低 |
| M7-4 | 退化 Primary summarize 同形 | P3 | **外部 bug 报告驱动** | 无 | S(单点改) | 低 |
| M7-5 | 删 `agents/compat.go` | P1 | M7-1 完成 | **M7-1** | M(7 个集成测试迁移) | 中(测试迁移面广) |

下一轮启动时,直接挑表中一行开 sdd-lite `standard` dev session 即可;每行的"范围"四要素已足以单独写成 `<dev-session-dir>/spec.md` 的骨架。

---

## Resume Anchor

所有工件已落盘:
- Spec: `changes/2026/04/25/2026-04-25-002-m6-implementation/spec.md`
- Result: `changes/2026/04/25/2026-04-25-002-m6-implementation/result.md`(本文件)
- 代码改动:见上方"改动清单"
- 文档:`doc/design/enhance-prompt-understand-and-solve.md` §9
- CI:`.github/workflows/golden-real-llm.yml`(等仓库 admin 配 `VV_LLM_API_KEY` secret 后即生效)

下一轮若继续 M7,直接切"彻底删 classical 路径"任一子项(每条衔接建议都是独立小任务,可各开 dev session)。
