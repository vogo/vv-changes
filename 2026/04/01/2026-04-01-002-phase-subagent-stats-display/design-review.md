# Design Review: Phase & Sub-Agent Stats Display

## Summary

The design is well-structured, correctly identifies the three layers of change, and makes sound architectural decisions. The code analysis below verifies each claim against the actual source and identifies issues and improvements.

## 1. Verified Assumptions

**Correct:**
- `PhaseEndData` has only `Phase`, `Duration`, `Summary` (event.go:183-188). No stats fields.
- `SubAgentEndData` has `AgentName`, `StepID`, `Duration`, `ToolCalls`, `TokensUsed` (event.go:203-209). No prompt/completion breakdown.
- `LLMCallEndData` has `PromptTokens`, `CompletionTokens`, `TotalTokens` (event.go:152-159).
- `aimodel.Usage` has `PromptTokens`, `CompletionTokens`, `TotalTokens` (schema.go:280-284).
- `exploreStream` accumulates `usage.PromptTokens` and `usage.CompletionTokens` but does NOT forward `EventLLMCallEnd` via `send` (explore.go:121-131). Only forwards `EventToolCallStart`, `EventToolResult`, `EventError`.
- `classifyStream` has the same pattern -- accumulates usage locally, does NOT forward `EventLLMCallEnd` (classify.go:138-154).
- `forwardSubAgentStream` tracks `toolCalls` and `tokensUsed` (total only) (stream.go:40-41, 67-69).
- `renderPhaseTransition` uses `dimStyle` for phase-end bullet, not `phaseBulletStyle` (render.go:211-212).
- `renderSubAgentEnd` uses combined `tokensUsed` parameter (render.go:175).
- `formatTokens` appends " tokens" suffix (render.go:299-307).
- CLI suppresses `EventLLMCallEnd` events (cli.go:569).
- CLI tracks `toolCallCount` per sub-agent and resets on `SubAgentStart`/`SubAgentEnd` (cli.go:528, 555).
- `DAGEventHandler` interface has only `OnNodeStart`, `OnNodeComplete`, `OnCheckpointError` -- no event stream access.
- `streamingDAGHandler.OnNodeComplete` emits `SubAgentEndData` with only `Duration` (stream.go:220-224).
- `renderPhaseTransition` takes `(phase string, starting bool, depth int)` -- 3 params (render.go:203).

## 2. Issues Found

### 2.1 DAG Path: No EventLLMCallEnd Events Reach the CLI

**Critical gap the design partially addresses but underestimates.** In the DAG path (`streamPlan`), nodes are executed by `ExecuteDAG` which calls `node.Runner.Run()` (non-streaming, dag.go:590-591 via `runWithTimeout`). The DAG engine uses `agent.Agent.Run()`, NOT `agent.StreamAgent.RunStream()`. This means:

- No streaming events are emitted from DAG sub-agents.
- No `EventLLMCallEnd`, `EventToolCallStart`, or `EventTextDelta` events flow through `send` during DAG execution.
- The CLI will NOT see any LLM call events between `SubAgentStart` and `SubAgentEnd` in the DAG path.

The design's "Option A" (track stats on CLI side from events between SubAgentStart/SubAgentEnd) **will not work for the DAG path** because there are no events between those boundaries. The DAG handler's `OnNodeComplete` is the only signal.

The design correctly notes `streamingDAGHandler.OnNodeComplete` only has `Duration`, but the proposed CLI-side fallback assumes events exist between boundaries.

**Fix:** For the DAG path, the `RunResponse.Usage` returned by each node is available in `dagExecutor.handleNodeSuccess` (dag.go:361-364). The `streamingDAGHandler` should capture stats from `DAGResult` after `ExecuteDAG` returns, OR the `OnNodeComplete` callback should receive the response. Since `OnNodeComplete` doesn't receive the response, a post-execution approach or interface extension is needed.

### 2.2 Explore Phase: EventToolCallStart Not Forwarded in Non-Streaming Fallback

The design says `exploreStream` forwards `EventToolCallStart` but the non-streaming fallback path (`explore()` at explore.go:19-52) does not emit any events. If the explorer agent is not a `StreamAgent`, no tool call events are forwarded. The `phaseSend` wrapper will see zero tool calls for the explore phase in this case. This is a minor edge case but worth noting.

