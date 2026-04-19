# Technical Design: Dispatcher Refactoring to Adaptive Decision Loop

## Architecture Overview

The current Dispatcher implements a rigid three-phase pipeline: **explore -> classify -> dispatch**. Every request traverses all phases regardless of complexity. This design replaces that pipeline with an adaptive decision loop modeled after the same ReAct pattern used by sub-agents (TaskAgent).

### Current Flow

```
User Request
  -> explore(always)     [1 LLM call + tool calls]
  -> classify(always)    [1 LLM call]
  -> dispatch(direct | DAG)
```

### New Flow

```
User Request
  -> intentRecognize     [1 direct LLM call: determines intent + optional plan]
     |-- may invoke explorer agent on-demand (full agent run with tools)
     |-- may re-assess intent after exploration (1 additional LLM call)
     |-- may produce plan inline (intent + plan merged)
  -> execute             [direct dispatch OR DAG with replanning]
     |-- may invoke explorer mid-execution
     |-- may trigger replanning on failure
  -> summarize(optional) [conditional on SummaryPolicy]
```

The key architectural shift: **explorer and planner become capabilities the LLM can choose to invoke**, rather than mandatory pipeline stages. A single "intent recognition" direct LLM call replaces both the explore and classify stages for simple tasks.

### Scope Constraint

All changes are confined to `vv/dispatches/`. No changes to `vage/` framework code, agent implementations in `vv/agents/`, or HTTP/CLI layers (except metadata propagation for mode detection -- see Section 6).

## Component Design

### 1. Dispatcher Struct Changes (`dispatch.go`)

The `Dispatcher` struct gains three new fields for the new policies and depth control. The `explorerAgent` and `plannerAgent` fields remain but their usage changes from "always invoked" to "invoked on-demand."

```go
type Dispatcher struct {
    // ... existing fields unchanged ...

    // New fields
    summaryPolicy     SummaryPolicy   // controls summarization behavior
    replanPolicy      ReplanPolicy    // controls dynamic replanning
    maxRecursionDepth int             // max nesting depth (default: 2)
    summarizer        agent.Agent     // agent for generating summaries (reuse planGen)
}
```

New functional options:

```go
func WithSummaryPolicy(p SummaryPolicy) Option
func WithReplanPolicy(p ReplanPolicy) Option
func WithMaxRecursionDepth(n int) Option
```

The existing `WithPlannerSystemPrompt` option is renamed to `WithIntentSystemPrompt` to reflect its new purpose. A deprecated alias is kept for backward compatibility.

### 2. Intent Recognition (`intent.go`, new file)

Replaces the separate `explore` + `classify` two-step pipeline with a direct LLM call that performs intent recognition and optional planning in one shot.

The intent recognition flow involves up to three distinct operations:
1. **Intent classification** -- a direct `aimodel.ChatCompletion` call (fast, no tools) that assesses the request and returns a structured JSON response.
2. **On-demand exploration** -- if the LLM returns `needs_exploration: true`, the explorer agent is invoked as a full agent run (may make multiple LLM calls and tool calls internally).
3. **Re-assessment** -- after exploration completes, a second direct LLM call re-classifies with the exploration context.

This preserves the existing dual-path behavior: when no planner agent is configured, `recognizeIntent` uses a direct LLM call (equivalent to the current `classifyDirect`). When a planner agent is configured, it can optionally be used for complex planning scenarios.

```go
// IntentResult represents the output of intent recognition.
type IntentResult struct {
    NeedsExploration bool           `json:"needs_exploration"` // LLM wants project context
    Mode             string         `json:"mode"`              // "direct" or "plan"
    Agent            string         `json:"agent"`             // for direct mode
    Plan             *Plan          `json:"plan"`              // for plan mode
}

// recognizeIntent performs intent recognition, optionally invoking the explorer.
// This replaces the separate explore() + classify() calls.
// Returns the intent result, context summary from exploration (empty if none),
// token usage, and error.
func (d *Dispatcher) recognizeIntent(ctx context.Context, req *schema.RunRequest) (*IntentResult, string, *aimodel.Usage, error)

// recognizeIntentStream is the streaming variant.
func (d *Dispatcher) recognizeIntentStream(ctx context.Context, req *schema.RunRequest, send func(schema.Event) error) (*IntentResult, string, *aimodel.Usage, error)
```

