# Design: Token Budget Enforcement / Consumption Limits (P1-3)

> Revision of `design-raw.md` following improver review. See `design-review.md`
> for the full critique and decision trail. This revision is deliberately
> narrower: Daily persistence, per-layer warn thresholds, three event types,
> recorder hook + hook.Manager wiring, and the `Scope` named type are **cut**.

## 0. 总览

一个 LLM 中间件 + 一个轻量 Tracker 把 Session 与 Daily 两层预算挂到现有 LLM 调用链上。

```
CLI / HTTP
   │
   ▼
Dispatcher (不变)
   │
   ▼
TaskAgent (Run budget 不变)
   │ uses
   ▼
wrappedLLM = BudgetMiddleware(
                pre-check: session.Check() + daily.Check()
                post-record: session.Add() + daily.Add()
             ).Wrap(existingLLM)
   │
   └─ 超限 → 返回 *BudgetExceededError（errors.Is(err, budgets.ErrBudgetExceeded) == true）
            发 EventBudgetExceeded
            首次越过软阈值 → 发 EventBudgetWarn
```

关键原则：

- **零配置零成本**：未设置预算 → `setup.Init` 不构造 Tracker、不插入 middleware。完全走旧路径。
- **单点耦合**：Middleware 直接持有 `*budgets.Tracker`；**不**走 `hook.Manager`，也**不**新增 `costtraces.Pricing` 依赖进 `budgets/`。记账只在 LLM 调用点发生一次，和 `costtraces` 是兄弟关系、不是叠加关系。
- **Daily = in-memory only（P1）**：重启当日清零。持久化留给 P1-6 SQLite 落地时再做。
- **中间件位置**：放在任何已有重试/断路/超时之前——预算错误必须先于 Retry，不能被多次计费。当前 vv 的 `setup.Init` 尚未装这些 middleware，只有可选的 Debug wrapper；把 Budget 装在 Debug 之后即可。

## 1. 数据结构

### 1.1 `vv/configs/config.go`（扩展 `Config`）

```go
type BudgetConfig struct {
    SessionHardTokens  int     `yaml:"session_hard_tokens"`   // 0=off
    SessionHardCostUSD float64 `yaml:"session_hard_cost_usd"` // 0=off
    DailyHardTokens    int     `yaml:"daily_hard_tokens"`     // 0=off
    DailyHardCostUSD   float64 `yaml:"daily_hard_cost_usd"`   // 0=off
    WarnPercent        float64 `yaml:"warn_percent"`          // default 0.8; shared between session+daily
}

// Config 增加：
type Config struct {
    …
    Budget BudgetConfig `yaml:"budget,omitempty"`
}
```

Env overrides（`configs.Load`）：

```
VV_BUDGET_SESSION_HARD_TOKENS
VV_BUDGET_SESSION_HARD_COST_USD
VV_BUDGET_DAILY_HARD_TOKENS
VV_BUDGET_DAILY_HARD_COST_USD
VV_BUDGET_WARN_PERCENT
```

`applyDefaults`：`Budget.WarnPercent == 0 → 0.8`。

Run 层保留 `agents.run_token_budget`，不动。

### 1.2 `vv/traces/budgets/tracker.go`（新建）

