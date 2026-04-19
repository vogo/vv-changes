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
4. **Validation remains in `ClassifyResult.validate()`** -- extended to validate dynamic specs.

## 2. Component Design

### 2.1 Modified Files

| File | Change |
|------|--------|
| `vv/agents/orchestrator.go` | Add `DynamicAgentSpec` type, extend `PlanStep`, update `validate()`, update `buildNodes()` to build dynamic agents, add tool registry references to `OrchestratorAgent` |
| `vv/agents/planner.go` | Update `PlannerSystemPrompt` to describe dynamic agent spec format |
| `vv/agents/agents.go` | Pass tool registries to `NewOrchestratorAgent` |

### 2.2 New File

| File | Purpose |
|------|---------|
| `vv/agents/dynamic_agent.go` | `DynamicAgentSpec` type, validation, and builder function |
| `vv/agents/dynamic_agent_test.go` | Unit tests for spec validation and agent building |

## 3. Data Models

### 3.1 DynamicAgentSpec (New)

```go
// ToolAccessLevel defines what tools a dynamic agent can access.
type ToolAccessLevel string

const (
    ToolAccessFull     ToolAccessLevel = "full"      // bash, read, write, edit, glob, grep
    ToolAccessReadOnly ToolAccessLevel = "read-only"  // read, glob, grep
    ToolAccessReview   ToolAccessLevel = "review"     // bash, read, glob, grep (no write/edit)
    ToolAccessNone     ToolAccessLevel = "none"        // no tools (chat-only)
)

// DynamicAgentSpec defines the configuration for a dynamically created sub-agent.
type DynamicAgentSpec struct {
    BaseType     string          `json:"base_type"`               // required: coder, researcher, reviewer, chat
    SystemPrompt string          `json:"system_prompt,omitempty"`  // optional: custom system prompt
    ToolAccess   ToolAccessLevel `json:"tool_access,omitempty"`    // optional: overrides base type default
    Model        string          `json:"model,omitempty"`          // optional: overrides configured model
}
```

**Base type to default tool access mapping:**

| Base Type | Default ToolAccess |
|-----------|--------------------|
| coder | full |
| researcher | read-only |
| reviewer | review |
| chat | none |

### 3.2 PlanStep (Updated)

```go
type PlanStep struct {
    ID          string           `json:"id"`
    Description string           `json:"description"`
    Agent       string           `json:"agent"`                    // static agent ID (used when DynamicSpec is nil)
    DependsOn   []string         `json:"depends_on"`
    DynamicSpec *DynamicAgentSpec `json:"dynamic_spec,omitempty"`  // NEW: optional dynamic agent specification
}
```

**Backward compatibility:** When `DynamicSpec` is nil, behavior is identical to today -- the step dispatches to the static agent named by `Agent`. When `DynamicSpec` is present, it takes precedence over `Agent` (per ORCH-19).

### 3.3 ToolRegistries (New field on OrchestratorAgent)

The orchestrator needs access to all three tool registries to build dynamic agents:

```go
type OrchestratorAgent struct {
    // ... existing fields ...
    toolRegistries map[ToolAccessLevel]tool.ToolRegistry  // NEW
}
```

This map is populated from the existing registries passed during construction:

```go
toolRegistries: map[ToolAccessLevel]tool.ToolRegistry{
    ToolAccessFull:     coderReg,
    ToolAccessReadOnly: readOnlyReg,
    ToolAccessReview:   reviewReg,
    // ToolAccessNone maps to nil (no registry)
}
```

## 4. Validation Rules

### 4.1 DynamicAgentSpec Validation

```go
func (s *DynamicAgentSpec) validate() error
```

Rules:
- `BaseType` is required and must be one of: `coder`, `researcher`, `reviewer`, `chat`.
- `ToolAccess`, if specified, must be one of: `full`, `read-only`, `review`, `none`.
- `SystemPrompt` and `Model` are optional with no additional constraints.

### 4.2 ClassifyResult Validation (Updated)

The existing `validate()` method on `ClassifyResult` is updated:

- For plan mode, each step is validated:
  - If `DynamicSpec` is present: validate the spec using `DynamicAgentSpec.validate()`. The `Agent` field is ignored but should still be set (it serves as documentation).
  - If `DynamicSpec` is nil: validate that `Agent` references a known static sub-agent (existing behavior).

This means a plan step with a valid `DynamicSpec` will NOT fail validation even if `Agent` references an unknown static agent, since the dynamic spec takes precedence.

## 5. Dynamic Agent Builder

```go
// buildDynamicAgent creates an ephemeral taskagent from a DynamicAgentSpec.
func (o *OrchestratorAgent) buildDynamicAgent(stepID string, spec *DynamicAgentSpec) agent.Agent
```

Implementation:
1. Determine tool access level: use `spec.ToolAccess` if set, otherwise derive from `spec.BaseType`.
2. Look up the tool registry from `o.toolRegistries[toolAccess]`.
3. Determine the system prompt: use `spec.SystemPrompt` if set, otherwise use the default prompt for the base type (the existing constants: `CoderSystemPrompt`, `ResearcherSystemPrompt`, `ReviewerSystemPrompt`, `ChatSystemPrompt`).
4. Determine the model: use `spec.Model` if set, otherwise use `o.model`.
5. Create and return:

```go
return taskagent.New(
    agent.Config{
        ID:          fmt.Sprintf("dynamic-%s-%s", spec.BaseType, stepID),
        Name:        fmt.Sprintf("Dynamic %s Agent (%s)", spec.BaseType, stepID),
        Description: fmt.Sprintf("Dynamically created %s agent for step %s", spec.BaseType, stepID),
    },
    taskagent.WithChatCompleter(o.llm),
    taskagent.WithModel(model),
    taskagent.WithToolRegistry(registry),  // nil for ToolAccessNone
    taskagent.WithSystemPrompt(prompt.StringPrompt(systemPrompt)),
    taskagent.WithMaxIterations(10),  // sensible default for ephemeral agents
)
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
        runner = o.buildDynamicAgent(stepCopy.ID, stepCopy.DynamicSpec)
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

Add three tool registry parameters:

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
    coderReg tool.ToolRegistry,       // NEW
    readOnlyReg tool.ToolRegistry,    // NEW
    reviewReg tool.ToolRegistry,      // NEW
) *OrchestratorAgent
```

### 6.3 agents.go Create (Updated)

Pass the registries to `NewOrchestratorAgent`:

```go
orchestratorAgent := NewOrchestratorAgent(
    // ... existing params ...
    coderReg,
    readOnlyReg,
    reviewReg,
)
```

## 7. Planner Prompt Update

The `PlannerSystemPrompt` in `planner.go` is extended with an additional section describing the dynamic agent specification format:

```
## Dynamic Agent Specification (Optional)

For complex or specialized sub-tasks, you can optionally specify a dynamic agent configuration
instead of using a pre-defined agent. Add a "dynamic_spec" object to a plan step:

{"mode": "plan", "plan": {"goal": "...", "steps": [
  {"id": "step_1", "description": "...", "agent": "coder", "depends_on": [],
   "dynamic_spec": {
     "base_type": "coder",
     "system_prompt": "You are a Go testing specialist. Focus on writing table-driven tests...",
     "tool_access": "full"
   }}
]}}

### dynamic_spec fields:
- "base_type" (required): One of "coder", "researcher", "reviewer", "chat". Determines default tools and behavior.
- "system_prompt" (optional): A custom system prompt tailored to this specific sub-task. If omitted, the default prompt for the base_type is used.
- "tool_access" (optional): Override tool access level. One of "full" (all tools), "read-only" (read/glob/grep), "review" (read/glob/grep/bash), "none" (no tools). If omitted, defaults based on base_type.
- "model" (optional): Override the LLM model for this step. If omitted, uses the configured default.

### When to use dynamic_spec:
- When a sub-task needs a specialized system prompt for better results
- When a sub-task needs different tool access than its base type
- For most tasks, simply specifying "agent" without dynamic_spec is sufficient
```

## 8. Implementation Plan

