# Design: Token Budget Enforcement / Consumption Limits (P1-3)

## 0. 总览

本需求在**不改 TaskAgent 主循环**的前提下，通过 **一个新的 LLM 中间件**（`BudgetMiddleware`）与 **一个新的 Tracker**（`budgets.Tracker`）把 Run/Session/Daily 三级预算串成一条线。所有拦截发生在 **LLM 调用之前**（pre-call 硬检查），所有记账发生在 **LLM 调用之后**（post-call 累加，复用 `EventLLMCallEnd`）。

```
CLI / HTTP
   │
   ▼
Dispatcher ── sessionTracker ──┐
   │         dailyTracker  ────┼── 通过 Option 注入
   ▼                           │
TaskAgent (Run budget 不变)    │
   │ uses                      │
   ▼                           ▼
LLM middleware chain:
  Log → BudgetMiddleware(pre-check + post-record) → CircuitBreaker → RateLimit → Retry → Timeout → Cache → Metrics → base
                         │
                         └─ 拒绝：return ErrBudgetExceeded (scope=session|daily)
                                emits EventBudgetExceeded
                                emits EventBudgetWarn (首次越过软阈值)
                                emits EventBudgetStatus (每次成功调用后)
```

关键设计原则：
- 零配置零成本：未设置预算 → Tracker 零值 → Middleware 直通，**无任何开销**。
- 预算 ≠ Cost Tracking：`budgets.Tracker` 可以复用 `costtraces.Tracker` 的数据源（同一个 `LLMCallEndData`），但职责分开——一个记账、一个拦截。
- 中间件位置靠外：放在 `CircuitBreaker` 之前，让"预算已超"的拒绝**先于**重试/断路统计。预算错误不触发断路器、不计入 retry。

## 1. 数据结构

### 1.1 `vv/configs/budget.go`（新建）

```go
// BudgetConfig 全部字段零值 = 禁用。
type BudgetConfig struct {
    // Session 层
    SessionHardTokens     int     `yaml:"session_hard_tokens"`       // 0=off
    SessionHardCostUSD    float64 `yaml:"session_hard_cost_usd"`     // 0=off
    SessionWarnPercent    float64 `yaml:"session_warn_percent"`      // default 0.8

    // Daily 层（UTC 自然日窗口）
    DailyHardTokens       int     `yaml:"daily_hard_tokens"`         // 0=off
    DailyHardCostUSD      float64 `yaml:"daily_hard_cost_usd"`       // 0=off
    DailyWarnPercent      float64 `yaml:"daily_warn_percent"`        // default 0.8

    // Daily 持久化（仅当 Daily 启用时使用）
    DailyPersistDir       string  `yaml:"daily_persist_dir"`         // default ~/.vv/budget
}
```

挂到 `Config` 顶层新增 `Budget BudgetConfig`（`vv/configs/config.go`）。env override：
```
VV_BUDGET_SESSION_HARD_TOKENS
VV_BUDGET_SESSION_HARD_COST_USD
VV_BUDGET_SESSION_WARN_PERCENT
VV_BUDGET_DAILY_HARD_TOKENS
VV_BUDGET_DAILY_HARD_COST_USD
VV_BUDGET_DAILY_WARN_PERCENT
VV_BUDGET_DAILY_PERSIST_DIR
```

*Run 层保持复用 `agents.run_token_budget`*，不引入新字段。

### 1.2 `vv/traces/budgets/tracker.go`（新建目录 `vv/traces/budgets/`）

```go
// Scope 明示本 Tracker 代表的层级。
type Scope string
const (
    ScopeSession Scope = "session"
    ScopeDaily   Scope = "daily"
)

// Tracker 单层级的聚合器。并发安全。
type Tracker struct {
    scope        Scope
    mu           sync.Mutex
    usedInTokens int64          // 累计 input+output+cache_read
    usedCostUSD  float64        // 累计估算美元；无定价时为 0

    hardTokens   int            // 0=off
    hardCostUSD  float64        // 0=off
    warnPercent  float64        // 0.8 default

    warnFired    bool           // 软阈值是否已触发过（一次性）
    windowStart  time.Time      // Daily 窗口起点；Session 为 Tracker 创建时间

    // Daily 专用
    persistPath  string         // 非空则启用持久化
}

func NewSession(cfg configs.BudgetConfig) *Tracker
func NewDaily(cfg configs.BudgetConfig, clock func() time.Time) (*Tracker, error)

// 主要 API：
func (t *Tracker) Add(in, out, cacheRead int, costUSD float64) // post-call 记账
func (t *Tracker) Check() error                                 // pre-call 硬检查；返回 *BudgetExceededError 或 nil
func (t *Tracker) WarnThresholdCrossed() bool                   // 仅当本次 Add 使得跨越 warn 阈值时返回 true（一次性）
func (t *Tracker) Snapshot() Snapshot                           // 只读快照
func (t *Tracker) ResetIfWindowElapsed(now time.Time) bool      // Daily 专用，跨 UTC 00:00 自动归零
```

