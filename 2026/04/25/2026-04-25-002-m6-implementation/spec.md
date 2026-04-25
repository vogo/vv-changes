# M6 — Primary `allow_bash` + Classical 代码彻底移除 + 真实 LLM Golden CI · Spec

> Session: `2026-04-25-002-m6-implementation`
> Status: Draft (awaiting approval)
> Date: 2026-04-25
> Mode: sdd-lite `deep`
> Spec is Truth · No Approval, No Execute

---

## 1. 任务复述

来源:
- `doc/design/enhance-prompt-understand-and-solve.md` §8.4(M6 草案三项)
- `changes/2026/04/25/2026-04-25-001-m5-default-unified/result.md` 「M6 衔接建议」(五项)

去重合并后实际是 6 个工作包:

| ID | 工作包 | 来源 |
|----|--------|------|
| **G1** | `primary.allow_bash` 开关 — Primary 工具集可选挂载 bash | §8.4 ①, 衔接 #1 |
| **G2** | 彻底移除 chat / explorer / classical intent / summarize-as-phase 代码 | §8.4 ②, 衔接 #2 |
| **G3** | `legacy_phase_events` 弃用路径(M6 加 `slog.Warn`,M7 移除) | 衔接 #4 |
| **G4** | unified fallback path 的 summarize 同形事件(若需要) | 衔接 #5 |
| **G5** | Golden baseline 接真实 LLM 的 weekly-cron CI job | §8.4 ③, 衔接 #3 |
| **G6** | `fastpath.go:71` 的 `Agent:"chat"` 硬编码改用 `fallbackAgentName()`(G2 子任务,但需独立列出便于追踪) | 衔接 #2(b) |

## 2. 核心目标(Loop Anchor)

让 vv 默认路径上**只剩 unified**:Primary Assistant + delegate_to / plan_task 工具集 + 退化 Primary 兜底。
classical 完全移除,代码库不再保留两套并存的派发流水线;Primary 自己可以在用户开启时跑单行 bash;线上有真实 LLM golden 性能基线作为后续优化的对比尺。

## 3. 边界与硬约束

### 3.1 范围内
- `vv/dispatches/` — 删除 classical 专属代码,保留 unified + fastpath。
- `vv/agents/` — 删除 `chat.go` / `explorer.go`,保留 coder / researcher / reviewer / planner / primary。
- `vv/setup/setup.go` — `cfg.Orchestrate.Mode` 分支收敛为单一 unified 路径。
- `vv/configs/config.go` — `OrchestrateModeClassical` 常量保留(deprecation),`Validate` 把 classical 视为 `unified` 并发 `slog.Warn`。
- 测试文件级联删除/重命名。
- `.github/workflows/` 新增 weekly-cron CI(若仓库已有 CI 基础设施)。

### 3.2 范围外(本次不做)
- 删除 `OrchestrateModeClassical` **常量本身** — 给一个迭代周期作为缓冲(M7 删)。
- 删除 `LegacyPhaseEvents` 字段本身 — M6 只加 deprecation,M7 删。
- 删除 `Dispatcher.fallbackAgent` 字段或 `WithFallbackAgent` Option — 仍有 fastpath / depth-exceed 兜底使用,只是兜底对象统一为退化 Primary。
- vage / aimodel 层零改动。

### 3.3 硬约束
- 公共 API 只允许新增/标注 deprecation,不允许签名破坏。CLI / HTTP / 集成测试若编译失败必须立即修复或在 spec 中记下偏差。
- `vv/integrations/golden_tests/` 在 mock LLM 路径上保持现状(继续是回归闸门),只新增真实 LLM CI job。
- 任何对 `OrchestrateModeClassical` 的引用必须改路由到 unified;**绝不改默认行为(已是 unified)**。
- 删除前必须先把对外可见的 prompt 常量(`agents.ChatSystemPrompt` 被 `golden_cases_test.go:71` 引用)迁出 chat.go,搬到 primary.go 或独立 fallback prompt 文件,避免 import 失败。

## 4. 行业方案对比与取舍(deep 必备)

