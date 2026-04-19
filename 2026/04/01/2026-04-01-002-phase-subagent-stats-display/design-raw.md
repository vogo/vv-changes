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
     -> RunStream phase loop accumulates per-phase
        -> PhaseEndData { PromptTokens, CompletionTokens, ToolCalls, Duration }
     -> CLI model accumulates per-task
        -> Task completion line { TotalPromptTokens, TotalCompletionTokens, TotalDuration }
```

## 2. Component Design

### 2.1 vage/schema/event.go -- Data Model Changes

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

### 2.2 vv/dispatches/stream.go -- Sub-Agent Stats Accumulation

**forwardSubAgentStream**: Split `tokensUsed` tracking into `promptTokens` and `completionTokens`. Populate all three fields in the emitted `SubAgentEndData`.

Current code (lines 40-41, 68-70):
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

And the final emit (lines 105-111) becomes:
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

Non-streaming fallback path (line 84-85): also populate the new fields from `resp.Usage`.

**streamingDAGHandler**: The DAG handler does not currently have access to per-node event streams (it only receives `OnNodeStart`/`OnNodeComplete` callbacks). Two options:

- **Option A (simple)**: Track stats on the CLI side instead. The CLI model already counts `toolCallCount` per sub-agent. Extend it to also count prompt/completion tokens from `EventLLMCallEnd` events received between `SubAgentStart` and `SubAgentEnd`. The `streamingDAGHandler.OnNodeComplete` would continue to emit `SubAgentEndData` with only Duration, and the CLI fills in the rest.
- **Option B (complete)**: Add tracking maps to `streamingDAGHandler` (`toolCalls map[string]int`, `promptTokens map[string]int`, `completionTokens map[string]int`). Populate them by intercepting events before they are sent. The DAG handler wraps the `send` function to observe events tagged with the node's agent ID.

**Decision**: Option A is simpler and sufficient. The DAG path's `OnNodeComplete` callback has no access to the event stream. Modifying the DAG engine to forward events through the handler would be a larger change. The CLI already tracks `toolCallCount` between sub-agent boundaries; we extend this pattern to tokens.

For the non-DAG path (`forwardSubAgentStream`), the stats are already available because it processes the event stream directly.

### 2.3 vv/dispatches/dispatch.go -- Phase Stats Accumulation

The `RunStream` method needs to accumulate stats per phase. Introduce a `phaseStats` helper struct:

```go
type phaseStats struct {
    toolCalls        int
    promptTokens     int
    completionTokens int
}
```

Wrap the `send` function inside each phase block to intercept events and accumulate counters:

```go
// Inside RunStream, for each phase:
var stats phaseStats
phaseSend := func(ev schema.Event) error {
    switch ev.Type {
    case schema.EventToolCallStart:
        stats.toolCalls++
    case schema.EventLLMCallEnd:
        if data, ok := ev.Data.(schema.LLMCallEndData); ok {
            stats.promptTokens += data.PromptTokens
            stats.completionTokens += data.CompletionTokens
        }
    }
    return send(ev)
}
```

Then pass `phaseSend` instead of `send` to `exploreStream`, `classifyStream`, and `forwardSubAgentStream`/`streamPlan`. When emitting `PhaseEndData`, populate the new fields:

```go
send(schema.NewEvent(schema.EventPhaseEnd, agentID, sessionID, schema.PhaseEndData{
    Phase:            "explore",
    Duration:         time.Since(exploreStart).Milliseconds(),
    ToolCalls:        stats.toolCalls,
    PromptTokens:     stats.promptTokens,
    CompletionTokens: stats.completionTokens,
}))
```

**Explore phase special case**: `exploreStream` already accumulates `usage.PromptTokens` and `usage.CompletionTokens` internally, but does NOT forward `EventLLMCallEnd` events via `send`. It only forwards `EventToolCallStart`, `EventToolResult`, and `EventError`. So the `phaseSend` wrapper would capture tool call starts but not LLM call ends from the explore phase.

Two approaches:
- (a) Modify `exploreStream` to also forward `EventLLMCallEnd` events via `send`. The CLI already suppresses these events in its handler, so forwarding them is safe.
- (b) After `exploreStream` returns, manually add `exploreUsage` to the phase stats.

**Decision**: Approach (b) is simpler and avoids changing the explore stream contract. After `exploreStream` returns:

```go
if exploreUsage != nil {
    stats.promptTokens += exploreUsage.PromptTokens
    stats.completionTokens += exploreUsage.CompletionTokens
}
```

**Classify phase**: Same situation -- `classifyStream` does not forward `EventLLMCallEnd` to `send`. It returns `*aimodel.Usage` directly. Use approach (b) again: add `planUsage` to the plan phase stats after `classifyStream` returns.

**Dispatch phase**: `forwardSubAgentStream` DOES forward all events via `send`, including `EventLLMCallEnd`. So the `phaseSend` wrapper will naturally capture dispatch phase stats.

### 2.4 vv/cli/render.go -- Display Formatting

**New helper -- `formatCompactTokens`**: The existing `formatTokens` appends " tokens" suffix (e.g., `"5.3k tokens"`). The new spec uses bare numbers with arrow prefixes (`"5.3k"`). Add a new function:

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
func buildStatsLine(toolCalls int, durationMs int64, promptTokens, completionTokens int) string {
    var parts []string
    if toolCalls > 0 {
        parts = append(parts, fmt.Sprintf("%d tool uses", toolCalls))
    }
    parts = append(parts, formatDuration(durationMs))
    if promptTokens > 0 {
        parts = append(parts, fmt.Sprintf("\u2191 %s", formatCompactTokens(promptTokens)))
    }
    if completionTokens > 0 {
        parts = append(parts, fmt.Sprintf("\u2193 %s", formatCompactTokens(completionTokens)))
    }
    return "(" + strings.Join(parts, " \u00b7 ") + ")"
}
```

