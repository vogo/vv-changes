# Design Review: Dynamic Sub-Agent Support

## Summary

The proposed design is solid and well-structured. It correctly identifies the minimal change surface (confined to `vv/agents/`), leverages existing `taskagent.New()` for building dynamic agents, and maintains backward compatibility. The review below identifies specific improvements around API simplicity, alignment with existing codebase patterns, validation robustness, and testability.

## Improvement Areas

### 1. Avoid Expanding the NewOrchestratorAgent Signature

**Problem:** The proposed design adds three new parameters (`coderReg`, `readOnlyReg`, `reviewReg`) to `NewOrchestratorAgent`, which already has 10 positional parameters. This makes the constructor fragile and hard to read. Every existing test that calls `NewOrchestratorAgent` must also be updated.

**Improvement:** The `Create()` function in `agents.go` already has all three registries in scope. Instead of passing them as three separate parameters, pass a single `map[ToolAccessLevel]tool.ToolRegistry` (the pre-built map). This keeps the constructor signature growth to a single parameter and matches the internal storage format. Even better, consider making it an optional field that can be nil for backward compatibility in tests -- when nil, dynamic agent building falls back to an error.

### 2. Separate File for DynamicAgentSpec Is Unnecessary

**Problem:** The design proposes a new file `dynamic_agent.go` for the `DynamicAgentSpec` type, validation, and builder. The type is small (4 fields, ~15 lines of validation, ~20 lines of builder). Creating a separate file fragments related orchestrator logic across two files.

**Improvement:** Place `DynamicAgentSpec`, `ToolAccessLevel`, the validation method, and the builder method directly in `orchestrator.go`. This follows the existing codebase pattern where `ClassifyResult`, `Plan`, `PlanStep`, and `PlanAggregator` are all in `orchestrator.go`. Keep the new test cases in `orchestrator_test.go` as well, rather than creating a new test file. The types are tightly coupled to the orchestrator; co-locating them improves discoverability.

### 3. Validation Should Reject DynamicSpec With Empty Agent Field

**Problem:** The design says "The `Agent` field is ignored but should still be set (it serves as documentation)." This is ambiguous guidance for the planner LLM and could lead to confusing JSON where `Agent` is empty or mismatched. The design also states in ORCH-19 that dynamic spec takes precedence over `Agent`, but does not enforce that `Agent` should be a valid base type name.

**Improvement:** When `DynamicSpec` is present, require `Agent` to equal `DynamicSpec.BaseType`. This provides consistency (the JSON is self-documenting), simplifies the planner prompt (the LLM just sets `agent` to the base type), and makes validation straightforward. If they do not match, validation should return a clear error.

### 4. Remove the "review" ToolAccessLevel -- Use Existing Registry Names

**Problem:** The requirement specifies three levels: full, read-only, none. The proposed design adds a fourth level `review` that maps to the reviewer's registry (bash + read tools, no write/edit). This level is not in the requirements and adds complexity to the planner prompt (the LLM must understand four levels instead of three).

