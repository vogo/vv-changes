# Technical Design: vv/agents Module Refactor

## 1. Architecture Overview

### Current State

The `vv/agents/` package is a flat directory with 8 Go source files (plus tests) where orchestration, agent definitions, registration, and tool access control are intertwined:

```
vv/agents/
  agents.go          -- Agents struct + Create() factory (wiring)
  orchestrator.go    -- OrchestratorAgent: 1247 lines mixing dispatch, DAG, streaming, dynamic agents
  planner.go         -- Planner agent + hardcoded agent list in PlannerSystemPrompt
  coder.go           -- Coder agent definition
  researcher.go      -- Researcher agent definition
  reviewer.go        -- Reviewer agent definition
  chat.go            -- Chat agent definition
  explorer.go        -- Explorer agent definition
  memory_prompt.go   -- PersistentMemoryPrompt template
```

`main.go` manually creates three tool registries (`coderReg`, `readOnlyReg`, `reviewReg`), then calls `agents.Create()` which constructs all agents and the orchestrator.

### Target State

Five focused sub-packages under `vv/`, each with a single responsibility:

```
vv/
  registry/          -- Agent registration + tool access model
    registries.go
    tool_access.go
  dispatch/          -- Orchestration dispatch (extracted from orchestrator.go)
    dispatch.go      -- Dispatcher struct, Run/RunStream entry points
    classify.go      -- explore, classify, route logic
    dag.go           -- Plan -> DAG node building + execution
    input.go         -- StepInput/StepResult, pure BuildMessages()
    stream.go        -- Streaming event forwarding, streamingDAGHandler
    helpers.go       -- extractJSON, aggregateUsage, enrichRequest
    types.go         -- ClassifyResult, Plan, PlanStep, DynamicAgentSpec, PlanAggregator
  agents/            -- Agent definitions (prompt + registration only)
    coder.go
    researcher.go
    reviewer.go
    chat.go
    explorer.go
    planner.go
    memory_prompt.go
  lifecycle/         -- Agent execution hooks
    hooks.go
  setup/             -- Application wiring
    setup.go
```

### Design Principles

1. **No behavior change** -- external callers see identical agent behavior
2. **No vage framework modifications** -- work within existing `agent.Agent`, `orchestrate.Node`, `tool.ToolRegistry` interfaces
3. **Single atomic change** -- all files move together to avoid broken intermediate states
4. **Composition over inheritance** -- consistent with existing codebase conventions
5. **Functional options** -- consistent with existing `taskagent.WithXxx()` patterns

### Dependency Graph

```
main.go
  --> setup/
      --> config/          (unchanged)
      --> registry/        (depends on: vage/tool, vage/agent)
      --> agents/          (depends on: registry/, vage/agent, vage/memory)
      --> dispatch/        (depends on: registry/, lifecycle/, vage/orchestrate, vage/agent)
      --> lifecycle/       (depends on: vage/schema)
      --> tools/           (unchanged)
      --> memory/          (unchanged)
  --> cli/                 (unchanged, receives dispatch.Dispatcher)
  --> service/             (unchanged, receives agents from setup)
```

No circular dependencies. Each layer depends only on layers below it.

---

## 2. Component Design

### 2.1 registry/ -- Agent Registration and Tool Access

**File: `vv/registry/tool_access.go`**

Defines a capability-based tool access model that replaces the current `ToolAccessLevel` string type and the three separate tool registries in `main.go`.

```go
package registry

import "github.com/vogo/vage/tool"

// ToolCapability represents a single tool access capability.
type ToolCapability string

const (
    CapRead    ToolCapability = "read"    // read files
    CapWrite   ToolCapability = "write"   // write, edit files
    CapExecute ToolCapability = "execute" // bash commands
    CapSearch  ToolCapability = "search"  // glob, grep
)

// ToolProfile defines a named set of tool capabilities.
type ToolProfile struct {
    Name         string
    Capabilities []ToolCapability
}

// Predefined profiles matching current access patterns.
var (
    ProfileFull     = ToolProfile{"full", []ToolCapability{CapRead, CapWrite, CapExecute, CapSearch}}
    ProfileReadOnly = ToolProfile{"read-only", []ToolCapability{CapRead, CapSearch}}
    ProfileReview   = ToolProfile{"review", []ToolCapability{CapRead, CapSearch, CapExecute}}
    ProfileNone     = ToolProfile{"none", nil}
)

// Has returns true if the profile includes the given capability.
func (p ToolProfile) Has(cap ToolCapability) bool

// capabilityToolNames maps each capability to the tool names it grants access to.
// This encodes the existing tool-to-capability mapping from tools/tools.go.
var capabilityToolNames = map[ToolCapability][]string{
    CapRead:    {"file_read"},
    CapWrite:   {"file_write", "file_edit"},
    CapExecute: {"bash"},
    CapSearch:  {"glob", "grep"},
}

// BuildRegistry constructs a tool.ToolRegistry by filtering tools from
// a complete (full) registry based on the profile's capabilities.
// Returns a new registry containing only the allowed tools.
func (p ToolProfile) BuildRegistry(fullReg *tool.Registry) (*tool.Registry, error)
```

**Design decisions:**
- `BuildRegistry` takes a full `*tool.Registry` (the one with all 6 tools, including bash working dir config) and creates a filtered copy. This preserves the tool configuration (bash timeout, working dir) that is currently set during `tools.Register()`.
- The capability-to-tool mapping is hardcoded because the tool names are stable (`file_read`, `file_write`, `file_edit`, `bash`, `glob`, `grep`) and defined by the `vage` framework's built-in tools. This avoids over-engineering a registration system for tool capabilities.
- `ProfileNone` returns an empty registry (no tools), matching the current chat agent behavior.

**File: `vv/registry/registries.go`**