**Modify `renderPhaseTransition`**: Change signature to accept stats fields. When `starting` is false, append the stats line.

```go
func renderPhaseTransition(phase string, starting bool, durationMs int64, toolCalls, promptTokens, completionTokens int, depth int) string {
    var sb strings.Builder
    phaseName := strings.ToUpper(phase[:1]) + phase[1:]

    if starting {
        sb.WriteString(phaseBulletStyle.Render(bullet))
        sb.WriteString(phaseStyle.Render(phaseName))
    } else {
        sb.WriteString(phaseBulletStyle.Render(bullet))
        sb.WriteString(fmt.Sprintf("phase %s complete.  ", phaseName))
        sb.WriteString(statsStyle.Render(buildStatsLine(toolCalls, durationMs, promptTokens, completionTokens)))
    }

    return indentBlock(sb.String(), depth)
}
```

Note: Phase-end rendering changes from `dimStyle` bullet to `phaseBulletStyle` bullet (yellow), matching the spec requirement that the bullet uses `phaseBulletStyle`.

**Modify `renderSubAgentEnd`**: Replace the combined `tokensUsed` parameter with `promptTokens, completionTokens`. Use `buildStatsLine`.

```go
func renderSubAgentEnd(agentName, stepID string, durationMs int64, toolCalls, promptTokens, completionTokens int, depth int) string {
    var sb strings.Builder
    sb.WriteString(subAgentBulletStyle.Render(bullet))
    sb.WriteString(subAgentStyle.Render(agentName))

    if stepID != "" {
        sb.WriteString(dimStyle.Render(fmt.Sprintf(" (%s)", stepID)))
    }

    sb.WriteString("\n" + "  " + statsStyle.Render(
        "Done "+buildStatsLine(toolCalls, durationMs, promptTokens, completionTokens),
    ))

    return indentBlock(sb.String(), depth)
}
```

**New function -- `renderTaskComplete`**: Renders the task-level completion line.

```go
func renderTaskComplete(durationMs int64, promptTokens, completionTokens int) string {
    var sb strings.Builder
    sb.WriteString(phaseBulletStyle.Render(bullet))
    sb.WriteString("task complete.  ")
    sb.WriteString(statsStyle.Render(buildStatsLine(0, durationMs, promptTokens, completionTokens)))
    return sb.String()
}
```

### 2.5 vv/cli/cli.go -- Event Handling and Task-Level Aggregation

**Model struct additions**: Add fields for task-level accumulation.

```go
type model struct {
    // ... existing fields ...

    // Task-level stats accumulation.
    taskStart              time.Time
    totalPromptTokens      int
    totalCompletionTokens  int
    totalToolCalls         int

    // Sub-agent level stats accumulation (for DAG path where SubAgentEndData lacks token stats).
    subAgentPromptTokens     int
    subAgentCompletionTokens int
}
```

**handleStreamEvent changes**:

1. **EventLLMCallEnd** (currently suppressed): Still suppress display, but accumulate tokens.

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

