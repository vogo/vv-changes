# Design Review: Conversation Context Compression (Auto-Compact)

## Summary

The proposed design is well-structured and covers all requirement areas. The component interaction diagram is clear, and the overall architecture of placing core logic in `vage/memory/` with integration in `vv/` follows the existing dependency flow. Below are specific issues and improvements organized by severity.

---

## CRITICAL Issues

### C1: TruncatingToolRegistry Cannot Implement ToolRegistry Interface

The `ToolRegistry` interface (in `vage/tool/tool.go`) has six methods: `Execute`, `Register`, `Unregister`, `Get`, `List`, and `Merge`. The design proposes `TruncatingToolRegistry` as a decorator that wraps `ToolRegistry`, but only mentions implementing `List()` and `Execute()`.

The remaining mutation methods (`Register`, `Unregister`, `Get`, `Merge`) must also be delegated. More importantly, a decorator around the full `ToolRegistry` interface is heavyweight -- the only method that needs interception is `Execute`.

**Recommendation**: Instead of wrapping the entire `ToolRegistry`, implement tool output truncation as a `ToolExecutor` wrapper (the minimal interface with just `Execute`). However, since the setup code passes `tool.ToolRegistry` to `FactoryOptions`, the simplest correct approach is to delegate all 6 methods, with only `Execute` adding truncation behavior. The design must explicitly enumerate all delegated methods.

### C2: Token Estimate Only Counts Text in CLI History, Missing Tool Calls

The design states: "The CLI manages `history` directly (appending user messages before invoking, and agent messages after)." But examining `handleStreamDone`, only the final text output is appended to `m.app.history`. Tool call/result messages that the TaskAgent processes internally are NOT in `m.app.history` -- they live in the TaskAgent's session memory, not in the CLI history.

This means the CLI's `estimatedTokens` counter only tracks user + agent final-text messages. If the actual `req.Messages` sent to the orchestrator is just this history, the token estimate may be reasonably accurate. However, the orchestrator and sub-agents prepend their own system prompts and session history. The proactive threshold should account for this overhead.

**Recommendation**: Add a safety margin (e.g., reserve 10-15% of the context window for system prompts and agent overhead) when computing the threshold. Document this explicitly.

---

## HIGH Issues

### H1: ConversationCompactor Duplicates SummarizeAndTruncCompressor

The existing `SummarizeAndTruncCompressor` already does: split messages into older/recent, summarize older, prepend summary with metadata. The proposed `ConversationCompactor` adds system-prompt awareness and turn-pair counting, but the core summarize-and-keep-recent logic is identical.

**Recommendation**: Implement `ConversationCompactor` as a thin orchestrator that:
1. Extracts the system prompt (first message if role==system).
2. Identifies protected tail by walking backward for N turn pairs.
3. Delegates the actual summarization of the eligible middle segment to the existing `Summarizer` function type directly.
This avoids duplicating the summary message construction and metadata stamping pattern. The design already does this conceptually but should explicitly note reuse of metadata conventions from `SummarizeAndTruncCompressor`.

### H2: Summarizer Call Could Exceed Context Window Itself

When auto-compact triggers, the eligible messages are sent to the LLM as a summarization prompt. If the eligible messages are very large (which is exactly when compaction triggers), the summarization call itself may exceed the context window.

**Recommendation**: The summarizer should truncate the input messages if they are too large. A simple approach: cap the summarization input to ~80% of the model's context window. If the input exceeds this, include only the first and last portions with a "[... N messages omitted ...]" marker. Add this as a documented concern and a simple safeguard in the summarizer construction.

### H3: Emergency Compact in handleStreamDone Runs in the Bubbletea Update Loop

`handleStreamDone` is a synchronous Bubbletea `Update` handler. Emergency compact calls `compactor.Compact()`, which makes an LLM call (the summarizer). This blocks the entire TUI event loop, freezing the UI.

**Recommendation**: Emergency compact (and its retry) should be dispatched as an async `tea.Cmd`, similar to how `invokeAgent` works. The flow should be: detect overflow error -> show notification -> dispatch async emergency compact cmd -> on completion, retry agent invocation.

### H4: Proactive Compact in handleSubmit Also Blocks the TUI

Same issue as H3. The `compactor.Compact()` call in `handleSubmit` makes an LLM summarization call synchronously before the agent invocation command is dispatched.

**Recommendation**: Move proactive compaction into the async `invokeAgent` function. This is where `req.Messages` is built anyway. The compaction can happen at the start of the `invokeAgent` closure, before calling `orchestrator.RunStream`. This keeps the TUI responsive and simplifies the code.

### H5: Dispatcher Compression for HTTP Mode is Invasive

Adding `compactor` and `compactThreshold` fields to `Dispatcher` introduces memory-management concerns into the orchestration layer, which violates the current separation of concerns. The Dispatcher handles intent recognition and execution dispatch -- it should not own context compression.

**Recommendation**: For HTTP mode, apply compression in the HTTP handler layer (`vv/httpapis/`) before constructing the `RunRequest`, mirroring how the CLI applies it before calling the orchestrator. This keeps the Dispatcher clean. If a shared utility is needed, expose it as a standalone function (e.g., `memory.CompactIfNeeded(compactor, messages, threshold)`) that both CLI and HTTP layers call.

---

## MEDIUM Issues

### M1: Missing `aimodel.APIError.Code` Field Check for Anthropic Errors

