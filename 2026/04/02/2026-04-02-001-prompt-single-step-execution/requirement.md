# Requirement: Support `-p <prompt>` Single-Step Execution Mode

## Background and Objectives

Currently, the `vv` CLI application operates in two modes: an interactive TUI mode (default) and an HTTP server mode. Both are long-running processes. There is no way to invoke `vv` for a one-shot task from the command line -- for example, from a shell script, a CI/CD pipeline, or a quick terminal command -- where the user wants to pass a prompt, get the result, and have the process exit automatically.

The objective is to add a `-p <prompt>` flag that allows users to run `vv` in a non-interactive, single-step execution mode. When `-p` is provided, `vv` should:

1. Execute the given prompt through the existing orchestrator/dispatcher pipeline.
2. Stream the output to stdout (not via the Bubble Tea TUI).
3. Exit the process automatically once execution completes.

This enables scripting, automation, and quick one-off agent invocations without entering the interactive terminal.

## User Stories

### US-1: Single-Step Prompt Execution

**As a** developer,
**I want to** run `vv -p "explain the main.go file"` from my terminal,
**So that** I get the agent's response printed to stdout and the process exits without entering interactive mode.

**Acceptance Criteria:**
- A `-p` flag (short for "prompt") is available on the `vv` command.
- When `-p <prompt>` is provided, vv does NOT enter the interactive TUI.
- The prompt is dispatched through the same orchestrator pipeline used by the interactive CLI (explore, classify, dispatch).
- Agent text output is streamed to stdout in plain text (no TUI rendering, no Bubble Tea).
- Tool call activity (tool name, arguments, results) is printed to stderr or stdout in a readable format so the user can follow progress.
- The process exits with code 0 on success, non-zero on error.
- If the prompt string is empty (`-p ""`), vv prints a usage error and exits with non-zero code.

### US-2: Compatibility with Existing Flags

**As a** developer,
**I want to** combine `-p` with other existing flags like `-config`,
**So that** I can use custom configurations for one-shot execution.

**Acceptance Criteria:**
- `-p` works together with `-config <path>` for custom config file.
- `-p` is incompatible with `-mode http`; if both are specified, vv prints an error and exits.
- Environment variable overrides (`VV_LLM_API_KEY`, `VV_LLM_MODEL`, etc.) apply normally when `-p` is used.
- The interactive setup prompt is skipped when `-p` is provided and config is missing required fields; instead, vv prints an error and exits.

### US-3: Pipe-Friendly Output

**As a** developer or CI/CD pipeline,
**I want to** pipe the output of `vv -p "..."` to other tools,
**So that** I can integrate vv into automated workflows.

**Acceptance Criteria:**
- The final agent response text is written to stdout.
- Diagnostic/progress information (tool calls, phase transitions, errors) is written to stderr.
- No ANSI escape codes or TUI artifacts are present in stdout when output is piped (non-TTY detection or explicit plain mode).
- The output is suitable for piping to `jq`, `grep`, `pbcopy`, or file redirection.

## Scope

### In Scope

- New `-p <prompt>` CLI flag in `main.go`.
- A new non-interactive execution path that bypasses the Bubble Tea TUI.
- Streaming agent output to stdout/stderr in plain text.
- Proper exit code handling (0 for success, 1 for errors).
- Skipping the interactive config setup prompt when `-p` is used.
- Error handling for empty prompt, missing config, and execution failures.

### Out of Scope

- Reading prompt from stdin (e.g., `echo "prompt" | vv`). This can be a future enhancement.
- Batch execution of multiple prompts in sequence.
- JSON-structured output format (future enhancement).
- Changes to the HTTP mode or HTTP API.
- Changes to the interactive TUI mode behavior.
- New agent types or tool registrations.
- Memory persistence across `-p` invocations (each invocation is stateless).

## Involved System Roles

| Role | Involvement |
|------|-------------|
| Developer (CLI user) | Primary user; invokes `vv -p <prompt>` from terminal or scripts |
| External Systems / CI | Invokes `vv -p <prompt>` programmatically for automated tasks |

## Involved Models and State Changes

No new domain models are introduced. The existing models are used as-is:

- **Configuration** -- read and validated as normal, but interactive setup is skipped when `-p` is present.
- **CLI Session** -- a transient session is created for the single execution; it is not persisted.
- **Orchestrator / Dispatcher** -- used identically to the interactive mode (explore -> classify -> dispatch).

## Involved Business Processes

### Modified Process: Application Startup

The startup process gains a new branch:

1. Parse flags (including new `-p` flag).
2. Load and validate configuration.
3. **New decision point**: If `-p` is provided:
   a. If config needs setup (missing API key), print error and exit (do NOT prompt interactively).
   b. If `-mode http` is also set, print error and exit.
   c. Initialize LLM client, memory, and agents (same as current flow).
   d. Enter **single-step execution** (new path, bypasses TUI and HTTP server).
4. Otherwise, proceed with existing CLI or HTTP mode as before.

### New Process: Single-Step Execution

1. Build a `schema.RunRequest` containing the user prompt as a single user message.
2. Call `orchestrator.RunStream(ctx, req)` to get a streaming response.
3. Consume the stream:
   - `EventTextDelta` -- write delta text to stdout.
   - `EventToolCallStart` / `EventToolResult` -- write summary to stderr.
   - `EventPhaseStart` / `EventPhaseEnd` -- write phase info to stderr.
   - `EventSubAgentStart` / `EventSubAgentEnd` -- write sub-agent info to stderr.
   - `EventError` -- write error to stderr.
   - `EventAgentEnd` -- finalize.
4. On stream EOF, exit with code 0.
5. On error, print error to stderr and exit with code 1.

## Involved Applications and Pages

This change affects the **CLI application** only. No new pages or screens are added; the TUI is simply bypassed when `-p` is provided. The stdout/stderr output is plain text, not a TUI view.

## Assumptions

- Tool confirmation prompts are skipped in `-p` mode (all tools are auto-approved), since there is no interactive TUI to display confirmation dialogs. This matches the behavior expected for scripted/automated usage.
- The `-p` flag takes a single string argument. Multi-word prompts are handled by shell quoting (e.g., `vv -p "fix the bug in main.go"`).
- Session memory is created transiently for the single execution but is not loaded from or persisted to disk, keeping each `-p` invocation stateless.
- The graceful shutdown via SIGINT/SIGTERM remains functional in `-p` mode, cancelling the in-flight request and exiting cleanly.
