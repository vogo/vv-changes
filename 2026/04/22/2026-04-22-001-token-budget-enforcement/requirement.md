# Requirement: Token Budget Enforcement / Consumption Limits (P1-3)

## 1. 背景与目标

### 背景

vv 已具备两类独立但互不联通的能力：

1. **TaskAgent Run 级预算**（`vage/agent/taskagent/budget.go`）：原子计数器 + 迭代结束点硬停，`Exhausted` 时发 `EventTokenBudgetExhausted` 事件并以 `StopReasonBudgetExhausted` 返回。仅作用于**单次 Run**，跨 Run/Session/Day 不累计。
2. **会话级成本追踪**（`vv/traces/costtraces/`）：在事件链路上累积 `input/output/cache_read` tokens 并查价估算美元成本。**纯观测**，无任何限额动作。

两者共存的结果是：

- 交互式 CLI 单会话内连续发起 N 个 Run → 总消耗可能远超 Run-level budget，但不会被拦截。
- HTTP 模式一条 Session 可能由多个请求、多个子 Agent 组成 → 任何单 Run 拦截逻辑都无法统管。
- 全局 / 每日上限缺失 → 单用户跑远程任务/Cron 时存在数美元→数千美元滑坡风险（与行业"Runaway Cost"案例一致）。

原 PRD `overview.md` 已将 "Token / cost budget enforcement beyond Run-level" 明确列为 **Does Not Cover**，`feature-todo.md` 将其列为 P1-3。

### 目标

在不打破现有 Run 级预算语义的前提下，把"预算"升级为**分层、可配置、可观测、软硬两档**的消耗护栏：

- **分层**：Run ⊂ Session ⊂ Global / Time-window。子层不足时会比父层更早触发；子层宽松时父层仍能兜底。
- **软硬两档**：`warn` 阈值触发日志/事件但不拦截；`hard` 阈值拦截下一次 LLM 调用并以结构化停止原因返回。
- **可观测**：在 CLI 状态栏、HTTP 响应、事件流中暴露 "used / budget / remaining"，便于用户提前看到压力。
- **最小侵入**：复用 `vage/largemodel/` 中间件链 + `vage/hook/` 事件；不重写 TaskAgent 主循环；CLI/HTTP 两种形态复用同一份 TokenBudget 聚合器。

## 2. 行业与类似产品调研（2026-04-22）

| 产品/框架 | 预算粒度 | 软/硬 | 触发时机 | 恢复/重置 | 可借鉴点 |
|---|---|---|---|---|---|
| **Anthropic Task Budget (Beta)** | 单次 agentic loop | **软**（advisory，服务端注入倒计时 marker） | 模型自行收敛 | 跨 compaction 用 `remaining` 透传 | 软预算机制设计；`total + remaining` 语义；模型"优雅收尾"。`max_tokens` 作为硬上限兜底 |
| **Claude Code（用户侧）** | 5 小时窗口 + 周窗口 + 月支出 | **硬** | 窗口用尽拒绝服务 | 时间窗口自动重置 | 双时间窗（短周期恢复 + 长周期上限）；"active compute" vs idle 区分 |
| **OpenAI Agents SDK** | Per-run Usage 聚合（含 handoffs） | 无内建硬限 | Hook `on_agent_end` 自行实现 | 不适用 | 嵌套/派生 Agent 的 usage 聚合到同一 context |
| **LangGraph** | `recursion_limit`（步数） + Callback 统计 tokens | **硬**（步数）+ 自定义 Callback | 步数耗尽抛 `GraphRecursionError` | 不适用 | 步数+tokens 双维度；Callback 钩子在 `on_llm_start` 前做 pre-check |
| **LiteLLM** | `max_iterations` + `max_budget_per_session` | **硬**（429 Too Many Requests） | 请求前检查 | session TTL（默认 1h）自动过期 | session TTL 回收；`require_trace_id` 强制会话隔离；错误 code `budget_exceeded` |
| **MLflow AI Gateway** | 按时间窗聚合美元 | **硬** | 原子 window init + atomic accumulate | 窗口切换自动重置 | 多副本下 Redis 共享；单机本地模式 |
| **agent-cost-guardrails / TokenFence** | Per-call + session + daily 三级 | **硬 + 软（80% 预警）** | `pre_call_check` + `post_call_record` | N 连续违规 trip circuit-breaker | 双阶段检查（pre/post）；三级阈值告警 |