`Snapshot` 用于 CLI/HTTP/事件展示：

```go
type Snapshot struct {
    Scope        Scope
    UsedTokens   int64
    UsedCostUSD  float64
    HardTokens   int
    HardCostUSD  float64
    WarnPercent  float64
    WindowStart  time.Time
    RemainingTokens int64     // -1 if unlimited
    RemainingCostUSD float64
}
```

### 1.3 `BudgetExceededError`

```go
type BudgetExceededError struct {
    Scope     Scope     // "session" or "daily"
    Dimension string    // "tokens" or "cost"
    Used      int64     // 或 cost 用 UsedCostUSD
    Limit     int64
    UsedCostUSD  float64
    LimitCostUSD float64
}
func (e *BudgetExceededError) Error() string
// sentinel: errors.Is 支持
```

## 2. 中间件

### 2.1 `vage/largemodel/budget_middleware.go`（新建）

```go
// BudgetMiddleware 执行 pre-call 硬检查，post-call 通过事件让外部 Tracker 记账。
// 自身不持有 Tracker —— 持 Checker 函数与一个 Status 通知函数。
type BudgetMiddleware struct {
    precheck func(ctx context.Context) error           // 返回非 nil 则阻断
    dispatch schema.DispatchFunc                       // 发 EventBudgetStatus/Warn/Exceeded
}

func NewBudgetMiddleware(precheck func(ctx context.Context) error, dispatch schema.DispatchFunc) *BudgetMiddleware

func (m *BudgetMiddleware) Wrap(next aimodel.ChatCompleter) aimodel.ChatCompleter {
    return &budgetWrap{next: next, m: m}
}

// ChatCompletion: if pre-check 返回错误 → 直接返回 BudgetExceededError；不调 next；发 EventBudgetExceeded。
// 其余情况：调 next；成功后 **不记账**（记账由 costtraces & budgets 订阅 EventLLMCallEnd 完成）。
```

**为什么中间件不直接持有 Tracker？**

因为 Session Tracker 与 Daily Tracker 都是 `vv/` 层的对象，`vage/largemodel/` 不能依赖 `vv/`。改用两个回调注入：

1. `precheck` —— 主调用方（`vv/setup` 里组装）闭包捕获 `[]*budgets.Tracker`，顺序 check。
2. `dispatch` —— 继续用现有 `schema.DispatchFunc`。

Tracker 的 `Add` 由独立 Hook 订阅 `EventLLMCallEnd` 完成（见 §3.2），这样单次 LLM 调用的 pre/post 两个点都有最少耦合的注入方式。

### 2.2 中间件在链中的位置

```
Log → Budget → CircuitBreaker → RateLimit → Retry → Timeout → Cache → Metrics → base
        ^
       新增
```

放在 `Log` 之后（让拒绝能被日志记录），放在 `CircuitBreaker` 之前（让预算错误不触发断路），放在 `Retry` 之前（让 retry 不会反复消耗预算）。

`vage/largemodel/middleware.go:DefaultChain` 增加一个 optional 参数位。vv 层组装链时显式插入。

## 3. 记账路径与事件

### 3.1 Session / Daily 两个新事件

`vage/schema/event.go` 新增：