Central registry for agent descriptors. Replaces the `validBaseTypes`, `defaultBaseTypeToolAccess`, `defaultBaseTypePrompt` maps, and the hardcoded agent list in `PlannerSystemPrompt`.

```go
package registry

import (
    "fmt"
    "sort"
    "strings"
    "sync"

    "github.com/vogo/aimodel"
    "github.com/vogo/vage/agent"
    "github.com/vogo/vage/memory"
    "github.com/vogo/vage/tool"
)

// AgentDescriptor holds metadata and a factory function for a single agent type.
type AgentDescriptor struct {
    ID            string       // unique agent identifier (e.g., "coder")
    DisplayName   string       // human-readable name (e.g., "Coder")
    Description   string       // description for planner prompt auto-generation
    ToolProfile   ToolProfile  // capability-based tool access
    SystemPrompt  string       // default system prompt for dynamic agent creation
    Factory       AgentFactory // creates an agent instance
    Dispatchable  bool         // true if this agent can be a dispatch target (sub-agent)
}

// AgentFactory creates an agent.Agent from the given options.
type AgentFactory func(opts FactoryOptions) (agent.Agent, error)

// FactoryOptions holds the dependencies needed to create an agent.
type FactoryOptions struct {
    LLM            aimodel.ChatCompleter
    Model          string
    ToolRegistry   tool.ToolRegistry    // filtered by ToolProfile
    MaxIterations  int
    RunTokenBudget int
    Memory         *memory.Manager
}

// Registry is a thread-safe agent descriptor store.
type Registry struct {
    mu     sync.RWMutex
    agents map[string]AgentDescriptor
}

// New creates an empty Registry.
func New() *Registry

// Register adds an agent descriptor. Panics on duplicate ID (programming error).
func (r *Registry) Register(d AgentDescriptor)

// Get returns a descriptor by ID, or false if not found.
func (r *Registry) Get(id string) (AgentDescriptor, bool)

// All returns all registered descriptors, sorted by ID for deterministic output.
func (r *Registry) All() []AgentDescriptor

// Dispatchable returns only descriptors with Dispatchable=true, sorted by ID.
func (r *Registry) Dispatchable() []AgentDescriptor

// ValidateRef returns true if the given ID is registered.
func (r *Registry) ValidateRef(id string) bool

// PlannerAgentList generates the agent list section for the planner system prompt.
// Output format matches the existing hardcoded list:
//   - "coder": Full tool access, code modification and creation
//   - "researcher": Read-only access, explores codebases and gathers information
// Only includes Dispatchable agents.
func (r *Registry) PlannerAgentList() string
```

**Design decisions:**
- `Dispatchable` flag distinguishes sub-agents (coder, researcher, reviewer, chat) from infrastructure agents (explorer, planner). Only dispatchable agents appear in the planner prompt and can be dispatch targets.
- `SystemPrompt` is stored in the descriptor to replace `defaultBaseTypePrompt` map. This is used when creating dynamic agents.
- `Register` panics on duplicate ID because this is a programming error caught at startup, not a runtime condition.
- The `AgentFactory` function receives a `ToolRegistry` that has already been filtered by the setup layer using `ToolProfile.BuildRegistry()`. This keeps the factory simple.

### 2.2 dispatch/ -- Orchestration Dispatch

Extracted from the 1247-line `orchestrator.go`. The `Dispatcher` struct replaces `OrchestratorAgent` with identical behavior but cleaner internal organization.

**File: `vv/dispatch/dispatch.go`**

```go
package dispatch

import (
    "context"

    "github.com/vogo/aimodel"
    "github.com/vogo/vage/agent"
    "github.com/vogo/vage/agent/taskagent"
    "github.com/vogo/vage/schema"
    "github.com/vogo/vage/tool"
    "github.com/vogo/vv/hooks"
    "github.com/vogo/vv/registries"
)

// Dispatcher is the main orchestration agent. It receives user requests,
// explores project context, classifies the task, and dispatches to sub-agents.
// Behavioral equivalent of the former OrchestratorAgent.
type Dispatcher struct {
    agent.Base
    llm            aimodel.ChatCompleter
    model          string
    registry       *registries.Registry
    subAgents      map[string]agent.Agent // built from registry, keyed by descriptor ID
    planGen        *taskagent.Agent
    maxConcurrency int
    fallbackAgent  agent.Agent
    workingDir     string
    explorerAgent  agent.Agent
    plannerAgent   agent.Agent
    fullToolReg    *tool.Registry        // complete tool registry for dynamic agent creation
    hooks          []lifecycle.Hook
    maxIterations  int
    runTokenBudget int
}

// Options configures a Dispatcher.
type Options struct {
    LLM            aimodel.ChatCompleter
    Model          string
    Registry       *registries.Registry
    SubAgents      map[string]agent.Agent
    PlanGen        *taskagent.Agent
    MaxConcurrency int
    FallbackAgent  agent.Agent
    WorkingDir     string
    ExplorerAgent  agent.Agent
    PlannerAgent   agent.Agent
    FullToolReg    *tool.Registry
    Hooks          []lifecycle.Hook
    MaxIterations  int
    RunTokenBudget int
}

// New creates a Dispatcher from the given options.
func New(opts Options) *Dispatcher

// Run implements agent.Agent. Orchestrates: explore -> classify -> dispatch.
func (d *Dispatcher) Run(ctx context.Context, req *schema.RunRequest) (*schema.RunResponse, error)

// RunStream implements agent.StreamAgent. Same flow as Run but with streaming events.
func (d *Dispatcher) RunStream(ctx context.Context, req *schema.RunRequest) (*schema.RunStream, error)

// Compile-time interface checks.
var (
    _ agent.Agent       = (*Dispatcher)(nil)
    _ agent.StreamAgent = (*Dispatcher)(nil)
)
```

**File: `vv/dispatch/classify.go`**

