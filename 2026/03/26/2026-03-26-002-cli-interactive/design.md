# CLI Interactive Mode - Technical Design

## Architecture Overview

The CLI interactive mode adds a terminal user interface (TUI) as an alternative entry point to vv, alongside the existing HTTP server mode. The design introduces a new `cli` package under `vv/cli/` that contains the bubbletea TUI application. The core principle is **in-process agent invocation** -- the CLI calls task agents directly via their `RunStream()` method, consuming the same `schema.RunStream` event stream that the HTTP SSE handler uses.

A key architectural detail: the router agent's `RunStream()` delegates to `agent.RunToStream()`, which wraps a blocking `Run()` and emits only `EventAgentStart` and `EventAgentEnd` -- losing all intermediate streaming events (text deltas, tool calls). Therefore, the CLI performs routing separately (via `routeragent.LLMFunc`) and then calls the selected sub-agent's `RunStream()` directly to get full event granularity.

```
vv/main.go
  |
  +-- mode=http --> service.New(...).Start(ctx)    [existing path]
  |
  +-- mode=cli  --> cli.New(...).Run(ctx)           [new path]
                      |
                      +-- bubbletea Program
                      |     +-- Input area (textarea)
                      |     +-- Output area (viewport, scrollable)
                      |     +-- Confirmation dialog (huh)
                      |
                      +-- Agent invocation (in-process)
                            +-- Route selection via routeragent.LLMFunc
                            +-- selectedAgent.RunStream(ctx, req)
                            +-- Consume RunStream events (full streaming)
                            +-- Confirmation callback for tool calls
```

Both paths share the same initialization: config loading, LLM client creation, tool registration, and agent creation. The divergence happens only at the "start service or start TUI" decision point.

## Component Design

### Package Structure

```
vv/
  main.go              -- updated: add --mode flag, branch on mode
  cli/
    cli.go             -- App struct, New(), Run(), bubbletea model
    messages.go        -- bubbletea Msg types (streamEvent, confirmResult, etc.)
    render.go          -- Markdown rendering and event formatting helpers
    confirm.go         -- Tool call confirmation using confirmingExecutor and huh
  config/
    config.go          -- updated: add Mode and CLI fields
  agents/
    agents.go          -- updated: accept tool.ToolRegistry interface
  tools/               -- unchanged
```

### cli.App

The central struct that owns the bubbletea program and agent references.

```go
package cli

// App holds the CLI TUI application state.
type App struct {
    router    *routeragent.Agent     // the router agent (for routing decisions only)
    coder     agent.StreamAgent      // coder task agent (implements StreamAgent natively)
    chat      agent.StreamAgent      // chat task agent (implements StreamAgent natively)
    routes    []routeragent.Route    // route definitions for LLMFunc
    routeFn   routeragent.RouteFunc  // routing function (LLMFunc)
    cfg       *config.Config
    sessionID string
    history   []schema.Message       // full conversation history for multi-turn context
    messages  []DisplayMessage       // rendered conversation history for the viewport
    program   *tea.Program           // stored for p.Send() from goroutines
}

// New creates a new CLI App.
func New(
    routeFn routeragent.RouteFunc,
    routes []routeragent.Route,
    coder agent.StreamAgent,
    chat agent.StreamAgent,
    cfg *config.Config,
) *App

// Run starts the bubbletea program and blocks until exit.
func (a *App) Run(ctx context.Context) error
```

The `Run()` method generates a session ID, creates the bubbletea program with `tea.NewProgram()`, stores the program reference in `a.program` for use by stream consumer goroutines, then calls `a.program.Run()`.

Session ID generation follows the existing pattern from `service.go`:

```go
b := make([]byte, 8)
_, _ = rand.Read(b)
a.sessionID = hex.EncodeToString(b)
```

### Logging Redirection

The existing codebase uses `slog` for logging. When the TUI is active, slog output to stderr would corrupt the terminal display. Before starting the bubbletea program, CLI mode must redirect logging:

```go
func (a *App) Run(ctx context.Context) error {
    // Redirect slog to a file or discard to avoid corrupting TUI.
    logFile, err := os.OpenFile(
        filepath.Join(config.DefaultDir(), "vv.log"),
        os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0o600,
    )
    if err != nil {
        slog.SetDefault(slog.New(slog.DiscardHandler{}))
    } else {
        defer logFile.Close()
        slog.SetDefault(slog.New(slog.NewTextHandler(logFile, nil)))
    }

    // ... create and run bubbletea program
}
```