3. **EventPhaseEnd**: Pass new fields to `renderPhaseTransition`.

```go
case schema.EventPhaseEnd:
    if data, ok := event.Data.(schema.PhaseEndData); ok {
        // ... existing summary rendering ...
        rendered := renderPhaseTransition(data.Phase, false, data.Duration,
            data.ToolCalls, data.PromptTokens, data.CompletionTokens, 1)
        // ...
    }
```

4. **EventPhaseStart**: Pass zero stats (starting=true ignores them).

```go
rendered := renderPhaseTransition(data.Phase, true, 0, 0, 0, 0, 0)
```

5. **EventSubAgentStart**: Reset sub-agent token accumulators.

```go
case schema.EventSubAgentStart:
    m.nestingDepth++
    m.toolCallCount = 0
    m.subAgentPromptTokens = 0
    m.subAgentCompletionTokens = 0
```

6. **EventSubAgentEnd**: Use new fields when available, fall back to CLI-tracked stats.

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
        rendered := renderSubAgentEnd(data.AgentName, data.StepID, data.Duration,
            toolCalls, promptTokens, completionTokens, m.nestingDepth)
        // ...
        m.toolCallCount = 0
        m.subAgentPromptTokens = 0
        m.subAgentCompletionTokens = 0
    }
```

**handleSubmit change**: Record `taskStart`.

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

**handleStreamDone change**: Render task completion stats.

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
        if m.totalPromptTokens > 0 || m.totalCompletionTokens > 0 {
            rendered := renderTaskComplete(taskDuration, m.totalPromptTokens, m.totalCompletionTokens)
            cmds = append(cmds, tea.Println(rendered))
        }
    }

    // ... existing reset logic ...
}
```

### 2.6 vv/dispatches/explore.go -- Forward LLM Events (Optional Enhancement)

As discussed in Section 2.3, the simpler approach is to NOT modify `exploreStream`. Phase stats for the explore phase come from the `exploreUsage` return value added to the phase stats in `dispatch.go`.

However, if we want `EventLLMCallEnd` events from the explore phase to reach the CLI for task-level accumulation, we should also forward them. Add `EventLLMCallEnd` to the forwarded event types:

```go
case schema.EventToolCallStart, schema.EventToolResult, schema.EventError, schema.EventLLMCallEnd:
    if err := send(event); err != nil {
        slog.Warn("orchestrator: explorer stream send error", "error", err)
    }
```

Similarly in `classifyStream`, forward `EventLLMCallEnd`:

```go
case schema.EventToolCallStart, schema.EventToolResult, schema.EventError, schema.EventLLMCallEnd:
    if err := send(event); err != nil {
        slog.Warn("orchestrator: planner stream send error", "error", err)
    }
```

**Decision**: Do this. It keeps the task-level CLI accumulation consistent -- all token usage flows through `send` and gets captured by both the `phaseSend` wrapper and the CLI model. This avoids double-counting issues when both the return value and the forwarded event carry the same data. If we forward `EventLLMCallEnd`, we do NOT additionally add the returned `usage` to phase stats (the wrapper already captured it).

This simplifies the design: the `phaseSend` wrapper is the single source of truth for phase stats.

## 3. Data Model Changes Summary

### vage/schema/event.go

| Struct | Field | Type | Change |
|--------|-------|------|--------|
| `PhaseEndData` | `ToolCalls` | `int` | **Added** |
| `PhaseEndData` | `PromptTokens` | `int` | **Added** |
| `PhaseEndData` | `CompletionTokens` | `int` | **Added** |
| `SubAgentEndData` | `PromptTokens` | `int` | **Added** |
| `SubAgentEndData` | `CompletionTokens` | `int` | **Added** |
| `SubAgentEndData` | `TokensUsed` | `int` | Kept for backward compat |

## 4. Implementation Plan

### Task 1: Extend event data structs (`vage/schema/event.go`)

Add `ToolCalls`, `PromptTokens`, `CompletionTokens` to `PhaseEndData`. Add `PromptTokens`, `CompletionTokens` to `SubAgentEndData`.

Files: `vage/schema/event.go`

### Task 2: Forward LLM events from explore and classify streams

Modify `exploreStream` and `classifyStream` to forward `EventLLMCallEnd` via `send`. Also forward `EventToolCallStart` from `classifyStream` (it already forwards this).

Files: `vv/dispatches/explore.go`, `vv/dispatches/classify.go`

