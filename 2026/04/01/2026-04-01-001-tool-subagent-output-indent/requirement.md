# Requirement: Tool and Sub-agent Output Indentation

## Background

In the vv CLI application, tool calls, tool results, sub-agent output, and streaming text from sub-agents are currently displayed with minimal or inconsistent indentation relative to the surrounding context. When an agent invokes tools or delegates to sub-agents, the output visually blends with the top-level agent output, making it difficult for users to distinguish the hierarchical relationship between the main agent, its tool calls, and its sub-agents.

The existing indentation (defined as `indentStyle` with `PaddingLeft(2)`) is applied only to certain result lines (e.g., tool result summaries and sub-agent completion stats), but is not applied uniformly to all tool and sub-agent content, and does not use a 4-character indent.

## Objectives

- Improve readability of CLI output by establishing clear visual hierarchy through indentation.
- Make it immediately obvious which output belongs to a tool call or sub-agent versus the top-level agent.
- Apply a consistent 4-character indent to all content printed by tools and sub-agents relative to the current context level.

## User Stories

### US-1: Tool call output is indented

**As** a developer using the vv CLI,
**I want** tool call start lines, tool result summaries, and any tool output to be indented by 4 characters relative to the agent output,
**So that** I can visually distinguish tool activity from the agent's own reasoning text.

**Acceptance Criteria:**
- Tool call start lines (e.g., "● ToolName(summary)") are indented by 4 characters from the left margin.
- Tool result lines (e.g., "└ Done..." or "└ Added 3 lines") are indented by 4 characters from the left margin.
- The indentation is consistent across all tool types (bash, read, write, edit, glob, grep).

### US-2: Sub-agent output is indented

**As** a developer using the vv CLI,
**I want** sub-agent start/end lines, and all streaming text produced while a sub-agent is active, to be indented by 4 characters relative to the parent context,
**So that** I can clearly see the boundary between the orchestrator's output and a sub-agent's output.

**Acceptance Criteria:**
- Sub-agent start lines (e.g., "● Step 1/3: coder") are indented by 4 characters.
- Sub-agent end/summary lines (e.g., "● coder ... └ Done (...)") are indented by 4 characters.
- Streaming text deltas produced during a sub-agent's execution are indented by 4 characters.
- If a sub-agent invokes tools, those tool lines receive an additional 4-character indent (8 characters total from the left margin), maintaining relative hierarchy.

### US-3: Phase transition lines remain at top level

**As** a developer using the vv CLI,
**I want** phase transition lines (e.g., "Phase 1/3: Explore") to remain at the top level without indentation,
**So that** they serve as clear section headers in the output.

**Acceptance Criteria:**
- Phase start and phase end lines are not indented.
- They visually separate groups of indented sub-agent and tool output.

## Scope

### In Scope

- CLI output rendering for tool call start, tool call result, sub-agent start, sub-agent end, and streaming text within sub-agent context.
- Applying a 4-character indent per nesting level in the CLI display layer.
- Maintaining the existing color styling and formatting while adding indentation.

### Out of Scope

- HTTP API response formatting (this is a CLI-only visual change).
- Changes to the event schema or agent framework internals.
- Changes to the content of tool results or agent messages themselves.
- Nested sub-agent indentation beyond two levels (agent > sub-agent > tool) unless naturally supported.

## Involved Components

| Component | Path | Role |
|-----------|------|------|
| CLI render | `vv/cli/render.go` | Render functions for tool calls, sub-agent lines, and text formatting; defines indent styles |
| CLI app | `vv/cli/cli.go` | Stream event handling that calls render functions; manages output buffer and flush logic |
| CLI messages | `vv/cli/messages.go` | Display message types |
| Schema events | `vage/schema/event.go` | Event types and data structures (read-only context, no changes expected) |
