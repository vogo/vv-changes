# CLI Interactive Mode - Technical Design

## Architecture Overview

The CLI interactive mode adds a terminal user interface (TUI) as an alternative entry point to vv, alongside the existing HTTP server mode. The design introduces a new `cli` package under `vv/cli/` that contains the bubbletea TUI application. The core principle is **in-process agent invocation** -- the CLI calls the router agent directly via `agent.StreamAgent.RunStream()`, consuming the same `schema.RunStream` event stream that the HTTP SSE handler uses.

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
                            +-- router.RunStream(ctx, req)
                            +-- Consume RunStream events
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
    confirm.go         -- Tool call confirmation dialog using huh
  config/
    config.go          -- updated: add Mode and ConfirmTools fields
  agents/              -- unchanged
  tools/               -- unchanged
```

### cli.App

The central struct that owns the bubbletea program and agent references.

```go
package cli

// App holds the CLI TUI application state.
type App struct {
    router    agent.StreamAgent   // the router agent (implements StreamAgent via RunToStream)
    cfg       *config.Config
    sessionID string
    messages  []DisplayMessage    // conversation history for the viewport
}

// New creates a new CLI App.
func New(router agent.StreamAgent, cfg *config.Config) *App

// Run starts the bubbletea program and blocks until exit.
func (a *App) Run(ctx context.Context) error
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
    confirmCh chan bool            // send user's confirm/reject decision
    pendingTC *schema.ToolCallStartData  // tool call awaiting confirmation
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
    Role      string    // "user", "agent", "system", "tool"
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

### Session Model (In-Memory Only)

The CLI session is entirely in-memory within the `App` struct. No new persistent data model is needed. The `sessionID` is generated with `crypto/rand` at startup (matching the existing pattern in `service.go`). Conversation history is held as `[]DisplayMessage` in the model struct.

## Event Flow

### Message Processing Flow

1. User types message in textarea, presses Enter
2. Model transitions to `statusProcessing`, disables textarea
3. `model.Update` sends a `tea.Cmd` that:
   a. Creates a `schema.RunRequest` with the user's text
   b. Calls `router.RunStream(ctx, req)` to get a `*schema.RunStream`
   c. Enters a receive loop calling `stream.Recv()`
   d. For each event, sends a `streamEventMsg` back to the bubbletea program via `p.Send()`
   e. On `io.EOF`, sends `streamDoneMsg{}`
4. `model.Update` handles each `streamEventMsg`:
   - `EventTextDelta`: append delta text to the output builder, update viewport
   - `EventToolCallStart`: if tool is in `confirm_tools`, transition to `statusConfirming` and show confirmation dialog; otherwise display tool call info
   - `EventToolCallEnd`: display duration
   - `EventToolResult`: display result summary
   - `EventAgentEnd`: finalize the response message
5. On `streamDoneMsg`, transition back to `statusIdle`, re-enable textarea

### Tool Confirmation Flow

The confirmation mechanism uses a channel-based callback pattern. Instead of modifying the vage framework, the CLI wraps the tool registry with a confirming decorator.

```go
// confirmingRegistry wraps a tool.ToolRegistry and pauses execution
// for tools that require user confirmation.
type confirmingRegistry struct {
    inner        tool.ToolRegistry
    confirmTools map[string]bool
    confirmFn    func(ctx context.Context, toolName, args string) (bool, error)
}

func (r *confirmingRegistry) Execute(ctx context.Context, name, args string) (schema.ToolResult, error) {
    if r.confirmTools[name] {
        approved, err := r.confirmFn(ctx, name, args)
        if err != nil {
            return schema.ErrorResult("", err.Error()), nil
        }
        if !approved {
            return schema.ErrorResult("", "Tool call rejected by user"), nil
        }
    }
    return r.inner.Execute(ctx, name, args)
}
```

The `confirmFn` is provided by the TUI model. It:
1. Sends a bubbletea message to show the confirmation dialog
2. Blocks on a channel (`confirmCh`) waiting for the user's decision
3. Returns the decision