```go
EventBudgetWarn     = "budget_warn"      // 首次越过软阈值
EventBudgetExceeded = "budget_exceeded"  // 硬阈值触发拒绝（≠ EventTokenBudgetExhausted, 后者保留给 Run 级）
EventBudgetStatus   = "budget_status"    // 每次 LLM 成功结束后发一次快照

type BudgetWarnData struct {
    Scope     string    // "session" | "daily"
    Dimension string    // "tokens" | "cost"
    Used      int64
    UsedCost  float64
    Limit     int64
    LimitCost float64
    Percent   float64
}
type BudgetExceededData struct { /* 同 Warn + RejectedAt time.Time */ }
type BudgetStatusData struct {
    Session *Snapshot `json:"session,omitempty"`
    Daily   *Snapshot `json:"daily,omitempty"`
}
```

命名与 `TokenBudgetExhaustedData` 区分（后者是 Run-level 语义，**保留不改**，避免打破现有集成测试）。

### 3.2 Budget 记账 Hook

新建 `vv/traces/budgets/hook.go`：

```go
type recorder struct {
    session *Tracker // 可为 nil
    daily   *Tracker // 可为 nil
    pricing *costtraces.Pricing
    dispatch schema.DispatchFunc
}

func (r *recorder) OnEvent(ctx context.Context, e schema.Event) error {
    if e.Type != schema.EventLLMCallEnd { return nil }
    data := e.Data.(schema.LLMCallEndData)
    cost := 0.0
    if r.pricing != nil { cost = r.pricing.Estimate(data.PromptTokens, data.CompletionTokens, data.CacheReadTokens) }
    total := int64(data.PromptTokens + data.CompletionTokens) // cache_read 已在 prompt 里，不重复累加
    if r.session != nil {
        r.session.Add(data.PromptTokens, data.CompletionTokens, data.CacheReadTokens, cost)
        if r.session.WarnThresholdCrossed() { r.dispatch(ctx, newWarnEvent(r.session)) }
    }
    if r.daily != nil {
        r.daily.ResetIfWindowElapsed(time.Now().UTC()) // 每次记账前检查
        r.daily.Add(...)
        if r.daily.WarnThresholdCrossed() { r.dispatch(ctx, newWarnEvent(r.daily)) }
    }
    r.dispatch(ctx, newStatusEvent(r.session, r.daily))
    return nil
}
func (r *recorder) Filter() []string { return []string{schema.EventLLMCallEnd} }
```

由 `hook.Manager.Register` 注册为 sync hook。使用 sync 而非 async：记账需在下一次 pre-check 前确定生效；async 会引入 "先调 next-call 再处理 Add" 的竞态。

### 3.3 cache-read 计入规则

Anthropic API 将 `cache_read_input_tokens` 作为 `input_tokens` 的一部分返回（总 input 已包含它），OpenAI 在 `prompt_tokens_details.cached_tokens` 独立返回。**在 `Tracker.Add` 中只按 `input + output` 累加 tokens，不重复累加 `cache_read`。** 成本侧仍用 pricing 三档（input / output / cache）单独计价——Pricing.Estimate 已经正确分档，无需改动。

## 4. Pre-check 组装

`vv/setup/setup.go` 在中间件链构建点增加：

```go
// 生成一个 precheck 闭包，按 session → daily 顺序硬检查
func newBudgetPrecheck(session, daily *budgets.Tracker) func(context.Context) error {
    return func(ctx context.Context) error {
        if session != nil {
            if err := session.Check(); err != nil { return err }
        }
        if daily != nil {
            daily.ResetIfWindowElapsed(time.Now().UTC())
            if err := daily.Check(); err != nil { return err }
        }
        return nil
    }
}

// 插入链
llm = largemodel.NewBudgetMiddleware(precheck, hookMgr.Dispatch).Wrap(llm)
```

当 `session == nil && daily == nil` 时，跳过插入 Middleware（零开销路径）。

## 5. CLI `/budget` 命令

`vv/cli/budget.go`（新建）：

```go
func (m *model) handleBudgetCommand(args []string) tea.Cmd {
    session := m.sessionTracker.Snapshot() // 从 app state
    daily   := m.dailyTracker.Snapshot()
    // 渲染：
    //   Session: 15,320 / 200,000 tokens (7.7%)   $0.12 / $5.00
    //   Daily:   1,203,000 / 2,000,000 tokens (60%, window resets in 8h)
    //   Run:     budget per run = 50,000 (active)
    return m.appendMessage(render)
}
```

在 `vv/cli/memory.go:handleCommand` 分支中追加：

```go
if parts[0] == "/budget" { return m.handleBudgetCommand(parts[1:]) }
```

