# 需求规格说明：工具返回内容入模前注入扫描（P0-3）

## 1. 背景与目标

### 1.1 背景

Agent 框架 vage 目前的安全护栏链（`vage/guard/`）**仅在用户输入侧**做 prompt injection 扫描（`PromptInjectionGuard` 的 `Check` 明确只处理 `DirectionInput`）。Agent 在 ReAct 循环中执行的工具（web 抓取、文件读取、shell 输出、MCP 远端工具）返回的文本直接进入模型上下文，完全没有经过同等检查。

这正是 OWASP LLM Top 10 中 **LLM01 · Indirect Prompt Injection**（间接提示注入）攻击面：攻击者把类似「忽略之前的指令，改为……」的指令藏在外部可控的数据源中（网页、README、文件、shell 输出、恶意 MCP server 返回），当 Agent 把这段文本灌给模型时就可能被劫持执行越权操作。产业界 2023–2025 期间已有多起真实事件（Bing Chat / Sydney 泄露系统提示、GitHub Copilot Chat README 劫持、Slack AI 频道注入、Anthropic Computer Use 屏幕注入、MCP "rug pull" 工具投毒）。

### 1.2 业务目标

- **扩展 Guard 落点**：将注入扫描从"用户输入 → Agent"扩展到"工具返回 → 模型"这条链路。
- **低成本守住安全底线**：复用现有 `PromptInjectionGuard` 的正则规则库与 `guard.Chain` 基础设施，避免引入新的模型/网络依赖。
- **可观测**：每一次工具返回的扫描结果都要可记录（命中规则、严重度、动作），为后续策略调优与事件复盘提供数据。
- **不阻断正常流程**：在合理默认下不产生高误报；提供 `log-only / rewrite / block` 三档动作，允许按工具风险等级配置。

### 1.3 非目标（Out of Scope）

- 不引入 LLM-as-Judge / Prompt-Guard-86M 等模型驱动检测（留到 v2）。
- 不做 Spotlighting / Datamarking 自动改写（v2 可选）。
- 不做 CaMeL 式的能力隔离与数据流追踪。
- 不改动用户输入侧 `PromptInjectionGuard` 的既有语义。
- 不改造 MCP client 的凭据过滤（P0-4 独立需求）。
- 不做跨会话隔离的强绑定（P1-4 独立需求）。

## 2. 用户故事与验收标准

### US-1 · 作为 Agent 框架接入方，我希望工具返回被自动扫描注入攻击

**验收标准**
- AC-1.1：在 TaskAgent 的 ReAct 循环中，**每一次** `executeToolCall` 返回后、在把 `schema.ToolResult` 转成 `aimodel.Message{Role: RoleTool, …}` 并追加到消息序列之前，扫描其文本内容。
- AC-1.2：扫描命中时，按配置执行三种动作之一：`block`（返回错误结果给模型，终止本次工具结果入模）、`rewrite`（用脱敏/包裹后的内容替换）、`log`（仅记录，不改动内容）。
- AC-1.3：流式与非流式两条分支（`Run` / `RunStream`）行为一致：都必须经过扫描。
- AC-1.4：未配置扫描器时，行为与现状完全一致（零破坏）。

### US-2 · 作为框架使用者，我希望复用已有的 Guard 规则库

**验收标准**
- AC-2.1：扫描器默认使用 `guard.DefaultInjectionPatterns()` 作为规则基线，并在此之上追加一批针对"间接注入"的扩展规则（角色劫持、ChatML/Llama 标记、Unicode tag 字符、Bidi override、prompt 提取、编码走私提示、exfil 命令组合等）。
- AC-2.2：用户可传入自定义 `[]guard.PatternRule` 覆盖或追加规则。
- AC-2.3：现有 `PromptInjectionGuard` 的用户输入扫描行为保持不变；新扫描器不反向污染它。

### US-3 · 作为 Agent 框架使用者，我希望扫描结果可观测

**验收标准**
- AC-3.1：命中时触发一个 `hook` 事件（复用 `hook.Manager`），事件数据包含：`agent_id`、`session_id`、`tool_call_id`、`tool_name`、命中规则名列表、动作、内容摘要（hash 或截断前 200 字）。
- AC-3.2：命中时同时写入一条 `slog.Warn` 结构化日志，字段可用于报表。
- AC-3.3：未命中不产生日志噪音。

### US-4 · 作为 vv 应用层维护者，我希望把扫描器接到默认 Agent

**验收标准**
- AC-4.1：vv 侧 `registries/tool_access.go` 或 agent 构造处挂载默认扫描器（最小接入面），对 `coder`、`researcher`、`reviewer` 几类 Agent 默认启用。
- AC-4.2：vv 配置 `~/.vv/vv.yaml` 中可关闭或调整模式（`disabled` / `log` / `rewrite` / `block`）。
- AC-4.3：`chat` Agent 无工具，无需改造。

### US-5 · 合理的性能与大小边界

**验收标准**
- AC-5.1：正则扫描在 50KB 工具输出上 p99 < 2ms；整体链路开销 < 5ms（不含日志 I/O）。
- AC-5.2：超过配置上限（默认 256KB）的文本先截断再扫描，避免灾难式回溯；截断事件进入扫描日志。
- AC-5.3：非文本 `ContentPart`（如 `image` / `file` / `data`）不做文本扫描，原样透传。

## 3. 影响面（Impact Assessment）

### 3.1 角色（roles.md）

无新增角色；既有"Agent 框架接入方 / vv 应用维护者"职责中加一条"配置并观察工具返回注入扫描"。

### 3.2 字典（dictionaries.md）