This works because `RunStream` runs the producer in a separate goroutine, so blocking in `confirmFn` does not freeze the TUI event loop.

The confirmation dialog itself uses `huh` to render an inline confirmation form within the bubbletea program, following the huh bubbletea integration pattern (huh fields implement `tea.Model`).

### Cancellation Flow

- First Ctrl+C: if `statusProcessing`, cancel the stream context (via the per-run `context.CancelFunc`), which causes the producer goroutine to terminate. The model transitions back to `statusIdle`.
- First Ctrl+C while `statusIdle`: transition to `statusQuitting`, send `tea.Quit`.
- Second Ctrl+C during processing: immediate quit via `tea.Quit`.

### Exit Commands

`/exit` and `/quit` typed as input trigger `tea.Quit`. These are detected in `Update` before submitting to the agent.

## API Contracts

No new HTTP APIs. The CLI mode bypasses HTTP entirely.

The interface contract between the CLI and the agent layer is the existing `agent.StreamAgent` interface:

```go
RunStream(ctx context.Context, req *schema.RunRequest) (*schema.RunStream, error)
```

The router agent already implements `StreamAgent` (via `RunToStream`), and the task agents implement it natively. No changes to the vage framework are required.

## Markdown Rendering

For terminal markdown rendering, use `github.com/charmbracelet/glamour`. This is part of the Charm ecosystem alongside bubbletea and integrates naturally. The renderer is created once at startup with a terminal-appropriate style (auto-detected dark/light).

```go
renderer, _ := glamour.NewTermRenderer(
    glamour.WithAutoStyle(),
    glamour.WithWordWrap(width),
)
```

Agent text output is accumulated per-response and rendered through glamour before display in the viewport. During streaming, raw text is displayed; glamour rendering is applied when the response completes (re-rendering the full response) to avoid partial-markdown rendering artifacts.

## Implementation Plan

### Task 1: Add mode flag and config fields

**Files:** `vv/config/config.go`, `vv/config/config_test.go`

- Add `Mode string` and `CLI CLIConfig` fields to `Config`
- Add `CLIConfig` struct with `ConfirmTools []string`
- Add `VV_MODE` environment variable override in `Load()`
- Default `Mode` to `"cli"` in `applyDefaults()`
- Add tests for new config fields, defaults, and env override

### Task 2: Update main.go to branch on mode

**Files:** `vv/main.go`

- Add `--mode` flag (default: `"cli"`)
- After agent creation, branch:
  - `"http"`: existing `service.New(...).Start(ctx)` path
  - `"cli"`: call `cli.New(...).Run(ctx)` (initially a stub)
- Remove the `--addr` flag default override when in CLI mode (addr is irrelevant)
- Keep the `--addr` flag functional for HTTP mode

### Task 3: Create cli package with basic TUI skeleton

**Files:** `vv/cli/cli.go`, `vv/cli/messages.go`

- Implement `App` struct with `New()` and `Run()`
- Implement bubbletea `model` with:
  - `textarea` for input (single-line initially, Enter submits)
  - `viewport` for output (scrollable)
  - `spinner` for processing indicator
  - Window resize handling
- Welcome message on startup
- `/exit` and `/quit` commands
- Ctrl+C handling (quit when idle)

**Dependencies:** `github.com/charmbracelet/bubbletea`, `github.com/charmbracelet/bubbles` (textarea, viewport, spinner)

### Task 4: Implement agent invocation and streaming output

**Files:** `vv/cli/cli.go`, `vv/cli/messages.go`, `vv/cli/render.go`

- On Enter, create `RunRequest` and call `router.RunStream()`
- Spawn goroutine to receive stream events, send as bubbletea messages
- Handle `streamEventMsg` in `Update`:
  - `EventTextDelta`: append to output, update viewport
  - `EventToolCallStart`: display tool name and arguments
  - `EventToolCallEnd`: display duration
  - `EventToolResult`: display result (truncated if long)
  - `EventAgentEnd`: finalize response, add to conversation history
  - `EventError`: display error
