# Design Review: `ask_user` Tool

## Summary

The proposed design is solid in its overall structure: a `UserInteractor` interface at the `vage` layer with concrete implementations at `vv`, the channel-based CLI pattern mirroring `confirmingExecutor`, and the interaction store for HTTP mode. However, several architectural issues need addressing, ranging from unnecessary complexity in the HTTP event emission path to incorrect placement of components across module boundaries.

## Issues and Recommendations

### Issue 1: Context-Based EventSender Is Overcomplicated and Misplaced

**Problem:** The design spends considerable effort (and visible uncertainty in the text) trying to solve how the HTTP interactor emits a `pending_interaction` SSE event to the client. It proposes a context-based `EventSender` injected via `context.WithValue`. This is fragile, non-obvious, and couples the generic tool handler to a specific transport mechanism.

**Root Cause:** The design conflates two concerns: (a) blocking until the user responds, and (b) notifying the SSE stream. The tool handler should not care about SSE at all.

**Recommendation:** The HTTP interactor's `AskUser()` method already blocks on `interaction.done`. The stream consumer (in `vage/service/handler.go`) naturally serializes all events via `rs.Recv()`. The correct approach is to have the `ask_user` tool emit a standard `schema.Event` into the agent's stream -- which the TaskAgent already does for tool calls. Instead of a context-based `EventSender`, the `httpInteractor.AskUser()` should simply create the interaction and block. The `pending_interaction` notification should be emitted by the tool handler itself via the normal stream mechanism. Since tool handlers cannot emit events today, a simpler alternative: let the HTTP interactor return a structured "pending" marker in its response that the stream handler in `vv/httpapis/http.go` recognizes and emits as an SSE event. But even simpler: **skip the SSE event entirely from the tool handler**. Instead, emit it from the `vv/httpapis` layer by wrapping the agent stream. The HTTP serve function already has full control over the SSE stream. It can wrap the agent's `RunStream` to intercept events or inject new ones. This eliminates `context.go` entirely.

**Revised approach:** Move the `pending_interaction` event emission to `vv/httpapis/http.go` by wrapping the stream. Or, more practically, have the `HTTPInteractor.AskUser()` use a callback (set at construction time) to emit the SSE event, rather than context injection. The callback is just a `func(schema.Event)` passed to `NewHTTPInteractor()`.

Delete `vage/tool/askuser/context.go` from the plan entirely.

### Issue 2: InteractionStore and HTTP Interactor Belong in `vv/httpapis`, Not `vage`

**Problem:** The design correctly places these in `vv/httpapis`, but the narrative wavers, and some code snippets reference `vage` components. This is correct as-is in the file listing, but the design text should be clearer.

**Recommendation:** Confirmed: `InteractionStore`, `HTTPInteractor`, and the callback endpoint all stay in `vv/httpapis/`. No changes needed to `vage/service/` for this feature. The new HTTP endpoint is added to `vv/httpapis/http.go`'s custom mux, not to `vage/service/service.go`'s `buildMux()`.

### Issue 3: `ask_user` Tool Package Should Not Live Under `vage/tool/askuser/`

**Problem:** Every existing tool in `vage/tool/` (`bash`, `read`, `write`, `edit`, `glob`, `grep`) is a concrete, self-contained implementation. The proposed `askuser` package contains an interface (`UserInteractor`) with no concrete implementation -- all implementations live in `vv`. This makes `askuser` an interface-only package at the `vage` layer with no `vage`-level tests that exercise real behavior, which is an architectural anomaly.

**Recommendation:** This is acceptable given the design principle that `vage` provides the framework abstractions. The `UserInteractor` interface and `NonInteractiveInteractor` (a real implementation) do live in `vage`, which is consistent with how `tool.ToolRegistry` (interface) and `tool.Registry` (implementation) are structured. The `NonInteractiveInteractor` provides a testable default. Keep the current placement.

### Issue 4: Configuration Field Uses `int` Instead of `time.Duration`-Compatible Type

**Problem:** `AskUserTimeout int` (seconds) in the config is inconsistent with how `BashTimeout` works (also `int` seconds), but the design then converts it to `time.Duration` in the Options struct. The `0` default sentinel is also problematic: the design says "0 = no timeout" but `applyDefaults()` sets 0 to 300. These conflict.

**Recommendation:** Follow the existing `BashTimeout` pattern exactly: `AskUserTimeout int` in YAML config (seconds), default 300 in `applyDefaults()`, converted to `time.Duration` at usage sites. Remove the "0 = no timeout" claim. If a user wants effectively no timeout, they can set a very large value. This keeps the config consistent.

### Issue 5: The `ask_user` Tool Should Not Be Registered for Explorer or Planner

**Problem:** The design says to register `ask_user` into "every agent's tool registry" including explorer. But explorer is an internal agent used during the explore phase -- it runs autonomously without user interaction. The planner similarly has `MaxIterations: 1` and no tools. Giving them `ask_user` wastes tool-description tokens and risks the explorer asking user questions during the explore phase, which would stall the dispatch pipeline.