### Bubbletea Model

The bubbletea model is a single struct that manages TUI state and delegates to sub-components.

```go
type model struct {
    app       *App
    ctx       context.Context
    cancel    context.CancelFunc

    // UI components
    textarea  textarea.Model      // user input
    viewport  viewport.Model      // scrollable output area
    width     int
    height    int

    // State
    status    sessionStatus       // idle, processing, confirming, quitting
    spinner   spinner.Model
    output    strings.Builder     // current agent response being streamed
    err       error

    // Confirmation
    confirmCh   chan bool           // send user's confirm/reject decision
    confirmForm *huh.Form          // huh form for confirmation dialog
    pendingTC   *schema.ToolCallStartData  // tool call awaiting confirmation
}
```

**Session status enum:**

```go
type sessionStatus int

const (
    statusIdle sessionStatus = iota
    statusProcessing
    statusConfirming
    statusQuitting
)
```

### Bubbletea Message Types

```go
// streamEventMsg wraps a schema.Event received from RunStream.
type streamEventMsg struct {
    event schema.Event
}

// streamDoneMsg signals the stream has ended (EOF or error).
type streamDoneMsg struct {
    err error
}

// confirmRequestMsg signals that a tool call needs user confirmation.
// This is sent from the confirmFn goroutine to the bubbletea event loop.
type confirmRequestMsg struct {
    toolName  string
    arguments string
}

// confirmResultMsg carries the user's confirmation decision.
type confirmResultMsg struct {
    approved bool
}
```

### DisplayMessage

A simplified message model for rendering in the viewport.

```go
// DisplayMessage represents a rendered message in the conversation history.
type DisplayMessage struct {
    Role      string    // "user", "agent", "system", "tool", "error"
    Content   string    // rendered text content
    Timestamp time.Time
}
```

## Data Models / Schemas

### Config Changes

Two new fields are added to `config.Config`:

```go
type Config struct {
    LLM    LLMConfig    `yaml:"llm"`
    Server ServerConfig `yaml:"server"`
    Tools  ToolsConfig  `yaml:"tools"`
    Agents AgentsConfig `yaml:"agents"`
    Mode   string       `yaml:"mode"`           // "cli" or "http"; default "cli"
    CLI    CLIConfig    `yaml:"cli"`
}

type CLIConfig struct {
    ConfirmTools []string `yaml:"confirm_tools"` // tool names requiring confirmation
}
```

Environment variable override:

| Variable | Field | Description |
|----------|-------|-------------|
| `VV_MODE` | `Mode` | Application run mode ("cli" or "http") |

### agents.Create Refactoring

The `agents.Create()` function must accept `tool.ToolRegistry` (interface) instead of `*tool.Registry` (concrete) to allow the CLI to pass a confirming wrapper:

```go
// Before (current):
func Create(cfg *config.Config, llm aimodel.ChatCompleter, reg *tool.Registry) (...)

// After:
func Create(cfg *config.Config, llm aimodel.ChatCompleter, reg tool.ToolRegistry) (...)
```

This is a backward-compatible change since `*tool.Registry` satisfies `tool.ToolRegistry`.

### Session Model (In-Memory Only)

The CLI session is entirely in-memory within the `App` struct. No new persistent data model is needed. Conversation history is held in two forms:

1. `[]schema.Message` (`history`) -- the full message history passed to the agent on each turn for multi-turn context.
2. `[]DisplayMessage` (`messages`) -- the rendered conversation for viewport display.

## Event Flow

### Routing and Agent Selection

Because `routeragent.RunStream()` wraps `Run()` via `RunToStream()` and loses intermediate events, the CLI performs routing as a separate step:

```go
func (a *App) selectAgent(ctx context.Context, req *schema.RunRequest) (agent.StreamAgent, error) {
    result, err := a.routeFn(ctx, req, a.routes)
    if err != nil {
        return nil, fmt.Errorf("route select: %w", err)
    }
    sa, ok := result.Agent.(agent.StreamAgent)
    if !ok {
        return nil, fmt.Errorf("selected agent %q does not support streaming", result.Agent.ID())
    }
    return sa, nil
}
```