### 2.3 Double-Counting Risk with Forwarding Decision

The design's final decision in Section 2.6 says: "Forward `EventLLMCallEnd` from explore/classify AND use the `phaseSend` wrapper as the single source of truth." This is internally consistent. However, the design also says in Section 2.3 approach (b): "After `exploreStream` returns, manually add `exploreUsage` to phase stats." These two approaches are mutually exclusive. The design resolves this in Section 2.6 by choosing forwarding, but Section 2.3's approach (b) text remains, creating confusion. The final design should cleanly state ONE approach.

### 2.4 renderPhaseTransition Signature Change Breaks Existing Call Sites

The design proposes changing `renderPhaseTransition` from `(phase string, starting bool, depth int)` to `(phase string, starting bool, durationMs int64, toolCalls, promptTokens, completionTokens int, depth int)`. This changes a 3-param function to 7 params. The existing call sites in `cli.go` (lines 488, 513) must all be updated. While the design acknowledges this in Section 2.5, passing 7 parameters is unwieldy. A struct parameter would be cleaner.

### 2.5 renderSubAgentEnd Format Inconsistency

The current `renderSubAgentEnd` uses "Done (stats)" format with `\n` + `"└ "` prefix (render.go:197). The design's proposed replacement in Section 2.4 changes to "Done " + `buildStatsLine(...)` but the requirement says `"sub-agent <name> complete. (stats)"`. These formats differ from each other and from the requirement.

### 2.6 Task Completion Should Not Show When No Stats Available

The design shows task completion rendering conditioned on `m.totalPromptTokens > 0 || m.totalCompletionTokens > 0`. But in the DAG path (per issue 2.1), tokens may be zero. The task line would be suppressed even though the task completed. Duration should be sufficient to show the line.

### 2.7 Tool Call Count for "1 tool use" vs "N tool uses"

The design uses `fmt.Sprintf("%d tool uses", toolCalls)` but "1 tool uses" is grammatically incorrect. Should use `pluralS` or equivalent.

### 2.8 classifyStream Does Forward EventToolCallStart

The design's Section 2.3 says "Also forward `EventToolCallStart` from `classifyStream` (it already forwards this)." This is correct (classify.go:143). So the plan phase `phaseSend` wrapper will capture tool calls from classify. No issue here, just confirming.

## 3. Simplification Opportunities

### 3.1 Use a Stats Struct Instead of Multiple Parameters

Instead of passing `toolCalls, durationMs, promptTokens, completionTokens` as separate parameters everywhere, define a reusable stats struct:

```go
type execStats struct {
    ToolCalls        int
    DurationMs       int64
    PromptTokens     int
    CompletionTokens int
}
```

Use it in render functions, `phaseStats`, CLI model fields, and the `buildStatsLine` function.

### 3.2 Unify Phase Stats Tracking in a Method

Rather than inline `phaseSend` closures for each phase, create a `phaseTracker` type with `wrap(send)` and `stats()` methods. This reduces code duplication across the three phases.

### 3.3 DAG Path Stats: Use DAGResult.Usage

After `ExecuteDAG` returns, `result.Usage` contains aggregated token usage across all nodes (dag.go:474-476). Rather than trying to intercept per-node events (which don't exist in the DAG path), the dispatch phase stats for DAG mode should come from `result.Usage` after completion. Per-node stats in `SubAgentEndData` can remain zero for now -- this is an acceptable tradeoff documented as a known limitation.

## 4. Recommendations

1. **Fix DAG path stats**: Accept that per-sub-agent token stats are unavailable in the DAG path. Use `DAGResult.Usage` for aggregate dispatch phase stats. Document this limitation.
2. **Choose one approach for explore/classify**: Forward `EventLLMCallEnd` (as decided in Section 2.6) and remove the "add usage after return" approach from the design text.
3. **Use a stats struct**: Replace multi-parameter render function signatures.
4. **Fix grammar**: Use `pluralS` for "tool use/uses".
5. **Show task completion unconditionally**: Condition on task having started, not on tokens being non-zero.
6. **Match the requirement format**: Ensure render functions produce output matching the spec exactly.
