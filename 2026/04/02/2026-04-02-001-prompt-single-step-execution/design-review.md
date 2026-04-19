# Design Review: `-p <prompt>` Single-Step Execution Mode

## Summary

The proposed design is well-structured and aligns closely with the existing codebase patterns. The core approach -- a new `RunPrompt` function in `vv/cli/prompt.go` consuming the same `StreamAgent` interface -- is correct and minimal. Below are specific suggestions for improvement, ranging from correctness fixes to simplification opportunities.

## Suggestion 1: Move the prompt-mode branch BEFORE config interactive setup, not after

**Problem**: The design says to insert the prompt branch "after config loading and before `switch cfg.Mode`," but the current `main.go` runs the interactive setup prompt (`configs.Prompt`) *before* the mode switch. If `-p` is provided and `NeedsSetup` returns true, the current code would already have entered the interactive prompt before the `-p` check runs.

**Fix**: The `-p` check for `NeedsSetup` must happen *before* the existing interactive setup block in `main.go` (lines 55-65), not after it. The design's step 2c ("If `configs.NeedsSetup(cfg)`, print error to stderr and exit 1") is correct in intent but the placement description is ambiguous. The implementation must guard the interactive setup block with `*promptFlag == ""`.

**Revised flow in main.go**:
```
1. Parse flags (including -p)
2. Load config
3. If NeedsSetup:
   a. If -p is set: print error, exit 1  (NEW)
   b. Else: run interactive Prompt as before
4. Apply flag overrides (-mode, -addr)
5. If -p is set:
   a. Validate non-empty after trim
   b. Check -mode http incompatibility
   c. Init with nil WrapToolRegistry
   d. RunPrompt
   e. Exit
6. Else: existing cli/http switch
```

## Suggestion 2: The `-mode http` check should use `cfg.Mode` not just `*modeFlag`

**Problem**: The design checks `*modeFlag == "http"` for incompatibility. But `cfg.Mode` can also be set to `"http"` via the config file or the `VV_MODE` environment variable. If the user has `mode: http` in their config and runs `vv -p "hello"`, the check would pass incorrectly.

**Fix**: Perform the incompatibility check against `cfg.Mode` *after* applying the flag override (`cfg.Mode = *modeFlag`), rather than checking `*modeFlag` alone. Alternatively, apply the mode flag override early and check `cfg.Mode == "http"`.

## Suggestion 3: The stream must be closed after consumption

**Problem**: The design's `RunPrompt` description does not mention closing the `*schema.RunStream`. Looking at `stream.go`, `RunStream` has a `Close()` method that cancels the producer context. The TUI code in `cli.go` closes it in the goroutine's defer. `RunPrompt` must do the same.

**Fix**: Add `defer stream.Close()` immediately after the `orchestrator.RunStream()` call in `RunPrompt`.

## Suggestion 4: Track token stats in RunPrompt for the task-complete line

**Problem**: The design mentions reusing `buildStatsLine` for phase/sub-agent events but does not mention emitting a final task-completion summary. The TUI renders a `renderTaskComplete` line with total duration and token counts. Without this, the prompt mode gives no final summary, and the user sees no total execution time.

