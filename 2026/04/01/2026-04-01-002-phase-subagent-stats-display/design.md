# Design: Phase & Sub-Agent Stats Display

## 1. Architecture Overview

This change threads execution statistics (prompt tokens, completion tokens, tool call counts, duration) through three layers:

1. **Schema layer** (`vage/schema/event.go`) -- Extend `PhaseEndData` and `SubAgentEndData` to carry separate prompt/completion token counts.
2. **Dispatcher layer** (`vv/dispatches/`) -- Accumulate per-phase and per-sub-agent stats by observing `EventToolCallStart` and `EventLLMCallEnd` events flowing through the stream.
3. **CLI rendering layer** (`vv/cli/`) -- Display stats in the specified format at phase-end, sub-agent-end, and task-completion boundaries.

No changes to `aimodel/` are required. The existing `aimodel.Usage` struct already carries `PromptTokens`, `CompletionTokens`, and `TotalTokens`.

```
LLM call completes
  -> EventLLMCallEnd { PromptTokens, CompletionTokens }
     -> forwardSubAgentStream accumulates per-sub-agent
        -> SubAgentEndData { PromptTokens, CompletionTokens, ToolCalls, Duration }
     -> phaseSend wrapper accumulates per-phase
        -> PhaseEndData { PromptTokens, CompletionTokens, ToolCalls, Duration }
     -> CLI model accumulates per-task
        -> Task completion line { TotalPromptTokens, TotalCompletionTokens, TotalDuration }
```

### Known Limitation: DAG Path

In the DAG execution path (`streamPlan` -> `ExecuteDAG`), nodes are executed via `agent.Agent.Run()` (non-streaming). No `EventLLMCallEnd` or `EventToolCallStart` events are emitted from individual DAG sub-agents. Consequently:

- Per-sub-agent token stats in `SubAgentEndData` will be zero for DAG nodes.
- Aggregate dispatch phase stats for the DAG path are derived from `DAGResult.Usage` after execution completes.
- Per-sub-agent stats in the CLI will show duration and "Done" but not token counts for DAG nodes.

This is an acceptable tradeoff. Adding streaming support to the DAG engine is a larger change that can be addressed separately.

## 2. Shared Types

### 2.1 ExecStats Struct

A reusable struct for passing stats through render functions, reducing parameter sprawl:

```go
// execStats holds execution statistics for phases, sub-agents, and tasks.
type execStats struct {
    ToolCalls        int
    DurationMs       int64
    PromptTokens     int
    CompletionTokens int
}
```

This struct is used in render function signatures, `phaseTracker`, CLI model fields, and `buildStatsLine`.

## 3. Component Design

### 3.1 vage/schema/event.go -- Data Model Changes

**PhaseEndData**: Add three fields.

```go
type PhaseEndData struct {
    Phase            string `json:"phase"`
    Duration         int64  `json:"duration_ms"`
    Summary          string `json:"summary,omitempty"`
    ToolCalls        int    `json:"tool_calls,omitempty"`        // NEW
    PromptTokens     int    `json:"prompt_tokens,omitempty"`     // NEW
    CompletionTokens int    `json:"completion_tokens,omitempty"` // NEW
}
```

**SubAgentEndData**: Add `PromptTokens` and `CompletionTokens`. Keep `TokensUsed` for backward compatibility (HTTP clients may depend on it); it continues to hold `TotalTokens`.

```go
type SubAgentEndData struct {
    AgentName        string `json:"agent_name"`
    StepID           string `json:"step_id,omitempty"`
    Duration         int64  `json:"duration_ms"`
    ToolCalls        int    `json:"tool_calls"`
    TokensUsed       int    `json:"tokens_used"`                 // kept for compat
    PromptTokens     int    `json:"prompt_tokens,omitempty"`     // NEW
    CompletionTokens int    `json:"completion_tokens,omitempty"` // NEW
}
```

These are additive, JSON-compatible changes. Existing consumers that do not read the new fields are unaffected.

### 3.2 vv/dispatches/stream.go -- Sub-Agent Stats Accumulation

**forwardSubAgentStream**: Split `tokensUsed` tracking into `promptTokens` and `completionTokens`. Populate all fields in the emitted `SubAgentEndData`.

