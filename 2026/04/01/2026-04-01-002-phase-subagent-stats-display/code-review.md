# Code Review: Phase & Sub-Agent Stats Display

**Reviewer**: Claude Opus 4.6  
**Date**: 2026-04-01  
**Status**: Approved with minor suggestions

## Files Reviewed

| File | Module | Change Type |
|------|--------|-------------|
| `vage/schema/event.go` | vage | Data model extension |
| `vv/dispatches/dispatch.go` | vv | Phase stats accumulation via `phaseTracker` |
| `vv/dispatches/stream.go` | vv | Sub-agent token breakdown, `streamPlan` return type |
| `vv/dispatches/explore.go` | vv | Forward `EventLLMCallEnd` via `send` |
| `vv/dispatches/classify.go` | vv | Forward `EventLLMCallEnd` via `send` |
| `vv/cli/render.go` | vv | `execStats`, formatting helpers, updated render functions |
| `vv/cli/render_test.go` | vv | New and updated tests |
| `vv/dispatches/phase_tracker_test.go` | vv | `phaseTracker` unit tests |

## Overall Assessment

The implementation faithfully follows the design document. The code is clean, well-structured, and consistent with existing patterns. All tests pass in both `vage` and `vv` modules.

## Correctness

**Design conformance**: All six data-flow tasks from the design are implemented correctly:
1. `PhaseEndData` and `SubAgentEndData` extended with prompt/completion token fields.
2. `EventLLMCallEnd` forwarded from `exploreStream` and `classifyStream`.
3. `phaseTracker` introduced and used for all three phases in `RunStream`.
4. `forwardSubAgentStream` tracks `promptTokens`/`completionTokens` separately.
5. Rendering helpers (`formatCompactTokens`, `buildStatsLine`, `renderTaskComplete`) implemented correctly.
6. CLI event handling accumulates task-level and sub-agent-level stats.

**DAG path handling**: The `streamPlan` return type correctly changed to `(*aimodel.Usage, error)`. DAG usage is augmented into `dispatchTracker` after `ExecuteDAG` returns. Sub-agent end events from DAG nodes correctly show duration-only stats.

**Backward compatibility**: `TokensUsed` field kept in `SubAgentEndData` and populated as `promptTokens + completionTokens`. JSON `omitempty` tags on new fields prevent breaking existing consumers.

## Consistency

- `phaseTracker` follows the existing wrapper/decorator pattern used throughout the codebase.
- `execStats` struct cleanly reduces parameter sprawl in render functions.
- Test naming follows existing conventions (`TestXxx_Yyy`).
- Error handling in forwarded events uses `slog.Warn` consistent with existing code.

## Edge Cases

All edge cases are handled correctly:

1. **Non-streaming fallback**: When explorer/planner agents don't support streaming, `phaseTracker` gets zero stats. `PhaseEndData` will show duration only. Acceptable.
2. **DAG path**: `SubAgentEndData` from `streamingDAGHandler.OnNodeComplete` has zero tokens. CLI falls back to `subAgentPromptTokens`/`subAgentCompletionTokens` tracked from `EventLLMCallEnd`. Since DAG nodes use `Run()` (non-streaming), these will also be zero. Stats show duration only. Acceptable per design.
3. **Zero values**: `buildStatsLine` correctly omits tool calls when zero, omits token arrows when zero, always includes duration.
4. **Singular/plural**: "1 tool use" vs "N tool uses" handled correctly.
5. **Task completion on error**: `renderTaskComplete` renders even after errors, which matches the design intent.

## Test Coverage

Good coverage of the new functionality:

- `TestFormatCompactTokens`: Boundary values (0, 999, 1000, 5300, 1200000).
- `TestBuildStatsLine`: Full stats, duration-only, singular tool use, zero tool calls omitted, zero tokens omitted.
- `TestRenderPhaseTransition_End`/`_Start`: Both modes with stats verification.
- `TestRenderSubAgentEnd_WithTokenBreakdown`/`_DurationOnly`: Both cases.
- `TestRenderTaskComplete`/`_NoTokens`: Both cases.
- `TestPhaseTracker_Accumulation`/`_Empty`: Accumulation logic and zero state.
- Existing tests updated with new signatures.

## Issues Found

### Issue 1: `classify.go` and `explore.go` - `EventLLMCallEnd` not merged with tool event forwarding case (Informational)

The design suggested adding `EventLLMCallEnd` to the existing forwarded event type list (`case schema.EventToolCallStart, schema.EventToolResult, schema.EventError, schema.EventLLMCallEnd:`). The implementation keeps it as a separate case because it also needs to accumulate local usage stats. This is the correct choice since merging the cases would lose the local accumulation logic. The implementation is better than the design's suggestion here.

### Issue 2: Missing `totalToolCalls` in task completion stats (By Design)

The `renderTaskComplete` stats struct does not include `totalToolCalls`. The design explicitly excludes tool calls from the task completion line, showing only duration and token totals. This matches the requirement.

### Issue 3: `phaseTracker` is not thread-safe (Informational, Non-Issue)

`phaseTracker` does not use mutexes, but this is correct because `send` functions in the streaming pipeline are always called sequentially from a single goroutine (the stream consumer loop). The `streamingDAGHandler` uses a mutex because DAG nodes execute concurrently, but `phaseTracker.wrap` wraps around `send` which is called from a single consumer.

## Suggestions Applied

No code changes were necessary. The implementation is clean and correct.