Depth is extracted from context via `DepthFrom(ctx)` rather than passed as a parameter.

The intent recognition flow:
1. Extract depth with `DepthFrom(ctx)`. If `depth >= maxRecursionDepth`, skip intent recognition and force direct execution with the fallback agent.
2. Make one direct LLM call (`aimodel.ChatCompletion`) with the intent recognition prompt.
3. If the LLM returns `needs_exploration: true`, invoke the explorer agent (full agent run), then make a second direct LLM call with the exploration context to re-classify.
4. Parse and validate the `IntentResult` (reusing the existing `extractJSON` helper and validation logic adapted from `ClassifyResult.validate`).

This means: simple requests ("hello") require exactly 1 LLM call. Complex requests that need exploration require 1 LLM call + 1 explorer agent run + 1 LLM call. This is the same or fewer than today's mandatory 2+ calls.

### 3. Intent Recognition Prompt (`intent.go`, embedded)

The intent recognition system prompt is constructed dynamically from the registry (same as today's planner prompt) but with expanded instructions. The prompt template and builder are co-located in `intent.go` rather than a separate file, to reduce file count.

```go
// intentSystemPromptTemplate is the template for the intent recognition prompt.
const intentSystemPromptTemplate = `You are an intelligent task router. ...`

// buildIntentSystemPrompt constructs the prompt from the registry.
func buildIntentSystemPrompt(reg *registries.Registry) string
```

The prompt instructs the LLM to respond with JSON:
```json
{"needs_exploration": false, "mode": "direct", "agent": "chat"}
```
or
```json
{"needs_exploration": true}
```
or
```json
{"needs_exploration": false, "mode": "plan", "plan": {"goal": "...", "steps": [...]}}
```

### 4. Execution with Replanning (`execute.go`, replaces parts of `dag.go`)

The execution phase handles both direct dispatch and DAG execution. For DAG execution, it adds replanning capability.

```go
// executeTask runs the dispatched task(s), handling direct and plan modes.
// For plan mode, monitors step results and triggers replanning if needed.
func (d *Dispatcher) executeTask(ctx context.Context, req *schema.RunRequest, intent *IntentResult, contextSummary string) (*schema.RunResponse, *aimodel.Usage, error)

// executeTaskStream is the streaming variant.
func (d *Dispatcher) executeTaskStream(ctx context.Context, req *schema.RunRequest, intent *IntentResult, contextSummary string, send func(schema.Event) error) (*aimodel.Usage, error)
```

Depth is extracted from context via `DepthFrom(ctx)` inside the method.

#### Replanning Logic

Replanning is implemented as a post-layer evaluation within the DAG execution. Rather than modifying the `vage/orchestrate` package, the Dispatcher breaks the DAG into topological layers and executes each layer using `orchestrate.ExecuteDAG`.

**Topological layer partitioning:** A `topologicalLayers` utility function groups nodes by their topological depth: nodes with no dependencies form layer 0, nodes whose dependencies are all in layer 0 form layer 1, etc. Within each layer, nodes execute in parallel via `ExecuteDAG`.

```go
// topologicalLayers groups DAG nodes into layers by their topological depth.
// Layer 0 = nodes with no deps, layer N = nodes whose deps are all in layers < N.
func topologicalLayers(nodes []orchestrate.Node) [][]orchestrate.Node
```

The replanning strategy:
1. Partition the plan's nodes into topological layers.
2. Execute each layer using `orchestrate.ExecuteDAG` with the current layer's nodes.
3. After each layer completes, check results: if a step failed and `ReplanPolicy.TriggerOnFailure` is true, trigger replanning.
4. Replanning calls the planner LLM with: original goal, completed step results, the failure info, and asks for replacement steps for the remaining work. Completed nodes from all layers are preserved.
5. The remaining layers' nodes are replaced with the new plan's nodes. The `streamingDAGHandler` metadata (step indices, total steps, agent mappings) is rebuilt for the new plan segment.
6. Track `replanCount`; if it exceeds `ReplanPolicy.MaxReplans`, abort with partial results.

```go
// executePlanWithReplanning executes a plan with dynamic replanning support.
// It runs the DAG in topological layers, checking for replan triggers after each layer.
func (d *Dispatcher) executePlanWithReplanning(ctx context.Context, req *schema.RunRequest, plan *Plan, contextSummary string) (*schema.RunResponse, *aimodel.Usage, error)

// replanIfNeeded checks a completed layer's results and determines if replanning is required.
// Returns nil if no replanning is needed, or a new Plan for the remaining work.
func (d *Dispatcher) replanIfNeeded(ctx context.Context, plan *Plan, completedResults map[string]*schema.RunResponse, failedNodes map[string]error) (*Plan, error)
```

When replanning is disabled (the default for backward compatibility), the existing `orchestrate.ExecuteDAG` is used directly on the full DAG, preserving full parallelism.

**Deviation detection:** The `ReplanPolicy.TriggerOnDeviation` field is defined in the type system but **not implemented in this iteration**. Deviation detection requires an additional LLM call per completed step to assess output quality, which has significant cost implications. The field defaults to `false` and is reserved for future implementation. Only `TriggerOnFailure` (deterministic: `err != nil`) is implemented.

#### `hookedAgent` Streaming Support

The existing `hookedAgent` wrapper (dag.go) only implements `agent.Agent`, not `agent.StreamAgent`. This causes wrapped agents to always fall back to non-streaming execution in `forwardSubAgentStream`. This refactoring adds `RunStream` to `hookedAgent`:

```go
func (h *hookedAgent) RunStream(ctx context.Context, req *schema.RunRequest) (*schema.RunStream, error) {
    for _, hook := range h.hooks {
        if err := hook.OnBeforeRun(ctx, h.agentID, req); err != nil {
            return nil, fmt.Errorf("hook aborted run for %q: %w", h.agentID, err)
        }
    }

    sa, ok := h.inner.(agent.StreamAgent)
    if !ok {
        return agent.RunToStream(ctx, h.inner, req), nil
    }

    // Wrap the stream to call OnAfterRun when the stream completes.
    stream, err := sa.RunStream(ctx, req)
    if err != nil {
        for i := len(h.hooks) - 1; i >= 0; i-- {
            h.hooks[i].OnAfterRun(ctx, h.agentID, nil, err)
        }
        return nil, err
    }

    return stream, nil
}
```

This ensures hooked agents can stream when their inner agent supports it.

### 5. Recursion Depth Control (`depth.go`, new file)

Depth is propagated through context values, not method parameters, to avoid coupling and eliminate inconsistency risks.

```go
type depthKey struct{}

// WithDepth returns a context with the given recursion depth.
func WithDepth(ctx context.Context, depth int) context.Context

// DepthFrom returns the recursion depth from the context (0 if not set).
func DepthFrom(ctx context.Context) int

// IncrementDepth returns a new context with depth incremented by 1.
func IncrementDepth(ctx context.Context) context.Context
```

Depth-based behavior in the main loop:
- **Depth 0** (Orchestrator): Full decision loop -- intent recognition, optional planning, execution with replanning, optional summarization
- **Depth 1**: Can plan but does not further decompose. Replanning is disabled.
- **Depth >= maxRecursionDepth** (default 2): Direct execution only -- skip intent recognition, execute with the first matching agent

All methods (`recognizeIntent`, `executeTask`, etc.) read depth from context via `DepthFrom(ctx)` rather than accepting a `depth int` parameter.

### 6. Summarization (`summarize.go`, new file)

```go
// SummaryPolicy controls when summarization occurs.
type SummaryPolicy string

const (
    SummaryAuto   SummaryPolicy = "auto"   // HTTP=yes, CLI=no
    SummaryAlways SummaryPolicy = "always"
    SummaryNever  SummaryPolicy = "never"
)

// shouldSummarize determines if summarization should run based on policy and context.
func (d *Dispatcher) shouldSummarize(req *schema.RunRequest) bool

// summarize generates a summary of the execution results.
func (d *Dispatcher) summarize(ctx context.Context, req *schema.RunRequest, results []*schema.RunResponse) (*schema.RunResponse, error)
```

The `shouldSummarize` method checks:
1. If `SummaryPolicy` is `always` or `never`, use that directly.
2. If `auto`, check `req.Metadata["mode"]` for `"http"` vs `"cli"`.
3. Sub-agents only summarize if their parent requests it via `req.Metadata["request_summary"] == true`.

**Mode metadata propagation:** The `"mode"` metadata key must be set by callers of the Dispatcher. This is wired in `setup.go` Task 9: the CLI entry point sets `Metadata["mode"] = "cli"` and the HTTP entry point sets `Metadata["mode"] = "http"` on the `RunRequest` before calling the Dispatcher.

**Relationship to DAG summary node:** The existing `buildNodes` method appends a "summary" node to DAGs with multiple terminal nodes. This handles plan-level result aggregation (combining multiple step outputs into one response). The new summarization phase handles user-facing final summary generation. When both apply (multi-step plan in HTTP mode), the DAG summary node's output feeds into the final summarization phase. These are distinct concerns: internal aggregation vs. user-facing summary.

### 7. Updated Main Loop (`dispatch.go`)

The `Run` and `RunStream` methods are rewritten to follow the new decision loop.

```go
func (d *Dispatcher) Run(ctx context.Context, req *schema.RunRequest) (*schema.RunResponse, error) {
    depth := DepthFrom(ctx)

    // Fast path: at max depth, execute directly with fallback agent
    if depth >= d.maxRecursionDepth {
        return d.fallbackRun(ctx, req, nil)
    }

    // Phase 1: Intent Recognition (may invoke explorer on-demand)
    intent, contextSummary, intentUsage, err := d.recognizeIntent(ctx, req)
    if err != nil {
        return d.fallbackRun(ctx, req, nil)
    }

    // Phase 2: Execute
    enrichedReq := d.enrichRequest(req, contextSummary)
    childCtx := IncrementDepth(ctx)

    resp, execUsage, err := d.executeTask(childCtx, enrichedReq, intent, contextSummary)
    if err != nil {
        return nil, err
    }

    totalUsage := aggregateUsage(intentUsage, execUsage)
    resp.Usage = totalUsage

    // Phase 3: Summarize (optional)
    if d.shouldSummarize(req) && len(resp.Messages) > 0 {
        summaryResp, err := d.summarize(ctx, req, []*schema.RunResponse{resp})
        if err == nil {
            resp = summaryResp
            resp.Usage = aggregateUsage(totalUsage, summaryResp.Usage)
        }
    }

    return resp, nil
}
```

#### `RunStream` Pseudo-Code

The `RunStream` method is more complex due to incremental phase event emission. Here is the structure for the three main scenarios:

**Scenario 1: Simple intent (e.g., "hello")**
```
emit PhaseStart("intent", idx=1, total=0)
  -> 1 LLM call -> IntentResult{mode: "direct", agent: "chat"}
emit PhaseEnd("intent")
emit PhaseStart("execute", idx=2, total=0)
  -> forwardSubAgentStream(chatAgent)
emit PhaseEnd("execute")
```

**Scenario 2: Intent with exploration (e.g., "add tests for types.go")**
```
emit PhaseStart("intent", idx=1, total=0)
  -> 1 LLM call -> IntentResult{needs_exploration: true}
  emit PhaseStart("explore", idx=0, total=0)   // nested sub-phase
    -> explorer agent run (tool events forwarded)
  emit PhaseEnd("explore")
  -> 1 LLM call (re-assess) -> IntentResult{mode: "direct", agent: "coder"}
emit PhaseEnd("intent")
emit PhaseStart("execute", idx=2, total=0)
  -> forwardSubAgentStream(coderAgent)
emit PhaseEnd("execute")
```

**Scenario 3: Complex plan with replanning**
```
emit PhaseStart("intent", idx=1, total=0)
  -> 1 LLM call -> IntentResult{mode: "plan", plan: {...}}
emit PhaseEnd("intent")
emit PhaseStart("execute", idx=2, total=0)
  -> execute layer 0 (SubAgentStart/End events per step)
  -> execute layer 1 -> step fails
  emit PhaseStart("replan", idx=0, total=0)    // nested sub-phase
    -> 1 LLM call -> new replacement steps
  emit PhaseEnd("replan")
  -> execute replacement layer (SubAgentStart/End events)
emit PhaseEnd("execute")
emit PhaseStart("summarize", idx=3, total=0)   // only if SummaryPolicy requires it
  -> 1 LLM call -> summary
emit PhaseEnd("summarize")
```

`TotalPhase` is set to `0` to indicate "unknown total" since we cannot predict ahead of time whether exploration, replanning, or summarization will occur. The CLI adapts by showing phase names as they appear rather than a progress bar.

### 8. Phase Events (`stream.go` updates)

Phase events change from fixed three phases to dynamic phases that appear only when they actually execute:

| Phase Name | Condition |
|-----------|-----------|
| `"intent"` | Always emitted (replaces `"explore"` + `"plan"`) |
| `"explore"` | Emitted only when explorer is invoked on-demand (nested within intent phase) |
| `"execute"` | Always emitted (replaces `"dispatch"`) |
| `"replan"` | Emitted when replanning occurs (nested within execute phase) |
| `"summarize"` | Emitted only when summarization runs |

The `streamingDAGHandler` must be rebuilt after each replan to update step metadata (indices, total count, agent mappings). A `replanGeneration` field is added to `SubAgentStartData` for clients to distinguish original vs. replanned steps:

```go
// SubAgentStartData gains a ReplanGeneration field.
type SubAgentStartData struct {
    AgentName        string `json:"agent_name"`
    StepID           string `json:"step_id,omitempty"`
    Description      string `json:"description,omitempty"`
    StepIndex        int    `json:"step_index,omitempty"`
    TotalSteps       int    `json:"total_steps,omitempty"`
    ReplanGeneration int    `json:"replan_generation,omitempty"` // 0=original, 1+=replanned
}
```

Note: `SubAgentStartData` is defined in `vage/schema/event.go`, which is out of scope for this refactoring. Instead, the `replan_generation` information is conveyed through the step ID naming convention (e.g., `"step_1_r1"` for generation 1) and the "replan" phase events bracket replanning clearly. This avoids modifying the `vage` package.

## Data Models

### New Types (`types.go` additions)

```go
// IntentType classifies the complexity of a user request.
type IntentType string

const (
    IntentSimple  IntentType = "simple"  // single agent, no planning
    IntentComplex IntentType = "complex" // multi-step, needs planning
)

// IntentResult is the output of the intent recognition phase.
// Mode uses string values "direct" (maps to IntentSimple) and "plan" (maps to IntentComplex)
// for JSON compatibility with the LLM response format.
type IntentResult struct {
    NeedsExploration bool   `json:"needs_exploration"`
    Mode             string `json:"mode"`              // "direct" or "plan"
    Agent            string `json:"agent,omitempty"`    // for direct mode
    Plan             *Plan  `json:"plan,omitempty"`     // for plan mode
}

// ReplanPolicy controls dynamic replanning behavior.
type ReplanPolicy struct {
    TriggerOnFailure   bool `yaml:"trigger_on_failure"`
    TriggerOnDeviation bool `yaml:"trigger_on_deviation"` // reserved for future use; not implemented in this iteration
    MaxReplans         int  `yaml:"max_replans"`
}

// DefaultReplanPolicy returns a ReplanPolicy with sensible defaults.
// All triggers are disabled for backward compatibility.
func DefaultReplanPolicy() ReplanPolicy {
    return ReplanPolicy{
        TriggerOnFailure:   false,
        TriggerOnDeviation: false,
        MaxReplans:         2,
    }
}

// SummaryPolicy controls when summarization occurs.
type SummaryPolicy string

const (
    SummaryAuto   SummaryPolicy = "auto"
    SummaryAlways SummaryPolicy = "always"
    SummaryNever  SummaryPolicy = "never"
)
```

### Updated Plan Types

```go
// Plan gains replanning tracking fields.
type Plan struct {
    Goal         string     `json:"goal"`
    Steps        []PlanStep `json:"steps"`
    ReplanCount  int        `json:"replan_count,omitempty"`  // how many times replanned
    MaxReplans   int        `json:"max_replans,omitempty"`   // from ReplanPolicy
}

// PlanStep gains a generation tracker.
type PlanStep struct {
    ID               string            `json:"id"`
    Description      string            `json:"description"`
    Agent            string            `json:"agent"`
    DependsOn        []string          `json:"depends_on"`
    DynamicSpec      *DynamicAgentSpec `json:"dynamic_spec,omitempty"`
    ReplanGeneration int               `json:"replan_generation,omitempty"` // 0=original, 1=first replan, etc.
}
```

### Configuration Additions (`configs/config.go`)

```go
type OrchestrateConfig struct {
    MaxConcurrency    int           `yaml:"max_concurrency"`
    MaxRecursionDepth int           `yaml:"max_recursion_depth"` // default 2
    SummaryPolicy     string        `yaml:"summary_policy"`      // auto/always/never
    Replan            ReplanConfig  `yaml:"replan"`
}

type ReplanConfig struct {
    TriggerOnFailure   bool `yaml:"trigger_on_failure"`
    TriggerOnDeviation bool `yaml:"trigger_on_deviation"` // reserved for future use
    MaxReplans         int  `yaml:"max_replans"`
}
```

## API Contracts

### No HTTP API Changes

The HTTP endpoints remain unchanged. The streaming event types shift from fixed phases (`explore`, `plan`, `dispatch`) to dynamic phases (`intent`, `execute`, `summarize`), which is a breaking change for clients parsing phase names. This is documented as a known breaking change.

### Internal API: Dispatcher Constructor

```go
// Updated constructor -- explorerAgent and plannerAgent become optional
func New(
    reg *registries.Registry,
    subAgents map[string]agent.Agent,
    explorerAgent agent.Agent,     // can be nil (exploration disabled)
    plannerAgent agent.Agent,      // can be nil (uses direct LLM classification)
    planGen agent.Agent,
    opts ...Option,
) *Dispatcher

// New options
func WithSummaryPolicy(p SummaryPolicy) Option
func WithReplanPolicy(p ReplanPolicy) Option
func WithMaxRecursionDepth(n int) Option
func WithIntentSystemPrompt(p string) Option // replaces WithPlannerSystemPrompt

// Deprecated: Use WithIntentSystemPrompt. Kept for backward compatibility.
func WithPlannerSystemPrompt(p string) Option
```

The constructor signature does not change. New behavior is additive via options.

**Backward compatibility note:** The deprecated `agents/compat.go` shim (`NewOrchestratorAgent`, `Create`) requires no changes. It does not pass the new options, so the Dispatcher uses default values: no replanning, auto summary policy, max recursion depth 2.

## File Changes Summary

| File | Action | Description |
|------|--------|-------------|
| `dispatches/types.go` | Modify | Add `IntentResult`, `IntentType`, `ReplanPolicy`, `SummaryPolicy` types; update `Plan` and `PlanStep` with replan fields |
| `dispatches/intent.go` | Create | Intent recognition logic + prompt template (replaces explore+classify pipeline) |
| `dispatches/depth.go` | Create | Context-based recursion depth propagation |
| `dispatches/summarize.go` | Create | Conditional summarization logic |
| `dispatches/execute.go` | Create | Unified execution logic with replanning support, including `topologicalLayers` |
| `dispatches/dispatch.go` | Modify | Rewrite `Run`/`RunStream` to use new decision loop; add new option functions; rename `plannerSystemPrompt` to `intentSystemPrompt` |
| `dispatches/stream.go` | Modify | Update phase event emission for dynamic phases; add `RunStream` to `hookedAgent` |
| `dispatches/classify.go` | Delete | Logic moves into `intent.go` |
| `dispatches/explore.go` | Delete | Logic moves into `intent.go` (on-demand invocation) |
| `dispatches/dag.go` | Modify | Extract reusable parts into `execute.go`; keep `buildNodes`, `buildDynamicAgent`, `findTerminalNodes`, `wrapWithHooks`, `hookedAgent` (with new `RunStream`), `runWithHooks` |
| `dispatches/input.go` | Modify | Minor: adapt `BuildInputMapper` for replan generation tracking |
| `dispatches/helpers.go` | Modify | Minor: keep utility functions, add intent JSON parsing |
| `configs/config.go` | Modify | Add `MaxRecursionDepth`, `SummaryPolicy`, `ReplanConfig` to `OrchestrateConfig` |
| `setup/setup.go` | Modify | Wire new options into Dispatcher constructor; set mode metadata on requests; build intent system prompt instead of planner system prompt |

## Implementation Plan

### Task 1: Add New Types and Policies
**Files:** `dispatches/types.go`, `configs/config.go`
**Description:** Add `IntentResult`, `IntentType`, `ReplanPolicy`, `SummaryPolicy` types. Add `ReplanGeneration` to `PlanStep`. Add `ReplanCount`/`MaxReplans` to `Plan`. Add new config fields to `OrchestrateConfig`. Keep existing `ClassifyResult` temporarily for backward compatibility during migration.
**Dependencies:** None

### Task 2: Implement Recursion Depth Control
**Files:** `dispatches/depth.go`, `dispatches/depth_test.go`
**Description:** Implement `WithDepth`, `DepthFrom`, `IncrementDepth` context helpers. Pure utility with no dependencies on other new code.
**Dependencies:** None

### Task 3: Implement Intent Recognition
**Files:** `dispatches/intent.go`, `dispatches/intent_test.go`
**Description:** Implement `recognizeIntent` and `recognizeIntentStream` using direct `aimodel.ChatCompletion` calls (consistent with existing `classifyDirect` approach). Build the intent recognition system prompt from the registry, co-located in the same file. The intent recognizer calls the explorer agent on-demand when the LLM indicates `needs_exploration: true`. Internally reuses the existing `explore`/`exploreStream` helper methods (moved from `explore.go`) and the LLM classification logic (adapted from `classify.go`). Include `IntentResult` validation similar to existing `ClassifyResult.validate`. All methods read depth from context via `DepthFrom(ctx)`.
**Dependencies:** Task 1, Task 2

### Task 4: Implement Conditional Summarization
**Files:** `dispatches/summarize.go`, `dispatches/summarize_test.go`
**Description:** Implement `shouldSummarize` and `summarize` methods. Reuse the existing `planGen` agent for generating summaries. Test all three policy modes. Document the mode metadata contract.
**Dependencies:** Task 1

### Task 5: Implement Execution with Replanning
**Files:** `dispatches/execute.go`, `dispatches/execute_test.go`
**Description:** Implement `executeTask`, `executeTaskStream`, `executePlanWithReplanning`, and `topologicalLayers`. The replanning logic partitions the DAG into topological layers and executes each layer using `orchestrate.ExecuteDAG`. After each layer, evaluate results. If a step failed and replanning is triggered, call the planner LLM with context about what succeeded and what failed, get replacement steps, rebuild the remaining layers, and continue. Track `replanCount` against `MaxReplans`. Only `TriggerOnFailure` is implemented; `TriggerOnDeviation` is deferred.

Move `runDirect`, `runPlan`, `buildNodes`, `buildDynamicAgent`, `findTerminalNodes`, `wrapWithHooks`, `hookedAgent`, `runWithHooks` from `dag.go` into `execute.go` (or keep in `dag.go` -- the key refactoring is wrapping them with replanning logic). Add `RunStream` to `hookedAgent` to enable streaming through hooked agents.
**Dependencies:** Task 1, Task 2

### Task 6: Rewrite Dispatcher Main Loop
**Files:** `dispatches/dispatch.go`
**Description:** Rewrite `Run` and `RunStream` to use the new decision loop: intent recognition -> execution -> optional summarization. Add new `Option` functions (`WithSummaryPolicy`, `WithReplanPolicy`, `WithMaxRecursionDepth`, `WithIntentSystemPrompt`). Deprecate `WithPlannerSystemPrompt` with a forwarding alias. Add new fields to `Dispatcher` struct. Update `RunStream` phase events to be dynamic with `TotalPhase: 0`. Implement the three streaming scenarios as documented in Section 7.
**Dependencies:** Task 3, Task 4, Task 5

### Task 7: Update Streaming Phase Events
**Files:** `dispatches/stream.go`
**Description:** Update `forwardSubAgentStream` and `streamPlan` to work with the new execution model. Update phase event names from `explore`/`plan`/`dispatch` to `intent`/`execute`/`summarize`. Use `TotalPhase: 0` for dynamic phase counts. Ensure `streamingDAGHandler` can be rebuilt after replanning with updated step metadata.
**Dependencies:** Task 6

### Task 8: Delete Obsolete Files and Clean Up
**Files:** `dispatches/classify.go` (delete), `dispatches/explore.go` (delete)
**Description:** Remove the old `classify.go` and `explore.go` files. Their logic has been absorbed into `intent.go` and `execute.go`. Update any remaining references. Run `make build` to verify.
**Dependencies:** Task 6

### Task 9: Wire Configuration and Setup
**Files:** `setup/setup.go`, `configs/config.go`
**Description:** Update `setup.New` to pass new options to the Dispatcher constructor. Read new config fields (`MaxRecursionDepth`, `SummaryPolicy`, `ReplanConfig`) from `OrchestrateConfig`. Apply defaults. Build the intent system prompt instead of the planner system prompt (rename `BuildPlannerSystemPrompt` call to `BuildIntentSystemPrompt` or equivalent). Ensure CLI and HTTP entry points set `req.Metadata["mode"]` to `"cli"` or `"http"` respectively for summary policy auto-detection.
**Dependencies:** Task 6

### Task 10: Update Unit Tests
**Files:** Various `*_test.go` files in `dispatches/`
**Description:** Update existing tests that test the old pipeline flow. Add unit tests for: intent recognition with various inputs, depth-based behavior at different levels, replanning trigger conditions (failure only -- deviation deferred), replan count limits, summary policy evaluation, topological layer partitioning, hooked agent streaming.
**Dependencies:** Task 8, Task 9

## Integration Test Plan

All integration tests live in `vv/integrations/` and require LLM API keys.

### Test 1: Simple Intent Fast Path
**Setup:** Send a greeting ("hello") to the Dispatcher.
**Verify:** The explorer is NOT invoked. The request reaches the chat agent with at most 1 LLM call for intent recognition. No planning phase occurs.
**Acceptance Criteria:** US-1 (simple greeting), US-2 (no explorer for simple), US-3 (no planner for simple)

### Test 2: Code Request with On-Demand Exploration
**Setup:** Send a coding request that references project files ("add a test for the Plan struct in types.go").
**Verify:** Intent recognition identifies `needs_exploration: true`. Explorer agent is invoked (full agent run with tools). After exploration, the coder agent is dispatched with context.
**Acceptance Criteria:** US-1 (coding request), US-2 (explorer on-demand)

### Test 3: Complex Task Planning
**Setup:** Send a complex multi-step request ("refactor the error handling in the dispatches package and add tests for each function").
**Verify:** Intent recognition produces a plan with multiple steps. The DAG executes the steps. Results are aggregated.
**Acceptance Criteria:** US-1 (complex request), US-3 (planner on-demand)

### Test 4: Replan on Step Failure
**Setup:** Configure `ReplanPolicy{TriggerOnFailure: true, MaxReplans: 2}`. Send a plan where one step is designed to fail (e.g., referencing a non-existent file).
**Verify:** When the step fails, replanning is triggered. New replacement steps are generated and executed. The final result reflects the replanned execution.
**Acceptance Criteria:** US-4 (replan on failure)

### Test 5: Replan Count Limit
**Setup:** Configure `ReplanPolicy{TriggerOnFailure: true, MaxReplans: 1}`. Send a plan where steps repeatedly fail.
**Verify:** After 1 replan, further failures abort execution. Partial results are returned with an error explanation.
**Acceptance Criteria:** US-4 (replan limit exceeded)

### Test 6: Recursion Depth Enforcement
**Setup:** Set `MaxRecursionDepth: 2`. Use a request that would normally trigger sub-agent decomposition.
**Verify:** At depth 0, full decision loop runs. At depth 1, planning works but no further decomposition. At depth 2+, direct execution only.
**Acceptance Criteria:** US-5 (depth control)

### Test 7: Summary Policy -- Auto Mode (CLI)
**Setup:** Set `SummaryPolicy: auto`. Send request with `Metadata["mode"] = "cli"`.
**Verify:** No summarization is generated.
**Acceptance Criteria:** US-6 (CLI skip)

### Test 8: Summary Policy -- Auto Mode (HTTP)
**Setup:** Set `SummaryPolicy: auto`. Send request with `Metadata["mode"] = "http"`.
**Verify:** Summarization is generated.
**Acceptance Criteria:** US-6 (HTTP summarize)

### Test 9: Summary Policy -- Always/Never
**Setup:** Test with `SummaryPolicy: always` and `SummaryPolicy: never`.
**Verify:** Summarization is always/never generated regardless of mode.
**Acceptance Criteria:** US-6 (force on/off)

### Test 10: Streaming Phase Events
**Setup:** Run a streaming request through the full decision loop.
**Verify:** Phase events use the new names (`intent`, `execute`, optionally `explore`, `summarize`). Phases appear only when they actually execute. `TotalPhase` is 0 (dynamic).
**Acceptance Criteria:** US-1 (unified loop), ORCH-14, ORCH-21