### 4.1 G1(allow_bash)的几种实现

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| A. 配置开关静态决定 | `configs.OrchestrateConfig.PrimaryAllowBash bool`,`buildPrimaryAssistant` 启动时决定是否挂 bash 工具 | 实现最直接,与 `WithPermission` 兼容 | 运行时切换需重启 |
| B. CLI permission-mode 联动 | 在 `--permission-mode=accept-edits` 下自动开启,`plan` 下自动关 | 与 CLI 已有体系一致 | 与 HTTP 模式语义难统一 |
| C. Primary 自己 delegate_to bash agent | 不改工具集,新增一个 `bash-only` sub-agent 作为 delegate 目标 | 工具集干净 | 多一次 LLM 调用,失去本项目的目的(衔接建议 #1 明确说"无需 delegate") |

**取**: A。最直白、最贴合 §8.4 描述,默认关闭,通过 env `VV_PRIMARY_ALLOW_BASH` 也支持运行时覆盖以便排查。

### 4.2 G2(classical 删除)的几种节奏

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| α. 一次性大刀 | 一个 commit 删除全部 classical 代码 + 测试 | review 一目了然 | diff 巨大,回退颗粒粗 |
| β. 三步走 | (1) 把 unified 唯一路径锁死(setup.go 永远走 unified)+ deprecation Warn;(2) 删除 chat/explorer/intent/summarize-classical;(3) 清理测试与文档 | 每步可独立 ship,风险可控 | commit 数多,但同一 dev session 内可串成有序 patch |
| γ. 仅做 deprecation,不删 | 只加 Warn,等下个迭代再删 | 风险最低 | 没真正完成任务 |

**取**: β。在同一 PR / 同一 dev session 内顺序提交,每步绿测。Resume Anchor 也按 β 的检查点写。

### 4.3 G5(真实 LLM CI)实现选项

| 方案 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| I. GitHub Actions weekly cron + secret API key | `.github/workflows/golden-real-llm.yml`,`schedule: cron: "0 3 * * 1"` + `secrets.AI_API_KEY` | 与现有 CI 一致;数据可上传 artifact | 需仓库 admin 配 secret;成本约 $0.X/周 |
| II. 仅写脚本,运维手动跑 | `vv/scripts/run-golden-real-llm.sh` | 零 secret 配置 | 没人手动跑就废 |
| III. 在每个 PR 跑 | 每次 PR 都打一次真实 LLM | 实时反馈 | 成本爆炸,且 LLM flake 会卡 PR |

**取**: I + II 双轨。Workflow 文件 + 脚本同时给。如果用户当前 repo 的 GH secret 没就绪,workflow 默认 manual `workflow_dispatch` 触发,等 secret 配好再加 cron(spec Risk 一节会标这一条)。

## 5. 设计决定

### 5.1 G1: `primary.allow_bash`

```go
// configs/config.go
type OrchestrateConfig struct {
    Mode                string `yaml:"mode"`
    LegacyPhaseEvents   bool   `yaml:"legacy_phase_events"`
    PrimaryAllowBash    bool   `yaml:"primary_allow_bash"` // NEW: M6
}
```

环境变量: `VV_PRIMARY_ALLOW_BASH=true|false`(若设置则覆盖 YAML)。
Default: `false`。

`setup/setup.go` 在 unified 分支构造 Primary 时,根据 `cfg.Orchestrate.PrimaryAllowBash`:
- `false`(默认)→ Primary tools = `read,grep,glob,delegate_to_*,plan_task,todo_*`(与 M5 同)。
- `true` → 在以上基础上追加 `bash`(走与 coder 相同的 bash tool descriptor,但仍受 CLI `WrapRegistryWithPermission` 约束)。

退化 Primary(`buildFallbackPrimary`)**始终不挂 bash**,与 R2 安全属性一致。

测试:
- `setup_test.go` 加 `TestNew_UnifiedMode_PrimaryAllowBash_AddsBashTool`。
- `agents/primary_test.go` 验证默认无 bash。

### 5.2 G2 + G6: 移除 classical 路径

**保留并迁移:**
1. `recognizeIntentUnified`、`BuildUnifiedIntentSystemPrompt`、`UnifiedTool*` 常量 — 保留在 `unified_intent.go`(已自包含,无需迁移)。
2. `fallbackAgentName()`(intent.go:331)、`Dispatcher.fallbackAgent` 字段、`WithFallbackAgent` Option — 迁移到 `dispatch.go`(此 helper 不属于 classical intent,与 fastpath 共用)。
3. `agents.ChatSystemPrompt` 常量(被 `golden_cases_test.go:71` import)— 迁到一个新文件 `agents/fallback_prompt.go`,导出为 `agents.FallbackChatPrompt`,旧名标 deprecated alias 保留 1 个迭代。

**删除清单:**

| 文件 | 操作 |
|------|------|
| `vv/dispatches/intent.go` | **删除整个文件**;在删除前把 `fallbackAgentName` 迁到 `dispatch.go` |
| `vv/dispatches/intent_test.go` | **删除** |
| `vv/dispatches/summarize.go` | **保留** — fastpath 与 unified Primary 在某些路径仍触发(dispatch.go:393, fastpath.go:195/271);但内部把 "summarizer agent" 默认绑到 Primary 自身的 final 处理上 |
| `vv/dispatches/summarize_test.go` | **保留** |
| `vv/dispatches/router_llm_test.go` | **保留**(M2 unified path 仍有测试覆盖,需检查是否删 classical-only test cases) |
| `vv/agents/chat.go` | **删除**(把 `ChatSystemPrompt` 迁到 `fallback_prompt.go`) |
| `vv/agents/chat_test.go` | **删除** |
| `vv/agents/explorer.go` | **删除** |
| `vv/agents/explorer_test.go` | **删除** |
| `vv/agents/agents_test.go` | 删除 RegisterChat/RegisterExplorer 相关 case |
| `vv/agents/project_instructions_test.go` | 删除 chat / explorer 相关 case |
| `vv/setup/setup.go` | 移除 `agents.RegisterChat/Explorer` 调用;`cfg.Orchestrate.Mode` 分支收敛 |
| `vv/integrations/setup_tests/project_instructions_tests/...` | 同步删除 chat/explorer 用例 |
| `vv/integrations/setup_tests/setup_tests/setup_prompt_caching_test.go` | 同步 |
| `vv/integrations/golden_tests/golden_tests/golden_cases_test.go` | `ChatSystemPrompt` import 改 `FallbackChatPrompt`;**`runClassical` 与 5 case 中 classical 部分整体删除**,只保留 unified + 1 个保留为 "consistency 回归"(如 `Greeting_Hello_Unified`) |
| `vv/integrations/golden_tests/golden_tests/golden_helpers_test.go` | 删除 classical mock LLM 序列构造器 |
| `vv/dispatches/dispatch.go` `Run` / `RunStream` | classical 分支彻底删除;`if d.primaryAssistant == nil` 路径退化为"返回错误或直接走退化 Primary" |

**`fastpath.go:71` 改:**
```go
// before
Agent: "chat",
// after
Agent: "", // empty = use Dispatcher.fallbackAgentName() at execution time
```
`Dispatcher` 内部解析 fastpath rule 时,若 `Agent==""` 则 `agent = d.fallbackAgentName()`。

`forwardSubAgentStream` 调用点的字面 `"chat"` label,统一替换为 `d.fallbackAgentName()`(运行时取),保证 SSE 事件 phase 字段反映真实 fallback agent ID。

**Mode 校验:**

```go
// configs/config.go
func ValidateOrchestrateMode(mode string) string {
    switch mode {
    case "":
        return OrchestrateModeUnified
    case OrchestrateModeUnified:
        return OrchestrateModeUnified
    case OrchestrateModeClassical:
        slog.Warn("orchestrate.mode=classical is deprecated as of M6 and now routes to unified; remove the setting from your config")
        return OrchestrateModeUnified
    default:
        slog.Warn("unknown orchestrate.mode value; falling back to unified", "value", mode)
        return OrchestrateModeUnified
    }
}
```

迁移 log(M5 加的 `Load` 里那段)简化:**移除**(因为没有真正的 classical 行为可保了,Warn 已经在 Validate 里给了)。

### 5.3 G3: `legacy_phase_events` 弃用 slog.Warn

位置: `dispatches/primary.go:66`(开关唯一消费点),改成:

```go
if d.legacyPhaseEvents {
    slog.Warn("legacy_phase_events shim is deprecated as of M6 and will be removed in M7; migrate HTTP consumers to unified_primary phase events")
    return d.runPrimaryStreamLegacy(...)
}
```

测试:`primary_test.go` 加 `TestRunStream_LegacyShim_EmitsDeprecationWarning`(用 captureLogs helper)。

### 5.4 G4: unified fallback path summarize 同形

**核实步骤优先于动手**: 先用 `golden_tests` 跑 `depth-exceeded` case 看 unified fallback 实际事件序列。若 SSE 事件流缺 `summarize` phase,补一个空壳 phase(`Phase: "summarize", Duration: 0, Summary: "unified fallback: no summarization"`)以与 classical 同形。

若核实结论是 "无消费者依赖" 或 "已被 fastpath summarize 覆盖",此 G4 项 **降级为 doc-only**:在 design 文档 §8 加一段 explicit note "退化 Primary 路径无 summarize 事件;HTTP 消费者请订阅 `unified_primary` end 事件"。

### 5.5 G5: 真实 LLM Golden CI

新增:
- `vv/scripts/run-golden-real-llm.sh` — 本地脚本。读 `AI_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`,跑 `vv/integrations/golden_tests/real_llm_tests/`(本期新建),输出 latency / tokens 表格到 stdout + `vv/integrations/golden_tests/real_llm_tests/baseline.json`。
- `vv/integrations/golden_tests/real_llm_tests/real_llm_baseline_test.go` — 5 个 case 与 mock 集合一一对应,Skip 当 `AI_API_KEY` 未设。
- `.github/workflows/golden-real-llm.yml`(若仓库已有 .github/workflows 目录)— `workflow_dispatch` + `schedule: cron: "0 3 * * 1"`(每周一 UTC 3am),secret `AI_API_KEY`。
- 文档: 在 `vv/integrations/golden_tests/README.md`(新建,如不存在)中说明使用方式与判读基准。

### 5.6 G6: 已并入 G2 的 fastpath label 修改

无独立设计,见 5.2。

## 6. Done Contract

任意一项不通过 = 未完成,需要继续。

| # | 检查 | 证据 |
|---|------|------|
| D1 | `cfg.Orchestrate.PrimaryAllowBash=true` 时 Primary 工具集含 bash;`false` 不含 | `setup_test.go` 新测试通过 |
| D2 | `vv/agents/chat.go` 与 `vv/agents/explorer.go` 不存在 | `ls vv/agents/` |
| D3 | `vv/dispatches/intent.go` 不存在(`fallbackAgentName` 已迁移到 `dispatch.go`) | `ls vv/dispatches/` |
| D4 | `cfg.Orchestrate.Mode = "classical"` 在启动期发 `slog.Warn` 且实际行为 = unified | 单测 `TestValidateOrchestrateMode_ClassicalIsDeprecatedAndAliasesUnified` |
| D5 | `dispatcher.legacyPhaseEvents=true` 启动后发一条 deprecation `slog.Warn` | 单测 `TestRunStream_LegacyShim_EmitsDeprecationWarning` |
| D6 | `fastpath.go` 不再含字面 `Agent: "chat"`;运行时 `fallbackAgentName()` 决定 | `grep -n '"chat"' vv/dispatches/fastpath.go` 仅剩 SSE label 或全部消失 |
| D7 | `cd vv && make test` 全绿 | terminal 输出 |
| D8 | `cd vv && make lint` 0 issues | terminal 输出 |
| D9 | `cd vage && make test`、`cd aimodel && make test` 全绿(零回归) | terminal 输出 |
| D10 | `cd vv/integrations && make test`(若有此 target,否则各子模块 `make test`)全绿 | terminal 输出 |
| D11 | Golden mock 测试 5 case 全绿,且 classical 变体已删/合并 | golden_cases_test.go 输出 |
| D12 | `vv/integrations/golden_tests/real_llm_tests/` 在 `AI_API_KEY` 未设时 Skip;设了能跑通至少 1 个 case(本地手测) | local skip 行为 |
| D13 | `.github/workflows/golden-real-llm.yml` 存在(若该目录存在) | `ls .github/workflows/` |
| D14 | `result.md` 完整记录改动清单、Evidence、与 spec 偏差 | 文件存在且非空 |
| D15 | `doc/design/enhance-prompt-understand-and-solve.md` 加 §9 M6 Post-landing notes | 文件存在 §9 |

## 7. 计划(执行顺序)

按 4.2 β 分步,每一步要求 vv `make test` 全绿:

1. **Step 1 — Mode validation deprecation + 移除 Load 迁移日志**(G2 子任务,纯加 Warn,不改路径)
2. **Step 2 — `legacy_phase_events` deprecation Warn**(G3,~30 行)
3. **Step 3 — `primary.allow_bash` 开关**(G1,~80 行)
4. **Step 4 — `fastpath.go` label 与 fallback name 动态化**(G6,~30 行)
5. **Step 5 — 收敛 setup.go Mode 分支为单一 unified**(G2 中段,中风险)
6. **Step 6 — 删除 chat/explorer 与级联测试**(G2)
7. **Step 7 — 删除 intent.go(保留 fallbackAgentName 已迁移,recognizeIntentUnified 已自包含)**(G2)
8. **Step 8 — 核实 + 处理 unified fallback summarize 同形**(G4)
9. **Step 9 — 真实 LLM golden 脚本 + workflow**(G5)
10. **Step 10 — 文档(§9 M6 Post-landing notes)+ result.md**(收尾)

每 Step 一次 `cd vv && go test ./...` 验证。Step 5/6/7 是高风险段,完成后追加 `cd vage && make test && cd ../aimodel && make test`。

## 8. 风险与缓解

| 风险 | 缓解 |
|------|------|
| **R1**: 删 chat agent 后 `golden_cases_test.go` 引用 `agents.ChatSystemPrompt` 编译失败 | 先迁常量到 `agents/fallback_prompt.go`,导出新名 `FallbackChatPrompt`,golden 测试一并更新;chat.go 里只在最后一步删 |
| **R2**: `recognizeIntentUnified` 与 `fallbackAgentName` 在 intent.go 里被一起搬;漏迁导致编译失败 | Step 7 先 grep 所有 caller,迁完再删 |
| **R3**: HTTP 消费者依赖 `intent` / `summarize` SSE phase 名 | M5 `legacy_phase_events` shim 仍保留(只加 deprecation),消费者迁移期不被破坏 |
| **R4**: 真实 LLM workflow 暴露 secret 配置错误 | 默认 `workflow_dispatch`,cron 注释掉,等仓库 admin 配 secret 后再开 |
| **R5**: 删除范围过大导致一次 PR review 困难 | 分 10 个 step 顺序提交;最终 PR 也可拆 |
| **R6**: classical mode 用户在升级时无声破坏 | `ValidateOrchestrateMode` 见到 classical 必发 Warn 且行为转 unified,避免 silent change;但 **API 行为变了**(原本可能存在的 SSE phase 名差异),升级 note 必须写进文档 |
| **R7**: 删除后 `OrchestrateModeClassical` 常量仍存在,有人误以为可用 | 常量上加 `// Deprecated: M6+ 等价于 OrchestrateModeUnified;M7 移除`,godoc 显示 |

## 9. 决策点(需要用户审批)

请用户对以下 5 项分别明示 OK / 修改:

1. **G2 节奏**:选 β(分 10 步同 PR 渐进)— 是否同意?
2. **`OrchestrateModeClassical` 常量保留 1 迭代周期**(M7 删)— 是否同意?如不同意,本期一并删除。
3. **G4 处理**:先观测,若无消费者则降级为 doc-only — 是否同意?
4. **G5 CI**:本仓库 `.github/workflows/` 是否存在且允许 admin 配 `AI_API_KEY` secret?如不允许,G5 退化为 "脚本+本地说明"。
5. **`agents.ChatSystemPrompt` → `agents.FallbackChatPrompt` 重命名**:是否同意?或保留旧名?

## 10. Resume Anchor(Step Checkpoint)

| Step | Status | Note |
|------|--------|------|
| 1 | pending | Mode validation deprecation |
| 2 | pending | legacy_phase_events Warn |
| 3 | pending | primary.allow_bash |
| 4 | pending | fastpath label dynamic |
| 5 | pending | setup.go mode 分支收敛 |
| 6 | pending | 删 chat/explorer |
| 7 | pending | 删 intent.go |
| 8 | pending | unified fallback summarize 同形 |
| 9 | pending | 真实 LLM golden CI |
| 10 | pending | 文档 + result.md |

中断时把状态填到这张表,下次 resume 直接接着干即可。

---

**等待批准 — `No Approval, No Execute`**

---

## 11. Approved Decisions (2026-04-25)

| # | 决策 | 用户回复 | 落地影响 |
|---|------|----------|----------|
| 1 | G2 节奏选 β(分 10 步同 PR 渐进) | OK | 按 §7 计划执行 |
| 2 | `OrchestrateModeClassical` 常量去留 | **本期一起删** | §3.2 "范围外" 中此条改为 "范围内";Step 7 末尾追加 "删除常量 + 调整 `ValidateOrchestrateMode` 签名/语义";所有引用处级联清理(下表) |
| 3 | G4 unified fallback summarize | 先观测,无消费者降级为 doc-only | 按 §5.4 既定方案 |
| 4 | G5 真实 LLM CI | 可以(`.github/workflows/` 与 secret 可配) | 按 §5.5 实施,workflow 启用 cron |
| 5 | `ChatSystemPrompt` → `FallbackChatPrompt` 重命名 | OK | 按 §5.2 R1 缓解执行 |

### 决策 #2 详细落地(本期删除 `OrchestrateModeClassical`)

> **顺序微调**(用户回复 B 后):常量**定义本身**保留到 Step 7 末尾再删,以确保每个 Step 单独编译/测试通过。Step 1 只让 `ValidateOrchestrateMode` 把 classical 视为 unified + Warn;Step 5 删 setup 与 golden test 中的 classical 字面引用;Step 7 末尾删除常量定义、删除最后剩余的引用。

引用面已扫描:

| 位置 | 处理 |
|------|------|
| `vv/configs/config.go:99` 常量定义 | 删除 |
| `vv/configs/config.go:79, 96-118, 755` 注释 + Validate 分支 + Load 迁移日志 | 删除 / 重写 — `ValidateOrchestrateMode` 收敛为 "空 → unified;`unified` → unified;其它 → Warn + unified" |
| `vv/configs/config_test.go:1297-1404` 4 处测试 | 删除/改写为 unified-only |
| `vv/configs/config_test.go:1422-1425` YAML 检测 fixture | 改成检测 `unified` 或删除(因为 M5 加的迁移日志一并删掉) |
| `vv/setup/setup_test.go:413` `Mode: OrchestrateModeClassical` 用例 | 删除该测试用例(unified-only setup 不再有 classical 分支) |
| `vv/integrations/golden_tests/golden_tests/golden_cases_test.go:130, 498` `mode: "classical"` | 删除(classical case 整体删除,5 case 全部 unified-only) |

新增 deprecation 行为:`ValidateOrchestrateMode("classical")` → `slog.Warn("orchestrate.mode=classical is no longer supported as of M6; routing to unified") + return OrchestrateModeUnified`(不报错,平滑升级)。同步在 §5.2 落实。

`yamlHasExplicitOrchestrateMode` 函数(M5 加)与 `Load` 中 M5 迁移日志一并删除 — 因为 unified 是唯一行为,无需引导用户去显式锁 classical。

### 决策 #4 G5 落地确认

启用 GitHub Actions cron(`schedule: cron: "0 3 * * 1"`),secret 名 `AI_API_KEY`(若现有 secret 名不同,实现时通过 `gh secret list` 校准)。同时给 `workflow_dispatch` 让管理员可手动触发。

---

**已批准 — 进入执行,从 Step 1 起步。**
