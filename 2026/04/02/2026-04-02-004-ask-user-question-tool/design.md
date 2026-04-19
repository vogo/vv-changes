# Technical Design: `ask_user` Tool for Agent-Initiated User Interaction

## Architecture Overview

The `ask_user` tool enables agents to pause execution mid-task and ask the user a clarifying question. The user's free-form text response is returned as the tool result. The design introduces a **`UserInteractor` interface** at the `vage` framework level, with three concrete implementations wired at the `vv` application level based on run mode (CLI TUI, HTTP, non-interactive).

### Layered Architecture

```
vage/tool/askuser/        <-- Tool definition + UserInteractor interface + Register()
vage/schema/event.go      <-- New EventPendingInteraction event type + data struct

vv/cli/                   <-- CLI TUI implementation of UserInteractor (huh text input dialog)
vv/cli/askuser.go         <-- CLIInteractor for CLI mode
vv/httpapis/              <-- HTTP UserInteractor implementation + interaction store + callback endpoint
vv/configs/config.go      <-- New AskUserTimeout config field
vv/setup/setup.go         <-- Wire UserInteractor based on run mode
vv/agents/*.go            <-- System prompt updates for ask_user guidance
```

### Key Design Decisions

1. **UserInteractor interface lives in `vage/tool/askuser/`** -- This keeps the tool definition framework-level (like `bash`, `read`, etc.) while the concrete implementations live in `vv`. The interface is simple: one method that accepts a question string and returns the user's response string. The `NonInteractiveInteractor` provides a testable default at the framework level.

2. **The tool is registered into dispatchable agents' tool registries only** -- `ask_user` is registered for coder, researcher, and reviewer agents after profile-based tools are built, at the `setup.New()` stage. It is NOT registered for explorer (autonomous internal agent that should not stall the dispatch pipeline), planner (single-iteration, no tools), or chat (already in direct conversation with the user -- it can ask questions in its response text).

3. **CLI implementation follows the existing confirmation pattern** -- The `confirmingExecutor` wrapper in `cli/confirm.go` shows the established pattern: intercept tool execution, pause for user input via a channel, resume on response. The `ask_user` tool uses a similar channel-based pattern but with a text input dialog instead of a confirm dialog.

4. **HTTP implementation uses an interaction store with constructor-injected callback** -- A pending interaction is stored server-side. The HTTP interactor receives a `func(schema.Event)` callback at construction time (not via context injection) to emit the `pending_interaction` SSE event. The client POSTs the response to a callback endpoint in `vv/httpapis/`. No changes to `vage/service/` are needed.

5. **Non-interactive fallback is built into the tool** -- When constructed with a `NonInteractiveInteractor`, the tool returns a static fallback message immediately without blocking.

6. **HTTP sync endpoint uses non-interactive fallback** -- The sync `POST /v1/agents/{id}/run` endpoint cannot support mid-execution interaction, so it uses `NonInteractiveInteractor`, consistent with the `-p` flag behavior.

---

## Component Design

### 1. `UserInteractor` Interface (`vage/tool/askuser/askuser.go`)

```go
package askuser

import (
    "context"
    "github.com/vogo/vage/schema"
    "github.com/vogo/vage/tool"
)

// UserInteractor collects a free-form text response from a user.
// Implementations control how the question is presented and the response gathered
// (TUI dialog, HTTP callback, non-interactive fallback, etc.).
type UserInteractor interface {
    // AskUser presents the question to the user and returns their text response.
    // The context carries the configured timeout. If the context is canceled or
    // the timeout elapses, the implementation should return a timeout/fallback message
    // (not an error) so the agent can proceed.
    AskUser(ctx context.Context, question string) (string, error)
}
```

This file also contains:
- The `AskUserTool` struct holding a `UserInteractor` and timeout duration.
- `ToolDef()` returning the `schema.ToolDef` for registration.
- `Handler()` returning a `tool.ToolHandler` closure.
- `Register()` convenience function for registering into a `*tool.Registry`.
- Functional options: `WithTimeout(d time.Duration)`.

**Tool Definition:**

