# Requirement: Conversation Context Compression (Auto-Compact)

## 1. Background and Objectives

### Background

vv agent accumulates all messages (user, agent, tool calls, tool results) in the CLI Session conversation history throughout a session. This conversation history is passed in full to the LLM on every agent invocation (see CLIMSG-03). When the cumulative token count of the conversation approaches or exceeds the model's context window limit (typically 128K--200K tokens), the LLM call fails, terminating the session.

The existing **Session Memory** system (facts + summary with a 4,000-token budget) provides a *storage-level* compression mechanism for persisting conversation context across turns, but it does **not** manage the size of the **live conversation history** that is sent to the LLM. These are two distinct concerns:

- **Session Memory**: A compact representation of accumulated knowledge (facts, summary) used as supplementary context. Budget: ~4K tokens.
- **Conversation History**: The full sequence of CLI Messages (user, agent, tool, system, etc.) passed as the LLM message array. Budget: the model's entire context window.

No mechanism currently monitors, limits, or compresses the conversation history. This makes vv unsuitable for long-running sessions or complex multi-step tasks.

### Objectives

1. **Prevent context overflow**: Ensure vv can handle arbitrarily long conversations without hitting model context window limits.
2. **Preserve conversation quality**: Compression must retain enough context for the agent to continue working coherently.
3. **Minimize disruption**: Compression should be transparent to the user and not interrupt the conversation flow.
4. **Reuse existing infrastructure**: Leverage vage's existing memory compression capabilities where possible.

## 2. Assumptions

The following assumptions are made in lieu of direct user clarification:

- **A1**: Token counting can use a simple heuristic (e.g., ~4 characters per token) rather than requiring an exact tokenizer. Exact tokenizers are model-specific and add external dependencies; the heuristic is sufficient for triggering compression at a safe threshold.
- **A2**: The compression (summarization) LLM call uses the same model as the conversation. A separate, cheaper model for summarization is a future optimization.
- **A3**: The auto-compact feature is always enabled; there is no configuration to disable it entirely. The threshold is configurable.
- **A4**: Tool output truncation applies at the time of tool result insertion into conversation history, not retroactively to already-inserted messages.
- **A5**: The model's maximum context window size is either known from the model configuration or defaults to a conservative value (e.g., 128,000 tokens).
- **A6**: HTTP mode conversations also benefit from context compression, using the same mechanism applied to the message array before LLM invocation.
- **A7**: When auto-compact triggers, a brief system notification is shown to the user in CLI mode so they are aware context was compressed.

## 3. User Stories

### US-1: Tool Output Truncation

**As a** User conducting a long coding session,
**I want** excessively large tool outputs (e.g., reading a huge file, verbose bash output) to be automatically truncated,
**So that** a single tool result does not consume a disproportionate share of my context window.

**Acceptance Criteria:**
- AC-1.1: When a tool result exceeds the configured `tool_output_max_tokens` threshold, the result is truncated to that limit.
- AC-1.2: Truncated results have a `[truncated: showing first N of M estimated tokens]` marker appended.
- AC-1.3: Truncation applies to all tool types (bash, read, grep, glob, etc.).
- AC-1.4: The truncation threshold is configurable (default: 8,000 tokens).
- AC-1.5: Truncation happens at tool result insertion time, before the result enters conversation history.

### US-2: Automatic Conversation Compaction

**As a** User working on a complex multi-step task,
**I want** the conversation history to be automatically compressed when it approaches the model's context limit,
**So that** my session can continue indefinitely without failing due to context overflow.

**Acceptance Criteria:**
- AC-2.1: The system tracks the estimated cumulative token count of the conversation history.
- AC-2.2: When the estimated token count exceeds the configured compression threshold (default: 80% of model max context), auto-compact is triggered before the next LLM call.
- AC-2.3: Auto-compact produces a summary of the compressible portion of the conversation and replaces it with a single summary message.
- AC-2.4: Protected messages (system prompt and the last N user/assistant exchanges, where N is configurable, default: 4 turns) are never compressed.
- AC-2.5: After compaction, the conversation continues normally with no loss of the most recent context.
- AC-2.6: In CLI mode, a system notification is displayed when auto-compact occurs (e.g., "[context compressed: summarized N messages to stay within model limits]").

### US-3: Reactive Compaction on Context Overflow

**As a** User whose conversation hits the model context limit despite proactive compression,
**I want** the system to recover gracefully from a context overflow error,
**So that** my session does not terminate abruptly.

