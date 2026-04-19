# Design: Project Instructions File (VV.md)

## Architecture Overview

This feature adds project-level instruction loading from a `VV.md` file in the working directory, injecting its content into all agent system prompts and the intent recognition system prompt. The design touches four layers of the `vv` module:

1. **Config** -- new `ProjectInstructions` field on `configs.Config`, loader in dedicated file
2. **Setup** -- file loading in `setup.Init()` after working directory capture
3. **Registry** -- `ProjectInstructions` field added to `registries.FactoryOptions`
4. **Agents + Dispatcher** -- each factory function and the dispatcher's intent recognition incorporate project instructions via a shared helper

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

**File**: `vv/configs/project_instructions.go` (new file)

A dedicated file for the project instructions loader, keeping it separate from the YAML/env-var configuration logic in `config.go`.

```go
package configs

import (
    "log/slog"
    "os"
    "path/filepath"
)

// ProjectInstructionsFileName is the expected name of the project instructions file.
const ProjectInstructionsFileName = "VV.md"

// LoadProjectInstructions reads the VV.md file from the given directory.
// Returns the file content, or an empty string if the file does not exist or is empty.
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

This function is placed in a dedicated file within `configs` because it is a configuration-loading concern, but separated from the main `config.go` to avoid mixing YAML config logic with file-system scanning. It uses `log/slog` for the warning on read errors (consistent with the existing `slog` usage in `setup.go`).

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

This is a pure function with no side effects, trivially testable. It is used by agent factories, the `PersistentMemoryPrompt` base prompt, and the dispatcher's intent recognition.

### 6. Agent Factory Changes

Each agent factory incorporates project instructions. The changes are uniform across all six agents.

**Coder** (`vv/agents/coder.go`): The coder uses `PersistentMemoryPrompt` which takes a `basePrompt` string. The project instructions are appended to the base prompt before passing it to `PersistentMemoryPrompt`. Note: project instructions become permanently baked into the `PersistentMemoryPrompt.basePrompt` string. This is correct because project instructions do not change during a session (no hot-reload is in scope).

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

**Planner** (`vv/agents/planner.go`): The planner factory uses the hardcoded fallback `PlannerSystemPrompt`. Apply the same pattern for correctness (in case the planner is ever used standalone), but note that in the main code path the planner's system prompt is effectively overridden by the dispatcher's intent recognition system prompt (see Section 8):

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

### 8. Intent Recognition Integration

**File**: `vv/dispatches/dispatch.go`, `vv/dispatches/intent.go`

The intent recognition system makes direct LLM calls with `d.intentSystemPrompt` in `recognizeIntentDirect()` and `recognizeIntentViaPlannerStream()`. This prompt controls routing decisions and must also receive project instructions so that project context can influence intent classification.

Add a `projectInstructions` field to the `Dispatcher` and a corresponding option:

**File**: `vv/dispatches/dispatch.go`:

```go
type Dispatcher struct {
    // ... existing fields ...
    projectInstructions string // content from VV.md for intent recognition and dynamic agents
}

func WithProjectInstructions(instructions string) Option {
    return func(d *Dispatcher) {
        d.projectInstructions = instructions
    }
}
```

**File**: `vv/dispatches/intent.go` -- in `recognizeIntentDirect()`, after resolving the system prompt and before the working directory substitution:

```go
func (d *Dispatcher) recognizeIntentDirect(ctx context.Context, req *schema.RunRequest) (*IntentResult, string, *aimodel.Usage, error) {
    systemPrompt := d.intentSystemPrompt
    if systemPrompt == "" {
        systemPrompt = "You are a task planner. Respond with JSON: ..."
    }

    // Append project instructions to intent recognition prompt.
    if d.projectInstructions != "" {
        systemPrompt = agents.AppendProjectInstructions(systemPrompt, d.projectInstructions)
    }

    systemPrompt = strings.Replace(systemPrompt, "{{.WorkingDir}}", d.workingDir, 1)
    // ... rest unchanged ...
}
```

Apply the same pattern in `recognizeIntentViaPlannerStream()`.

### 9. Dynamic Agent Support

**File**: `vv/dispatches/dag.go`

Dynamic agents created by the dispatcher (via `DynamicAgentSpec` in DAG plans) use `desc.SystemPrompt` from the registry. The `projectInstructions` field on the Dispatcher (added in Section 8) is reused here.

In `buildDynamicAgent()`, after resolving the system prompt:

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

### 10. Setup Dispatcher Wiring

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
| `ProjectInstructions` | `string` | `VV.md` file | No (`yaml:"-"`) | Raw markdown content; empty if file absent or empty |

### FactoryOptions (modified)

| Field | Type | Description |
|-------|------|-------------|
| `ProjectInstructions` | `string` | Passed through from `Config.ProjectInstructions` |

### Dispatcher (modified)

| Field | Type | Description |
|-------|------|-------------|
| `projectInstructions` | `string` | Used for intent recognition and dynamic agent creation |

## System Prompt Assembly Order

For each agent, the final system prompt is assembled in this order:

1. Base system prompt (e.g., `CoderSystemPrompt`)
2. Project instructions section (from `VV.md`, wrapped with delimiter)
3. Persistent memory section (coder only, appended by `PersistentMemoryPrompt` at render time)

This ordering ensures project instructions have higher priority than default behavior (positioned after the base prompt) while persistent memory entries from previous sessions appear last as dynamic context.

For intent recognition, the assembly order is:

1. Intent system prompt template (with `{{.AgentList}}` resolved)
2. Project instructions section (appended via `AppendProjectInstructions`)
3. Working directory substitution (`{{.WorkingDir}}`)

## File Change Summary

| File | Change Type | Description |
|------|-------------|-------------|
| `vv/configs/config.go` | Modify | Add `ProjectInstructions` field to `Config` |
| `vv/configs/project_instructions.go` | New | `LoadProjectInstructions()` function and `ProjectInstructionsFileName` constant |
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
| `vv/dispatches/intent.go` | Modify | Append project instructions to intent recognition system prompt |
| `vv/dispatches/dag.go` | Modify | Apply project instructions to dynamic agent system prompts |
| `vv/configs/project_instructions_test.go` | New | Tests for `LoadProjectInstructions` |
| `vv/agents/project_instructions_test.go` | New | Tests for `AppendProjectInstructions` |

## Implementation Plan

### Task 1: Add `ProjectInstructions` field and loader

**Files**: `vv/configs/config.go`, `vv/configs/project_instructions.go` (new)

1. In `config.go`: Add `ProjectInstructions string \`yaml:"-"\`` field to `Config` struct.
2. In `project_instructions.go` (new): Add `ProjectInstructionsFileName` constant and `LoadProjectInstructions(dir string) string` function.

