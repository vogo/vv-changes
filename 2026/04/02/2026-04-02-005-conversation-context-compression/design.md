# Design: Conversation Context Compression (Auto-Compact)

## 1. Architecture Overview

This feature adds a three-layer defense against context window overflow in vv conversations:

1. **Tool output truncation** -- limits individual tool results at insertion time.
2. **Proactive auto-compact** -- compresses conversation history before LLM calls when token count approaches the limit.
3. **Reactive emergency compact** -- catches context overflow errors from the LLM and retries with aggressive compression.

The design places the core compression logic in `vage/memory/` (reusable framework layer) and the integration/configuration in `vv/` (application layer). The conversation history (`[]schema.Message`) managed by the CLI `App.history` is the primary target. The same mechanism applies in HTTP mode via a shared utility function called from the HTTP handler layer.

### Component Interaction Diagram

```
User Input
    |
    v
CLI App (vv/cli/)
    |
    |-- 1. Add user message to history
    |-- 2. Estimate tokens for new message (memory.EstimateTextTokens)
    |-- 3. Dispatch invokeAgent (async tea.Cmd)
    |       |
    |       |-- 3a. Check: estimatedTokenCount > threshold?
    |       |       YES --> ConversationCompactor.Compact(history) --> compressed history
    |       |-- 3b. Build RunRequest with (possibly compressed) history
    |       |-- 3c. Invoke orchestrator.RunStream(req)
    |       |       |
    |       |       v
    |       |   TaskAgent (vage/agent/taskagent/)
    |       |       |-- Tool calls produce results
    |       |       |-- Tool results truncated by TruncatingToolRegistry wrapper
    |       |       |-- LLM error? --> check if context overflow
    |       |
    |-- 4. On context overflow error (via streamDoneMsg):
    |       Dispatch async emergencyCompactCmd --> retry once
    |-- 5. On success: add agent response to history, update token estimate
    v
Display to user
```

### Key Design Decisions

- **Compaction runs inside async `tea.Cmd` functions**, not in synchronous Bubbletea `Update` handlers, to avoid blocking the TUI during LLM summarization calls.
- **Summary messages use `aimodel.RoleSystem`** (not `RoleUser`), since they represent system-injected context, not user input. This avoids confusing the LLM.
- **Compression is applied at the caller layer** (CLI and HTTP handler), not inside the `Dispatcher`, to maintain separation of concerns.
- **A 10% safety margin** is reserved from the context window for system prompts and agent overhead that the CLI cannot see.

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
//
// TruncatingToolRegistry is safe for concurrent use if the underlying
// ToolRegistry is safe for concurrent use.
type TruncatingToolRegistry struct {
    inner     ToolRegistry
    maxTokens int
}

// NewTruncatingToolRegistry creates a truncating wrapper.
// If maxTokens <= 0, no truncation is applied.
func NewTruncatingToolRegistry(inner ToolRegistry, maxTokens int) *TruncatingToolRegistry

// Compile-time check: TruncatingToolRegistry implements ToolRegistry.
var _ ToolRegistry = (*TruncatingToolRegistry)(nil)

// All ToolRegistry methods delegate to inner, except Execute which adds truncation.
func (t *TruncatingToolRegistry) Register(def schema.ToolDef, handler ToolHandler) error
func (t *TruncatingToolRegistry) Unregister(name string) error
func (t *TruncatingToolRegistry) Get(name string) (schema.ToolDef, bool)
func (t *TruncatingToolRegistry) List() []schema.ToolDef
func (t *TruncatingToolRegistry) Merge(defs []schema.ToolDef)
func (t *TruncatingToolRegistry) Execute(ctx context.Context, name, args string) (schema.ToolResult, error)
// Execute delegates to inner, then truncates text content parts exceeding maxTokens.
```

The `Execute` method:
1. Calls `inner.Execute(ctx, name, args)`.
2. For each `ContentPart` with `Type == "text"`, estimates tokens via `memory.EstimateTextTokens`.
3. If tokens exceed `maxTokens`, truncates the text to approximately `maxTokens * 4` characters and appends `\n[truncated: showing first N of M estimated tokens]`.
4. Returns the (possibly truncated) result.

All other methods (`Register`, `Unregister`, `Get`, `List`, `Merge`) delegate directly to `inner` without modification.

### 2.3 Conversation Compactor (vage/memory/)

A new `ConversationCompactor` that understands conversation structure (system prompt, turns, protected messages) and uses the existing `Summarizer` function type for compression.

**New file: `vage/memory/compactor.go`**

```go
package memory

