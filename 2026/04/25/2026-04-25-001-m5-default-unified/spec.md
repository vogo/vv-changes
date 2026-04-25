# M5 — 默认切 Unified + HTTP Legacy Shim + Golden 基准集 · Spec

> Session: `2026-04-25-001-m5-default-unified`
> Status: Draft (awaiting approval)
> Date: 2026-04-25
> 关联: `doc/design/enhance-prompt-understand-and-solve.md` §4 M5 · `changes/2026/04/24/2026-04-24-005-m4-primary-assistant/result.md` §"M5 衔接建议"

---

## Core Goal (Loop Anchor)

把 unified 模式从"灰度"升级为**默认**,同时提供两项兼容层(HTTP legacy event shim + classical 兜底)与一套回归闸门(golden baseline),让既有 classical 消费者无感平迁、让性能收益实际落地,且可以随时回退。

---

## 范围(Scope)· M5-core

本期交付 **5 个子目标**,对应 M4 result.md "M5 衔接建议" 的 #1、#2、#3、#4、#6。衔接建议 #5(`primary.allow_bash`)明确推迟到后续迭代。

| ID | 子目标 | 对应建议 | 必/可 |
|----|--------|----------|------|
| G1 | 默认 `mode` 从 classical 切到 unified,并提供"未显式设置"的一次性迁移日志 | #1 | 必 |
| G2 | HTTP legacy event shim:可选开关把 `unified_primary` 映射为 legacy `intent+execute` 事件对 | #2 | 必 |
| G3 | Unified 下 `fallbackAgent` 改为 Primary(chat 在 unified 路径下成为死代码,classical 保持不变) | #3 | 必 |
| G4 | Explorer / Chat 下线文档化(代码层 classical 继续保留 explorer/chat;仅在 design 文档中说明"unified 下已无引用") | #4 | 必 |
| G5 | Golden 基准集 `vv/integrations/golden_tests/`:5 条用例作为切默认的回归闸门 | #6 | 必 |

**明确不做**:
- 衔接建议 #5 `primary.allow_bash` — 新功能,留给后续 session。
- chat agent 代码删除 — unified 下死代码即可,完全移除需要再观察一个迭代。
- classical 路径任何改动 — 严格零侵入,回退安全。

---

## Done Contract

**算完成的硬标准**:

1. **G1 默认切 unified**:
   - `configs.ValidateOrchestrateMode("")` 返回 `OrchestrateModeUnified`(而非 `OrchestrateModeClassical`)。
   - `configs.Load` 在 `cfg.Orchestrate.Mode == ""` 且配置文件存在时,发出一条 `slog.Info` "mode defaulted to unified (M5); set orchestrate.mode: classical in ~/.vv/vv.yaml to keep pre-M5 behaviour" — 仅当 YAML 中真的没写 mode 字段时触发(不影响新用户 / 全新 zero-config 启动时的 noise)。
   - 未显式写 mode 的现有用户,启动后 Dispatcher 走 unified 路径(`primaryAssistant != nil`)。
   - `orchestrate.mode: classical` 仍可用,行为与 M4 前完全一致。

2. **G2 HTTP legacy event shim**:
   - 新增 `configs.OrchestrateConfig.LegacyPhaseEvents bool`(默认 false)。
   - 开启时,unified 路径在原 `unified_primary` 事件之外,**额外**发出 legacy 事件对:
     - `EventPhaseStart{Phase:"intent", PhaseIndex:1}` → 紧跟 `EventPhaseEnd{Phase:"intent"}`(极短,Duration 0 或 ~1ms;表示"unified 模式下 intent 已被吸收到 primary")
     - `EventPhaseStart{Phase:"execute", PhaseIndex:2}` / `EventPhaseEnd{Phase:"execute"}` 包住真正的 primary 流
   - 关闭时(默认),unified 路径严格只发 `unified_primary` 一对事件 — 与 M4 行为完全一致,零回归。
   - 对应测试断言:`legacy_phase_events=true` 时事件序列 ≥ `[start:intent, end:intent, start:execute, end:execute]`,且**不发** `unified_primary`(避免 HTTP 消费者收到重复事件);`legacy_phase_events=false` 时与 M4 测试一致。

3. **G3 Unified 下 fallback 改 Primary**:
   - `setup.go:306` 的 `dispatches.WithFallbackAgent(subAgents["chat"])` 在 `mode == unified` 时改为 `WithFallbackAgent(primary)`;classical 分支保持 `subAgents["chat"]`。
   - `dispatch.go:386/436` 的 `depth >= maxRecursionDepth` 早返回在 unified 下走 Primary(而非 chat);classical 下仍走 chat。
   - 测试:新增 `TestDispatcher_UnifiedDepthExceeded_UsesPrimary` — 构造 `depth=maxDepth`,断言 `subAgents["chat"].ranCount == 0` 且 primary 被调用。