**Tests** in `vv/configs/project_instructions_test.go` (new):
- `TestLoadProjectInstructions_FileExists` -- create temp dir with `VV.md`, verify content returned.
- `TestLoadProjectInstructions_FileNotExists` -- temp dir without file, verify empty string, no error.
- `TestLoadProjectInstructions_EmptyFile` -- temp dir with empty `VV.md`, verify empty string returned.
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
- `TestFactory_WithProjectInstructions` -- for each agent, call the factory with `ProjectInstructions` set, verify agent is created successfully.

### Task 5: Wire project instructions through Setup, Dispatcher, and Intent Recognition

**Files**: `vv/setup/setup.go`, `vv/dispatches/dispatch.go`, `vv/dispatches/intent.go`, `vv/dispatches/dag.go`

1. In `setup.Init()`: load `VV.md` after working dir capture.
2. In `setup.New()`: pass `cfg.ProjectInstructions` in all `FactoryOptions` instances and via `dispatches.WithProjectInstructions()`.
3. In `dispatches/dispatch.go`: add `projectInstructions` field and `WithProjectInstructions` option.
4. In `dispatches/intent.go`: append project instructions to `systemPrompt` in `recognizeIntentDirect()` and `recognizeIntentViaPlannerStream()`, before the `{{.WorkingDir}}` substitution.
5. In `dispatches/dag.go`: apply project instructions in `buildDynamicAgent()`.

**Tests**: Add to `vv/dispatches/dispatch_test.go`:
- `TestWithProjectInstructions` -- verify the option sets the field on the dispatcher.

## Test Plan

### Unit Tests (deterministic, no LLM required)

**Test 1: LoadProjectInstructions file loading**

Covers all file-system scenarios: file exists with content, file does not exist, empty file, permission error. Uses `t.TempDir()` for isolation.

**Test 2: AppendProjectInstructions assembly**

Verifies the pure function: empty input passthrough, non-empty input appending with delimiter, markdown preservation.

**Test 3: YAML exclusion**

Marshal a `Config` with `ProjectInstructions` set; verify the field does not appear in the YAML output.

**Test 4: Agent factory accepts ProjectInstructions**

For each of the six agents, call the factory with a non-empty `ProjectInstructions` in `FactoryOptions`. Verify no error. (System prompt content is not directly inspectable from the agent interface.)

**Test 5: System prompt content verification**

Create a mock `ChatCompleter` that captures the `ChatRequest`. Build a chat agent with project instructions, run it, and assert the system message in the captured request contains the VV.md content and the delimiter header. This provides deterministic verification that the project instructions reach the LLM.

**Test 6: Dispatcher option**

Verify `WithProjectInstructions()` sets the field on the `Dispatcher`.

### Integration Tests (require LLM API key, in `vv/integrations/`)

**Test 1: No VV.md -- unchanged behavior**

Setup: Create a temp directory without `VV.md`. Call `setup.Init()`. Verify `cfg.ProjectInstructions` is empty and agents are created successfully.

**Test 2: VV.md with unreadable permissions**

Setup: Create `VV.md` with `chmod 000`. Call `configs.LoadProjectInstructions()`. Verify empty string returned with no panic. (Note: this test may need to be skipped on systems where tests run as root.)

**Test 3: End-to-end with VV.md**

Setup: Create a temp directory with a `VV.md` containing distinctive content. Call `setup.Init()` with the config pointing to that directory. Verify `cfg.ProjectInstructions` contains the expected content. Create a mock `ChatCompleter`, build agents, and verify the system prompt includes the instructions. This avoids relying on LLM output heuristics.

Note: The deprecated `compat.go` `Create()` function does not use `FactoryOptions` and will not receive project instructions. This is acceptable as the active code path uses `setup.New()`.