This calls `routeragent.LLMFunc` to pick the right agent, then the CLI calls `selectedAgent.RunStream()` directly, getting the full event stream including text deltas and tool call events.

### Message Processing Flow

1. User types message in textarea, presses Enter.
2. Model transitions to `statusProcessing`, disables textarea.
3. `model.Update` returns a `tea.Cmd` that spawns a goroutine:
   a. Creates a `schema.RunRequest` with the full conversation history plus the new user message.
   b. Calls `app.selectAgent(ctx, req)` to resolve the target agent.
   c. Calls `selectedAgent.RunStream(ctx, req)` to get a `*schema.RunStream`.
   d. Enters a receive loop calling `stream.Recv()`.
   e. For each event, sends a `streamEventMsg` via `app.program.Send()`.
   f. On `io.EOF`, sends `streamDoneMsg{}` via `app.program.Send()`.
   g. On error, sends `streamDoneMsg{err: err}` via `app.program.Send()`.
4. `model.Update` handles each `streamEventMsg`:
   - `EventAgentStart`: display "Agent started" indicator.
   - `EventIterationStart`: display iteration progress (e.g., "Iteration 2/10...").
   - `EventTextDelta`: append delta text to the output builder, update viewport.
   - `EventToolCallStart`: if tool is in `confirm_tools`, transition to `statusConfirming` and show confirmation dialog; otherwise display tool call info.
   - `EventToolCallEnd`: display tool duration.
   - `EventToolResult`: display result summary (truncated if long).
   - `EventTokenBudgetExhausted`: display budget warning.
   - `EventAgentEnd`: finalize the response message, apply glamour rendering to the complete response.
   - `EventLLMCallStart` / `EventLLMCallEnd` / `EventLLMCallError`: optionally display (can be suppressed for cleaner UX).
   - `EventError`: display error as a styled system message (red).
   - Unknown event types: ignore silently.
5. On `streamDoneMsg`:
   - If `err != nil`: display error as a styled system message in the viewport.
   - Append the agent's final text to `app.history` as a `schema.Message` for multi-turn context.
   - Transition back to `statusIdle`, re-enable textarea.

### Tool Confirmation Flow

The confirmation mechanism uses a channel-based callback pattern with a confirming executor wrapper.

```go
// confirmingExecutor wraps a tool.ToolRegistry, intercepting Execute() calls
// for tools that require user confirmation.
type confirmingExecutor struct {
    tool.ToolRegistry              // embed to delegate Register, Unregister, Get, List, Merge
    confirmTools map[string]bool
    confirmFn    func(ctx context.Context, toolName, args string) (bool, error)
}

func (r *confirmingExecutor) Execute(ctx context.Context, name, args string) (schema.ToolResult, error) {
    if r.confirmTools[name] {
        approved, err := r.confirmFn(ctx, name, args)
        if err != nil {
            return schema.ErrorResult("", err.Error()), nil
        }
        if !approved {
            return schema.ErrorResult("", "Tool call rejected by user"), nil
        }
    }
    return r.ToolRegistry.Execute(ctx, name, args)
}
```

The `confirmFn` is provided by the TUI model. It:
1. Sends a `confirmRequestMsg` via `app.program.Send()` to show the confirmation dialog.
2. Blocks on a channel (`confirmCh`) with context cancellation awareness:
   ```go
   select {
   case approved := <-confirmCh:
       return approved, nil
   case <-ctx.Done():
       return false, ctx.Err()
   }
   ```
3. Returns the decision.

This works because `RunStream` runs the producer in a separate goroutine, so blocking in `confirmFn` does not freeze the TUI event loop. The `select` on `ctx.Done()` prevents goroutine leaks when the stream is cancelled (e.g., Ctrl+C during confirmation).

The confirmation dialog uses `huh.NewConfirm()` wrapped in a `huh.NewForm()`:

```go
func (m *model) showConfirmDialog(toolName, args string) tea.Cmd {
    var approved bool
    m.confirmForm = huh.NewForm(
        huh.NewGroup(
            huh.NewConfirm().
                Title(fmt.Sprintf("Allow tool call: %s?", toolName)).
                Description(truncate(args, 200)).
                Affirmative("Yes").
                Negative("No").
                Value(&approved),
        ),
    ).WithShowHelp(false)

    return m.confirmForm.Init()
}
```