**Fix**: Track `totalPromptTokens`, `totalCompletionTokens`, `totalToolCalls`, and `taskStart` time inside `RunPrompt` (mirroring the TUI model's fields). Accumulate from `EventLLMCallEnd` events. After the stream loop, emit a plain-text task-complete line to stderr: `[done] (3 tool uses . 26s . 5.3k in . 10.5k out)`.

## Suggestion 5: Handle `EventToolCallEnd` for tool duration reporting

**Problem**: The design's event table suppresses `EventToolCallEnd`, but this event carries `Duration` which could be useful in the diagnostic output. The TUI also suppresses it, so this is consistent, but worth noting as a conscious choice.

**Decision**: Keep suppressed for consistency with TUI. No change needed.

## Suggestion 6: Use `agent.RunStreamText` convenience function

**Problem**: The design manually constructs a `schema.RunRequest` with `Messages: []schema.Message{schema.NewUserMessage(prompt)}` and a session ID. The `agent` package provides `RunStreamText(ctx, agent, input)` which does the same thing (minus the session ID).

**Fix**: Since the session ID is needed (for memory scoping), keep the manual `RunRequest` construction. However, document this as the reason for not using `RunStreamText`. No code change, just a design clarification.

## Suggestion 7: Simplify the `RunPrompt` function signature

**Problem**: The design initially proposes `RunPrompt(ctx, orchestrator, prompt)` then revises to `RunPrompt(ctx, orchestrator, prompt, stdout, stderr)` for testability. Five parameters is reasonable, but consider using an options struct.

**Fix**: Keep the 5-parameter signature. It is simple, explicit, and Go-idiomatic for this case. An options struct would be over-engineering for two writers. The revised signature is good.

## Suggestion 8: Handle the `EventPhaseSummary` in phase end

**Problem**: The design's event table for `EventPhaseEnd` mentions writing phase info to stderr but does not mention the `PhaseEndData.Summary` field. The TUI renders this summary (e.g., plan overview) before the phase-end line. The prompt mode should do the same for parity.

**Fix**: When handling `EventPhaseEnd`, if `data.Summary != ""`, write the summary to stderr before the phase-complete line (prefixed with something like `[plan]` or indented under the phase line).

## Suggestion 9: Signal handling context placement

**Problem**: The design places `setup.Init` before context creation. Looking at `main.go`, the signal context is created at line 88, *after* `setup.Init`. The design's prompt-mode branch calls `setup.Init` then `RunPrompt`. The signal context must be created before `RunPrompt` is called, but the current `main.go` creates it after `setup.Init`, which is fine.

**Fix**: In the prompt-mode branch, the signal context creation should happen before `RunPrompt` but can stay after `setup.Init` (same as existing code). The design implicitly assumes this but should be explicit: create `signal.NotifyContext` before calling `RunPrompt`, and pass the resulting `ctx` to it.

## Suggestion 10: Add `--` separator to prevent flag ambiguity with prompts starting with `-`

**Problem**: If a user runs `vv -p "-explain flags"`, the Go `flag` package may misinterpret the prompt string. Standard Go `flag.Parse()` handles this correctly since `-p` takes a string argument, but it is worth noting in the usage text.

**Fix**: Add a note in the usage examples that prompts are shell-quoted strings. No code change needed since Go's `flag` package handles `-p "-something"` correctly.

## Suggestion 11: NeedsSetup is too narrow -- only checks API key

**Problem**: `configs.NeedsSetup` only checks `cfg.LLM.APIKey == ""`. But `aimodel.NewClient` can fall back to environment variables (`AI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) when no explicit key is set. So `NeedsSetup` returning true does not necessarily mean the config is broken -- the key might come from the environment at the `aimodel` level.

**Fix**: In the prompt-mode branch, do not check `NeedsSetup` directly. Instead, attempt `setup.Init` and let it fail naturally if the API key is truly missing. This avoids false negatives. The interactive setup guard should remain for the TUI path (where prompting makes sense), but for `-p` mode, just try to initialize and report the error.

This is the most impactful correctness fix: the current design would reject `vv -p "hello"` when `VV_LLM_API_KEY` is unset but `OPENAI_API_KEY` is set, because `NeedsSetup` would return true even though `aimodel.NewClient` would succeed.

## Suggestion 12: New file placement in `vv/cli/` package is appropriate

The `cli` package is the right home. The new `prompt.go` file accesses unexported helpers (`buildStatsLine`, `extractToolSummary`, etc.) from `render.go` in the same package. No concerns here.

## Suggestion 13: Nesting depth tracking in prompt mode

**Problem**: The TUI tracks `nestingDepth` for sub-agent indentation. The design does not mention nesting depth tracking in `RunPrompt`, but the diagnostic output would benefit from indentation to show sub-agent hierarchy.

**Fix**: Track a `nestingDepth int` in `RunPrompt` (increment on `EventSubAgentStart`, decrement on `EventSubAgentEnd`). Use it to indent stderr diagnostic lines. This makes plan-mode output with multiple sub-agents much more readable:
```
[phase] Dispatch
  [agent] coder (step-1)
    [tool] read(main.go)
    [tool] edit(main.go)
  [agent] coder complete (2 tool uses . 8s)
[phase] Dispatch complete
```

## Suggestion 14: Integration test should not be in `vv/integrations/`

**Problem**: The design proposes `vv/integrations/prompt_test.go` for a mock-based test that does not require API keys. Tests in `integrations/` conventionally require LLM API keys (as stated in CLAUDE.md).

**Fix**: The mock-based unit test belongs in `vv/cli/prompt_test.go` (same package, as the design already proposes in Task 3). The `integrations/` test should only be added if a real LLM integration test is desired. Remove the `vv/integrations/prompt_test.go` proposal from the design, or explicitly mark it as requiring API keys.

## Priority Ranking

| Priority | Suggestion | Impact |
|----------|-----------|--------|
| High | #1 (branch placement before interactive setup) | Correctness: prevents accidental interactive prompt in -p mode |
| High | #3 (close the stream) | Correctness: resource leak without Close() |
| High | #11 (NeedsSetup too narrow) | Correctness: false rejection when API key comes from aimodel env vars |
| Medium | #2 (check cfg.Mode not just modeFlag) | Correctness: config/env-based http mode not caught |
| Medium | #4 (task-complete stats) | Usability: no execution summary without this |
| Medium | #8 (phase summary in phase end) | Completeness: plan overview missing from output |
| Medium | #13 (nesting depth tracking) | Usability: flat diagnostic output hard to read for plan-mode |
| Low | #9 (signal context placement) | Clarity: make explicit in design |
| Low | #14 (integration test location) | Convention compliance |
| Low | #5, #6, #7, #10, #12 | Minor clarifications, no code changes |
