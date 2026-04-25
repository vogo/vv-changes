# M4 — Primary Assistant 原型(Unified Mode)

> Session: 2026-04-24-005-m4-primary-assistant
> Design: `doc/design/enhance-prompt-understand-and-solve.md` §3.4 / M4
> Owner: vv dispatches / agents

## Core Goal(loop anchor)

当 `orchestrate.mode: unified` 启用时, Dispatcher 的 `fastPath → intent → execute → summarize` 流水线整体让位给一个**Primary Assistant** —— 一个自带只读工具(read/glob/grep/todo/ask_user)并持有两类委派工具(`delegate_to_<agent>` 与 `plan_task`)的 TaskAgent。它自主决定:直接回答、就地探索后回答、委派给 specialist(coder/researcher/reviewer)、或触发多步计划。`classical` 模式(默认)行为完全不变,`unified` 为显式 opt-in 的灰度开关。

**一句话验收**:同一个 `Dispatcher.New(...)` + `mode=unified` 下,一条打招呼请求由 Primary 一次 LLM 调用直接回答;一条"修改 a.go"请求由 Primary 调用 `delegate_to_coder` 工具后返回 coder 的输出;一条多步请求由 Primary 调用 `plan_task` 工具后走现有 `orchestrate.ExecuteDAG`。三条路径的 **phase 事件** 统一使用 `"unified_primary"`,**不再发射** classical 的 `intent/execute/summarize` 事件。

Non-goals:
- 不删除 classical 路径、不删除 `chat`/`explorer`/`planner` sub-agent —— 这些继续支持 `mode=classical`(M5 再考虑下线);本次只新增并行分支。
- 不修改 CLI 权限模型 —— Primary 的工具经过 `opts.WrapToolRegistry` 包一次,与 sub-agent 复用同一把权限护栏。
- 不引入写权限工具(edit/write/bash) —— Primary 默认只读;写操作必须经由 `delegate_to_coder` 走权限受控的 coder。
- 不改变 `maxRecursionDepth=2` 的语义 —— Primary 在深度 0 运行,`delegate_to_*` 工具内部递增深度到 1,被委派的 agent 受深度 2 限制无法再次递归回来。
- 不实现 fastPath 与 Primary 的共存逻辑 —— unified 模式下 fastPath 完全关闭(Primary 自身足以处理寒暄场景;保留 fastPath 会和 Primary 的决策语义冲突)。
- 不改 HTTP `EventPhase*` 订阅者兼容层 —— 新增 `unified_primary` 只是新类型,不删除旧类型,若现有订阅者按字符串过滤旧 phase 会忽略新 phase,这是预期行为(M5 会做兼容层)。

## Done Contract

1. **Config**:`OrchestrateConfig` 新增 `Mode string \`yaml:"mode,omitempty"\`` 字段。合法值:`""`/`"classical"` → 走旧路径(默认);`"unified"` → 走 Primary Assistant。未知值在 `Load` 里报错。环境变量 `VV_ORCHESTRATE_MODE` override。
2. **Primary Assistant 构造**:新增 `vv/agents/primary.go`:
   - `PrimarySystemPrompt` 常量 —— 描述 Primary 的职责、可用的 `delegate_to_*` 与 `plan_task` 工具、何时直接回答 / 何时委派。
   - `RegisterPrimary(reg *registries.Registry)` 把 Primary 注册成非 dispatchable 的 `AgentDescriptor`(ID=`primary`, `ProfileReadOnly`)。
   - Factory 构造一个 TaskAgent,其 `taskagent.WithToolRegistry(...)` 接收由 setup 组装好的 finalToolReg(包含 read/glob/grep + todo + ask_user + 委派工具 + plan_task)。
3. **委派工具**:新增 `vv/dispatches/primary_tools.go`:
   - `RegisterDelegateTools(reg *tool.Registry, subAgents map[string]agent.Agent) error` —— 对 `coder`/`researcher`/`reviewer` 三个 specialist,各通过 `agenttool.Register` 注入一个 tool,名字分别为 `delegate_to_coder`/`delegate_to_researcher`/`delegate_to_reviewer`,参数 schema 为 `{"task": "...", "context": "..."}`,extractor 拼接成 specialist 的 user message。
   - `RegisterPlanTaskTool(reg *tool.Registry, exec PlanExecutor) error` —— 注入 `plan_task` 工具,参数 schema 沿用 M2 `UnifiedToolPlanTask` 的 `{goal, steps}`,handler 调用 `exec.RunPlan(ctx, plan)` 并返回文本结果。
   - `type PlanExecutor interface { RunPlan(ctx, *Plan, req) (*schema.RunResponse, error) }` —— 由 Dispatcher 实现以复用现有 `runPlan` 逻辑(零新 DAG 代码)。
