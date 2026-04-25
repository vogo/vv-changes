# M3 路由小模型（Router Model）落地结果

> Session: 2026-04-24-004-m3-router-model
> 对应设计: `doc/design/enhance-prompt-understand-and-solve.md` §3.3 / 里程碑 M3
> 状态: 已完成，配置未开启时行为与 pre-M3 完全一致

## 1. 核心目标对齐（Core Goal 回写）

原核心目标：允许用户为 Dispatcher 的路由/分类 LLM 调用单独配置更便宜小模型（如 `claude-haiku-4-5`），与主模型解耦；与 M2 unified-intent 正交兼容——unified-intent 这一次调用就走 router。

验证证据（由独立的双 mock LLM 测试证明 "路由 LLM 与执行 LLM 物理隔离"）：

- 单测 `TestRouterLLM_UnifiedIntentUsesRouter`（`vv/dispatches/router_llm_test.go`）：`router.calls == 1`、`main.calls == 0`、`chat` sub-agent 未运行。
- 单测 `TestRouterLLM_ClassicDirectUsesRouter`：经典 `recognizeIntentDirect` 路径下 `router.calls == 1`、`main.calls == 0`。
- 单测 `TestRouterLLM_ReassessUsesRouter`：`intent → explore → reassess` 路径下 `router.calls == 2`（intent + reassess）、`main.calls == 0`。
- 集成测 `TestRouterModel_AnsweredUsesOnlyRouter`：端到端 answered 路径 `router.callCount == 1`、`main.callCount == 0`、`chat/coder` 均未运行。
- 集成测 `TestRouterModel_DelegateIsolatesRoutingFromExecution`：delegate_to(coder) 路径 `router.callCount == 1`、`main.callCount == 0`、`coder` sub-agent 运行 1 次（sub-agent 走自己的 LLM 链路，不经 Dispatcher 的 main client，故 `main.callCount == 0` 是正确隔离证据）。
- 集成测 `TestRouterModel_Disabled_UsesMain`：未设 router 时走回主 LLM（`main.callCount == 1`）——向后兼容证明。

## 2. 实际改动清单（Change Log）

### 新增文件
- `vv/dispatches/router_llm_test.go` — 6 个单测：fallback 行为 / `WithRouterLLM(nil,"")` 幂等 / 字段设置 / 三条路径（unified_intent / classicDirect / reassess）均用 router。
- `vv/integrations/dispatches_tests/dispatches_tests/router_model_integration_test.go` — 3 个集成测：answered 仅 router / delegate 路由与执行物理隔离 / 未开启走回主 LLM。
- `changes/2026/04/24/2026-04-24-004-m3-router-model/spec.md` — 事前 spec（含 open questions 默认值）。
- `changes/2026/04/24/2026-04-24-004-m3-router-model/result.md` — 本文件。

### 修改文件
- `vv/configs/config.go`
  - `OrchestrateConfig` 新增字段 `Router LLMConfig \`yaml:"router,omitempty"\``。
  - 新增四个环境变量覆盖：`VV_ROUTER_MODEL` / `VV_ROUTER_PROVIDER` / `VV_ROUTER_API_KEY` / `VV_ROUTER_BASE_URL`。
  - 新增 `EffectiveRouterConfig(cfg *Config) (LLMConfig, bool)`：router.Model 为空（含纯空白）返回 `(_, false)`；否则做字段级合并（router 优先，缺省取 `cfg.LLM`）。
- `vv/configs/config_test.go`
  - 追加 7 个测：`Disabled` / `InheritsFromMain` / `OverridesMain` / `WhitespaceModelIsDisabled` / `NilConfig` / `Load_RouterFromYAML` / `Load_RouterEnvOverride`。
- `vv/dispatches/dispatch.go`
  - `Dispatcher` 新增字段 `routerLLM aimodel.ChatCompleter`、`routerModel string`。
  - 新增 `WithRouterLLM(llm, model)` 函数式选项；`nil` 或空 model 时不覆写，保持幂等语义。
  - 新增内部方法 `routerClient()` / `routerModelName()`：优先返回 router，缺省回落主 LLM。
- `vv/dispatches/intent.go`
  - `recognizeIntentDirect` / `reassessIntent` / `classifyDirect`：`d.llm` → `d.routerClient()`，`d.model` → `d.routerModelName()`。
  - `useUnifiedIntent` 守卫从 `d.llm != nil` 改为 `d.routerClient() != nil`（后者在 router 未设时就是 `d.llm`，语义不变）。
