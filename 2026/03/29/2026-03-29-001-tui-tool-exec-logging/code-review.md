# Code Review: TUI Tool Execution Logging

## Files Reviewed

- `vv/agents/orchestrator.go` -- new `exploreStream` and `planTaskStream` methods, updated `RunStream`
- `vv/agents/orchestrator_test.go` -- 4 new test cases and 2 new test helper types

## Summary

The implementation correctly follows the design. The new streaming methods (`exploreStream`, `planTaskStream`) mirror the structure of the existing non-streaming methods and are wired into `RunStream` with minimal changes. Test coverage is thorough, covering tool event forwarding, text capture/context enrichment, planner tool events, and non-streaming explorer fallback.

## Issues Found and Fixed

### 1. Bug: Usage token accumulation overwrites instead of summing (FIXED)

**Severity:** Medium

**Location:** `exploreStream` (line ~578) and `planTaskStream` (line ~660)

**Problem:** Both methods used `usage = &aimodel.Usage{...}` inside the `EventLLMCallEnd` handler, which overwrites the usage pointer on each event. For multi-iteration agents that make multiple LLM calls, only the last call's token counts would be reported. This is inconsistent with `forwardSubAgentStream` which correctly uses `tokensUsed += data.TotalTokens`.

**Fix:** Changed both methods to use a stack-allocated `aimodel.Usage` struct with a `hasUsage` flag, accumulating tokens with `+=` across all `EventLLMCallEnd` events. A pointer is returned only when at least one LLM call was observed.

### 2. Bug: Silent error on send failure allows wasted work (FIXED)

**Severity:** Low

**Location:** `exploreStream` (line ~575) and `planTaskStream` (line ~657)

**Problem:** Both methods used `_ = send(event)` to forward tool events, silently discarding send errors. If the stream consumer has disconnected, the methods would continue processing all remaining events unnecessarily. The existing `forwardSubAgentStream` correctly checks and returns send errors.

**Fix:** Changed `_ = send(event)` to `if err := send(event); err != nil { ... break }` with a warning log. A full `return err` was not used because these methods need to return their accumulated results even on partial send failure, but the loop is now broken to avoid unnecessary work.

## No Issues (Positive Observations)

- **Correct fallback path:** Both methods correctly fall back to non-streaming equivalents when the agent does not implement `StreamAgent`.
- **Event forwarding strategy matches design:** Only `EventToolCallStart`, `EventToolResult`, and `EventError` are forwarded; all others are suppressed or captured.
- **Request construction is consistent:** Both streaming methods build their requests identically to the non-streaming versions.
- **`RunStream` wiring is minimal:** Only two call-site changes in `RunStream`, exactly as the design specified.
- **Test coverage is comprehensive:** All 4 tests from the design spec are implemented. The `stubStreamExplorer` and `capturingStreamAgent` helpers are well-designed and reusable.
- **No changes to `Run()` path:** The non-streaming path is completely untouched.

## Tests

All tests pass after the fixes:

```
ok  github.com/vogo/vv/agents  0.896s
```
