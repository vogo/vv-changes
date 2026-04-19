# Design: Conversation Context Compression (Auto-Compact)

## 1. Architecture Overview

This feature adds a three-layer defense against context window overflow in vv conversations:

1. **Tool output truncation** -- limits individual tool results at insertion time.
2. **Proactive auto-compact** -- compresses conversation history before LLM calls when token count approaches the limit.
3. **Reactive emergency compact** -- catches context overflow errors from the LLM and retries with aggressive compression.

The design places the core compression logic in `vage/memory/` (reusable framework layer) and the integration/configuration in `vv/` (application layer). The conversation history (`[]schema.Message`) managed by the CLI `App.history` is the primary target. The same mechanism applies in HTTP mode via the same code path.

### Component Interaction Diagram

```
User Input
    |
    v
CLI App (vv/cli/)
    |
    |-- 1. Add user message to history
    |-- 2. Estimate tokens for new message (memory.EstimateTokens)
    |-- 3. Check: estimatedTokenCount > threshold?
    |       YES --> ConversationCompactor.Compact(history) --> compressed history
    |-- 4. Build RunRequest with (possibly compressed) history
    |-- 5. Invoke orchestrator.RunStream(req)
    |       |
    |       v
    |   TaskAgent (vage/agent/taskagent/)
    |       |-- Tool calls produce results
    |       |-- Tool results truncated by TruncatingToolRegistry wrapper
    |       |-- LLM error? --> check if context overflow
    |       
    |-- 6. On context overflow error:
    |       EmergencyCompact(history) --> retry once
    |-- 7. On success: add agent response to history, update token estimate
    v
Display to user
```

## 2. Component Design

### 2.1 Token Estimation Utility (vage/memory/)

The existing `DefaultTokenEstimator` in `vage/memory/token_estimate.go` operates on `schema.Message`. We add a simple text-level estimation function for use outside the message context (e.g., tool output truncation).

**New function in `vage/memory/token_estimate.go`:**

```go
// EstimateTextTokens returns the estimated token count for a plain text string.
// Uses the same heuristic as DefaultTokenEstimator: len(text) / 4.
func EstimateTextTokens(text string) int {
    if len(text) == 0 {
        return 0
    }
    tokens := len(text) / 4
    if tokens == 0 {
        tokens = 1
    }
    return tokens
}
```

No changes to `DefaultTokenEstimator` itself.

### 2.2 Tool Output Truncation (vage/tool/)

A new `TruncatingToolRegistry` wraps any `tool.ToolRegistry` and truncates tool results that exceed a configured token limit. This is a decorator applied at the `vv/setup/` wiring layer.

**New file: `vage/tool/truncate.go`**

```go
package tool

// TruncatingToolRegistry wraps a ToolRegistry and truncates tool results
// that exceed maxTokens.
type TruncatingToolRegistry struct {
    inner     ToolRegistry
    maxTokens int
}

// NewTruncatingToolRegistry creates a truncating wrapper.
// If maxTokens <= 0, no truncation is applied.
func NewTruncatingToolRegistry(inner ToolRegistry, maxTokens int) *TruncatingToolRegistry

func (t *TruncatingToolRegistry) List() []schema.ToolDef
func (t *TruncatingToolRegistry) Execute(ctx context.Context, name, args string) (schema.ToolResult, error)
// Execute delegates to inner, then truncates text content parts exceeding maxTokens.
```

The `Execute` method:
1. Calls `inner.Execute(ctx, name, args)`.
2. For each `ContentPart` with `Type == "text"`, estimates tokens via `memory.EstimateTextTokens`.
3. If tokens exceed `maxTokens`, truncates the text to approximately `maxTokens * 4` characters and appends `\n[truncated: showing first N of M estimated tokens]`.
4. Returns the (possibly truncated) result.

### 2.3 Conversation Compactor (vage/memory/)

A new `ConversationCompactor` that understands conversation structure (system prompt, turns, protected messages) and uses the existing `Summarizer` for compression.

**New file: `vage/memory/compactor.go`**

