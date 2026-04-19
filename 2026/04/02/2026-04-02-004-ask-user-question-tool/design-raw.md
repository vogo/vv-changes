# Technical Design: `ask_user` Tool for Agent-Initiated User Interaction

## Architecture Overview

The `ask_user` tool enables agents to pause execution mid-task and ask the user a clarifying question. The user's free-form text response is returned as the tool result. The design introduces a **`UserInteractor` interface** at the `vage` framework level, with three concrete implementations wired at the `vv` application level based on run mode (CLI TUI, HTTP, non-interactive).

### Layered Architecture

```
vage/tool/askuser/        <-- Tool definition + UserInteractor interface + Register()
vage/schema/event.go      <-- New EventPendingInteraction event type + data struct
vage/service/             <-- Interaction store + HTTP callback endpoint

vv/cli/                   <-- CLI TUI implementation of UserInteractor (huh text input dialog)
vv/cli/askuser.go         <-- askUserInteractor for CLI mode
vv/httpapis/              <-- HTTP UserInteractor implementation + callback endpoint wiring
vv/configs/config.go      <-- New AskUserTimeout config field
vv/setup/setup.go         <-- Wire UserInteractor based on run mode
vv/agents/*.go            <-- System prompt updates for ask_user guidance
```

### Key Design Decisions

1. **UserInteractor interface lives in `vage/tool/askuser/`** -- This keeps the tool definition framework-level (like `bash`, `read`, etc.) while the concrete implementations live in `vv`. The interface is simple: one method that accepts a question string and returns the user's response string.

2. **The tool is registered into every agent's tool registry** -- Rather than adding it as a new ToolCapability, `ask_user` is a cross-cutting "meta-tool" that should be available regardless of tool profile. It is registered after profile-based tools, at the `setup.New()` stage.

3. **CLI implementation follows the existing confirmation pattern** -- The `confirmingExecutor` wrapper in `cli/confirm.go` shows the established pattern: intercept tool execution, pause for user input via a channel, resume on response. The `ask_user` tool uses a similar channel-based pattern but with a text input dialog instead of a confirm dialog.

4. **HTTP implementation uses an interaction store** -- A pending interaction is stored server-side, a `pending_interaction` SSE event is emitted, and the client POSTs the response to a callback endpoint. This is consistent with the existing async task pattern.

5. **Non-interactive fallback is built into the tool** -- When constructed with a `NonInteractiveInteractor`, the tool returns a static fallback message immediately without blocking.

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

**New Bubble Tea messages:**

```go
// askUserRequestMsg signals that the agent wants to ask the user a question.
type askUserRequestMsg struct {
    question string
}

// askUserResponseMsg carries the user's response back (not a tea.Msg -- internal).
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
- `askUserCh chan string` -- buffered channel for the user's text response.
- `askUserForm *huh.Form` -- the text input form.
- `askUserQuestion string` -- the displayed question text.

**Dialog implementation:**

When the agent calls `ask_user`, the tool handler sends an `askUserRequestMsg` to the TUI program. The `Update()` method handles this message by:

1. Setting `status = statusAskingUser`.
2. Creating a `huh.Form` with a `huh.NewText()` (multi-line) or `huh.NewInput()` field and a title showing the question.
3. When the form completes, sending the response text to `askUserCh`.
4. Transitioning back to `statusProcessing`.

The interactor implementation blocks on `askUserCh` (or context cancellation for timeout).

```go
type cliInteractor struct {
    program *tea.Program
    respCh  chan string
}

func (c *cliInteractor) AskUser(ctx context.Context, question string) (string, error) {
    c.program.Send(askUserRequestMsg{question: question})
    select {
    case resp := <-c.respCh:
        return resp, nil
    case <-ctx.Done():
        return fmt.Sprintf("User did not respond within the timeout. "+
            "Proceed with your best judgment."), nil
    }
}
```

**View rendering:** The `View()` method adds a case for `statusAskingUser` that renders the `askUserForm`, similar to the existing `statusConfirming` case.

**Ctrl+C handling:** When status is `statusAskingUser`, Ctrl+C sends a cancellation message to `askUserCh` and returns to `statusProcessing`. The tool returns the timeout/fallback message.

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
}
```

The store provides:
- `Create(question string) *Interaction` -- generates a UUID-based ID, stores the interaction.
- `Respond(id, response string) error` -- submits the response (409 if already responded, 404 if not found).
- `Get(id string) (*Interaction, bool)` -- retrieves an interaction.
- Automatic cleanup of expired interactions (older than 2x timeout).

**HTTP Interactor:**

```go
type httpInteractor struct {
    store   *InteractionStore
    emitFn  func(schema.Event) // emit SSE event to the stream
}

func (h *httpInteractor) AskUser(ctx context.Context, question string) (string, error) {
    interaction := h.store.Create(question)

    // Emit pending_interaction event.
    // This event is injected into the agent's stream via the emitFn callback.
    // (See "Stream integration" below for how this is wired.)

    select {
    case <-interaction.done:
        return interaction.Response, nil
    case <-ctx.Done():
        return "User did not respond within the timeout. " +
            "Proceed with your best judgment.", nil
    }
}
```

