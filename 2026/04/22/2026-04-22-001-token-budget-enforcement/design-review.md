# Design Review: Token Budget Enforcement (P1-3)

Senior-engineer review of `design-raw.md`. Ordered by impact — simplifications first.

## 1. Cut Daily persistence from P1. Biggest single simplification.

Daily JSONL persistence is the largest chunk of code and risk in the design (`~150 LOC` + replay + corruption handling + a promise we can't keep for multi-process). The requirement spec calls for persistence only as a "simplest" choice; it does not forbid dropping it. Given:

- vv is single-user / single-process in P1 (explicit assumption #1).
- A restart mid-day is rare; daily budget is coarse-grained.
- JSONL append-per-LLM-call adds a disk syscall on the hot path.
- The feature is trivially re-addable later when SQLite lands (P1-6).

**Decision**: P1-3 ships Daily as **in-memory** only, resetting on restart. Add a one-line note: "on restart, today's accumulated usage is lost; acceptable tradeoff; SQLite-backed persistence lands with P1-6." This also lets us drop `DailyPersistDir` from config, `persist.go` + its tests, and the replay logic in `NewDaily()`.

This alone cuts ~200 LOC of code + tests and eliminates the thorniest error paths.

## 2. Collapse three events into one.

The design adds `EventBudgetWarn`, `EventBudgetExceeded`, `EventBudgetStatus`. This mirrors the three states of the tracker, but it's not how downstream consumers want to read it.

- `EventBudgetStatus` fires on every LLM call — this is a firehose. CLI already reads `EventLLMCallEnd` and can compute a snapshot itself. HTTP `GET /v1/budget` is a pull endpoint and doesn't need a push event. Stream clients of HTTP receive existing events and can compute snapshot client-side. **Cut it.**
- `EventBudgetWarn` fires once. `EventBudgetExceeded` fires on rejection. These are genuinely different semantics: one is advisory, one is a hard wall. Keep both but name them consistently: `EventBudgetWarn` + `EventBudgetExceeded`.

**Decision**: 2 new events, not 3. Drop `EventBudgetStatus` and its data struct.

## 3. Drop `Scope` as a named type; use a plain string.

`type Scope string` with two constants is ceremony. It doesn't gain type safety because the field is still assigned from config and serialized to JSON as a string. `vage/schema/event.go` does not use named string types for `Phase`, `Direction`, `Action` (all plain strings). Match the existing convention: just use `string` with documented values `"session"` and `"daily"`.

**Decision**: Drop the `Scope` type. Use `scope string` in the Tracker constructor and event data.

## 4. Pricing coupling: push cost computation up to the caller.

The design has `budgets.Tracker.Add(in, out, cacheRead, costUSD)` — but to compute `costUSD` the Hook in §3.2 has to hold a `*costtraces.Pricing` pointer. That bakes a pricing dependency inside the budgets package. Cleaner: the Tracker takes already-computed cost, and the recorder (which has to wrap `costtraces` anyway) does the arithmetic using `costtraces.Pricing.Estimate()`.

Actually, read the existing `costtraces.Tracker`: **it already computes cost per Add**. We're duplicating its work. The cleanest wiring is:

```
LLM call → EventLLMCallEnd → recorder
  → costtraces.Tracker.Add(…)            // existing: cost + tokens
  → budgets.Tracker.Add(tokens, cost)     // thin delta-add
```

The budgets Tracker stays pricing-agnostic. The recorder orchestrates both.

**Decision**: `Tracker.Add(tokens int64, costUSD float64)` — tokens already summed (input+output; cache-read is inside input for Anthropic, don't double-count, matches existing costtraces logic). No pricing import inside `budgets/`.

## 5. Dispatcher does not need `WithSessionBudget` / `WithDailyBudget`.

The raw design already notes (§8) that "Dispatcher side changes are limited to: retain references for rendering/snapshot." So why add two new options? The Tracker references are owned by `setup.Init` (via the recorder closure & the pre-check closure) and by `App` / HTTP handler (for rendering). They do not need to travel through Dispatcher.

**Decision**: No Dispatcher options. `InitResult` exposes `SessionBudget *budgets.Tracker` and `DailyBudget *budgets.Tracker` (may be nil) for CLI/HTTP to wire up.

## 6. Merge recorder + middleware interaction — be realistic about what's wired today.

Design assumes `hook.Manager` dispatches `EventLLMCallEnd` to a recorder Hook. **This is not how vv is wired today.** There is no `MetricsMiddleware` installed on the LLM client in vv's setup — `TaskAgent` emits `EventLLMCallEnd` directly from its own stream loop (`task.go:1107`). The CLI listens to the stream, not to a hook manager. There is no `hookMgr.Dispatch` available at the LLM layer in vv.

Two viable fixes:

A. Install `MetricsMiddleware` on `wrappedLLM` in `setup.Init` and create a `hook.Manager` to dispatch events — but then TaskAgent's direct `send()` of `EventLLMCallEnd` will either duplicate or get suppressed. Looking at `task.go:965`, there's already a guard ("Skip hook dispatch for LLM lifecycle events — MetricsMiddleware already dispatches these directly to hooks to avoid double-counting"). The hook path is plumbed but not wired in vv.

B. Simpler: skip the hook manager entirely. The **BudgetMiddleware itself** does pre-check *and* post-record, directly on the Tracker references. No events need to drive accounting — the middleware sits on the LLM client and sees every `ChatResponse.Usage` directly. This is how every other production budget middleware works (LangGraph callbacks, LiteLLM).

**Decision**: Go with B. One middleware, pre-check + post-record inline. The middleware holds `Tracker` references directly and only emits `EventBudgetWarn` / `EventBudgetExceeded` via a `DispatchFunc` (the same pattern `MetricsMiddleware` uses). This drops `recorder`, the separate hook registration, and the whole "sync vs async hook" concern.

The CLI already extracts `LLMCallEndData.PromptTokens` off the stream for `costTracker` — we do **not** touch that path. Budget accounting runs in parallel, in the middleware, at the exact point of LLM response. Zero race.

This flips the design's own premise ("tracker updated via event subscription"). Good — the raw design was too clever.

## 7. Middleware position: confirm, not re-invent.

The raw design puts Budget before CircuitBreaker / Retry. Correct instinct. But since vv's `setup.go` does not currently build any middleware chain beyond the optional Debug wrapper, the whole "Log → Budget → CircuitBreaker → …" ordering from `DefaultChain` is aspirational. The real change is:

```go
// setup.Init, after wrappedLLM:
if budgetCfg.Enabled() {
    mw := largemodel.NewBudgetMiddleware(session, daily, hookMgr.Dispatch)
    wrappedLLM = mw.Wrap(wrappedLLM)
}
```

No `hookMgr` exists yet. For now, pass a dispatch func that writes `slog.Warn` + stashes warn state — the CLI reads warn state via the Tracker's `Snapshot()`, not via events. HTTP can later subscribe. **Accept that events are emitted but initially observed only by logging.**

**Decision**: The middleware emits events (for future stream subscribers / tests), but we do not build a hook.Manager for P1-3. `DispatchFunc` may be a no-op or a slog wrapper. This is consistent with the existing `MetricsMiddleware` usage pattern — it's optional.

## 8. `BudgetExceededError`: sentinel via `errors.Is`.

Raw design declares a struct with fields but doesn't implement `Is()`. Callers that want to distinguish budget errors from other LLM errors (CLI renderer, HTTP 429 mapper) need:

```go
var ErrBudgetExceeded = errors.New("budget exceeded")

type BudgetExceededError struct { Scope, Dimension string; Used, Limit int64; UsedUSD, LimitUSD float64 }
func (e *BudgetExceededError) Error() string { … }
func (e *BudgetExceededError) Is(target error) bool { return target == ErrBudgetExceeded }
func (e *BudgetExceededError) Unwrap() error { return ErrBudgetExceeded }
```

Plus public helper `budgets.IsExceeded(err) bool` for ergonomics. This is the pattern used by `taskagent.ErrBudgetExhausted`-style codes elsewhere.

**Decision**: Add sentinel + `Is()` as above.

## 9. Race window in `ResetIfWindowElapsed` + `Add`.

Raw design has:
```go
r.daily.ResetIfWindowElapsed(time.Now().UTC())
r.daily.Add(...)
```
Two separate mutex acquisitions. Between them the window can tick over — not bad for a benign Daily tracker (worst case: one extra token charged to previous day), but noisy. **Fold into a single method** `Add(tokens, cost)` that checks the window internally under the same mutex. The Tracker already needs the mutex for `used*` fields.

**Decision**: Collapse `ResetIfWindowElapsed` into `Add` (and into `Check`) under the same lock. Remove the public `ResetIfWindowElapsed` API.

## 10. `WarnThresholdCrossed` state machine: inline into `Add`.

Current design: `Add` mutates counters; then caller separately calls `WarnThresholdCrossed()` which returns true once. Two calls, implicit ordering dependency. Simpler: `Add` returns a `result` value:

```go
type AddResult struct {
    WarnCrossed bool   // first crossing of warn threshold this run
    Exceeded    *BudgetExceededError // non-nil iff hard limit crossed
}
func (t *Tracker) Add(tokens int64, cost float64) AddResult
```

This also lets the middleware post-check exit naturally — no need to call `Check` again after `Add`.

**Decision**: Return `AddResult` from `Add`. Drop `WarnThresholdCrossed()`.

## 11. Config: one `WarnPercent` is enough.

Raw design has `SessionWarnPercent` and `DailyWarnPercent`. In practice they're always set to the same value (industry default 0.8). Two knobs double the config surface for ~zero user benefit. Collapse to a single `WarnPercent` that applies to whichever layers are enabled. If someone genuinely wants different thresholds later, split it.

**Decision**: One `BudgetConfig.WarnPercent` field (default 0.8). Drop the daily-specific one.

## 12. CLI state: `m.sessionTracker` is wrong.

Raw design writes `m.sessionTracker` but the CLI `App`/`model` currently holds `costTracker *costtraces.Tracker`, not a budget tracker. Add:

```go
type App struct {
    …existing…
    sessionBudget *budgets.Tracker // may be nil
    dailyBudget   *budgets.Tracker // may be nil
}
```

And plumb from `setup.InitResult` → `cli.New(...)` → `App`. `model` accesses via `m.app.sessionBudget`. Exactly how `m.app.costTracker` is accessed today.

**Decision**: Call out "threaded through App, not model, same path as costTracker".

## 13. Keep the rest as-is.

- Pre-check in middleware + BudgetExceededError wrapping: correct.
- Run-level budget unchanged: correct.
- cache-read not double-counted in tokens: correct.
- Config env overrides: match existing `VV_…` pattern, correct.
- Zero-config zero-overhead path: correct.
- Integration test list: good coverage; drop `daily_persist_*` cases per §1.

## Net Scope Cut

| Before | After | Delta |
|---|---|---|
| 3 new event types | 2 | -50 LOC |
| Daily JSONL persist + replay + tests | none | -200 LOC |
| recorder hook + hook.Manager wiring | none | -60 LOC |
| Pricing import in budgets pkg | none | cleaner deps |
| 2 warn_percent knobs | 1 | simpler config |
| Scope named type | plain string | -10 LOC |

Estimated total: **~1370 → ~900 LOC**. Closer to the "low" complexity target in feature-todo.

## Items the design must explicitly state post-update

1. Daily tracker is in-memory only for P1; restart resets it. Explicit follow-up: "persist daily usage with P1-6 SQLite".
2. The BudgetMiddleware does both pre-check and post-record, holds Tracker refs directly. No hook.Manager required.
3. `InitResult.SessionBudget` / `InitResult.DailyBudget` are the plumbing point.
4. `errors.Is(err, budgets.ErrBudgetExceeded)` is the supported caller-side check.