`/help` 文案同步加一行。

## 6. HTTP `GET /v1/budget`

`vv/httpapis/budget.go`（新建）：

```go
func handleGetBudget(session, daily *budgets.Tracker) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        resp := map[string]any{}
        if session != nil { resp["session"] = session.Snapshot() }
        if daily != nil   { resp["daily"]   = daily.Snapshot() }
        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(resp)
    }
}
```

挂到 `http.go`:
```go
if initResult.SessionBudget != nil || initResult.DailyBudget != nil {
    mux.HandleFunc("GET /v1/budget", handleGetBudget(initResult.SessionBudget, initResult.DailyBudget))
}
```

Cost enrich middleware增加：若 handler 内产生 `BudgetExceededError`，转换为 `429` + 结构化 body：
```json
{"error":{"type":"budget_exceeded","scope":"daily","dimension":"tokens","used":2000000,"limit":2000000}}
```

## 7. Daily 持久化

简易 JSONL，文件名 `~/.vv/budget/daily-YYYY-MM-DD.jsonl`。

**写入**：每次 `Add` 后 append 一行：
```json
{"ts":"2026-04-22T10:15:00Z","tokens":{"in":120,"out":30,"cr":0},"cost_usd":0.003,"model":"claude-sonnet-4"}
```

**启动时回放**：`budgets.NewDaily()` 加载当日文件（如存在）、累加到 `usedInTokens` / `usedCostUSD`；过期文件不清理（由用户 cron / `find -mtime` 自理）。

**并发**：单进程模式下单文件追加 + `sync.Mutex` 即可；不做文件锁。多进程下可能重复 / 丢失——本轮不解决（P4-5 分布式再处理）。

**错误容忍**：写磁盘失败只打 `slog.Warn`，不阻塞 LLM；读失败视为空文件（安全降级）。

## 8. Dispatcher / 子 Agent 传递

`vv/dispatches/dispatch.go`：
- `Dispatcher` 增加 `sessionBudget *budgets.Tracker`, `dailyBudget *budgets.Tracker` 字段。
- 新增 `WithSessionBudget(t)`, `WithDailyBudget(t)` options。
- 目前 Dispatcher 在 `dag.go:218` 给子 agent 传 `WithRunTokenBudget(d.runTokenBudget)`；**不需要** 给子 agent 传 budget tracker——因为预算拦截统一在 **LLM middleware chain** 上，所有 agent 共享同一份 `ChatCompleter`（middleware 已包装），嵌套子 agent 自然共享。
- 故 Dispatcher 侧改动仅限于：保留引用以便渲染 / 快照（`GET /v1/budget` 从这里读）。

Agent-as-tool / CustomAgent 嵌套：一样基于共享 `ChatCompleter`，无需额外处理。

## 9. Run-level 预算保留

`vage/agent/taskagent/budget.go` 完全不改。TaskAgent 的 `StopReasonBudgetExhausted` 与 `EventTokenBudgetExhausted` 保留原语义，继续服务"单 Run 过长"场景。与 Session/Daily 拦截**不冲突**：Run 级耗尽 → StopReason；Session/Daily 耗尽 → `BudgetExceededError`（下一次调用即拒）。

## 10. 测试计划

### 单元测试（全部在 `_test.go` 里）

| 测试文件 | 场景 |
|---|---|
| `vv/traces/budgets/tracker_test.go` | 零值配置（未启用）、Add 累加、Check 阈值、WarnThresholdCrossed 一次性、Snapshot 只读 |
| `vv/traces/budgets/tracker_test.go` | Daily 跨 UTC 00:00 自动归零（注入 `clock`） |
| `vv/traces/budgets/tracker_test.go` | 并发 1000 goroutine 各 Add(1,1,0) 累计一致 |
| `vv/traces/budgets/persist_test.go` | 空文件启动、单日累计、损坏行跳过、写失败降级 |
| `vage/largemodel/budget_middleware_test.go` | pre-check 返回错误 → 不调 next + 发事件；pre-check nil → 透传 |
| `vv/configs/config_test.go` | yaml/env 解析；零值默认；warn_percent=0 取 0.8 |
| `vv/cli/budget_test.go` | `/budget` 命令在三种状态（关闭/启用未触发/已触发）下的渲染字符串 |