```go
package memory

// ConversationCompactor compresses a conversation history by summarizing
// older messages while preserving protected messages (system prompt and
// recent turn pairs).
type ConversationCompactor struct {
    summarizer     Summarizer
    estimator      TokenEstimator
    protectedTurns int // number of recent user/assistant turn pairs to protect
}

// NewConversationCompactor creates a ConversationCompactor.
// protectedTurns is the number of recent turn pairs (user + assistant) to keep verbatim.
func NewConversationCompactor(summarizer Summarizer, protectedTurns int) *ConversationCompactor

// WithTokenEstimator sets a custom token estimator.
func (c *ConversationCompactor) WithTokenEstimator(est TokenEstimator) *ConversationCompactor

// Compact compresses messages by summarizing eligible (non-protected) messages.
// Returns the compressed message slice and the estimated token count of the result.
//
// Protected messages: the first message if it is a system message, plus the last
// protectedTurns user/assistant exchange pairs, plus any existing context_summary
// messages within the protected window.
//
// The summary replaces all eligible messages with a single message carrying
// metadata {"compressed": true, "source_count": N, "strategy": "conversation_compact"}.
func (c *ConversationCompactor) Compact(ctx context.Context, messages []schema.Message) ([]schema.Message, int, error)

// EstimateTokens returns the total estimated token count for a message slice.
func (c *ConversationCompactor) EstimateTokens(messages []schema.Message) int
```

**Compaction algorithm:**

1. Identify the system prompt (first message if `Role == system`).
2. Walk backward from the end to identify the last `protectedTurns` user/assistant pairs.
3. Everything between the system prompt and the protected tail is "eligible" for compression.
4. If no eligible messages exist, return the original slice unchanged.
5. Call `summarizer(ctx, eligibleMessages)` to produce a summary string.
6. Build a summary message with role `aimodel.RoleUser` and metadata `{"compressed": true, "source_count": len(eligible), "strategy": "conversation_compact"}`.
7. Reassemble: `[systemPrompt] + [summaryMsg] + [protectedTail]`.
8. Estimate tokens on the result and return.

The `Summarizer` function type already exists in `vage/memory/compressor_summarize_trunc.go`. We reuse it.

### 2.4 Context Overflow Detection (vage/largemodel/)

A helper function to detect context overflow errors from LLM responses.

**New file: `vage/largemodel/overflow.go`**

```go
package largemodel

import (
    "errors"
    "strings"

    "github.com/vogo/aimodel"
)

// IsContextOverflowError returns true if the error indicates the request
// exceeded the model's context window limit.
// Checks for HTTP 413 (Payload Too Large) and common error message patterns
// from OpenAI and Anthropic APIs.
func IsContextOverflowError(err error) bool {
    if err == nil {
        return false
    }

    var apiErr *aimodel.APIError
    if errors.As(err, &apiErr) {
        if apiErr.StatusCode == 413 {
            return true
        }
        // Check common error message patterns.
        msg := strings.ToLower(apiErr.Message)
        if strings.Contains(msg, "context_length_exceeded") ||
            strings.Contains(msg, "maximum context length") ||
            strings.Contains(msg, "token limit") ||
            strings.Contains(msg, "request too large") {
            return true
        }
    }

    // Also check the raw error message for non-API errors.
    msg := strings.ToLower(err.Error())
    return strings.Contains(msg, "context_length_exceeded") ||
        strings.Contains(msg, "maximum context length")
}
```

### 2.5 Configuration (vv/configs/)

**Extended `Config` struct in `vv/configs/config.go`:**

```go
// ContextConfig holds conversation context compression configuration.
type ContextConfig struct {
    ModelMaxContextTokens       int     `yaml:"model_max_context_tokens"`       // default: 128000
    CompressionThreshold        float64 `yaml:"compression_threshold"`          // default: 0.8
    ToolOutputMaxTokens         int     `yaml:"tool_output_max_tokens"`         // default: 8000
    ProtectedTurns              int     `yaml:"context_protected_turns"`        // default: 4
}
```

Added to `Config`:

```go
type Config struct {
    // ... existing fields ...
    Context ContextConfig `yaml:"context"`
}
```

**Defaults in `applyDefaults`:**

```go
if cfg.Context.ModelMaxContextTokens == 0 {
    cfg.Context.ModelMaxContextTokens = 128000
}
if cfg.Context.CompressionThreshold == 0 {
    cfg.Context.CompressionThreshold = 0.8
}
if cfg.Context.ToolOutputMaxTokens == 0 {
    cfg.Context.ToolOutputMaxTokens = 8000
}
if cfg.Context.ProtectedTurns == 0 {
    cfg.Context.ProtectedTurns = 4
}
```

**Environment variable overrides** (in `Load`):

