# Technical Design: Dynamic Sub-Agent Support

## 1. Architecture Overview

This feature extends the orchestrator's plan execution to support dynamically configured sub-agents alongside the existing statically registered ones. The changes are confined to the `vv/agents` package -- no changes to the `vage` framework, CLI, HTTP interfaces, or tool registries.

### Current Flow

```
User Request -> Orchestrator -> Explorer (context) -> Planner (classify/plan)
  -> Direct dispatch to static sub-agent (coder/researcher/reviewer/chat)
  -> OR DAG plan execution with static sub-agents
```

### New Flow

```
User Request -> Orchestrator -> Explorer (context) -> Planner (classify/plan)
  -> Direct dispatch to static sub-agent (unchanged)
  -> OR DAG plan execution with static AND/OR dynamic sub-agents
       -> For steps with DynamicSpec: build ephemeral taskagent from spec, execute, discard
       -> For steps without DynamicSpec: use static sub-agent (unchanged)
```

### Key Design Decisions

1. **No new agent types** -- dynamic agents are standard `taskagent.Agent` instances created at step execution time via the existing `taskagent.New()` API.
2. **Tool access levels map to existing registries** -- the orchestrator already holds `coderReg` (full), `readOnlyReg` (read-only), and `reviewReg` (review). Dynamic agents pick one of these based on their base type or explicit tool access level.
3. **Planner prompt update** -- the planner system prompt is extended to optionally produce dynamic agent specifications in plan steps. The JSON schema is backward-compatible (new fields are optional).
4. **Validation in `ClassifyResult.validate()`** -- extended to validate dynamic specs.
5. **All new types co-located in `orchestrator.go`** -- following the existing pattern where `ClassifyResult`, `Plan`, `PlanStep`, and `PlanAggregator` are all in `orchestrator.go`.

## 2. Component Design

### 2.1 Modified Files

| File | Change |
|------|--------|
| `vv/agents/orchestrator.go` | Add `ToolAccessLevel`, `DynamicAgentSpec` types, extend `PlanStep`, update `validate()`, add `buildDynamicAgent()`, add `toolRegistries` field, update `buildNodes()` |
| `vv/agents/planner.go` | Update `PlannerSystemPrompt` to describe dynamic agent spec format |
| `vv/agents/agents.go` | Build and pass tool registries map to `NewOrchestratorAgent` |
| `vv/agents/orchestrator_test.go` | Add tests for dynamic spec validation, builder, and mixed plans |

### 2.2 No New Files

All new types are small and tightly coupled to the orchestrator. They are co-located in `orchestrator.go` following the existing pattern. Tests are added to `orchestrator_test.go` using the existing `stubAgent` and `mockChatCompleter` helpers.

## 3. Data Models

### 3.1 ToolAccessLevel and DynamicAgentSpec (New, in orchestrator.go)

```go
// ToolAccessLevel defines what tools a dynamic agent can access.
type ToolAccessLevel string

const (
    ToolAccessFull     ToolAccessLevel = "full"       // bash, read, write, edit, glob, grep
    ToolAccessReadOnly ToolAccessLevel = "read-only"  // read, glob, grep
    ToolAccessNone     ToolAccessLevel = "none"       // no tools (chat-only)
)

// DynamicAgentSpec defines the configuration for a dynamically created sub-agent.
type DynamicAgentSpec struct {
    BaseType     string          `json:"base_type"`               // required: coder, researcher, reviewer, chat
    SystemPrompt string          `json:"system_prompt,omitempty"` // optional: custom system prompt
    ToolAccess   ToolAccessLevel `json:"tool_access,omitempty"`   // optional: overrides base type default
    Model        string          `json:"model,omitempty"`         // optional: overrides configured model
}
```

**Base type to default tool registry mapping:**

| Base Type | Default Registry | Rationale |
|-----------|-----------------|-----------|
| coder | coderReg (full) | Needs all tools to write code |
| researcher | readOnlyReg | Read-only exploration |
| reviewer | reviewReg | Bash + read tools for running tests/linters, no write/edit |
| chat | nil (none) | No tools needed |

**ToolAccess override mapping:**

| ToolAccess | Registry |
|------------|----------|
| full | coderReg |
| read-only | readOnlyReg |
| none | nil |

When `ToolAccess` is explicitly set, it overrides the base type default. The three `ToolAccessLevel` values correspond to the three levels in the requirements. The reviewer's registry (bash + read, no write) is an internal implementation detail used as the default for `base_type: "reviewer"` but is not exposed as a standalone `ToolAccessLevel` -- this keeps the planner prompt simpler.