**Stream integration challenge:** The HTTP interactor needs to emit events into the agent's SSE stream. Since the tool handler runs inside the agent's execution loop (which produces stream events), the interactor does not directly access the SSE writer. Instead, the `ask_user` tool handler emits a new event type into the stream, and the service layer's SSE writer forwards it to the client.

The approach: rather than injecting a callback into the interactor, the HTTP interactor uses a simpler design -- it creates the interaction in the store and returns a `pending_interaction` event as a special schema.Event. The `handleStream` handler in `vage/service/handler.go` already serializes all events to SSE; we add the new event type to `schema/event.go` so it flows through naturally.

However, the tool handler cannot directly emit stream events. Instead, the HTTP interactor:
1. Creates the interaction in the store.
2. The tool handler, before calling `AskUser()`, emits a stream event via context-injected send function -- but this is complex.

**Simpler approach:** The HTTP interactor embeds a reference to the `*tea.Program`-equivalent (the SSE event channel). But since the tool handler runs on the agent's goroutine and the stream events are emitted by the TaskAgent, the cleanest solution is:

- The `ask_user` tool handler itself returns a stream event. But tool handlers return `ToolResult`, not events.

**Chosen approach:** Use a **context-based event emitter**. The `vage` framework already passes `context.Context` to tool handlers. We define a context key that carries an optional event sender function. The HTTP service layer injects this sender into the context before agent execution. The `ask_user` tool checks for this sender and emits the `pending_interaction` event through it.

```go
// In vage/tool/askuser/context.go
type contextKey struct{}

// EventSender is a function that emits an event into the agent's stream.
type EventSender func(schema.Event)

// WithEventSender returns a context with the event sender attached.
func WithEventSender(ctx context.Context, sender EventSender) context.Context {
    return context.WithValue(ctx, contextKey{}, sender)
}

// GetEventSender retrieves the event sender from context, or nil.
func GetEventSender(ctx context.Context) EventSender {
    if s, ok := ctx.Value(contextKey{}).(EventSender); ok {
        return s
    }
    return nil
}
```

The HTTP handler injects the sender before starting the agent. The `ask_user` tool handler emits the `pending_interaction` event via this sender.

For the CLI, this is not needed -- the CLI interactor communicates with the TUI directly via `tea.Program.Send()`.

**New endpoint in `vv/httpapis/http.go`:**

```
POST /v1/interactions/{interactionID}/respond
```

Request body: `{"response": "user's text"}`
Responses: 200 OK, 404 Not Found, 409 Conflict.

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

Add to `CLIConfig`:

```go
type CLIConfig struct {
    ConfirmTools   []string `yaml:"confirm_tools"`
    AskUserTimeout int      `yaml:"ask_user_timeout"` // seconds, default 300 (5 minutes)
}
```

Also add a top-level or nested field so HTTP mode can use it too. Since both modes share it, place it at the `Config` level:

```go
type Config struct {
    // ... existing fields ...
    AskUserTimeout int `yaml:"ask_user_timeout"` // seconds, default 300; 0 = no timeout
}
```

In `applyDefaults()`:

```go
if cfg.AskUserTimeout == 0 {
    cfg.AskUserTimeout = 300 // 5 minutes
}
```

### 7. Registration Wiring (`vv/setup/setup.go`)

The `ask_user` tool must be registered into **every** agent's tool registry, regardless of `ToolProfile`. This is done after profile-based tools are built.

Changes to `setup.New()`:

1. Accept a `UserInteractor` (or construct one based on mode).
2. After building each agent's tool registry via `desc.ToolProfile.BuildRegistry()`, register the `ask_user` tool into it.
3. Also register it into the explorer's tool registry.

The `Options` struct gains:

```go
type Options struct {
    WrapToolRegistry func(*tool.Registry) tool.ToolRegistry
    UserInteractor   askuser.UserInteractor // <-- new
    AskUserTimeout   time.Duration          // <-- new
}
```

In the agent-building loop:

```go
// After building toolReg from profile:
if opts != nil && opts.UserInteractor != nil {
    askuserTool := askuser.New(opts.UserInteractor, askuser.WithTimeout(opts.AskUserTimeout))
    _ = askuserTool.Register(toolReg) // Register into each agent's registry
}
```

In `main.go`, construct the appropriate interactor:

```go
// For CLI interactive mode:
interactor := cli.NewAskUserInteractor() // wired to the TUI later

// For -p (non-interactive) mode:
interactor := &askuser.NonInteractiveInteractor{}

// For HTTP mode:
interactionStore := httpapis.NewInteractionStore()
interactor := httpapis.NewHTTPInteractor(interactionStore)
```

### 8. System Prompt Updates (`vv/agents/*.go`)

Add the following section to each agent's system prompt (coder, researcher, reviewer, chat):

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