Current code (stream.go:40-41, 67-69):
```go
toolCalls := 0
tokensUsed := 0
// ...
case schema.EventLLMCallEnd:
    if data, ok := event.Data.(schema.LLMCallEndData); ok {
        tokensUsed += data.TotalTokens
    }
```

Changed to:
```go
toolCalls := 0
promptTokens := 0
completionTokens := 0
// ...
case schema.EventLLMCallEnd:
    if data, ok := event.Data.(schema.LLMCallEndData); ok {
        promptTokens += data.PromptTokens
        completionTokens += data.CompletionTokens
    }
```

And the final emit (stream.go:105-111) becomes:
```go
return send(schema.NewEvent(schema.EventSubAgentEnd, d.ID(), sessionID, schema.SubAgentEndData{
    AgentName:        agentName,
    StepID:           stepID,
    Duration:         time.Since(start).Milliseconds(),
    ToolCalls:        toolCalls,
    TokensUsed:       promptTokens + completionTokens,
    PromptTokens:     promptTokens,
    CompletionTokens: completionTokens,
}))
```

Non-streaming fallback path (stream.go:83-84): also populate the new fields from `resp.Usage`:
```go
if resp.Usage != nil {
    promptTokens = resp.Usage.PromptTokens
    completionTokens = resp.Usage.CompletionTokens
}
```

**streamingDAGHandler**: No changes. The DAG engine calls `agent.Agent.Run()` (not streaming), so no events flow through `send` during node execution. `OnNodeComplete` continues to emit `SubAgentEndData` with only `Duration` populated. The CLI will display "Done" with duration only for DAG sub-agents.

### 3.3 vv/dispatches/explore.go and classify.go -- Forward LLM Events

Both `exploreStream` and `classifyStream` accumulate `EventLLMCallEnd` data locally but do not forward it via `send`. To enable the `phaseSend` wrapper (Section 3.4) to capture all token usage, we forward `EventLLMCallEnd` from both functions.

**exploreStream** (explore.go:121): Add `EventLLMCallEnd` to the forwarded event types:
```go
case schema.EventToolCallStart, schema.EventToolResult, schema.EventError, schema.EventLLMCallEnd:
    if err := send(event); err != nil {
        slog.Warn("orchestrator: explorer stream send error", "error", err)
    }
```

The local usage accumulation remains for the return value but is no longer used for phase stats (the `phaseSend` wrapper captures it from the forwarded events).

**classifyStream** (classify.go:143): Same change:
```go
case schema.EventToolCallStart, schema.EventToolResult, schema.EventError, schema.EventLLMCallEnd:
    if err := send(event); err != nil {
        slog.Warn("orchestrator: planner stream send error", "error", err)
    }
```

This ensures the `phaseSend` wrapper is the single source of truth for per-phase stats. No double-counting occurs because the wrapper intercepts events before they reach the outer `send`, and we do NOT additionally add the returned `usage` to phase stats.

### 3.4 vv/dispatches/dispatch.go -- Phase Stats Accumulation

Introduce a `phaseTracker` helper to reduce code duplication across phases:

```go
// phaseTracker intercepts events to accumulate per-phase execution stats.
type phaseTracker struct {
    toolCalls        int
    promptTokens     int
    completionTokens int
}

// wrap returns a send function that intercepts stats events before forwarding.
func (pt *phaseTracker) wrap(send func(schema.Event) error) func(schema.Event) error {
    return func(ev schema.Event) error {
        switch ev.Type {
        case schema.EventToolCallStart:
            pt.toolCalls++
        case schema.EventLLMCallEnd:
            if data, ok := ev.Data.(schema.LLMCallEndData); ok {
                pt.promptTokens += data.PromptTokens
                pt.completionTokens += data.CompletionTokens
            }
        }
        return send(ev)
    }
}

// stats returns the accumulated stats as an execStats value.
func (pt *phaseTracker) stats(duration int64) execStats {
    return execStats{
        ToolCalls:        pt.toolCalls,
        DurationMs:       duration,
        PromptTokens:     pt.promptTokens,
        CompletionTokens: pt.completionTokens,
    }
}
```

Use in `RunStream` for each phase:

```go
// Explore phase
var tracker phaseTracker
phaseSend := tracker.wrap(send)
contextSummary, exploreUsage = d.exploreStream(ctx, req, phaseSend)

exploreStats := tracker.stats(time.Since(exploreStart).Milliseconds())
if err := send(schema.NewEvent(schema.EventPhaseEnd, agentID, sessionID, schema.PhaseEndData{
    Phase:            "explore",
    Duration:         exploreStats.DurationMs,
    ToolCalls:        exploreStats.ToolCalls,
    PromptTokens:     exploreStats.PromptTokens,
    CompletionTokens: exploreStats.CompletionTokens,
})); err != nil {
    return err
}
```

Same pattern for the plan and dispatch phases. Each phase gets its own `phaseTracker` instance.

**Dispatch phase -- DAG path special case**: In the DAG path (`streamPlan`), no `EventLLMCallEnd` events flow through `send`. After `ExecuteDAG` returns, `result.Usage` contains aggregated token usage. Augment the dispatch phase stats from the DAG result:

```go
case "plan":
    dispatchErr = d.streamPlan(ctx, phaseSend, enrichedReq, result.Plan, contextSummary, sessionID)
```

Modify `streamPlan` to return `*aimodel.Usage` from the DAG result:

```go
func (d *Dispatcher) streamPlan(...) (*aimodel.Usage, error) {
    // ... existing code ...
    result, err := orchestrate.ExecuteDAG(ctx, dagCfg, nodes, req)
    // ... existing code ...
    return result.Usage, nil  // or (result.Usage, dispatchErr)
}
```

Then in `RunStream`, after the dispatch phase:

```go
if dagUsage != nil {
    tracker.promptTokens += dagUsage.PromptTokens
    tracker.completionTokens += dagUsage.CompletionTokens
}
dispatchStats := tracker.stats(time.Since(dispatchStart).Milliseconds())
```

This ensures the dispatch phase's `PhaseEndData` has aggregate token counts even for the DAG path.

### 3.5 vv/cli/render.go -- Display Formatting

**New helper -- `formatCompactTokens`**: The existing `formatTokens` appends " tokens" suffix. The new spec uses bare numbers with arrow prefixes. Add a new function:

```go
// formatCompactTokens formats a token count without suffix.
func formatCompactTokens(tokens int) string {
    if tokens >= 1000000 {
        return fmt.Sprintf("%.1fM", float64(tokens)/1000000)
    }
    if tokens >= 1000 {
        return fmt.Sprintf("%.1fk", float64(tokens)/1000)
    }
    return fmt.Sprintf("%d", tokens)
}
```

**New helper -- `buildStatsLine`**: Builds the parenthetical stats string shared by phase, sub-agent, and task completions:

```go
// buildStatsLine builds a parenthetical stats string like "(3 tool uses . 26s . up 5.3k . down 10.5k)".
func buildStatsLine(s execStats) string {
    var parts []string
    if s.ToolCalls > 0 {
        noun := "tool use"
        if s.ToolCalls != 1 {
            noun = "tool uses"
        }
        parts = append(parts, fmt.Sprintf("%d %s", s.ToolCalls, noun))
    }
    parts = append(parts, formatDuration(s.DurationMs))
    if s.PromptTokens > 0 {
        parts = append(parts, fmt.Sprintf("\u2191 %s", formatCompactTokens(s.PromptTokens)))
    }
    if s.CompletionTokens > 0 {
        parts = append(parts, fmt.Sprintf("\u2193 %s", formatCompactTokens(s.CompletionTokens)))
    }
    return "(" + strings.Join(parts, " \u00b7 ") + ")"
}
```

**Modify `renderPhaseTransition`**: Change signature to accept `execStats`. When `starting` is false, append the stats line. Use `phaseBulletStyle` (yellow) for the phase-end bullet instead of `dimStyle` (matching the requirement spec).

```go
func renderPhaseTransition(phase string, starting bool, stats execStats, depth int) string {
    var sb strings.Builder
    phaseName := strings.ToUpper(phase[:1]) + phase[1:]

    if starting {
        sb.WriteString(phaseBulletStyle.Render(bullet))
        sb.WriteString(phaseStyle.Render(phaseName))
    } else {
        sb.WriteString(phaseBulletStyle.Render(bullet))
        sb.WriteString(fmt.Sprintf("phase %s complete.  ", phaseName))
        sb.WriteString(statsStyle.Render(buildStatsLine(stats)))
    }

    return indentBlock(sb.String(), depth)
}
```

