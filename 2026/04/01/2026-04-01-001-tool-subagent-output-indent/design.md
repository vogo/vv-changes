# Technical Design: Tool and Sub-agent Output Indentation

## Architecture Overview

This change is scoped entirely to the CLI display layer (`vv/cli/`). No changes to the event schema (`vage/schema/event.go`), agent framework, or HTTP API are required.

The core idea is to introduce a **nesting depth tracker** on the `model` struct that increments when a sub-agent starts and decrements when it ends. All render functions gain an `indent` parameter (the number of nesting levels), and the CLI event handler passes the current depth when calling render functions. Streaming text deltas accumulated in `m.output` are indented at flush time based on the current depth.

### Visual Output Example

```
● Phase 1/3: Explore                          (depth 0 — no indent)
    ● Step 1/3: researcher                    (depth 1 — 4 chars)
        ● Glob(src/**/*.go)                   (depth 2 — 8 chars, tool inside sub-agent)
        └ 12 files matched                    (depth 2)
    ● researcher └ Done (3 tool uses · 2.1k tokens · 5s)  (depth 1)
● Phase 2/3: Dispatch                         (depth 0)
    ● Step 1/1: coder                         (depth 1)
    Agent: Here is my analysis...             (depth 1, flushed text)
        ● Edit(.../main.go)                   (depth 2)
        └ Added 5 lines                       (depth 2)
    ● coder └ Done (1 tool uses · 1.5k tokens · 3s)       (depth 1)
```

## Component Design

### 1. Depth Tracking (cli.go — `model` struct)

Add a `nestingDepth int` field to `model`. This tracks the current visual nesting level:

- **Phase start/end**: depth stays at 0 (phases are top-level headers).
- **Sub-agent start**: depth increments by 1 before rendering.
- **Sub-agent end**: render at current depth, then decrement by 1.
- **Tool call start/result**: rendered at current depth + 1 (tools are children of whoever invoked them). However, if `nestingDepth == 0` (top-level agent calling a tool directly, no sub-agent), tools render at depth 1.
- **Text delta / flush**: rendered at current depth (depth 0 for top-level agent, depth 1+ inside sub-agents).

Effective tool indent rule: `depth = max(nestingDepth, 1)` for tool calls and results. This ensures tools always have at least one level of indent even when called by the top-level agent.

### 2. Indent Helper (render.go)

Replace the fixed `indentStyle` (PaddingLeft(2)) with a function:

```go
const indentUnit = 4 // 4 characters per nesting level

// indentBlock prepends `depth * indentUnit` spaces to each line of text.
func indentBlock(text string, depth int) string {
    if depth <= 0 {
        return text
    }
    prefix := strings.Repeat(" ", depth*indentUnit)
    lines := strings.Split(text, "\n")
    for i, line := range lines {
        if line != "" {
            lines[i] = prefix + line
        }
    }
    return strings.Join(lines, "\n")
}
```

This is a pure string-based approach rather than lipgloss `PaddingLeft`, because:
- It handles multi-line content (sub-agent descriptions, tool results) correctly.
- It composes cleanly with existing lipgloss color styling (indent applied after styling).
- It avoids creating new lipgloss styles per depth level.

The existing `indentStyle` variable (PaddingLeft(2)) will be removed. All call sites that used `indentStyle.Render(...)` will switch to `indentBlock(...)`.

### 3. Render Function Signature Changes (render.go)

Each render function that produces indented output gains a `depth int` parameter:

| Function | Current Signature | New Signature |
|----------|------------------|---------------|
| `renderToolCallStart` | `(toolName, arguments string) string` | `(toolName, arguments string, depth int) string` |
| `renderToolCallResult` | `(toolName, resultText string) string` | `(toolName, resultText string, depth int) string` |
| `renderSubAgentStart` | `(agentName, stepID, description string, stepIndex, totalSteps int) string` | `(agentName, stepID, description string, stepIndex, totalSteps int, depth int) string` |
| `renderSubAgentEnd` | `(agentName, stepID string, durationMs int64, toolCalls, tokensUsed int) string` | `(agentName, stepID string, durationMs int64, toolCalls, tokensUsed int, depth int) string` |
| `renderPhaseTransition` | `(phase string, phaseIndex, totalPhases int, starting bool) string` | No change (always depth 0) |

