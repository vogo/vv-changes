# Requirement: Phase & Sub-Agent Stats Display

## 1. Background

The vv agent application orchestrates user requests through a multi-phase dispatch pipeline (Explore, Plan, Dispatch) with sub-agent execution. While the framework already emits lifecycle events (`phase_start`, `phase_end`, `sub_agent_start`, `sub_agent_end`) and the CLI renders phase transitions and sub-agent boundaries, the completion messages lack meaningful execution statistics. Users cannot see how long each phase or sub-agent took, how many tokens were consumed, or how many tool calls were made.

The existing `PhaseEndData` carries only `Duration` and an optional `Summary`. The existing `SubAgentEndData` carries `Duration`, `ToolCalls`, and `TokensUsed` but these are not always populated. The CLI's `renderPhaseTransition` for phase-end shows only a dim "Phase complete." message with no stats. The `renderSubAgentEnd` shows stats but uses a combined `tokensUsed` (total tokens) rather than the requested input/output breakdown.

## 2. Objective

Display execution statistics (duration, tool call count, input token count, output token count) when each phase, sub-agent, and overall task completes, so users can understand resource consumption and performance at every level.

## 3. User Stories

### US-1: Phase Completion Stats
**As** a vv CLI user,
**I want** to see stats when each phase completes,
**So that** I can understand how long each orchestration phase took and how many resources it consumed.

**Acceptance Criteria:**
- When a phase ends, the CLI displays: `● phase <Name> complete.  (<tool_count> tool uses · <duration> · ↑ <input_tokens> · ↓ <output_tokens>)`
- Example: `● phase Dispatch complete.  (3 tool uses · 26s · ↑ 5.3k · ↓ 10.5k)`
- Duration uses human-readable format: `<1s` shows milliseconds, `<60s` shows seconds, `>=60s` shows `Xm Ys`
- Token counts use compact format: raw number below 1000, `X.Xk` for thousands, `X.XM` for millions
- `↑` represents input/prompt tokens; `↓` represents output/completion tokens
- Tool call count is omitted from the stats parenthetical if zero

### US-2: Sub-Agent Completion Stats
**As** a vv CLI user,
**I want** to see stats when each sub-agent completes,
**So that** I can understand the cost and performance of individual agent executions.

**Acceptance Criteria:**
- When a sub-agent ends, the CLI displays: `● sub-agent <name> complete.  (<tool_count> tool uses · <duration> · ↑ <input_tokens> · ↓ <output_tokens>)`
- Example: `● sub-agent research complete.  (5 tool uses · 26s · ↑ 5.3k · ↓ 10.5k)`
- Same formatting rules as US-1 for duration and token counts
- Tool call count is omitted if zero

### US-3: Overall Task Completion Stats
**As** a vv CLI user,
**I want** to see aggregated stats when the entire task completes,
**So that** I can understand the total resource consumption for my request.

**Acceptance Criteria:**
- When the full stream completes, the CLI displays: `● task complete.  (<duration> · ↑ <total_input_tokens> · ↓ <total_output_tokens>)`
- Example: `● task complete.  (5m 26s · ↑ 11.3k · ↓ 30.5k)`
- Aggregates all token usage across all phases and sub-agents
- Tool call count is optional at the task level (may be omitted for brevity)

## 4. Scope

### In Scope
- Tracking and displaying input (prompt) and output (completion) token counts separately
- Tracking tool call counts per phase, per sub-agent, and overall
- Tracking duration per phase, per sub-agent, and overall
- Formatting stats in the specified human-readable format
- CLI (Bubble Tea) rendering of stats lines
- HTTP/SSE streaming mode: enriching event data so HTTP clients receive the same stats

### Out of Scope
- Cost estimation (dollar amounts based on token pricing)
- Persistent stats storage or historical analysis
- Per-tool-call duration display (already exists elsewhere)
- Stats configuration (enable/disable, verbosity levels) -- can be added later

## 5. Involved Components

### 5.1 vage Module (schema layer)

**File: `vage/schema/event.go`**

- `PhaseEndData`: Currently has `Phase`, `Duration`, `Summary`. Needs additional fields:
  - `ToolCalls int` -- number of tool calls during this phase
  - `PromptTokens int` -- input tokens consumed during this phase
  - `CompletionTokens int` -- output tokens consumed during this phase
- `SubAgentEndData`: Currently has `AgentName`, `StepID`, `Duration`, `ToolCalls`, `TokensUsed`. Needs:
  - Replace `TokensUsed int` with `PromptTokens int` and `CompletionTokens int` (or add the two new fields alongside for backward compatibility)

