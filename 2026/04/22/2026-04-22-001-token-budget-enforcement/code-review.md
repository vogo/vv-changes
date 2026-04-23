# Code Review — P1-3 Token Budget Enforcement

Scope: the delta described by `design.md`, i.e. the touch list across `vage/`
(schema, largemodel) and `vv/` (configs, setup, cli, httpapis, main,
traces/budgets). Review runs on the in-tree state produced by the developer
phase; baseline unit tests were green before and after the changes below.

## Files reviewed

- `vv/traces/budgets/{tracker.go,error.go,wire.go,tracker_test.go,error_test.go}`
- `vage/largemodel/{budget_middleware.go,budget_middleware_test.go}`
- `vage/schema/event.go` (2 new event constants + `BudgetWarnData` /
  `BudgetExceededData`)
- `vv/configs/{config.go,config_test.go}` (`BudgetConfig`, `applyBudgetEnv`,
  default `WarnPercent=0.8`)
- `vv/setup/setup.go` (`InitResult.SessionBudget`/`DailyBudget`,
  `buildBudgetTrackers`, `budgetEventDispatcher`)
- `vv/cli/{cli.go,memory.go,budget.go,budget_test.go}` (`WithBudgetTrackers`,
  `/budget` command)
- `vv/httpapis/{http.go,budget.go,budget_test.go}` (`/v1/budget` route +
  `budgetErrorMiddleware` 429 mapping)
- `vv/main.go` (plumb `initResult.SessionBudget` / `DailyBudget` into CLI
  and HTTP wiring)
- `vv/integrations/eval_tests/http_eval_test.go` (updated `Serve` signature)

## Principle checks

**Correctness**

- `Tracker.Add`/`Check`/`Snapshot` hold the mutex exactly once per call and
  roll the daily window under the same lock, matching design §0 guarantees.
  Concurrency test (`TestConcurrentAddExactTotals`) verifies the invariant.
- `BudgetExceededError.Unwrap()` returns `ErrBudgetExceeded`, so
  `errors.Is(err, ErrBudgetExceeded)` and `IsExceeded` both work through any
  `fmt.Errorf("...: %w", e)` wrapping. Tests
  (`TestWrappedSentinelMatchesErrorsIs`) confirm.
- Warn-threshold detection uses `prev < warnPct && cur >= warnPct`, and
  `warnFired` is single-shot under the mutex; daily roll resets it. No false
  re-fires in the same window.
- `wire.Wire` feeds the middleware; `session.Add` and `daily.Add` run
  sequentially on different mutexes (no deadlock). Pre-check returns the
  first violation (session before daily), matching design §2.1.
- `BudgetMiddleware.Wrap` on streams installs the post-record via
  `aimodel.WrapStream`; the host nil-usage guard in the wire closure
  makes the test case (`TestBudgetMiddlewareStreamPassesThroughWhenUpstreamNil`)
  correct.

**Design fit**

- Zero-cost when disabled: `setup.Init` skips installing the middleware
  entirely when both trackers are nil; `wire.Wire` returns `(nil, nil)` in
  that case.
- CLI/HTTP plumbing flows from `InitResult` (no Dispatcher option changes),
  matching design §8.
- Events (`EventBudgetWarn`, `EventBudgetExceeded`) are additive and do not
  regress `EventTokenBudgetExhausted` (still Run-level, taskagent path
  unchanged).

**Security**

- API-key and budget values are only logged through `slog` with structured
  fields; no plain-text leakage added.
- `budgetErrorMiddleware` regex/string matching is bounded to the body
  recorder buffer (no reflection/unsafe paths). Matches only on JSON
  content-type AND explicit "budget exceeded" signature; false positives
  across unrelated error strings are unlikely.

**Concurrency**

- Tracker: single mutex, no goroutines spawned internally. Safe.
- Middleware: `DispatchFunc` is optional and is called outside the tracker
  lock (inside `wire.apply`), so dispatch handlers cannot deadlock the
  tracker.
- Stream path: `WrapStream`'s `onClose` is called from `Close()` which uses a
  CAS to ensure at-most-once invocation (per `aimodel.Stream.Close`). Good.

**Error handling**