### 关键教训

1. **软 vs 硬要清晰**：advisory 让模型优雅收敛是体验最佳，但必须有 `max_tokens` 或硬预算兜底以防渣模型忽略信号。
2. **pre-call + post-call 双点**：仅 post-call 记录会在单次大请求里超支；仅 pre-call 估算会在响应流里低估。vv 已有 `EventLLMCallEnd` 的 post 通道，需要补 pre-check 中间件。
3. **嵌套 Agent 必须聚合**：子 Agent、agent-as-tool、Dispatcher 派生的动态 Agent 都必须共享同一个 Session Tracker，否则 fanout 秒超预算。
4. **窗口重置语义**：会话 TTL、每日窗口、滑动窗口三选一即可；vv 首版取"每日窗口 + 会话自然生命周期"最简组合。
5. **预算过小会触发"拒绝症"**：Anthropic 实测显示预算过小，模型会直接放弃。→ vv 要 sanity-check 并在 CLI 日志里提示。
6. **缓存友好**：Anthropic 警告每次改 `remaining` 会打碎 prompt cache。→ vv 仅在必要点重算，不把 remaining 写进 system prompt。
7. **计数要原子**：并发工具调用（P1-7 正在做）会让 tracker 同时被多个 goroutine 累加，必须使用 `atomic` 或 mutex。

### 不采纳的方案

- **Redis/分布式共享**：vv 当前单进程单用户，过度设计。
- **按用户/租户分桶**：依赖 P4-1 认证体系，暂不做。
- **动态模型降级（超 80% 切便宜模型）**：依赖 `aimodel/composes/` 已有但未暴露到 vv 的 weighted 策略；放 P2-8 摘要召回链之后再做。
- **预测性估算（pre-call 用 tiktoken 估算 input+output 大小）**：首版不做，直接用真实 `usage` 记账；估算误差反而引入 bug。

## 3. 用户故事与验收标准

### US-1：单 Run 内无超支（已实现，保持不变）

> 作为 CLI/HTTP 用户，我配置了 `agents.run_token_budget = 50000` 后，任何一次 Agent.Run 在累计 tokens ≥ 50000 时应立刻停止，并返回 `StopReason = budget_exhausted`。

**验收**：保持现有行为；回归测试 `vage/agent/taskagent/budget_test.go` 全绿。

### US-2：会话级硬上限

> 作为 CLI 用户，配置 `budget.session_hard_tokens = 200000`，一个 CLI 会话内所有 Run 总 token 消耗到达 200000 时，**下一次** Agent.Run 在调用 LLM 前立即拒绝，返回错误 `ErrSessionBudgetExceeded`；CLI 显示提示："Session budget exhausted (200000/200000)；用 `/reset` 开新会话或 `/budget` 查看。"

**验收**：
- 硬上限到达后，无论什么子 Agent、无论 tool-call 层级多深，下一次 LLM 调用必拒。
- 拒绝发生在 LLM call 之前（通过中间件），而不是消耗完才发现。
- 剩余额度显示保持 0，不出负数。
- CLI `/budget` 命令展示当前 Session used/budget 与 Global daily used/budget。

### US-3：会话级软阈值预警

> 当会话消耗超过 `budget.session_warn_pct × session_hard_tokens`（默认 0.8）时，CLI 状态栏变黄并日志打印一行 warning，但继续执行。HTTP 响应头追加 `X-Budget-Warn: session`。

**验收**：
- 预警仅触发一次（不刷屏）。
- 不做 ABORT，不改变业务语义。
- 事件 `EventBudgetWarn`（新增）带 scope 与 used/limit 信息。

### US-4：全局/每日硬上限