### Task 1: Add DynamicAgentSpec type and validation
**File:** `vv/agents/dynamic_agent.go`
- Define `ToolAccessLevel` type and constants
- Define `DynamicAgentSpec` struct
- Implement `DynamicAgentSpec.validate()` method
- Define `defaultToolAccess` mapping from base type to tool access level
- Define `defaultSystemPrompts` mapping from base type to system prompt constant

### Task 2: Add dynamic agent builder
**File:** `vv/agents/dynamic_agent.go`
- Implement `buildDynamicAgent(stepID string, spec *DynamicAgentSpec) agent.Agent` on `OrchestratorAgent`
- Uses `o.toolRegistries`, `o.llm`, `o.model` to construct the ephemeral `taskagent.Agent`

### Task 3: Update PlanStep and ClassifyResult validation
**File:** `vv/agents/orchestrator.go`
- Add `DynamicSpec *DynamicAgentSpec` field to `PlanStep`
- Update `ClassifyResult.validate()` to handle steps with `DynamicSpec`

### Task 4: Add tool registries to OrchestratorAgent
**File:** `vv/agents/orchestrator.go`
- Add `toolRegistries map[ToolAccessLevel]tool.ToolRegistry` field
- Update `NewOrchestratorAgent` signature to accept `coderReg`, `readOnlyReg`, `reviewReg`
- Build the `toolRegistries` map in the constructor

### Task 5: Update buildNodes for dynamic dispatch
**File:** `vv/agents/orchestrator.go`
- Modify `buildNodes()` to check `step.DynamicSpec` and call `buildDynamicAgent()` when present

### Task 6: Update agents.go Create function
**File:** `vv/agents/agents.go`
- Pass `coderReg`, `readOnlyReg`, `reviewReg` to `NewOrchestratorAgent`

### Task 7: Update planner system prompt
**File:** `vv/agents/planner.go`
- Extend `PlannerSystemPrompt` with dynamic agent specification documentation

### Task 8: Unit tests for dynamic agent spec
**File:** `vv/agents/dynamic_agent_test.go`
- Test `DynamicAgentSpec.validate()` with valid and invalid inputs
- Test `buildDynamicAgent()` creates agents with correct configuration
- Test tool access level defaults per base type

### Task 9: Update existing orchestrator tests
**File:** `vv/agents/orchestrator_test.go`
- Update `NewOrchestratorAgent` calls to include the new registry parameters
- Add tests for plan execution with dynamic specs
- Add tests for mixed static/dynamic plans
- Add test for dynamic spec taking precedence over agent field (ORCH-19)

### Task 10: Update agents_test.go
**File:** `vv/agents/agents_test.go`
- Verify `Create()` still works correctly (it should, since internal wiring only)

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
- Verify: Validation fails with descriptive errors.
- **Location:** `vv/agents/dynamic_agent_test.go`

### Test 5: Tool access level correctness
- Create dynamic agents with each tool access level (`full`, `read-only`, `review`, `none`).
- Verify: The correct tool registry (or nil) is assigned to each.
- **Location:** `vv/agents/dynamic_agent_test.go`

### Test 6: Default fallback behavior
- Create a dynamic agent spec with only `base_type` set (no optional fields).
- Verify: Uses the default system prompt, default tool access, and default model for that base type.
- **Location:** `vv/agents/dynamic_agent_test.go`

### Test 7: Custom system prompt
- Create a dynamic agent with a custom `system_prompt`.
- Verify: The ephemeral agent uses the custom prompt (not the base type default).
- **Location:** `vv/agents/dynamic_agent_test.go`

### Test 8: ORCH-19 precedence rule
- Create a plan step with `Agent: "chat"` and `DynamicSpec: {BaseType: "coder", ...}`.
- Verify: The dynamic spec takes precedence; the agent behaves as a coder, not chat.
- **Location:** `vv/agents/orchestrator_test.go`

### Test 9: Backward compatibility
- Verify that existing `ClassifyResult` JSON without `dynamic_spec` fields parses correctly.
- Verify that `PlanStep` without `DynamicSpec` dispatches to static agents as before.
- **Location:** `vv/agents/orchestrator_test.go`