4. **G4 Explorer/Chat 下线文档**:
   - `doc/design/enhance-prompt-understand-and-solve.md` 追加 "§8 M5 Post-landing notes",说明:
     - unified 模式下 explorer 独立 phase 不被触发(Primary 自带 read/glob/grep)。
     - unified 模式下 chat sub-agent 不被调用(Primary 取代)。
     - classical 模式保留以上两项;彻底移除计划在后续 milestone。
   - 不改代码。

5. **G5 Golden 基准集**:
   - 新建 `vv/integrations/golden_tests/` 包,包含 `golden_cases_test.go`。
   - 5 条 case(使用 mock LLM,无外部依赖):
     - `TestGolden_Greeting_Hello` — 输入 "hello",断言 LLM 调用数 ≤ 2(unified:1 次 primary;classical:≤ 2 次 intent+chat),响应 token ≤ 200。
     - `TestGolden_SimpleMath_Calc` — 输入 "calc 5^6" 或等价,primary 直接 answer,LLM 调用数 = 1 (unified) / ≤ 3 (classical)。
     - `TestGolden_SimpleRead_ExplainFile` — 让 primary/researcher 读一个 fixture 文件并摘要,断言工具调用数 = 1(一次 read)。
     - `TestGolden_SimpleEdit_DelegateToCoder` — 让 primary 委派给 coder 做单点修改,断言 `delegate_to_coder` 被调用 1 次。
     - `TestGolden_MultiStepRefactor_Plan` — 让 primary 调用 `plan_task` 产出 2+ step DAG,断言多个 sub-agent 都运行。
   - 每条 case 同时跑 classical + unified,比较两条路径的 `llmCalls` / `toolCalls` / `promptTokens` 指标;unified 必须 ≤ classical 的对应指标(**不是**严格 <,greeting 可能相等)。
   - 结构:每个 case 一个 sub-test,共享 `goldenTestHarness`(mock LLM + mock registry + dispatcher builder)。

---

## 事实清单(Facts)

核心约束与当前代码现状:

1. **M4 已稳定**:Primary Assistant 在 `mode=unified` 下完整运转,4 条 E2E 通过。本期不回滚 M4 任何代码。
2. **fallbackAgent 引用面**(`grep -rn fallbackAgent`):6 处非测试引用 — `dispatch.go:51/149/386/436`、`execute.go:56/68`、`helpers.go:130/145`、`intent.go:185/326`、`fastpath.go:71/242`。G3 只需在 setup.go 注入时分支即可,dispatch 内部所有 `d.fallbackAgent` 引用无需改动 — 无论是 chat 还是 primary,行为统一是"fallback 兜底"。
3. **Event 消费者**(`grep EventPhaseStart`):CLI (`cli/cli.go:695`, `cli/prompt.go:118`) 通过 phase label 驱动 UI;HTTP SSE 透传事件;`integrations/httpapis_tests/` 有对 phase 序列断言的测试 — G2 必须保证 shim 关闭时严格等价于 M4 行为。
4. **Chat agent 依赖**:除 fallback 外,`fastpath.go:71` 的 `Agent:"chat"` 硬编码是 fast-path greeting 默认路由;该逻辑仅在 classical 路径执行,unified 下 `fastPathClassify` 根本不会被调用(`dispatch.go:329` primary 分支比 fastpath 靠前),所以无需改动。
5. **Explorer / Planner**:`reg.Get("explorer")` / `reg.Get("planner")` 在 `setup.go:194/228` 构造时用,若返回 nil 会 `return nil, fmt.Errorf` — 即使 unified 下它们不被调用,注册表里必须仍然存在。本期不改。
6. **Env override 保持**:`VV_ORCHESTRATE_MODE` 仍覆盖 YAML。设置 `VV_ORCHESTRATE_MODE=classical` 可在运行时强制老路径。
7. **迁移日志的"无声行为变更"风险**:YAML 里明文没写 `orchestrate.mode` 的用户,是本次默认切换的**唯一**被影响群体;迁移日志为他们提供可见信号。新建 / 全新 config 的用户看到的 default 直接就是 unified,这本就是设计目标。

---

## 计划(Plan)

执行顺序严格按依赖:G5(基线)→ G3(fallback 切换)→ G2(shim)→ G1(默认切换)→ G4(文档)。G5 先落地,这样 G1 切默认后 Golden 可以同时测两条路径。

### Step 1 · G5 Golden 基准集 先行