**Modify `renderSubAgentEnd`**: Replace the combined `tokensUsed` parameter with `execStats`. Match the requirement format: `sub-agent <name> complete. (stats)`.

```go
func renderSubAgentEnd(agentName, stepID string, stats execStats, depth int) string {
    var sb strings.Builder
    sb.WriteString(subAgentBulletStyle.Render(bullet))
    sb.WriteString(fmt.Sprintf("sub-agent "))
    sb.WriteString(subAgentStyle.Render(agentName))

    if stepID != "" {
        sb.WriteString(dimStyle.Render(fmt.Sprintf(" (%s)", stepID)))
    }

    sb.WriteString(" complete.  ")
    sb.WriteString(statsStyle.Render(buildStatsLine(stats)))

    return indentBlock(sb.String(), depth)
}
```

**New function -- `renderTaskComplete`**: Renders the task-level completion line.

```go
func renderTaskComplete(stats execStats) string {
    var sb strings.Builder
    sb.WriteString(phaseBulletStyle.Render(bullet))
    sb.WriteString("task complete.  ")
    sb.WriteString(statsStyle.Render(buildStatsLine(stats)))
    return sb.String()
}
```

### 3.6 vv/cli/cli.go -- Event Handling and Task-Level Aggregation

**Model struct additions**: Add fields for task-level accumulation and per-sub-agent tracking.

```go
type model struct {
    // ... existing fields ...

    // Task-level stats accumulation.
    taskStart              time.Time
    totalPromptTokens      int
    totalCompletionTokens  int
    totalToolCalls         int

    // Sub-agent level stats accumulation (for DAG path where SubAgentEndData
    // may lack token stats).
    subAgentPromptTokens     int
    subAgentCompletionTokens int
}
```

**handleStreamEvent changes**:

1. **EventLLMCallEnd** (currently suppressed): Still suppress display, but accumulate tokens at both task and sub-agent level.

```go
case schema.EventLLMCallEnd:
    if data, ok := event.Data.(schema.LLMCallEndData); ok {
        m.totalPromptTokens += data.PromptTokens
        m.totalCompletionTokens += data.CompletionTokens
        m.subAgentPromptTokens += data.PromptTokens
        m.subAgentCompletionTokens += data.CompletionTokens
    }
```

2. **EventToolCallStart**: Also increment `totalToolCalls`.

```go
case schema.EventToolCallStart:
    // ... existing code ...
    m.toolCallCount++
    m.totalToolCalls++
```

3. **EventPhaseEnd**: Pass stats to `renderPhaseTransition`.

```go
case schema.EventPhaseEnd:
    if data, ok := event.Data.(schema.PhaseEndData); ok {
        // ... existing summary rendering ...
        stats := execStats{
            ToolCalls:        data.ToolCalls,
            DurationMs:       data.Duration,
            PromptTokens:     data.PromptTokens,
            CompletionTokens: data.CompletionTokens,
        }
        rendered := renderPhaseTransition(data.Phase, false, stats, 1)
        // ...
    }
```

4. **EventPhaseStart**: Pass zero stats (starting=true ignores them).

```go
rendered := renderPhaseTransition(data.Phase, true, execStats{}, 0)
```

5. **EventSubAgentStart**: Reset sub-agent token accumulators.

```go
case schema.EventSubAgentStart:
    m.nestingDepth++
    m.toolCallCount = 0
    m.subAgentPromptTokens = 0
    m.subAgentCompletionTokens = 0
```

6. **EventSubAgentEnd**: Use new fields when available, fall back to CLI-tracked stats for the DAG path.

```go
case schema.EventSubAgentEnd:
    if data, ok := event.Data.(schema.SubAgentEndData); ok {
        // ...
        toolCalls := data.ToolCalls
        if toolCalls == 0 {
            toolCalls = m.toolCallCount
        }
        promptTokens := data.PromptTokens
        if promptTokens == 0 {
            promptTokens = m.subAgentPromptTokens
        }
        completionTokens := data.CompletionTokens
        if completionTokens == 0 {
            completionTokens = m.subAgentCompletionTokens
        }
        stats := execStats{
            ToolCalls:        toolCalls,
            DurationMs:       data.Duration,
            PromptTokens:     promptTokens,
            CompletionTokens: completionTokens,
        }
        rendered := renderSubAgentEnd(data.AgentName, data.StepID, stats, m.nestingDepth)
        // ...
        m.toolCallCount = 0
        m.subAgentPromptTokens = 0
        m.subAgentCompletionTokens = 0
    }
```