```go
const (
    ToolName = "ask_user"
    toolDescription = "Ask the user a clarifying question when the task is ambiguous " +
        "or critical information is missing. The user's free-form text response is " +
        "returned as the result. Use this sparingly -- only when the answer cannot " +
        "be reasonably inferred from context."
)

func (t *AskUserTool) ToolDef() schema.ToolDef {
    return schema.ToolDef{
        Name:        ToolName,
        Description: toolDescription,
        Source:      schema.ToolSourceLocal,
        Parameters: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "question": map[string]any{
                    "type":        "string",
                    "description": "The clarifying question to ask the user",
                },
            },
            "required":             []string{"question"},
            "additionalProperties": false,
        },
    }
}
```

**Handler Logic:**

```go
func (t *AskUserTool) Handler() tool.ToolHandler {
    return func(ctx context.Context, _, args string) (schema.ToolResult, error) {
        var parsed struct {
            Question string `json:"question"`
        }
        if err := json.Unmarshal([]byte(args), &parsed); err != nil {
            return schema.ErrorResult("", "ask_user: invalid arguments: "+err.Error()), nil
        }
        if parsed.Question == "" {
            return schema.ErrorResult("", "ask_user: question must not be empty"), nil
        }

        // Apply timeout.
        askCtx, cancel := context.WithTimeout(ctx, t.timeout)
        defer cancel()

        response, err := t.interactor.AskUser(askCtx, parsed.Question)
        if err != nil {
            return schema.ErrorResult("", "ask_user: "+err.Error()), nil
        }

        return schema.TextResult("", response), nil
    }
}
```

**Register function** (follows `bash.Register()` pattern):

```go
func Register(registry *tool.Registry, interactor UserInteractor, opts ...Option) error {
    t := New(interactor, opts...)
    return registry.RegisterIfAbsent(t.ToolDef(), t.Handler())
}
```

### 2. Non-Interactive Interactor (`vage/tool/askuser/noninteractive.go`)

```go
// NonInteractiveInteractor returns a static fallback message without blocking.
type NonInteractiveInteractor struct{}

func (NonInteractiveInteractor) AskUser(_ context.Context, _ string) (string, error) {
    return "Running in non-interactive mode. No user available to answer questions. " +
        "Proceed with your best judgment.", nil
}
```

### 3. CLI TUI Interactor (`vv/cli/askuser.go`)

The CLI interactor follows the same channel-based pattern used by `confirmingExecutor`. It sends a Bubble Tea message to trigger a dialog, then blocks on a response channel.

**CLIInteractor struct:**

```go
type CLIInteractor struct {
    program *tea.Program // set after tea.NewProgram() via SetProgram()
    respCh  chan string
}

func NewCLIInteractor() *CLIInteractor {
    return &CLIInteractor{
        respCh: make(chan string, 1),
    }
}

// SetProgram wires the tea.Program reference. Must be called from App.Run()
// after creating the program and before any agent execution.
func (c *CLIInteractor) SetProgram(p *tea.Program) {
    c.program = p
}

func (c *CLIInteractor) AskUser(ctx context.Context, question string) (string, error) {
    if c.program == nil {
        return "User interaction not available.", nil
    }
    c.program.Send(askUserRequestMsg{question: question})
    select {
    case resp := <-c.respCh:
        return resp, nil
    case <-ctx.Done():
        return "User did not respond within the timeout. " +
            "Proceed with your best judgment.", nil
    }
}
```

**New Bubble Tea message:**

```go
// askUserRequestMsg signals that the agent wants to ask the user a question.
type askUserRequestMsg struct {
    question string
}
```

**New session status:**

```go
const (
    statusIdle       sessionStatus = iota
    statusProcessing
    statusConfirming
    statusAskingUser   // <-- new
    statusQuitting
)
```

**Model changes in `cli/cli.go`:**

The `model` struct gains:
- `askUserCh chan string` -- buffered channel for the user's text response (same as `CLIInteractor.respCh`).
- `askUserForm *huh.Form` -- the text input form.
- `askUserQuestion string` -- the displayed question text.

