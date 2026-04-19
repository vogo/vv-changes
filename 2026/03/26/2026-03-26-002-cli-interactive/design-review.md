# CLI Interactive Mode - Design Review

## Review Summary

The proposed design is well-structured and demonstrates a solid understanding of the bubbletea ecosystem and the vage framework's streaming architecture. However, several issues were found by validating the design against the actual codebase. The issues range from interface mismatches to missing goroutine safety considerations and incomplete event handling.

## Critical Issues

### 1. Router Agent Does Not Stream Sub-Agent Events

**Problem:** The design states that calling `router.RunStream()` will produce a full event stream including `EventTextDelta`, `EventToolCallStart`, etc. In reality, `routeragent.RunStream()` calls `agent.RunToStream()`, which wraps a blocking `Run()` call and emits only `EventAgentStart` and `EventAgentEnd`. All intermediate events (text deltas, tool calls, tool results, iteration starts) are lost.

This means the CLI will NOT receive streaming text deltas or tool call events when going through the router -- it will receive a single `EventAgentEnd` containing the final message, defeating the entire streaming UX.

**Fix:** The CLI must not call `router.RunStream()`. Instead, it should either:
- (a) Call the router's `Run()` (non-streaming) first to resolve the target agent, then call the target agent's `RunStream()` directly. This requires extracting the routing decision from the router without executing the sub-agent.
- (b) Replicate the routing logic in the CLI: call `routeragent.LLMFunc` to select the sub-agent, then call the sub-agent's `RunStream()`.

Option (b) is cleaner because it avoids modifying the vage framework. The routing function is already available as `routeragent.LLMFunc` and can be invoked independently.

### 2. confirmingRegistry Does Not Implement ToolRegistry

**Problem:** The design proposes a `confirmingRegistry` that wraps `tool.ToolRegistry` and overrides `Execute()`. However, `ToolRegistry` is a full interface with 6 methods (`Execute`, `Register`, `Unregister`, `Get`, `List`, `Merge`). The design only shows `Execute()`, meaning `confirmingRegistry` does not satisfy `tool.ToolRegistry` and cannot be passed as a tool registry to the task agent.

**Fix:** The `confirmingRegistry` must either:
- (a) Embed the inner `tool.ToolRegistry` and only override `Execute()`.
- (b) Instead of wrapping the registry, wrap the `ToolHandler` functions at registration time.

Option (a) is straightforward:
```go
type confirmingRegistry struct {
    tool.ToolRegistry  // embed to delegate all other methods
    confirmTools map[string]bool
    confirmFn    func(ctx context.Context, toolName, args string) (bool, error)
}
```

### 3. Tool Registry Type Mismatch

**Problem:** The `agents.Create()` function takes `*tool.Registry` (concrete type), not `tool.ToolRegistry` (interface). The `confirmingRegistry` wrapper cannot be passed to `agents.Create()` because the function signature requires the concrete `*tool.Registry`.

**Fix:** The CLI should not wrap the registry before passing it to agent creation. Instead, it should wrap the registry *after* agent creation by replacing the tool registry on the task agent. However, `taskagent.Agent` does not expose a method to replace the tool registry after construction.

A better approach: the CLI should use a `tool.ToolHandler` wrapper instead of a registry wrapper. Register tools with a confirming handler that delegates to the original handler. But this also requires changes since `Register()` takes a `ToolHandler`.

The most practical approach: add the confirmation logic as a `StreamMiddleware` on the task agent, or use the existing `ExternalToolCaller` mechanism. Alternatively, pass the concrete `*tool.Registry` to `agents.Create()` as usual, then set the `ExternalToolCaller` on the registry to intercept execution.

Actually the cleanest solution is: since `*tool.Registry` is already passed to agents and `Registry.Execute()` is the entry point, create a `confirmingExecutor` that wraps the `*tool.Registry` using Go's embedding, where `Execute` is overridden but `Register`/`List`/etc. delegate to the embedded `*tool.Registry`. Then pass this to `taskagent.WithToolRegistry()` which accepts `tool.ToolRegistry` (interface). The `agents.Create` function needs to be refactored to accept `tool.ToolRegistry` instead of `*tool.Registry`, or the CLI creates agents directly.

### 4. Confirmation Channel Deadlock Risk

**Problem:** The `confirmFn` blocks on `confirmCh` waiting for the user's decision. If the stream context is cancelled (e.g., user presses Ctrl+C) while `confirmFn` is blocked, the goroutine will leak because nobody sends on `confirmCh`.

**Fix:** `confirmFn` must select on both `confirmCh` and `ctx.Done()`:
```go
select {
case approved := <-confirmCh:
    return approved, nil
case <-ctx.Done():
    return false, ctx.Err()
}
```

## Moderate Issues