**Acceptance Criteria:**
- AC-3.1: If the LLM returns a context overflow error (e.g., HTTP 413 or equivalent error code/message), the system triggers an emergency compact.
- AC-3.2: Emergency compact uses a more aggressive compression: fewer protected turns (minimum: last 1 user/assistant exchange) and summarizes everything else.
- AC-3.3: After emergency compact, the failed LLM call is retried once with the compressed history.
- AC-3.4: If the retry also fails, the error is reported to the user with a suggestion to start a new session.

### US-4: Context Compression Configuration

**As an** Operator deploying vv,
**I want** to configure context compression parameters,
**So that** I can tune the behavior for different models and use cases.

**Acceptance Criteria:**
- AC-4.1: The following configuration options are available:
  - `model_max_context_tokens` -- Maximum context window of the model (default: 128000).
  - `context_compression_threshold` -- Fraction of max context at which auto-compact triggers (default: 0.8).
  - `tool_output_max_tokens` -- Maximum token count for a single tool result before truncation (default: 8000).
  - `context_protected_turns` -- Number of recent user/assistant turn pairs to protect from compression (default: 4).
- AC-4.2: All options have sensible defaults and are optional.
- AC-4.3: Options can be set via the YAML config file and overridden via environment variables.

## 4. Scope Boundaries

### In Scope

- Token counting heuristic for estimating conversation history size
- Tool output truncation at insertion time
- Proactive auto-compact when approaching context limits
- Reactive compact on context overflow errors
- Summary generation for compressed conversation segments (reusing vage memory summarization)
- Protected message rules (system prompt + recent turns)
- CLI notification when compression occurs
- New configuration options for compression parameters
- Applies to both CLI and HTTP mode conversations

### Out of Scope

- Exact tokenizer integration (model-specific tokenizers like tiktoken)
- Separate/cheaper model for summarization calls
- Micro-compact (replacing stale tool results with cache markers) -- future enhancement
- Snip-compact (trimming individual long messages other than tool output) -- future enhancement
- User-configurable token budget hints or nudge messages
- Persistent storage of compressed conversation history
- Compression of the session memory itself (already has its own budget management)
- Per-agent or per-sub-agent context management (compression applies at the conversation level)

## 5. Involved Roles

| Role | Involvement |
|------|-------------|
| **Operator** | Configures compression parameters (`model_max_context_tokens`, `context_compression_threshold`, `tool_output_max_tokens`, `context_protected_turns`) |
| **User** | Observes compression notifications in CLI; benefits from longer sessions |
| **System** | Performs token counting, triggers compression, executes summarization, handles overflow errors |
| **Orchestrator Agent** | Receives compressed conversation history; behavior is unchanged |
| **All Sub-Agents** | Receive tool output that may be truncated; behavior is unchanged |

No new roles are introduced.

## 6. Involved Models and State Changes

### Modified Models

#### Configuration (model-configuration.md)

New attributes:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| model_max_context_tokens | Maximum context window size of the LLM in tokens | number | No | Default: 128000 |
| context_compression_threshold | Fraction of max context at which auto-compact triggers | number (0.0--1.0) | No | Default: 0.8 |
| tool_output_max_tokens | Maximum tokens for a single tool result before truncation | number | No | Default: 8000 |
| context_protected_turns | Number of recent turn pairs protected from compression | number | No | Default: 4 |

#### CLI Session (model-cli-session.md)

New attributes:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| estimated_token_count | Running estimate of total conversation token count | number | Yes | Updated after each message is added; used to trigger compression |

New state:

| State | Description | Notes |
|-------|-------------|-------|
| compressing | Conversation context is being compressed | Brief transient state during auto-compact |

New state transitions:

| Current State | Trigger Action | Target State | Preconditions | Post Actions |
|---------------|----------------|--------------|---------------|--------------|
| processing | Token count exceeds compression threshold before LLM call | compressing | estimated_token_count > threshold | Begin summarization of eligible messages |
| compressing | Compression complete | processing | Summary generated | Replace eligible messages with summary message; update estimated_token_count; display notification; proceed with LLM call |

#### CLI Message (model-cli-message.md)

New attribute:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| estimated_tokens | Estimated token count of this message's content | number | No | Calculated at creation time; used for cumulative tracking |

New role value:

| Value | Label | Description |
|-------|-------|-------------|
| context_summary | Context Summary | A summary message that replaces a set of compressed messages |

### No New Models

No entirely new models are introduced. The changes extend existing models.

## 7. Involved Business Processes

### Modified Processes

#### CLI Message Processing (procedure-cli-message-processing.md)

New step inserted between current Step 2 (Create User Message) and Step 3 (Invoke Orchestrator Agent):