```go
if v := os.Getenv("VV_MODEL_MAX_CONTEXT_TOKENS"); v != "" {
    // parse int, set cfg.Context.ModelMaxContextTokens
}
if v := os.Getenv("VV_CONTEXT_COMPRESSION_THRESHOLD"); v != "" {
    // parse float, set cfg.Context.CompressionThreshold
}
if v := os.Getenv("VV_TOOL_OUTPUT_MAX_TOKENS"); v != "" {
    // parse int, set cfg.Context.ToolOutputMaxTokens
}
if v := os.Getenv("VV_CONTEXT_PROTECTED_TURNS"); v != "" {
    // parse int, set cfg.Context.ProtectedTurns
}
```

### 2.6 CLI Integration (vv/cli/)

The `App` struct gains fields for context management:

```go
type App struct {
    // ... existing fields ...
    compactor          *memory.ConversationCompactor
    estimatedTokens    int  // running total of estimated tokens in history
    contextCfg         configs.ContextConfig
}
```

**Modified `handleSubmit` flow in `cli.go`:**

After adding the user message to `m.app.history`:

```go
// Estimate tokens for new message and update running total.
m.app.estimatedTokens += memory.DefaultTokenEstimator(userMsg)

// Check if auto-compact is needed.
threshold := int(float64(m.app.contextCfg.ModelMaxContextTokens) * m.app.contextCfg.CompressionThreshold)
if m.app.estimatedTokens > threshold {
    compressed, newTokens, err := m.app.compactor.Compact(runCtx, m.app.history)
    if err == nil {
        n := len(m.app.history) - len(compressed)
        m.app.history = compressed
        m.app.estimatedTokens = newTokens
        // Show notification.
        cmds = append(cmds, m.printSystem(
            fmt.Sprintf("[context compressed: summarized %d messages to stay within model limits]", n)))
    }
}
```

**Modified `handleStreamDone` for reactive compact:**

When a stream error indicates context overflow:

```go
if msg.err != nil && largemodel.IsContextOverflowError(msg.err) {
    // Emergency compact with minimum protection (1 turn pair).
    emergencyCompactor := memory.NewConversationCompactor(m.app.compactor.Summarizer(), 1)
    compressed, newTokens, err := emergencyCompactor.Compact(m.ctx, m.app.history)
    if err == nil {
        m.app.history = compressed
        m.app.estimatedTokens = newTokens
        cmds = append(cmds, m.printSystem(
            "[emergency context compression: retrying with compressed history]"))
        // Retry the agent invocation once.
        cmds = append(cmds, m.invokeAgent(runCtx, ""))
        return m, tea.Batch(cmds...)
    }
    // If emergency compact also fails, fall through to normal error display
    // with suggestion to start a new session.
    cmds = append(cmds, m.printError(
        "Context overflow persists after emergency compression. Please start a new session."))
}
```

**Token tracking for agent responses and tool results:**

After adding the agent response to history in `handleStreamDone`:

```go
m.app.estimatedTokens += memory.DefaultTokenEstimator(agentMsg)
```

Tool result tokens are already accounted for because tool results flow through the `TruncatingToolRegistry` (capped at `tool_output_max_tokens`) and are part of the internal TaskAgent message loop, not the CLI history directly. The CLI history only contains user messages and final agent responses; intermediate tool calls are within the TaskAgent's internal loop. Therefore, the primary token tracking in the CLI focuses on user and agent messages.

However, tool call/result messages **do** appear in `m.app.history` when the TaskAgent's session memory promotes them. Since the CLI manages `history` directly (appending user messages before invoking, and agent messages after), and the orchestrator receives the full history via `RunRequest.Messages`, the token estimate should account for what is actually sent. The CLI currently sends `m.app.history` as `req.Messages`, and the TaskAgent's `buildInitialMessages` also loads from session memory. To avoid double-counting, we track tokens based on what the CLI puts in `m.app.history` (user + agent messages only), which is the array sent to the orchestrator. The orchestrator/sub-agents add their own system prompts and session memory internally, but that is within their own token budget management.

### 2.7 Setup Wiring (vv/setup/)

**In `setup.New()`:**

Wrap each agent's tool registry with `TruncatingToolRegistry`:

```go
// After building finalToolReg:
if cfg.Context.ToolOutputMaxTokens > 0 {
    finalToolReg = tool.NewTruncatingToolRegistry(finalToolReg, cfg.Context.ToolOutputMaxTokens)
}
```

**Create the summarizer for the compactor** in `setup.Init()`:

```go
// Create conversation compactor summarizer using the LLM.
compactSummarizer := func(ctx context.Context, messages []schema.Message) (string, error) {
    // Build a summarization prompt from the messages.
    var sb strings.Builder
    sb.WriteString("Summarize the following conversation, preserving key decisions, ")
    sb.WriteString("file changes, task progress, and important context:\n\n")
    for _, msg := range messages {
        sb.WriteString(fmt.Sprintf("[%s]: %s\n", msg.Role, msg.Content.Text()))
    }

    req := &aimodel.ChatRequest{
        Model: cfg.LLM.Model,
        Messages: []aimodel.Message{
            {Role: aimodel.RoleUser, Content: aimodel.NewTextContent(sb.String())},
        },
    }
    resp, err := llmClient.ChatCompletion(ctx, req)
    if err != nil {
        return "", err
    }
    if len(resp.Choices) == 0 {
        return "", fmt.Errorf("empty summarization response")
    }
    return resp.Choices[0].Message.Content.Text(), nil
}

compactor := memory.NewConversationCompactor(compactSummarizer, cfg.Context.ProtectedTurns)
```

The `compactor` is passed to the CLI `App` constructor (or added to `InitResult`).

### 2.8 HTTP Mode Integration

In HTTP mode, the same compression applies. The `Dispatcher.Run` / `RunStream` receives `req.Messages` which is the conversation history. The HTTP handler constructs `RunRequest` from the client-provided messages.

For HTTP mode, context compression should be applied at the point where `RunRequest` is built. This can be done in the HTTP handler layer (`vv/httpapis/`) or as a middleware on the Dispatcher. The simplest approach: add a `WithConversationCompactor` option to the Dispatcher that applies compression to `req.Messages` at the start of `Run`/`RunStream` if the estimated token count exceeds the threshold.

**New option in `vv/dispatches/dispatch.go`:**

```go
// WithConversationCompactor sets the conversation compactor for auto-compact.
func WithConversationCompactor(c *memory.ConversationCompactor, threshold int) Option {
    return func(d *Dispatcher) {
        d.compactor = c
        d.compactThreshold = threshold
    }
}
```

At the start of `Dispatcher.Run` and `Dispatcher.RunStream`, before intent recognition:

```go
if d.compactor != nil && d.compactThreshold > 0 {
    est := d.compactor.EstimateTokens(req.Messages)
    if est > d.compactThreshold {
        compressed, _, err := d.compactor.Compact(ctx, req.Messages)
        if err == nil {
            req = &schema.RunRequest{
                Messages:  compressed,
                SessionID: req.SessionID,
                Options:   req.Options,
                Metadata:  req.Metadata,
            }
        }
    }
}
```

This ensures both CLI and HTTP modes benefit from compression. In CLI mode, proactive compression happens before building the request (in `handleSubmit`), so the Dispatcher check serves as a safety net. In HTTP mode, it is the primary compression trigger.

## 3. Data Models / Schemas

### 3.1 Modified Models

**`configs.ContextConfig`** (new struct):

| Field | Type | YAML Key | Default | Env Var |
|-------|------|----------|---------|---------|
| ModelMaxContextTokens | int | `model_max_context_tokens` | 128000 | `VV_MODEL_MAX_CONTEXT_TOKENS` |
| CompressionThreshold | float64 | `compression_threshold` | 0.8 | `VV_CONTEXT_COMPRESSION_THRESHOLD` |
| ToolOutputMaxTokens | int | `tool_output_max_tokens` | 8000 | `VV_TOOL_OUTPUT_MAX_TOKENS` |
| ProtectedTurns | int | `context_protected_turns` | 4 | `VV_CONTEXT_PROTECTED_TURNS` |

**`cli.App`** (extended):

| Field | Type | Description |
|-------|------|-------------|
| compactor | `*memory.ConversationCompactor` | Conversation compactor instance |
| estimatedTokens | `int` | Running token count estimate for `history` |
| contextCfg | `configs.ContextConfig` | Context compression config |

**Summary messages** use `schema.Message` with:
- `Role`: `aimodel.RoleUser` (LLM-compatible; avoids needing a new role constant)
- `Metadata`: `{"compressed": true, "source_count": N, "strategy": "conversation_compact"}`

We intentionally use `aimodel.RoleUser` for the summary message rather than introducing a new role constant. This ensures compatibility with all LLM providers. The `compressed` metadata flag distinguishes summary messages from regular user messages when needed (e.g., for display rendering).

### 3.2 CLI Display