```go
package budgets

// ErrBudgetExceeded is the sentinel for errors.Is.
var ErrBudgetExceeded = errors.New("budget exceeded")

// BudgetExceededError is the concrete error returned by a BudgetMiddleware
// pre-check and by Tracker.Add when a limit is first crossed.
type BudgetExceededError struct {
    Scope        string  // "session" | "daily"
    Dimension    string  // "tokens" | "cost"
    Used         int64
    Limit        int64
    UsedCostUSD  float64
    LimitCostUSD float64
}

func (e *BudgetExceededError) Error() string { /* human-readable */ }
func (e *BudgetExceededError) Is(target error) bool { return target == ErrBudgetExceeded }
func (e *BudgetExceededError) Unwrap() error { return ErrBudgetExceeded }

// IsExceeded is a convenience: errors.Is(err, ErrBudgetExceeded).
func IsExceeded(err error) bool { return errors.Is(err, ErrBudgetExceeded) }

// Tracker aggregates used tokens + cost against hard limits.
// Concurrency-safe via a single mutex. Pricing-agnostic: caller passes pre-computed cost.
type Tracker struct {
    scope       string     // "session" | "daily"
    mu          sync.Mutex
    usedTokens  int64
    usedCostUSD float64
    hardTokens  int64      // 0 = off
    hardCostUSD float64    // 0 = off
    warnPct     float64    // 0.0–1.0
    warnFired   bool       // one-shot per Tracker
    windowStart time.Time  // session: tracker creation; daily: UTC 00:00 of creation
    clock       func() time.Time // injected for tests; default time.Now
    daily       bool       // if true, Add/Check roll the window forward at UTC 00:00
}

func NewSession(cfg configs.BudgetConfig) *Tracker  // returns nil if no session limits set
func NewDaily(cfg configs.BudgetConfig) *Tracker    // returns nil if no daily limits set
func NewDailyWithClock(cfg configs.BudgetConfig, clock func() time.Time) *Tracker // tests

// AddResult reports the outcome of accounting this call.
type AddResult struct {
    WarnCrossed bool                  // true iff this call first crossed warn threshold
    Exceeded    *BudgetExceededError  // non-nil iff hard limit now exceeded
}

// Add records one LLM call's usage. Tokens = input + output (cache-read is
// already inside input for Anthropic; caller must not double-count).
// Safe for concurrent use. Rolls daily window internally under the lock.
func (t *Tracker) Add(tokens int64, costUSD float64) AddResult

// Check is the pre-call gate. Returns *BudgetExceededError if usage already
// equals or exceeds any hard limit. Rolls daily window internally.
func (t *Tracker) Check() *BudgetExceededError

// Snapshot returns a read-only view for rendering.
type Snapshot struct {
    Scope            string
    UsedTokens       int64
    UsedCostUSD      float64
    HardTokens       int64
    HardCostUSD      float64
    WarnPercent      float64
    WindowStart      time.Time
    RemainingTokens  int64   // -1 if unlimited
    RemainingCostUSD float64 // -1 if unlimited
}
func (t *Tracker) Snapshot() Snapshot
```

Design notes:

- `Add` and `Check` each hold the mutex once. The daily window roll happens
  under the same lock — no race window between roll and account.
- `warnFired` is reset whenever the daily window rolls.
- When `hardTokens == 0 && hardCostUSD == 0`, `NewSession`/`NewDaily` return
  `nil`; the middleware treats nil as "layer disabled".

### 1.3 Events (`vage/schema/event.go`)

Two new constants + two data structs. `EventBudgetStatus` is **cut** — snapshots are pulled via `Tracker.Snapshot()` / `GET /v1/budget`, not pushed every call.

```go
EventBudgetWarn     = "budget_warn"      // first crossing of warn threshold
EventBudgetExceeded = "budget_exceeded"  // hard-limit rejection (distinct from Run-level EventTokenBudgetExhausted)

type BudgetWarnData struct {
    Scope     string  `json:"scope"`       // "session" | "daily"
    Dimension string  `json:"dimension"`   // "tokens" | "cost"
    Used      int64   `json:"used"`
    UsedCost  float64 `json:"used_cost,omitempty"`
    Limit     int64   `json:"limit"`
    LimitCost float64 `json:"limit_cost,omitempty"`
    Percent   float64 `json:"percent"`     // used / limit at the crossing point
}
func (BudgetWarnData) eventData() {}

type BudgetExceededData struct {
    Scope     string  `json:"scope"`
    Dimension string  `json:"dimension"`
    Used      int64   `json:"used"`
    UsedCost  float64 `json:"used_cost,omitempty"`
    Limit     int64   `json:"limit"`
    LimitCost float64 `json:"limit_cost,omitempty"`
}
func (BudgetExceededData) eventData() {}
```