**Recommendation:** Only register `ask_user` into dispatchable agents' tool registries (coder, researcher, reviewer, chat). Do not register it for explorer or planner. The chat agent (ProfileNone) will get it since `ask_user` is registered after profile-based tools.

### Issue 6: CLI Interactor Has Circular Dependency Risk

**Problem:** The `CLIInteractor` needs a `*tea.Program` reference to send messages, but `*tea.Program` is created in `App.Run()` after `setup.Init()` has already run. The design mentions this will be "wired later" but does not specify the mechanism clearly.

**Recommendation:** Use the same deferred-wiring pattern as `confirmingExecutor`. Create the `CLIInteractor` with a nil program reference. In `App.Run()`, after creating the `tea.Program`, set `interactor.program = p`. The interactor's `AskUser()` method should panic or return an error if called before wiring (defensive check). Alternatively, use a `sync.Once`-guarded channel that gets the program reference. The simplest approach: add a `SetProgram(*tea.Program)` method on the CLI interactor and call it from `App.Run()` right after `tea.NewProgram()`.

### Issue 7: Ctrl+C During `statusAskingUser` Should Not Send to `askUserCh`

**Problem:** The design says Ctrl+C sends a "cancellation message" to `askUserCh`. But the interactor is blocking on `askUserCh` or `ctx.Done()`. Sending a string to the channel would make the agent receive that string as a "user response." Instead, Ctrl+C should cancel the context (via `runCancel()`), and the interactor should return the timeout fallback message via the `ctx.Done()` path.

**Recommendation:** On Ctrl+C during `statusAskingUser`:
1. Call `m.runCancel()` to cancel the context.
2. Transition to `statusIdle`.
3. The interactor's `AskUser()` returns via `<-ctx.Done()` with the fallback message.
Do not send anything on `askUserCh`. This is cleaner and consistent with the confirm flow's approach.

### Issue 8: Missing `schema.ToolResult` Helpers -- Verify Existing API

**Problem:** The design references `schema.TextResult("", response)` and `schema.ErrorResult("", ...)`. Need to verify these exist.

**Recommendation:** These already exist in `vage/schema/`. No issue here, just confirming.

### Issue 9: HTTP Sync Mode Fallback Not Handled

**Problem:** The requirement says "For synchronous mode, `ask_user` is not supported and returns a fallback message similar to non-interactive mode." But the design does not address how `handleRun()` (sync endpoint) would handle `ask_user`. When `a.Run()` is called synchronously, the `HTTPInteractor.AskUser()` would block waiting for a callback POST, but the sync HTTP request is still in-flight. The client has no way to learn the question or submit a response.

**Recommendation:** For the sync endpoint (`POST /v1/agents/{id}/run`), the `ask_user` tool should return a non-interactive fallback. The simplest approach: detect sync mode via context. Inject a "sync mode" marker into the context in `handleRun()`, and have the `HTTPInteractor.AskUser()` check for it. If present, return the non-interactive fallback immediately. Alternatively, use a separate `NonInteractiveInteractor` for sync-mode agents. The cleanest approach: make the `HTTPInteractor` configurable with a fallback mode, and have the `handleRun()` handler use a non-interactive interactor instead.

### Issue 10: Interaction Cleanup Goroutine Needs Lifecycle Management

**Problem:** The design mentions "automatic cleanup of expired interactions (older than 2x timeout)" but does not specify how the cleanup goroutine is started or stopped. An unbounded goroutine is a resource leak.

**Recommendation:** The `InteractionStore` should accept a `context.Context` at construction. The cleanup goroutine runs in a loop with `time.Ticker`, selecting on the context. When the context is canceled (server shutdown), the goroutine exits. Use a cleanup interval of `timeout` (not 2x timeout) and expire entries older than `2 * timeout`.

### Issue 11: Task 8 (CLI Prompt Mode) is Unnecessary

**Problem:** The non-interactive prompt mode (`-p`) uses `NonInteractiveInteractor`, which returns immediately without emitting any `pending_interaction` event. So there is no `EventPendingInteraction` to handle in `prompt.go`.

**Recommendation:** Remove Task 8 from the implementation plan. The `NonInteractiveInteractor` handles this case completely. No changes to `prompt.go` are needed.

### Issue 12: System Prompt for Chat Agent is Awkward

**Problem:** The chat agent has `ProfileNone` (no tools) and its purpose is general conversation. Adding `ask_user` to it means a chat agent that exists to answer user questions can also... ask the user questions. While technically harmless, it adds noise.

**Recommendation:** Do not add `ask_user` to the chat agent. The chat agent is already in a direct conversation with the user; it does not need a tool to ask questions -- it can just ask in its response text. Register `ask_user` only for coder, researcher, and reviewer.
