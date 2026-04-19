# Code Review: vv/agents Module Refactor

## Summary

Reviewed all 48 changed files across 8 packages (`registry/`, `dispatch/`, `lifecycle/`, `setup/`, `agents/`, `cli/`, `config/`, `integrations/`). The refactor is well-executed with clean package boundaries, proper interface compliance checks, comprehensive tests, and backward-compatible shims.

Build: PASS | Vet: PASS | Tests: PASS (all 11 packages)

---

## Issues Found and Fixed

### 1. Race condition in `TimingHook` (lifecycle/hooks.go) -- FIXED

**Severity: High**

`TimingHook.starts` map was accessed without synchronization. Both `OnBeforeRun` and `OnAfterRun` read/write to `h.starts`, and the lazy `if h.starts == nil` initialization in `OnBeforeRun` was also unprotected. If hooks are invoked concurrently (e.g., parallel DAG nodes), this is a data race.

**Fix:** Added `sync.Mutex` to guard all map access. Moved the logger call outside the critical section to minimize lock duration.

### 2. Non-deterministic message ordering in `StepInput.BuildMessages` (dispatch/input.go) -- FIXED

**Severity: Medium**

`BuildMessages()` iterated over `s.Upstream` (a `map[string]StepResult`) directly, producing non-deterministic ordering of upstream result messages. This means identical inputs could produce different LLM prompts across runs, leading to non-reproducible behavior.

**Fix:** Added sorted iteration over upstream keys before appending messages.

### 3. Non-deterministic agent ordering in `setup.Result.Agents()` (setup/setup.go) -- FIXED

**Severity: Low**

`Agents()` iterated over `r.subAgents` map, returning agents in arbitrary order. Callers iterating over the result (e.g., HTTP service registration) would see non-deterministic ordering.

**Fix:** Added `sort.Slice` by agent ID to ensure deterministic output.

---

## Observations (No Fix Required)

### 4. Stale template placeholder in `classifyDirect` (dispatch/classify.go:184)

`classifyDirect` calls `strings.Replace(systemPrompt, "{{.WorkingDir}}", d.workingDir, 1)`, but neither `PlannerSystemPrompt` nor `PlannerSystemPromptTemplate` contain a `{{.WorkingDir}}` placeholder. This is a harmless no-op but suggests a stale artifact from an earlier iteration. Consider removing the line or documenting the intent if a future template will use it.

### 5. Planner factory ignores dynamic system prompt (agents/planner.go:103)

The planner factory always uses the hardcoded `PlannerSystemPrompt` constant, ignoring the dynamic prompt built by `BuildPlannerSystemPrompt(reg)`. The dynamic prompt is only used via `WithPlannerSystemPrompt` in the `classifyDirect` fallback path. This is correct for the current setup flow but means the planner agent itself does not benefit from the registry-generated agent list. This is by design (the planner is created before all agents are registered, and setup calls `BuildPlannerSystemPrompt` separately), but worth noting for future maintainers.

### 6. Backward-compatible `compat.go` is thorough but verbose

The `agents/compat.go` file provides a full backward-compatible `Create()` function and `NewOrchestratorAgent()` shim. The shim manually reconstructs a registry from the `subAgents` map. This is correct and necessary for the deprecation period. Consider adding a deprecation timeline or removal target date.

### 7. `hookedAgent` does not implement `StreamAgent` (dispatch/dag.go)

The `hookedAgent` wrapper only implements `agent.Agent` (the `Run` method), not `agent.StreamAgent`. When a hooked agent is used in streaming DAG execution, it will fall back to non-streaming. This matches current behavior (DAG nodes use `Run`, not `RunStream`) but would need updating if streaming DAG support is added.

---

## Code Quality Assessment

**Strengths:**
- Clean separation of concerns across packages
- Compile-time interface checks (`var _ agent.Agent = (*Dispatcher)(nil)`)
- Comprehensive test coverage for all new packages
- Proper use of functional options pattern
- Thread-safe registry with `sync.RWMutex`
- Graceful degradation in error paths (explore/classify failures fall back to chat)
- `extractJSON` handles multiple JSON extraction patterns robustly

**Architecture:**
- Dependency graph is acyclic as designed
- No circular imports between `registry/`, `dispatch/`, `lifecycle/`, `setup/`, `agents/`
- `setup/` is the only package that knows about all other packages (proper composition root)