**Dialog implementation:**

When the agent calls `ask_user`, the tool handler sends an `askUserRequestMsg` to the TUI program. The `Update()` method handles this message by:

1. Setting `status = statusAskingUser`.
2. Creating a `huh.Form` with a `huh.NewText()` (multi-line) field and a title showing the question.
3. When the form completes, sending the response text to `askUserCh`.
4. Transitioning back to `statusProcessing`.

**View rendering:** The `View()` method adds a case for `statusAskingUser` that renders the `askUserForm`, similar to the existing `statusConfirming` case.

**Ctrl+C handling:** When status is `statusAskingUser`, Ctrl+C calls `m.runCancel()` to cancel the context and transitions to `statusIdle`. The interactor's `AskUser()` returns via the `<-ctx.Done()` path with the timeout fallback message. No data is sent on `askUserCh`.

### 4. HTTP Interactor (`vv/httpapis/askuser.go`)

**Interaction store (`vv/httpapis/interactions.go`):**

```go
type Interaction struct {
    ID        string    `json:"id"`
    Question  string    `json:"question"`
    Response  string    `json:"response,omitempty"`
    Responded bool      `json:"responded"`
    CreatedAt time.Time `json:"created_at"`
    done      chan struct{} // closed when response is submitted
}

type InteractionStore struct {
    mu           sync.Mutex
    interactions map[string]*Interaction
    timeout      time.Duration
}
```

The store provides:
- `Create(question string) *Interaction` -- generates a UUID-based ID, stores the interaction.
- `Respond(id, response string) error` -- submits the response (409 if already responded, 404 if not found).
- `Get(id string) (*Interaction, bool)` -- retrieves an interaction.
- Automatic cleanup of expired interactions via a background goroutine.

**Lifecycle management:** `NewInteractionStore()` accepts a `context.Context`. The cleanup goroutine runs a `time.Ticker` (interval = `timeout`), selecting on `ctx.Done()` for graceful shutdown. Interactions older than `2 * timeout` are removed.

**HTTP Interactor:**

```go
type HTTPInteractor struct {
    store   *InteractionStore
    emitFn  func(schema.Event) // emit SSE event; set at construction time
}

func NewHTTPInteractor(store *InteractionStore, emitFn func(schema.Event)) *HTTPInteractor {
    return &HTTPInteractor{store: store, emitFn: emitFn}
}

func (h *HTTPInteractor) AskUser(ctx context.Context, question string) (string, error) {
    interaction := h.store.Create(question)

    // Emit pending_interaction event via the constructor-injected callback.
    if h.emitFn != nil {
        h.emitFn(schema.NewEvent(schema.EventPendingInteraction, "", "", schema.PendingInteractionData{
            InteractionID:  interaction.ID,
            Question:       question,
            TimeoutSeconds: int(h.store.timeout.Seconds()),
        }))
    }

    select {
    case <-interaction.done:
        return interaction.Response, nil
    case <-ctx.Done():
        return "User did not respond within the timeout. " +
            "Proceed with your best judgment.", nil
    }
}
```

**Stream integration:** The `emitFn` callback is injected at construction. In `vv/httpapis/http.go`, when setting up the HTTP interactor for streaming endpoints, the callback writes directly to the SSE event channel. For the sync endpoint (`handleRun`), a `NonInteractiveInteractor` is used instead, so no stream integration is needed.

**New endpoint in `vv/httpapis/http.go`:**

```
POST /v1/interactions/{interactionID}/respond
```

Request body: `{"response": "user's text"}`
Responses: 200 OK, 404 Not Found, 409 Conflict.

This endpoint is added to the custom mux in `vv/httpapis/http.go` alongside the memory endpoints.

### 5. New Event Type (`vage/schema/event.go`)

```go
const EventPendingInteraction = "pending_interaction"

type PendingInteractionData struct {
    InteractionID  string `json:"interaction_id"`
    Question       string `json:"question"`
    TimeoutSeconds int    `json:"timeout_seconds"`
}

func (PendingInteractionData) eventData() {}
```