新增术语：
- **Indirect Prompt Injection** — 指令通过外部数据（网页、文件、工具返回）到达模型的注入攻击
- **Tool-Result Guard** — 挂在工具返回到模型之间的扫描器
- **Quarantine** — 一种扫描命中后的"包裹"动作，把内容标记为 untrusted 再交给模型

### 3.3 模型（models/）

不新增领域模型。扩展：
- `vage/guard/` 新增 `ToolResultGuard`、`DirectionToolResult` 常量、`InjectionAction` 配置枚举、`ExtendedInjectionPatterns()` 规则集
- `vage/schema/Message` 无改动
- `vage/hook/` 新增一种事件类型 `EventGuardHit`（或复用现有 `EventGuardCheck`——在设计阶段裁决）

### 3.4 流程（procedures/）

修改 `vage/agent/taskagent/task.go` 两个工具循环（`Run` 分支 L710–L732、`RunStream` 分支 L960–L994），在 `toolMsg` 入队前插入扫描。

### 3.5 应用（applications/）

- `vage` 框架层新增中间件/Guard 点位。
- `vv` 应用层新增默认配置与 YAML 暴露。
- CLI/HTTP 两种模式都自动生效。

## 4. 业界实现参考（摘要）

详见工业调研报告（本轮 analyst 阶段附带产出），要点：

| 维度 | 业界做法 | 本次 v1 选型 |
|------|----------|-------------|
| 检测层 | 正则/启发式、Unicode 走私、LLM-Judge（Prompt-Guard-86M、Llama-Guard-3、Lakera Guard、Azure Prompt Shields）、Rebuff、LLM-Guard、NeMo Guardrails | **仅正则** + Unicode tag/Bidi codepoint 检测；LLM-Judge 留 v2 |
| 动作 | block / redact / quarantine / log / downgrade / human-in-the-loop | **block / rewrite / log** 三档；`rewrite` 默认为 quarantine 包裹 |
| 放置点 | input / tool-result / pre-model / output / tool-call | **tool-result 到 model 之间**（ReAct 循环内），在入 memory 前后只走一次 |
| 大小策略 | 截断 + 采样 | 默认 **256KB 硬截断**；超限事件打点 |
| 产业事件 | Bing Chat、Copilot Workspace README 劫持、Slack AI 频道注入、MCP 工具投毒（Invariant Labs） | 规则库覆盖 role-hijack、prompt-extract、exfil 命令组合、markdown-image-exfil 模板 |
| 参考 OSS | Rebuff / LLM-Guard / NeMo / Guardrails AI（Python 生态为主，Go 无等价项） | 自行在 `vage/guard/` 扩展，规则 YAML 化 |

**关键坑点**（直接影响 US-1 / US-5）：
1. 误报：安全类文本（CVE/CTF/规则库本身）会命中 "ignore previous instructions"。→ 规则打上 severity，低置信度默认 `log`，高置信度（ChatML、Unicode tag、exfil 组合）才 `block`。
2. 绕过：同义词、翻译、Markdown/HTML 注释、URL 片段、CSS 隐藏等。→ v1 承认只是减速带；文档里显式说明其局限性。
3. 放大重扫：memory 回放历史工具返回被重复扫。→ 扫描点固定在 `executeToolCall` 返回处，只扫一次；不做 memory 重扫。
4. 大文本回溯爆炸：grep/cat 大文件的返回可能是 MB 级。→ 硬截断 + 只对 `text` 型 ContentPart 扫描。
5. MCP 工具描述投毒（"rug pull"）不属于本需求范畴，留给 P0-4。

## 5. 明确的技术约束

- **最小侵入**：不新增单独模块，复用 `vage/guard/` 现有骨架（`Guard`、`Message`、`Result`、`RunGuards`）；通过新增一种 `Direction` 值或新类型让既有 Chain 能跑。
- **接入点收敛**：只在 TaskAgent 的两个工具循环处挂钩；其他 Agent 类型（Router/Workflow/Custom）本身不直接跑工具，不需要改。
- **零配置即可跑**：无 Option → 行为与现状一致；`WithToolResultGuards(…)` Option 启用。
- **规则库分级**：`DefaultInjectionPatterns()` 保留现状（供输入侧用）；新增 `DefaultToolResultInjectionPatterns()`，把业界推荐的 15+ 条规则加进来，带 severity。

## 6. 验证计划（交接 tester）

- 单元测试：覆盖三档动作、规则命中、规则分级、Unicode tag 命中、ChatML 标记命中、截断行为、非文本内容透传。
- 集成测试（`vage/integrations/guard_tests/`）：使用一个可控的假工具模拟恶意返回，跑一次完整 TaskAgent `Run` 流程，验证 model 看到的 tool message 已被 block/rewrite。
- 性能基线：单元级基准测试（`*_bench_test.go`）在 50KB 负载上 p99 < 2ms。

## 7. 待澄清（Open Questions）

没有阻塞性问题。以下为已在本文件内做出合理假设、交 designer 裁决的事项：

1. 命中事件是**新增 `EventGuardHit`** 还是**复用 `EventGuardCheck`**。→ designer 裁决，倾向**复用**（事件系统尽量稳定）。
2. `rewrite` 默认产物是"整段替换为 `[REDACTED:…]`" 还是"包裹 `<untrusted_content source=…>…</untrusted_content>`"。→ designer 裁决，倾向**包裹**（保留信息熵，让模型自己识别"这是数据不是指令"）。
3. vv 侧默认动作是 `log` 还是 `rewrite`。→ designer 裁决，倾向**首发 `log`**（先观察误报 1–2 个版本再升 `rewrite`）。
