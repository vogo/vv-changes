# Integration Test Report: Orchestrator Agent

## Summary

**Status:** ALL TESTS PASSED
**Date:** 2026-03-26
**Total Tests:** 68 integration tests
**New Tests Added:** 6
**Duration:** 0.870s

## Design Test Plan Coverage

All 10 integration tests from the design document are covered:

| Design Test | Test Name | Status |
|---|---|---|
| Test 1: Direct Dispatch | `TestIntegration_Agents_OrchestratorDirectDispatch` | PASS |
| Test 1: Streaming | `TestIntegration_Agents_OrchestratorImplementsStreamAgent` | PASS |
| Test 2: Plan Execution | `TestIntegration_Agents_OrchestratorPlanExecution` | PASS |
| Test 3: Fallback (invalid JSON) | `TestIntegration_Agents_OrchestratorFallbackOnInvalidJSON` | PASS |
| Test 3: Fallback (empty plan) | `TestIntegration_Agents_OrchestratorFallbackOnEmptyPlan` | PASS |
| Test 3: Fallback (invalid agent) | `TestIntegration_Agents_OrchestratorFallbackOnInvalidAgent` | PASS |
| Test 3: Fallback (LLM error) | `TestIntegration_Agents_OrchestratorFallbackOnLLMError` | PASS (NEW) |
| Test 4: Working Directory | `TestIntegration_Agents_WorkingDirectoryCaptureAndPropagation` | PASS (NEW) |
| Test 5: CLI Wiring | `TestIntegration_CLI_OrchestratorWiring` + `TestIntegration_CLI_AppConstruction` | PASS |
| Test 6: HTTP Service | `TestIntegration_FullWiring` | PASS |
| Test 7: Parallel Steps | `TestIntegration_Agents_OrchestratorParallelStepExecution` | PASS (NEW) |
| Test 8: Step Failure | `TestIntegration_Agents_OrchestratorPlanStepFailure` | PASS (NEW) |
| Test 9: Token Usage | `TestIntegration_Agents_OrchestratorTokenUsageAggregation` | PASS (NEW) |
| Test 10: Chat in Plan | `TestIntegration_Agents_OrchestratorChatInPlanSteps` | PASS (NEW) |

## New Tests Added

1. **TestIntegration_Agents_WorkingDirectoryCaptureAndPropagation** (Design Test 4)
   - Verifies `os.Getwd()` populates `BashWorkingDir` when empty
   - Verifies sub-agent request contains working directory context as first message
   - Uses a `recordingStubAgent` to capture the enriched request

2. **TestIntegration_Agents_OrchestratorParallelStepExecution** (Design Test 7)
   - Verifies two independent plan steps start within 100ms of each other
   - Uses `timingStubAgent` with `atomic.Int64` timestamps to verify parallelism
   - Both steps use 50ms delay to simulate work

3. **TestIntegration_Agents_OrchestratorPlanStepFailure** (Design Test 8)
   - Verifies one failing step does not cause panic or unhandled error
   - Uses `failingStubAgent` that always returns an error
   - Confirms response contains output from the successful step

4. **TestIntegration_Agents_OrchestratorTokenUsageAggregation** (Design Test 9)
   - Verifies classification usage (50/20/70) + sub-agent usage (200/100/300) = aggregated (250/120/370)
   - Checks all three fields: PromptTokens, CompletionTokens, TotalTokens

5. **TestIntegration_Agents_OrchestratorChatInPlanSteps** (Design Test 10)
   - Verifies chat agent can be used in plan steps alongside coder
   - Uses `callbackStubAgent` with `atomic.Bool` to confirm both agents invoked

6. **TestIntegration_Agents_OrchestratorFallbackOnLLMError** (Design Test 3, LLM error variant)
   - Verifies fallback when LLM returns an error (not just invalid JSON)
   - Confirms chat agent handles the request as fallback

## Helper Types Added

- `recordingStubAgent`: Records last request for assertion
- `timingStubAgent`: Records invocation timestamps with atomic operations
- `failingStubAgent`: Always returns a configured error
- `callbackStubAgent`: Invokes a callback on Run for invocation tracking

## Existing Tests (unchanged, comments added)

All existing integration tests continue to pass. Test case comments were added to all test functions documenting the specific assertions being verified.
