# Design: `-p <prompt>` Single-Step Execution Mode

## Architecture Overview

The feature adds a third execution branch to `vv/main.go` alongside the existing `cli` and `http` modes. When `-p <prompt>` is provided, the application bypasses both the Bubble Tea TUI and the HTTP server, instead running a simple stream-consume loop that writes agent text to stdout and diagnostic events to stderr, then exits.

The new code lives in a single new file `vv/cli/prompt.go` containing a `RunPrompt` function. This function reuses the same `agent.StreamAgent` (the `Dispatcher`) and `schema.RunRequest`/`schema.RunStream` types used by the interactive TUI, ensuring identical orchestration behavior.

```
main.go
  |
  +-- flag.Parse() adds "-p" flag
  |
  +-- Load config
  |
  +-- if NeedsSetup:
  |     - if -p set: print error, exit 1  (skip interactive prompt)
  |     - else: run interactive Prompt as before
  |
  +-- Apply flag overrides (-mode, -addr)
  |
  +-- if prompt != "" :
  |     - validate (non-empty after trim)
  |     - if cfg.Mode == "http": print error, exit 1
  |     - setup.Init() with nil WrapToolRegistry (auto-approve all tools)
  |     - signal.NotifyContext for graceful shutdown
  |     - cli.RunPrompt(ctx, dispatcher, prompt, os.Stdout, os.Stderr)
  |     - os.Exit(0 or 1)
  |
  +-- else: existing cli/http path (unchanged)
```

## Component Design

### 1. Flag Addition (`vv/main.go`)

Add a single `-p` string flag to the existing `flag` set. The flag is parsed alongside `-config`, `-addr`, and `-mode`.

```go
promptFlag := flag.String("p", "", "run a single prompt non-interactively and exit")
```

### 2. Decision Logic (`vv/main.go`)

After flag parsing and config loading, the prompt-mode branch is inserted in two places:

**First insertion point** -- immediately after config loading, *before* the existing interactive setup block:

```go
// If required config is missing and -p is set, fail fast instead of prompting.
if configs.NeedsSetup(cfg) {
    if *promptFlag != "" {
        fmt.Fprintf(os.Stderr, "vv: configuration incomplete (missing API key); "+
            "run `vv` interactively to set up, or set VV_LLM_API_KEY\n")
        os.Exit(1)
    }
    // ... existing interactive Prompt code ...
}
```

Note: `NeedsSetup` only checks `cfg.LLM.APIKey`, but `aimodel.NewClient` can fall back to `AI_API_KEY`/`OPENAI_API_KEY`/`ANTHROPIC_API_KEY` environment variables. To avoid false rejections when those env vars are set, the prompt-mode branch should attempt `setup.Init` and let it fail naturally rather than blocking solely on `NeedsSetup`. The `NeedsSetup` guard above provides a fast, user-friendly error for the common case (no key anywhere), while the `setup.Init` call below catches the remaining failures.

**Second insertion point** -- after applying flag overrides, before `switch cfg.Mode`:

1. If `*promptFlag != ""`:
   a. Validate prompt is non-empty after trimming (reject `-p ""`).
   b. If `cfg.Mode == "http"` (checking the resolved mode from flags, config, and env), print error to stderr and exit 1.
   c. Call `setup.Init(cfg, &setup.Options{})` -- nil `WrapToolRegistry` means all tools are auto-approved.
   d. Create signal context: `ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM); defer stop()`.
   e. Call `cli.RunPrompt(ctx, initResult.SetupResult.Dispatcher, *promptFlag, os.Stdout, os.Stderr)`.
   f. Exit 0 on success, 1 on error.

### 3. `RunPrompt` Function (`vv/cli/prompt.go`)

New file in the `cli` package. This is the core of the feature -- a stateless, non-interactive stream consumer.

```go
package cli

func RunPrompt(ctx context.Context, orchestrator agent.StreamAgent, prompt string, stdout, stderr io.Writer) error
```

The `stdout` and `stderr` parameters (instead of hardcoded `os.Stdout`/`os.Stderr`) enable unit testing with captured buffers.

