# Code Review: Orchestrator Context Exploration

## Files Reviewed

- `vv/agents/orchestrator.go` -- main implementation
- `vv/agents/agents.go` -- Create function updates
- `vv/agents/orchestrator_test.go` -- unit tests
- `vv/integrations/agents_test.go` -- integration tests

## Summary

The implementation closely follows the design document. The code is well-structured, readable, and follows existing project conventions. Tests are comprehensive with good coverage of happy paths, edge cases, and fallback behavior. Build, vet, and all tests pass cleanly.

## Issues Found

### Issue 1: Plan steps do not receive exploration context summary (Medium)

**Location:** `orchestrator.go`, `buildNodes` method (line 658) and `runPlan` call site (line 265).

**Description:** The design (section 2.8) states: "Plan step InputMapper functions are also updated to include the context summary." However, the current `buildNodes` InputMapper closures only inject `workingDir` and `plan.Goal` into plan step messages. The exploration context summary is not passed to plan step sub-agents.

The `Run` method creates `enrichedReq` with the context summary and passes it to `runPlan`, but `buildNodes` only uses `req.SessionID` from that request -- it reconstructs messages from scratch in each InputMapper. As a result, plan steps lose the exploration context that direct dispatch correctly receives.

**Fix:** Pass the exploration context summary into `buildNodes` and include it in the InputMapper closure messages, between the working directory and the original request goal.

### Issue 2: No issues with correctness, security, or performance

The remaining implementation is correct:
- `parseExplorationResponse` handles all edge cases (missing delimiters, empty sections, fallback to raw JSON).
- `exploreAndClassify` gracefully handles nil explorer, empty responses, parse failures, and invalid classifications.
- `enrichRequestWithContext` correctly short-circuits when both working dir and context are empty.
- Constructor properly gates explorer creation on both `workingDir` and `readOnlyReg` being non-nil.
- Usage aggregation handles nil cases correctly.
- Fallback paths work as expected in both `Run` and `RunStream`.

## Test Coverage Assessment

Tests are thorough:
- Unit tests: parseExplorationResponse (5 variants), exploreAndClassify (nil explorer, valid, invalid classification), enrichRequestWithContext (4 combinations), Run with exploration, Run with exploration fallback, constructor explorer creation (3 variants), helper functions.
- Integration tests: direct dispatch, plan execution, fallback on invalid JSON/empty plan/invalid agent, streaming, parallel execution, step failure, token usage aggregation, working directory propagation.
- All existing tests pass with the new constructor parameters (nil registry, zero-value config).

## Changes Applied

1. Fixed Issue 1: Updated `buildNodes` to accept and propagate exploration context summary to plan step InputMapper closures.
