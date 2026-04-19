# Design Review: Project Instructions File (VV.md)

## Overall Assessment

The design is well-structured, minimal, and aligns closely with existing codebase patterns. The single-string-flows-through approach is appropriately simple. The following observations identify gaps, over-engineering, and alignment issues found by validating the design against the actual code.

## Issues

### Issue 1: Intent Recognition System Prompt Is Missing

**Severity**: High

The design covers all six agent factories and the dynamic agent path in `dag.go`, but it completely ignores the **intent recognition system prompt** (`intentSystemPrompt`). This is a separate system prompt passed as a raw string to a direct LLM call in `dispatches/intent.go` (lines 102-114). It is not an agent factory -- it is a raw `ChatCompletion` call with a system message.

If a user writes `VV.md` saying "This is a Python project, always route coding tasks to the coder agent", the intent recognition LLM call would never see that instruction. The intent recognition phase makes routing decisions that should be informed by project context.

**Recommendation**: Apply project instructions to the `intentSystemPrompt` in the Dispatcher, either by appending to it in `recognizeIntentDirect()` / `recognizeIntentViaPlannerStream()`, or by pre-composing it when the Dispatcher is constructed (cleaner). Since the design already proposes adding a `projectInstructions` field to the Dispatcher, the simplest approach is to prepend/append project instructions to the intent system prompt in the `recognizeIntentDirect` method, the same way `workingDir` is substituted.

### Issue 2: `LoadProjectInstructions` Placement Is Slightly Off

**Severity**: Low

The design places `LoadProjectInstructions()` in `configs/config.go`, justified as a "configuration-loading concern." However, looking at `configs/config.go`, it is purely about YAML config, env vars, and defaults -- no file-system scanning for project-level files. The `slog` import would be new to this file.

This is not a blocking issue, but placing the function in `setup/setup.go` (which already imports `slog` and `os`, and is where the working directory is resolved) would be more consistent with the existing code organization. Alternatively, a small standalone file `configs/project_instructions.go` avoids mixing concerns.

**Recommendation**: Keep it in `configs/` but in a dedicated file `configs/project_instructions.go` to maintain separation of concerns within the package. This is a stylistic preference, not a functional issue.

### Issue 3: Planner Factory Does Not Actually Use `opts` for System Prompt

**Severity**: Medium

The design says the planner factory will use `AppendProjectInstructions(PlannerSystemPrompt, opts.ProjectInstructions)`. But looking at the actual planner factory code (`agents/planner.go:98-117`), the system prompt is hardcoded to the fallback `PlannerSystemPrompt` constant. It does not use `opts` for the system prompt at all -- the comment even says "The system prompt should be set by the caller via opts or computed from the registries."

In practice, `setup.New()` does not pass any system prompt through `FactoryOptions` to the planner. The actual intent system prompt is built separately via `BuildPlannerSystemPrompt(reg)` and passed to the Dispatcher via `WithIntentSystemPrompt()`. The planner factory's prompt is essentially dead code since the planner agent is only used through the dispatcher's intent recognition, which overrides the prompt.

**Recommendation**: For the planner, injecting project instructions into the factory is harmless but effectively a no-op since the planner's system prompt is overridden by the dispatcher. The design should acknowledge this and focus on injecting project instructions at the dispatcher level (see Issue 1). Still include it in the factory for correctness (in case the planner is used standalone), but note it is not the primary injection path.

### Issue 4: Dynamic Agent Spec May Have Custom System Prompts

**Severity**: Low

The design correctly handles dynamic agents in `dag.go`. However, when `spec.SystemPrompt` is non-empty (the planner explicitly provides a custom system prompt for a specialized sub-task), the design would append project instructions to this custom prompt. This is correct behavior -- the user's project instructions should apply universally. Just confirming this is intentional and appropriate.

### Issue 5: `PersistentMemoryPrompt` Integration Order

**Severity**: Medium

The design proposes appending project instructions to the base prompt string *before* passing it to `PersistentMemoryPrompt`. This means the prompt assembly order is: base prompt -> project instructions -> persistent memory entries (rendered dynamically). This is correct and well-considered.

However, the design should explicitly note that `PersistentMemoryPrompt.Render()` returns the `basePrompt` field directly (it doesn't re-process it). So the project instructions become permanently baked into the `basePrompt` string of the `PersistentMemoryPrompt` instance. This is fine because project instructions don't change during a session (no hot-reload), but it should be documented as an assumption.

### Issue 6: Integration Test 1 Is Brittle

**Severity**: Medium

Integration Test 1 proposes verifying that an LLM response "follows the project instruction (heuristic check)" -- e.g., checking if a response is in haiku format. This is inherently non-deterministic and will produce flaky CI results. LLMs do not reliably follow formatting instructions 100% of the time.

**Recommendation**: Replace with a deterministic unit test approach. Instead of testing end-to-end through an LLM, test that the system prompt passed to the LLM contains the project instructions content. This can be done by:
1. Creating a mock `ChatCompleter` that captures the request.
2. Running a chat agent created with project instructions.
3. Asserting the system message in the captured request contains the VV.md content.

This is a unit test, not an integration test, and provides deterministic verification of the data flow.

### Issue 7: Integration Test 4 Is Impractical

**Severity**: Low

Integration Test 4 proposes triggering a plan-mode request to verify dynamic agents receive project instructions. This requires a live LLM to generate a plan with dynamic specs, which is both expensive and non-deterministic. The dynamic agent path can be verified with a unit test of `buildDynamicAgent()` by checking the constructed taskagent options.

**Recommendation**: Replace with a unit test that calls `buildDynamicAgent()` directly and inspects the resulting agent's configuration.

### Issue 8: Missing Edge Case -- Empty VV.md File

**Severity**: Low

The design does not address what happens when VV.md exists but is empty (0 bytes). `os.ReadFile` will return an empty byte slice, which converts to `""`. The `AppendProjectInstructions` function returns the base prompt unchanged for empty strings. This is correct behavior, but should be documented as an explicit test case.

### Issue 9: The `compat.go` Discussion Is Unnecessary

**Severity**: Low (editorial)

Task 6 discusses `compat.go` at length only to conclude "no changes needed." This adds noise to the design. A single sentence noting the deprecated path is not covered would suffice.

## Summary of Recommendations

1. **Add project instructions to the intent recognition system prompt** in the Dispatcher (high priority).
2. **Move `LoadProjectInstructions` to a dedicated file** `configs/project_instructions.go` for cleaner separation.
3. **Acknowledge the planner factory prompt is not the primary injection path** -- the dispatcher's intent system prompt matters more.
4. **Replace flaky integration tests with deterministic unit tests** that capture and inspect the system prompt.
5. **Add a test case for empty VV.md**.
6. **Remove the Task 6 compat.go discussion** or reduce to a single line.