> 配置 `budget.daily_hard_tokens = 2000000`（或 `budget.daily_hard_cost_usd = 10.0`），在 UTC 自然日窗口内累计消耗到达阈值时，所有会话、所有模式（CLI / HTTP / MCP）拒绝新请求，直至窗口切换。

**验收**：
- 窗口边界为 UTC 00:00，切换时自动归零。
- 重启进程后，当日已消耗从本地持久化恢复（简易：JSONL 追加；启动时回放当日记录）。
- HTTP 返回 `429` 带 `{ "error": {"type":"budget_exceeded","scope":"daily_tokens", ...} }`。
- CLI 显示 "Daily budget exhausted. Resets at 00:00 UTC (~X hours)."

### US-5：成本维度预算

> 除 tokens 之外，用户可以按 **USD** 设置预算：`budget.session_hard_cost_usd`, `budget.daily_hard_cost_usd`。未配置模型定价时退化到 tokens-only，CLI 显示 "cost budget disabled (no pricing for <model>)"。

**验收**：
- tokens 与 cost 两种限额同时配置时，任一触发即拦截（"先达为准"）。
- 未查到 model pricing 时 cost 预算被忽略，tokens 预算仍然有效。
- 精度不丢失（`float64`，不四舍五入到分）。

### US-6：嵌套 Agent / Agent-as-Tool 聚合

> Dispatcher 派生的动态子 Agent、agent-as-tool 调用的 Agent，以及 CustomAgent 内部嵌套 Agent.Run 调用，都共享同一个 Session Tracker。

**验收**：
- 父 Agent 请求 100 tokens、子 Agent-as-tool 请求 50 tokens，会话累计 = 150，不是 100 也不是 200。
- 并发子 Agent（P1-7 并行工具调用引入）累加使用原子操作，不丢计数。

### US-7：可观测性

> 用户在任何时候能查到：
> - 当前 Run 的 used/budget/remaining。
> - 当前 Session 的 used/budget/remaining（tokens + USD）。
> - 当日 Global 的 used/budget/remaining（tokens + USD）。

**验收**：
- CLI 新 slash 命令 `/budget`。
- HTTP 新只读端点 `GET /v1/budget`（响应 JSON 含三层状态）。
- Stream 事件 `BudgetStatus`（新增）在每次 LLM end 后发一次，供客户端实时显示。
- 事件 `EventBudgetExceeded`（新增）区别于 `EventTokenBudgetExhausted`（Run 级保留），携带 `scope` 与详细使用量。

### US-8：配置与 env override

> 所有预算字段可通过 `~/.vv/vv.yaml` 的 `budget:` 区块 + env var（`VV_BUDGET_SESSION_HARD_TOKENS` 等）覆盖；零值/未设置 = 不启用。

**验收**：
- env var 优先级高于 yaml。
- 不配置预算时行为与当前完全一致（零侵入）。
- 首次运行向导（`vv/setup/`）不做新字段提示，保持向导简洁；新用户默认禁用。

## 4. 范围

### In-Scope

- `vage/agent/taskagent/budget.go`：保留现有，不重命名。
- 新增 `vage/largemodel/budget_middleware.go`：pre-call 硬限检查 + post-call 记账。
- 新增 `vv/traces/budgets/`（或直接在 `vv/traces/costtraces/` 扩展）：Session 级 + Daily 级 tracker；注意 tracker 要共享给 `costtraces`（同一份 tokens 数据）。
- 新增 `vv/configs/budget.go` 的 `BudgetConfig` 结构 + yaml/env 解析。
- 新增 `vage/schema/event.go` 事件：`EventBudgetWarn`, `EventBudgetExceeded`, `EventBudgetStatus`。
- CLI `/budget` slash 命令。
- HTTP `GET /v1/budget` 只读端点（HTTP 模式）。
- Daily 持久化：轻量 JSONL 追加到 `~/.vv/budget/daily-YYYY-MM-DD.jsonl`（当日），启动时 replay。不走 SQLite（P1-6 未落地，不做依赖）。
- 测试：单元测试 + 集成测试（TaskAgent + Middleware + 配置）。