The `IsContextOverflowError` function checks `StatusCode` and `Message` but not the `Code` field. Anthropic returns errors with code `"overloaded"` for context limits and OpenAI uses `"context_length_exceeded"` as the error code. Checking `apiErr.Code` would improve detection accuracy.

**Recommendation**: Add `apiErr.Code` checks:
```go
if apiErr.Code == "context_length_exceeded" || apiErr.Code == "request_too_large" {
    return true
}
```

### M2: Race Condition on estimatedTokens

`estimatedTokens` is modified in `handleSubmit` and `handleStreamDone`, both of which run in the Bubbletea `Update` method (single-threaded). This is safe as-is because Bubbletea serializes `Update` calls. However, if compaction moves to an async `tea.Cmd` (per H4), the field must be accessed carefully -- either all reads/writes happen in `Update`, or synchronization is needed.

**Recommendation**: Keep all `estimatedTokens` reads and writes within `Update` handlers. If compaction is async, have it return a result message that `Update` processes to update the token count.

### M3: Compaction Should Also Count Tool Call Messages in History

Currently `DefaultTokenEstimator` only counts `Content.Text()`. Messages with tool calls (`ToolCalls` field on `aimodel.Message`) and tool results (passed as role=tool messages) contain significant content that is not captured by `.Content.Text()`. The `estimatedTokens` tracker will undercount.

**Recommendation**: Enhance `DefaultTokenEstimator` (or create a new estimator) to also account for tool call function names, arguments, and tool result content. Alternatively, if the CLI history truly only contains user and final-agent-text messages (no tool messages), document this assumption clearly so future refactors don't break the invariant.

### M4: Default `CompressionThreshold` of 0.0 Cannot Be Distinguished from "Not Set"

In the `applyDefaults` function, `CompressionThreshold == 0` is used to detect "not set" and apply the default of 0.8. But 0.0 could be a deliberate value (meaning "always compress"). Using zero-value detection for a float64 threshold is problematic.

**Recommendation**: Use a pointer (`*float64`) for `CompressionThreshold`, or use a sentinel value like -1 to mean "not set". Alternatively, check for the specific default range (0.0 is not a useful threshold) and document that 0.0 means "use default".

### M5: Summary Message Role Should Be RoleSystem, Not RoleUser

The design uses `aimodel.RoleUser` for summary messages to "ensure compatibility." However, a user-role summary message will be interpreted by the LLM as something the user said, which can confuse the model. The existing `SummarizeAndTruncCompressor` defaults to `RoleUser` for memory compression contexts where messages are injected as context.

For conversation history, a `RoleSystem` message is more semantically correct -- it represents system-injected context, not user input. Most LLM providers support multiple system messages (OpenAI does; Anthropic extracts system messages into the system field).

**Recommendation**: Use `aimodel.RoleSystem` for summary messages in the conversation compactor. This is semantically correct and avoids confusing the LLM.

### M6: No `/compact` Manual Command

The requirement says auto-compact is always enabled, but users may want to trigger compaction manually (e.g., before a complex operation when they know the context is bloated).

**Recommendation**: Add a `/compact` slash command to the CLI that triggers compaction on demand, similar to `/memory`. This is low effort and high utility.

---

## LOW Issues

### L1: `askUserRequestMsg` Pattern Not Listed in messages.go

The `askUserRequestMsg` type is defined in `askuser.go`, not `messages.go`. This is fine but the design references `messages.go` for adding the `context_summary` role. This role constant should follow the existing pattern.

### L2: Test Plan References Non-Existent Directories

Test 4 references `vv/integrations/cli_tests/compression_test.go`, which does not currently exist. This is expected for new feature tests but should be noted as new directory creation.

### L3: Config Environment Variable Naming

The env var `VV_MODEL_MAX_CONTEXT_TOKENS` is verbose. Consider `VV_MAX_CONTEXT_TOKENS` for brevity and consistency with other `VV_*` env vars.

### L4: Missing Concurrent Safety Documentation for ConversationCompactor

The `ConversationCompactor` should document whether it is safe for concurrent use. Since it holds no mutable state (only the summarizer function and immutable config), it is safe, but this should be stated in the doc comment.

---

## Architecture Recommendations

### R1: Consolidate Compaction as a Pre-LLM-Call Hook

Rather than embedding compaction logic in two places (CLI `handleSubmit` + Dispatcher for HTTP), consider implementing it as a `largemodel` middleware. The middleware chain already wraps `aimodel.ChatCompleter` with decorators (log, circuit breaker, rate limit, retry, timeout, cache, metrics). A "context compactor" middleware that inspects the request messages and compresses them before forwarding would:
- Apply uniformly to all code paths (CLI, HTTP, any future modes).
- Follow the existing middleware pattern.
- Keep the CLI and Dispatcher free of compression concerns.

However, this requires the middleware to have access to the conversation compactor and config, which would need to be injected. This is a design trade-off: cleaner separation vs. additional middleware complexity. Given the current design's simplicity, the CLI + HTTP handler approach is acceptable for the initial implementation, with middleware extraction as a future refactor.

### R2: Reuse SummarizeAndTruncCompressor Instead of New Type

Consider whether `ConversationCompactor` could simply be a wrapper around `SummarizeAndTruncCompressor` that:
1. Splits out the system prompt.
2. Calls `SummarizeAndTruncCompressor.Compress()` on the remaining messages with `keepLastN = protectedTurns * 2` (each turn pair is 2 messages).
3. Reassembles the result with the system prompt prepended.

This maximizes code reuse and ensures consistent metadata conventions.