4. **Dispatcher 接入**:
   - `dispatch.go` 新增 `primaryAssistant agent.Agent` 字段 + `WithPrimaryAssistant(a agent.Agent) Option`。
   - `Dispatcher.Run`:在 `depth >= maxRecursionDepth` 之后、`fastPathClassify` 之前,若 `d.primaryAssistant != nil` 则直接 `return d.runPrimary(ctx, req)`(不再走 fastPath / intent / execute / summarize)。
   - `Dispatcher.RunStream`:同上,增加 `runPrimaryStream`,发 `EventPhaseStart{Phase:"unified_primary"}` + 转发 Primary 的 stream events + `EventPhaseEnd`。
   - 实现 `Dispatcher.RunPlan(ctx, *Plan, req) (*schema.RunResponse, error)` 以满足 `PlanExecutor` 接口 —— 复用现有 `runPlan` / `buildNodes` / `ExecuteDAG`(从 `execute.go` 抽取,签名 `runPlan(ctx, req, plan, upstream, contextSummary)`;给个薄 public 包装)。
5. **Setup 接线**(`vv/setup/setup.go`):
   - 读取 `cfg.Orchestrate.Mode`。当 `=="unified"` 时:
     1. 先构造 sub-agents(现有流程不变)。
     2. 构造 Dispatcher 的骨架(先 `New(...)`,但不注册 primary)。
     3. 构造 Primary 的 tool registry:从 `ProfileReadOnly.BuildRegistry(...)` 开始,追加 todo、ask_user、三个 delegate_to 工具,并把 Dispatcher 作为 `PlanExecutor` 注入 `RegisterPlanTaskTool`。
     4. 工具注册表经过与 sub-agent 相同的包装链:`WrapToolRegistry → TruncatingToolRegistry → DebuggingToolRegistry`。
     5. 通过 `agents.RegisterPrimary` + `desc.Factory(FactoryOptions{ToolRegistry: primaryFinalReg, ...})` 构造 Primary agent。
     6. 通过 `dispatches.WithPrimaryAssistant(primary)` 注入 Dispatcher,完成闭环。
   - `classical` 模式时,这段构造**完全跳过**(Primary 为零成本特性)。
6. **向后兼容**:
   - `mode=""`(未设置)或 `mode="classical"` 走旧流水线,所有 pre-M4 单测 / 集成测不改动地通过。
   - `registries.New()` 仍然只注册原有六个 agent;`RegisterPrimary` 只在 `mode=unified` 分支才调用 —— Primary 默认不出现在 `Dispatchable()` 列表(descriptor 的 `Dispatchable: false`),HTTP 层不会自动暴露。
7. **测试**:
   - 单测 `vv/agents/primary_test.go`:验证 `RegisterPrimary` 产出 ID/Name/Description/Profile 正确;Factory 构造的 agent 是 `taskagent` 且 `agent.StreamAgent`。
   - 单测 `vv/dispatches/primary_tools_test.go`:
     - 三个 `delegate_to_*` 工具被正确注册、schema 一致;handler 调 stub sub-agent 并返回其文本。
     - `plan_task` 工具的 JSON schema 与 handler 反序列化正确,且 handler 触发 `PlanExecutor.RunPlan` 一次。
   - 单测 `vv/dispatches/dispatch_primary_test.go`:
     - `mode=unified` + `primaryAssistant` 非空 → `Run`/`RunStream` 完全跳过 fastPath/intent/execute,只调用 Primary 一次。
     - `mode=classical` → Primary 即便注入也**不生效**(因为 `WithPrimaryAssistant` 只在 setup unified 分支被调用;单测显式不调用它)。
   - 集成测 `vv/integrations/dispatches_tests/dispatches_tests/primary_assistant_integration_test.go`:
     - `TestPrimary_AnswersDirectly`:Primary mock LLM 返回无工具调用的 assistant text → dispatcher 返回该 text、无 sub-agent 调用、一次 LLM。
     - `TestPrimary_DelegatesToCoder`:Primary mock LLM 先发 `delegate_to_coder` tool call → coder stub 被调用 1 次 → Primary 第二轮收到 tool_result → 产出最终文本。
     - `TestPrimary_PlanTask`:Primary mock LLM 发 `plan_task` tool call (2 步,depends_on) → DAG 被执行 → Primary 第二轮收 tool_result → 输出汇总。
     - `TestPrimary_DisabledInClassicalMode`:`mode=classical` 下 `primaryAssistant == nil`,流程与 pre-M4 完全一致。