// ConversationCompactor compresses a conversation history by summarizing
// older messages while preserving protected messages (system prompt and
// recent turn pairs).
//
// ConversationCompactor is safe for concurrent use; it holds no mutable state.
type ConversationCompactor struct {
    summarizer     Summarizer
    estimator      TokenEstimator
    protectedTurns int // number of recent user/assistant turn pairs to protect
    maxInputTokens int // max tokens for summarizer input; 0 means no limit
}

// NewConversationCompactor creates a ConversationCompactor.
// protectedTurns is the number of recent turn pairs (user + assistant) to keep verbatim.
func NewConversationCompactor(summarizer Summarizer, protectedTurns int) *ConversationCompactor

// WithTokenEstimator sets a custom token estimator.
func (c *ConversationCompactor) WithTokenEstimator(est TokenEstimator) *ConversationCompactor

// WithMaxInputTokens sets the maximum token count for the summarizer input.
// If the eligible messages exceed this, the input is truncated with an omission marker.
// This prevents the summarization call itself from exceeding the context window.
func (c *ConversationCompactor) WithMaxInputTokens(n int) *ConversationCompactor

// Compact compresses messages by summarizing eligible (non-protected) messages.
// Returns the compressed message slice and the estimated token count of the result.
//
// Protected messages: the first message if it is a system message, plus the last
// protectedTurns user/assistant exchange pairs.
//
// The summary replaces all eligible messages with a single system-role message
// carrying metadata {"compressed": true, "source_count": N, "strategy": "conversation_compact"}.
func (c *ConversationCompactor) Compact(ctx context.Context, messages []schema.Message) ([]schema.Message, int, error)

// EstimateTokens returns the total estimated token count for a message slice.
func (c *ConversationCompactor) EstimateTokens(messages []schema.Message) int

// Summarizer returns the underlying summarizer function.
func (c *ConversationCompactor) Summarizer() Summarizer
```

**Compaction algorithm:**

1. Identify the system prompt (first message if `Role == system`).
2. Walk backward from the end to identify the last `protectedTurns` user/assistant pairs (counting each user+assistant as one pair, so `protectedTurns * 2` messages at most).
3. Everything between the system prompt and the protected tail is "eligible" for compression.
4. If no eligible messages exist, return the original slice unchanged.
5. If `maxInputTokens > 0` and the eligible messages exceed that budget, truncate the input to the summarizer (keep first and last portions with a `[... N messages omitted ...]` marker).
6. Call `summarizer(ctx, eligibleMessages)` to produce a summary string.
7. Build a summary message with role `aimodel.RoleSystem` and metadata `{"compressed": true, "source_count": len(eligible), "strategy": "conversation_compact"}`.
8. Reassemble: `[systemPrompt] + [summaryMsg] + [protectedTail]`.
9. Estimate tokens on the result and return.

The `Summarizer` function type is reused from `vage/memory/compressor_summarize_trunc.go`. The metadata conventions (compressed, source_count, strategy) follow the pattern established by `SummarizeAndTruncCompressor`.

**Standalone utility function:**

```go
// CompactIfNeeded checks the estimated token count and compresses if above the threshold.
// Returns the (possibly compressed) messages, updated token estimate, and whether compaction occurred.
// This is a convenience for callers (CLI, HTTP handler) to avoid duplicating the check logic.
func CompactIfNeeded(
    ctx context.Context,
    compactor *ConversationCompactor,
    messages []schema.Message,
    threshold int,
) ([]schema.Message, int, bool, error)
```

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
// Checks for HTTP 413 (Payload Too Large), error codes, and common error
// message patterns from OpenAI and Anthropic APIs.
func IsContextOverflowError(err error) bool {
    if err == nil {
        return false
    }

    var apiErr *aimodel.APIError
    if errors.As(err, &apiErr) {
        if apiErr.StatusCode == 413 {
            return true
        }
        // Check error codes used by providers.
        switch apiErr.Code {
        case "context_length_exceeded", "request_too_large":
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
    ModelMaxContextTokens  int      `yaml:"model_max_context_tokens"`  // default: 128000
    CompressionThreshold   *float64 `yaml:"compression_threshold"`     // default: 0.8; pointer to distinguish "not set" from 0.0
    ToolOutputMaxTokens    int      `yaml:"tool_output_max_tokens"`    // default: 8000
    ProtectedTurns         int      `yaml:"context_protected_turns"`   // default: 4
}

// EffectiveCompressionThreshold returns the compression threshold,
// falling back to 0.8 if not explicitly set.
func (c ContextConfig) EffectiveCompressionThreshold() float64 {
    if c.CompressionThreshold != nil {
        return *c.CompressionThreshold
    }
    return 0.8
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
if cfg.Context.ToolOutputMaxTokens == 0 {
    cfg.Context.ToolOutputMaxTokens = 8000
}
if cfg.Context.ProtectedTurns == 0 {
    cfg.Context.ProtectedTurns = 4
}
// CompressionThreshold uses a pointer; nil means "use default 0.8" via EffectiveCompressionThreshold().
```