Extracted from `orchestrator.go` methods: `explore`, `exploreStream`, `planTask`, `planTaskStream`, `classifyTaskDirect`.

```go
package dispatch

// explore calls the explorer sub-agent to build project context.
func (d *Dispatcher) explore(ctx context.Context, req *schema.RunRequest) (string, *aimodel.Usage)

// exploreStream calls the explorer using streaming, forwarding tool events.
func (d *Dispatcher) exploreStream(ctx context.Context, req *schema.RunRequest, send func(schema.Event) error) (string, *aimodel.Usage)

// classify calls the planner sub-agent to classify/plan the request.
func (d *Dispatcher) classify(ctx context.Context, req *schema.RunRequest, contextSummary string) (*ClassifyResult, *aimodel.Usage, error)

// classifyStream calls the planner using streaming.
func (d *Dispatcher) classifyStream(ctx context.Context, req *schema.RunRequest, contextSummary string, send func(schema.Event) error) (*ClassifyResult, *aimodel.Usage, error)

// classifyDirect makes a direct LLM call to classify (fallback when no planner agent).
func (d *Dispatcher) classifyDirect(ctx context.Context, req *schema.RunRequest) (*ClassifyResult, *aimodel.Usage, error)
```

**File: `vv/dispatch/types.go`**

All type definitions extracted from `orchestrator.go`:

```go
package dispatch

// ClassifyResult is the structured LLM response for task classification.
type ClassifyResult struct {
    Mode  string `json:"mode"`
    Agent string `json:"agent"`
    Plan  *Plan  `json:"plan"`
}

// validate checks that the classification result references valid agents.
// Uses the registry instead of a map[string]agent.Agent.
func (cr *ClassifyResult) validate(reg *registries.Registry) error

// Plan represents a parsed execution plan.
type Plan struct {
    Goal  string     `json:"goal"`
    Steps []PlanStep `json:"steps"`
}

// PlanStep represents a single step in the plan.
type PlanStep struct {
    ID          string            `json:"id"`
    Description string            `json:"description"`
    Agent       string            `json:"agent"`
    DependsOn   []string          `json:"depends_on"`
    DynamicSpec *DynamicAgentSpec `json:"dynamic_spec,omitempty"`
}

// DynamicAgentSpec defines configuration for a dynamically created sub-agent.
type DynamicAgentSpec struct {
    BaseType     string `json:"base_type"`
    SystemPrompt string `json:"system_prompt,omitempty"`
    ToolAccess   string `json:"tool_access,omitempty"`   // kept as string for JSON compat
    Model        string `json:"model,omitempty"`
}

// validate checks that a DynamicAgentSpec is well-formed.
// Uses registry to validate base_type and tool_access instead of hardcoded maps.
func (s *DynamicAgentSpec) validate(reg *registries.Registry) error

// PlanAggregator synthesizes sub-task outputs into a coherent response.
type PlanAggregator struct {
    Summarizer *taskagent.Agent
}

func (a *PlanAggregator) Aggregate(ctx context.Context, results map[string]*schema.RunResponse) (*schema.RunResponse, error)
```

**Design decision on `ClassifyResult.validate()`**: The current implementation validates against `subAgents map[string]agent.Agent`. The refactored version validates against `*registries.Registry`, checking `reg.ValidateRef(id)` for direct mode and `reg.Get(step.Agent)` for plan steps. For dynamic specs, `validate` checks that `BaseType` is a registered agent ID instead of using the `validBaseTypes` map.

**File: `vv/dispatch/dag.go`**

DAG building and execution, extracted from `orchestrator.go` methods: `buildNodes`, `buildDynamicAgent`, `runPlan`, `findTerminalNodes`.

```go
package dispatch

// runDirect dispatches to a single sub-agent.
func (d *Dispatcher) runDirect(ctx context.Context, req *schema.RunRequest, cr *ClassifyResult, usage *aimodel.Usage) (*schema.RunResponse, error)

// runPlan builds and executes a DAG from the plan.
func (d *Dispatcher) runPlan(ctx context.Context, req *schema.RunRequest, plan *Plan, usage *aimodel.Usage, contextSummary string) (*schema.RunResponse, error)

// buildNodes converts a Plan into orchestrate.Node slices for DAG execution.
func (d *Dispatcher) buildNodes(plan *Plan, req *schema.RunRequest, contextSummary string) ([]orchestrate.Node, error)

// buildDynamicAgent creates an ephemeral taskagent from a DynamicAgentSpec.
// Uses the registry to resolve tool profile and system prompt instead of hardcoded maps.
func (d *Dispatcher) buildDynamicAgent(stepID string, spec *DynamicAgentSpec) (*taskagent.Agent, error)

// findTerminalNodes returns IDs of nodes with no downstream dependents.
func findTerminalNodes(nodes []orchestrate.Node) []string
```

**Key change in `buildDynamicAgent`**: Instead of using `defaultBaseTypeToolAccess` and `defaultBaseTypePrompt` maps plus the special `reviewReg` field, the method now:
1. Looks up the `AgentDescriptor` from the registry by `spec.BaseType`
2. Gets the `ToolProfile` from the descriptor (or parses the `spec.ToolAccess` override)
3. Calls `profile.BuildRegistry(d.fullToolReg)` to create the filtered tool registry
4. Uses `descriptor.SystemPrompt` as the default prompt (or `spec.SystemPrompt` override)

This eliminates the `toolRegistries map[ToolAccessLevel]tool.ToolRegistry`, the `reviewReg tool.ToolRegistry` field, and the `validBaseTypes` / `defaultBaseTypeToolAccess` / `defaultBaseTypePrompt` maps.

**File: `vv/dispatch/input.go`**

Testable input mapping extracted from the inline `InputMapper` closure in `buildNodes`.

