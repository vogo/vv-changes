# Requirement: TUI Tool Execution Logging for Explore and Plan Phases

## Background

The vv CLI TUI uses a three-phase orchestration pipeline: Explore, Plan, and Dispatch. During the Dispatch phase, tool execution events (tool call start, tool result) are properly streamed to the TUI via `forwardSubAgentStream`, giving users real-time visibility into what the agents are doing. However, the Explore and Plan phases invoke their sub-agents using non-streaming `Run()` calls (`o.explore()` and `o.planTask()`), which means all tool execution events during these phases are silently consumed and never forwarded to the TUI.

Users see phase start/end markers (e.g., "Phase 1/3: Explore" ... "Explore phase complete.") but have no visibility into what tool calls are happening within those phases. This creates a "black box" experience during exploration, where significant file reading, globbing, and grepping activity occurs invisibly.

## Objective

Surface tool execution logs in the TUI for all orchestration phases -- specifically the Explore phase (and Plan phase if it gains tools in the future) -- so users can see what files are being read, what patterns are being searched, and what operations agents are performing in real time.

## User Stories

### US-1: View Tool Calls During Explore Phase

**As a** developer using the vv CLI,
**I want to** see tool execution logs (read, glob, grep) during the Explore phase,
**So that** I understand what parts of my codebase the agent is examining to build context.

**Acceptance Criteria:**
- When the Explore phase is active, each tool call (read, glob, grep) appears in the TUI output as it happens
- Tool call display follows the existing format: `"  [tool_indicator] tool_name: summary"` (e.g., `"  ○ read: cli/cli.go"`)
- Tool calls appear indented under the phase header to indicate they belong to that phase
- The Explore phase start/end markers continue to display as before
- Tool result details are shown in compact form consistent with existing Dispatch phase rendering

### US-2: View Tool Calls During Plan Phase

**As a** developer using the vv CLI,
**I want to** see any tool execution logs during the Plan phase (if applicable),
**So that** I have full visibility into all agent operations.

**Acceptance Criteria:**
- If the Plan phase sub-agent executes tools, those tool calls appear in the TUI
- The display format is consistent with the Explore phase tool logging
- Currently the planner agent has no tools (MaxIterations=1, no tool registry), so this is primarily future-proofing

### US-3: Consistent Tool Call Display Across Phases

**As a** developer using the vv CLI,
**I want** tool execution logs to look the same regardless of which phase they occur in,
**So that** the TUI output is coherent and easy to follow.

**Acceptance Criteria:**
- Tool call rendering during Explore/Plan phases uses the same styles and formatting as Dispatch phase tool calls
- The existing `renderToolCallStart` and `renderToolCallResult` functions are reused
- Phase context is visually clear (tool calls appear between phase start and phase end markers)

## Scope

### In Scope

- Streaming tool execution events from the Explore phase sub-agent to the TUI
- Streaming tool execution events from the Plan phase sub-agent to the TUI (future-proofing)
- Reusing existing TUI rendering for tool call start and tool result events
- Maintaining the existing phase start/end event structure

### Out of Scope

- Changing the visual design of tool call rendering (already well-designed)
- Adding new tool call event types or data structures
- Modifying the Dispatch phase behavior (already works correctly)
- Adding filtering or verbosity controls for tool logs
- Changing the non-streaming `Run()` path (HTTP API / synchronous mode)

## Involved System Roles

| Role | Involvement |
|------|-------------|
| Developer (CLI user) | Primary consumer of tool execution logs in the TUI |

## Involved Models and State Changes

No model or state changes required. The existing `schema.Event` types (`EventToolCallStart`, `EventToolCallEnd`, `EventToolResult`) and `DisplayMessage` types are sufficient.

## Involved Business Processes

### Orchestration - RunStream

The `OrchestratorAgent.RunStream()` method currently:
1. **Explore phase**: Calls `o.explore(ctx, req)` which uses `explorerAgent.Run()` (non-streaming). Tool events are lost.
2. **Plan phase**: Calls `o.planTask(ctx, req, contextSummary)` which uses `plannerAgent.Run()` (non-streaming). Tool events are lost.
3. **Dispatch phase**: Calls `o.forwardSubAgentStream()` which streams all events including tool calls. Works correctly.

**Required change**: Phases 1 and 2 need to use streaming execution (similar to `forwardSubAgentStream`) so that tool events are forwarded to the TUI via the `send` callback. The explore and plan results (context summary, classification JSON) still need to be captured from the stream output for use in subsequent phases.

### TUI Event Handling

The TUI `handleStreamEvent()` method already handles `EventToolCallStart` and `EventToolResult` correctly. No changes needed in the TUI layer -- once the orchestrator forwards the events, they will render automatically.

## Involved Applications and Pages

| Application | Page/Component | Impact |
|-------------|---------------|--------|
| CLI TUI | Viewport (output area) | Tool call messages will appear during Explore/Plan phases (no code change needed in TUI) |

## Key Technical Insight

The fix is concentrated in the orchestrator layer (`vv/agents/orchestrator.go`). The Explore and Plan phases need to switch from non-streaming `Run()` to streaming `RunStream()` execution, forwarding tool events through the `send` callback while still capturing the final text output for use in subsequent phases. The TUI rendering layer already fully supports the required event types.

## Expected TUI Output (After Change)

```
vv · openai · doubao-1-5-pro-32k-250115
  ~/workspaces/github/vogo/vagents/vv

● Phase 1/3: Explore
● read(cli/cli.go)
● read(agents/orchestrator.go)
● glob(**/*.go)
● grep(handleStreamEvent)
● Explore phase complete.
● Phase 2/3: Plan
● Plan phase complete.
● Phase 3/3: Dispatch
● coder
Agent: [response text]
● coder
  └ Done (11s)
● Dispatch phase complete.
```