### 集成测试（`vv/integrations/budget_tests/`）

| 用例 | 描述 |
|---|---|
| `session_hard_tokens` | 会话预算 200 tokens；模拟 LLM 调用耗掉 180 → 下一次 LLM 调用前触发 BudgetExceededError；CLI 拿到对应停止原因 |
| `session_warn_percent` | 会话预算 1000 tokens，warn=0.5；消耗到 510 后首次发 EventBudgetWarn；再消耗 100 不再发 |
| `daily_reset` | 注入 clock；消耗到 90% → 跨午夜 → 自动归零 → 重新计数 |
| `nested_agent` | Dispatcher 派生子 Agent，子 Agent 消耗 tokens；Session Tracker 应包含子 Agent 消耗 |
| `concurrent_tools` | 两个并发子 Agent 同时消耗；最终总和正确 |
| `cost_and_tokens_both` | 同时设置 tokens 和 cost 限额；任一先达触发 |
| `no_pricing_model` | 配置 cost 限额但模型无 pricing → cost 限额静默忽略，tokens 限额仍生效 |
| `disabled_all` | 不配置任何字段 → 无中间件插入 → 行为与 baseline 完全一致 |

### 性能基线

在 `_test.go` 里加一个简单 benchmark：Tracker.Add 的并发累加 + Check 单次调用延迟。目标：单次 Add+Check 总时延 < 1µs（纯内存原子/锁），确保对 LLM 调用无感。

## 11. 迁移与兼容

- 未触过 `~/.vv/vv.yaml` 的用户、或 yaml 不含 `budget:` 块的用户：零影响。
- 环境变量未设置：零影响。
- 已经在使用 `agents.run_token_budget` 的用户：保留生效，不冲突。
- `overview.md` 中 "Token / cost budget enforcement beyond Run-level" 从 Does Not Cover 移到 Features（文档阶段处理）。

## 12. 风险与缓解

| 风险 | 概率 | 缓解 |
|---|---|---|
| 中间件插入位置错，Retry 反复消耗预算 | 中 | 定位在 Retry 之前；集成测试专门验证"触发 Retry 的失败调用不会让 Session 多一次记账" |
| Daily 持久化坏文件阻塞启动 | 低 | 读失败视空；行 JSON 解析失败跳过；只在启动期写 `slog.Warn` |
| Stream 调用的 usage 延后到结束才可见 | 中 | 现有 `EventLLMCallEnd` 已在流结束时带 usage；BudgetMiddleware pre-check 只看累计值，不依赖当前 stream 体量 |
| 同时超 tokens 与 cost 两个阈值、错误类型冲突 | 低 | Check 返回 **第一个** 越限错误；附 `Dimension` 字段 |
| 测试中 daily 窗口切换难模拟 | 中 | Tracker 接收注入的 `clock func() time.Time`；测试传 fake clock |

## 13. 与 P1-7（并行工具调用）的交互

P1-7 在推进中，会让 TaskAgent 并发派发多个 tool_calls，下游可能并发发起 LLM（如 agent-as-tool 内部再调模型）。本设计通过：
- Tracker 的并发安全累加（`sync.Mutex` 保护 used*）；
- Pre-check 在每次 LLM 调用前独立执行；
- 并发集成测试覆盖。

两者落地顺序无依赖，但 merge 时需要合并 `taskagent/task.go` 与中间件链的小冲突。

## 14. 工作量预估

- `vv/configs/budget.go` + config 解析：~80 LOC
- `vv/traces/budgets/tracker.go` + test：~350 LOC
- `vv/traces/budgets/persist.go` + test：~150 LOC
- `vv/traces/budgets/hook.go`：~60 LOC
- `vage/largemodel/budget_middleware.go` + test：~120 LOC
- `vage/schema/event.go` 新 3 个事件 + 数据结构：~50 LOC
- `vv/cli/budget.go` + 接线：~90 LOC
- `vv/httpapis/budget.go` + 接线：~70 LOC
- `vv/setup/setup.go` 组装：~50 LOC
- 集成测试：~350 LOC

共计约 **1370 LOC**，与 feature-todo 粗估的 "低" 难度 200-500 LOC 有偏差，但主要来自 Daily 持久化 + 集成测试两块。主逻辑本身仍在 300-400 LOC 内。