Run-level `EventTokenBudgetExhausted` and `TokenBudgetExhaustedData` remain untouched.

## 2. Middleware

### 2.1 `vage/largemodel/budget_middleware.go`（新建）

```go
package largemodel

// BudgetEnforcer is the minimal Tracker-shaped interface the middleware needs.
// Defining it here (instead of importing vv/traces/budgets) keeps vage free of
// a reverse dependency on vv.
type BudgetEnforcer interface {
    Check() error                          // pre-call gate; nil = ok
    Add(tokens int64, costUSD float64) any // returns caller-opaque result; see vv budgets.AddResult
    Snapshot() any                         // opaque snapshot for ergonomics
}

// BudgetMiddleware enforces hard budget limits before each LLM call and
// records usage after each successful call. Takes a caller-supplied
// PreCheckFunc + PostRecordFunc so vage/largemodel stays decoupled from
// the vv/budgets package that owns the concrete Tracker.
type BudgetMiddleware struct {
    preCheck   func(ctx context.Context) error
    postRecord func(ctx context.Context, usage aimodel.Usage)
    dispatch   DispatchFunc // may be nil (events are best-effort; middleware still enforces)
}

func NewBudgetMiddleware(
    preCheck func(ctx context.Context) error,
    postRecord func(ctx context.Context, usage aimodel.Usage),
    dispatch DispatchFunc,
) *BudgetMiddleware

func (m *BudgetMiddleware) Wrap(next aimodel.ChatCompleter) aimodel.ChatCompleter {
    // ChatCompletion:
    //   if err := m.preCheck(ctx); err != nil {
    //       if m.dispatch != nil { m.dispatch(ctx, NewEvent(EventBudgetExceeded, …, exceededDataFrom(err))) }
    //       return nil, err
    //   }
    //   resp, err := next.ChatCompletion(ctx, req)
    //   if err != nil { return nil, err }
    //   m.postRecord(ctx, resp.Usage)    // caller fires Warn/Exceeded events
    //   return resp, nil
    //
    // ChatCompletionStream:
    //   same pre-check; wrap stream with aimodel.WrapStream so usage arrives on Close.
}
```

`preCheck` + `postRecord` are closures built in `vv/setup/setup.go`. The middleware itself has zero dependency on `vv/`. `DispatchFunc` is the same type already exported by `largemodel/metrics.go`.

**Why not hook.Manager?** (1) vv does not currently install one on the LLM path; (2) Daily/Session accounting must finish before the *next* pre-check — inline synchronous update is the simplest way to guarantee that; (3) this exactly mirrors how `MetricsMiddleware` already handles `DispatchFunc` as an optional observability sink.

### 2.2 Position in the middleware chain

Today `setup.Init` only optionally wraps the LLM with `DebugMiddleware`. With this feature, the chain becomes:

```
Budget → Debug → base   (when both are enabled)
Budget → base           (debug off)
base                    (no budgets configured)
```

Budget is outermost so its rejection happens before any logging noise and before any future Retry/CircuitBreaker/Cache lands. If later a `MetricsMiddleware` or retry layer is added, Budget stays outermost.

## 3. 记账与事件路径

The recorder Hook from `design-raw.md` §3.2 is **removed**. Replaced by a closure in `setup.Init`:

```go
// vv/setup/setup.go (sketch)
var session *budgets.Tracker // may be nil
var daily   *budgets.Tracker // may be nil
if cfg.Budget != (configs.BudgetConfig{}) {
    session = budgets.NewSession(cfg.Budget)
    daily   = budgets.NewDaily(cfg.Budget)
}

if session != nil || daily != nil {
    pricing := costtraces.LookupPricing(cfg.LLM.Model, configs.ConvertPricing(cfg.ModelPricing))

    preCheck := func(ctx context.Context) error {
        if session != nil { if e := session.Check(); e != nil { return e } }
        if daily   != nil { if e := daily.Check();   e != nil { return e } }
        return nil
    }

    postRecord := func(ctx context.Context, u aimodel.Usage) {
        tokens := int64(u.PromptTokens + u.CompletionTokens) // cache-read already in PromptTokens (Anthropic)
        var costUSD float64
        if pricing != nil {
            nonCached := u.PromptTokens - u.CacheReadTokens
            costUSD = float64(nonCached)/1e6*pricing.InputPerMTokens +
                     float64(u.CompletionTokens)/1e6*pricing.OutputPerMTokens +
                     float64(u.CacheReadTokens)/1e6*pricing.CachePerMTokens
        }
        if session != nil { dispatchBudgetEvents(ctx, session.Add(tokens, costUSD), "session") }
        if daily   != nil { dispatchBudgetEvents(ctx, daily.Add(tokens, costUSD),   "daily")   }
    }

    wrappedLLM = largemodel.NewBudgetMiddleware(preCheck, postRecord, dispatchFn).Wrap(wrappedLLM)
}
```

`dispatchBudgetEvents` is a local helper that maps `budgets.AddResult` to `EventBudgetWarn` / `EventBudgetExceeded`. When no subscriber is wired (current vv), `dispatchFn` is a `slog.Warn` adapter — nothing is lost; tests can assert via Tracker state directly.

`costtraces.Tracker` continues to be updated from `EventLLMCallEnd` on the stream path — unchanged.

### 3.1 Cache-read tokens

Matches `costtraces.Tracker.Add` semantics: `PromptTokens` already includes `CacheReadTokens`, so the tokens counter uses `PromptTokens + CompletionTokens` (no double-add); cost uses three separate rates. Not new logic.

## 4. Run-level budget unchanged

`vage/agent/taskagent/budget.go` untouched. `StopReasonBudgetExhausted` + `EventTokenBudgetExhausted` retain Run semantics. A Session/Daily hard hit returns `*BudgetExceededError` from `ChatCompletion`/`Stream`, which bubbles up TaskAgent's normal error path — no new branching needed inside TaskAgent.

## 5. CLI `/budget` command

`vv/cli/budget.go`（新建）：renders three layers (Run/Session/Daily):

```
Run:     budget per run = 50,000 tokens  (active)
Session: 15,320 / 200,000 tokens (7.7%)   $0.12 / $5.00
Daily:   1,203,000 / 2,000,000 tokens (60%, window resets in ~8h)
```

If a layer is not configured, print `"not configured"` for that row.

Wiring (`vv/cli/memory.go:handleCommand`):

```go
if parts[0] == "/budget" { return m.handleBudgetCommand(parts[1:]) }
```

Plumbing: `App` gets two new fields `sessionBudget, dailyBudget *budgets.Tracker` (may be nil), set via a new `WithBudgetTrackers(session, daily)` option on `cli.New`. Mirrors the existing `costTracker` plumbing exactly — no changes to the `model` struct shape.

`/help` text gets one new line.

## 6. HTTP `GET /v1/budget`

`vv/httpapis/budget.go`（新建）：

```go
func handleGetBudget(session, daily *budgets.Tracker) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        resp := map[string]any{}
        if session != nil { resp["session"] = session.Snapshot() }
        if daily   != nil { resp["daily"]   = daily.Snapshot() }
        writeJSON(w, http.StatusOK, resp)
    }
}
```

Route registered in `Serve()` (`http.go`) only when either tracker is non-nil. Passed from `setup.InitResult.SessionBudget` / `DailyBudget`.

HTTP 429 mapping: `costEnrichMiddleware` (or a new narrow `budgetErrorMiddleware`) inspects written responses — out-of-scope for P1-3 **unless** the existing service layer naturally surfaces the error. Starting point: rely on the agent service path returning the `*BudgetExceededError` in the response body, and document the error shape; add a small middleware to rewrite status to 429 when `budgets.IsExceeded(...)` matches. ~25 LOC.

## 7. Daily 持久化 — 显式不做（P1-3 out of scope）

Daily tracker 是**纯内存**：进程重启归零，UTC 00:00 自动滚动窗口。

