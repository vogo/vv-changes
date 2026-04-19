# Code Review: Dispatcher Refactoring to Adaptive Decision Loop

## Summary

The refactoring replaces the rigid explore -> classify -> dispatch pipeline with an adaptive decision loop: intent recognition -> execution (with replanning) -> optional summarization. All tests pass and lint is clean. The implementation is well-structured and consistent with the design document. Below are the issues found, categorized by severity.

## Issues Found

### Issue 1 (Bug): `topologicalLayers` infinite loop on cyclic dependencies

**File:** `dispatches/execute.go`, lines 233-291

**Description:** The `calcDepth` function uses memoization via the `depths` map but has no cycle detection. If plan steps contain a cyclic dependency (e.g., step A depends on B, B depends on A), `calcDepth` will recurse infinitely and cause a stack overflow.

**Severity:** Medium -- the LLM would have to produce a cyclic plan, but there is no validation preventing it.

**Fix:** Add a "visiting" sentinel to detect cycles and return an error or break the cycle.

### Issue 2 (Bug): `partialResponse` uses `context.Background()` instead of caller's context

**File:** `dispatches/execute.go`, line 301

**Description:** `partialResponse` calls `aggregator.Aggregate(context.Background(), ...)` which ignores the caller's context. If the parent context is cancelled, this aggregation will still proceed. This is inconsistent with the rest of the codebase which propagates context throughout.

**Severity:** Low -- in practice the aggregation is fast, but it is a correctness issue.

**Fix:** Accept `ctx context.Context` as a parameter and pass it through.

### Issue 3 (Bug): `RunStream` summarize phase is a no-op

**File:** `dispatches/dispatch.go`, lines 317-334

**Description:** The `RunStream` method emits `PhaseStart("summarize")` and `PhaseEnd("summarize")` but does not actually perform any summarization between them. The `summarizeStart` and `summarizeDuration` are captured immediately with no work in between. This means the streaming path never generates a summary even when `shouldSummarize` returns true.

**Severity:** Medium -- the non-streaming `Run` method correctly summarizes, so this is a streaming-only gap.

**Fix:** Implement actual summarization in the streaming path (invoke the summarizer agent and forward its events).

### Issue 4 (Code Quality): Duplicated validation logic between `ClassifyResult` and `IntentResult`

**File:** `dispatches/types.go`, lines 33-64 and 85-116

**Description:** The `validate` methods on `ClassifyResult` and `IntentResult` are nearly identical. Both check `"direct"` vs `"plan"` modes, validate agent references, and validate dynamic specs. This duplicated code increases maintenance burden and risk of divergence.

**Severity:** Low -- not a bug, but a code smell.

**Fix:** Extract a shared validation helper function.

### Issue 5 (Code Quality): Custom `contains`/`containsSubstring` in tests instead of `strings.Contains`

**File:** `dispatches/intent_test.go`, lines 316-328

**Description:** The test file defines custom `contains` and `containsSubstring` functions that replicate `strings.Contains`. The `strings` package is already imported in many other test files.

**Severity:** Low -- no functional impact, just unnecessary code.

**Fix:** Replace with `strings.Contains`.

### Issue 6 (Robustness): `replanIfNeeded` ignores the context parameter

**File:** `dispatches/execute.go`, line 202

**Description:** The `replanIfNeeded` method accepts a context but uses `_` to discard it. The design document specifies that replanning should "call the planner LLM with context about what succeeded and what failed." The current implementation only does simple retries of failed steps without LLM involvement. While the comment notes this is intentional ("Future: use LLM to generate intelligent replacement steps"), the unused context parameter is misleading.

**Severity:** Low -- the retry-only approach is documented as intentional for this iteration.

**Fix:** No code change needed for this iteration; the unused context is acceptable as a placeholder for future LLM-based replanning.

### Issue 7 (Robustness): `shouldSummarize` logic has unreachable code path for `request_summary`

**File:** `dispatches/summarize.go`, lines 20-35

**Description:** In the `SummaryAuto` case, if `mode` is present in metadata (even if it is not `"http"`), the function returns `false` before ever checking `request_summary`. This means a CLI-mode request with `request_summary: true` would return `false` instead of `true`. The `request_summary` check is only reached if the `mode` key is absent entirely.

**Severity:** Medium -- parent-requested summarization is silently ignored when mode metadata is present.

**Fix:** Check `request_summary` before returning `false` from the mode check, or restructure the conditionals.

### Issue 8 (Consistency): `applyDefaults` does not set defaults for new orchestrate config fields

**File:** `configs/config.go`, function `applyDefaults`

**Description:** The `MaxRecursionDepth`, `SummaryPolicy`, and `Replan.MaxReplans` fields in `OrchestrateConfig` have no defaults applied in `applyDefaults`. Defaults are applied later in `setup/setup.go`. This is inconsistent with how other config fields (like `MaxConcurrency`) are handled and means direct users of `configs.Load` without going through `setup.New` get zero-value behavior.

**Severity:** Low -- all production paths go through `setup.New`.

**Fix:** Add defaults for the new fields in `applyDefaults`.

## Positive Observations

1. **Clean separation of concerns:** Each new file (`depth.go`, `intent.go`, `summarize.go`, `execute.go`) has a clear single responsibility.
2. **Good test coverage:** All new code has corresponding unit tests with meaningful assertions.
3. **Backward compatibility:** The deprecated `WithPlannerSystemPrompt` alias is properly forwarded.
4. **Functional options pattern:** New options follow the established pattern consistently.
5. **Context propagation:** Depth is properly propagated through context values.
6. **Phase event structure:** The streaming phase events are well-structured with proper start/end pairing.

## Applied Fixes

- Issue 1: Added cycle detection to `topologicalLayers` with a `visiting` set.
- Issue 2: Changed `partialResponse` to accept and use the caller's context.
- Issue 3: Implemented actual summarization in the `RunStream` summarize phase.
- Issue 5: Replaced custom `contains`/`containsSubstring` with `strings.Contains`.
- Issue 7: Fixed `shouldSummarize` to check `request_summary` even when `mode` metadata is present.
- Issue 8: Added defaults for new orchestrate config fields in `applyDefaults`.