### Task 3: Accumulate per-phase stats in RunStream (`vv/dispatches/dispatch.go`)

Introduce `phaseStats` struct and `phaseSend` wrapper function. Wrap `send` for each phase to intercept `EventToolCallStart` and `EventLLMCallEnd`. Populate new fields in `PhaseEndData`.

Files: `vv/dispatches/dispatch.go`

### Task 4: Track per-sub-agent prompt/completion tokens (`vv/dispatches/stream.go`)

In `forwardSubAgentStream`, replace `tokensUsed` with `promptTokens`/`completionTokens`. Populate both the new fields and `TokensUsed` (as their sum) in `SubAgentEndData`. Also handle the non-streaming fallback path.

Files: `vv/dispatches/stream.go`

### Task 5: Add rendering helpers and update render functions (`vv/cli/render.go`)

Add `formatCompactTokens`, `buildStatsLine`, `renderTaskComplete`. Update `renderPhaseTransition` signature and implementation. Update `renderSubAgentEnd` signature to use prompt/completion instead of combined tokens.

Files: `vv/cli/render.go`

### Task 6: Update CLI event handling and task-level aggregation (`vv/cli/cli.go`)

Add task-level and sub-agent-level accumulator fields to `model`. Update `handleStreamEvent` for `EventLLMCallEnd`, `EventToolCallStart`, `EventPhaseEnd`, `EventSubAgentStart`, `EventSubAgentEnd`. Update `handleSubmit` to reset accumulators. Update `handleStreamDone` to render task completion line.

Files: `vv/cli/cli.go`

### Task 7: Unit tests for formatting helpers (`vv/cli/render_test.go`)

Add tests for `formatCompactTokens`, `buildStatsLine`, `formatDuration` (already exists but verify edge cases), `renderTaskComplete`, updated `renderPhaseTransition`, updated `renderSubAgentEnd`.

Files: `vv/cli/render_test.go`

## 5. Integration Test Plan

### 5.1 Unit Tests (no LLM keys required)

**File: `vv/cli/render_test.go`**

| Test | Validates |
|------|-----------|
| `TestFormatCompactTokens` | 0 -> "0", 999 -> "999", 1000 -> "1.0k", 5300 -> "5.3k", 1200000 -> "1.2M" |
| `TestBuildStatsLine` | Zero tool calls omitted; all parts joined with " . "; zero tokens omitted |
| `TestRenderPhaseTransition_End` | Contains "phase Dispatch complete." and stats parenthetical |
| `TestRenderSubAgentEnd_WithTokenBreakdown` | Shows separate up/down token counts |
| `TestRenderTaskComplete` | Shows "task complete." with duration and token stats |

**File: `vv/dispatches/stream_test.go`** (if exists, or new)

| Test | Validates |
|------|-----------|
| `TestPhaseStats_Accumulation` | `phaseSend` wrapper correctly counts tool calls and tokens from mock events |

### 5.2 Integration Tests (require LLM API keys)

**File: `vv/integrations/stats_test.go`**

| Test | Validates |
|------|-----------|
| `TestStreamEvents_PhaseEndContainsStats` | Run a Dispatcher.RunStream with a simple request. Collect all events. Verify `PhaseEndData` events have non-zero `PromptTokens` and `CompletionTokens`. |
| `TestStreamEvents_SubAgentEndContainsTokenBreakdown` | Verify `SubAgentEndData` events have non-zero `PromptTokens` and `CompletionTokens` (and `TokensUsed == PromptTokens + CompletionTokens`). |
| `TestStreamEvents_TaskStatsAccumulation` | Sum all `PromptTokens` from `PhaseEndData` events; verify they are consistent with summing `LLMCallEndData.PromptTokens` from all `EventLLMCallEnd` events in the stream. |

### 5.3 Manual Testing Checklist

- [ ] Run `vv` CLI with a simple question (direct mode) -- verify phase stats and task stats display
- [ ] Run `vv` CLI with a complex request (plan mode) -- verify sub-agent stats show token breakdown
- [ ] Verify zero tool calls are omitted from stats lines
- [ ] Verify token formatting: numbers below 1000 show raw, above show "Xk" / "XM"
- [ ] Verify duration formatting: <1s shows ms, <60s shows seconds, >=60s shows "Xm Ys"
- [ ] Run HTTP/SSE mode and inspect event JSON -- verify new fields present in PhaseEndData and SubAgentEndData