Each function applies `indentBlock(result, depth)` as a final step before returning. Internal uses of `indentStyle.Render(...)` (for the "corner" result lines like `└ Done(...)`) are replaced: the `└` line is built as plain text with styling, and the entire block is indented by `indentBlock`.

### 4. Event Handler Changes (cli.go — `handleStreamEvent`)

The `handleStreamEvent` method is updated to pass depth to render functions:

- **`EventPhaseStart` / `EventPhaseEnd`**: No depth change. Render at depth 0 (no indent).
- **`EventSubAgentStart`**: Increment `m.nestingDepth`, then render at `m.nestingDepth`.
- **`EventSubAgentEnd`**: Flush agent output at `m.nestingDepth`, render end at `m.nestingDepth`, then decrement `m.nestingDepth`.
- **`EventToolCallStart`**: Render at `toolDepth()` (see below).
- **`EventToolResult`**: Render at `toolDepth()`.
- **`EventTextDelta`**: No change to accumulation. Indentation applied at flush time.

Helper on `model`:

```go
// toolDepth returns the indent depth for tool call output.
// Tools are always indented at least 1 level, and 1 level deeper than
// any active sub-agent.
func (m *model) toolDepth() int {
    return m.nestingDepth + 1
}
```

### 5. Flush with Indentation (cli.go — `flushAgentOutput`)

`flushAgentOutput` currently prints `agentStyle.Render("Agent: ") + rendered`. Updated to apply indentation:

```go
func (m *model) flushAgentOutput() tea.Cmd {
    if m.output.Len() == 0 {
        return nil
    }
    text := m.output.String()
    rendered := renderAgentMessage(text, m.width-4-(m.nestingDepth*indentUnit))
    // ... store in messages ...
    m.output.Reset()
    line := agentStyle.Render("Agent: ") + rendered
    return tea.Println(indentBlock(line, m.nestingDepth))
}
```

The markdown render width is reduced by the indent amount so text wraps correctly within the available space.

### 6. Live View Indentation (cli.go — `View`)

The `View()` method shows `m.output` as live streaming text. This also needs indentation:

```go
if m.output.Len() > 0 {
    line := agentStyle.Render("Agent: ") + m.output.String()
    sb.WriteString(indentBlock(line, m.nestingDepth))
    sb.WriteString("\n")
}
```

## Data Models / Schemas

No schema changes are required. The `Event` and `EventData` types in `vage/schema/event.go` remain unchanged.

The only data model change is the addition of `nestingDepth int` to the `model` struct in `cli.go`:

```go
type model struct {
    // ... existing fields ...

    // Nesting depth for indentation. 0 = top-level agent,
    // 1 = inside a sub-agent, etc.
    nestingDepth int

    // ... rest of fields ...
}
```

## Implementation Plan

### Task 1: Add `indentBlock` helper and remove `indentStyle`

**File**: `vv/cli/render.go`

1. Add the `indentUnit` constant (value 4).
2. Add the `indentBlock(text string, depth int) string` function.
3. Remove the `indentStyle` variable.

### Task 2: Update render functions to accept depth

**File**: `vv/cli/render.go`

1. Update `renderToolCallStart` — add `depth int` parameter, apply `indentBlock` to the entire output.
2. Update `renderToolCallResult` — add `depth int` parameter, replace `indentStyle.Render(...)` with plain string building, then apply `indentBlock` to the entire output.
3. Update `renderSubAgentStart` — add `depth int` parameter, replace `indentStyle.Render(...)` with plain string building for the description line, then apply `indentBlock`.
4. Update `renderSubAgentEnd` — add `depth int` parameter, replace `indentStyle.Render(...)` with plain string building for the stats line, then apply `indentBlock`.
5. Confirm `renderPhaseTransition` needs no changes (always depth 0).

