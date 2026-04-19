# Design: `-p <prompt>` Single-Step Execution Mode

## Architecture Overview

The feature adds a third execution branch to `vv/main.go` alongside the existing `cli` and `http` modes. When `-p <prompt>` is provided, the application bypasses both the Bubble Tea TUI and the HTTP server, instead running a simple stream-consume loop that writes agent text to stdout and diagnostic events to stderr, then exits.

The new code lives in a single new file `vv/cli/prompt.go` containing a `RunPrompt` function. This function reuses the same `agent.StreamAgent` (the `Dispatcher`) and `schema.RunRequest`/`schema.RunStream` types used by the interactive TUI, ensuring identical orchestration behavior.

```
main.go
  |
  +-- flag.Parse() adds "-p" flag
  |
  +-- if prompt != "" :
  |     - validate (non-empty, not http mode, config present)
  |     - setup.Init() with NO tool confirmation wrapper (auto-approve all)
  |     - cli.RunPrompt(ctx, orchestrator, prompt)  -->  stdout/stderr
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

After flag parsing and config loading, insert a new branch before the existing `switch cfg.Mode`:

1. If `*promptFlag != ""`:
   a. Validate prompt is non-empty after trimming (reject `-p ""`).
   b. If `*modeFlag == "http"`, print error to stderr and exit 1.
   c. If `configs.NeedsSetup(cfg)`, print error to stderr and exit 1 (skip interactive prompt).
   d. Call `setup.Init(cfg, &setup.Options{})` -- no `WrapToolRegistry` (all tools auto-approved).
   e. Call `cli.RunPrompt(ctx, dispatcher, prompt)`.
   f. Exit 0 on success, 1 on error.

### 3. `RunPrompt` Function (`vv/cli/prompt.go`)

New file in the `cli` package. This is the core of the feature -- a stateless, non-interactive stream consumer.

```go
package cli

