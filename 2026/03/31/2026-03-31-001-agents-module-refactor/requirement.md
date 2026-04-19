# Requirement: vv/agents Module Refactor

## 1. Background and Objectives

### Background

The `vv/agents` package has accumulated significant technical debt as the product evolved. What began as a manageable set of agent definitions and an orchestrator has grown into a monolithic package where orchestration logic, agent registration, tool access control, DAG construction, streaming, and agent creation are all intertwined in a single directory with tightly coupled files.

Key pain points observed in the current codebase:

| Problem | Current State |
|---------|---------------|
| Orchestrator is oversized | `orchestrator.go` is 1246 lines, mixing request dispatch, DAG building, dynamic agent creation, streaming event forwarding, and result aggregation |
| Agent registration is scattered | Adding a new agent requires modifications to `agents.go` (factory struct + Create function), `planner.go` (hardcoded agent list in prompt), and `orchestrator.go` (validation maps: `validBaseTypes`, `defaultBaseTypeToolAccess`, `defaultBaseTypePrompt`) |
| Tool access control is coarse-grained | Only three levels (`full` / `read-only` / `none`), with `reviewer` bypassing the model entirely via a separate `reviewReg` parameter |
| InputMapper closures are untestable | DAG node input construction uses inline closures in `buildNodes()`, making unit testing impossible |
| Dynamic agents lack lifecycle management | Created and discarded with no hooks, no reuse, no observability |
| DAG errors silently skipped | `ErrorStrategy: Skip` is hardcoded; downstream steps cannot detect upstream failures |
| Configuration misplacement | `MaxConcurrency` lives inside `MemoryConfig` (with an existing TODO comment acknowledging this) |

### Objectives