```go
package dispatch

import "github.com/vogo/vage/schema"

// StepInput holds all parameters needed to construct input messages for a DAG step.
type StepInput struct {
    WorkingDir      string
    ContextSummary  string
    OriginalGoal    string
    StepDescription string
    Upstream        map[string]StepResult
}

// StepResult holds the output of a completed upstream step.
type StepResult struct {
    Output string
    Status StepStatus
}

// StepStatus represents the completion status of a step.
type StepStatus int

const (
    StepCompleted StepStatus = iota
    StepFailed
    StepSkipped
)

// BuildMessages constructs the input messages for a DAG step.
// This is a pure function with no side effects, making it fully testable.
func (s *StepInput) BuildMessages() []schema.Message

// HasFailedDependency returns true if any upstream step has a failed status.
func (s *StepInput) HasFailedDependency() bool

// BuildInputMapper creates an orchestrate.InputMapFunc from step parameters.
// Replaces the inline closure in buildNodes.
func BuildInputMapper(workDir, contextSummary, goal string, step PlanStep, depIDs []string) orchestrate.InputMapFunc
```

**Design decision:** `BuildInputMapper` returns an `orchestrate.InputMapFunc` (which is `func(map[string]*schema.RunResponse) (*schema.RunRequest, error)`) to maintain compatibility with `orchestrate.Node.InputMapper`. Internally it constructs a `StepInput`, extracts upstream results from the `map[string]*schema.RunResponse`, and calls `BuildMessages()`.

**File: `vv/dispatch/stream.go`**

Streaming logic extracted from `orchestrator.go`: `forwardSubAgentStream`, `streamPlan`, `streamingDAGHandler`.

```go
package dispatch

// forwardSubAgentStream runs a sub-agent and forwards its stream events.
func (d *Dispatcher) forwardSubAgentStream(ctx context.Context, send func(schema.Event) error, subAgent agent.Agent, req *schema.RunRequest, agentName, stepID, sessionID string) error

// streamPlan executes a plan's steps with streaming sub-agent lifecycle events.
func (d *Dispatcher) streamPlan(ctx context.Context, send func(schema.Event) error, req *schema.RunRequest, plan *Plan, contextSummary, sessionID string) error

// streamingDAGHandler implements orchestrate.DAGEventHandler.
type streamingDAGHandler struct { /* same fields as current */ }
func (h *streamingDAGHandler) OnNodeStart(nodeID string)
func (h *streamingDAGHandler) OnNodeComplete(nodeID string, status orchestrate.NodeStatus, err error)
func (h *streamingDAGHandler) OnCheckpointError(_ string, _ error)
```

**File: `vv/dispatch/helpers.go`**

Utility functions extracted from `orchestrator.go`:

```go
package dispatch

// extractJSON extracts a JSON object from text that may contain markdown fences.
func extractJSON(text string) string

// aggregateUsage merges two usage structs.
func aggregateUsage(a, b *aimodel.Usage) *aimodel.Usage

// enrichRequest prepends working directory and context to a request.
func (d *Dispatcher) enrichRequest(req *schema.RunRequest, contextSummary string) *schema.RunRequest

// fallbackRun delegates to the fallback agent.
func (d *Dispatcher) fallbackRun(ctx context.Context, req *schema.RunRequest, usage *aimodel.Usage) (*schema.RunResponse, error)
```

### 2.3 agents/ -- Agent Definitions

Each agent file is refactored from a private `newXxxAgent()` function to a public `RegisterXxx()` function. The file retains the system prompt constant and registration logic only.

**Example: `vv/agents/coder.go`**

```go
package agents

import (
    "github.com/vogo/vage/agent"
    "github.com/vogo/vage/agent/taskagent"
    "github.com/vogo/vage/prompt"
    "github.com/vogo/vv/registries"
)

const CoderSystemPrompt = `...` // unchanged

// RegisterCoder registers the coder agent descriptor with the registries.
func RegisterCoder(reg *registries.Registry) {
    reg.Register(registries.AgentDescriptor{
        ID:           "coder",
        DisplayName:  "Coder",
        Description:  "Reads, writes, edits files, runs commands, searches codebases, debugs",
        ToolProfile:  registries.ProfileFull,
        SystemPrompt: CoderSystemPrompt,
        Dispatchable: true,
        Factory: func(opts registries.FactoryOptions) (agent.Agent, error) {
            var taskOpts []taskagent.Option
            taskOpts = append(taskOpts,
                taskagent.WithChatCompleter(opts.LLM),
                taskagent.WithModel(opts.Model),
                taskagent.WithSystemPrompt(prompt.StringPrompt(CoderSystemPrompt)),
                taskagent.WithMaxIterations(opts.MaxIterations),
                taskagent.WithRunTokenBudget(opts.RunTokenBudget),
            )
            if opts.ToolRegistry != nil {
                taskOpts = append(taskOpts, taskagent.WithToolRegistry(opts.ToolRegistry))
            }
            if opts.Memory != nil {
                taskOpts = append(taskOpts, taskagent.WithMemory(opts.Memory))
            }
            return taskagent.New(
                agent.Config{ID: "coder", Name: "Coder Agent", Description: "Performs coding tasks"},
                taskOpts...,
            ), nil
        },
    })
}
```

**Special cases:**
- **Coder with persistent memory prompt**: The `RegisterCoder` factory checks if a persistent memory store is available in `FactoryOptions` and wraps the system prompt with `PersistentMemoryPrompt` if so. `FactoryOptions` gains an optional `PersistentMemory memory.Memory` field for this.
- **Explorer**: Registered with `Dispatchable: false` (infrastructure agent, not a dispatch target).
- **Planner**: Registered with `Dispatchable: false`. The `PlannerSystemPrompt` is modified to use a placeholder `{{.AgentList}}` that the setup layer fills from `registries.PlannerAgentList()`.
- **Chat**: `MaxIterations` is hardcoded to 1 in the factory regardless of config, preserving current behavior.

**File: `vv/agents/memory_prompt.go`**