8. **Lint/Test 全绿**:`cd vv && make test && make lint` + `cd vv/integrations/dispatches_tests && go test ./... && golangci-lint run`。

**Evidence of done**:
- 新增集成测 `TestPrimary_AnswersDirectly` 明确断言 `primaryLLM.callCount == 1`、`coder.callCount == 0`、`chat.callCount == 0`,且 `resp.Messages[0].Content.Text()` 等于 mock 返回文本。
- `go test ./dispatches/ -run Primary -v` 和 `go test ./agents/ -run Primary -v` 全通过。
- 设 `mode=classical`(或不设)时 pre-M4 的所有测试无改动地绿。

## Facts(from exploration)

- **Dispatcher 结构**(`vv/dispatches/dispatch.go:43-85`):字段集中 + `Option` pattern,新增 `primaryAssistant` 字段零侵入。
- **Run 入口**(`dispatch.go:284-334`):目前顺序为 `depth 检查 → fastPathClassify → recognizeIntent → (Mode=answered 捷径|executeTask) → summarize`。插入 Primary 分支的最佳位置:紧跟 `depth >= maxRecursionDepth` 的 fallback 之后。
- **unified_intent 事件形态**(`unified_intent.go:34`):`unifiedAnsweredSummary = "unified_intent -> answered"`;新增 `unified_primary` phase 不与之冲突。
- **agent-as-tool**(`vage/tool/agenttool/agent_tool.go:100-124`):`Register(*tool.Registry, agent.Agent, ...Option)` 支持自定义 parameter schema + ArgExtractor;完全满足三个 `delegate_to_*` 工具的需求。
- **现有 sub-agents**(`setup.go:113-190`):`coder`/`researcher`/`reviewer` 都是 `agent.Agent` 实例,直接拿 `subAgents["coder"]` 等注入 `agenttool.Register` 即可。
- **ToolProfile 构造**(`registries/tool_access.go:87-106`):`ProfileReadOnly.BuildRegistry(cfg.Tools, regOpts...)` 产出 read+glob+grep 的 `*tool.Registry`,已支持 PathGuard 注入。
- **taskagent.Option**(`vage/agent/taskagent/task.go:99 WithToolRegistry`):tool registry 是 `tool.ToolRegistry` 接口,与现有 wrapping 链路完全兼容。
- **runPlan 现状**(`vv/dispatches/execute.go:32` + `vv/dispatches/dag.go`):`runPlan(ctx, req, plan, upstream, contextSummary)` 已经是私有方法,输出 `*schema.RunResponse, error`;给一个 public 包装 `RunPlan(ctx, *Plan, *schema.RunRequest)` 即可,不必抽取内部逻辑。
- **深度控制**(`dispatch.go:314` `childCtx := IncrementDepth(ctx)`):Primary 调用 delegate_to 走 agent-as-tool 内部,**不会自动递增 depth** —— 这是漏洞。修复:在 `delegate_to_*` 的 handler closure 里手动 `ctx = IncrementDepth(ctx)` 再调 `agent.Run`。否则委派链路可以无限深。
- **没有循环依赖**:`vv/dispatches` 依赖 `vv/registries`、`vv/agents` 依赖 `vv/registries` —— Primary 的构造放 `vv/agents` 下(产出 descriptor + factory),委派工具放 `vv/dispatches`(持有 `PlanExecutor = *Dispatcher`)。setup 作为顶层胶水把两边拼起来,零包循环。
- **事件字段**(`vage/schema/...EventPhaseStart`):`PhaseStartData{Phase, PhaseIndex, TotalPhase}` —— 对 Primary 发 `PhaseIndex=1`、`TotalPhase=1` 最贴合"单相位"语义。
- **MaxIterations for Primary**:沿用 `cfg.Agents.MaxIterations`(默认 10);考虑到 Primary 可能会连发 `delegate_to → tool_result → 再调用`,10 轮是合理上限。