理由（见 design-review §1）：
- P1-3 assumption 已声明单进程单用户。
- JSONL 持久化的工程成本（corruption handling, replay, disk syscall per call）不匹配 P1 的价值回报。
- P1-6 SQLite 上线时可以自然接管当日状态。需求已经把跨天场景 out-of-scope。

需求 US-4 中"重启后当日已消耗从本地持久化恢复"一条**在 design-review 与本次改稿中被明确降级为 P1-6 随 SQLite 一起交付**。本 P1-3 不承诺重启持久化。feature-todo 应同步调整——见本文 §11。

## 8. Dispatcher 不需要新 option

与 raw 不同：**不**在 `Dispatcher` 上增加 `WithSessionBudget` / `WithDailyBudget`。Tracker 引用流向是：

```
setup.Init → InitResult.SessionBudget / DailyBudget
              │
              ├→ cli.New(..., WithBudgetTrackers(s, d))   // CLI 模式
              └→ httpapis.Serve(..., s, d)                // HTTP 模式
```

Middleware 已经绑在 `wrappedLLM` 上；Dispatcher 及所有派生子 Agent、agent-as-tool、CustomAgent 共享这同一份 `ChatCompleter`，预算拦截**自动**随共享 LLM 下潜。这也是 raw 设计 §8 自己的结论——只是 raw 仍保留了两个没必要的 options。

## 9. InitResult changes

```go
type InitResult struct {
    …existing fields…
    SessionBudget *budgets.Tracker // nil if session budget disabled
    DailyBudget   *budgets.Tracker // nil if daily budget disabled
}
```

`setup.Init` returns these so both CLI and HTTP paths can read/render without re-constructing.

## 10. Tests

### 10.1 Unit tests

| File | Scenarios |
|---|---|
| `vv/traces/budgets/tracker_test.go` | Disabled (zero config) → NewSession/NewDaily return nil. Add accumulates. Check pre-gates at/after limit. Warn fires once. Concurrency: 1000 goroutines × Add(1,0), total exact. |
| `vv/traces/budgets/tracker_test.go` | Daily window rolls at UTC 00:00 under injected clock; warnFired resets on roll. Add-during-roll is accounted in the new window. |
| `vv/traces/budgets/error_test.go` | `errors.Is(err, ErrBudgetExceeded)` passes; `Error()` is human-readable. |
| `vage/largemodel/budget_middleware_test.go` | preCheck err → no call to next + event emitted; preCheck nil → transparent; stream path emits on close. |
| `vv/configs/config_test.go` | yaml + env parse; zero value = disabled; default WarnPercent=0.8. |
| `vv/cli/budget_test.go` | `/budget` render in 3 states: none configured, active under limit, exhausted. |

### 10.2 Integration tests (`vv/integrations/budget_tests/`)

| Case | Description |
|---|---|
| `session_hard_tokens` | Session 200 token limit; after consuming 180, next LLM call rejected before reaching network (assert via mock completer call count). |
| `session_warn_percent` | Limit 1000, warn 0.5; at 510 the first EventBudgetWarn is dispatched; no second event up to 999. |
| `daily_window_roll` | Injected clock; usage at 90% → advance past 00:00 → counter = 0 and new usage counted cleanly. |
| `nested_agent` | Dispatcher → sub-agent → agent-as-tool; session tracker captures all three layers through the shared wrappedLLM. |
| `concurrent_tools` | Two goroutines driving the same wrappedLLM; final session total exact, no race. |
| `cost_and_tokens_both` | Both limits set; whichever is hit first wins, error Dimension reflects it. |
| `no_pricing_model` | Cost limit configured but pricing missing → cost limit silently skipped; tokens limit still enforced. |
| `disabled_all` | No budget config → no middleware wrapping → identical to baseline behavior (byte-for-byte trace comparison). |

Daily persistence cases (`daily_reset_across_restart`, `daily_persist_corrupt_file`) are **cut** with Daily persistence itself.

### 10.3 Benchmark