- `BudgetExceededError.Error()` has branches for "cost" vs "tokens" so
  displayed text reflects the dimension that triggered the rejection.
- `applyBudgetEnv` logs-and-ignores invalid env values, preserving the YAML
  value. Consistent with the rest of `configs.Load`.

## Findings

### Applied (fixed in-tree as part of this review)

1. **`vv/httpapis/http.go` — `budgetErrorMiddleware` installed unconditionally.**
   `Serve` wrapped the service handler with `budgetErrorMiddleware` on every
   request, including when no budget tracker was configured. That middleware
   buffers the full response body, which is pure overhead when no
   budget_exceeded error can ever occur. Fixed: only wrap when
   `sessionBudget != nil || dailyBudget != nil`. This matches the design
   §0 "Zero configuration zero cost" invariant. No test regressions.

2. **`vv/cli/budget.go` — redundant nested `if snap.HardTokens > 0`.**
   The inner guard was unreachable (outer `if` already asserted the same
   condition). Collapsed to a single expression. Behavior unchanged.

3. **`vv/cli/budget.go` — daily-vs-session detection relied on `now != nil`.**
   The "(resets in …)" hint was gated only by `now != nil` and a non-zero
   `WindowStart`, but session trackers also have a non-zero `WindowStart`
   (set at creation). If a future caller accidentally passed a non-nil clock
   for a session tracker, the session row would render a bogus window
   countdown. Tightened the guard to also require
   `snap.Scope == budgets.ScopeDaily`. Defensive; no behavior change for
   current callers.

### Noted, not changed

4. **`aimodel.WrapStream` overwrites any existing `onClose` callback.**
   Pre-existing aimodel behavior — not introduced by this feature. In the
   current vv chain (`Budget → Debug → base`) nothing sits inside Budget
   that sets `onClose`, so there is no observable breakage today. But if
   anyone later inserts `MetricsMiddleware` between Budget and the base
   client, Budget's `WrapStream` would silently overwrite Metrics'
   `EventLLMCallEnd` emission. Recommend `aimodel.WrapStream` be made
   additive (chain like `InterceptStream` does) in a follow-up; out of
   scope for P1-3.

5. **Event payload dimension mix-up for cost-only trackers.**
   `wire.newWarnEvent` always populates `Used=snap.UsedTokens` and
   `Limit=snap.HardTokens` even when `dimension=="cost"`, so a pure
   cost-only warn event ends up with `Used=0 Limit=0` and a meaningful
   `Percent`/`UsedCost`/`LimitCost`. The slog dispatcher in `setup.go`
   logs both sets, so the information is not lost. Acceptable in P1 given
   there are no external subscribers yet; if a structured dashboard
   consumes these events later, consider sourcing `Used`/`Limit` from the
   exceeded dimension.

6. **`budgetErrorMiddleware` body buffering vs SSE.**
   The middleware buffers the entire upstream response, including SSE for
   `/stream` paths. The outer `costEnrichMiddleware` already buffers via
   `streamRecorder{Flush: no-op}`, so this is not a regression — SSE
   is already delivered only after the handler returns. This is worth
   revisiting holistically when SSE latency becomes a product concern;
   outside the scope of P1-3.

7. **`Tracker.Add` accepts negative `tokens` / `costUSD`.**
   The tracker decrements freely on negative inputs. Every caller today
   passes positive usage (`aimodel.Usage` fields are ints ≥ 0), and
   defensive clamping would mask caller bugs. Acceptable as-is.

8. **`budgetErrorMiddleware` match is textual ("budget exceeded").**
   Any upstream handler that happens to include "budget exceeded" in
   its JSON error message would get rewritten to 429. The risk is low —
   no other component in the monorepo uses that string — but tests (or a
   dedicated error `type` field) would be the preferred signal long
   term. The middleware already prefers the structured
   `{"error":{"type":"budget_exceeded"}}` shape when present, which is
   the correct fallback.

## Test results

- `cd vv && go test ./...` → **all packages pass** (incl. `traces/budgets`,
  `cli`, `configs`, `httpapis`, `setup`, every `integrations/*_tests`).
- `cd vage && go test ./...` → **all packages pass** (incl. `largemodel`,
  `schema`).

No unit-test regressions from the applied fixes. No new flakiness observed.
