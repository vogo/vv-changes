# Test Report v1 — Token Budget Enforcement (P1-3)

## Summary

- **Test file created:** `vv/integrations/budget_tests/budget_test.go`
- **Total integration cases:** 9 (8 from design §10.2 + 1 supplemental exceeded-event check)
- **Pass / Fail:** 9 / 0
- **Full `go test ./...` in `vv/`:** ALL PASS (including existing 20+ integration sub-packages)
- **Race detector (`-race`):** enabled, clean

## Scope

Design §10.2 lists 8 integration scenarios. All 8 are implemented without requiring live LLM credentials — every test stubs `aimodel.ChatCompleter` with a counting fake and drives it through `largemodel.NewBudgetMiddleware` + `budgets.Wire`.

## Scenario Coverage

| # | Scenario                  | Test Function                                    | Status |
|---|---------------------------|--------------------------------------------------|--------|
| 1 | `session_hard_tokens`     | `TestIntegration_SessionHardTokens`              | PASS   |
| 2 | `session_warn_percent`    | `TestIntegration_SessionWarnPercent`             | PASS   |
| 3 | `daily_window_roll`       | `TestIntegration_DailyWindowRoll`                | PASS   |
| 4 | `nested_agent`            | `TestIntegration_NestedAgent`                    | PASS   |
| 5 | `concurrent_tools`        | `TestIntegration_ConcurrentTools`                | PASS   |
| 6 | `cost_and_tokens_both`    | `TestIntegration_CostAndTokensBoth`              | PASS   |
| 7 | `no_pricing_model`        | `TestIntegration_NoPricingModel`                 | PASS   |
| 8 | `disabled_all`            | `TestIntegration_DisabledAll`                    | PASS   |
| + | exceeded event dispatch   | `TestIntegration_ExceededEventDispatch`          | PASS   |

No scenarios were skipped. No scenarios gated behind `VV_LLM_API_KEY` — all paths are covered deterministically with the stub completer.

## What Each Test Verifies

- **session_hard_tokens** — After 2 successful calls that fill a 200-token session budget, the 3rd call is rejected at pre-check: `budgets.IsExceeded(err) == true`, `Dimension == "tokens"`, and the stub completer's call counter stays at 2 (no network contact).
- **session_warn_percent** — With limit=1000, warn=0.5: 0 warn events below 500 used; exactly 1 `EventBudgetWarn` when usage first reaches 500; still exactly 1 after usage climbs to 900 (one-shot semantic).
- **daily_window_roll** — Injected clock at day N 12:00 UTC; 900/1000 tokens consumed (1 warn fired); clock advanced to day N+1 01:00 UTC; next call rolls the window, counter resets to 0 and then lands at 100; subsequent fill to 800 produces a fresh (2nd) warn event — confirming `warnFired` resets on roll.
- **nested_agent** — Two goroutines (parent + child) drive the same wrapped completer; shared session tracker total equals `(parent+child) × per_call` exactly; stub call count matches the sum.
- **concurrent_tools** — 100 goroutines call the wrapped completer in parallel; final `UsedTokens` equals `goroutines × per_call` exactly under `-race`.
- **cost_and_tokens_both** — Pricing set so cost accrues $0.018/call against a $0.030 limit while tokens stay at 2000/call against a 10_000 limit; on the 3rd call the pre-check rejects with `Dimension == "cost"` — first-to-exceed semantics.
- **no_pricing_model** — Part A: `pricing=nil` with cost-only limit → 20 calls accumulate `UsedCostUSD == 0` and never trip the limit. Part B: pricing=nil with tokens-only limit → limit still enforced normally.
- **disabled_all** — `budgets.Wire(nil, nil, nil, nil)` returns `(nil, nil)`; a defensive `NewBudgetMiddleware(nil, nil).Wrap(c)` produces responses byte-identical to the underlying completer (ID + Usage match).
- **exceeded_event_dispatch** — One `EventBudgetExceeded` fires via the dispatcher when Add first crosses the limit; follow-up call hits the pre-check error path.

## Run Command

```bash
cd /Users/hk/workspaces/github/vogo/vagents/vv
go test ./integrations/budget_tests/... -v -race
# 9/9 PASS in ~1.6s

go test ./...
# All packages OK, no regressions
```

## Notes

- Source code was NOT modified — only the new test file was added.
- No external services required; CI-safe without `VV_LLM_API_KEY`.
- The stub completer also stands in for the LLM network boundary, so assertions on its call counter cleanly prove pre-check rejection happened before any "network" contact.

No failures occurred, so no `integrations-error-<N>.md` report was written.