### 5.2 vv Module (dispatcher layer)

**File: `vv/dispatches/dispatch.go` (RunStream method)**

- The `RunStream` method orchestrates phases and emits `PhaseStartData`/`PhaseEndData` events. Currently, `PhaseEndData` only includes `Duration` and `Summary`.
- Each phase needs to accumulate: tool call count, prompt tokens, and completion tokens from all events received during that phase.
- The dispatcher must track these counters per-phase by observing `EventToolCallStart` and `EventLLMCallEnd` events flowing through.

**File: `vv/dispatches/stream.go`**

- `forwardSubAgentStream`: Currently tracks `toolCalls` and `tokensUsed` (total). Must separately track `promptTokens` and `completionTokens` from `LLMCallEndData` events.
- `streamingDAGHandler.OnNodeComplete`: Currently emits `SubAgentEndData` with only `Duration`. Should also emit token and tool call stats (requires the handler to track these from forwarded events).

**File: `vv/dispatches/explore.go`**

- `exploreStream`: Already accumulates `usage.PromptTokens` and `usage.CompletionTokens`. This data needs to flow into the `PhaseEndData` for the explore phase.

### 5.3 vv Module (CLI rendering layer)

**File: `vv/cli/render.go`**

- `renderPhaseTransition`: Currently shows only "Phase complete." for phase-end. Must accept and display stats (tool calls, duration, prompt tokens, completion tokens).
- `renderSubAgentEnd`: Currently shows stats with combined `tokensUsed`. Must display `↑` and `↓` separately.
- New function or modification: render task completion stats line.
- `formatTokens`: Already exists and formats token counts. May need minor adjustment to match the `↑ 5.3k` format (currently outputs `5.3k tokens`).

**File: `vv/cli/cli.go`**

- `handleStreamEvent` for `EventPhaseEnd`: Must pass new stats fields to the render function.
- `handleStreamEvent` for `EventSubAgentEnd`: Must pass separate prompt/completion token counts.
- `handleStreamDone`: Should render the overall task completion stats line. Needs to accumulate total duration (from task start) and total token counts across all phases.
- The `model` struct may need additional fields to track task-level aggregates: `taskStart time.Time`, `totalPromptTokens int`, `totalCompletionTokens int`, `totalToolCalls int`.

## 6. Data Flow

```
LLM Call completes
  → EventLLMCallEnd { PromptTokens, CompletionTokens, TotalTokens }
     → forwardSubAgentStream accumulates per-sub-agent
        → SubAgentEndData { PromptTokens, CompletionTokens, ToolCalls, Duration }
     → RunStream phase loop accumulates per-phase
        → PhaseEndData { PromptTokens, CompletionTokens, ToolCalls, Duration }
     → CLI model accumulates per-task
        → Task completion line { TotalPromptTokens, TotalCompletionTokens, TotalDuration }
```

## 7. Display Format Specification

### Token Format
| Value | Display |
|-------|---------|
| 0 | omit |
| 1-999 | `N` (raw number) |
| 1000-999999 | `X.Xk` (one decimal, e.g., `5.3k`) |
| 1000000+ | `X.XM` (one decimal, e.g., `1.2M`) |

Prefix: `↑` for input/prompt tokens, `↓` for output/completion tokens.

### Duration Format
| Value | Display |
|-------|---------|
| < 1s | `Nms` |
| 1s - 59s | `Ns` |
| 60s+ | `Nm Ns` (omit seconds part if zero) |

### Stats Line Format
Parts are joined by ` · ` (space-dot-space). Parts with zero values are omitted.

- Phase: `● phase <Name> complete.  (<tool_count> tool uses · <duration> · ↑ <input> · ↓ <output>)`
- Sub-agent: `● sub-agent <name> complete.  (<tool_count> tool uses · <duration> · ↑ <input> · ↓ <output>)`
- Task: `● task complete.  (<duration> · ↑ <input> · ↓ <output>)`

### Styling
- The stats parenthetical uses the existing `statsStyle` (dim gray, lipgloss color "8")
- The `↑` and `↓` arrows are part of the dim stats text (no special coloring)
- Phase bullet uses `phaseBulletStyle` (yellow)
- Sub-agent bullet uses `subAgentBulletStyle` (green)
- Task bullet uses `phaseBulletStyle` (yellow) or a dedicated style