- Handle `streamDoneMsg`: transition to idle
- Implement Ctrl+C cancellation during processing

**Dependencies:** add `github.com/charmbracelet/glamour` for markdown rendering

### Task 5: Implement tool confirmation dialog

**Files:** `vv/cli/confirm.go`, `vv/cli/cli.go`

- Implement `confirmingRegistry` wrapper
- Wire `confirmFn` to send a bubbletea message and block on `confirmCh`
- In `model.Update`, handle the confirmation message:
  - Transition to `statusConfirming`
  - Show huh confirm form (tool name, arguments summary)
  - On confirm/reject, send result to `confirmCh`
- Integrate `confirmingRegistry` into agent creation when `ConfirmTools` is non-empty

**Dependencies:** `github.com/charmbracelet/huh`

### Task 6: Conversation history and scrolling

**Files:** `vv/cli/cli.go`, `vv/cli/render.go`

- Maintain `[]DisplayMessage` in the model
- Render full conversation in the viewport (user messages styled differently from agent messages)
- Auto-scroll to bottom on new content
- Support PgUp/PgDn and mouse wheel for scrolling (built into viewport)

### Task 7: Integration tests

**Files:** `vv/integrations/cli_test.go`

- Test mode flag parsing (default cli, explicit http, explicit cli)
- Test config loading with new fields
- Test `confirmingRegistry` behavior (approve, reject, non-confirm tool passthrough)
- Note: full TUI tests with bubbletea are best done via `teatest` package for programmatic interaction, but manual testing is the primary validation for the TUI UX

## Integration Test Plan

### Unit Tests

| Test | Package | Description |
|------|---------|-------------|
| `TestConfigMode` | `config` | Verify Mode defaults to "cli", env override works, YAML parsing works |
| `TestConfigCLI` | `config` | Verify CLIConfig.ConfirmTools parsed from YAML |
| `TestConfirmingRegistryApprove` | `cli` | Verify tool executes when confirmFn returns true |
| `TestConfirmingRegistryReject` | `cli` | Verify tool returns rejection error when confirmFn returns false |
| `TestConfirmingRegistryPassthrough` | `cli` | Verify non-confirmed tools execute without calling confirmFn |
| `TestDisplayMessageRendering` | `cli` | Verify user/agent/tool messages render with correct styling |
| `TestExitCommands` | `cli` | Verify /exit and /quit are recognized |

### Integration Tests

| Test | Description |
|------|-------------|
| `TestCLIModeStartup` | Verify CLI mode initializes without error with valid config |
| `TestHTTPModeUnchanged` | Verify HTTP mode still works identically with `--mode http` |
| `TestCLIAgentInvocation` | Using a mock LLM, verify that sending a message through the CLI path produces streaming events and a final response |
| `TestCLIToolConfirmation` | Using a mock LLM that returns a tool call, verify the confirmation flow pauses and resumes correctly |
| `TestCLICancellation` | Verify that cancelling during processing terminates the stream gracefully |

### Manual Test Checklist

- [ ] `vv` starts in CLI mode by default, displays welcome message
- [ ] `vv --mode http` starts HTTP server as before
- [ ] `vv --mode cli` explicitly starts CLI mode
- [ ] Type a message, press Enter, see streaming output
- [ ] Tool calls are displayed with name and arguments
- [ ] Tool results are displayed
- [ ] Markdown output is formatted (bold, code blocks, lists)
- [ ] Viewport scrolls automatically on new content
- [ ] PgUp/PgDn scrolls through history
- [ ] Ctrl+C during processing cancels the current run
- [ ] Ctrl+C while idle exits the application
- [ ] `/exit` and `/quit` exit the application
- [ ] Terminal is restored to normal state after exit
- [ ] Confirmation dialog appears for tools in `confirm_tools`
- [ ] Approving confirmation allows tool execution
- [ ] Rejecting confirmation prevents tool execution, agent adapts
- [ ] Window resize adjusts layout properly

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