### 3.2 PlanStep (Updated)

```go
type PlanStep struct {
    ID          string           `json:"id"`
    Description string           `json:"description"`
    Agent       string           `json:"agent"`                   // static agent ID or base type for dynamic
    DependsOn   []string         `json:"depends_on"`
    DynamicSpec *DynamicAgentSpec `json:"dynamic_spec,omitempty"` // NEW: optional dynamic agent specification
}
```

**Backward compatibility:** When `DynamicSpec` is nil, behavior is identical to today -- the step dispatches to the static agent named by `Agent`. When `DynamicSpec` is present, it takes precedence over `Agent` (per ORCH-19).

**Consistency rule:** When `DynamicSpec` is present, `Agent` must equal `DynamicSpec.BaseType`. This keeps the JSON self-documenting and simplifies the planner prompt (the LLM sets `agent` to the base type in both cases).

### 3.3 ToolRegistries (New field on OrchestratorAgent)

The orchestrator needs access to tool registries to build dynamic agents:

```go
type OrchestratorAgent struct {
    // ... existing fields ...
    toolRegistries map[ToolAccessLevel]tool.ToolRegistry // NEW
    reviewReg      tool.ToolRegistry                     // NEW: reviewer registry (not exposed as ToolAccessLevel)
    maxIterations  int                                   // NEW: from config, for dynamic agents
}
```

The `toolRegistries` map is built in `agents.go` and passed as a single parameter:

```go
toolRegistries: map[ToolAccessLevel]tool.ToolRegistry{
    ToolAccessFull:     coderReg,
    ToolAccessReadOnly: readOnlyReg,
    // ToolAccessNone maps to nil (no registry) -- handled in code, not in map
}
```

The `reviewReg` is stored separately since it does not have a corresponding `ToolAccessLevel` value -- it is used only when `base_type` is "reviewer" and no explicit `ToolAccess` override is set.

## 4. Validation Rules

### 4.1 DynamicAgentSpec Validation

```go
func (s *DynamicAgentSpec) validate() error
```

Rules:
- `BaseType` is required and must be one of: `coder`, `researcher`, `reviewer`, `chat`.
- `ToolAccess`, if specified, must be one of: `full`, `read-only`, `none`.
- `SystemPrompt` and `Model` are optional with no additional constraints.

### 4.2 ClassifyResult Validation (Updated)

The existing `validate()` method on `ClassifyResult` is updated:

- For plan mode, each step is validated:
  - If `DynamicSpec` is present:
    - Validate the spec using `DynamicAgentSpec.validate()`.
    - Verify that `Agent` equals `DynamicSpec.BaseType` (consistency rule).
  - If `DynamicSpec` is nil: validate that `Agent` references a known static sub-agent (existing behavior).

## 5. Dynamic Agent Builder

```go
// buildDynamicAgent creates an ephemeral taskagent from a DynamicAgentSpec.
// Returns an error if the spec references unavailable registries or the
// orchestrator was not configured with tool registries.
func (o *OrchestratorAgent) buildDynamicAgent(stepID string, spec *DynamicAgentSpec) (*taskagent.Agent, error)
```

Implementation:
1. If `o.toolRegistries` is nil and the resolved tool access is not "none", return an error.
2. Determine tool access level: use `spec.ToolAccess` if set, otherwise derive from `spec.BaseType`.
3. Look up the tool registry:
   - If explicit `ToolAccess` is set: use `o.toolRegistries[toolAccess]`.
   - If base type is "reviewer" and no explicit `ToolAccess`: use `o.reviewReg`.
   - Otherwise: use the default from the base-type-to-registry mapping.
4. Determine the system prompt: use `spec.SystemPrompt` if set, otherwise use the default prompt for the base type (`CoderSystemPrompt`, `ResearcherSystemPrompt`, `ReviewerSystemPrompt`, `ChatSystemPrompt`).
5. Determine the model: use `spec.Model` if set, otherwise use `o.model`.
6. Create and return:

```go
return taskagent.New(
    agent.Config{
        ID:          fmt.Sprintf("dynamic_%s_%s", spec.BaseType, stepID),
        Name:        fmt.Sprintf("Dynamic %s Agent (%s)", spec.BaseType, stepID),
        Description: fmt.Sprintf("Dynamically created %s agent for step %s", spec.BaseType, stepID),
    },
    taskagent.WithChatCompleter(o.llm),
    taskagent.WithModel(model),
    taskagent.WithToolRegistry(registry),  // nil for ToolAccessNone
    taskagent.WithSystemPrompt(prompt.StringPrompt(systemPrompt)),
    taskagent.WithMaxIterations(o.maxIterations),
), nil
```

