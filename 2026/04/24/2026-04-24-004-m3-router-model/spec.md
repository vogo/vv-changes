# M3 — 路由小模型（Router Model for Routing-Only LLM Calls）

> Session: 2026-04-24-004-m3-router-model
> Design: `doc/design/enhance-prompt-understand-and-solve.md` §3.3 / M3
> Owner: vv dispatches

## Core Goal（loop anchor）

允许用户为 Dispatcher 的**路由/分类 LLM 调用**单独配置一个更便宜/更小的模型（如 `claude-haiku-4-5`），与主模型（sub-agent 执行、summarize、explorer 等）完全解耦。与 M2 unified intent 正交兼容——unified intent 这一次调用就走 router 模型。

**一句话验收**：当 `orchestrate.router.model: claude-haiku-4-5` 被配置时，intent-level LLM 调用都落在 router client 上，sub-agent 的执行调用仍落在主 client 上（用两个独立 mock LLM 就能证明两者不串扰）。

Non-goals:
- 不引入多路由模型的 `composes`/failover/负载均衡逻辑——只是一个 "主 / 路由" 二元分离。
- 不改 M4 Primary Assistant 架构；本阶段只让现有 intent/classify/reassess/unified-intent 四个调用点走 router。
- 不改 planner-agent 分支（该分支 LLM 由 planner sub-agent 自身的构造决定，不在 dispatcher LLM 范围内）；之所以 scope out 是因为那是一条完整 sub-agent 路径，和 dispatcher 直接发的 intent LLM 调用不是同一层。
- 不改 explorer 的 LLM（explorer 本质也是一个 sub-agent，使用主模型的 ReAct）。
- 不做真实模型回归；mock-based 单测 + 集成测即可，与 M1/M2 的验证 posture 一致。

## Done Contract

1. **配置**：`OrchestrateConfig` 新增 `Router LLMConfig`（YAML key: `router`）。默认零值表示 "router 未启用，路由沿用主 LLM"。
2. **有效配置解析**：新增 `configs.EffectiveRouterConfig(cfg *Config) (LLMConfig, bool)`：当 `cfg.Orchestrate.Router.Model == ""` 时返回 `(LLMConfig{}, false)`；否则返回一个字段级合并后的 `LLMConfig`（router 字段优先，未设置的字段回落到 `cfg.LLM`）。
3. **Dispatcher**：新增 `routerLLM aimodel.ChatCompleter` 和 `routerModel string` 字段，以及 `WithRouterLLM(llm, model) Option`。新增内部辅助 `routerClient()`/`routerModelName()`：当 router 未配置时返回主 LLM/主 model，保证既有单测断言不变。
4. **替换调用点**：以下四个函数改为用 `routerClient()`/`routerModelName()` 构造 `ChatRequest`：
   - `recognizeIntentDirect`
   - `reassessIntent`
   - `classifyDirect`
   - `recognizeIntentUnified`
5. **Setup 接线**（`vv/setup/setup.go`）：
   - 读取 `cfg.Orchestrate.Router`，若启用则 `NewLLMClient(effectiveRouterConfig)` 构造独立 router client，否则 `routerClient = nil`。
   - 对 router client 应用与主 LLM 相同的中间件链（debug、budget），顺序与主 LLM 一致。
   - 把 router client 和其 model 通过新增的 `WithRouterLLM` 传入 Dispatcher。
6. **可观测性**：LLMCallEnd 事件及 budget 计费应能准确把 router 调用计到 router model（通过 router client 自带的 pricing lookup；我们不需要新加 tag，现有 middleware 用的是请求里 `ChatRequest.Model`）。
7. **向后兼容**：未配置 router 时，所有现有单测、集成测不变地通过。
8. **测试**：
   - 单测 `router_llm_test.go`：验证 `routerClient()`/`routerModelName()` fallback、四个调用点都走 router（用独立 mock LLM）、`WithRouterLLM(nil, "")` 等价于未设置。
   - 单测 `config_test.go` 扩展：`EffectiveRouterConfig` 的合并规则、YAML 往返。
   - 集成测 `router_model_integration_test.go`：两个独立 `sequentialMockLLM` 分别充当 router 与主 LLM，断言 "unified_intent answer_directly" 下只 router 被调用 1 次，而 "unified_intent delegate_to coder" 下 router 被调用 1 次、coder sub-agent（主 LLM 驱动的 stub）被调用 1 次。
9. **`make test` + `make lint`** 在 `vv/` 和 `vv/integrations/dispatches_tests/` 全绿。

**Evidence of done**：
- 新增集成测 `TestRouterModel_RoutingIsolatedFromExecution` 明确断言两个 LLM 的 `callCount`，且 `chat` / `coder` sub-agent 的调用次数与设计语义一致。
- `go test ./dispatches/ -run Router -v` 全通过。
- 关闭 router 时所有 pre-M3 测试无改动地通过（向后兼容证明）。

## Facts（from exploration）