**Environment variable overrides** (in `Load`):

```go
if v := os.Getenv("VV_MAX_CONTEXT_TOKENS"); v != "" {
    // parse int, set cfg.Context.ModelMaxContextTokens
}
if v := os.Getenv("VV_CONTEXT_COMPRESSION_THRESHOLD"); v != "" {
    // parse float64, set cfg.Context.CompressionThreshold (pointer)
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

**Modified `New` constructor:**

```go
func New(
    orchestrator agent.StreamAgent,
    cfg *configs.Config,
    persistentMem memory.Memory,
    interactor *CLIInteractor,
    compactor *memory.ConversationCompactor,
) *App
```

**Modified `handleSubmit` flow in `cli.go`:**

After adding the user message to `m.app.history`, update the running token estimate. The estimate is maintained in the `Update` handler (single-threaded by Bubbletea):

```go
// Estimate tokens for new message and update running total.
m.app.estimatedTokens += memory.DefaultTokenEstimator(userMsg)
```

Compaction is NOT performed here (it involves an LLM call). Instead, the threshold check and compaction happen inside the async `invokeAgent` function.

**Modified `invokeAgent` -- proactive compact inside async `tea.Cmd`:**

```go
func (m *model) invokeAgent(ctx context.Context, _ string) tea.Cmd {
    return func() tea.Msg {
        // Check if auto-compact is needed before building the request.
        safetyMargin := 0.10 // reserve 10% for system prompts and agent overhead
        effectiveMax := float64(m.app.contextCfg.ModelMaxContextTokens) * (1.0 - safetyMargin)
        threshold := int(effectiveMax * m.app.contextCfg.EffectiveCompressionThreshold())

        if m.app.estimatedTokens > threshold {
            compressed, newTokens, compacted, err := memory.CompactIfNeeded(
                ctx, m.app.compactor, m.app.history, threshold)
            if err == nil && compacted {
                n := len(m.app.history) - len(compressed)
                m.app.history = compressed
                m.app.estimatedTokens = newTokens
                // Send notification to TUI (will be processed in Update).
                m.app.program.Send(compactionDoneMsg{summarizedCount: n})
            }
        }

        // Build request with (possibly compressed) history.
        req := &schema.RunRequest{
            Messages:  m.app.history,
            SessionID: m.app.sessionID,
        }

        stream, err := m.app.orchestrator.RunStream(ctx, req)
        // ... rest as before ...
    }
}
```

**New message type for compaction notifications:**

```go
// compactionDoneMsg signals that context compression completed.
type compactionDoneMsg struct {
    summarizedCount int
}
```

Handle in `Update`:

```go
case compactionDoneMsg:
    return m, m.printSystem(
        fmt.Sprintf("[context compressed: summarized %d messages to stay within model limits]", msg.summarizedCount))