For rendering in the CLI, messages with `Metadata["compressed"] == true` are detected and rendered with a `[Previous context (summarized)]` header in dimmed style, similar to system messages.

## 4. API Contracts

### 4.1 New Public API (vage/memory/)

```go
// token_estimate.go
func EstimateTextTokens(text string) int

// compactor.go
type ConversationCompactor struct { ... }
func NewConversationCompactor(summarizer Summarizer, protectedTurns int) *ConversationCompactor
func (c *ConversationCompactor) WithTokenEstimator(est TokenEstimator) *ConversationCompactor
func (c *ConversationCompactor) Compact(ctx context.Context, messages []schema.Message) ([]schema.Message, int, error)
func (c *ConversationCompactor) EstimateTokens(messages []schema.Message) int
func (c *ConversationCompactor) Summarizer() Summarizer
```

### 4.2 New Public API (vage/tool/)

```go
// truncate.go
type TruncatingToolRegistry struct { ... }
func NewTruncatingToolRegistry(inner ToolRegistry, maxTokens int) *TruncatingToolRegistry
// Implements ToolRegistry interface (List, Execute, and any other methods)
```

### 4.3 New Public API (vage/largemodel/)

```go
// overflow.go
func IsContextOverflowError(err error) bool
```

### 4.4 Modified API (vv/cli/)

```go
// New constructor signature adds compactor and context config:
func New(
    orchestrator agent.StreamAgent,
    cfg *configs.Config,
    persistentMem memory.Memory,
    interactor *CLIInteractor,
    compactor *memory.ConversationCompactor,
) *App
```

### 4.5 Modified API (vv/dispatches/)

```go
// New option:
func WithConversationCompactor(c *memory.ConversationCompactor, threshold int) Option
```

## 5. Implementation Plan

### Task 1: Token Estimation Utility
**Module:** `vage/memory/`
**Files:** `token_estimate.go`, `token_estimate_test.go`
- Add `EstimateTextTokens(text string) int` function.
- Add unit tests.

### Task 2: Tool Output Truncation
**Module:** `vage/tool/`
**Files:** `truncate.go`, `truncate_test.go`
- Implement `TruncatingToolRegistry` struct.
- Implement `List()` (delegates to inner), `Execute()` (delegates then truncates).
- Implement any other `ToolRegistry` interface methods by delegation.
- Add unit tests: no truncation below limit, truncation above limit, marker text, error passthrough, zero/negative maxTokens disables truncation.

### Task 3: Conversation Compactor
**Module:** `vage/memory/`
**Files:** `compactor.go`, `compactor_test.go`
- Implement `ConversationCompactor` with `Compact`, `EstimateTokens`, `Summarizer` methods.
- Protected message identification algorithm.
- Summary message construction with metadata.
- Unit tests: no compression when few messages, correct protected turn identification, summary message structure, token estimation, context with existing summary messages.

### Task 4: Context Overflow Detection
**Module:** `vage/largemodel/`
**Files:** `overflow.go`, `overflow_test.go`
- Implement `IsContextOverflowError`.
- Unit tests: various `APIError` status codes, message patterns, nil error, non-API errors.

### Task 5: Configuration
**Module:** `vv/configs/`
**Files:** `config.go`, `config_test.go`
- Add `ContextConfig` struct to `Config`.
- Add defaults in `applyDefaults`.
- Add env var overrides in `Load`.
- Update existing tests, add new tests for defaults and env var parsing.

### Task 6: Setup Wiring
**Module:** `vv/setup/`
**Files:** `setup.go`, `setup_test.go`
- Wire `TruncatingToolRegistry` around each agent's tool registry.
- Create the conversation compactor summarizer using the LLM client.
- Pass compactor to `InitResult` for CLI/HTTP consumption.
- Wire `WithConversationCompactor` on the Dispatcher.

### Task 7: CLI Integration -- Proactive Compact
**Module:** `vv/cli/`
**Files:** `cli.go`
- Add `compactor`, `estimatedTokens`, `contextCfg` fields to `App`.
- Update `New()` constructor.
- In `handleSubmit`: estimate tokens, check threshold, compact if needed, show notification.
- In `handleStreamDone`: update token estimate after adding agent response.

### Task 8: CLI Integration -- Reactive Compact
**Module:** `vv/cli/`
**Files:** `cli.go`
- In `handleStreamDone`: detect context overflow errors via `largemodel.IsContextOverflowError`.
- On overflow: emergency compact with `protectedTurns=1`, retry once, show notification.
- On retry failure: show error with "start a new session" suggestion.
- Add `retryCount` or `emergencyCompacted` flag to `model` to prevent infinite retry loops.