- `vv/dispatches/unified_intent.go`
  - `recognizeIntentUnified`：同样切到 `routerClient()`/`routerModelName()`。
- `vv/setup/setup.go`
  - 抽出共享 `wrapLLMClient(client, cfg, pricingModel, opts, sessionBudget, dailyBudget)`：按固定顺序套上 debug → budget middleware。主 LLM 与 router LLM 共用同一个 wrap helper。
  - `Options` 新增 `RouterLLM`、`RouterModel` 字段；`Init` 在 `EffectiveRouterConfig` 返回 enabled 时构造 router client、应用同形中间件、写入 `opts`。
  - `New` 的 dispatcher option 列表改为 `[]dispatches.Option` 切片拼接，开启时追加 `WithRouterLLM(opts.RouterLLM, opts.RouterModel)`——不改 `New` 的函数签名，向后兼容 `integrations/httpapis_tests/askuser_tests`、`integrations/setup_tests/setup_misc_test` 等既有外部调用。
  - 构造 router client 后 `slog.Info("vv: router LLM enabled", model=, provider=)`，便于启动日志排查。

### 未触达
- `recognizeIntentViaPlanner` / `classify`（planner sub-agent 分支）—— 如 spec 所述，planner 是一个完整 sub-agent，其 LLM 由 `setup.New` 构造时直接绑主模型；让 planner 走 router 属于 M4 话题，本次不动。
- `explorer` 的 LLM —— 同理，explorer 是一个带工具的 ReAct sub-agent，使用主模型。
- `vv/dispatches/fastpath.go` —— 快速路径根本不发 LLM 调用，跟 router 无关。

## 3. 验证结果（Evidence / Done Contract）

### 单测（`vv/`）
- `go test ./dispatches/ -run "Router|WithRouter" -v`：6/6 通过。
- `go test ./configs/ -run "Router|EffectiveRouter" -v`：7/7 通过。
- `make test`（完整扫一遍所有包）：无 FAIL，唯一 skip 是 `TestDebug_RealLLM_Smoke`（预期，需真实 LLM）。
- `make lint`：`0 issues`。

### 集成测
- `cd vv/integrations/dispatches_tests && go test ./...`：全绿，含新增 3 个 `TestRouterModel_*` + 既有 M1 `TestFastPathIntegration_*` / M2 `TestUnifiedIntent_*` / 经典 intent / replan / streaming / summary。
- `cd vv/integrations/dispatches_tests && golangci-lint run`：`0 issues`。
- `cd vv/integrations/setup_tests && go test ./...`：`project_instructions_tests` / `setup_tests` / `wiring_tests` 三模块全绿。
- `cd vv/integrations/httpapis_tests && go test ./...`：`askuser_tests` / `http_tests` / `shutdown_tests` 三模块全绿（证明 `Options` 字段加法改动对 HTTP 层无回归）。

### Done Contract 逐条核对
| # | Done 项 | 状态 |
|---|---------|------|
| 1 | `orchestrate.router` YAML 字段加入 `OrchestrateConfig` | ✅ `configs/config.go` |
| 2 | `EffectiveRouterConfig` 合并器 | ✅ `configs/config.go` + 7 单测 |
| 3 | Dispatcher 新增 `routerLLM`/`routerModel` + `WithRouterLLM` + fallback helper | ✅ `dispatches/dispatch.go` |
| 4 | 四个调用点切换 client：intent-direct / reassess / classify-direct / unified-intent | ✅ `intent.go` + `unified_intent.go` |
| 5 | setup 构造 router client 并应用与主 LLM 同形 middleware 链 | ✅ `setup/setup.go` 抽出 `wrapLLMClient` helper |
| 6 | 可观测性：router 调用计入自己的 pricing + session/daily budget 合并累积 | ✅ `wrapLLMClient(..., routerCfg.Model, ...)` 传入的是 router model，budget tracker 是进程级单例 |
| 7 | 向后兼容：未配置 router 时全部既有测试不变通过 | ✅ `make test` / `make lint` 全绿 |
| 8 | 单测覆盖 fallback + nil 幂等 + 三个路径 | ✅ `router_llm_test.go` 6 case |
| 9 | 集成测覆盖 answered / delegate 隔离 / disabled | ✅ `router_model_integration_test.go` 3 case |