**Behavior:**

1. Generate a transient session ID (same `crypto/rand` hex pattern used by the TUI).
2. Build a `schema.RunRequest` with a single user message and the session ID. Note: we do not use `agent.RunStreamText` because we need the session ID for memory scoping.
3. Call `orchestrator.RunStream(ctx, req)`.
4. **Immediately `defer stream.Close()`** to ensure the producer context is cancelled and resources are freed.
5. Track execution state:
   - `nestingDepth int` -- incremented on `EventSubAgentStart`, decremented on `EventSubAgentEnd`, used to indent diagnostic lines.
   - `toolCallCount int` -- tool calls in current sub-agent scope.
   - `taskStart time.Time` -- set at function entry.
   - `totalPromptTokens`, `totalCompletionTokens`, `totalToolCalls int` -- accumulated from `EventLLMCallEnd` events.
   - `hasOutput bool` -- whether any text delta was written to stdout.
6. Consume the stream in a loop, dispatching on `event.Type`:

| Event Type | Action |
|---|---|
| `EventTextDelta` | Write `data.Delta` to `stdout`; set `hasOutput = true` |
| `EventToolCallStart` | Write `"[tool] <name>(<summary>)"` to `stderr`, indented by nesting depth + 1; increment `totalToolCalls` and `toolCallCount` |
| `EventToolResult` | Write compact result summary to `stderr`, indented (same logic as TUI's `renderToolCallResult` but plain text) |
| `EventPhaseStart` | Write `"[phase] <Phase>"` to `stderr` |
| `EventPhaseEnd` | If `data.Summary != ""`, write summary to `stderr` indented under phase. Write `"[phase] <Phase> complete (<stats>)"` to `stderr` |
| `EventSubAgentStart` | Increment `nestingDepth`; reset `toolCallCount`. Write `"[agent] <name> (<step>)"` to `stderr`, indented by nesting depth |
| `EventSubAgentEnd` | Write `"[agent] <name> complete (<stats>)"` to `stderr`, indented by nesting depth. Decrement `nestingDepth` |
| `EventError` | Write `"[error] <message>"` to `stderr` |
| `EventAgentEnd` | No-op (stream will EOF next) |
| `EventTokenBudgetExhausted` | Write `"[warning] token budget exhausted"` to `stderr` |
| `EventLLMCallEnd` | Accumulate `data.PromptTokens` and `data.CompletionTokens` into totals (suppressed from display) |
| Others (`LLMCallStart`, `LLMCallError`, `IterationStart`, `AgentStart`, `ToolCallEnd`) | Suppressed |

7. On `io.EOF`, write a trailing newline to stdout (if `hasOutput` is true). Emit a task-complete summary line to stderr: `[done] (<totalToolCalls> tool uses . <duration> . <tokens in> . <tokens out>)` using `buildStatsLine`. Return nil.
8. On error, return the error (caller in `main.go` prints it and exits 1).

**Output format:**

- **stdout**: Raw agent response text only. No ANSI codes, no prefixes, no markdown rendering. Just the plain text deltas concatenated. This makes it pipe-friendly.
- **stderr**: Diagnostic lines prefixed with bracketed tags (`[phase]`, `[tool]`, `[agent]`, `[error]`, `[done]`). Plain text, no ANSI escape codes. One line per event. Sub-agent content is indented by `nestingDepth * 2` spaces for visual hierarchy.

**Indentation example** (stderr):
```
[phase] Explore
[phase] Explore complete (2 tool uses . 5s . 3.2k in . 1.1k out)
[phase] Classify
[phase] Classify complete (0s . 1.5k in . 0.2k out)
[phase] Dispatch
  [agent] coder (step-1)
    [tool] read(main.go)
    [tool] edit(main.go)
      └ Added 3 lines
  [agent] coder complete (2 tool uses . 8s . 5.3k in . 2.1k out)
[phase] Dispatch complete (2 tool uses . 8s . 5.3k in . 2.1k out)
[done] (4 tool uses . 14s . 10.0k in . 3.4k out)
```

The existing render helpers that compute stats strings (`formatDuration`, `formatCompactTokens`, `buildStatsLine`, `extractToolSummary`, `truncate`) are reused directly since they are in the same `cli` package and produce plain text. The lipgloss styling is applied only by TUI-specific callers, not by these helpers.

### 4. Plain-Text Formatting

No new exported functions are needed in `render.go`. The helpers `buildStatsLine`, `formatDuration`, `formatCompactTokens`, `extractToolSummary`, and `truncate` are all unexported functions in the `cli` package that return plain strings. `RunPrompt` in `prompt.go` (same package) calls them directly.

For tool result summaries, `RunPrompt` uses the same switch logic as `renderToolCallResult` (suppress read results, show change counts for write/edit, show truncated bash output) but without lipgloss styling -- just `fmt.Fprintf(stderr, ...)`.

## Data Models / Schemas

No new data models. The feature uses existing types:

- `schema.RunRequest` -- with `Messages: []schema.Message{schema.NewUserMessage(prompt)}` and a transient `SessionID`.
- `schema.RunStream` -- consumed via `Recv()` loop; **must be closed** via `defer stream.Close()`.
- `schema.Event` and its `EventData` subtypes -- dispatched by `event.Type`.
- `configs.Config` -- unchanged; loaded and validated as normal.

## API Contracts

No new APIs. This is a CLI-only feature. The stdin/stdout/stderr contract is:

| Channel | Content |
|---|---|
| **stdout** | Agent response text (plain, no ANSI, no prefixes) |
| **stderr** | Diagnostic lines: `[phase]`, `[tool]`, `[agent]`, `[error]`, `[done]`, `[warning]` prefixed |
| **exit code 0** | Success |
| **exit code 1** | Error (empty prompt, missing config, runtime failure) |

## Implementation Plan

### Task 1: Add `-p` flag, validation, and branching in `main.go`

**File**: `vv/main.go`

1. Add `promptFlag := flag.String("p", "", "run a single prompt non-interactively and exit")` alongside existing flags.
2. Update `flag.Usage` to document the `-p` flag and include usage examples.
3. Insert the NeedsSetup guard *before* the existing interactive setup block:
   - If `*promptFlag != ""` and `configs.NeedsSetup(cfg)`, print error to stderr and exit 1.
4. After applying flag overrides (`cfg.Mode`, `cfg.Server.Addr`), add the prompt-mode branch:
   - Validate non-empty trimmed prompt.
   - Check `cfg.Mode == "http"` incompatibility (this catches mode from `-mode` flag, config file, and `VV_MODE` env var).
   - Call `setup.Init(cfg, &setup.Options{})` with nil `WrapToolRegistry`.
   - Create signal context with `signal.NotifyContext`.
   - Call `cli.RunPrompt(ctx, initResult.SetupResult.Dispatcher, *promptFlag, os.Stdout, os.Stderr)`.
   - Exit with appropriate code.

**Estimated size**: ~35 lines changed in `main.go`.

### Task 2: Implement `RunPrompt` in `vv/cli/prompt.go`

**File**: `vv/cli/prompt.go` (new file)

Implement the `RunPrompt` function as described above. Key aspects:

- Session ID generation (reuse `crypto/rand` hex pattern from `cli.go`).
- Build `schema.RunRequest` with single user message and session ID.
- `defer stream.Close()` immediately after obtaining the stream.
- Stream consumption loop with `switch event.Type`.
- Nesting depth tracking for indented diagnostic output.
- Token stats accumulation from `EventLLMCallEnd` events.
- Task-complete summary line on stream EOF.
- Plain-text stderr output using `fmt.Fprintf`.
- Reuse `extractToolSummary`, `truncate`, `formatDuration`, `formatCompactTokens`, `buildStatsLine` from `render.go` (same package, no export needed).
- Track `hasOutput` bool to conditionally emit trailing newline on stdout.

**Estimated size**: ~150 lines.

### Task 3: Unit tests for `RunPrompt`

**File**: `vv/cli/prompt_test.go` (new file)

Test scenarios using a mock `agent.StreamAgent` that returns a `schema.RunStream` built with `schema.NewRunStream`:

1. **Basic text output**: Verify text deltas are written to stdout, nothing on stderr except `[done]`.
2. **Tool events to stderr**: Verify tool call events appear on stderr with proper indentation.
3. **Phase events to stderr**: Verify phase start/end events appear on stderr, including phase summary.
4. **Sub-agent nesting**: Verify sub-agent events produce indented output on stderr.
5. **Error propagation**: Verify stream errors are returned.
6. **Empty stream**: Verify clean exit on immediate EOF (no trailing newline on stdout).
7. **Context cancellation**: Verify graceful handling of cancelled context.
8. **No ANSI codes in stdout**: Verify stdout output contains no escape sequences.
9. **Task-complete summary**: Verify `[done]` line appears on stderr with accumulated stats.

The `stdout` and `stderr` `io.Writer` parameters make testing straightforward with `bytes.Buffer`.

**Estimated size**: ~180 lines.

### Task 4: Update flag usage text

**File**: `vv/main.go`

Update the `flag.Usage` function to include `-p` documentation and usage examples:

```
Usage: vv [options]

Options:
  -p string     run a single prompt non-interactively and exit
  -config string ...
  ...

Examples:
  vv                                        # interactive TUI mode
  vv -p "explain the main.go file"          # single prompt, exit after response
  vv -p "fix the bug in auth.go" 2>/dev/null  # suppress diagnostics
```

**Estimated size**: ~8 lines.

### Task Order

1. Task 2 (implement `RunPrompt`) -- core feature, no dependencies on main.go changes.
2. Task 3 (unit tests) -- validates Task 2.
3. Task 1 + Task 4 (wire into `main.go` + usage text) -- integrates Task 2 into the application.

## Integration Test Plan

### Test 1: Basic prompt execution (manual / integration)

```bash
# Requires VV_LLM_API_KEY set
vv -p "What is 2+2? Reply with just the number."
# Expected: "4" (or similar) on stdout, phase/dispatch info on stderr
# Expected: exit code 0
```

### Test 2: Empty prompt rejection

```bash
vv -p ""
# Expected: error message on stderr
# Expected: exit code 1
```

### Test 3: Missing config rejection

```bash
VV_LLM_API_KEY="" vv -p "hello" -config /nonexistent/path
# Expected: error about missing config on stderr
# Expected: exit code 1
```

### Test 4: HTTP mode incompatibility (via flag)

```bash
vv -p "hello" -mode http
# Expected: error about incompatible flags on stderr
# Expected: exit code 1
```

### Test 5: HTTP mode incompatibility (via env)

```bash
VV_MODE=http vv -p "hello"
# Expected: error about incompatible mode on stderr
# Expected: exit code 1
```

### Test 6: Pipe-friendly output

```bash
vv -p "Say exactly: hello world" 2>/dev/null | grep -c "hello world"
# Expected: "1" (stdout contains the text, no ANSI artifacts)
```

### Test 7: Stderr diagnostics

```bash
vv -p "list files in the current directory" 2>&1 1>/dev/null | grep "\[phase\]"
# Expected: at least one line matching [phase]
```

### Test 8: Task-complete summary

```bash
vv -p "Say hello" 2>&1 1>/dev/null | grep "\[done\]"
# Expected: one line matching [done] with stats
```

### Test 9: SIGINT handling

```bash
vv -p "write a very long essay about everything" &
PID=$!
sleep 2
kill -INT $PID
wait $PID
# Expected: exit code non-zero (130 for SIGINT), clean termination
```

### Note on automated tests

The mock-based tests (Task 3) live in `vv/cli/prompt_test.go` as unit tests -- they do not require LLM API keys and run in CI via `make test`. No separate `vv/integrations/` test is needed for the mock-based scenarios.

If a real LLM integration test is desired in the future, it would go in `vv/integrations/prompt_test.go` and follow the existing convention of requiring API keys via environment variables.