### Out-of-Scope（显式列出，避免蔓延）

- 跨用户/多租户/按租户分桶（依赖 P4-1 认证）。
- 预算到达后的智能降级（模型切换、摘要短路）—— 属 P2-8。
- 预算持久化到 SQLite —— 等 P1-6。
- Prompt cache 感知的"真实计费 tokens 抵扣"—— cache-read 已有从 API 响应直接读取的字段，按响应值记账即可，不另做推断。
- CLI 首次向导新增预算配置项（保持向导简洁）。
- 预算告警的邮件/webhook 推送（属 P3-11 通知系统）。
- 分布式共享计数（单进程即可）。

## 5. 受影响对象

### 角色（`doc/prd/architecture/roles.md`）

- **Operator**（运行 vv 的个人用户）：可配置预算、查看状态、收到预警。

### 模型（`doc/prd/models/core/`）

- 新增 `model-budget-config.md`（BudgetConfig：session_hard_tokens / session_warn_pct / session_hard_cost_usd / daily_hard_tokens / daily_hard_cost_usd）。
- 新增 `model-budget-tracker.md`（运行时三层聚合器，字段：scope / used_tokens / used_cost_usd / hard_tokens / hard_cost_usd / window_start）。
- 新增字典 `dictionary-budget-scope.md`（`run` / `session` / `daily`）。
- 更新 `model-session-cost-tracker.md`：增加 "与 Budget Tracker 的关系" 段落（Cost Tracker 产出 Token Usage，Budget Tracker 消费它）。
- 更新 `model-configuration.md`：新增 `budget:` 块。

### 流程（`doc/prd/procedures/`）

- 新增 `procedure-budget-enforcement.md`：LLM 调用前后的 pre/post 流程、预警/拒绝判定。

### 应用（`doc/prd/applications/cli/`, `api/`）

- CLI：新增 `/budget` 命令。
- HTTP：新增 `GET /v1/budget`。

## 6. 显式假设

1. **单用户单进程**：一个 vv 进程 = 一个配置来源 = 一个用户。Daily 计数本地持久化即可。
2. **OS 时钟可信**：Daily 窗口用本地 UTC，系统时钟被调整后行为以配置为准（极端场景 out-of-scope）。
3. **cache-read tokens 计入**：与 Anthropic 计费一致，cache-read 算 input（按 Anthropic 官方计费规则，缓存命中是按"读取"计费的较低费率，但仍计入 input_tokens 聚合）。pricing 表已能区分 cache 费率。
4. **不精确统计预估**：只在 LLM 返回 `usage` 后记账，pre-check 阶段使用"已消耗"与"限额"比较，不做未来预估。
5. **预警仅一次**：进入 warn 区间触发；退不出（已消耗不会减）；同一 session 内不再重复触发。

## 7. 成功度量

| 指标 | 目标 |
|---|---|
| 单元 + 集成测试覆盖预算模块 | ≥ 85% |
| 未配置预算时的行为差异 | 0（功能完全兼容） |
| 硬预算触发后，再下一次 LLM 调用是否被拦截 | 100% |
| Warn 阈值是否仅触发一次 | 100% |
| 并发 10 路子 Agent 各自消耗 1000 tokens，Session Tracker 总和 | 恰好 10000 ±0 |
| Daily 重置：跨 UTC 00:00 自动归零 | 100% |
| CLI `/budget` 显示 3 层状态 | 完整 |

## 8. 现有文档不一致点（留档）

调研阶段发现以下 PRD 内部轻微不一致，**不在本需求修复**，但文档阶段顺手对齐：

- `overview.md` 已列 "Token / cost budget enforcement beyond Run-level" 为 Does Not Cover；实现后需要从 Does Not Cover 移到 Features。
- `feature-implement.md` 中 "Token 用量 + 成本追踪（会话级）" 的 "会话级" 严格说只是会话内部的 Token Usage 累计（无阈值动作），文档阶段可以把新的 Budget Tracker 分开表述。
- `model-session-cost-tracker.md` 目前未提及与预算的关系；文档阶段补关系条目。