## 4. 与原计划的偏差

| 计划项 | 实际处理 | 原因 |
|--------|----------|------|
| Dispatcher `New` 不改签名 | 维持不改，用 `Options.RouterLLM/RouterModel` 把 router 从 `Init` 透传到 `New` | 直接扩 `New` 的位置参数会波及 `integrations/httpapis_tests/askuser_tests:257,297`、`integrations/setup_tests/setup_tests/setup_misc_test.go:46,103,158` 五处调用点；加到 `Options` 是零成本向后兼容扩展。 |
| Setup 直接展开两次 middleware 链 | 抽出 `wrapLLMClient(client, cfg, pricingModel, opts, sessionBudget, dailyBudget)` 共享 | 避免 debug/budget 包装顺序在主/路由两处漂移。`pricingModel` 专门按调用方传入（主传 `cfg.LLM.Model`，router 传 `routerCfg.Model`），budget tracker 单例天然合并总花费。 |
| 单测 mock | 新增 `taggingChatCompleter` / `sequentialTaggingCompleter` 本地轻量 mock | 复用现有 `countingChatCompleter` 会让多响应路径（reassess）失去序列化能力；起两个本地 mock 更直观。 |
| Planner-agent 分支一并走 router | 按 spec 默认 **out of scope** | spec 开放问题 #4 的默认值（M4 再处理）。 |

## 5. 未解决 / 后续建议

1. **真实模型冒烟**：mock 证明了 client 物理隔离，但不同 router 模型对 tool_choice="auto" 的支持质量仍要实测，尤其是 Haiku 4.5 / GPT-4o-mini 等。建议在上线 `orchestrate.router` 前用真实后端跑 `integrations/` 的 smoke（手动，带 API key）。
2. **Planner-agent + router 合并**：若未来想让 planner sub-agent 也走小模型，需要把 `registries.FactoryOptions.LLM` 拆成 "routing LLM / execution LLM" 两份，并在 setup 的 planner factory 注入 router。属 M4 话题，不在本 session。
3. **UX / 文档**：`~/.vv/vv.yaml` 配置示例未更新到 README / CLAUDE.md。下次 docs 更新时可补一段：
   ```yaml
   orchestrate:
     router:
       model: claude-haiku-4-5
       # provider / api_key / base_url 省略则继承顶层 llm:
   ```
4. **日志/事件路径标签**：当前 router 的调用通过 `ChatRequest.Model = routerModel` 自然被 pricing lookup 识别，但 `debugs` sink 与 `LLMCallEnd` 事件没有显式标注 "这是 router"。若 dashboards 想按 "路由 vs 执行" 分桶，目前只能从 model 名字推断——未来可以考虑加 `ChatRequest.Tag` 或类似的透传字段（跨 `aimodel`），不是 M3 必做项。

## 6. 恢复/交接锚点（Resume Ready）

- 配置开关：
  ```yaml
  orchestrate:
    router:
      model: claude-haiku-4-5        # 只要这行非空就启用
      # provider / api_key / base_url 可选，缺省继承 llm:
  ```
- 环境变量：`VV_ROUTER_MODEL` / `VV_ROUTER_PROVIDER` / `VV_ROUTER_API_KEY` / `VV_ROUTER_BASE_URL`。
- 关键代码位点：
  - `vv/configs/config.go`：`OrchestrateConfig.Router` 字段 + `EffectiveRouterConfig` 函数。
  - `vv/dispatches/dispatch.go`：`routerLLM`/`routerModel` 字段、`WithRouterLLM` option、`routerClient()`/`routerModelName()` helper。
  - `vv/dispatches/intent.go` + `unified_intent.go`：四处 `ChatCompletion` 调用。
  - `vv/setup/setup.go`：`wrapLLMClient` helper、`Init` 中 `routerWrappedLLM` 构造与 `opts.Router*` 透传、`New` 中的条件性 `dispatches.WithRouterLLM` 追加。
- 测试路径：`vv/dispatches/router_llm_test.go`、`vv/configs/config_test.go`（7 个新 case）、`vv/integrations/dispatches_tests/dispatches_tests/router_model_integration_test.go`。
- 默认状态：router 未设置时一切行为等价于 pre-M3；开关时只需在 YAML 或环境变量里设 model。
