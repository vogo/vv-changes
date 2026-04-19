# Code Review: CLI Interactive Mode

## Files Reviewed

### New files (cli package)
- `vv/cli/cli.go` -- App struct, bubbletea model, TUI logic
- `vv/cli/messages.go` -- bubbletea message types, session status, DisplayMessage
- `vv/cli/render.go` -- styling and markdown rendering helpers
- `vv/cli/confirm.go` -- confirmingExecutor wrapping tool.ToolRegistry
- `vv/cli/cli_test.go` -- tests for App, selectAgent, multi-turn history
- `vv/cli/confirm_test.go` -- tests for confirmingExecutor
- `vv/cli/render_test.go` -- tests for rendering and truncation helpers

### Modified files
- `vv/main.go` -- mode branching (cli/http), flag handling, WrapRegistry integration
- `vv/config/config.go` -- Mode and CLIConfig fields, VV_MODE env var
- `vv/config/config_test.go` -- tests for new config fields
- `vv/agents/agents.go` -- changed Create() parameter from *tool.Registry to tool.ToolRegistry

## Issues Found and Fixed

### 1. Unused type `confirmResultMsg` (messages.go)

**Severity:** Low (dead code)

The `confirmResultMsg` struct was defined but never referenced anywhere in the codebase. The confirmation flow uses `confirmCh chan bool` directly instead.

**Fix:** Removed the unused type definition.

### 2. Non-idiomatic EOF comparison (cli.go)

**Severity:** Low (correctness risk)

Line 383 used `recvErr == io.EOF` instead of `errors.Is(recvErr, io.EOF)`. While `io.EOF` is a sentinel value and direct comparison typically works, `errors.Is` is the idiomatic Go pattern and handles wrapped errors correctly.

**Fix:** Changed to `errors.Is(recvErr, io.EOF)`.

### 3. State mutation inside refreshViewport (cli.go)

**Severity:** Medium (correctness)

`refreshViewport()` was supposed to be a pure rendering method but contained logic that mutated `m.app.messages` -- appending or updating agent messages during streaming. This caused several problems:

- Every call to `refreshViewport()` (from `appendSystemMessage`, `appendToolMessage`, etc.) would trigger the agent message tracking logic, not just text delta events.
- The method mixed concerns: rendering and state management were interleaved.
- Potential for duplicate agent messages if `refreshViewport` was called when no streaming output existed yet.

**Fix:** Moved the agent message tracking logic from `refreshViewport()` into `updateOutputInViewport()`, which is only called on text delta events. `refreshViewport()` now only reads `m.app.messages` without mutating it. The "Agent: " prefix is applied uniformly in `refreshViewport` for all agent-role messages, and `renderAgentMessage` no longer adds the prefix (avoiding duplication).

### 4. Verbose no-op wireConfirmFn (cli.go)

**Severity:** Low (readability)

The `wireConfirmFn` method was a no-op with a long internal monologue as comments, including crossed-out reasoning. This is confusing for readers.

**Fix:** Replaced with a concise comment explaining it is a placeholder.

## Items Reviewed -- No Issues

- **Config changes (config.go):** Mode field with "cli" default, CLIConfig with ConfirmTools, VV_MODE env override -- all correct and well-tested.
- **agents.Create interface change (agents.go):** Changed from `*tool.Registry` to `tool.ToolRegistry` -- backward-compatible, clean.
- **confirmingExecutor (confirm.go):** Correct embedding pattern, proper context cancellation in confirmFn, thorough test coverage.
- **main.go mode branching:** Proper default to CLI mode, flag override, route setup for CLI path.
- **Error handling:** Stream errors, routing errors, and LLM errors all surface as user-visible messages.
- **Goroutine safety:** The stream consumer goroutine communicates via `program.Send()` (thread-safe), confirmation uses a buffered channel with context awareness.
- **Test coverage:** Good coverage for selectAgent, multi-turn history, confirmingExecutor (approve/reject/passthrough/cancel/error/delegation), rendering helpers, exit commands, config.

## Observations (No Action Taken)

- `renderMarkdown` creates a new `glamour.NewTermRenderer` on every call. The design doc suggested creating it once at startup. Since it is only called on `EventAgentEnd` (once per response), the performance impact is negligible. Could be optimized later if needed.
- The `invokeAgent` tea.Cmd returns `nil` after spawning the stream consumer goroutine. This is valid in bubbletea (nil messages are ignored) but is an unusual pattern. The alternative would be to use `program.Send()` for everything including error returns from `selectAgent`/`RunStream`, but the current approach works correctly.
- `wireConfirmFn` is a no-op placeholder. The confirmation dialog UI is wired up but the TUI-to-channel bridging for real tool confirmation during agent runs is not yet connected. The default `confirmFn` allows all tools.