Unchanged. The `PersistentMemoryPrompt` struct and `NewPersistentMemoryPrompt` function stay in the `agents` package.

### 2.4 lifecycle/ -- Agent Execution Hooks

**File: `vv/lifecycle/hooks.go`**

```go
package lifecycle

import (
    "context"
    "log/slog"

    "github.com/vogo/vage/schema"
)

// Hook defines lifecycle callbacks for agent executions.
type Hook interface {
    // OnBeforeRun is called before an agent runs. Returning an error aborts the run.
    OnBeforeRun(ctx context.Context, agentID string, req *schema.RunRequest) error
    // OnAfterRun is called after an agent runs. Errors are logged but do not fail the response.
    OnAfterRun(ctx context.Context, agentID string, resp *schema.RunResponse, err error)
}

// Chain composes multiple hooks into a single Hook.
// OnBeforeRun calls are executed in order; if any returns an error, subsequent hooks are skipped.
// OnAfterRun calls are executed in reverse order.
func Chain(hooks ...Hook) Hook

// chainedHook implements Hook by invoking a slice of hooks.
type chainedHook struct {
    hooks []Hook
}

// LoggingHook logs agent start/end events.
type LoggingHook struct {
    Logger *slog.Logger
}

func (h *LoggingHook) OnBeforeRun(ctx context.Context, agentID string, req *schema.RunRequest) error
func (h *LoggingHook) OnAfterRun(ctx context.Context, agentID string, resp *schema.RunResponse, err error)

// MetricsHook is a stub for future metrics collection.
type MetricsHook struct{}

func (h *MetricsHook) OnBeforeRun(ctx context.Context, agentID string, req *schema.RunRequest) error
func (h *MetricsHook) OnAfterRun(ctx context.Context, agentID string, resp *schema.RunResponse, err error)

// RateLimitHook is a stub for future rate limiting.
type RateLimitHook struct{}

func (h *RateLimitHook) OnBeforeRun(ctx context.Context, agentID string, req *schema.RunRequest) error
func (h *RateLimitHook) OnAfterRun(ctx context.Context, agentID string, resp *schema.RunResponse, err error)
```

**Hook integration in Dispatcher**: The `Dispatcher.runDirect` and `forwardSubAgentStream` methods invoke hooks around sub-agent calls:

```go
func (d *Dispatcher) runWithHooks(ctx context.Context, agentID string, req *schema.RunRequest, fn func() (*schema.RunResponse, error)) (*schema.RunResponse, error) {
    for _, h := range d.hooks {
        if err := h.OnBeforeRun(ctx, agentID, req); err != nil {
            return nil, fmt.Errorf("hook aborted run for %q: %w", agentID, err)
        }
    }
    resp, err := fn()
    for i := len(d.hooks) - 1; i >= 0; i-- {
        d.hooks[i].OnAfterRun(ctx, agentID, resp, err)
    }
    return resp, err
}
```

### 2.5 setup/ -- Application Wiring

**File: `vv/setup/setup.go`**

Replaces the `Agents` struct and `Create()` function. This is the single assembly point.

```go
package setup

import (
    "fmt"

    "github.com/vogo/aimodel"
    "github.com/vogo/vage/agent"
    "github.com/vogo/vage/agent/taskagent"
    "github.com/vogo/vage/memory"
    "github.com/vogo/vage/prompt"
    "github.com/vogo/vage/tool"
    "github.com/vogo/vv/agents"
    "github.com/vogo/vv/configs"
    "github.com/vogo/vv/dispatches"
    "github.com/vogo/vv/hooks"
    "github.com/vogo/vv/registries"
)

// Result holds the assembled components for the application.
type Result struct {
    Dispatcher *dispatch.Dispatcher
    Registry   *registries.Registry
    SubAgents  map[string]agent.Agent  // for HTTP service registration
}

// New reads config, registers all agents, and constructs the Dispatcher.
func New(
    cfg *config.Config,
    llm aimodel.ChatCompleter,
    fullToolReg *tool.Registry,
    memMgr *memory.Manager,
    persistentMem memory.Memory,
) (*Result, error) {
    // 1. Create registry and register all agents.
    reg := registries.New()
    agents.RegisterCoder(reg)
    agents.RegisterResearcher(reg)
    agents.RegisterReviewer(reg)
    agents.RegisterChat(reg)
    agents.RegisterExplorer(reg)
    agents.RegisterPlanner(reg)

    // 2. Build sub-agents from registries.
    subAgents := make(map[string]agent.Agent)
    for _, desc := range reg.Dispatchable() {
        toolReg, err := desc.ToolProfile.BuildRegistry(fullToolReg)
        if err != nil {
            return nil, fmt.Errorf("build tool registry for %q: %w", desc.ID, err)
        }

        opts := registries.FactoryOptions{
            LLM:            llm,
            Model:          cfg.LLM.Model,
            ToolRegistry:   toolReg,
            MaxIterations:  cfg.Agents.MaxIterations,
            RunTokenBudget: cfg.Agents.RunTokenBudget,
            Memory:         memMgr,
        }

        a, err := desc.Factory(opts)
        if err != nil {
            return nil, fmt.Errorf("create agent %q: %w", desc.ID, err)
        }
        subAgents[desc.ID] = a
    }

    // 3. Build explorer and planner (non-dispatchable).
    explorerDesc, _ := reg.Get("explorer")
    explorerToolReg, _ := explorerDesc.ToolProfile.BuildRegistry(fullToolReg)
    explorer, _ := explorerDesc.Factory(registries.FactoryOptions{
        LLM: llm, Model: cfg.LLM.Model, ToolRegistry: explorerToolReg,
        MaxIterations: min(cfg.Agents.MaxIterations, 15),
    })

    plannerDesc, _ := reg.Get("planner")
    planner, _ := plannerDesc.Factory(registries.FactoryOptions{
        LLM: llm, Model: cfg.LLM.Model, MaxIterations: 1,
    })

    // 4. Build plan summarizer.
    planGen := taskagent.New(
        agent.Config{ID: "plan-gen", Name: "Plan Generator", Description: "Summarizes execution plan results"},
        taskagent.WithChatCompleter(llm),
        taskagent.WithModel(cfg.LLM.Model),
        taskagent.WithSystemPrompt(prompt.StringPrompt(dispatch.PlanSummaryPrompt)),
        taskagent.WithMaxIterations(1),
    )

    // 5. Apply persistent memory prompt to coder if available.
    // (The coder factory handles this via FactoryOptions.PersistentMemory)

    // 6. Wrap coder tool registry with CLI confirmation if needed.
    // This is done by the caller (main.go) before passing fullToolReg.

    // 7. Configure hooks.
    hooks := []lifecycle.Hook{
        &lifecycle.LoggingHook{Logger: slog.Default()},
    }

    // 8. Construct dispatcher.
    dispatcher := dispatch.New(dispatch.Options{
        LLM:            llm,
        Model:          cfg.LLM.Model,
        Registry:       reg,
        SubAgents:      subAgents,
        PlanGen:        planGen,
        MaxConcurrency: cfg.Orchestrate.MaxConcurrency,
        FallbackAgent:  subAgents["chat"],
        WorkingDir:     cfg.Tools.BashWorkingDir,
        ExplorerAgent:  explorer,
        PlannerAgent:   planner,
        FullToolReg:    fullToolReg,
        Hooks:          hooks,
        MaxIterations:  cfg.Agents.MaxIterations,
        RunTokenBudget: cfg.Agents.RunTokenBudget,
    })

    return &Result{
        Dispatcher: dispatcher,
        Registry:   reg,
        SubAgents:  subAgents,
    }, nil
}
```