**Step 2a: Check Context Budget**
- **Executing Role**: System
- **Description**: After adding the user message to conversation history, estimate the cumulative token count. If it exceeds the compression threshold (`context_compression_threshold * model_max_context_tokens`), trigger auto-compact before invoking the agent.
- **Input**: Conversation history, configuration
- **Output**: Possibly compressed conversation history
- **Model State Changes**: CLI Session status -> compressing (briefly) -> processing; conversation messages may be replaced; estimated_token_count updated

**Step 2b: Auto-Compact (Conditional)**
- **Executing Role**: System
- **Description**: Identify messages eligible for compression (all messages except: system prompt, last `context_protected_turns` user/assistant turn pairs, and any existing context_summary messages that are within the protected window). Summarize eligible messages using vage's memory summarization. Replace eligible messages with a single context_summary message. Update estimated_token_count. Display a system notification in CLI.
- **Input**: Eligible messages, summarization service
- **Output**: Compressed conversation history with summary message
- **Business Rules**: Protected messages are never compressed. The summary must capture key decisions, file changes, and task progress.

New step added to Step 6 (Complete Processing) as a sub-step:

**Step 6a: Handle Context Overflow Error**
- **Executing Role**: System
- **Description**: If the agent execution failed due to a context overflow error from the LLM, trigger emergency compact with reduced protection (1 turn pair), then retry the agent invocation once.
- **Input**: Context overflow error
- **Output**: Retry result or final error
- **Model State Changes**: Conversation compressed more aggressively; retry attempted

New business rules:

| Rule ID | Rule Name | Rule Description | Applicable Scenario |
|---------|-----------|------------------|---------------------|
| CLIMSG-09 | Token Estimation | Each message's token count is estimated at creation time and added to the session's running total | Message creation |
| CLIMSG-10 | Proactive Compression | When estimated token count exceeds the configured threshold, auto-compact runs before the next LLM call | Step 2a |
| CLIMSG-11 | Protected Messages | System prompt and the last N turn pairs (configurable) are never eligible for compression | Step 2b |
| CLIMSG-12 | Compression Notification | A system message is displayed when auto-compact occurs, indicating how many messages were summarized | Step 2b |
| CLIMSG-13 | Reactive Compression | On context overflow error, emergency compact with minimum protection is attempted before retry | Step 6a |
| CLIMSG-14 | Tool Output Truncation | Tool results exceeding `tool_output_max_tokens` are truncated with a marker before insertion into conversation | Tool result handling |

#### Session Memory Management (procedure-session-memory-management.md)

No structural changes. Session memory management continues to operate independently on its own token budget. However, the context assembly step (Step 3) should account for the fact that conversation history may now contain context_summary messages. These are treated as regular messages in the conversation array.

#### Synchronous/Streaming/Async Request (HTTP mode procedures)

The same compression logic applies when assembling the message array for LLM calls in HTTP mode. The implementation is shared -- compression runs at the point where conversation history is prepared for LLM invocation, regardless of interface mode.

### New Processes

No entirely new top-level processes. The compression behavior is integrated into the existing CLI Message Processing flow and the agent invocation pipeline.

## 8. Involved Applications and Pages

### CLI Application

#### Main Conversation View (001-main-conversation-view.md)

New behavior:
- **Compression notification**: When auto-compact occurs, a system message is rendered in the conversation output area: `[context compressed: summarized N messages to stay within model limits]`. This uses the existing system message rendering (role: "system").
- **Context summary display**: Messages with role `context_summary` are displayed as dimmed/muted text blocks (similar to system messages) with a heading like "Previous context (summarized)".

No new pages are required.

### HTTP API Application

No new endpoints. The compression is transparent to API consumers -- they send messages and receive responses. The compression happens internally when preparing the LLM call. Optionally, a `context_compressed` flag could be included in response metadata, but this is not required for the MVP.

## 9. Summary

This requirement adds a two-stage conversation context compression system to vv:

1. **Tool Output Truncation** -- prevents individual oversized tool results from consuming excessive context by truncating them at a configurable threshold (default: 8,000 tokens).

2. **Auto-Compact** -- monitors the cumulative token count of the conversation history and, when it approaches the model's context limit (default: 80% threshold), compresses older messages into a summary while preserving the system prompt and recent exchanges.

3. **Reactive Compact** -- provides a safety net by catching context overflow errors from the LLM and performing emergency compression with a retry.

The feature extends four existing product elements:
- **Configuration model** -- four new optional attributes
- **CLI Session model** -- token tracking attribute and a transient `compressing` state
- **CLI Message model** -- per-message token estimate and a new `context_summary` role
- **CLI Message Processing procedure** -- compression check and auto-compact steps

It reuses vage's existing memory summarization capability for generating conversation summaries, introduces no new models or top-level processes, and requires no changes to agent behavior -- agents receive a conversation history that may contain summary messages but are otherwise unaffected.