No memory manager is set -- dynamic agents are ephemeral per ORCH-18 and operate with working memory only.

## 6. Orchestrator Changes

### 6.1 buildNodes (Updated)

The `buildNodes` method is updated to check each step for a `DynamicSpec`:

```go
for _, step := range plan.Steps {
    stepCopy := step
    var runner agent.Agent

    if stepCopy.DynamicSpec != nil {
        // Build dynamic agent from spec
        dynAgent, err := o.buildDynamicAgent(stepCopy.ID, stepCopy.DynamicSpec)
        if err != nil {
            return nil, fmt.Errorf("orchestrator: build dynamic agent for step %q: %w", stepCopy.ID, err)
        }
        runner = dynAgent
    } else {
        // Existing static dispatch
        subAgent, ok := o.subAgents[step.Agent]
        if !ok {
            subAgent, ok = o.subAgents["coder"]
            if !ok {
                return nil, fmt.Errorf("orchestrator: no agent available for %q", step.Agent)
            }
        }
        runner = subAgent
    }

    nodes = append(nodes, orchestrate.Node{
        ID:     step.ID,
        Runner: runner,
        Deps:   step.DependsOn,
        InputMapper: func(...) { ... },  // same as existing
        Optional: true,
    })
}
```

### 6.2 NewOrchestratorAgent (Updated signature)

Add a single registries map parameter plus the reviewer registry and max iterations:

```go
func NewOrchestratorAgent(
    cfg agent.Config,
    llm aimodel.ChatCompleter,
    model string,
    subAgents map[string]agent.Agent,
    planGen *taskagent.Agent,
    maxConcurrency int,
    fallback agent.Agent,
    workingDir string,
    explorerAgent agent.Agent,
    plannerAgent agent.Agent,
    toolRegistries map[ToolAccessLevel]tool.ToolRegistry,  // NEW: may be nil for tests
    reviewReg tool.ToolRegistry,                            // NEW: reviewer registry
    maxIterations int,                                      // NEW: from config
) *OrchestratorAgent
```

When `toolRegistries` is nil, `buildDynamicAgent` returns an error and `buildNodes` falls back gracefully. This maintains full backward compatibility for existing tests -- they can pass nil for the new parameters.

### 6.3 agents.go Create (Updated)

Build the registries map and pass it to `NewOrchestratorAgent`:

```go
toolRegs := map[ToolAccessLevel]tool.ToolRegistry{
    ToolAccessFull:     coderReg,
    ToolAccessReadOnly: readOnlyReg,
}

orchestratorAgent := NewOrchestratorAgent(
    // ... existing params ...
    toolRegs,
    reviewReg,
    cfg.Agents.MaxIterations,
)
```

## 7. Planner Prompt Update

The `PlannerSystemPrompt` in `planner.go` is extended with a concise section appended after the existing rules:

```
7. For specialized sub-tasks, add an optional "dynamic_spec" to a plan step:
   {"id": "step_1", "description": "...", "agent": "coder", "depends_on": [],
    "dynamic_spec": {"base_type": "coder", "system_prompt": "You are a Go testing specialist...", "tool_access": "full"}}
   Fields: "base_type" (required, same as "agent"), "system_prompt" (optional), "tool_access" (optional: "full"/"read-only"/"none"), "model" (optional).
   Only use dynamic_spec when a sub-task needs a specialized prompt or different tool access. For most tasks, omit it.
```

This adds ~4 lines to the existing prompt, keeping token usage minimal.

## 8. Implementation Plan

### Task 1: Add ToolAccessLevel, DynamicAgentSpec, and validation
**File:** `vv/agents/orchestrator.go`
- Define `ToolAccessLevel` type and constants (`full`, `read-only`, `none`)
- Define `DynamicAgentSpec` struct
- Implement `DynamicAgentSpec.validate()` method
- Define `defaultBaseTypeRegistryKey` map (base type -> ToolAccessLevel or special reviewer key)
- Define `defaultSystemPrompts` map (base type -> system prompt constant)

### Task 2: Add tool registries and maxIterations to OrchestratorAgent
**File:** `vv/agents/orchestrator.go`
- Add `toolRegistries map[ToolAccessLevel]tool.ToolRegistry`, `reviewReg tool.ToolRegistry`, and `maxIterations int` fields
- Update `NewOrchestratorAgent` signature and constructor