**Design decision on `main.go` changes**: The `main.go` simplification:
- Creates a single `fullToolReg` (all 6 tools) using `tools.Register()` -- same as today
- Wraps it with CLI confirmation (`vvcli.WrapRegistry`) -- same as today
- Calls `setup.New()` instead of `agents.Create()`
- Uses `result.Dispatcher` for CLI and HTTP modes
- Uses `result.SubAgents` for HTTP service agent registration
- The three separate `RegisterReadOnly` / `RegisterReviewTools` calls in `main.go` are eliminated

### 2.6 config/ -- MaxConcurrency Relocation

**File: `vv/config/config.go` (modified)**

```go
// OrchestrateConfig holds orchestration configuration.
type OrchestrateConfig struct {
    MaxConcurrency int `yaml:"max_concurrency"` // DAG concurrency, default 2
}

type Config struct {
    // ... existing fields ...
    Orchestrate OrchestrateConfig `yaml:"orchestrate"`
    Memory      MemoryConfig      `yaml:"memory"`
}
```

Changes to `MemoryConfig`:
- Keep `MaxConcurrency` field with `yaml:"max_concurrency"` tag for backward compatibility
- In `applyDefaults()`, if `cfg.Memory.MaxConcurrency != 0 && cfg.Orchestrate.MaxConcurrency == 0`, copy the value (migration)
- Remove the TODO comment
- Add a deprecation comment on `MemoryConfig.MaxConcurrency`

```go
type MemoryConfig struct {
    Dir            string `yaml:"dir"`
    SessionWindow  int    `yaml:"session_window"`
    PersistentLoad bool   `yaml:"persistent_load"`
    MaxConcurrency int    `yaml:"max_concurrency"` // Deprecated: use orchestrate.max_concurrency
}

func applyDefaults(cfg *Config) {
    // ... existing defaults ...

    // Migrate MaxConcurrency from memory to orchestrate.
    if cfg.Memory.MaxConcurrency != 0 && cfg.Orchestrate.MaxConcurrency == 0 {
        cfg.Orchestrate.MaxConcurrency = cfg.Memory.MaxConcurrency
    }
    if cfg.Orchestrate.MaxConcurrency == 0 {
        cfg.Orchestrate.MaxConcurrency = 2
    }
}
```

---

## 3. Data Models / Schemas

### New Types

| Type | Package | Fields | Purpose |
|------|---------|--------|---------|
| `ToolCapability` | `registry` | `string` enum: `read`, `write`, `execute`, `search` | Individual tool access capability |
| `ToolProfile` | `registry` | `Name string`, `Capabilities []ToolCapability` | Named set of capabilities |
| `AgentDescriptor` | `registry` | `ID`, `DisplayName`, `Description`, `ToolProfile`, `SystemPrompt`, `Factory`, `Dispatchable` | Agent metadata + factory |
| `FactoryOptions` | `registry` | `LLM`, `Model`, `ToolRegistry`, `MaxIterations`, `RunTokenBudget`, `Memory`, `PersistentMemory` | Dependencies for agent creation |
| `Registry` | `registry` | `mu sync.RWMutex`, `agents map[string]AgentDescriptor` | Thread-safe agent store |
| `StepInput` | `dispatch` | `WorkingDir`, `ContextSummary`, `OriginalGoal`, `StepDescription`, `Upstream map[string]StepResult` | DAG step input construction |
| `StepResult` | `dispatch` | `Output string`, `Status StepStatus` | Upstream step result |
| `StepStatus` | `dispatch` | `int` enum: `StepCompleted`, `StepFailed`, `StepSkipped` | Step completion status |
| `Hook` | `lifecycle` | Interface: `OnBeforeRun`, `OnAfterRun` | Agent execution lifecycle |
| `OrchestrateConfig` | `config` | `MaxConcurrency int` | Orchestration settings |
| `Dispatcher` | `dispatch` | Replaces `OrchestratorAgent` | Main orchestration agent |

### Relocated Types