When in `statusConfirming`, the parent model delegates `Update()` and `View()` to the huh form. When the form completes, the result is sent to `confirmCh` and the model transitions back to `statusProcessing`.

### Cancellation Flow

- First Ctrl+C: if `statusProcessing`, cancel the per-run context (via a per-run `context.CancelFunc`), which causes the producer goroutine to terminate. The `confirmFn` also returns immediately via `ctx.Done()` if blocked. The model transitions back to `statusIdle`.
- First Ctrl+C while `statusConfirming`: send `false` to `confirmCh`, cancel the run, transition to `statusIdle`.
- First Ctrl+C while `statusIdle`: transition to `statusQuitting`, send `tea.Quit`.
- Second Ctrl+C during processing: immediate quit via `tea.Quit`.

### Exit Commands

`/exit` and `/quit` typed as input trigger `tea.Quit`. These are detected in `Update` before submitting to the agent.

## API Contracts

No new HTTP APIs. The CLI mode bypasses HTTP entirely.

The interface contracts between the CLI and the agent layer are:

1. **Routing:** `routeragent.RouteFunc` -- used to select the target agent.
   ```go
   func(ctx context.Context, req *schema.RunRequest, routes []routeragent.Route) (*routeragent.RouteResult, error)
   ```

2. **Streaming:** `agent.StreamAgent` -- used to invoke the selected agent.
   ```go
   RunStream(ctx context.Context, req *schema.RunRequest) (*schema.RunStream, error)
   ```

3. **Tool execution with confirmation:** `tool.ToolRegistry` (via `confirmingExecutor`).
   ```go
   Execute(ctx context.Context, name, args string) (schema.ToolResult, error)
   ```

The task agents implement `StreamAgent` natively (not via `RunToStream`), producing the full event stream. No changes to the vage framework are required.

## Markdown Rendering

For terminal markdown rendering, use `github.com/charmbracelet/glamour`. This is part of the Charm ecosystem alongside bubbletea and integrates naturally. The renderer is created once at startup with a terminal-appropriate style (auto-detected dark/light).

```go
renderer, _ := glamour.NewTermRenderer(
    glamour.WithAutoStyle(),
    glamour.WithWordWrap(width),
)
```

Agent text output is accumulated per-response. During streaming, raw text is displayed in the viewport. When the response completes (`EventAgentEnd`), the full response is re-rendered through glamour and the viewport content is updated. This causes a brief visual reflow but avoids partial-markdown rendering artifacts from incomplete blocks.

For v1, this reflow is accepted as a known limitation. A future optimization could render complete markdown blocks incrementally.

## Terminal Display

The TUI uses bubbletea's alternate screen mode (`tea.WithAltScreen()`) to provide a full-screen experience and ensure clean terminal restoration on exit:

```go
p := tea.NewProgram(m, tea.WithAltScreen(), tea.WithMouseCellMotion())
```

This ensures the terminal is properly restored even on crashes or unexpected exits.

### Error Display

Errors from LLM API failures, network issues, tool execution failures, or stream errors are displayed as styled system messages in the viewport:

```go
func renderError(err error) string {
    style := lipgloss.NewStyle().Foreground(lipgloss.Color("196")) // red
    return style.Render(fmt.Sprintf("Error: %s", err.Error()))
}
```

After displaying the error, the model transitions to `statusIdle` so the user can retry.

## Implementation Plan

### Task 1: Add mode flag and config fields

**Files:** `vv/config/config.go`, `vv/config/config_test.go`

- Add `Mode string` and `CLI CLIConfig` fields to `Config`
- Add `CLIConfig` struct with `ConfirmTools []string`
- Add `VV_MODE` environment variable override in `Load()`
- Default `Mode` to `"cli"` in `applyDefaults()`
- Add tests for new config fields, defaults, and env override

### Task 2: Refactor agents.Create to accept tool.ToolRegistry interface

**Files:** `vv/agents/agents.go`

- Change `Create()` parameter type from `*tool.Registry` to `tool.ToolRegistry`
- This is backward-compatible since `*tool.Registry` satisfies `tool.ToolRegistry`
- Update any callers (main.go) -- no code change needed since the concrete type satisfies the interface

### Task 3: Update main.go to branch on mode

**Files:** `vv/main.go`