func RunPrompt(ctx context.Context, orchestrator agent.StreamAgent, prompt string) error
```

**Behavior:**

1. Generate a transient session ID (same `crypto/rand` hex pattern used by the TUI).
2. Build a `schema.RunRequest` with a single user message and the session ID.
3. Call `orchestrator.RunStream(ctx, req)`.
4. Consume the stream in a loop, dispatching on `event.Type`:

| Event Type | Action |
|---|---|
| `EventTextDelta` | Write `data.Delta` to `os.Stdout` |
| `EventToolCallStart` | Write `"[tool] <name>(<summary>)"` to `os.Stderr` |
| `EventToolResult` | Write compact result summary to `os.Stderr` |
| `EventPhaseStart` | Write `"[phase] <Phase>"` to `os.Stderr` |
| `EventPhaseEnd` | Write `"[phase] <Phase> complete (<stats>)"` to `os.Stderr` |
| `EventSubAgentStart` | Write `"[agent] <name> (<step>)"` to `os.Stderr` |
| `EventSubAgentEnd` | Write `"[agent] <name> complete (<stats>)"` to `os.Stderr` |
| `EventError` | Write `"error: <message>"` to `os.Stderr` |
| `EventAgentEnd` | No-op (stream will EOF next) |
| `EventTokenBudgetExhausted` | Write warning to `os.Stderr` |
| Others (`LLMCall*`, `Iteration*`, `AgentStart`) | Suppressed |

5. On `io.EOF`, write a trailing newline to stdout (if any text was emitted) and return nil.
6. On error, return the error (caller in `main.go` prints it and exits 1).

**Output format:**

- **stdout**: Raw agent response text only. No ANSI codes, no prefixes, no markdown rendering. Just the plain text deltas concatenated. This makes it pipe-friendly.
- **stderr**: Diagnostic lines prefixed with bracketed tags (`[phase]`, `[tool]`, `[agent]`, `[error]`). Plain text, no ANSI escape codes. One line per event. Uses the same `extractToolSummary` helper from `render.go` for compact tool argument display.

The `extractToolSummary`, `truncate`, `formatDuration`, `formatCompactTokens`, and `buildStatsLine` helpers in `cli/render.go` are already non-TUI pure functions that return strings (the lipgloss styling is applied by callers). For the prompt mode, we call the underlying formatting logic but strip ANSI or use plain-text equivalents.

To keep it simple, `RunPrompt` uses `fmt.Fprintf(os.Stderr, ...)` for diagnostic output with its own plain-text formatting -- no lipgloss dependency. The existing render helpers that compute stats strings (`formatDuration`, `formatCompactTokens`) are reused directly since they produce plain text.

### 4. Plain-Text Stats Helper (`vv/cli/render.go`)

Add one small exported function to `render.go`:

```go
// BuildPlainStatsLine builds a stats string without ANSI styling.
func BuildPlainStatsLine(s execStats) string {
    // Same logic as buildStatsLine but without lipgloss/dimStyle wrapping.
    // Returns e.g. "(3 tool uses . 26s . 5.3k in . 10.5k out)"
}
```

Actually, examining the code more carefully, `buildStatsLine` already returns a plain string -- the `statsStyle.Render()` call happens in the callers (`renderPhaseTransition`, `renderSubAgentEnd`), not inside `buildStatsLine` itself. So `buildStatsLine`, `formatDuration`, and `formatCompactTokens` can be called directly from `RunPrompt` with no changes needed. We just need to export `buildStatsLine` (or duplicate the logic inline -- the latter is simpler and avoids changing the existing API).

**Decision**: Keep `buildStatsLine` unexported. `RunPrompt` in the same `cli` package can call it directly since it is in the same package.

## Data Models / Schemas

No new data models. The feature uses existing types:

- `schema.RunRequest` -- with `Messages: []schema.Message{schema.NewUserMessage(prompt)}` and a transient `SessionID`.
- `schema.RunStream` -- consumed via `Recv()` loop.
- `schema.Event` and its `EventData` subtypes -- dispatched by `event.Type`.
- `configs.Config` -- unchanged; loaded and validated as normal.

## API Contracts

No new APIs. This is a CLI-only feature. The stdin/stdout/stderr contract is:

| Channel | Content |
|---|---|
| **stdout** | Agent response text (plain, no ANSI, no prefixes) |
| **stderr** | Diagnostic lines: `[phase]`, `[tool]`, `[agent]`, `[error]` prefixed |
| **exit code 0** | Success |
| **exit code 1** | Error (empty prompt, missing config, runtime failure) |

## Implementation Plan

### Task 1: Add `-p` flag and validation logic in `main.go`

**File**: `vv/main.go`

1. Add `promptFlag := flag.String("p", "", "run a single prompt non-interactively and exit")` alongside existing flags.
2. Update `flag.Usage` to document the `-p` flag.
3. After config loading and before `switch cfg.Mode`, add the prompt-mode branch:
   - Validate non-empty trimmed prompt.
   - Check incompatibility with `-mode http`.
   - Skip interactive setup: if `configs.NeedsSetup(cfg)`, print error and exit 1.
   - Call `setup.Init` with `nil` `WrapToolRegistry` (no confirmation wrapping).
   - Call `cli.RunPrompt(ctx, initResult.SetupResult.Dispatcher, *promptFlag)`.
   - Exit with appropriate code.

**Estimated size**: ~30 lines changed in `main.go`.

### Task 2: Implement `RunPrompt` in `vv/cli/prompt.go`

**File**: `vv/cli/prompt.go` (new file)

Implement the `RunPrompt` function as described above. Key aspects:

- Session ID generation (reuse pattern from `cli.go`).
- Build `schema.RunRequest` with single user message.
- Stream consumption loop with `switch event.Type`.
- Plain-text stderr output using `fmt.Fprintf`.
- Reuse `extractToolSummary`, `truncate`, `formatDuration`, `formatCompactTokens`, `buildStatsLine` from `render.go` (same package).
- Track `hasOutput` bool to conditionally emit trailing newline.

**Estimated size**: ~120 lines.

### Task 3: Unit tests for `RunPrompt`

**File**: `vv/cli/prompt_test.go` (new file)

Test scenarios using a mock `agent.StreamAgent` that returns a pre-built `schema.RunStream`:

1. **Basic text output**: Verify text deltas are written to stdout.
2. **Tool events to stderr**: Verify tool call events appear on stderr.
3. **Phase events to stderr**: Verify phase start/end events appear on stderr.
4. **Error propagation**: Verify stream errors are returned.
5. **Empty stream**: Verify clean exit on immediate EOF.
6. **Context cancellation**: Verify graceful handling of cancelled context.

To capture stdout/stderr in tests, `RunPrompt` should accept `io.Writer` parameters (or use an option struct) rather than writing to `os.Stdout`/`os.Stderr` directly. This is cleaner for testing.

**Revised signature**:

```go
func RunPrompt(ctx context.Context, orchestrator agent.StreamAgent, prompt string, stdout, stderr io.Writer) error
```

The `main.go` caller passes `os.Stdout` and `os.Stderr`.

**Estimated size**: ~150 lines.

### Task 4: Update flag usage text

**File**: `vv/main.go`

Update the `flag.Usage` function to include `-p` documentation and a usage example:

```
Usage: vv [options]

Options:
  -p string     run a single prompt non-interactively and exit
  -config string ...
  ...

Examples:
  vv -p "explain the main.go file"
  vv -p "fix the bug in auth.go" 2>/dev/null
```

**Estimated size**: ~5 lines.

### Task Order

1. Task 2 (implement `RunPrompt`) -- core feature, no dependencies.
2. Task 3 (unit tests) -- validates Task 2.
3. Task 1 (wire into `main.go`) -- integrates Task 2 into the application.
4. Task 4 (usage text) -- polish, done alongside Task 1.

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

### Test 4: HTTP mode incompatibility

```bash
vv -p "hello" -mode http
# Expected: error about incompatible flags on stderr
# Expected: exit code 1
```

### Test 5: Pipe-friendly output

```bash
vv -p "Say exactly: hello world" 2>/dev/null | grep -c "hello world"
# Expected: "1" (stdout contains the text, no ANSI artifacts)
```

### Test 6: Stderr diagnostics

```bash
vv -p "list files in the current directory" 2>&1 1>/dev/null | grep "\[phase\]"
# Expected: at least one line matching [phase]
```

### Test 7: SIGINT handling

```bash
vv -p "write a very long essay about everything" &
PID=$!
sleep 2
kill -INT $PID
wait $PID
# Expected: exit code non-zero (130 for SIGINT), clean termination
```

### Automated Integration Test (`vv/integrations/`)

Add a test file `vv/integrations/prompt_test.go` that:

1. Uses `setup.InitFromFile` or constructs a mock LLM to avoid real API calls.
2. Calls `cli.RunPrompt` with a mock `StreamAgent` that emits a known sequence of events.
3. Asserts stdout contains expected text.
4. Asserts stderr contains expected diagnostic lines.
5. Asserts no ANSI escape codes in stdout output.

This test does not require LLM API keys and can run in CI.