- 新建 `vv/integrations/golden_tests/golden_cases_test.go`。
- 实现 `type goldenHarness` — 封装 mock LLM(可编程响应序列)、registries、subAgents、dispatcher 构造器;支持 `.WithMode(OrchestrateModeClassical | OrchestrateModeUnified)` 切换。
- 实现 5 个 sub-test;每个 case 同时跑两种 mode,断言 unified 指标 ≤ classical。
- 过 `go test ./integrations/golden_tests/ -v`,所有 case 绿(classical 基准 + unified 基准同时通过)。

这一步的价值:**先固化"unified 不比 classical 差"的证据**,任何后续改动若让 golden 失败,就是回归信号。

### Step 2 · G3 Unified 下 fallbackAgent 切 Primary

- 改 `vv/setup/setup.go` L306 附近:把单行 `dispatches.WithFallbackAgent(subAgents["chat"])` 抽成变量 `fallback`,classical 指向 `subAgents["chat"]`,unified 指向稍后构造的 primary。
- 注意 ordering:primary 在 dispatcher 之后才构造(M4 的设计),所以要么**先构造 primary 再构造 dispatcher**(调整 setup.go §10 §11 顺序),要么**用 post-construction setter** 替换 fallback。最简:在 `if cfg.Orchestrate.Mode == configs.OrchestrateModeUnified` 分支末尾,除了 `dispatcher.SetPrimaryAssistant(primary)` 外,再加一次 `dispatcher.SetFallbackAgent(primary)` — 但 dispatcher 目前没有 SetFallbackAgent 公开方法,需要添加一个 1 行 setter。
- 新增 `dispatcher.SetFallbackAgent(a agent.Agent)` 方法(与 SetPrimaryAssistant 对称)。
- 新增测试 `vv/dispatches/primary_test.go` 中的 `TestDispatcher_UnifiedDepthExceeded_UsesPrimary`:构造 depth=maxDepth,`subAgents["chat"]` 与 primary 分别是可计数的 stubAgent,断言 primary ranCount==1 且 chat ranCount==0。

### Step 3 · G2 HTTP legacy event shim

- 新增 `configs.OrchestrateConfig.LegacyPhaseEvents bool` 字段(yaml:`legacy_phase_events,omitempty`),默认 false。
- 新增 `dispatches.WithLegacyPhaseEvents(bool) Option`,`Dispatcher` 结构体加同名字段。
- 改 `dispatches/primary.go:runPrimaryStream`:
  ```go
  if d.legacyPhaseEvents {
      // 发射 legacy 事件对替代 unified_primary
      send(start:intent) / send(end:intent, Duration:0)
      send(start:execute) // 包住实际流
      forwardSubAgentStream(...)
      send(end:execute, 聚合 tracker 字段)
      return
  }
  // 默认:M4 行为,原样发 unified_primary
  ...
  ```
- `setup.go` 读 `cfg.Orchestrate.LegacyPhaseEvents`,通过 `dispatches.WithLegacyPhaseEvents` 传入。
- 新增测试 `TestRunStream_UnifiedMode_LegacyShim_EmitsIntentAndExecutePhases`:断言事件序列恰为 `[start:intent, end:intent, start:execute, {primary 工具事件}, end:execute]`,且**没有** `unified_primary`。

### Step 4 · G1 默认切 unified

- 改 `configs/config.go:ValidateOrchestrateMode`:空值映射到 `OrchestrateModeUnified`(而非 `OrchestrateModeClassical`)。同步修订 doc 注释("\"\" defaults to unified (M5+)")。
- 改 `configs/config.go:Load`:在 `ValidateOrchestrateMode` 的前面,先探测"YAML 是否显式声明了 mode"。具体实现:在 yaml.Unmarshal 之前 / 之后,用 `bytes.Contains(data, []byte("mode:"))` 对 `orchestrate` 段做粗匹配(绝大多数情况 `orchestrate.mode:` 是顶格或缩进 2/4 空格的单行);记录一个 bool `modeExplicit`。若 `!modeExplicit && 文件存在`,发一条 `slog.Info("vv: orchestrate.mode defaulted to unified (M5); set orchestrate.mode: classical in ~/.vv/vv.yaml to keep pre-M5 behaviour", "path", path)`。
  - 简化方案:重新解析 YAML 到一个 `map[string]any`,检查 `m["orchestrate"]["mode"]` 是否存在;零依赖。
- 更新 `vv/configs/config_test.go`:
  - `TestValidateOrchestrateMode` 空值期望从 classical 改成 unified。
  - 新增 `TestLoad_ExplicitClassicalPreserved` + `TestLoad_NoModeSet_DefaultsToUnified_WithInfoLog`。