### Task 3: Add dynamic agent builder
**File:** `vv/agents/orchestrator.go`
- Implement `buildDynamicAgent(stepID string, spec *DynamicAgentSpec) (*taskagent.Agent, error)` on `OrchestratorAgent`
- Uses `o.toolRegistries`, `o.reviewReg`, `o.llm`, `o.model`, `o.maxIterations`

### Task 4: Update PlanStep and ClassifyResult validation
**File:** `vv/agents/orchestrator.go`
- Add `DynamicSpec *DynamicAgentSpec` field to `PlanStep`
- Update `ClassifyResult.validate()` to handle steps with `DynamicSpec`
- Enforce `Agent == DynamicSpec.BaseType` consistency rule

### Task 5: Update buildNodes for dynamic dispatch
**File:** `vv/agents/orchestrator.go`
- Modify `buildNodes()` to check `step.DynamicSpec` and call `buildDynamicAgent()` when present
- Handle builder errors gracefully

### Task 6: Update agents.go Create function
**File:** `vv/agents/agents.go`
- Build `map[ToolAccessLevel]tool.ToolRegistry` from existing registries
- Pass map, `reviewReg`, and `cfg.Agents.MaxIterations` to `NewOrchestratorAgent`

### Task 7: Update planner system prompt
**File:** `vv/agents/planner.go`
- Append concise dynamic agent specification documentation to `PlannerSystemPrompt`

### Task 8: Update existing tests and add new tests
**File:** `vv/agents/orchestrator_test.go`
- Update `NewOrchestratorAgent` calls to include the new parameters (nil for registries in existing tests)
- Add `TestDynamicAgentSpec_Validate` with valid and invalid inputs
- Add `TestBuildDynamicAgent` for agent building with correct configuration
- Add tests for tool access level defaults per base type
- Add tests for plan execution with dynamic specs
- Add tests for mixed static/dynamic plans
- Add test for ORCH-19 precedence rule
- Add backward compatibility test for PlanStep JSON without dynamic_spec

## 9. Integration Test Plan

### Test 1: Static-only plan (regression)
- Submit a request that triggers a multi-step plan using only static agents (coder, researcher).
- Verify: Plan executes as before. No dynamic agents created.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 2: Dynamic-only plan
- Mock the planner to return a plan where all steps have `DynamicSpec`.
- Verify: Each step creates an ephemeral agent with the specified system prompt and tool access.
- Verify: Results are collected and aggregated correctly.
- Verify: Token usage includes dynamic agent usage.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 3: Mixed static and dynamic plan
- Mock the planner to return a plan with both static and dynamic steps, with dependencies between them.
- Verify: Static steps use registered sub-agents, dynamic steps use ephemeral agents.
- Verify: Dependencies work correctly (dynamic step depends on static step and vice versa).
- **Location:** `vv/agents/orchestrator_test.go`

### Test 4: Dynamic spec validation failures
- Test plan with missing `base_type` in dynamic spec.
- Test plan with invalid `tool_access` value.
- Test plan with mismatched `agent` and `base_type`.
- Verify: Validation fails with descriptive errors.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 5: Tool access level correctness
- Create dynamic agents with each tool access level (`full`, `read-only`, `none`).
- Verify: The correct tool registry (or nil) is assigned to each.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 6: Default fallback behavior
- Create a dynamic agent spec with only `base_type` set (no optional fields).
- Verify: Uses the default system prompt, default tool access, and default model for that base type.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 7: Custom system prompt
- Create a dynamic agent with a custom `system_prompt`.
- Verify: The ephemeral agent uses the custom prompt (not the base type default).
- **Location:** `vv/agents/orchestrator_test.go`

### Test 8: ORCH-19 precedence rule
- Create a plan step with `Agent: "coder"` and `DynamicSpec: {BaseType: "coder", ToolAccess: "read-only"}`.
- Verify: The dynamic spec takes precedence; the agent gets the read-only registries.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 9: Backward compatibility
- Verify that existing `ClassifyResult` JSON without `dynamic_spec` fields parses correctly.
- Verify that `PlanStep` without `DynamicSpec` dispatches to static agents as before.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 10: Nil tool registries (graceful degradation)
- Create an orchestrator with nil `toolRegistries` (simulating tests or misconfiguration).
- Attempt to build a plan with a dynamic spec.
- Verify: `buildDynamicAgent` returns an error, and `buildNodes` propagates it.
- **Location:** `vv/agents/orchestrator_test.go`