| Type | From | To |
|------|------|----|
| `ClassifyResult` | `agents` | `dispatch` |
| `Plan` | `agents` | `dispatch` |
| `PlanStep` | `agents` | `dispatch` |
| `DynamicAgentSpec` | `agents` | `dispatch` |
| `PlanAggregator` | `agents` | `dispatch` |
| `PlanSummaryPrompt` | `agents` | `dispatch` |

### Removed Types

| Type | Reason |
|------|--------|
| `ToolAccessLevel` | Replaced by `ToolProfile` |
| `Agents` struct | Replaced by `setup.Result` |
| `OrchestratorAgent` | Replaced by `dispatch.Dispatcher` |
| `validBaseTypes` map | Derived from registry |
| `defaultBaseTypeToolAccess` map | Derived from registry |
| `defaultBaseTypePrompt` map | Derived from registry |
| `validToolAccessLevels` map | Derived from registry |

---

## 4. API Contracts

### External API (No Change)

The HTTP API and CLI interface are unaffected. The `dispatch.Dispatcher` implements the same `agent.Agent` and `agent.StreamAgent` interfaces as the current `OrchestratorAgent`. The HTTP service registers the same agents with the same IDs.

### Internal API Changes

| Caller | Current | After |
|--------|---------|-------|
| `main.go` | `agents.Create(cfg, llm, coderReg, readOnlyReg, reviewReg, memMgr, prompt)` | `setup.New(cfg, llm, fullToolReg, memMgr, persistentMem)` |
| `main.go` | `allAgents.Orchestrator` | `result.Dispatcher` |
| `main.go` | `allAgents.Coder`, `allAgents.Chat`, etc. | `result.SubAgents["coder"]`, `result.SubAgents["chat"]`, etc. |
| `main.go` | `tools.RegisterReadOnly()`, `tools.RegisterReviewTools()` | Eliminated (handled by `ToolProfile.BuildRegistry()`) |
| `cli.New()` | Receives `*OrchestratorAgent` | Receives `*dispatch.Dispatcher` (both satisfy `agent.Agent`) |

---

## 5. Implementation Plan

### Phase 1: Foundation (no behavior change)

**Task 1.1: Create `registry/` package**
- Create `vv/registry/tool_access.go` with `ToolCapability`, `ToolProfile`, predefined profiles, `BuildRegistry()`
- Create `vv/registry/registries.go` with `Registry`, `AgentDescriptor`, `AgentFactory`, `FactoryOptions`
- Write unit tests: `registry/tool_access_test.go`, `registry/registry_test.go`

**Task 1.2: Create `lifecycle/` package**
- Create `vv/lifecycle/hooks.go` with `Hook` interface, `Chain()`, `LoggingHook`, `MetricsHook` stub, `RateLimitHook` stub
- Write unit tests: `lifecycle/hooks_test.go`

**Task 1.3: Create `dispatch/input.go`**
- Create `vv/dispatch/input.go` with `StepInput`, `StepResult`, `StepStatus`, `BuildMessages()`, `HasFailedDependency()`, `BuildInputMapper()`
- Write unit tests: `dispatch/input_test.go`

### Phase 2: Extract dispatch package

**Task 2.1: Create `dispatch/` package structure**
- Create `vv/dispatch/types.go` -- move `ClassifyResult`, `Plan`, `PlanStep`, `DynamicAgentSpec`, `PlanAggregator`, `PlanSummaryPrompt`
- Create `vv/dispatch/helpers.go` -- move `extractJSON`, `aggregateUsage`
- Create `vv/dispatch/dispatch.go` -- `Dispatcher` struct, `New()`, `Run()`, `RunStream()`
- Create `vv/dispatch/classify.go` -- move `explore`, `exploreStream`, `planTask`/`classify`, `planTaskStream`/`classifyStream`, `classifyTaskDirect`/`classifyDirect`
- Create `vv/dispatch/dag.go` -- move `buildNodes`, `buildDynamicAgent`, `runPlan`, `runDirect`, `findTerminalNodes`, `fallbackRun`, `enrichRequest`
- Create `vv/dispatch/stream.go` -- move `forwardSubAgentStream`, `streamPlan`, `streamingDAGHandler`

**Task 2.2: Update `buildNodes` to use `BuildInputMapper`**
- Replace the inline `InputMapper` closure with `BuildInputMapper()` call

**Task 2.3: Update `buildDynamicAgent` to use registry**
- Replace `defaultBaseTypeToolAccess`, `defaultBaseTypePrompt`, `validBaseTypes` lookups with registry lookups
- Replace `toolRegistries` map and `reviewReg` with `ToolProfile.BuildRegistry(d.fullToolReg)`

**Task 2.4: Update `ClassifyResult.validate` to use registry**
- Replace `subAgents map[string]agent.Agent` parameter with `*registries.Registry`

**Task 2.5: Migrate orchestrator tests**
- Move and adapt `orchestrator_test.go` to `dispatch/dispatch_test.go`
- Update imports and type references

### Phase 3: Refactor agent definitions

**Task 3.1: Convert agent files to registration pattern**
- Refactor `agents/coder.go`: `newCoderAgent()` -> `RegisterCoder(reg)`
- Refactor `agents/researcher.go`: `newResearcherAgent()` -> `RegisterResearcher(reg)`
- Refactor `agents/reviewer.go`: `newReviewerAgent()` -> `RegisterReviewer(reg)`
- Refactor `agents/chat.go`: `newChatAgent()` -> `RegisterChat(reg)`
- Refactor `agents/explorer.go`: `newExplorerAgent()` -> `RegisterExplorer(reg)`
- Refactor `agents/planner.go`: update `PlannerSystemPrompt` to use `{{.AgentList}}` placeholder; `newPlannerAgent()` -> `RegisterPlanner(reg)`
- Keep `agents/memory_prompt.go` unchanged