### Task 9: CLI Display for Summary Messages
**Module:** `vv/cli/`
**Files:** `render.go`
- Detect messages with `Metadata["compressed"] == true` in display rendering.
- Render with dimmed style and `[Previous context (summarized)]` header.

### Task 10: Dispatcher Compression (HTTP Mode)
**Module:** `vv/dispatches/`
**Files:** `dispatch.go`
- Add `compactor` and `compactThreshold` fields.
- Add `WithConversationCompactor` option.
- Add compression check at start of `Run` and `RunStream`.

### Task Order and Dependencies

```
Task 1 (token estimation)
    |
    +---> Task 2 (tool truncation)     Task 4 (overflow detection)
    |         |                              |
    +---> Task 3 (compactor)           Task 5 (configuration)
              |                              |
              +-------+------+---------------+
                      |      |
                      v      v
                   Task 6 (setup wiring)
                      |
              +-------+-------+
              |       |       |
              v       v       v
         Task 7   Task 8   Task 10
        (CLI      (CLI     (Dispatcher
        proactive) reactive) HTTP)
              |
              v
           Task 9 (CLI display)
```

Tasks 1 and 4 can be done in parallel. Tasks 2, 3, 5 can follow in parallel once Task 1 is complete. Task 6 depends on Tasks 2, 3, 4, 5. Tasks 7, 8, 10 depend on Task 6. Task 9 depends on Task 7.

## 6. Integration Test Plan

Integration tests require LLM API keys and live in `vv/integrations/` or `vage/integrations/`.

### Test 1: Tool Output Truncation (Unit -- no LLM needed)
**Location:** `vage/tool/truncate_test.go`
- Create a mock `ToolRegistry` that returns a large text result (e.g., 50,000 characters).
- Wrap with `TruncatingToolRegistry(inner, 2000)`.
- Call `Execute` and verify result is truncated to approximately 2000 tokens.
- Verify truncation marker is present.
- Verify results below the limit are not modified.

### Test 2: Conversation Compactor (Integration -- needs LLM)
**Location:** `vage/integrations/compressor_tests/compactor_test.go`
- Build a conversation with 20+ messages (mix of user, assistant, tool).
- Create a `ConversationCompactor` with a real LLM summarizer and `protectedTurns=2`.
- Call `Compact` and verify:
  - System prompt is preserved.
  - Last 2 user/assistant pairs are preserved verbatim.
  - A summary message exists with `compressed` metadata.
  - Total token count is reduced.

### Test 3: Context Overflow Detection (Unit -- no LLM needed)
**Location:** `vage/largemodel/overflow_test.go`
- Create `aimodel.APIError` with status 413 -> returns true.
- Create `aimodel.APIError` with message containing "context_length_exceeded" -> returns true.
- Create `aimodel.APIError` with status 400, no relevant message -> returns false.
- Nil error -> returns false.

### Test 4: End-to-End Proactive Compression (Integration -- needs LLM)
**Location:** `vv/integrations/cli_tests/compression_test.go`
- Create a `cli.App` with a small `ModelMaxContextTokens` (e.g., 4000) and low threshold (0.5).
- Simulate adding messages to history until token estimate exceeds threshold.
- Verify that compression is triggered (history length decreases).
- Verify a summary message is present in the compressed history.
- Verify the agent can still process the next request successfully.

### Test 5: Configuration Defaults and Overrides (Unit)
**Location:** `vv/configs/config_test.go`
- Load config with no context section -> verify defaults applied.
- Load config with explicit values -> verify they are used.
- Set environment variables -> verify overrides take effect.

### Test 6: Reactive Compression on Overflow (Integration -- needs LLM)
**Location:** `vv/integrations/cli_tests/compression_test.go`
- Use a mock LLM that returns a context overflow error on the first call and succeeds on the second.
- Verify emergency compact is triggered.
- Verify the retry succeeds.
- Verify the retry-failure path shows the appropriate error message when both calls fail.

### Test 7: TruncatingToolRegistry Interface Compliance (Unit)
**Location:** `vage/tool/truncate_test.go`
- Verify `TruncatingToolRegistry` satisfies the `ToolRegistry` interface.
- Verify `List()` passes through unchanged.
- Verify error results are not truncated.