- 主 LLM 当前的四个调用点：
  - `dispatches/intent.go:145 recognizeIntentDirect` — `d.llm.ChatCompletion(ctx, chatReq)`，`chatReq.Model = d.model`。
  - `dispatches/intent.go:226 reassessIntent` — 同上。
  - `dispatches/intent.go:690 classifyDirect` — 同上（planner-agent 为 nil 时的直连分类）。
  - `dispatches/unified_intent.go:200 recognizeIntentUnified` — 同上。
- 以上四处是 Dispatcher 自发的 LLM 调用；`classify → plannerAgent.Run` 走的是 sub-agent 的 LLM（在 setup 构造 planner 时注入的主 LLM），不在本次改造范围内。
- `vv/configs/config.go:932 NewLLMClient(LLMConfig)` 已是创建 aimodel.Client 的唯一入口；直接复用即可。
- `LLMConfig` 字段：`Provider`、`Model`、`APIKey`、`BaseURL`——自然可分块合并。
- Middleware 链在 `setup.go:543-556`：`wrappedLLM = llmClient` → debug wrap → budget wrap。Router 需要走一样的链（成本/budget 也要计入 router 消费；debug 也要能看到 router 调用），否则会漏观测。
- 现有 mock：
  - `dispatches/fastpath_test.go:17 countingChatCompleter` — 单 response 的计数器，适合单测。
  - `integrations/.../dispatches_helper_test.go:20 sequentialMockLLM` — 顺序返回响应，适合集成测。
- `IntentResult.validate` 无需改动。
- `d.llm != nil` 守卫仍然是 "能否做 LLM 路由" 的底线——router 配置只是"如果有就优先用它"。即使 `d.routerLLM == nil` 但 `d.llm != nil` 时，路由继续走 `d.llm`（现有行为）。

## Plan（sequenced）

1. **Config 层**
   - `vv/configs/config.go`：在 `OrchestrateConfig` 增 `Router LLMConfig \`yaml:"router,omitempty"\``。
   - 新增 `func EffectiveRouterConfig(cfg *Config) (LLMConfig, bool)`。规则：若 `cfg.Orchestrate.Router.Model == ""`，返回 `(_, false)`；否则字段级合并（router 优先，缺省取 `cfg.LLM`）。Provider 空时 provider-specific BaseURL 默认需要和主路径一致（复用 `NewLLMClient` 中的 OpenAI baseURL 回填即可）。
   - 环境变量 override：`VV_ROUTER_MODEL`（主要的、常用的）、`VV_ROUTER_PROVIDER`、`VV_ROUTER_API_KEY`、`VV_ROUTER_BASE_URL`。四个都可选；与 `VV_LLM_*` 对齐。
   - 单测：3-4 个 case（router 未设置 → disabled；只设 model → provider/key 继承主；显式覆盖 provider/key → 走 router 自己的；YAML 往返）。

2. **Dispatcher 字段 + Option**
   - `vv/dispatches/dispatch.go`：加 `routerLLM aimodel.ChatCompleter` + `routerModel string` 字段。
   - 新增 `func WithRouterLLM(llm aimodel.ChatCompleter, model string) Option`。`nil` / 空 model 视为未设置。
   - 新增内部方法 `routerClient() aimodel.ChatCompleter`（返回 `d.routerLLM` 或 `d.llm`）、`routerModelName() string`（返回 `d.routerModel` 或 `d.model`）。

3. **改造调用点**（四处）
   - `recognizeIntentDirect` / `reassessIntent` / `classifyDirect` / `recognizeIntentUnified`：把 `d.llm` 替换为 `d.routerClient()`、`d.model` 替换为 `d.routerModelName()`。
   - 保持原有错误语义（返回错误与之前完全一致；mock LLM 换成 router 不改变行为）。

4. **Setup 层**
   - `vv/setup/setup.go:Init`：在构造完主 `wrappedLLM` 后，若 `EffectiveRouterConfig` 返回 `(cfg, true)`，构造 router client 并应用与主 LLM 同形的 middleware（debug wrap + budget wrap）。
   - `setup.New` 函数签名扩一个可选参数不合适（会波及调用方）；改为把 `routerLLM`、`routerModel` 传入现有 `*Result` 构造路径——最简做法是给 `Init` 内部拼好后通过新增的 option 传给 `dispatches.New`。
   - 增加 `dispatches.WithRouterLLM(routerLLM, routerModel)` 到现有 dispatcher option 列表。

5. **单测 `vv/dispatches/router_llm_test.go`**
   - `TestRouterLLM_FallsBackToMain`：不设 `WithRouterLLM`，`routerClient()` 返回 `d.llm`。
   - `TestRouterLLM_UnifiedIntentUsesRouter`：两个 `countingChatCompleter`，分别作为主/路由；unified-intent 命中 answer_directly，断言 router.calls == 1、main.calls == 0。
   - `TestRouterLLM_ClassicDirectUsesRouter`：classic intent-direct 流程下，router.calls == 1、main.calls == 0。
   - `TestRouterLLM_ReassessUsesRouter`：explore → reassess 场景，确认 reassess 用 router。
   - `TestWithRouterLLM_NilIsIdempotent`：`WithRouterLLM(nil, "")` 不修改 Dispatcher 的 router 字段，保持 `routerClient() == d.llm`。