**Improvement:** Keep the three levels from the requirements: `full`, `read-only`, `none`. The reviewer's registry is a subset of full access; if a dynamic agent truly needs reviewer-style access, the planner can specify `base_type: "reviewer"` which uses the reviewer's default registries. This simplifies the `ToolAccessLevel` type and the planner prompt. The `defaultToolAccess` map for `reviewer` base type should map to `read-only` (researchers and reviewers share read-only as the safe default; the reviewer's bash access is a static agent implementation detail, not a generalizable access level).

**Revision after deeper analysis:** Looking more carefully at the existing registries, the reviewer registry includes bash (for running tests/linters) but not write/edit, which is functionally different from both full and read-only. Collapsing this distinction would change reviewer behavior. The better approach: keep three external-facing levels (`full`, `read-only`, `none`) but internally map the reviewer base type to the `reviewReg` registries. The `ToolAccessLevel` type only needs three values; the base-type-to-registry mapping is a separate internal concern handled by the builder. This means `ToolAccessLevel` is about what the planner can explicitly override, while base type defaults still select the correct registries.

### 5. MaxIterations for Dynamic Agents Should Be Configurable

**Problem:** The design hardcodes `WithMaxIterations(10)` for dynamic agents. The static agents use `cfg.Agents.MaxIterations` which is user-configurable. This discrepancy means dynamic agents could have a different iteration limit than static agents, leading to surprising behavior.

**Improvement:** The orchestrator should store `cfg.Agents.MaxIterations` (or a sensible default from the existing config) and use it when building dynamic agents. This aligns dynamic agent behavior with static agents. If needed, `DynamicAgentSpec` could include an optional `max_iterations` field in the future, but for the initial implementation, inheriting the config default is sufficient and simpler.

### 6. Planner Prompt Should Be More Concise

**Problem:** The proposed planner prompt addition is verbose (~20 lines) with a full JSON example, field descriptions, and usage guidelines. This consumes significant context window tokens on every planner invocation, even when dynamic specs are not needed.

**Improvement:** Compress the prompt addition to essential information only. The JSON example can be inline with the existing plan format example. Keep field descriptions to one line each. Remove the "When to use dynamic_spec" section entirely -- the LLM can infer this from the field descriptions. Target ~8-10 lines of addition to the existing prompt, not ~20.

### 7. Handle the Case Where toolRegistries Is Nil in buildDynamicAgent

**Problem:** The builder accesses `o.toolRegistries[toolAccess]` directly. If the orchestrator was constructed without registries (e.g., in tests or edge cases), this will panic with a nil map access.

**Improvement:** Add a nil check at the top of `buildDynamicAgent`. If `o.toolRegistries` is nil, return an error (change return type to `(agent.Agent, error)`). This also makes the builder signature consistent with Go patterns where construction can fail. Update `buildNodes` to handle this error.

### 8. The buildDynamicAgent Return Type Should Be *taskagent.Agent

**Problem:** The design specifies `buildDynamicAgent` returns `agent.Agent`. However, `taskagent.New()` returns `*taskagent.Agent`, and returning the concrete type is more idiomatic in Go when the concrete type is known and provides additional methods.

**Improvement:** Return `*taskagent.Agent` from `buildDynamicAgent`. The `orchestrate.Node.Runner` field accepts any `Runner` interface, and `*taskagent.Agent` satisfies it. Returning the concrete type preserves access to methods like `Tools()` for debugging/logging if needed.

### 9. Missing Error Return in buildDynamicAgent

**Problem:** The design shows `buildDynamicAgent` returning only `agent.Agent` with no error. But several things can go wrong: unknown tool access level, nil registry when tool access is not "none", or future validation issues. Silently returning a misconfigured agent would cause confusing runtime failures.

**Improvement:** Change the signature to return `(*taskagent.Agent, error)`. On validation failure (e.g., unknown tool access level), return an error. The caller in `buildNodes` already handles errors and can fall back gracefully.

### 10. Test Strategy: Use Existing stubAgent Pattern

**Problem:** The design mentions creating tests in a new `dynamic_agent_test.go` file. The existing tests use a `stubAgent` and `mockChatCompleter` pattern defined in the test files.

**Improvement:** Add all new tests to `orchestrator_test.go`, reusing the existing `stubAgent` and `mockChatCompleter` test helpers. This avoids duplicating test infrastructure. The tests for `DynamicAgentSpec.validate()` and `buildDynamicAgent()` are small enough to fit naturally alongside the existing `ClassifyResult_Validate` and `buildNodes` tests.

### 11. Agent ID Naming Convention

**Problem:** The design uses `fmt.Sprintf("dynamic-%s-%s", spec.BaseType, stepID)` for agent IDs. This works but introduces a new naming pattern that does not follow the existing convention of simple lowercase IDs (e.g., "coder", "researcher", "planner").

**Improvement:** Use `fmt.Sprintf("dynamic_%s_%s", spec.BaseType, stepID)` (underscores instead of hyphens) to be consistent with the step ID convention that uses underscores (e.g., "step_1", "step_2"). Minor but keeps consistency.

## Items That Are Correct and Should Be Kept

1. **Ephemeral agents with no memory** -- correct per the requirement and avoids complexity.
2. **DynamicSpec as optional pointer field on PlanStep** -- clean backward compatibility via `omitempty`.
3. **Base type to default tool access mapping** -- good default behavior.
4. **Planner prompt extension approach** -- updating the existing prompt constant is the right approach.
5. **No changes to vage framework** -- correctly scoped to `vv/agents/`.
6. **Validation in ClassifyResult.validate()** -- right place for it.
7. **The overall flow: check DynamicSpec -> build or dispatch static** -- clean and simple.