### 6. Configuration (`vv/configs/config.go`)

Add to `Config` (top-level, shared across CLI and HTTP modes):

```go
type Config struct {
    // ... existing fields ...
    AskUserTimeout int `yaml:"ask_user_timeout"` // seconds, default 300 (5 minutes)
}
```

In `applyDefaults()`:

```go
if cfg.AskUserTimeout == 0 {
    cfg.AskUserTimeout = 300 // 5 minutes
}
```

This follows the same pattern as `BashTimeout int` -- integer seconds in YAML, converted to `time.Duration` at usage sites.

### 7. Registration Wiring (`vv/setup/setup.go`)

The `ask_user` tool is registered into dispatchable agents' tool registries (coder, researcher, reviewer) but NOT for explorer, planner, or chat.

The `Options` struct gains:

```go
type Options struct {
    WrapToolRegistry func(*tool.Registry) tool.ToolRegistry
    UserInteractor   askuser.UserInteractor // <-- new
    AskUserTimeout   time.Duration          // <-- new
}
```

In the dispatchable agent-building loop:

```go
for _, desc := range reg.Dispatchable() {
    toolReg, err := desc.ToolProfile.BuildRegistry(cfg.Tools)
    if err != nil { ... }

    // Register ask_user for agents that should have it.
    // Skip chat (direct conversation; no need for ask_user tool).
    if opts != nil && opts.UserInteractor != nil && desc.ID != "chat" {
        askuserTool := askuser.New(opts.UserInteractor, askuser.WithTimeout(opts.AskUserTimeout))
        _ = askuserTool.Register(toolReg)
    }

    // Apply optional tool registry wrapping (e.g., CLI confirmation).
    var finalToolReg tool.ToolRegistry = toolReg
    if opts != nil && opts.WrapToolRegistry != nil {
        finalToolReg = opts.WrapToolRegistry(toolReg)
    }
    // ... rest of agent creation
}
```

In `main.go`, construct the appropriate interactor:

```go
// For CLI interactive mode:
interactor := cli.NewCLIInteractor() // wired to the TUI later via SetProgram()

// For -p (non-interactive) mode:
interactor := &askuser.NonInteractiveInteractor{}

// For HTTP mode:
interactionStore := httpapis.NewInteractionStore(ctx, askUserTimeout)
interactor := httpapis.NewHTTPInteractor(interactionStore, nil) // emitFn set per-request for streaming
```

In `cli/cli.go` `App.Run()`, after `tea.NewProgram()`:

```go
p := tea.NewProgram(m)
a.program = p
a.interactor.SetProgram(p) // wire the program reference
```

### 8. System Prompt Updates (`vv/agents/*.go`)

Add the following section to coder, researcher, and reviewer agent system prompts:

```
## Clarifying Questions
- **ask_user**: Ask the user a clarifying question when you encounter ambiguity. The user's text response is returned as the result.

Use ask_user when:
- The user's instruction is ambiguous and multiple interpretations exist.
- Multiple valid approaches exist and the choice significantly affects the outcome.
- A destructive or irreversible action is about to be taken and the intent is unclear.
- Critical information (file paths, variable names, scope) is missing and cannot be reasonably inferred.

Do NOT use ask_user when:
- The answer can be reasonably inferred from context.
- The question is trivial or would interrupt flow unnecessarily.
- You have already asked a question in the current turn.
```

The chat agent does NOT get this prompt addition (it has no `ask_user` tool).

---

## Data Models / Schemas

### New: `AskUserTool` (`vage/tool/askuser/`)

| Field | Type | Description |
|-------|------|-------------|
| interactor | UserInteractor | Implementation for user interaction |
| timeout | time.Duration | Timeout for user response (default 5m) |

### New: `Interaction` (`vv/httpapis/`)

| Field | Type | Description |
|-------|------|-------------|
| ID | string | Unique interaction ID (UUID) |
| Question | string | The agent's question text |
| Response | string | The user's response (empty until responded) |
| Responded | bool | Whether a response has been submitted |
| CreatedAt | time.Time | When the interaction was created |
| done | chan struct{} | Closed when response is submitted (internal) |

