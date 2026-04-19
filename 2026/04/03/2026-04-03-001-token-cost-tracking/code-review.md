# Code Review: Token Usage & Cost Tracking

## Summary

The implementation faithfully follows the design document across all three modules. The code is clean, well-documented, and follows project conventions (functional options, context.Context, composition). Below are the findings organized by severity, with notes on which were fixed during review.

## CRITICAL

None.

## HIGH

### H1: [FIXED] `costtraces` files missing Apache 2.0 license headers

**Files:** `vv/traces/costtraces/tracker.go`, `vv/traces/costtraces/defaults.go`, `vv/traces/costtraces/tracker_test.go`

All other `.go` files in the repo have the Apache 2.0 license header. These new files were missing it.

**Fix applied:** Added license headers to all three files.

### H2: Prefix matching is susceptible to false positives (accepted risk)

**File:** `vv/traces/costtraces/defaults.go:54`

The prefix check `model[:len(k)] == k` can match unintended models. For example, if a model named `gpt-4` existed, it would match `gpt-4o`. The design mitigates this via longest-prefix-first sorting, but there is no delimiter check. Consider requiring that the character after the prefix is a `-` or end-of-string to prevent ambiguous prefix matches. This is a design consideration more than a bug given the current pricing table. Not fixed -- accepted risk for now.

### H3: [FIXED] `VV_MODEL_PRICING` env parse error was silently swallowed

**File:** `vv/configs/config.go:200`

If `VV_MODEL_PRICING` contains malformed JSON, the error was silently ignored, which could confuse users who think their pricing override is active.

**Fix applied:** Added `slog.Warn` log when JSON parsing fails.

### H4: [FIXED] `enrichSyncResponse` did not copy response headers from recorder

**File:** `vv/httpapis/cost.go:37-55`

The `responseRecorder` captured the body, but `enrichSyncResponse` wrote to `w` without first copying headers (Content-Type, etc.) from the recorded response. This meant the Content-Type header set by the upstream handler was lost.

**Fix applied:** Added header copying loop before `WriteHeader` call.

## MEDIUM

### M1: [FIXED] Cost calculation in `enrichStreamResponse` duplicated tracker logic

**File:** `vv/httpapis/cost.go:199-205`

The cost calculation after the stream was done manually with the same formula as `costtraces.Tracker.Add()`.

**Fix applied:** Replaced manual cost calculation with a priced `costtraces.Tracker` instance that uses `Add()` + `Snapshot()`.

### M2: `streamRecorder` buffers the entire SSE stream before replaying (not fixed)

**File:** `vv/httpapis/cost.go:118-216`

The current `enrichStreamResponse` implementation buffers the entire SSE stream body in `streamRecorder`, then replays it line by line. This defeats the purpose of streaming -- the client receives nothing until the stream is complete. A proper implementation should use a streaming interceptor that writes events through in real-time while accumulating usage data, then appends the final usage event. This is a larger architectural change deferred for a follow-up.

### M3: [FIXED] `responseRecorder.WriteHeader` header forwarding (resolved via H4 fix)

The H4 fix ensures headers are copied from the recorder to the actual ResponseWriter before writing the status code and body.

### M4: [FIXED] `convertPricing` function was duplicated between CLI and HTTP

**Files:** `vv/cli/cli.go`, `vv/httpapis/http.go`

Both packages had identical functions converting `configs.ModelPricingEntry` to `costtraces.Pricing`.

**Fix applied:** Added `configs.ConvertPricing()` as a single shared function. Removed `convertPricing` from `cli/cli.go` and `convertHTTPPricing` from `httpapis/http.go`. Both callers now use `configs.ConvertPricing()`.

### M5: `shortModelName` date suffix regex only matches 8-digit suffixes (not fixed)

**File:** `vv/cli/render.go:382`

The regex `-\d{8}$` correctly matches Anthropic's date format (`-20250514`) but does not match OpenAI's format like `gpt-4o-2024-08-06` (hyphenated date). The test confirms `gpt-4o-20240806` works, but the real OpenAI date format uses hyphens: `gpt-4o-2024-08-06`. This would not be stripped correctly. Minor cosmetic issue, deferred.

## LOW

### L1: Test helpers use `new(0.7)` and `new(0.0)` syntax

**Files:** `aimodel/schema_test.go:32`, `vv/cli/render_test.go:402`

These use a Go generics helper or similar mechanism. Verified they compile and run. No action needed.

### L2: `costtraces.Tracker` recomputes cost from cumulative totals on every `Add()`

**File:** `vv/traces/costtraces/tracker.go:51-61`

Each call to `Add()` recomputes cost from the full cumulative state rather than incrementally. Fine for current scale.

### L3: `formatCost` display inconsistency at $1.00 boundary

**File:** `vv/cli/render.go:397-407`

The cost `$0.999` shows as `$0.999` (3 decimals) while `$1.000` shows as `$1.00` (2 decimals). Acceptable UX choice.

## POSITIVE OBSERVATIONS

1. **Thread safety**: `costtraces.Tracker` properly uses mutex for concurrent access, and the test verifies this with 100 goroutines.
2. **Snapshot isolation**: `Snapshot()` correctly deep-copies the `EstimatedCostUSD` pointer to prevent mutations.
3. **Cache-read double-charge prevention**: The cost calculation correctly subtracts cache-read tokens from input tokens before applying the input rate. This is tested.
4. **Prefix matching correctness**: The `gpt-4o` vs `gpt-4o-mini` disambiguation works correctly via longest-prefix-first sorting. Test coverage verifies this.
5. **Custom pricing override**: The custom pricing takes precedence over defaults, and `VV_MODEL_PRICING` env var merges correctly.
6. **Both emission paths updated**: The design requirement to update both MetricsMiddleware (sync) and TaskAgent.runStream() (direct) paths is fulfilled.
7. **OpenAI `prompt_tokens_details.cached_tokens` extraction**: Handled elegantly via custom `UnmarshalJSON` with a private `usageJSON` intermediary struct.
8. **Comprehensive test coverage**: All new functionality has tests -- cost accumulation, pricing lookup, CLI rendering helpers, HTTP middleware, and Anthropic stream cache tokens.

## Changes Applied During Review

| Issue | File(s) | Change |
|-------|---------|--------|
| H1 | `vv/traces/costtraces/tracker.go`, `defaults.go`, `tracker_test.go` | Added Apache 2.0 license headers |
| H3 | `vv/configs/config.go` | Added `slog.Warn` for malformed `VV_MODEL_PRICING` JSON |
| H4 | `vv/httpapis/cost.go` | Added header copying in `enrichSyncResponse` |
| M1 | `vv/httpapis/cost.go` | Replaced manual cost calc with `costtraces.Tracker` |
| M4 | `vv/configs/config.go`, `cli/cli.go`, `httpapis/http.go` | Consolidated `ConvertPricing` into `configs` package |

All three modules (`aimodel`, `vage`, `vv`) pass `make build` (format + lint + test) after these changes.
