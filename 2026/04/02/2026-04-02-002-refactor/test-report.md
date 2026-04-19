# Integration Test Report: Dispatcher Refactoring to Adaptive Decision Loop

**Date:** 2026-04-02
**Module:** `vv/dispatches`
**Test File:** `vv/dispatches/dispatcher_integration_test.go`
**Runner:** `cd vv && go test ./dispatches/ -run TestIntegration -v`

## Summary

All 13 integration tests passed. All 59 existing unit tests also passed, confirming no regressions.

| Metric | Value |
|--------|-------|
| Total integration tests | 13 (including 5 subtests) |
| Passed | 13 |
| Failed | 0 |
| Total tests in package | 72 |
| Duration | 0.296s |

## Integration Tests Mapped to Design Test Plan

| # | Test | Design Ref | Acceptance Criteria | Status |
|---|------|-----------|---------------------|--------|
| 1 | `TestIntegration_SimpleIntentFastPath` | Test 1 | US-1, US-2, US-3 | PASS |
| 2 | `TestIntegration_CodeRequestWithExploration` | Test 2 | US-1, US-2 | PASS |
| 3 | `TestIntegration_ComplexTaskPlanning` | Test 3 | US-1, US-3 | PASS |
| 4 | `TestIntegration_ReplanOnStepFailure` | Test 4 | US-4 | PASS |
| 5 | `TestIntegration_ReplanCountLimit` | Test 5 | US-4 | PASS |
| 6 | `TestIntegration_RecursionDepthEnforcement` | Test 6 | US-5 | PASS |
|   | - `depth_0_full_loop` | | Depth 0: full decision loop | PASS |
|   | - `depth_at_max_skips_intent` | | Depth >= max: skip intent | PASS |
|   | - `depth_above_max` | | Depth > max: skip intent | PASS |
| 7 | `TestIntegration_SummaryPolicy_Auto_CLI` | Test 7 | US-6 (CLI skip) | PASS |
| 8 | `TestIntegration_SummaryPolicy_Auto_HTTP` | Test 8 | US-6 (HTTP summarize) | PASS |
| 9 | `TestIntegration_SummaryPolicy_AlwaysNever` | Test 9 | US-6 (force on/off) | PASS |
|   | - `always_summarizes_in_cli` | | SummaryAlways in CLI | PASS |
|   | - `never_skips_in_http` | | SummaryNever in HTTP | PASS |
| 10 | `TestIntegration_StreamingPhaseEvents` | Test 10 | US-1, ORCH-14, ORCH-21 | PASS |
| 11 | `TestIntegration_StreamingWithSummarizePhase` | (supplemental) | US-6 + streaming | PASS |
| 12 | `TestIntegration_DepthPropagationThroughExecute` | (supplemental) | US-5 propagation | PASS |
| 13 | `TestIntegration_EndToEnd_ExplorerPlusDirect` | (supplemental) | Full pipeline E2E | PASS |

## Test Approach

Since LLM API keys are not available, all integration tests mock the LLM layer using:

- `sequentialMockLLM` -- returns different responses on successive `ChatCompletion` calls, enabling multi-call flows (intent + re-assessment).
- `callTrackingAgent` -- records whether an agent was invoked, verifying correct routing.
- `failingAgent` -- always returns an error, used for replanning tests.
- Existing test helpers (`stubAgent`, `stubStreamAgent`) from the unit test suite.

Tests exercise the full Dispatcher pipeline end-to-end: constructor with options, `Run`/`RunStream`, intent recognition, exploration, execution, replanning, and summarization.

## Key Verifications

1. **Simple requests** use exactly 1 LLM call and skip exploration.
2. **Exploration** is triggered on-demand when LLM returns `needs_exploration: true`, followed by re-assessment.
3. **Plan mode** builds and executes a DAG with correct agent dispatch.
4. **Replanning** triggers on step failure when policy is enabled, respects `MaxReplans`.
5. **Depth control** skips intent recognition at `maxRecursionDepth` and dispatches directly to fallback.
6. **Summary policy** correctly gates summarization based on mode metadata and policy setting.
7. **Streaming** emits correct phase events (`intent`, `execute`, `summarize`) with `TotalPhase=0` and proper ordering.
8. **Usage aggregation** correctly sums prompt/completion tokens across intent, exploration, and execution phases.

## Existing Test Suite

All 59 pre-existing unit tests continue to pass, confirming backward compatibility of the refactoring.