## Plan(sequenced)

1. **Config 层**
   - `vv/configs/config.go`:`OrchestrateConfig` 加 `Mode string`;常量 `OrchestrateModeClassical = "classical"`、`OrchestrateModeUnified = "unified"`;加 env override `VV_ORCHESTRATE_MODE`;`applyDefaults` 里若 `Mode==""` → 保持空(表示 classical,向后兼容);新增 `ValidateOrchestrateMode(s string) error` 在 `Load` 尾调用。
   - 单测 3 case:default/classical/unified 正常,未知值报错,env 覆盖 YAML。

2. **委派工具 + plan 工具**
   - 新建 `vv/dispatches/primary_tools.go`:
     - `type PlanExecutor interface { RunPlan(ctx context.Context, plan *Plan, req *schema.RunRequest) (*schema.RunResponse, error) }`
     - `RegisterDelegateTools(reg *tool.Registry, subAgents map[string]agent.Agent, ids []string) error` —— 对 `ids` 里的每个 agent id 调 `agenttool.Register` 加 `WithName("delegate_to_"+id)`、`WithDescription(...)`、自定义 parameter schema `{task, context}`、`WithArgExtractor`(拼接"Task: ...\nContext: ..."成 input string),且 handler 内 `IncrementDepth(ctx)`。
     - `RegisterPlanTaskTool(reg *tool.Registry, exec PlanExecutor) error` —— `schema.ToolDef{Name:"plan_task", Description:..., Parameters: {goal, steps}}`,handler 反序列化 args、调 `exec.RunPlan`,返回 `schema.TextResult`。
   - 单测 `primary_tools_test.go`:三个 stub `agent.Agent`,注册后 `reg.Get("delegate_to_coder").Handler(...)` 触发 stub 的 `Run`;plan_task 触发 `PlanExecutor.RunPlan`,断言参数传递正确。

3. **Primary Assistant**
   - 新建 `vv/agents/primary.go`:
     - `const PrimarySystemPrompt = ...`(描述职责 + 委派工具清单 + 何时不该委派)。
     - `RegisterPrimary(reg *registries.Registry)` —— `AgentDescriptor{ID:"primary", ToolProfile: ProfileReadOnly, Dispatchable: false, Factory: ...}`。Factory 接收 `opts.ToolRegistry`(由 setup 装配好)、构造 taskagent、`WithSystemPrompt`、`WithMaxIterations(opts.MaxIterations)`、`WithPromptCaching`、`WithHookManager`。
   - 单测 `primary_test.go`:descriptor 字段正确、Factory 在 `ToolRegistry=nil` 时能构造(但无工具)、`a.ID()=="primary"`。

4. **Dispatcher 接入**
   - `dispatch.go`:加 `primaryAssistant agent.Agent`;`WithPrimaryAssistant(a agent.Agent) Option`。
   - 在 `Run` 顶部 `depth >= maxRecursionDepth` 分支之后插入:
     ```go
     if d.primaryAssistant != nil {
         return d.runPrimary(ctx, req)
     }
     ```
   - 同理 `RunStream`。
   - 新建 `vv/dispatches/primary.go`:
     - `func (d *Dispatcher) runPrimary(ctx, req) (*schema.RunResponse, error)` —— 直接 `d.primaryAssistant.Run(ctx, req)`。
     - `func (d *Dispatcher) runPrimaryStream(ctx, send, req) error` —— 发 `EventPhaseStart{"unified_primary"}`、把 primary 的 stream 透传、发 `EventPhaseEnd`。
     - `func (d *Dispatcher) RunPlan(ctx, plan *Plan, req *schema.RunRequest) (*schema.RunResponse, error)` —— 薄 public 包装,调现有 `d.runPlan(ctx, req, plan, nil, "")`;用于满足 `PlanExecutor` 接口。
   - 单测 `dispatch_primary_test.go`:mock primaryAssistant,设 `mode=unified` + `WithPrimaryAssistant(mock)` + 注入任意 fastPath/intent mock LLM → 确认 intent mock 未被调用、primary mock 被调用 1 次。