```

**Modified `handleStreamDone` for reactive compact:**

When a stream error indicates context overflow, dispatch an async emergency compact command instead of running synchronously:

```go
if msg.err != nil && largemodel.IsContextOverflowError(msg.err) {
    cmds = append(cmds, m.printSystem("[context overflow detected: attempting emergency compression...]"))
    cmds = append(cmds, m.emergencyCompactCmd())
    return m, tea.Batch(cmds...)
}
```

**Emergency compact as async `tea.Cmd`:**

```go
func (m *model) emergencyCompactCmd() tea.Cmd {
    return func() tea.Msg {
        emergencyCompactor := memory.NewConversationCompactor(m.app.compactor.Summarizer(), 1)
        compressed, newTokens, err := emergencyCompactor.Compact(m.ctx, m.app.history)
        if err != nil {
            return emergencyCompactResultMsg{err: err}
        }
        return emergencyCompactResultMsg{
            compressed: compressed,
            newTokens:  newTokens,
        }
    }
}

type emergencyCompactResultMsg struct {
    compressed []schema.Message
    newTokens  int
    err        error
}
```

Handle the result in `Update`:

```go
case emergencyCompactResultMsg:
    if msg.err != nil {
        cmds = append(cmds,
            m.printError("Context overflow persists after emergency compression. Please start a new session."))
        m.status = statusIdle
        m.textarea.Focus()
        return m, tea.Batch(cmds...)
    }
    m.app.history = msg.compressed
    m.app.estimatedTokens = msg.newTokens
    cmds = append(cmds, m.printSystem("[emergency context compression complete: retrying...]"))
    // Retry the agent invocation once.
    runCtx, cancel := context.WithCancel(m.ctx)
    m.runCancel = cancel
    cmds = append(cmds, m.invokeAgent(runCtx, ""))
    return m, tea.Batch(cmds...)
```

Add an `emergencyCompacted` flag to `model` to prevent infinite retry loops. Set it to `true` when dispatching emergency compact; check it in `handleStreamDone` -- if already true and another overflow occurs, show the final error.

**Token tracking for agent responses:**

After adding the agent response to history in `handleStreamDone`:

```go
m.app.estimatedTokens += memory.DefaultTokenEstimator(agentMsg)
```

**Note on token estimate accuracy:** The CLI history contains only user messages and final agent response text. Tool call/result messages live inside the TaskAgent's internal ReAct loop and session memory, not in `m.app.history`. The 10% safety margin on the context window accounts for system prompts, session memory, and agent overhead that the CLI cannot see.

**New `/compact` slash command:**

Add to `handleCommand` in `cli/memory.go`:

```go
case "/compact":
    if m.app.compactor == nil {
        return m.printSystem("Context compression is not configured.")
    }
    // Dispatch async compaction.
    return m.manualCompactCmd()
