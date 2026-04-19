# Code Review: Dynamic Sub-Agent Support

## Files Reviewed
- `vv/agents/orchestrator.go` - Main changes for dynamic agent support
- `vv/agents/agents.go` - Wiring of tool registries
- `vv/agents/planner.go` - Planner prompt updates
- `vv/agents/orchestrator_test.go` - New tests
- `vv/integrations/agents_test.go` - Updated constructor calls

## Summary

The implementation is well-structured and closely follows the design document. Code is idiomatic Go, validation is thorough, and test coverage is comprehensive. A few issues were found and fixed.

## Issues Found and Fixed

### Issue 1: Missing `RunTokenBudget` for dynamic agents (Medium)

**File:** `vv/agents/orchestrator.go`

The static coder agent passes `RunTokenBudget` from config (via `taskagent.WithRunTokenBudget`), but `buildDynamicAgent` does not. This means dynamic agents have no token budget limit, which is inconsistent with how static agents are configured and could lead to runaway token consumption.

**Fix:** Added `runTokenBudget` field to `OrchestratorAgent`, passed it through from `agents.go`, and applied it in `buildDynamicAgent` when non-zero.

### Issue 2: Constructor parameter count growing unwieldy (Low - not fixed)

`NewOrchestratorAgent` now has 13 parameters. This is approaching the point where an options pattern or config struct would improve readability. Not fixed in this review since it would be a larger refactor, but flagged for future consideration.

## Positive Observations

1. **Clean separation of concerns**: Dynamic agent building is isolated in `buildDynamicAgent`, making it easy to test independently.
2. **Backward compatibility**: The `DynamicSpec` field on `PlanStep` uses `omitempty` and nil-checks, so existing JSON without `dynamic_spec` continues to work.
3. **Validation is thorough**: The `validate()` methods cover missing fields, invalid values, and the agent/base_type consistency rule.
4. **Test coverage is comprehensive**: Tests cover defaults, overrides, nil registries, JSON backward compatibility, mixed plans, and error paths.
5. **Planner prompt update is minimal**: Only ~4 lines added, keeping token usage low.
6. **Good error messages**: Error strings include step IDs and field values for debugging.