5. **Setup 接线**
   - `setup.go:New`:在构造 subAgents 之后、planner / planGen 之前,加一个分支:
     ```go
     if cfg.Orchestrate.Mode == configs.OrchestrateModeUnified {
         primary, err := buildPrimaryAssistant(cfg, llm, regOpts, subAgents, dispatcherPlaceholder, opts)
         dispatcherOpts = append(dispatcherOpts, dispatches.WithPrimaryAssistant(primary))
     }
     ```
   - 棘手点:Primary 的 `plan_task` 工具需要 `PlanExecutor = *Dispatcher` 的引用,但 Dispatcher 是在 Primary **之后** 才构造的。解决:
     - 方案 A:在 `primary_tools.go` 里用一个 `*DispatcherRef` 指针包装器(延迟绑定);setup 先创建空 ref、构造 Primary、构造 Dispatcher、然后 `ref.Set(d)`。
     - 方案 B:先构造 Dispatcher(不带 Primary)、再构造 Primary(持有 Dispatcher 引用)、再 `dispatcher.SetPrimaryAssistant(primary)`(新增 setter)。
     - **采用方案 B** —— 更直观、无新类型;`SetPrimaryAssistant(a agent.Agent)` 是 option 的 post-hoc 等价物,只在 unified 分支调用,classical 路径零影响。
   - 改造 setup 顺序:
     1. 构造 subAgents。
     2. 构造 explorer/planner/planGen。
     3. `dispatches.New(...)` 得到 `d`(不带 Primary)。
     4. 若 `mode=unified` → 构造 Primary 的 tool registry(含 `RegisterDelegateTools(reg, subAgents, []string{"coder","researcher","reviewer"})` + `RegisterPlanTaskTool(reg, d)`),包装权限/截断/debug,用 `agents.RegisterPrimary` 得 descriptor、调 Factory 得 `primary`,`d.SetPrimaryAssistant(primary)`。
   - 单测 setup 层用已有 helper 验证 `mode=unified` 分支 Dispatcher 的 primary 字段非空、classical 分支为 nil。

6. **集成测**
   - `vv/integrations/dispatches_tests/dispatches_tests/primary_assistant_integration_test.go`:复用 `sequentialMockLLM`,注入一个假的 Primary(本身也是 TaskAgent 包 mock LLM),加上 coder stub。四个 case 与 Done Contract 第 7 条一一对应。

7. **Validate**
   - `cd vv && make test && make lint`。
   - `cd vv/integrations/dispatches_tests && go test ./... && golangci-lint run`。

8. **Reverse Sync**
   - 写 `result.md`:改动清单 + 文件行数 + 测试结果 + 与计划的偏差 + M5 衔接建议(默认切 unified / legacy event 映射层)。

## Risks

- **Primary 的 system prompt 过长**:一旦描述三个 delegate_to + plan_task + 只读工具的使用时机 + 谁不该委派,就容易膨胀到 2-3k tokens,抵消单次调用的节省。
  - 缓解:system prompt 精简到 <800 tokens,详细使用指引放到各自工具的 `description` 字段里(LLM 会自动读),并打开 prompt caching。
- **plan_task 里的 Dispatcher 反向引用**:setup 里先建 Dispatcher 再塞 Primary,看起来像循环;但 primary.tool.plan_task.handler 只在**运行期**调 Dispatcher.RunPlan(那时 Dispatcher 已完全就绪),构造期无循环,安全。
- **深度失控**:agent-as-tool 默认不递增 depth。必须在 `delegate_to_*` 的自定义 handler 里显式 `IncrementDepth(ctx)`,否则 Primary → coder → 再回调 Primary 的 delegate 链可以无限深。测试要覆盖 `depth >= maxRecursionDepth` 时 delegate 是否退化为 fallback(本期先检测 depth,直接把 tool_result 返回 "recursion depth exceeded",让 Primary 自己处理)。
- **Permission wrapping 错位**:Primary 的工具也必须经过 `opts.WrapToolRegistry`(CLI 权限层)。如果 setup 代码走了一条捷径没包装,则 Primary 会绕过用户确认。计划里明确包装链路与 sub-agent 同形。
- **`classical` 模式回归**:引入的 Option 默认零值时行为必须和 pre-M4 完全一致。单测覆盖:`Dispatcher.New(...)` 未调 `WithPrimaryAssistant` 的既有测试一律不改动通过。
- **HTTP 事件订阅者**:现在订阅 `EventPhaseStart{Phase:"intent"}` 的客户端在 unified 模式下收不到 intent 事件。这是 M4 的已知行为(见 design doc §6.1);本期不做 shim。
- **mode 写错**:用户把 `mode: unifiec` 打错 → 我们在 `Load` 里明确报错(而不是静默 fallback 到 classical),防止用户以为在用新路径但其实没。
- **plan_task 的 `steps[].agent` 校验**:Primary 的 LLM 可能在 plan_task 参数里传 `"agent": "primary"` 或未知 agent。handler 在调用 `exec.RunPlan` 之前需复用 `IntentResult.validate` 路径,遇错返回结构化 tool_error 让 Primary 改正。