- Add `--mode` flag (default: `"cli"`)
- After agent creation, branch:
  - `"http"`: existing `service.New(...).Start(ctx)` path
  - `"cli"`: call `cli.New(...).Run(ctx)` (initially a stub)
- Pass the routing function, routes, and individual agents to the CLI (not just the router)
- Keep the `--addr` flag functional for HTTP mode

### Task 4: Create cli package with basic TUI skeleton

**Files:** `vv/cli/cli.go`, `vv/cli/messages.go`

- Implement `App` struct with `New()` and `Run()`
- Implement bubbletea `model` with:
  - `textarea` for input (Enter submits)
  - `viewport` for output (scrollable)
  - `spinner` for processing indicator
  - Window resize handling
  - Alternate screen mode (`tea.WithAltScreen()`)
- Redirect slog to file or discard before starting TUI
- Welcome message on startup
- `/exit` and `/quit` commands
- Ctrl+C handling (quit when idle)
- Store `tea.Program` reference for `p.Send()` from goroutines

**Dependencies:** `github.com/charmbracelet/bubbletea`, `github.com/charmbracelet/bubbles` (textarea, viewport, spinner), `github.com/charmbracelet/lipgloss`

### Task 5: Implement routing and agent invocation with streaming output

**Files:** `vv/cli/cli.go`, `vv/cli/messages.go`, `vv/cli/render.go`

- Implement `selectAgent()` using `routeragent.LLMFunc` to pick the target agent
- Call the selected agent's `RunStream()` directly (not `router.RunStream()`)
- Spawn goroutine to receive stream events, send as bubbletea messages via `app.program.Send()`
- Handle all event types in `Update`:
  - `EventAgentStart`: display agent indicator
  - `EventIterationStart`: display iteration progress
  - `EventTextDelta`: append to output, update viewport
  - `EventToolCallStart`: display tool name and arguments
  - `EventToolCallEnd`: display duration
  - `EventToolResult`: display result (truncated if long)
  - `EventTokenBudgetExhausted`: display budget warning
  - `EventAgentEnd`: finalize response, apply glamour rendering
  - `EventLLMCallStart/End/Error`: suppress or display based on verbosity
  - `EventError`: display as red styled system message
  - Unknown types: ignore
- Handle `streamDoneMsg`: display error if present, transition to idle
- Maintain `[]schema.Message` history for multi-turn context
- Implement Ctrl+C cancellation during processing

**Dependencies:** add `github.com/charmbracelet/glamour` for markdown rendering

### Task 6: Implement tool confirmation dialog

**Files:** `vv/cli/confirm.go`, `vv/cli/cli.go`

- Implement `confirmingExecutor` wrapping `tool.ToolRegistry` via embedding
  - Override only `Execute()`, delegating all other methods to the embedded registry
- Wire `confirmFn` to send a `confirmRequestMsg` via `app.program.Send()` and block on `confirmCh`
  - Use `select` on both `confirmCh` and `ctx.Done()` to prevent goroutine leaks
- In `model.Update`, handle the `confirmRequestMsg`:
  - Transition to `statusConfirming`
  - Create `huh.NewForm()` with `huh.NewConfirm()` (tool name, arguments summary)
  - Delegate `Update()`/`View()` to the huh form while confirming
  - On form completion, send result to `confirmCh`, transition back to `statusProcessing`
- Handle Ctrl+C during confirmation: send `false` to `confirmCh`, cancel run
- Integrate `confirmingExecutor` into agent creation when `ConfirmTools` is non-empty

**Dependencies:** `github.com/charmbracelet/huh`

### Task 7: Conversation history and scrolling

**Files:** `vv/cli/cli.go`, `vv/cli/render.go`

- Maintain `[]DisplayMessage` in the model for viewport rendering
- Maintain `[]schema.Message` in the App for multi-turn agent context
- On each agent invocation, pass full history in `RunRequest.Messages`
- Render full conversation in the viewport (user messages styled differently from agent messages)
- Auto-scroll to bottom on new content
- Support PgUp/PgDn and mouse wheel for scrolling (built into viewport)

### Task 8: Integration tests

**Files:** `vv/integrations/cli_test.go`

- Test mode flag parsing (default cli, explicit http, explicit cli)
- Test config loading with new fields
- Test `confirmingExecutor` behavior:
  - Approve: tool executes normally
  - Reject: tool returns rejection error
  - Non-confirm tool: passthrough without calling confirmFn
  - Context cancellation: confirmFn returns immediately without leak