The chat agent, which currently has no tools (`ProfileNone`), will gain access to `ask_user` since it is registered universally. Its system prompt update is minimal since it already converses with users.

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
- Create `vage/tool/askuser/context.go` -- `EventSender` context helpers
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
- Edit `vv/setup/setup.go` -- Add `UserInteractor` and `AskUserTimeout` to `Options` struct. After building each agent's tool registry, register `ask_user` tool. Do the same for explorer.

**Depends on:** Task 1, Task 3.

### Task 5: Implement CLI TUI interactor (`vv`)

**Files:**
- Create `vv/cli/askuser.go` -- `CLIInteractor` struct with `AskUser()` method, `NewCLIInteractor()`, `askUserRequestMsg` type
- Edit `vv/cli/messages.go` -- Add `statusAskingUser` constant, `askUserRequestMsg` type (or put in askuser.go)
- Edit `vv/cli/cli.go`:
  - Add `askUserCh chan string`, `askUserForm *huh.Form`, `askUserQuestion string` to `model` struct
  - Add `handleAskUserRequest()` method (creates huh form with text input)
  - Add `statusAskingUser` case in `View()` to render the form
  - Add `statusAskingUser` handling in `Update()` for form updates and completion
  - Add `statusAskingUser` case in `handleCtrlC()` (cancel and send fallback)
  - Initialize `askUserCh` in `newModel()`
- Edit `vv/main.go` -- Create `CLIInteractor` and pass to `setup.Options` for interactive CLI mode; use `NonInteractiveInteractor` for `-p` mode

**Depends on:** Task 1, Task 4.

### Task 6: Implement HTTP interactor and callback endpoint (`vv`)

**Files:**
- Create `vv/httpapis/interactions.go` -- `InteractionStore` struct, `Create()`, `Respond()`, `Get()`, cleanup goroutine
- Create `vv/httpapis/askuser.go` -- `HTTPInteractor` struct implementing `UserInteractor`, uses `InteractionStore` and `EventSender`
- Edit `vv/httpapis/http.go` -- Add `POST /v1/interactions/{interactionID}/respond` handler, create `InteractionStore` in `Serve()`, wire `HTTPInteractor` into setup
- Create `vv/httpapis/interactions_test.go` -- Unit tests for store operations (create, respond, double-respond 409, not-found 404, expiry)

**Depends on:** Task 1, Task 2, Task 4.

### Task 7: Update agent system prompts (`vv`)

**Files:**
- Edit `vv/agents/coder.go` -- Append `ask_user` guidance to `CoderSystemPrompt`
- Edit `vv/agents/researcher.go` -- Append to `ResearcherSystemPrompt`
- Edit `vv/agents/reviewer.go` -- Append to `ReviewerSystemPrompt`
- Edit `vv/agents/chat.go` -- Append to `ChatSystemPrompt`
- Edit `vv/agents/explorer.go` -- Append to `ExplorerSystemPrompt`
- Edit existing tests if prompt content is asserted

**Depends on:** Nothing (can be done in parallel with other tasks).

### Task 8: CLI prompt mode support (`vv`)

**Files:**
- Edit `vv/cli/prompt.go` -- Add handling for the new `EventPendingInteraction` event type in the non-interactive prompt consumer (log question to stderr, note it was skipped)

**Depends on:** Task 2.

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
- Verify the fallback text appears in stderr output.

### Test 3: HTTP Streaming `ask_user`

**Location:** `vv/integrations/http_tests/` or `vv/httpapis/`

- Start the HTTP service with an agent that calls `ask_user`.
- Send a `POST /v1/agents/{id}/stream` request.
- Verify a `pending_interaction` SSE event is emitted with an interaction ID and the question.
- `POST /v1/interactions/{id}/respond` with a response.
- Verify the agent resumes and the stream continues with subsequent events.
- Verify the tool result contains the submitted response.

### Test 4: HTTP Interaction Timeout

**Location:** `vv/httpapis/`

- Create a pending interaction via the HTTP interactor with a short timeout (1 second).
- Do NOT submit a response.
- Verify the tool returns a timeout fallback message after the timeout.
- Verify the agent continues execution.

### Test 5: HTTP Double-Response (409)

**Location:** `vv/httpapis/`

- Create a pending interaction, submit a response.
- Submit a second response to the same interaction ID.
- Verify 409 Conflict is returned.

### Test 6: HTTP Expired Interaction (404)

**Location:** `vv/httpapis/`

- Request `POST /v1/interactions/{nonexistent}/respond`.
- Verify 404 Not Found.

### Test 7: Tool Registration Across All Agents

**Location:** `vv/setup/` or `vv/integrations/`

- Initialize setup with a mock interactor.
- Verify every dispatchable agent's tool registry contains the `ask_user` tool.
- Verify the explorer's tool registry also contains it.

### Test 8: `ask_user` Tool Unit Tests

**Location:** `vage/tool/askuser/`

- Test valid invocation with mock interactor returns user's response as `TextResult`.
- Test empty question returns error result.
- Test invalid JSON args returns error result.
- Test `NonInteractiveInteractor` returns fallback message immediately.
- Test timeout behavior: mock interactor that blocks; verify timeout message is returned after deadline.
