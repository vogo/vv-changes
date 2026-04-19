# Design Review: Dispatcher Refactoring to Adaptive Decision Loop

## Summary

The design is well-structured and addresses the requirements comprehensively. The key architectural decision -- replacing the rigid explore-classify-dispatch pipeline with an adaptive intent-recognition loop -- is sound. However, several issues were found by validating the design against the actual codebase. The issues range from incorrect assumptions about existing code to missing edge cases and implementation feasibility concerns.

---

## Issue 1: `hookedAgent` Does Not Implement `StreamAgent`

**Severity:** High  
**Location:** Design Section 4 (Execution with Replanning), existing `dag.go`

The existing `hookedAgent` wrapper in `dag.go` (lines 263-288) only implements `agent.Agent`, not `agent.StreamAgent`. When a sub-agent is wrapped with hooks, it loses its streaming capability. The design proposes `forwardSubAgentStream` for streaming execution of sub-agents, which checks for `agent.StreamAgent` via type assertion. A hooked agent will always fail this check and fall back to non-streaming `Run`.

This is an existing deficiency, but the new design makes it more visible since `executeTaskStream` relies on streaming paths. The design should either:
- (a) Add `RunStream` to `hookedAgent` (delegating to inner's `RunStream` if it implements `StreamAgent`), or
- (b) Explicitly document that hooked agents use the non-streaming fallback path and accept the limitation.

**Recommendation:** Add a `RunStream` method to `hookedAgent` that delegates to the inner agent's `RunStream` when available.

---

## Issue 2: Replanning Layer-by-Layer Strategy Loses Intra-Layer Parallelism Information

**Severity:** Medium  
**Location:** Design Section 4 (Execution with Replanning)

The design proposes executing the DAG in "topological layers" when replanning is enabled, checking for replan triggers after each layer. However, the `orchestrate` package's `topologicalSort` function (in `topo.go`) returns a flat list of node IDs, not layers. The design does not specify how to partition the topological order into layers.

Additionally, executing layer-by-layer means all nodes in a layer must complete before the next layer starts. This serializes work unnecessarily within a layer when only one node failed. For example, if layer 2 has nodes A, B, C and only A fails, B and C's results are still valid and should be preserved.

**Recommendation:**
- Add a `topologicalLayers` utility function that groups nodes by their topological depth (nodes with no deps = layer 0, nodes whose deps are all in layer 0 = layer 1, etc.).
- Clarify that within each layer, nodes still execute in parallel (using `ExecuteDAG` with just that layer's nodes).
- Specify that replanning only replaces unexecuted nodes plus the failed node, preserving completed nodes' results from the current and previous layers.

---

## Issue 3: Missing `Metadata` Propagation for Mode Detection

**Severity:** Medium  
**Location:** Design Section 6 (Summarization)

The `shouldSummarize` method checks `req.Metadata["mode"]` for `"http"` vs `"cli"`. However, examining the codebase, there is no existing mechanism that sets this metadata field. The `RunRequest.Metadata` field exists (`map[string]any`), but neither the CLI nor HTTP entry points currently populate a `"mode"` key.

**Recommendation:**
- Document that the `"mode"` metadata key must be set by the CLI and HTTP entry points.
- Add the mode-setting logic to the `setup.go` or the caller of `Dispatcher.Run`/`RunStream` as part of Task 9.
- Consider using a typed context value instead of stringly-typed metadata, consistent with how the design handles depth via `context.Context`.

---

## Issue 4: Intent Recognition Uses Raw LLM Call But Explorer Is an Agent

**Severity:** Medium  
**Location:** Design Section 2 (Intent Recognition)

The design says `recognizeIntent` makes "one LLM call" for intent recognition, implying a direct `aimodel.ChatCompletion` call (similar to `classifyDirect`). However, the explorer is invoked as an `agent.Agent` (which internally makes its own LLM calls and tool calls). The design's `recognizeIntentStream` signature returns `(*IntentResult, string, *aimodel.Usage, error)` where `string` is presumably the context summary.

The issue: the design conflates two distinct operations in the flow description:
1. The LLM call for intent classification (direct `ChatCompletion`)
2. The explorer agent invocation (a full agent run with tools)

The flow should be clearer about which component handles which call. Specifically:
- Intent recognition = direct LLM call (fast, no tools)
- On-demand exploration = full explorer agent run (may make multiple LLM calls and tool calls)
- Re-assessment after exploration = another direct LLM call (to re-classify with context)

**Recommendation:** Rename the second return value from the generic `string` to a named parameter or document it clearly. Restructure the flow description to clearly separate the three potential LLM interactions.

---

## Issue 5: `classifyDirect` Fallback Should Be Preserved

**Severity:** Medium  
**Location:** Design Section 2 and File Changes Summary

The existing `classifyDirect` method (classify.go:182-227) is a useful fallback that works without a planner agent, using a direct `aimodel.ChatCompletion` call. The design deletes `classify.go` and absorbs its logic into `intent.go`. However, the design's `recognizeIntent` description only mentions using "one LLM call" without specifying whether it uses the planner agent or a direct LLM call.

The current code has two paths: planner-agent-based classification and direct-LLM classification. The new intent recognizer should preserve this dual-path behavior.

**Recommendation:** Explicitly state that `recognizeIntent` uses a direct LLM call (similar to `classifyDirect`) by default, and only uses the planner agent if one is configured and the task is complex. This aligns with the goal of reducing mandatory pipeline stages.

---

## Issue 6: Backward Compatibility Concern for `agents/compat.go`

**Severity:** Medium  
**Location:** File Changes Summary, `agents/compat.go`

The `agents/compat.go` file contains deprecated shims (`NewOrchestratorAgent`, `Create`) that construct a `dispatches.Dispatcher` with the old constructor signature. The design says "the constructor signature does not change" but adds new `Option` functions. The compat layer does not pass the new options.

While this is acceptable (new options have sensible defaults), it should be explicitly documented that the deprecated compat path uses default policies (no replanning, auto summary, depth 2).

**Recommendation:** Add a note that `agents/compat.go` requires no changes and uses default values for all new options.

---

## Issue 7: Deviation Detection Is Under-Specified

**Severity:** Medium  
**Location:** Design Section 4 (Replanning Logic)

The design mentions `ReplanPolicy.TriggerOnDeviation` and "the step's output is flagged as deviated," but does not specify:
1. What constitutes a "deviation" from expectations
2. Who/what flags the deviation (the LLM? a heuristic? the step result metadata?)
3. How the deviation assessment is triggered without an additional LLM call per step

Since `OnNodeComplete` only receives `nodeID`, `status`, and `err`, there is no mechanism to assess content quality without an extra LLM call.

**Recommendation:** Either:
- (a) Define deviation detection as an LLM call that evaluates each completed step's output against the original plan step description (expensive but accurate). Specify the prompt and cost implications.
- (b) Defer `TriggerOnDeviation` to a future iteration and only implement `TriggerOnFailure` in this refactoring. Failure is deterministic (err != nil); deviation is subjective.
- (c) Use a lightweight heuristic (e.g., empty response, response too short, explicit error keywords).

Option (b) is recommended to keep the scope manageable.

---

## Issue 8: `RunStream` Main Loop Missing Error Accumulation

**Severity:** Low  
**Location:** Design Section 7 (Updated Main Loop)

The design shows the `Run` method but only describes `RunStream` as "follows the same structure." However, `RunStream` is significantly more complex because:
1. Phase events must be emitted incrementally
2. Error handling during streaming cannot easily "restart" a phase
3. The `send` function can itself fail

The current `RunStream` implementation (dispatch.go:191-323) is ~130 lines with careful event management. The design should provide more detail on the streaming variant, especially:
- How intent recognition phase events are emitted when exploration is triggered mid-phase (does the "intent" phase nest an "explore" sub-phase, or does it end and restart?)
- How partial failures during streaming are surfaced

**Recommendation:** Add a pseudo-code outline for `RunStream` that shows the phase event emission sequence for the three main scenarios: (1) simple intent, (2) intent with exploration, (3) complex plan with replanning.

---

## Issue 9: Summary Node in DAG vs. New Summarization Phase

**Severity:** Low  
**Location:** Design Sections 4 and 6

The existing `buildNodes` method (dag.go:70-148) appends a "summary" node to the DAG when there are multiple terminal nodes. This is a DAG-level aggregation mechanism. The design introduces a separate `summarize` phase after execution. These two summarization paths could conflict or produce redundant summaries.

**Recommendation:** Clarify the relationship:
- The DAG summary node (in `buildNodes`) handles plan-level result aggregation (combining multiple step outputs).
- The new summarization phase (Section 6) handles user-facing final summary generation.
- When both apply (multi-step plan in HTTP mode), the DAG summary feeds into the final summarization phase.
- Consider removing the DAG summary node in favor of always using the new summarization phase, to avoid confusion.

---

## Issue 10: `depth` Parameter Is Both In Context and In Method Signatures

**Severity:** Low  
**Location:** Design Sections 2, 4, 5

The design propagates depth through `context.Context` (Section 5: `WithDepth`, `DepthFrom`) but also passes `depth int` as a method parameter to `recognizeIntent` and `executeTask`. This is redundant and creates a risk of inconsistency.

**Recommendation:** Use only context-based depth. Extract it with `DepthFrom(ctx)` inside each method. This simplifies method signatures and eliminates the risk of passing a depth value that differs from what is in the context.

---

## Issue 11: Task Dependencies Have a Gap Between Tasks 3 and 5

**Severity:** Low  
**Location:** Implementation Plan

Task 5 (Execution with Replanning) depends on Task 3 (Intent Recognition). However, `executeTask` only needs `IntentResult`, not the full intent recognition system. The execution logic should be implementable with just the types from Task 1, not the intent recognition implementation.

**Recommendation:** Change Task 5 dependencies from "Task 1, Task 2, Task 3" to "Task 1, Task 2". Task 5 consumes `IntentResult` as a data structure, not the `recognizeIntent` function.

---

## Issue 12: Missing Consideration for `PlannerSystemPrompt` Option

**Severity:** Low  
**Location:** Design Sections 2-3, API Contracts

The current Dispatcher has a `plannerSystemPrompt` field set via `WithPlannerSystemPrompt`, used by `classifyDirect`. The design replaces `classifyDirect` with `recognizeIntent` but does not address what happens to this option. It introduces `IntentSystemPromptTemplate` and `BuildIntentSystemPrompt` but does not mention whether `WithPlannerSystemPrompt` is deprecated, renamed, or repurposed.

**Recommendation:** Rename `WithPlannerSystemPrompt` to `WithIntentSystemPrompt` (or deprecate and add a new option). Update `setup.go` Task 9 to call `BuildIntentSystemPrompt` instead of `BuildPlannerSystemPrompt`.

---

## Issue 13: Streaming DAG Handler Needs Replanning Awareness

**Severity:** Low  
**Location:** Design Section 7 (Phase Events)

The existing `streamingDAGHandler` (stream.go:190-235) uses fixed `stepIndex`, `totalSteps`, and `stepAgent` maps built once from the original plan. When replanning occurs, these maps become stale -- new steps have unknown indices, the total step count changes, and agent assignments may differ.

**Recommendation:** The `streamingDAGHandler` must be rebuilt (or updated) after each replan. Add a method to update the handler's step metadata when a new plan segment begins. Include a `replanGeneration` field in `SubAgentStartData` for clients to distinguish original vs. replanned steps.

---

## Minor Observations

1. The `IntentResult.Mode` field uses `string` type with magic values `"direct"` and `"plan"`. Consider using the `IntentType` enum already defined, or clarify why both `IntentType` and `Mode` exist.

2. The design creates 5 new files (`intent.go`, `intent_prompt.go`, `depth.go`, `summarize.go`, `execute.go`) and deletes 2 (`classify.go`, `explore.go`). This is a net increase of 3 files. Consider whether `intent_prompt.go` could be merged into `intent.go` to keep file count manageable.

3. The `DefaultReplanPolicy` has all triggers disabled (`TriggerOnFailure: false`). This means replanning is opt-in, which is good for backward compatibility but means the feature is effectively dormant unless explicitly configured. This is the right default.

4. The `ReplanPolicy` struct uses `json` tags alongside `yaml` tags. The `json` tags are useful for wire format but `ReplanPolicy` instances are configured via YAML config, not HTTP requests. Consider whether the `json` tags are necessary on the policy struct (they are needed on `Plan` and `PlanStep` which are LLM responses).