1. **Decompose the monolithic `agents` package** into focused sub-packages with clear single responsibilities
2. **Centralize agent registration** into a registry pattern so adding a new agent is a single-file change
3. **Unify tool access control** under a capability-based `ToolProfile` model that eliminates special-case handling (reviewer's separate registry)
4. **Make input mapping testable** by extracting DAG input construction into pure functions with a dedicated `StepInput` struct
5. **Introduce agent lifecycle hooks** enabling logging, metrics, and rate limiting without modifying dispatch logic
6. **Create a dedicated assembly layer** that replaces the current `agents.Create()` factory with explicit registration-based wiring
7. **Maintain full behavioral compatibility** -- external callers (main.go, CLI, HTTP) should see no change in agent behavior

## 2. User Stories

### US-1: Decompose orchestrator into dispatch sub-package

**As a** developer maintaining the vv codebase,
**I want** the orchestrator logic split into focused files within a `dispatch/` sub-package,
**So that** I can understand, modify, and test each concern (routing, DAG building, input mapping, streaming) independently.

**Acceptance Criteria:**
- The `OrchestratorAgent` struct and its `Run`/`RunStream` methods move to `vv/dispatch/dispatch.go` as `Dispatcher`
- Request classification logic (`explore`, `classify/planTask`, `classifyTaskDirect`) moves to `vv/dispatch/dispatcher.go`
- DAG building (`buildNodes`, `buildDynamicAgent`, `findTerminalNodes`) and execution (`runPlan`) moves to `vv/dispatch/dag.go`
- Input mapping (`InputMapper` closures) is extracted to pure functions in `vv/dispatch/input.go` with a `StepInput` struct
- Streaming logic (`forwardSubAgentStream`, `streamPlan`, `streamingDAGHandler`, `exploreStream`, `planTaskStream`) moves to `vv/dispatch/stream.go`
- The `PlanAggregator`, `ClassifyResult`, `Plan`, `PlanStep`, `DynamicAgentSpec` types move to appropriate files within `dispatch/`
- Helper functions (`extractJSON`, `aggregateUsage`, `enrichRequest`) move to `dispatch/`
- No circular dependencies exist between sub-packages
- All existing tests pass or are migrated to the new locations

### US-2: Introduce agent registry

**As a** developer adding a new agent type,
**I want** a central `Registry` where agents self-register with their descriptor (ID, display name, description, tool profile, factory),
**So that** I only need to create one file and call one `Register()` function instead of editing three files.

**Acceptance Criteria:**
- A `vv/registry/` sub-package is created containing `registries.go` and `tool_access.go`
- `Registry` struct provides `Register()`, `Get()`, `All()`, `ValidateRef()` methods with thread-safe access
- `AgentDescriptor` holds `ID`, `DisplayName`, `Description`, `ToolProfile`, and `Factory` function
- `PlannerAgentList()` method auto-generates the agent list string for the planner prompt, eliminating the hardcoded list in `PlannerSystemPrompt`
- The `validBaseTypes`, `defaultBaseTypeToolAccess`, `defaultBaseTypePrompt` maps in `orchestrator.go` are eliminated -- these are derived from registered descriptors
- `ClassifyResult.validate()` uses the registry instead of a `map[string]agent.Agent`

### US-3: Unify tool access control with ToolProfile

**As a** developer configuring agent tool access,
**I want** a capability-based `ToolProfile` model that handles all agents uniformly,
**So that** the reviewer no longer needs a special `reviewReg` parameter and tool access is composable.

**Acceptance Criteria:**
- `ToolProfile` struct with a `Name` and `[]ToolCapability` is defined in `vv/registry/tool_access.go`
- Four capabilities: `read`, `write`, `execute` (bash), `search` (glob/grep)
- Four predefined profiles: `ProfileFull` (all), `ProfileReadOnly` (read + search), `ProfileReview` (read + search + execute), `ProfileNone` (empty)
- `ToolProfile.BuildRegistry()` method constructs a `tool.ToolRegistry` by filtering from a complete tool set based on capabilities
- The three separate tool registries created in `main.go` (`coderReg`, `readOnlyReg`, `reviewReg`) are replaced by `ToolProfile.BuildRegistry()` calls
- The separate `reviewReg` field in `OrchestratorAgent` is eliminated
- The `ToolAccessLevel` string type and its associated maps are replaced by `ToolProfile`

### US-4: Extract testable input mapping

**As a** developer writing unit tests for DAG step input construction,
**I want** input mapping extracted into pure functions with explicit parameters,
**So that** I can test message construction logic without needing a full orchestrator setup.

**Acceptance Criteria:**
- `StepInput` struct is defined in `vv/dispatch/input.go` with fields: `WorkingDir`, `ContextSummary`, `OriginalGoal`, `StepDescription`, `Upstream` (map of `StepResult`)
- `StepResult` struct has `Output` and `Status` (completed/failed/skipped)
- `StepInput.BuildMessages()` is a pure function returning `[]schema.Message`
- `StepInput.HasFailedDependency()` returns whether any upstream step failed
- `BuildInputMapper()` factory function returns an `orchestrate.InputMapper` from step parameters
- The inline closure in `buildNodes()` is replaced by `BuildInputMapper()` calls
- Unit tests exist for `BuildMessages()` and `HasFailedDependency()`

### US-5: Add agent lifecycle hooks

**As a** developer needing observability for agent executions,
**I want** a hook system with `OnBeforeRun` and `OnAfterRun` callbacks,
**So that** I can add logging, metrics collection, and rate limiting without modifying the dispatcher.

**Acceptance Criteria:**
- `Hook` interface defined in `vv/lifecycle/hooks.go` with `OnBeforeRun(ctx, agentID, req)` and `OnAfterRun(ctx, agentID, resp, err)` methods
- `LoggingHook`, `MetricsHook`, and `RateLimitHook` struct stubs provided
- `Chain()` function composes multiple hooks into a single `Hook`
- `Dispatcher` accepts a `[]lifecycle.Hook` and invokes them around sub-agent runs
- Hook errors in `OnBeforeRun` can abort the agent run; `OnAfterRun` errors are logged but do not fail the response

### US-6: Create setup assembly layer

**As a** developer tracing how the application is wired together,
**I want** a single `setup/` package that reads config, registers all agents, and constructs the dispatcher,
**So that** the wiring logic is in one place instead of scattered across `main.go` and `agents.Create()`.

**Acceptance Criteria:**
- `vv/setup/setup.go` contains a `New(cfg *config.Config) (*dispatch.Dispatcher, error)` function
- `New()` performs: create LLM client, create memory store, create registry, register all agents (via `agents.RegisterXxx()` calls), configure hooks, construct dispatcher
- The `Agents` struct in `agents.go` and the `Create()` function are removed
- `main.go` calls `setup.New()` instead of manually wiring agents
- Each agent file in `vv/agents/` exports a `RegisterXxx(registry, ...)` function instead of a private `newXxxAgent()` function

### US-7: Relocate MaxConcurrency configuration

**As a** developer reading the configuration,
**I want** `MaxConcurrency` in a semantically correct location,
**So that** it is not confusingly placed inside `MemoryConfig`.

**Acceptance Criteria:**
- A new `OrchestrateConfig` (or similar) struct is added to `config.go`
- `MaxConcurrency` moves from `MemoryConfig` to the new struct
- The existing TODO comment in `config.go` is resolved
- Default value of 2 is preserved
- Existing YAML configurations with `memory.max_concurrency` continue to work (migration note in comments) or the YAML key is moved to `orchestrate.max_concurrency`

## 3. Scope Boundaries

### In Scope

- Restructuring `vv/agents/` into sub-packages: `dispatch/`, `registry/`, `agents/`, `lifecycle/`, `setup/`
- Introducing the `Registry` and `AgentDescriptor` pattern
- Introducing the `ToolProfile` / `ToolCapability` model
- Extracting `StepInput` and pure input mapping functions
- Adding the `lifecycle.Hook` interface and basic implementations
- Creating the `setup/` assembly layer
- Moving `MaxConcurrency` out of `MemoryConfig`
- Migrating and maintaining all existing tests
- Updating `main.go` to use the new `setup.New()` entrypoint

### Out of Scope

- Changing agent behavior, prompts, or capabilities
- Changing the DAG `ErrorStrategy` from `Skip` to something else (noted for future work)
- Adding new agents or tools
- Modifying the `cli/`, `memory/`, `tools/`, or `config/` packages beyond what is needed for the refactor
- Changing the HTTP API or CLI interface
- Adding actual metrics collection or rate limiting logic (only hook interfaces/stubs)
- Modifying the upstream `vage` framework

## 4. Involved System Roles

This is an internal code refactoring. No user-facing roles are affected. The refactoring is relevant to:

- **Developer (maintainer)**: Primary beneficiary; reduced cognitive load, easier to add agents, better testability
- **CI/CD pipeline**: Must verify all existing tests continue to pass after restructuring

## 5. Involved Models and State Changes

No business models or state machines are affected. The refactoring introduces new code-level abstractions:

| New Abstraction | Location | Purpose |
|-----------------|----------|---------|
| `Registry` | `registry/registries.go` | Holds registered `AgentDescriptor` entries |
| `AgentDescriptor` | `registry/registries.go` | Metadata + factory for a single agent type |
| `ToolProfile` | `registry/tool_access.go` | Capability-based tool access definition |
| `ToolCapability` | `registry/tool_access.go` | Individual capability enum (read/write/execute/search) |
| `Dispatcher` | `dispatch/dispatch.go` | Renamed from `OrchestratorAgent`, same behavior |
| `StepInput` / `StepResult` | `dispatch/input.go` | Explicit struct for DAG input construction |
| `Hook` | `lifecycle/hooks.go` | Interface for agent execution lifecycle |
| `StreamHandler` | `dispatch/stream.go` | Replaces `streamingDAGHandler` |

## 6. Involved Business Processes and Rules

No business processes change. The orchestration flow remains identical:

1. **Explore**: Explorer agent gathers project context (unchanged)
2. **Classify**: Planner agent classifies as direct or plan mode (unchanged, but planner prompt is now auto-generated from registry)
3. **Route/Dispatch**: Direct mode routes to a single agent; plan mode builds DAG and executes (unchanged behavior, reorganized code)

The only process-adjacent change is that lifecycle hooks can now wrap agent executions, but this is additive and does not alter existing flows.

## 7. Involved Applications and Pages

None. This is a backend code restructuring with no UI impact.

## 8. Target Directory Structure

```
vv/
├── dispatch/              # Orchestration dispatch (from orchestrator.go)
│   ├── dispatch.go        # Dispatcher struct, Run/RunStream entry points
│   ├── dispatcher.go      # explore, classify, route logic
│   ├── dag.go             # Plan -> DAG node building + execution
│   ├── input.go           # StepInput/StepResult, pure BuildMessages()
│   └── stream.go          # Streaming event forwarding, StreamHandler
│
├── registry/              # Agent registration and discovery
│   ├── registries.go        # Registry, AgentDescriptor, AgentFactory
│   └── tool_access.go     # ToolProfile, ToolCapability, predefined profiles
│
├── agents/                # Agent definitions (prompt + registration only)
│   ├── coder.go           # RegisterCoder()
│   ├── researcher.go      # RegisterResearcher()
│   ├── reviewer.go        # RegisterReviewer()
│   ├── chat.go            # RegisterChat()
│   ├── explorer.go        # RegisterExplorer()
│   ├── planner.go         # RegisterPlanner()
│   └── memory_prompt.go   # PersistentMemoryPrompt (unchanged)
│
├── lifecycle/             # Agent execution hooks
│   └── hooks.go           # Hook interface, LoggingHook, Chain()
│
├── setup/                 # Application wiring
│   └── setup.go           # New() -> reads config, registers agents, returns Dispatcher
│
├── config/                # Unchanged (except MaxConcurrency relocation)
├── tools/                 # Unchanged
├── memory/                # Unchanged
├── cli/                   # Unchanged
├── integrations/          # Test updates as needed
└── main.go                # Simplified to call setup.New()
```

## 9. Dependency Graph

```
main.go
  └-> setup/
      ├-> config/
      ├-> registry/       <- no external dependencies (only vage/agent interfaces)
      ├-> agents/         -> registry/
      ├-> dispatch/       -> registry/, lifecycle/, vage/orchestrate
      ├-> lifecycle/      <- only vage/schema
      ├-> tools/
      └-> memory/
```

No circular dependencies. Each layer depends only on layers below it.

## 10. Migration Notes

- The refactoring must be done as a single atomic change to avoid broken intermediate states
- All files currently in `vv/agents/` will either be moved to new sub-packages or deleted
- `main.go` must be updated in the same change to use `setup.New()` instead of `agents.Create()`
- Integration tests under `vv/integrations/` must be reviewed and updated for new import paths
- The `Agents` struct (used in `main.go` for HTTP service agent registration) needs to be replaced -- the HTTP service should register agents from the registry or dispatcher
