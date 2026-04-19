# Design: Project Instructions File (VV.md)

## Architecture Overview

This feature adds project-level instruction loading from a `VV.md` file in the working directory, injecting its content into all agent system prompts. The design touches four layers of the `vv` module:

1. **Config** -- new `ProjectInstructions` field on `configs.Config`
2. **Setup** -- file loading in `setup.Init()` after working directory capture
3. **Registry** -- `ProjectInstructions` field added to `registries.FactoryOptions`
4. **Agents** -- each factory function incorporates project instructions into its system prompt via a shared helper

The approach is deliberately minimal: a single string flows from file to config to factory options to system prompt. No new packages, no new dependencies, no new interfaces.

## Component Design

### 1. Config Model Extension

**File**: `vv/configs/config.go`

Add a `ProjectInstructions` field to `Config`. This field is not YAML-serializable (it is runtime-only, populated from the `VV.md` file, not from `vv.yaml`).

```go
type Config struct {
    LLM         LLMConfig         `yaml:"llm"`
    Server      ServerConfig      `yaml:"server"`
    Tools       ToolsConfig       `yaml:"tools"`
    Agents      AgentsConfig      `yaml:"agents"`
    Mode        string            `yaml:"mode"`
    CLI         CLIConfig         `yaml:"cli"`
    Memory      MemoryConfig      `yaml:"memory"`
    Orchestrate OrchestrateConfig `yaml:"orchestrate"`

    // ProjectInstructions holds content loaded from VV.md in the working directory.
    // Runtime-only; not persisted to vv.yaml.
    ProjectInstructions string `yaml:"-"`
}
```

The `yaml:"-"` tag ensures this field is never read from or written to `vv.yaml`.

### 2. File Loading

**File**: `vv/configs/config.go`

Add a standalone function to load the project instructions file:

```go
// ProjectInstructionsFileName is the expected name of the project instructions file.
const ProjectInstructionsFileName = "VV.md"

// LoadProjectInstructions reads the VV.md file from the given directory.
// Returns the file content, or an empty string if the file does not exist.
// Logs a warning and returns empty string on read errors (permissions, I/O).
func LoadProjectInstructions(dir string) string {
    path := filepath.Join(dir, ProjectInstructionsFileName)
    data, err := os.ReadFile(path)
    if err != nil {
        if !os.IsNotExist(err) {
            slog.Warn("failed to read project instructions file", "path", path, "error", err)
        }
        return ""
    }
    return string(data)
}
```

This function is placed in `configs` because it is a configuration-loading concern. It uses `log/slog` for the warning on read errors (consistent with the existing `slog` usage in `setup.go`).

### 3. Setup Integration

**File**: `vv/setup/setup.go`

In `Init()`, after the working directory is captured and before the LLM client is created, load project instructions:

```go
func Init(cfg *configs.Config, opts *Options) (*InitResult, error) {
    // Capture working directory.
    if cfg.Tools.BashWorkingDir == "" {
        wd, wdErr := os.Getwd()
        if wdErr != nil {
            wd = "."
        }
        cfg.Tools.BashWorkingDir = wd
    }

    // Load project instructions from VV.md (new step).
    if cfg.ProjectInstructions == "" {
        cfg.ProjectInstructions = configs.LoadProjectInstructions(cfg.Tools.BashWorkingDir)
    }

    // ... rest unchanged ...
}
```

The guard `cfg.ProjectInstructions == ""` allows callers (e.g., tests) to pre-set instructions without file I/O.

### 4. FactoryOptions Extension

**File**: `vv/registries/registry.go`

Add `ProjectInstructions` to `FactoryOptions`:

```go
type FactoryOptions struct {
    LLM                 aimodel.ChatCompleter
    Model               string
    ToolRegistry         tool.ToolRegistry
    MaxIterations        int
    RunTokenBudget       int
    Memory               *memory.Manager
    PersistentMemory     memory.Memory
    ProjectInstructions  string // content from VV.md; empty if no file
}
```

### 5. System Prompt Assembly Helper

**File**: `vv/agents/project_instructions.go` (new file)

A single helper function used by all agent factories:

```go
package agents

// AppendProjectInstructions appends project instructions to a base system
// prompt. If instructions is empty, the base prompt is returned unchanged.
func AppendProjectInstructions(basePrompt, instructions string) string {
    if instructions == "" {
        return basePrompt
    }
    return basePrompt + "\n\n# Project Instructions\n\n" +
        "IMPORTANT: The following are project-specific instructions provided by the user. " +
        "These instructions should be followed and take precedence over default behaviors when applicable.\n\n" +
        instructions
}
```

This is a pure function with no side effects, trivially testable. It is used both by agent factories (for the base prompt string) and by the `PersistentMemoryPrompt` (for its base prompt).

### 6. Agent Factory Changes

Each agent factory incorporates project instructions. The changes are uniform across all six agents.

**Coder** (`vv/agents/coder.go`): The coder uses `PersistentMemoryPrompt` which takes a `basePrompt` string. The project instructions are appended to the base prompt before passing it to `PersistentMemoryPrompt`:

```go
Factory: func(opts registries.FactoryOptions) (agent.Agent, error) {
    basePrompt := AppendProjectInstructions(CoderSystemPrompt, opts.ProjectInstructions)

    var sysPrompt prompt.PromptTemplate
    if opts.PersistentMemory != nil {
        sysPrompt = NewPersistentMemoryPrompt(basePrompt, opts.PersistentMemory)
    } else {
        sysPrompt = prompt.StringPrompt(basePrompt)
    }
    // ... rest unchanged ...
}
```

**Chat, Researcher, Reviewer, Explorer** (`vv/agents/{chat,researcher,reviewer,explorer}.go`): Each uses `prompt.StringPrompt(XxxSystemPrompt)`. Change to:

```go
sysPrompt := AppendProjectInstructions(XxxSystemPrompt, opts.ProjectInstructions)
// ...
taskagent.WithSystemPrompt(prompt.StringPrompt(sysPrompt)),
```

**Planner** (`vv/agents/planner.go`): The planner factory currently uses the hardcoded `PlannerSystemPrompt`. Apply the same pattern:

```go
Factory: func(opts registries.FactoryOptions) (agent.Agent, error) {
    sysPrompt := AppendProjectInstructions(PlannerSystemPrompt, opts.ProjectInstructions)
    // ...
    taskagent.WithSystemPrompt(prompt.StringPrompt(sysPrompt)),
}
```

### 7. Setup Wiring

**File**: `vv/setup/setup.go`

Pass `ProjectInstructions` when building `FactoryOptions` in `New()`:

```go
factoryOpts := registries.FactoryOptions{
    LLM:                 llm,
    Model:               cfg.LLM.Model,
    ToolRegistry:         finalToolReg,
    MaxIterations:        cfg.Agents.MaxIterations,
    RunTokenBudget:       cfg.Agents.RunTokenBudget,
    Memory:               memMgr,
    PersistentMemory:     persistentMem,
    ProjectInstructions:  cfg.ProjectInstructions, // new
}
```

Similarly for the explorer and planner factory calls in `New()`.

### 8. Dynamic Agent Support

**File**: `vv/dispatches/dag.go`

Dynamic agents created by the dispatcher (via `DynamicAgentSpec` in DAG plans) use `desc.SystemPrompt` from the registry. When `spec.SystemPrompt` is empty, the fallback `desc.SystemPrompt` is used, which does not include project instructions (it is the static const).

To support project instructions for dynamic agents, add a `projectInstructions` field to the `Dispatcher` and apply it in `createDynamicAgent()`:

**File**: `vv/dispatches/dispatch.go` -- add field and option:

```go
type Dispatcher struct {
    // ... existing fields ...
    projectInstructions string // content from VV.md for dynamic agents
}

func WithProjectInstructions(instructions string) Option {
    return func(d *Dispatcher) {
        d.projectInstructions = instructions
    }
}
```

**File**: `vv/dispatches/dag.go` -- in `createDynamicAgent()`, after resolving the system prompt:

```go
systemPrompt := spec.SystemPrompt
if systemPrompt == "" {
    systemPrompt = desc.SystemPrompt
}
// Append project instructions to dynamic agent prompts.
if d.projectInstructions != "" {
    systemPrompt = agents.AppendProjectInstructions(systemPrompt, d.projectInstructions)
}
```

**File**: `vv/setup/setup.go` -- pass to dispatcher:

```go
dispatcher := dispatches.New(
    reg,
    subAgents,
    explorer,
    planner,
    planGen,
    // ... existing options ...
    dispatches.WithProjectInstructions(cfg.ProjectInstructions), // new
)
```

## Data Model

### Config (modified)

| Field | Type | Source | Persisted | Description |
|-------|------|--------|-----------|-------------|
| `ProjectInstructions` | `string` | `VV.md` file | No (`yaml:"-"`) | Raw markdown content; empty if file absent |

### FactoryOptions (modified)

| Field | Type | Description |
|-------|------|-------------|
| `ProjectInstructions` | `string` | Passed through from `Config.ProjectInstructions` |

### Dispatcher (modified)

| Field | Type | Description |
|-------|------|-------------|
| `projectInstructions` | `string` | Used when creating dynamic agents in DAG execution |

## System Prompt Assembly Order

For each agent, the final system prompt is assembled in this order:

1. Base system prompt (e.g., `CoderSystemPrompt`)
2. Project instructions section (from `VV.md`, wrapped with delimiter)
3. Persistent memory section (coder only, appended by `PersistentMemoryPrompt` at render time)

This ordering ensures project instructions have higher priority than default behavior (positioned after the base prompt) while persistent memory entries from previous sessions appear last as dynamic context.

## File Change Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `vv/configs/config.go` | Modify | Add `ProjectInstructions` field to `Config`, add `LoadProjectInstructions()` function, add `ProjectInstructionsFileName` constant |
| `vv/registries/registry.go` | Modify | Add `ProjectInstructions` field to `FactoryOptions` |
| `vv/agents/project_instructions.go` | New | `AppendProjectInstructions()` helper function |
| `vv/agents/coder.go` | Modify | Use `AppendProjectInstructions` before building prompt |
| `vv/agents/chat.go` | Modify | Use `AppendProjectInstructions` for system prompt |
| `vv/agents/researcher.go` | Modify | Use `AppendProjectInstructions` for system prompt |
| `vv/agents/reviewer.go` | Modify | Use `AppendProjectInstructions` for system prompt |
| `vv/agents/explorer.go` | Modify | Use `AppendProjectInstructions` for system prompt |
| `vv/agents/planner.go` | Modify | Use `AppendProjectInstructions` for system prompt |
| `vv/setup/setup.go` | Modify | Load `VV.md` in `Init()`, pass `ProjectInstructions` in `FactoryOptions` and dispatcher option |
| `vv/dispatches/dispatch.go` | Modify | Add `projectInstructions` field and `WithProjectInstructions` option |
| `vv/dispatches/dag.go` | Modify | Apply project instructions to dynamic agent system prompts |
| `vv/agents/project_instructions_test.go` | New | Tests for `AppendProjectInstructions` |
| `vv/configs/config_test.go` | Modify | Tests for `LoadProjectInstructions` |

## Implementation Plan

### Task 1: Add `ProjectInstructions` field and loader to Config

**File**: `vv/configs/config.go`

1. Add `import "log/slog"` to imports.
2. Add `ProjectInstructionsFileName` constant.
3. Add `ProjectInstructions string \`yaml:"-"\`` field to `Config` struct.
4. Add `LoadProjectInstructions(dir string) string` function.

**Tests**: Add unit tests in `vv/configs/config_test.go`:
- `TestLoadProjectInstructions_FileExists` -- create temp dir with `VV.md`, verify content returned.
- `TestLoadProjectInstructions_FileNotExists` -- temp dir without file, verify empty string, no error.
- `TestLoadProjectInstructions_YAMLExclusion` -- marshal/unmarshal a `Config` with `ProjectInstructions` set, verify it does not appear in YAML.

### Task 2: Add `ProjectInstructions` to `FactoryOptions`

**File**: `vv/registries/registry.go`

1. Add `ProjectInstructions string` field to `FactoryOptions`.