### Modified: `Config` (`vv/configs/`)

| New Field | Type | Default | Description |
|-----------|------|---------|-------------|
| AskUserTimeout | int (seconds) | 300 | Timeout for ask_user responses |

### New: `PendingInteractionData` (`vage/schema/`)

| Field | Type | Description |
|-------|------|-------------|
| InteractionID | string | Unique interaction identifier |
| Question | string | The question text |
| TimeoutSeconds | int | Remaining timeout in seconds |

### New: session status value (`vv/cli/`)

| Constant | Value | Description |
|----------|-------|-------------|
| statusAskingUser | 3 (iota) | TUI is displaying an ask_user dialog |

---

## API Contracts

### New Endpoint: Submit Interaction Response

```
POST /v1/interactions/{interactionID}/respond
Content-Type: application/json

Request:
{
  "response": "The user's text response"
}

Responses:
  200 OK              -- Response accepted, agent resumes
  404 Not Found       -- Interaction ID not found or expired
  409 Conflict        -- Response already submitted
```

### New SSE Event Type

```
event: pending_interaction
data: {
  "type": "pending_interaction",
  "interaction_id": "abc123",
  "question": "Which database migration strategy do you prefer?",
  "timeout_seconds": 300
}
```

---

## Implementation Plan

### Task 1: Add `ask_user` tool definition and `UserInteractor` interface (`vage`)

**Files:**
- Create `vage/tool/askuser/askuser.go` -- `UserInteractor` interface, `AskUserTool` struct, `ToolDef()`, `Handler()`, `New()`, `Register()`, functional options (`WithTimeout`)
- Create `vage/tool/askuser/noninteractive.go` -- `NonInteractiveInteractor`
- Create `vage/tool/askuser/askuser_test.go` -- Unit tests for tool handler (mock interactor), non-interactive interactor, empty question validation, timeout behavior

**Depends on:** Nothing.

### Task 2: Add `PendingInteractionData` event type (`vage`)

**Files:**
- Edit `vage/schema/event.go` -- Add `EventPendingInteraction` constant, `PendingInteractionData` struct with `eventData()` marker

**Depends on:** Nothing.

### Task 3: Add `AskUserTimeout` configuration (`vv`)

**Files:**
- Edit `vv/configs/config.go` -- Add `AskUserTimeout int` field to `Config`, add default in `applyDefaults()` (300 seconds)
- Edit `vv/configs/config_test.go` -- Test default application

**Depends on:** Nothing.

### Task 4: Wire `ask_user` tool into agent setup (`vv`)

**Files:**
- Edit `vv/setup/setup.go` -- Add `UserInteractor` and `AskUserTimeout` to `Options` struct. After building each dispatchable agent's tool registry (except chat), register `ask_user` tool.

**Depends on:** Task 1, Task 3.

### Task 5: Implement CLI TUI interactor (`vv`)

**Files:**
- Create `vv/cli/askuser.go` -- `CLIInteractor` struct with `AskUser()` method, `SetProgram()` method, `NewCLIInteractor()`, `askUserRequestMsg` type
- Edit `vv/cli/messages.go` -- Add `statusAskingUser` constant
- Edit `vv/cli/cli.go`:
  - Add `askUserCh chan string`, `askUserForm *huh.Form`, `askUserQuestion string` to `model` struct
  - Add `handleAskUserRequest()` method (creates huh form with text input)
  - Add `statusAskingUser` case in `View()` to render the form
  - Add `statusAskingUser` handling in `Update()` for form updates and completion
  - Add `statusAskingUser` case in `handleCtrlC()` (cancel context via `runCancel()`, transition to idle)
  - Initialize `askUserCh` in `newModel()`
  - Wire `CLIInteractor.SetProgram()` in `App.Run()` after `tea.NewProgram()`
- Edit `vv/main.go` -- Create `CLIInteractor` and pass to `setup.Options` for interactive CLI mode; use `NonInteractiveInteractor` for `-p` mode

**Depends on:** Task 1, Task 4.