`BenchmarkTrackerAddCheck` — target < 500ns per `Add`+`Check` at 8 goroutines. Feature must not add measurable LLM-call-path latency.

## 11. 迁移与文档

- `~/.vv/vv.yaml` 不含 `budget:` 块 → 零影响。
- 环境变量未设置 → 零影响。
- `agents.run_token_budget` 老用户 → 行为不变。
- 需求文档 US-4 验收项里"重启后当日已消耗从本地持久化恢复"改为"留待 P1-6 SQLite"注脚（`requirement.md` 不在本次修改，documenter 阶段同步）。
- `overview.md` 的 "Does Not Cover" 条目移入 Features。

## 12. 风险

| 风险 | 概率 | 缓解 |
|---|---|---|
| 中间件位置错导致 Retry 反复记账 | 低 | 中间件必须最外层；测试专门验证（mock Retry middleware attempt=3，Session 仅记账 1 次）。 |
| Daily 重置在并发调用下抖动 | 低 | Add/Check 持同一 mutex；roll 在 lock 内完成。 |
| Stream usage 延后到 Close 才可见，期间大量 tokens 已发出 | 中 | 这是固有行为；Check 只比较累计值，不预测流内大小。与 `costtraces` 一致。 |
| tokens + cost 两个维度同时命中 | 低 | Check 返回第一个越限错误（语义："先达为准"）；`BudgetExceededError.Dimension` 区分。 |
| 重启丢失 Daily 累计 → 用户跑大任务的"午夜重启绕过" | 中 | 这是有意牺牲；文档明确；P1-6 修复。 |

## 13. 与 P1-7 并行工具调用的交互

无依赖，无改动需求：

- Tracker 并发安全（mutex）。
- Pre-check / post-record 在每次 LLM 调用独立执行。
- `concurrent_tools` 集成测试覆盖。

## 14. 工作量预估

> 相比 raw 的 ~1370 LOC 缩水一圈。

| 目标 | LOC 估算 |
|---|---|
| `vv/configs/config.go` 新增字段 + env + defaults | ~60 |
| `vv/traces/budgets/tracker.go` + `error.go` | ~220 |
| `vv/traces/budgets/*_test.go` | ~250 |
| `vage/largemodel/budget_middleware.go` + test | ~130 |
| `vage/schema/event.go` 新 2 事件 + data | ~35 |
| `vv/cli/budget.go` + 接线 + test | ~110 |
| `vv/httpapis/budget.go` + 429 mapping + test | ~90 |
| `vv/setup/setup.go` 装配（preCheck/postRecord 闭包 + InitResult 字段） | ~60 |
| Integration tests | ~200 |

合计 **≈ 1,155 LOC**。主逻辑（tracker + middleware + wiring）约 420 LOC，其余为测试与接线。对应 feature-todo 的 "低" 复杂度。

## 15. 本次修订相对 raw 的 Diff 总览

| 项目 | raw | 本文 |
|---|---|---|
| 新增事件类型 | 3（Warn + Exceeded + Status） | 2（Warn + Exceeded） |
| Daily 持久化 | JSONL + replay | 无（P1-6 承接） |
| 预算记账路径 | 独立 Hook 订阅 EventLLMCallEnd | 中间件 post-record 闭包 |
| Tracker 参数 | 持 `*costtraces.Pricing` | 只吃预先算好的 cost |
| Scope 类型 | `type Scope string` + 常量 | 普通 `string` |
| warn_percent | 两个（session + daily） | 一个（共享） |
| Dispatcher options | `WithSessionBudget` + `WithDailyBudget` | 无 |
| 错误协议 | 仅结构体 | 结构体 + `ErrBudgetExceeded` sentinel + `Is()` |
| CLI 状态字段 | `m.sessionTracker`（位置误写） | `m.app.sessionBudget` / `dailyBudget` |
| Tracker.Add + WarnThresholdCrossed | 两步（有隐式顺序依赖） | 一步，`Add` 返回 `AddResult` |
| ResetIfWindowElapsed | 公开 API | 折叠进 Add/Check 内部 |
