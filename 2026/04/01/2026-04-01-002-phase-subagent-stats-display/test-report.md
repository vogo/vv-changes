# Test Report: Phase & Sub-Agent Stats Display

## Summary

All tests PASS across both modules (`vage` and `vv`). Integration tests were written and verified for the stats tracking and rendering pipeline without requiring real LLM API keys.

## Test Execution

### vage module

```
cd /Users/hk/workspaces/github/vogo/vagents/vage && make test
```

**Result:** All packages PASS. No failures. Coverage unchanged from baseline (no source changes in vage for this test task).

### vv module

```
cd /Users/hk/workspaces/github/vogo/vagents/vv && make test
```

**Result:** All packages PASS. `dispatches` coverage at 65.9%, `cli` coverage at 37.4%.

## New Test Files

### `vv/dispatches/stats_integration_test.go` (7 tests)

| Test | What It Verifies |
|------|-----------------|
| `TestRunStream_PhaseEndContainsStats` | All 3 phase (explore, plan, dispatch) PhaseEndData events carry correct PromptTokens, CompletionTokens, ToolCalls, and Duration from mock streaming agents |
| `TestRunStream_SubAgentEndContainsTokenBreakdown` | SubAgentEndData has separate PromptTokens/CompletionTokens and TokensUsed equals their sum |
| `TestRunStream_StatsConsistency` | Sum of PhaseEndData tokens equals sum of LLMCallEndData tokens across the entire event stream |
| `TestRunStream_NonStreamingSubAgentStats` | Non-streaming (Run fallback) sub-agent path correctly populates SubAgentEndData from resp.Usage |
| `TestRunStream_ZeroStatsPhase` | Phases with no LLM events emit PhaseEndData with zero token/tool counts and non-negative duration |
| `TestRunStream_LargeTokenCounts` | Large token values (1.2M prompt, 100k tool calls) accumulate correctly without overflow |
| `TestRunStream_PhaseStartAndEndPairing` | Every PhaseStart has a corresponding PhaseEnd, phases appear in explore/plan/dispatch order, and Start always precedes End in the event stream |

### `vv/cli/stats_integration_test.go` (9 tests, some with sub-tests)

| Test | What It Verifies |
|------|-----------------|
| `TestModelAccumulatesTaskStats` | CLI model accumulates totalPromptTokens/totalCompletionTokens from multiple LLMCallEnd events |
| `TestModelAccumulatesToolCalls` | ToolCallStart events increment both toolCallCount and totalToolCalls |
| `TestModelSubAgentStatsReset` | SubAgentStart resets sub-agent accumulators; SubAgentEnd resets them again; task-level totals persist |
| `TestModelSubAgentFallbackStats` | When SubAgentEndData has zero tokens (DAG path), CLI model's locally tracked sub-agent stats are used as fallback |
| `TestPhaseEndRendering` | Phase transition rendering with all stats, no tools, zero tokens (DAG), and large values (1.5M tokens) |
| `TestTaskCompleteRendering` | Task completion line with full stats, duration-only, sub-second, and minute-plus durations |
| `TestMultiSubAgentAccumulation` | Task-level stats correctly accumulate across two sequential sub-agents |
| `TestFormatDuration_EdgeCases` | Duration formatting at boundaries: 0ms, 999ms, 1s, 59s, 60s/1m, 61s, 1h, 61m1s |
| `TestBuildStatsLine_AllCombinations` | Stats line with zero duration, tools-only, prompt-only, completion-only, singular tool use |

## Existing Tests (Pre-Existing, Verified Still Passing)

### `vv/dispatches/phase_tracker_test.go` (2 tests)

- `TestPhaseTracker_Accumulation` -- phaseTracker.wrap correctly counts events
- `TestPhaseTracker_Empty` -- Zero stats when no events sent

### `vv/cli/render_test.go` (16 tests covering stats features)

- `TestFormatCompactTokens` -- 0, 999, 1.0k, 5.3k, 1.2M
- `TestBuildStatsLine` -- Full stats with all parts
- `TestBuildStatsLine_DurationOnly` -- Duration-only produces "(500ms)"
- `TestBuildStatsLine_SingularToolUse` -- "1 tool use" vs "tool uses"
- `TestBuildStatsLine_ZeroToolCallsOmitted` -- No "tool" when ToolCalls=0
- `TestBuildStatsLine_ZeroTokensOmitted` -- No arrows when tokens=0
- `TestRenderPhaseTransition_End` -- "phase Dispatch complete." with stats
- `TestRenderPhaseTransition_Start` -- Starting mode ignores stats
- `TestRenderSubAgentEnd_WithTokenBreakdown` -- Separate up/down token counts
- `TestRenderSubAgentEnd_DurationOnly` -- Duration only, no arrows
- `TestRenderTaskComplete` -- "task complete." with tokens
- `TestRenderTaskComplete_NoTokens` -- "task complete." without tokens

## Test Architecture

The integration tests use custom mock agents (`statsStreamAgent`, `statsExplorerAgent`, `statsPlannerAgent`) that emit configurable LLM call events and tool call events during streaming. This allows end-to-end testing of the full stats pipeline (agent -> forwardSubAgentStream -> phaseTracker -> PhaseEndData/SubAgentEndData -> CLI model accumulation -> rendering) without any real LLM API calls.

## Coverage of Design Sections

| Design Section | Test Coverage |
|---------------|--------------|
| 3.1 Schema changes (PhaseEndData, SubAgentEndData) | Verified via dispatcher integration tests |
| 3.2 forwardSubAgentStream token tracking | TestRunStream_SubAgentEndContainsTokenBreakdown, TestRunStream_NonStreamingSubAgentStats |
| 3.3 Explore/classify LLM event forwarding | TestRunStream_PhaseEndContainsStats (explore/plan phases have non-zero tokens) |
| 3.4 phaseTracker accumulation | TestPhaseTracker_Accumulation, TestPhaseTracker_Empty, TestRunStream_StatsConsistency |
| 3.5 Render helpers (formatCompactTokens, buildStatsLine, renderTaskComplete) | Extensive unit + integration render tests |
| 3.6 CLI model event handling & task aggregation | TestModelAccumulatesTaskStats, TestMultiSubAgentAccumulation, TestModelSubAgentStatsReset |
| Known limitation: DAG path | TestRunStream_ZeroStatsPhase, TestModelSubAgentFallbackStats |
| Edge cases: zero values | TestBuildStatsLine_DurationOnly, TestRunStream_ZeroStatsPhase |
| Edge cases: large numbers | TestRunStream_LargeTokenCounts, TestPhaseEndRendering/large_token_values |