No tests needed -- this is a data-only change exercised by Task 4 tests.

### Task 3: Create `AppendProjectInstructions` helper

**File**: `vv/agents/project_instructions.go` (new)

1. Implement `AppendProjectInstructions(basePrompt, instructions string) string`.

**Tests** in `vv/agents/project_instructions_test.go` (new):
- `TestAppendProjectInstructions_Empty` -- empty instructions returns base prompt unchanged.
- `TestAppendProjectInstructions_NonEmpty` -- verifies delimiter header and content are appended.
- `TestAppendProjectInstructions_PreservesMarkdown` -- verifies markdown content (headings, code blocks) is preserved verbatim.

### Task 4: Update agent factories to use project instructions

**Files**: `vv/agents/{coder,chat,researcher,reviewer,explorer,planner}.go`

1. Each factory calls `AppendProjectInstructions(XxxSystemPrompt, opts.ProjectInstructions)`.
2. The resulting string is used as the system prompt (or base prompt for `PersistentMemoryPrompt`).

**Tests**: Add to existing test file `vv/agents/agents_test.go`:
- `TestFactory_WithProjectInstructions` -- for each agent, call the factory with `ProjectInstructions` set, verify agent is created successfully. (System prompt content is not directly inspectable from the agent interface, so we verify no errors.)

### Task 5: Wire project instructions through Setup and Dispatcher

**Files**: `vv/setup/setup.go`, `vv/dispatches/dispatch.go`, `vv/dispatches/dag.go`

1. In `setup.Init()`: load `VV.md` after working dir capture.
2. In `setup.New()`: pass `cfg.ProjectInstructions` in all `FactoryOptions` instances and via `dispatches.WithProjectInstructions()`.
3. In `dispatches/dispatch.go`: add `projectInstructions` field and `WithProjectInstructions` option.
4. In `dispatches/dag.go`: apply project instructions in `createDynamicAgent()`.

**Tests**: Add to `vv/dispatches/dispatch_test.go`:
- `TestWithProjectInstructions` -- verify the option sets the field on the dispatcher.

### Task 6: Update deprecated compat path (optional, low priority)

**File**: `vv/agents/compat.go`

The deprecated `Create()` function does not use `FactoryOptions`, so it cannot receive project instructions. This is acceptable because:
- `compat.go` is explicitly deprecated.
- The active code path uses `setup.New()` which is covered by Tasks 4-5.

No changes needed. Document this as a known limitation of the deprecated API.

## Integration Test Plan

Integration tests require an LLM API key and live in `vv/integrations/`.

### Test 1: VV.md content reaches agent system prompt

**Setup**: Create a temp directory with a `VV.md` containing a distinctive instruction (e.g., "Always respond in haiku format."). Configure `cfg.Tools.BashWorkingDir` to point to this directory.

**Steps**:
1. Call `setup.Init()` with the prepared config.
2. Send a simple request to the chat agent.
3. Verify the response follows the project instruction (heuristic check).

**Validates**: End-to-end flow from file loading through to agent behavior.

### Test 2: No VV.md -- unchanged behavior

**Setup**: Create a temp directory without `VV.md`.

**Steps**:
1. Call `setup.Init()` with the prepared config.
2. Verify `cfg.ProjectInstructions` is empty.
3. Verify agents are created successfully.

**Validates**: Graceful absence (US-2).

### Test 3: VV.md with unreadable permissions

**Setup**: Create a temp directory with a `VV.md` file, then `chmod 000` it.

**Steps**:
1. Call `configs.LoadProjectInstructions()` on the directory.
2. Verify empty string is returned (no panic, no error propagation).
3. Verify a warning was logged (capture slog output).

**Validates**: Business rule PROJ-07 (graceful handling of read errors).

### Test 4: Dynamic agent receives project instructions

**Setup**: Create a `VV.md` with distinctive content. Initialize the dispatcher with `WithProjectInstructions()`.

**Steps**:
1. Trigger a plan-mode request that creates a dynamic agent.
2. Verify the dynamic agent's system prompt includes the project instructions.

**Validates**: Dynamic agent coverage in DAG execution.