### Task 6: Implement HTTP interactor and callback endpoint (`vv`)

**Files:**
- Create `vv/httpapis/interactions.go` -- `InteractionStore` struct with context-based lifecycle, `Create()`, `Respond()`, `Get()`, cleanup goroutine
- Create `vv/httpapis/askuser.go` -- `HTTPInteractor` struct implementing `UserInteractor`, uses `InteractionStore` and constructor-injected `emitFn`
- Edit `vv/httpapis/http.go` -- Add `POST /v1/interactions/{interactionID}/respond` handler to the custom mux, create `InteractionStore` in `Serve()`, wire `HTTPInteractor`
- Create `vv/httpapis/interactions_test.go` -- Unit tests for store operations (create, respond, double-respond 409, not-found 404, expiry)

**Depends on:** Task 1, Task 2, Task 4.

### Task 7: Update agent system prompts (`vv`)

**Files:**
- Edit `vv/agents/coder.go` -- Append `ask_user` guidance to `CoderSystemPrompt`
- Edit `vv/agents/researcher.go` -- Append to `ResearcherSystemPrompt`
- Edit `vv/agents/reviewer.go` -- Append to `ReviewerSystemPrompt`
- Edit existing tests if prompt content is asserted

**Depends on:** Nothing (can be done in parallel with other tasks).

---

## Integration Test Plan

### Test 1: CLI Interactive `ask_user` (Manual / E2E)

**Location:** `vv/integrations/cli_tests/`

- Start vv in CLI mode with a mock LLM that always calls `ask_user`.
- Verify the TUI displays the question dialog.
- Type a response and submit.
- Verify the agent receives the response as the tool result and continues.

### Test 2: CLI Non-Interactive `ask_user`

**Location:** `vv/integrations/cli_tests/`

- Run `vv -p "ambiguous prompt"` with a mock LLM that calls `ask_user`.
- Verify the tool returns the non-interactive fallback message.
- Verify the agent continues without hanging.

### Test 3: HTTP Streaming `ask_user`

**Location:** `vv/integrations/http_tests/` or `vv/httpapis/`

- Start the HTTP service with an agent that calls `ask_user`.
- Send a `POST /v1/agents/{id}/stream` request.
- Verify a `pending_interaction` SSE event is emitted with an interaction ID and the question.
- `POST /v1/interactions/{id}/respond` with a response.
- Verify the agent resumes and the stream continues with subsequent events.
- Verify the tool result contains the submitted response.

### Test 4: HTTP Sync Endpoint Fallback

**Location:** `vv/httpapis/`

- Send a `POST /v1/agents/{id}/run` request with an agent that calls `ask_user`.
- Verify the tool returns the non-interactive fallback message (no blocking).

### Test 5: HTTP Interaction Timeout

**Location:** `vv/httpapis/`

- Create a pending interaction via the HTTP interactor with a short timeout (1 second).
- Do NOT submit a response.
- Verify the tool returns a timeout fallback message after the timeout.
- Verify the agent continues execution.

### Test 6: HTTP Double-Response (409)

**Location:** `vv/httpapis/`

- Create a pending interaction, submit a response.
- Submit a second response to the same interaction ID.
- Verify 409 Conflict is returned.

### Test 7: HTTP Expired Interaction (404)

**Location:** `vv/httpapis/`

- Request `POST /v1/interactions/{nonexistent}/respond`.
- Verify 404 Not Found.

### Test 8: Tool Registration Across Agents

**Location:** `vv/setup/`

- Initialize setup with a mock interactor.
- Verify coder, researcher, and reviewer tool registries contain the `ask_user` tool.
- Verify chat, explorer, and planner tool registries do NOT contain the `ask_user` tool.

### Test 9: `ask_user` Tool Unit Tests

**Location:** `vage/tool/askuser/`

- Test valid invocation with mock interactor returns user's response as `TextResult`.
- Test empty question returns error result.
- Test invalid JSON args returns error result.
- Test `NonInteractiveInteractor` returns fallback message immediately.
- Test timeout behavior: mock interactor that blocks; verify timeout message is returned after deadline.