## Checkpoint — awaiting approval before implementation

- 当前理解(Restate):加一个 `orchestrate.mode: unified` 灰度开关。开启后 Dispatcher 不再走 fastPath/intent/execute/summarize,而是把整个请求交给 Primary Assistant —— 一个 TaskAgent,自带 read/glob/grep/todo/ask_user 工具,外加 3 个 `delegate_to_<agent>` 工具(coder/researcher/reviewer)和 1 个 `plan_task` 工具。Primary 自主决定:直接回答 / 委派 / 发计划。classical 模式保持不变。
- Core goal:用户配了 `mode: unified` → dispatcher 一路走到 Primary;dispatch 级的 phase 事件换成 `"unified_primary"`;三条代表路径(直接答 / 委派 coder / 下 plan)都打通,并各有一个集成测证明。
- Next 3 actions:(1) 加 config `OrchestrateConfig.Mode` + env + validation;(2) 实现 `primary_tools.go`(delegate_to_* + plan_task)并配好深度控制;(3) 实现 `agents/primary.go` 和 dispatcher 的 `WithPrimaryAssistant`/`runPrimary`/`runPrimaryStream`/`RunPlan`,setup 里按 mode 组装。
- Risks + mitigations:见 Risks 一节。关键一个是"agent-as-tool 不自增 depth",已规划在 delegate handler 里手动增。
- Validation:`make test` + `make lint` 两个 module 全绿;新增 4 个集成测 + 5-8 个单测明确断言事件序列与 call count。

**Open questions for the user(默认值在中括号,用户不表态即按默认推进)**:

1. **Config key**:`orchestrate.mode`(与 `fast_path`/`unified_intent`/`router` 同级)还是独立顶层 `dispatcher.mode`?[默认 `orchestrate.mode` —— 与 M1/M2/M3 的 YAML 布局保持一致。]
2. **mode 取值**:是否接受 `"classical"` 作为显式值,还是只接受空字符串表 classical?[默认**两者都接受**,`""` 是隐式 classical,`"classical"` 是显式,`"unified"` 是新路径。]
3. **Primary 的写工具**:本期是否给 Primary 任何写权限(edit/write/bash)?[默认**不给**—— 所有写操作必须走 delegate_to_coder,这样权限审计链路保持单一。]
4. **delegate_to 工具粒度**:一个 `delegate_to(agent, task)` 工具带 `agent` 参数,还是三个分开的 `delegate_to_coder` / `delegate_to_researcher` / `delegate_to_reviewer`?[默认**三个分开** —— LLM 的 tool-calling 语义下,分开的工具对模型更直观,tool description 也更精准;单工具需要 LLM 在 args 里挑 agent,类似 M2 的 delegate_to,重复了决策层。]
5. **plan_task 的执行**:走 Dispatcher 现有的 `runPlan` 还是直接调 `orchestrate.ExecuteDAG`?[默认**走 `Dispatcher.RunPlan`** —— 复用现有 replanning / aggregator / plan summarizer,零新 DAG 代码。]
6. **`unified_primary` phase 命名**:用 `"unified_primary"` 还是 `"primary"`?[默认 `"unified_primary"` —— 与 M2 的 `unified_intent` 命名风格对齐,强调是"unified 模式下的"相位。]
7. **是否同步更新 `classical` 路径使其能被 M5 无损下线**?[默认**不改** —— 本期专注新增 Primary 路径,对 classical 零侵入;M5 才决定下线节奏。]
8. **集成测覆盖面**:是否要补一个"Primary 在 depth >= max 时退化为 fallbackAgent"的测试?[默认**是** —— 作为 invariant guard,避免未来在递归控制上踩坑。]