**Task 3.2: Update agent unit tests**
- Update `agents/agents_test.go`, `agents/coder_test.go`, etc. for new registration pattern

### Phase 4: Assembly and wiring

**Task 4.1: Create `setup/` package**
- Create `vv/setup/setup.go` with `New()` function
- Write unit tests: `setup/setup_test.go`

**Task 4.2: Relocate `MaxConcurrency`**
- Add `OrchestrateConfig` to `config/config.go`
- Add migration logic in `applyDefaults()`
- Update `config/config_test.go`

**Task 4.3: Update `main.go`**
- Remove `tools.RegisterReadOnly()` and `tools.RegisterReviewTools()` calls
- Replace `agents.Create()` with `setup.New()`
- Update HTTP service agent registration to use `result.SubAgents`
- Update CLI initialization to use `result.Dispatcher`

**Task 4.4: Delete old files**
- Delete `agents/agents.go` (the `Agents` struct and `Create()` function)
- Delete `agents/orchestrator.go` (fully replaced by `dispatch/`)
- Clean up any remaining dead code

### Phase 5: Integration validation

**Task 5.1: Update integration tests**
- Update `integrations/agents_test.go` to use `setup.New()` instead of `agents.Create()`
- Update `integrations/wiring_test.go` for new import paths
- Ensure all other integration tests pass

**Task 5.2: Final verification**
- Run `make build` (license-check, format, lint, test)
- Verify no circular dependencies: `go vet ./...`
- Verify all 6 tools are still present for each profile

---

## 6. Integration Test Plan

### Unit Tests (new)

| Test | Package | Validates |
|------|---------|-----------|
| `TestToolProfile_BuildRegistry_Full` | `registry` | Full profile returns all 6 tools |
| `TestToolProfile_BuildRegistry_ReadOnly` | `registry` | ReadOnly profile returns read, glob, grep (3 tools) |
| `TestToolProfile_BuildRegistry_Review` | `registry` | Review profile returns bash, read, glob, grep (4 tools) |
| `TestToolProfile_BuildRegistry_None` | `registry` | None profile returns empty registry |
| `TestToolProfile_Has` | `registry` | Capability membership check |
| `TestRegistry_Register_Get` | `registry` | Register and retrieve a descriptor |
| `TestRegistry_Register_Duplicate_Panics` | `registry` | Duplicate ID panics |
| `TestRegistry_All_Sorted` | `registry` | All() returns sorted descriptors |
| `TestRegistry_Dispatchable` | `registry` | Only dispatchable agents returned |
| `TestRegistry_PlannerAgentList` | `registry` | Auto-generated planner prompt matches expected format |
| `TestRegistry_ValidateRef` | `registry` | Valid/invalid ID check |
| `TestStepInput_BuildMessages` | `dispatch` | Message construction with all fields |
| `TestStepInput_BuildMessages_Minimal` | `dispatch` | Message construction with minimal fields |
| `TestStepInput_HasFailedDependency` | `dispatch` | Returns true when upstream failed |
| `TestStepInput_HasFailedDependency_AllCompleted` | `dispatch` | Returns false when all completed |
| `TestBuildInputMapper` | `dispatch` | Returns valid InputMapFunc |
| `TestHook_Chain_BeforeRun` | `lifecycle` | Hooks called in order |
| `TestHook_Chain_BeforeRun_Abort` | `lifecycle` | First error aborts chain |
| `TestHook_Chain_AfterRun_ReverseOrder` | `lifecycle` | After hooks called in reverse |
| `TestLoggingHook` | `lifecycle` | No panics, correct log output |

### Migrated Tests (from existing)

| Current Test | New Location | Changes |
|-------------|--------------|---------|
| `TestOrchestratorAgent_ImplementsInterfaces` | `dispatch/dispatch_test.go` | Assert `Dispatcher` implements `agent.Agent` + `agent.StreamAgent` |
| `TestClassifyResult_Validate_Direct` | `dispatch/types_test.go` | Use registry instead of `map[string]agent.Agent` |
| `TestClassifyResult_Validate_Plan` | `dispatch/types_test.go` | Use registry |
| `TestExtractJSON_*` | `dispatch/helpers_test.go` | No changes |
| `TestDynamicAgentSpec_Validate` | `dispatch/types_test.go` | Use registry |
| `TestCreate_AllAgents` | `setup/setup_test.go` | Use `setup.New()` and verify all agents created |
| `TestCreate_Names` | `setup/setup_test.go` | Verify agent names via registry |

### Integration Tests (updated)

| Test | Changes |
|------|---------|
| `TestIntegration_Agents_CoderHasTools` | Use `setup.New()` instead of `agents.Create()`; verify coder has 6 tools via `ToolProfile.BuildRegistry()` |
| `TestIntegration_FullWiring` | Use `setup.New()` and `result.Dispatcher`; verify HTTP service still serves 5 agents + 6 tools |
| `TestIntegration_Agents_Streaming*` | Verify streaming still works with `Dispatcher` |
| `TestIntegration_Agents_PlanExecution*` | Verify DAG plan execution unchanged |

### Behavioral Regression Checks

1. **Agent IDs unchanged**: orchestrator -> "orchestrator" (via `Dispatcher`), coder -> "coder", etc.
2. **Tool counts preserved**: coder=6, researcher=3, reviewer=4, chat=0, explorer=3
3. **Streaming events preserved**: Same event types emitted in same order (PhaseStart/End, SubAgentStart/End, TextDelta)
4. **Planner prompt compatibility**: Auto-generated agent list produces the same agent descriptions as the hardcoded `PlannerSystemPrompt`
5. **Dynamic agent creation**: `buildDynamicAgent` produces agents with same tool access and prompts via registry lookup
6. **Config backward compatibility**: YAML with `memory.max_concurrency` still works; new `orchestrate.max_concurrency` also works
7. **Fallback behavior preserved**: Planning failures fall back to chat agent