```

### 2.7 Setup Wiring (vv/setup/)

**In `New()`:**

Wrap each agent's tool registry with `TruncatingToolRegistry`:

```go
// After building finalToolReg:
if cfg.Context.ToolOutputMaxTokens > 0 {
    finalToolReg = tool.NewTruncatingToolRegistry(finalToolReg, cfg.Context.ToolOutputMaxTokens)
}
```

Also wrap the explorer's tool registry (explorer tools can also produce large output).

**Create the summarizer for the compactor** in `Init()`:

```go
// Create conversation compactor summarizer using the LLM.
compactSummarizer := func(ctx context.Context, messages []schema.Message) (string, error) {
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

// Limit summarizer input to 80% of context window to prevent the summarization
// call itself from exceeding the context limit.
maxSummarizerInput := int(float64(cfg.Context.ModelMaxContextTokens) * 0.8)

compactor := memory.NewConversationCompactor(compactSummarizer, cfg.Context.ProtectedTurns).
    WithMaxInputTokens(maxSummarizerInput)
```

The `compactor` is added to `InitResult` for CLI/HTTP consumption:

```go
type InitResult struct {
    Config        *configs.Config
    LLMClient     *aimodel.Client
    MemoryManager *memory.Manager
    PersistentMem memory.Memory
    SetupResult   *Result
    Compactor     *memory.ConversationCompactor // conversation context compactor
}
```

### 2.8 HTTP Mode Integration

In HTTP mode, context compression is applied in the HTTP handler layer (`vv/httpapis/`), before constructing the `RunRequest` for the dispatcher. This keeps the `Dispatcher` free of memory-management concerns.

**In the HTTP handler that builds `RunRequest`:**

```go
// Before dispatching:
if compactor != nil {
    safetyMargin := 0.10
    effectiveMax := float64(cfg.Context.ModelMaxContextTokens) * (1.0 - safetyMargin)
    threshold := int(effectiveMax * cfg.Context.EffectiveCompressionThreshold())

    compressed, _, compacted, err := memory.CompactIfNeeded(ctx, compactor, req.Messages, threshold)
    if err == nil && compacted {
        req.Messages = compressed
    }
}
```

This mirrors the CLI's proactive compression logic. The `compactor` and `cfg` are injected into the HTTP handler during initialization.

For reactive compression on context overflow in HTTP mode: the HTTP handler catches the error from `Dispatcher.Run`/`RunStream`, applies emergency compact, and retries once -- same pattern as CLI but without TUI notification concerns.

### 2.9 CLI Display for Summary Messages

Messages with `Metadata["compressed"] == true` are detected in the CLI rendering layer and displayed with a dimmed style and a `[Previous context (summarized)]` header.

**In `vv/cli/render.go`:**

```go
// renderSummaryMessage renders a compressed context summary with dimmed style.
func renderSummaryMessage(text string) string {
    header := dimStyle.Render("[Previous context (summarized)]")
    body := dimStyle.Render(text)
    return header + "\n" + body
}
```

Detection in `handleStreamEvent` or wherever messages are rendered for display:

```go
if msg.Metadata != nil {
    if compressed, ok := msg.Metadata["compressed"].(bool); ok && compressed {
        return renderSummaryMessage(msg.Content.Text())
    }
}
```

## 3. Data Models / Schemas

### 3.1 Modified Models

**`configs.ContextConfig`** (new struct):

| Field | Type | YAML Key | Default | Env Var |
|-------|------|----------|---------|---------|
| ModelMaxContextTokens | int | `model_max_context_tokens` | 128000 | `VV_MAX_CONTEXT_TOKENS` |
| CompressionThreshold | *float64 | `compression_threshold` | 0.8 | `VV_CONTEXT_COMPRESSION_THRESHOLD` |
| ToolOutputMaxTokens | int | `tool_output_max_tokens` | 8000 | `VV_TOOL_OUTPUT_MAX_TOKENS` |
| ProtectedTurns | int | `context_protected_turns` | 4 | `VV_CONTEXT_PROTECTED_TURNS` |

**`cli.App`** (extended):

| Field | Type | Description |
|-------|------|-------------|
| compactor | `*memory.ConversationCompactor` | Conversation compactor instance |
| estimatedTokens | `int` | Running token count estimate for `history` |
| contextCfg | `configs.ContextConfig` | Context compression config |

**`cli.model`** (extended):

| Field | Type | Description |
|-------|------|-------------|
| emergencyCompacted | `bool` | True if emergency compact already attempted this turn |

**Summary messages** use `schema.Message` with:
- `Role`: `aimodel.RoleSystem` (semantically correct for system-injected context)
- `Metadata`: `{"compressed": true, "source_count": N, "strategy": "conversation_compact"}`

### 3.2 New Message Types (cli)

```go
type compactionDoneMsg struct {
    summarizedCount int
}

type emergencyCompactResultMsg struct {
    compressed []schema.Message
    newTokens  int
    err        error
}
```

## 4. API Contracts

### 4.1 New Public API (vage/memory/)

```go
// token_estimate.go
func EstimateTextTokens(text string) int

// compactor.go
type ConversationCompactor struct { ... }
func NewConversationCompactor(summarizer Summarizer, protectedTurns int) *ConversationCompactor
func (c *ConversationCompactor) WithTokenEstimator(est TokenEstimator) *ConversationCompactor
func (c *ConversationCompactor) WithMaxInputTokens(n int) *ConversationCompactor
func (c *ConversationCompactor) Compact(ctx context.Context, messages []schema.Message) ([]schema.Message, int, error)
func (c *ConversationCompactor) EstimateTokens(messages []schema.Message) int
func (c *ConversationCompactor) Summarizer() Summarizer
func CompactIfNeeded(ctx context.Context, compactor *ConversationCompactor, messages []schema.Message, threshold int) ([]schema.Message, int, bool, error)
```

### 4.2 New Public API (vage/tool/)

```go
// truncate.go
type TruncatingToolRegistry struct { ... }
func NewTruncatingToolRegistry(inner ToolRegistry, maxTokens int) *TruncatingToolRegistry
// Implements full ToolRegistry interface: Register, Unregister, Get, List, Merge, Execute.
// Only Execute adds truncation behavior; all other methods delegate directly.
```

### 4.3 New Public API (vage/largemodel/)

```go
// overflow.go
func IsContextOverflowError(err error) bool
```

### 4.4 Modified API (vv/cli/)

```go
// New constructor signature adds compactor:
func New(
    orchestrator agent.StreamAgent,
    cfg *configs.Config,
    persistentMem memory.Memory,
    interactor *CLIInteractor,
    compactor *memory.ConversationCompactor,
) *App
```

### 4.5 Modified API (vv/setup/)

```go
// InitResult gains Compactor field:
type InitResult struct {
    Config        *configs.Config
    LLMClient     *aimodel.Client
    MemoryManager *memory.Manager
    PersistentMem memory.Memory
    SetupResult   *Result
    Compactor     *memory.ConversationCompactor
}
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
- Implement all `ToolRegistry` interface methods: `Register`, `Unregister`, `Get`, `List`, `Merge` (delegating to inner), and `Execute` (delegating then truncating).
- Compile-time interface check: `var _ ToolRegistry = (*TruncatingToolRegistry)(nil)`.
- Add unit tests: no truncation below limit, truncation above limit, marker text, error passthrough, zero/negative maxTokens disables truncation, all delegated methods work correctly.

### Task 3: Conversation Compactor
**Module:** `vage/memory/`
**Files:** `compactor.go`, `compactor_test.go`
- Implement `ConversationCompactor` with `Compact`, `EstimateTokens`, `Summarizer`, `WithMaxInputTokens` methods.
- Implement `CompactIfNeeded` standalone function.
- Protected message identification algorithm (system prompt + N turn pairs from the end).
- Summary message construction with `RoleSystem` and metadata.
- Summarizer input truncation when exceeding `maxInputTokens`.
- Unit tests: no compression when few messages, correct protected turn identification, summary message structure, token estimation, messages with existing summaries, summarizer input truncation.

### Task 4: Context Overflow Detection
**Module:** `vage/largemodel/`
**Files:** `overflow.go`, `overflow_test.go`
- Implement `IsContextOverflowError` with `StatusCode`, `Code`, and `Message` checks.
- Unit tests: various `APIError` status codes, error codes, message patterns, nil error, non-API errors.

### Task 5: Configuration
**Module:** `vv/configs/`
**Files:** `config.go`, `config_test.go`
- Add `ContextConfig` struct to `Config` with `*float64` for `CompressionThreshold`.
- Add `EffectiveCompressionThreshold()` method.
- Add defaults in `applyDefaults` (skip CompressionThreshold since it uses pointer semantics).
- Add env var overrides in `Load`.
- Update existing tests, add new tests for defaults and env var parsing.

### Task 6: Setup Wiring
**Module:** `vv/setup/`
**Files:** `setup.go`, `setup_test.go`
- Wire `TruncatingToolRegistry` around each agent's tool registry (including explorer).
- Create the conversation compactor summarizer using the LLM client with `WithMaxInputTokens`.
- Add `Compactor` field to `InitResult`.

### Task 7: CLI Integration -- Proactive Compact
**Module:** `vv/cli/`
**Files:** `cli.go`, `messages.go`
- Add `compactor`, `estimatedTokens`, `contextCfg` fields to `App`.
- Add `emergencyCompacted` field to `model`.
- Add `compactionDoneMsg` and `emergencyCompactResultMsg` types to `messages.go`.
- Update `New()` constructor.
- In `handleSubmit`: update `estimatedTokens` (no LLM call here).
- In `invokeAgent` (async): check threshold, call `CompactIfNeeded`, send `compactionDoneMsg`.
- In `handleStreamDone`: update token estimate after adding agent response.
- Handle `compactionDoneMsg` in `Update`.

### Task 8: CLI Integration -- Reactive Compact
**Module:** `vv/cli/`
**Files:** `cli.go`
- In `handleStreamDone`: detect context overflow via `IsContextOverflowError`.
- On overflow: dispatch async `emergencyCompactCmd`, set `emergencyCompacted = true`.
- Handle `emergencyCompactResultMsg` in `Update`: apply compressed history, retry or show error.
- Prevent infinite retry via `emergencyCompacted` flag.

### Task 9: CLI Display and /compact Command
**Module:** `vv/cli/`
**Files:** `render.go`, `memory.go`
- Add `renderSummaryMessage` function.
- Detect messages with `Metadata["compressed"] == true` in display rendering.
- Add `/compact` slash command for manual compaction.

### Task 10: HTTP Mode Integration
**Module:** `vv/httpapis/`
**Files:** Handler files that build `RunRequest`.
- Inject `compactor` and `ContextConfig` into HTTP handler.
- Apply `CompactIfNeeded` before dispatching.
- Add reactive compression on overflow error with single retry.

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
        (CLI      (CLI     (HTTP
        proactive) reactive) handler)
              |
              v
           Task 9 (CLI display + /compact)
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
- Verify all delegated methods (`Register`, `Unregister`, `Get`, `List`, `Merge`) pass through correctly.
- Verify compile-time interface compliance.

### Test 2: Conversation Compactor (Integration -- needs LLM)
**Location:** `vage/integrations/compressor_tests/compactor_test.go`
- Build a conversation with 20+ messages (mix of user, assistant, tool).
- Create a `ConversationCompactor` with a real LLM summarizer and `protectedTurns=2`.
- Call `Compact` and verify:
  - System prompt is preserved.
  - Last 2 user/assistant pairs are preserved verbatim.
  - A summary message exists with `compressed` metadata and `RoleSystem`.
  - Total token count is reduced.

### Test 3: Conversation Compactor with Input Truncation (Unit)
**Location:** `vage/memory/compactor_test.go`
- Create a compactor with `WithMaxInputTokens(100)`.
- Provide eligible messages totaling 500+ estimated tokens.
- Verify the summarizer receives a truncated input.

### Test 4: Context Overflow Detection (Unit -- no LLM needed)
**Location:** `vage/largemodel/overflow_test.go`
- `APIError` with status 413 -> returns true.
- `APIError` with code `"context_length_exceeded"` -> returns true.
- `APIError` with message containing "maximum context length" -> returns true.
- `APIError` with status 400, code "invalid_request", no relevant message -> returns false.
- Nil error -> returns false.
- Non-API error with "context_length_exceeded" in message -> returns true.

### Test 5: Configuration Defaults and Overrides (Unit)
**Location:** `vv/configs/config_test.go`
- Load config with no context section -> verify defaults applied.
- Load config with explicit values -> verify they are used.
- Load config with `CompressionThreshold: 0.5` -> verify 0.5 is used (not overridden by default).
- Set environment variables -> verify overrides take effect.
- Verify `EffectiveCompressionThreshold()` returns 0.8 when nil, and the set value otherwise.

### Test 6: CompactIfNeeded Utility (Unit)
**Location:** `vage/memory/compactor_test.go`
- Messages below threshold -> returns original, compacted=false.
- Messages above threshold -> returns compressed, compacted=true.
- Nil compactor -> returns original, compacted=false.

### Test 7: End-to-End Proactive Compression (Integration -- needs LLM)
**Location:** `vv/integrations/cli_tests/compression_test.go`
- Create a CLI `App` with small `ModelMaxContextTokens` (e.g., 4000) and low threshold (0.5).
- Simulate adding messages to history until token estimate exceeds threshold.
- Verify that compression is triggered (history length decreases).
- Verify a summary message is present in the compressed history.

### Test 8: TruncatingToolRegistry Interface Compliance (Unit)
**Location:** `vage/tool/truncate_test.go`
- Verify `TruncatingToolRegistry` satisfies the `ToolRegistry` interface.
- Verify `List()` passes through unchanged.
- Verify error results are not truncated.