- 更新 `vv/setup/setup_test.go` 中涉及默认行为的 smoke 测(如 `TestNew_DefaultMode_` 类)让其断言 primary != nil。
- 更新 M4 result 测试中凡是 `cfg.Orchestrate.Mode = ""` 隐式期望 classical 的用例,显式写 `mode=classical`(避免被默认切换误伤)。

### Step 5 · G4 下线文档

- 追加 `doc/design/enhance-prompt-understand-and-solve.md §8`:
  - "Unified 已成为默认":说明 mode=classical 仍保留。
  - "unified 下已不再调用的组件":explorer phase、chat sub-agent、独立 intent classifier、summarize phase。
  - "代码保留原因":classical 回退路径仍需这些组件;彻底删除至少再等 1 个迭代观察真实用户反馈。

### Step 6 · 全量验证

- `cd vv && make test` 全绿。
- `cd vv && make lint` 0 issues。
- `cd aimodel && make test` 全绿(不应受影响,但跑一下确认)。
- `cd vage && make test` 全绿(不应受影响)。
- Golden baseline 在 classical + unified 两条路径上都通过,且 unified 的 LLM 调用数 / token 在 greeting 和 simple-math case 上**严格 <** classical(这是本次切默认的量化收益证据)。

---

## 风险(Risks)

| # | 风险 | 缓解 |
|---|------|------|
| R1 | 默认切 unified 导致**静默行为变更**,某些老用户的 HTTP 消费者依赖 `intent/execute/summarize` phase 崩掉 | G2 legacy shim 提供 opt-in 兼容层;G1 的 `slog.Info` 给运维可见信号;README / release notes 提示 |
| R2 | G3 把 fallback 从 chat 换成 primary,在 `depth >= maxRecursionDepth` 场景下**primary 也可能触发 delegate/plan_task**,导致无限递归 | primary 的 delegate_to_* 工具内部已有 `IncrementDepth`;dispatch.go:321 的 `depth >= maxRecursionDepth` 早返回会拒绝继续递归,并直接调 fallback — 但此时 fallback 是 primary,primary 调 delegate 又会撞到 depth 检查,进入无限循环的风险存在。**缓解**:在 `fallbackRun` / forwardSubAgentStream 的 fallback 路径,用一个特殊 context flag 禁用 primary 的工具调用,或直接让 fallback 路径的 primary 以 "tools=nil" 模式回答。**决策**:最简方案 — 让 fallback 路径绕过 primary 的工具注册,只保留 answer_directly 风格的纯对话。需要 G3 实现时细化。 |
| R3 | Golden baseline 用 mock LLM,与真实 LLM 行为有差距,指标只是相对对比,不能证明"真实环境下确实省 token" | Golden 的价值是**回归闸门**而非**性能基准**;真实环境测量已在 M2/M4 的集成测试中覆盖;明确写在文档里 |
| R4 | `configs.Load` 的 "YAML 显式检测 mode" 逻辑脆弱 — 若用户用 anchors、comments、不同缩进,map 解析可能漏判 | 使用 `map[string]any` 二次解析而非字符串匹配;漏判只导致多发一条 info 日志,不改变行为 — 低影响 |
| R5 | 5 条 golden case 用例跑起来慢 → 跑 make test 从秒级变成十秒级 | 使用 mock LLM 即时返回预设响应;每条 case 预估 < 100ms;5 条合计 < 1s |

---

## 兼容性承诺

- `orchestrate.mode: classical` 在 M5 后行为与 M4 完全一致,byte-for-byte。
- `orchestrate.legacy_phase_events: false` (默认) 下 unified 模式行为与 M4 完全一致。
- 所有 M4 及以前的集成测试不修改任何 assertion(仅在 `configs.Orchestrate.Mode` 隐式 "" 的用例里显式写 `classical`,避免默认切换的误伤 — 这些是本地测试 fixture 调整,不是行为回归)。
- `VV_ORCHESTRATE_MODE=classical` 环境变量在容器 / CI 场景下提供零配置回退。

---

## Resume Anchor

- Spec: `changes/2026/04/25/2026-04-25-001-m5-default-unified/spec.md`(本文件)
- Result: `changes/2026/04/25/2026-04-25-001-m5-default-unified/result.md`(待完成时写入)
- 前置:M4 已完整落盘在 `changes/2026/04/24/2026-04-24-005-m4-primary-assistant/`。

**若中断后续轮次继续推进**:Step 1(golden)→ Step 2(fallback)→ Step 3(shim)→ Step 4(切默认)→ Step 5(doc)→ Step 6(验证)。