- Test routing + streaming: mock LLM, verify that routing selects correct agent and full event stream is received
- Test multi-turn context: verify conversation history is passed to subsequent agent invocations
- Note: full TUI tests with bubbletea are best done via `teatest` package for programmatic interaction, but manual testing is the primary validation for the TUI UX

## Integration Test Plan

### Unit Tests

| Test | Package | Description |
|------|---------|-------------|
| `TestConfigMode` | `config` | Verify Mode defaults to "cli", env override works, YAML parsing works |
| `TestConfigCLI` | `config` | Verify CLIConfig.ConfirmTools parsed from YAML |
| `TestConfirmingExecutorApprove` | `cli` | Verify tool executes when confirmFn returns true |
| `TestConfirmingExecutorReject` | `cli` | Verify tool returns rejection error when confirmFn returns false |
| `TestConfirmingExecutorPassthrough` | `cli` | Verify non-confirmed tools execute without calling confirmFn |
| `TestConfirmingExecutorCancelledCtx` | `cli` | Verify confirmFn returns immediately on cancelled context |
| `TestConfirmingExecutorDelegation` | `cli` | Verify Register, List, Get, Merge delegate to inner registry |
| `TestSelectAgent` | `cli` | Verify routing selects correct agent via LLMFunc |
| `TestDisplayMessageRendering` | `cli` | Verify user/agent/tool/error messages render with correct styling |
| `TestExitCommands` | `cli` | Verify /exit and /quit are recognized |
| `TestMultiTurnHistory` | `cli` | Verify conversation history accumulates across turns |

### Integration Tests

| Test | Description |
|------|-------------|
| `TestCLIModeStartup` | Verify CLI mode initializes without error with valid config |
| `TestHTTPModeUnchanged` | Verify HTTP mode still works identically with `--mode http` |
| `TestCLIAgentInvocation` | Using a mock LLM, verify that sending a message produces full streaming events (text deltas, tool calls, agent end) |
| `TestCLIToolConfirmation` | Using a mock LLM that returns a tool call, verify the confirmation flow pauses and resumes correctly |
| `TestCLICancellation` | Verify that cancelling during processing terminates the stream gracefully without goroutine leaks |
| `TestCLIMultiTurn` | Verify that conversation history is passed to subsequent agent calls |

### Manual Test Checklist

- [ ] `vv` starts in CLI mode by default, displays welcome message
- [ ] `vv --mode http` starts HTTP server as before
- [ ] `vv --mode cli` explicitly starts CLI mode
- [ ] slog output does not corrupt the TUI (redirected to file or suppressed)
- [ ] Type a message, press Enter, see streaming output with text deltas
- [ ] Tool calls are displayed with name and arguments
- [ ] Tool results are displayed
- [ ] Iteration progress is displayed (e.g., "Iteration 2/10")
- [ ] Markdown output is formatted (bold, code blocks, lists) after response completes
- [ ] Viewport scrolls automatically on new content
- [ ] PgUp/PgDn scrolls through history
- [ ] Ctrl+C during processing cancels the current run, returns to idle
- [ ] Ctrl+C while idle exits the application
- [ ] Ctrl+C during confirmation dialog rejects and cancels the run
- [ ] `/exit` and `/quit` exit the application
- [ ] Terminal is restored to normal state after exit (alternate screen cleanup)
- [ ] Confirmation dialog appears for tools in `confirm_tools`
- [ ] Approving confirmation allows tool execution
- [ ] Rejecting confirmation prevents tool execution, agent adapts
- [ ] Window resize adjusts layout properly
- [ ] Errors (LLM failures, network errors) are displayed as red system messages
- [ ] Multi-turn conversation maintains context across turns
- [ ] Token budget exhaustion is displayed as a warning

## Dependencies

New Go module dependencies for `vv/go.mod`:

```
github.com/charmbracelet/bubbletea   v1.x
github.com/charmbracelet/bubbles     v0.x    (textarea, viewport, spinner)
github.com/charmbracelet/lipgloss    v1.x    (styling, used by bubbles)
github.com/charmbracelet/glamour     v0.x    (markdown rendering)
github.com/charmbracelet/huh         v0.x    (confirmation forms)
```

These are all part of the Charm ecosystem and are well-maintained, widely-used Go TUI libraries.