6. **集成测 `vv/integrations/dispatches_tests/dispatches_tests/router_model_integration_test.go`**
   - Helper：`newRouterDispatcher(t, routerLLM, mainLLM, subAgents...)`，复用已有 `newIntegrationRegistry`/`makeSubAgents`。
   - `TestRouterModel_RoutingIsolatedFromExecution`：unified_intent + delegate_to(coder) 场景；router mock 返回 tool_call，主 LLM mock 不应被 dispatcher 本体调用（sub-agent 目前是 stub，不会实际打主 LLM，但断言 `mainLLM.callCount == 0` 能捕获未来回归）。
   - `TestRouterModel_OnlyAffectsRouting`：unified_intent answer_directly 场景；router.calls == 1 且 chat/coder 都未被调用。
   - `TestRouterModel_Disabled_UsesMain`：不配 router，unified_intent 走主 LLM（main.calls == 1）——pre-M3 行为。

7. **`configs` YAML + env 单测**
   - `configs/config_test.go`：YAML 能读到 `orchestrate.router.model`；env `VV_ROUTER_MODEL` 覆盖 YAML；`EffectiveRouterConfig` 的三种合并 case。

8. **Validate**
   - `cd vv && make test && make lint`。
   - `cd vv/integrations/dispatches_tests && go test ./... && golangci-lint run`。

9. **Reverse Sync**
   - 写 `result.md`：改动清单 + 测试结果 + 与计划的偏差 + 后续建议。

## Risks

- **Budget 双计**：router 与主 LLM 共享 `budgetEventDispatcher`/session budget trackers。两者都应计入同一 budget（一个 session 的总花费），这是正确行为；单元测不专门验证 budget，但集成测里若路由命中 budget 上限会失败——默认 budget 关闭，零风险。
- **Debug wrap 对齐**：debug middleware 必须同样包住 router，否则 `--debug` 看不到 router 的请求。计划里明确 "应用与主 LLM 相同的 middleware 链" 即可。
- **Router model 与 provider 不兼容**：用户设了 `router.model: claude-haiku-4-5` 但 `router.provider` 继承自主 LLM 的 `openai`——会发出异常请求。通过 `NewLLMClient` 本就有的 provider 校验返回错误，启动即失败，安全。
- **Tool-calling 能力**：router 小模型（如 Haiku、GPT-4o-mini）对 tool_choice="auto" 的支持稳定；即便回退到 plain text，unified_intent 已有 "plain text → answered" 兜底（见 M2 的 parse 逻辑）。
- **sub-agent 路径被误替换**：planner/explorer 的 LLM 在 setup 构造时绑定，不依赖 dispatcher 的 `d.llm`，改造范围只碰 dispatcher 自己发的四个调用点。

## Checkpoint — awaiting approval before implementation

- 当前理解（Restate）：把 Dispatcher 发起的四类 LLM 调用切到一个可选的 router LLM client；配置通过 `orchestrate.router: {model, provider?, api_key?, base_url?}`；未配置时行为完全不变。
- Core goal：path label / token attribution / sub-agent 链路都维持现状，仅在 Dispatcher 的 intent/classify/reassess/unified 四点把 client 换成 router。
- Next 3 actions：(1) 改 config + 加 `EffectiveRouterConfig`；(2) 加 Dispatcher 字段、Option 与内部 fallback helper，并替换四处调用；(3) setup 构造 router client、加集成/单测。
- Risks + mitigations：见 Risks 一节；全部由 mock-based 测试覆盖。
- Validation：`make test` + `make lint` 两个 module 全绿；新增 Router 相关单测 + 集成测明确断言 router/主 LLM 调用次数互不串扰。

**Open questions for the user（默认值在中括号，用户不表态即按默认推进）：**
1. Config key：`orchestrate.router`（继承 `OrchestrateConfig` 下集中管理）还是顶层独立 `router_llm`？[默认 `orchestrate.router`，与 M2 `orchestrate.unified_intent` 同级，维持聚合。]
2. 环境变量前缀：`VV_ROUTER_MODEL`/`VV_ROUTER_PROVIDER`/`VV_ROUTER_API_KEY`/`VV_ROUTER_BASE_URL` 还是 `VV_LLM_ROUTER_*`？[默认 `VV_ROUTER_*`——更短，与 `VV_LLM_MODEL` 并列。]
3. 是否把 `reassess`/`classifyDirect` 也算"路由调用"一起切换？[默认**是**——它们都是 "给定用户输入，决定下一步怎么走" 的分类性质调用。如果用户倾向保守，只切 unified_intent + recognizeIntentDirect 两点，代价是 reassess 继续用主模型拿不到小模型的成本节省。]
4. Planner-agent 分支是否也要享受 router？[默认**否**——那是 sub-agent 完整执行链路，不在 "dispatcher 直发调用" 范围；若需要，属 M4 话题（planner sub-agent 的 LLM 注入现在由 setup 从 `cfg.LLM` 来，改造成本不低）。]