**handleSubmit change**: Record `taskStart` and reset accumulators.

```go
func (m *model) handleSubmit() (tea.Model, tea.Cmd) {
    // ... existing code ...
    m.taskStart = time.Now()
    m.totalPromptTokens = 0
    m.totalCompletionTokens = 0
    m.totalToolCalls = 0
    // ...
}
```

**handleStreamDone change**: Always render task completion stats when a task was started (not conditional on token counts being non-zero, since DAG paths may have zero CLI-visible tokens).

```go
func (m *model) handleStreamDone(msg streamDoneMsg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    if msg.err != nil {
        cmds = append(cmds, m.printError(msg.err.Error()))
    }

    // ... existing flush logic ...

    // Render task completion line.
    if !m.taskStart.IsZero() {
        taskDuration := time.Since(m.taskStart).Milliseconds()
        stats := execStats{
            DurationMs:       taskDuration,
            PromptTokens:     m.totalPromptTokens,
            CompletionTokens: m.totalCompletionTokens,
        }
        rendered := renderTaskComplete(stats)
        cmds = append(cmds, tea.Println(rendered))
    }

    // ... existing reset logic ...
}
```

## 4. Data Model Changes Summary

### vage/schema/event.go

| Struct | Field | Type | Change |
|--------|-------|------|--------|
| `PhaseEndData` | `ToolCalls` | `int` | **Added** |
| `PhaseEndData` | `PromptTokens` | `int` | **Added** |
| `PhaseEndData` | `CompletionTokens` | `int` | **Added** |
| `SubAgentEndData` | `PromptTokens` | `int` | **Added** |
| `SubAgentEndData` | `CompletionTokens` | `int` | **Added** |
| `SubAgentEndData` | `TokensUsed` | `int` | Kept for backward compat |

## 5. Implementation Plan

### Task 1: Extend event data structs (`vage/schema/event.go`)

Add `ToolCalls`, `PromptTokens`, `CompletionTokens` to `PhaseEndData`. Add `PromptTokens`, `CompletionTokens` to `SubAgentEndData`.

Files: `vage/schema/event.go`

### Task 2: Forward LLM events from explore and classify streams

Modify `exploreStream` and `classifyStream` to forward `EventLLMCallEnd` via `send` by adding it to the existing forwarded event type list.

Files: `vv/dispatches/explore.go`, `vv/dispatches/classify.go`

### Task 3: Add phaseTracker and accumulate per-phase stats in RunStream (`vv/dispatches/dispatch.go`)

Introduce `phaseTracker` type with `wrap()` and `stats()` methods. Use it in each phase of `RunStream` to intercept `EventToolCallStart` and `EventLLMCallEnd`. Populate new fields in `PhaseEndData`. For the DAG dispatch path, augment stats from `DAGResult.Usage` (requires `streamPlan` to return `*aimodel.Usage`).

Files: `vv/dispatches/dispatch.go`, `vv/dispatches/stream.go` (streamPlan return type)

### Task 4: Track per-sub-agent prompt/completion tokens (`vv/dispatches/stream.go`)

In `forwardSubAgentStream`, replace `tokensUsed` with `promptTokens`/`completionTokens`. Populate both the new fields and `TokensUsed` (as their sum) in `SubAgentEndData`. Also handle the non-streaming fallback path.

Files: `vv/dispatches/stream.go`

### Task 5: Add execStats type and rendering helpers, update render functions (`vv/cli/render.go`)

Add `execStats` struct, `formatCompactTokens`, `buildStatsLine`, `renderTaskComplete`. Update `renderPhaseTransition` signature to use `execStats`. Update `renderSubAgentEnd` to use `execStats` and match the requirement's `"sub-agent <name> complete."` format.

Files: `vv/cli/render.go`

### Task 6: Update CLI event handling and task-level aggregation (`vv/cli/cli.go`)

Add task-level and sub-agent-level accumulator fields to `model`. Update `handleStreamEvent` for `EventLLMCallEnd`, `EventToolCallStart`, `EventPhaseEnd`, `EventPhaseStart`, `EventSubAgentStart`, `EventSubAgentEnd`. Update `handleSubmit` to reset accumulators. Update `handleStreamDone` to render task completion line unconditionally when `taskStart` is set.