### Task 3: Add nesting depth tracking to model

**File**: `vv/cli/cli.go`

1. Add `nestingDepth int` field to the `model` struct.
2. Add `toolDepth() int` method on `model`.

### Task 4: Update event handler to pass depth

**File**: `vv/cli/cli.go`

1. `EventSubAgentStart` handler: increment `m.nestingDepth` before rendering, pass `m.nestingDepth` to `renderSubAgentStart`.
2. `EventSubAgentEnd` handler: pass `m.nestingDepth` to `renderSubAgentEnd`, then decrement `m.nestingDepth`.
3. `EventToolCallStart` handler: pass `m.toolDepth()` to `renderToolCallStart`.
4. `EventToolResult` handler: pass `m.toolDepth()` to `renderToolCallResult`.

### Task 5: Update flush and live view for indentation

**File**: `vv/cli/cli.go`

1. Update `flushAgentOutput` to apply `indentBlock` at `m.nestingDepth`, and reduce markdown render width.
2. Update `View()` to apply `indentBlock` to live streaming output at `m.nestingDepth`.

### Task 6: Update existing tests and add new tests

**Files**: `vv/cli/render_test.go`, `vv/cli/cli_test.go`

See unit test plan below.

## Unit Test Plan

### render_test.go — New Tests

1. **`TestIndentBlock_ZeroDepth`** — Verify `indentBlock("hello", 0)` returns `"hello"` unchanged.

2. **`TestIndentBlock_SingleLevel`** — Verify `indentBlock("hello", 1)` returns `"    hello"` (4 spaces).

3. **`TestIndentBlock_MultiLevel`** — Verify `indentBlock("hello", 2)` returns `"        hello"` (8 spaces).

4. **`TestIndentBlock_Multiline`** — Verify multi-line input gets each non-empty line indented:
   - Input: `"line1\nline2\n\nline3"`, depth 1
   - Expected: `"    line1\n    line2\n\n    line3"`

5. **`TestIndentBlock_EmptyString`** — Verify `indentBlock("", 1)` returns `""`.

6. **`TestRenderToolCallStart_Indented`** — Call `renderToolCallStart("bash", `{"command":"ls"}`, 1)` and verify the output starts with 4 spaces.

7. **`TestRenderToolCallStart_Depth2`** — Call with depth 2 and verify 8 leading spaces.

8. **`TestRenderToolCallResult_Indented`** — Call `renderToolCallResult("bash", "output text", 1)` and verify 4-space indent prefix.

9. **`TestRenderSubAgentStart_Indented`** — Call `renderSubAgentStart("coder", "step1", "do coding", 1, 3, 1)` and verify 4-space indent. Also verify description sub-line has the same indent base.

10. **`TestRenderSubAgentEnd_Indented`** — Call `renderSubAgentEnd("coder", "step1", 5000, 3, 1500, 1)` and verify 4-space indent.

11. **`TestRenderPhaseTransition_NoIndent`** — Verify `renderPhaseTransition(...)` output does NOT start with spaces (unchanged behavior).

### render_test.go — Updated Tests

12. **Update `TestRenderToolMessage`** — No change needed (this function is unrelated to indentation).

### cli_test.go — New/Updated Tests

13. **`TestToolDepth_NoSubAgent`** — Create a `model` with `nestingDepth: 0`, verify `toolDepth()` returns 1.

14. **`TestToolDepth_InsideSubAgent`** — Create a `model` with `nestingDepth: 1`, verify `toolDepth()` returns 2.

15. **`TestNestingDepth_SubAgentLifecycle`** — Simulate a sequence of `handleStreamEvent` calls: SubAgentStart (verify depth becomes 1), ToolCallStart (verify tool rendered at depth 2), SubAgentEnd (verify depth returns to 0). Assert depth at each step by checking `m.nestingDepth`.
