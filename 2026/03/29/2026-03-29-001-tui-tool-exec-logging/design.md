# Design: TUI Tool Execution Logging for Explore and Plan Phases

## Architecture Overview

The change is entirely within the orchestrator layer (`vv/agents/orchestrator.go`). The core idea is to replace the non-streaming `Run()` calls in the Explore and Plan phases with streaming `RunStream()` calls, forwarding tool events through the `send` callback while still capturing the final text output for downstream phases.

No changes are needed in the TUI layer (`vv/cli/`), the agent framework (`vage/`), or the event schema. The existing `handleStreamEvent` method in `cli.go` already handles `EventToolCallStart`, `EventToolResult`, and all other event types correctly.

### Current Flow (Broken)

```
RunStream() producer goroutine
  |
  +-- Phase 1: o.explore(ctx, req)
  |     calls explorerAgent.Run()          <-- non-streaming, tool events lost
  |     returns (contextSummary, usage)
  |
  +-- Phase 2: o.planTask(ctx, req, ctx)
  |     calls plannerAgent.Run()           <-- non-streaming, tool events lost
  |     returns (ClassifyResult, usage)
  |
  +-- Phase 3: o.forwardSubAgentStream()   <-- streaming, tool events forwarded
```

### Proposed Flow (Fixed)

```
RunStream() producer goroutine
  |
  +-- Phase 1: o.exploreStream(ctx, req, send)
  |     calls explorerAgent.RunStream()    <-- streaming, tool events forwarded via send
  |     collects TextDelta events into contextSummary string
  |     returns (contextSummary, usage)
  |
  +-- Phase 2: o.planTaskStream(ctx, req, contextSummary, send)
  |     calls plannerAgent.RunStream()     <-- streaming, tool events forwarded via send
  |     collects TextDelta events into JSON text
  |     returns (ClassifyResult, usage)
  |
  +-- Phase 3: o.forwardSubAgentStream()   <-- unchanged
```

## Component Design

### Modified File: `vv/agents/orchestrator.go`

Two new private methods are introduced. The existing `explore()` and `planTask()` methods are kept unchanged for use by the non-streaming `Run()` path.

#### 1. `exploreStream` method

```go
func (o *OrchestratorAgent) exploreStream(
    ctx context.Context,
    req *schema.RunRequest,
    send func(schema.Event) error,
) (string, *aimodel.Usage)
```

**Behavior:**
- Returns early with `("", nil)` if `o.explorerAgent == nil`.
- Builds the same explorer request as `explore()` (prepending working directory).
- Casts `o.explorerAgent` to `agent.StreamAgent` (safe: `explorerAgent` is a `*taskagent.Agent` which implements `StreamAgent`).
- Calls `RunStream()` on the explorer agent.
- Consumes all events from the stream in a loop:
  - `EventTextDelta`: accumulates `data.Delta` into a `strings.Builder` to capture the context summary.
  - `EventToolCallStart`, `EventToolResult`: forwards to `send()` so they appear in the TUI.
  - `EventLLMCallEnd`: extracts `TotalTokens` to build usage.
  - `EventAgentEnd`: extracts the final message if present (fallback for text capture).
  - All other events (`EventAgentStart`, `EventIterationStart`, `EventToolCallEnd`, `EventLLMCallStart`, `EventLLMCallError`): suppressed (not forwarded), consistent with how the TUI already suppresses these in `handleStreamEvent`.
- On stream error, logs warning and returns whatever was collected.
- Returns `(contextSummary, usage)` -- same signature as `explore()`.

#### 2. `planTaskStream` method

```go
func (o *OrchestratorAgent) planTaskStream(
    ctx context.Context,
    req *schema.RunRequest,
    contextSummary string,
    send func(schema.Event) error,
) (*ClassifyResult, *aimodel.Usage, error)
```

**Behavior:**
- Falls back to `o.classifyTaskDirect(ctx, req)` if `o.plannerAgent == nil` (same as `planTask`).
- Builds the same planner request as `planTask()`.
- Casts `o.plannerAgent` to `agent.StreamAgent`.
- Calls `RunStream()` on the planner agent.
- Consumes all events:
  - `EventTextDelta`: accumulates delta into a `strings.Builder` to capture the JSON classification output.
  - `EventToolCallStart`, `EventToolResult`: forwards to `send()` (future-proofing; planner currently has no tools).
  - `EventLLMCallEnd`: extracts usage tokens.
  - Other events: suppressed.
- After stream completes, parses the collected text as JSON into `ClassifyResult` (same logic as existing `planTask`).
- Returns `(*ClassifyResult, *aimodel.Usage, error)` -- same signature as `planTask()`.

#### 3. Modified `RunStream` method

The `RunStream` producer function body is updated to call the new streaming methods instead of the non-streaming ones:

```
// Before:
contextSummary, exploreUsage = o.explore(ctx, req)

// After:
contextSummary, exploreUsage = o.exploreStream(ctx, req, send)
```

```
// Before:
result, planUsage, planErr := o.planTask(ctx, req, contextSummary)

// After:
result, planUsage, planErr = o.planTaskStream(ctx, req, contextSummary, send)
```

Only these two call sites change within `RunStream`. Everything else (phase events, dispatch logic) remains identical.

#### 4. Unchanged: `explore()` and `planTask()` methods

The original non-streaming methods are preserved for the `Run()` code path (non-streaming / HTTP API). The `Run()` method continues to use `explore()` and `planTask()` as before.

### Event Forwarding Strategy

Not all events from sub-agent streams should be forwarded. The strategy mirrors what the TUI already suppresses:

| Event Type | Forward? | Reason |
|---|---|---|
| `EventToolCallStart` | Yes | Primary goal of this change |
| `EventToolResult` | Yes | Shows tool outcomes |
| `EventTextDelta` | No (capture only) | Text is the context summary / plan JSON, not user-facing output |
| `EventAgentStart` | No | Suppressed in TUI handler |
| `EventAgentEnd` | No | Suppressed in TUI handler |
| `EventIterationStart` | No | Suppressed in TUI handler |
| `EventToolCallEnd` | No | Suppressed in TUI handler (result event is richer) |
| `EventLLMCallStart` | No | Suppressed in TUI handler |
| `EventLLMCallEnd` | No (capture usage) | Suppressed in TUI handler |
| `EventLLMCallError` | No | Suppressed in TUI handler |
| `EventError` | Yes | Surface errors to user |

## Data Models / Schemas

No changes to data models. All existing types are sufficient:
- `schema.Event`, `schema.EventData` subtypes
- `schema.RunStream`
- `aimodel.Usage`

## API Contracts

No API changes. The `RunStream()` return type and event contract remain the same. The only difference is that more events (tool calls from explore/plan phases) will now appear in the stream, but they use the same event types the TUI already handles.

## Implementation Plan

### Task 1: Add `exploreStream` method

**File:** `vv/agents/orchestrator.go`

Add the `exploreStream` method after the existing `explore` method (~line 549). The method:
1. Checks `o.explorerAgent == nil`, returns early if so.
2. Builds explorer request (same as `explore`).
3. Type-asserts explorer to `agent.StreamAgent`. If it does not implement `StreamAgent`, falls back to calling `o.explore()` (defensive, but all `taskagent.Agent` instances implement it).
4. Calls `RunStream()`, iterates events, forwards tool events via `send`, accumulates text deltas, tracks usage.
5. Returns `(contextSummary, usage)`.

### Task 2: Add `planTaskStream` method

**File:** `vv/agents/orchestrator.go`

Add the `planTaskStream` method after the existing `planTask` method (~line 600). The method:
1. Checks `o.plannerAgent == nil`, falls back to `classifyTaskDirect` if so.
2. Builds planner request (same as `planTask`).
3. Type-asserts planner to `agent.StreamAgent`. Falls back to `o.planTask()` if not streamable.
4. Calls `RunStream()`, iterates events, forwards tool events via `send`, accumulates text deltas, tracks usage.
5. Parses accumulated text into `ClassifyResult` (same JSON extraction + validation as `planTask`).
6. Returns `(*ClassifyResult, *aimodel.Usage, error)`.

### Task 3: Update `RunStream` to use streaming methods

**File:** `vv/agents/orchestrator.go`

In the `RunStream` method's producer function:
1. Replace `o.explore(ctx, req)` call (line 248) with `o.exploreStream(ctx, req, send)`.
2. Replace `o.planTask(ctx, req, contextSummary)` call (line 264) with `o.planTaskStream(ctx, req, contextSummary, send)`.

This is a two-line change in the `RunStream` method body.

### Task 4: Add unit tests

**File:** `vv/agents/orchestrator_test.go`

Add test cases:

1. **`TestOrchestratorAgent_RunStream_ExploreToolEvents`**: Creates an orchestrator with a stub streaming explorer agent that emits `ToolCallStart` and `ToolResult` events. Verifies that these events appear in the orchestrator's `RunStream` output between the `PhaseStart("explore")` and `PhaseEnd("explore")` events.

2. **`TestOrchestratorAgent_RunStream_ExploreTextCapture`**: Verifies that text deltas from the explorer stream are correctly accumulated and used as context summary (passed to the enriched request for the dispatch phase).

3. **`TestOrchestratorAgent_RunStream_PlanToolEvents`**: Similar to test 1 but for the plan phase. Uses a stub streaming planner agent that emits tool events. Verifies forwarding. (Future-proofing test.)

4. **`TestOrchestratorAgent_RunStream_NonStreamExplorerFallback`**: Verifies that if the explorer agent does NOT implement `StreamAgent`, the orchestrator gracefully falls back to the non-streaming `explore()` path without crashing.

### Task Ordering

```
Task 1 (exploreStream) ──┐
                          ├── Task 3 (wire into RunStream) ── Task 4 (tests)
Task 2 (planTaskStream) ──┘
```

Tasks 1 and 2 are independent. Task 3 depends on both. Task 4 depends on Task 3.

## Integration Test Plan

- [ ] **Explore phase tool events visible**: Run the TUI with a real LLM and working directory. Submit a coding question. Verify that tool calls (read, glob, grep) appear in the TUI output between "Phase 1/3: Explore" and "Explore phase complete." markers.
- [ ] **Explore result correctness**: Verify that after the explore phase completes, the dispatch phase still receives correct project context (i.e., the text accumulation from `exploreStream` produces the same context summary as the old `explore` path).
- [ ] **Plan phase passthrough**: Submit a complex multi-step task. Verify that the plan phase completes correctly and the dispatch phase receives a valid `ClassifyResult`. The planner currently has no tools, so no tool events are expected, but the phase should not error.
- [ ] **Dispatch phase unchanged**: Verify that tool events during the dispatch phase continue to display correctly (no regression).
- [ ] **Non-streaming Run() path unchanged**: Verify that the HTTP API / non-streaming `Run()` path still works correctly (it uses the original `explore()` and `planTask()` methods).
- [ ] **Cancellation during explore**: Press Ctrl+C during the explore phase. Verify that the stream is cancelled cleanly and the TUI returns to idle state.
- [ ] **Explorer disabled**: Configure without an explorer agent. Verify that the explore phase is skipped (2 phases total) and the plan + dispatch phases work normally.
- [ ] **Error handling**: If the explorer agent's stream returns an error mid-stream, verify that the orchestrator logs the warning and continues with whatever context was gathered (same resilience as the current `explore()` method).