Files: `vv/cli/cli.go`

### Task 7: Unit tests for formatting helpers (`vv/cli/render_test.go`)

Add tests for `formatCompactTokens`, `buildStatsLine`, `formatDuration` edge cases, `renderTaskComplete`, updated `renderPhaseTransition`, updated `renderSubAgentEnd`. Update existing tests that call `renderPhaseTransition` and `renderSubAgentEnd` with the new signatures.

Files: `vv/cli/render_test.go`

## 6. Test Plan

### 6.1 Unit Tests (no LLM keys required)

**File: `vv/cli/render_test.go`**

| Test | Validates |
|------|-----------|
| `TestFormatCompactTokens` | 0 -> "0", 999 -> "999", 1000 -> "1.0k", 5300 -> "5.3k", 1200000 -> "1.2M" |
| `TestBuildStatsLine` | Zero tool calls omitted; all parts joined with " . "; zero tokens omitted; singular "1 tool use" |
| `TestBuildStatsLine_DurationOnly` | When only duration is non-zero, produces "(Ns)" |
| `TestRenderPhaseTransition_End` | Contains "phase Dispatch complete." and stats parenthetical |
| `TestRenderPhaseTransition_Start` | Starting mode ignores stats, shows phase name |
| `TestRenderSubAgentEnd_WithTokenBreakdown` | Shows "sub-agent <name> complete." with separate up/down token counts |
| `TestRenderSubAgentEnd_DurationOnly` | No tokens, shows "(Ns)" only |
| `TestRenderTaskComplete` | Shows "task complete." with duration and token stats |
| `TestRenderTaskComplete_NoTokens` | Shows "task complete." with duration only |

**File: `vv/dispatches/dispatch_test.go`** (or new file)

| Test | Validates |
|------|-----------|
| `TestPhaseTracker_Accumulation` | `phaseTracker.wrap` correctly counts tool calls and tokens from mock events |
| `TestPhaseTracker_Empty` | Zero stats when no events are intercepted |

### 6.2 Integration Tests (require LLM API keys)

**File: `vv/integrations/stats_test.go`**

| Test | Validates |
|------|-----------|
| `TestStreamEvents_PhaseEndContainsStats` | Run a Dispatcher.RunStream with a simple request. Collect all events. Verify `PhaseEndData` events have non-zero `PromptTokens` and `CompletionTokens`. |
| `TestStreamEvents_SubAgentEndContainsTokenBreakdown` | Verify `SubAgentEndData` events have non-zero `PromptTokens` and `CompletionTokens` (and `TokensUsed == PromptTokens + CompletionTokens`). |
| `TestStreamEvents_TaskStatsAccumulation` | Sum all `PromptTokens` from `PhaseEndData` events; verify they are consistent with summing `LLMCallEndData.PromptTokens` from all `EventLLMCallEnd` events in the stream. |

### 6.3 Existing Test Updates

The following existing tests call `renderPhaseTransition` or `renderSubAgentEnd` with the old signatures and must be updated:

- `TestRenderPhaseTransition_StartNoIndent` (render_test.go:161) -- add `execStats{}` parameter
- `TestRenderPhaseTransition_EndIndented` (render_test.go:167) -- add `execStats{DurationMs: 1000}` parameter
- `TestRenderSubAgentEnd_Indented` (render_test.go:150) -- change from `(name, stepID, duration, toolCalls, tokensUsed, depth)` to `(name, stepID, execStats{...}, depth)`

### 6.4 Manual Testing Checklist

- [ ] Run `vv` CLI with a simple question (direct mode) -- verify phase stats and task stats display
- [ ] Run `vv` CLI with a complex request (plan mode) -- verify sub-agent stats show token breakdown where available, duration-only for DAG nodes
- [ ] Verify zero tool calls are omitted from stats lines
- [ ] Verify "1 tool use" (singular) vs "N tool uses" (plural)
- [ ] Verify token formatting: numbers below 1000 show raw, above show "Xk" / "XM"
- [ ] Verify duration formatting: <1s shows ms, <60s shows seconds, >=60s shows "Xm Ys"
- [ ] Verify task completion line appears even when no tokens are tracked (DAG-only path)
- [ ] Run HTTP/SSE mode and inspect event JSON -- verify new fields present in PhaseEndData and SubAgentEndData