### 5. Missing Conversation History Across Turns

**Problem:** The design mentions `[]DisplayMessage` for rendering but does not address passing conversation history back to the agent on subsequent turns. Each agent invocation receives only the current user message, losing multi-turn context.

**Fix:** The CLI must accumulate `schema.Message` entries (not just `DisplayMessage`) and pass the full conversation history in each `RunRequest.Messages`. This is critical for coherent multi-turn conversations.

### 6. p.Send() Not Available from tea.Cmd Goroutine

**Problem:** The design says the stream consumer goroutine sends events "back to the bubbletea program via `p.Send()`". However, `tea.Cmd` functions do not have access to the `tea.Program` instance. The standard bubbletea pattern is for `tea.Cmd` to return a single `tea.Msg`.

**Fix:** The `tea.Program` must be stored after calling `tea.NewProgram()` and passed to the stream consumer goroutine. The `App.Run()` method should create the program, store a reference, then call `p.Run()`. The stream goroutine uses `p.Send()` to inject messages. This is a valid bubbletea pattern but must be explicit.

### 7. Event Type Handling Incomplete

**Problem:** The design lists handling for `EventTextDelta`, `EventToolCallStart`, `EventToolCallEnd`, `EventToolResult`, and `EventAgentEnd`. It does not mention:
- `EventIterationStart` -- emitted at the start of each ReAct loop iteration
- `EventLLMCallStart` / `EventLLMCallEnd` / `EventLLMCallError` -- emitted by the metrics middleware
- `EventTokenBudgetExhausted` -- emitted when the token budget is exceeded
- `EventError` -- only mentioned in passing

**Fix:** The `Update` handler should explicitly handle or ignore all known event types. At minimum, `EventIterationStart` should be displayed (e.g., "Iteration 2/10..."), and `EventTokenBudgetExhausted` should warn the user.

### 8. Session ID Generation

**Problem:** The design says `sessionID` is generated with `crypto/rand` "matching the existing pattern in `service.go`". This is correct, but the implementation detail is missing from the code snippets.

**Fix:** Explicitly use the same pattern:
```go
b := make([]byte, 8)
_, _ = rand.Read(b)
sessionID := hex.EncodeToString(b)
```

## Minor Issues

### 9. huh Confirmation Integration Pattern

**Problem:** The design mentions using huh's bubbletea integration but does not specify how the huh form is embedded in the bubbletea model. huh forms in bubbletea mode require calling `huh.NewForm().WithShowHelp(false)` and embedding the form as a `tea.Model` within the parent model's `Update` and `View`.

**Fix:** Specify that the confirmation dialog uses `huh.NewConfirm()` wrapped in a `huh.NewForm()`, and the form's `Init()`, `Update()`, and `View()` are delegated to from the parent model when in `statusConfirming` state.

### 10. Glamour Re-rendering Performance

**Problem:** The design says glamour rendering is applied "when the response completes (re-rendering the full response)". For long responses, this causes a visible flash/jump as raw text is replaced with rendered markdown.

**Fix:** Consider rendering through glamour incrementally by accumulating complete markdown blocks (paragraphs, code blocks) and rendering them as they complete. Alternatively, accept the flash and document it as a known limitation for v1.

### 11. Missing `agents.Create` Refactoring in Task List

**Problem:** If the confirmation wrapper approach requires passing `tool.ToolRegistry` instead of `*tool.Registry`, the `agents.Create()` function signature needs updating. This is not mentioned in any task.

**Fix:** Add a task to refactor `agents.Create()` to accept `tool.ToolRegistry` (interface) instead of `*tool.Registry` (concrete), or have the CLI create agents directly without using the shared `agents.Create()` helper.

### 12. Missing Error Display in TUI

**Problem:** The design does not specify how errors (LLM API failures, network errors, tool execution failures) are displayed in the TUI. `streamDoneMsg` carries an error but the display behavior is unspecified.

**Fix:** Errors should be displayed as styled system messages (e.g., red text) in the viewport, and the model should transition to `statusIdle` so the user can retry.

### 13. Terminal Alternate Screen

**Problem:** The design mentions terminal restoration but does not specify whether to use bubbletea's alternate screen mode (`tea.WithAltScreen()`). Alternate screen mode is better for full-screen TUIs but prevents seeing output after the program exits.

**Fix:** Recommend using alternate screen mode for the TUI, which bubbletea handles automatically including cleanup on exit/crash.

### 14. slog Output Conflicts with TUI

**Problem:** The existing codebase uses `slog` for logging throughout (main.go, agents, tools). When the TUI is active, slog output to stderr will corrupt the terminal display.

**Fix:** In CLI mode, redirect slog output to a file or suppress it entirely. Add a `slog.SetDefault()` call with a file handler or `slog.DiscardHandler` before starting the TUI.
