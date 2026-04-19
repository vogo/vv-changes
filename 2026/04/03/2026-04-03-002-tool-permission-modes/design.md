# Design: Tool Permission Modes

## 1. Architecture Overview

This feature replaces the static `confirm_tools` list with a mode-based permission system that classifies tools by their read/write nature and applies per-mode policies. The changes span two modules:

- **vage** (schema layer): Add `ReadOnly` field to `ToolDef` so each tool self-declares its nature.
- **vv** (application layer): New permission mode config, new permission executor replacing `confirmingExecutor`, updated CLI confirmation dialog, `/permission` runtime command.

The design preserves the existing decorator pattern: the permission executor wraps a `tool.ToolRegistry` and intercepts `Execute()` calls, identical to how `confirmingExecutor` works today. No changes to the agent framework, orchestration, or HTTP layer are required.

**Key architectural decision:** Because `WrapToolRegistry` in `setup.go` is called once per dispatchable agent (creating a separate wrapped registry for coder, researcher, reviewer, chat), the mutable permission state (active mode and session-allowed set) is extracted into a shared `PermissionState` struct. All `permissionExecutor` instances reference the same `PermissionState`, so a `/permission` mode change affects all agents uniformly.

```
                     ToolRegistry (inner, per agent)
                           |
                  permissionExecutor (wraps inner, per agent)
                     |          |
              [shared PermissionState]   [confirmFn]
                     |
           Approve / Reject / Show confirmation dialog
```

**Mode applicability by run mode:**
- **CLI interactive mode:** Full permission logic with confirmation dialog.
- **CLI `-p` (single prompt) mode:** No `WrapToolRegistry` is applied; all tools execute without permission checks (equivalent to `auto`).
- **HTTP mode:** No `WrapToolRegistry` is applied; permission modes do not apply (per requirement A3/PERM-09).

## 2. Component Design

### 2.1 vage/schema: ToolDef Extension

Add a `ReadOnly` boolean field to `schema.ToolDef`. Each tool package sets this field in its `ToolDef()` method.

**File:** `vage/schema/tooldef.go`

```go
type ToolDef struct {
    Name         string `json:"name"`
    Description  string `json:"description"`
    Parameters   any    `json:"parameters,omitempty"`
    ForceUse     bool   `json:"force_use,omitempty"`
    Source       string `json:"source,omitempty"`
    MCPServerURI string `json:"mcp_server_uri,omitempty"`
    AgentID      string `json:"agent_id,omitempty"`
    ReadOnly     bool   `json:"read_only,omitempty"` // NEW
}
```

**Tool classification (set in each tool's `ToolDef()` method):**

| Package | Tool Name | ReadOnly |
|---------|-----------|----------|
| `tool/read` | read | true |
| `tool/glob` | glob | true |
| `tool/grep` | grep | true |
| `tool/askuser` | ask_user | true |
| `tool/write` | write | false (default) |
| `tool/edit` | edit | false (default) |
| `tool/bash` | bash | false (default) |

Since `false` is the zero value, only the read-only tools need an explicit `ReadOnly: true` in their `ToolDef()` return.

### 2.2 vv/configs: Permission Mode Configuration

**File:** `vv/configs/config.go`

```go
// PermissionMode defines the tool permission mode for CLI mode.
type PermissionMode string

const (
    PermissionModeDefault     PermissionMode = "default"
    PermissionModeAcceptEdits PermissionMode = "accept-edits"
    PermissionModeAuto        PermissionMode = "auto"
    PermissionModePlan        PermissionMode = "plan"
)

// ValidPermissionModes lists all valid permission mode values.
var ValidPermissionModes = []PermissionMode{
    PermissionModeDefault,
    PermissionModeAcceptEdits,
    PermissionModeAuto,
    PermissionModePlan,
}

// IsValidPermissionMode returns true if the given mode is a recognized permission mode.
func IsValidPermissionMode(m PermissionMode) bool {
    for _, v := range ValidPermissionModes {
        if v == m {
            return true
        }
    }
    return false
}
```

**CLIConfig changes:**

```go
type CLIConfig struct {
    ConfirmTools   []string       `yaml:"confirm_tools,omitempty"`   // DEPRECATED
    PermissionMode PermissionMode `yaml:"permission_mode,omitempty"` // NEW
}
```

**Load function additions (ordering: YAML parse -> env var override -> applyDefaults -> validate):**

1. After YAML parsing, read `VV_PERMISSION_MODE` env var override.
2. In `applyDefaults`, set `PermissionMode` to `"default"` if empty.
3. After `applyDefaults` returns, validate `PermissionMode` against `ValidPermissionModes`; return error on invalid value.
4. If `ConfirmTools` is non-empty, log a deprecation warning.

```go
// In Load(), after applyDefaults(cfg):
if !IsValidPermissionMode(cfg.CLI.PermissionMode) {
    return nil, fmt.Errorf("invalid permission_mode %q; valid values: default, accept-edits, auto, plan",
        cfg.CLI.PermissionMode)
}
if len(cfg.CLI.ConfirmTools) > 0 {
    slog.Warn("vv: confirm_tools is deprecated; use permission_mode instead")
}
```

**main.go additions:**

1. Add `--permission-mode` flag.
2. Apply flag override after config load (flag > env > yaml > default).

### 2.3 vv/cli: Permission State and Executor

Replace `confirmingExecutor` with `permissionExecutor`. The mutable state is separated into `PermissionState` so it can be shared across multiple executor instances (one per agent).

**File:** `vv/cli/permission.go` (new file, replaces `confirm.go`)

```go
// PermissionAction represents the user's response to a confirmation dialog.
type PermissionAction int

const (
    PermissionAllow       PermissionAction = iota // approve this invocation only
    PermissionAllowAlways                         // approve this and future invocations in session
    PermissionDeny                                // reject this invocation
)

// PermissionState holds the mutable permission state shared across all
// permissionExecutor instances. This is necessary because setup.New()
// creates a separate wrapped registry per agent, but permission policy
// must be uniform across all agents (requirement A5).
type PermissionState struct {
    mu             sync.Mutex
    mode           configs.PermissionMode
    sessionAllowed map[string]bool
}

// NewPermissionState creates a PermissionState with the given initial mode.
func NewPermissionState(mode configs.PermissionMode) *PermissionState {
    return &PermissionState{
        mode:           mode,
        sessionAllowed: make(map[string]bool),
    }
}

// Mode returns the current permission mode.
func (s *PermissionState) Mode() configs.PermissionMode {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.mode
}

// SetMode updates the permission mode and clears session-allowed tools.
func (s *PermissionState) SetMode(mode configs.PermissionMode) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.mode = mode
    s.sessionAllowed = make(map[string]bool)
}

// IsSessionAllowed returns true if the tool has been approved via "Allow Always".
func (s *PermissionState) IsSessionAllowed(name string) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    return s.sessionAllowed[name]
}

// AddSessionAllowed marks a tool as approved for the remainder of the session.
func (s *PermissionState) AddSessionAllowed(name string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.sessionAllowed[name] = true
}

// permissionExecutor wraps a tool.ToolRegistry, intercepting Execute() calls
// based on the active permission mode and session-allowed tools.
type permissionExecutor struct {
    tool.ToolRegistry
    state     *PermissionState
    confirmFn func(ctx context.Context, toolName, args string) (PermissionAction, error)
}
```

**Key methods:**

```go
// Execute intercepts tool calls and applies permission logic.
func (p *permissionExecutor) Execute(ctx context.Context, name, args string) (schema.ToolResult, error) {
    mode := p.state.Mode()

    // 1. Auto mode: approve everything.
    if mode == configs.PermissionModeAuto {
        return p.ToolRegistry.Execute(ctx, name, args)
    }

    // 2. Look up tool definition to check ReadOnly.
    def, found := p.ToolRegistry.Get(name)
    readOnly := found && def.ReadOnly

    // 3. Plan mode: reject non-read-only tools.
    if mode == configs.PermissionModePlan && !readOnly {
        return schema.ErrorResult("",
            fmt.Sprintf("Tool %q is not permitted in plan mode (read-only).", name)), nil
    }

    // 4. Read-only tools are always approved in default/accept-edits/plan modes.
    if readOnly {
        return p.ToolRegistry.Execute(ctx, name, args)
    }

    // 5. Accept-edits mode: also approve write and edit.
    // NOTE: These tool names are hardcoded because the requirement explicitly
    // scopes accept-edits to file modification tools (write, edit). If the
    // tool set grows to include more file-modifying tools, consider adding a
    // tool category attribute instead of extending this list.
    if mode == configs.PermissionModeAcceptEdits && (name == "write" || name == "edit") {
        return p.ToolRegistry.Execute(ctx, name, args)
    }

    // 6. Check session-allowed set.
    if p.state.IsSessionAllowed(name) {
        return p.ToolRegistry.Execute(ctx, name, args)
    }

    // 7. Show confirmation dialog.
    action, err := p.confirmFn(ctx, name, args)
    if err != nil {
        return schema.ErrorResult("", err.Error()), nil
    }

    switch action {
    case PermissionAllowAlways:
        p.state.AddSessionAllowed(name)
        return p.ToolRegistry.Execute(ctx, name, args)
    case PermissionAllow:
        return p.ToolRegistry.Execute(ctx, name, args)
    default:
        return schema.ErrorResult("", "Tool call rejected by user"), nil
    }
}
```

**Constructor:**

```go
// WrapRegistryWithPermission wraps a tool.ToolRegistry with permission logic.
// The shared PermissionState ensures all wrapped registries (one per agent)
// share the same mode and session-allowed set.
// Replaces the old WrapRegistry function.
func WrapRegistryWithPermission(
    inner tool.ToolRegistry,
    state *PermissionState,
) *permissionExecutor {
    return &permissionExecutor{
        ToolRegistry: inner,
        state:        state,
        confirmFn: func(_ context.Context, _, _ string) (PermissionAction, error) {
            // Default: allow all. App.wireConfirmFn() replaces this with the
            // real TUI confirmation function once the program is started.
            return PermissionAllow, nil
        },
    }
}
```

The old `WrapRegistry` function and `confirmingExecutor` type are **removed**. The old `confirm.go` file is replaced by `permission.go`.

### 2.4 vv/cli: Updated Confirmation Dialog

**File:** `vv/cli/cli.go`

The confirmation dialog changes from a `huh.Confirm` (Yes/No) to a `huh.Select` with three options.

**Updated `handleConfirmRequest`:**

```go
func (m *model) handleConfirmRequest(msg confirmRequestMsg) (tea.Model, tea.Cmd) {
    m.status = statusConfirming
    m.pendingTC = &schema.ToolCallStartData{
        ToolName:  msg.toolName,
        Arguments: msg.arguments,
    }

    var action string
    m.confirmForm = huh.NewForm(
        huh.NewGroup(
            huh.NewSelect[string]().
                Key("confirm").
                Title(fmt.Sprintf("Allow tool call: %s?", msg.toolName)).
                Description(truncate(msg.arguments, 200)).
                Options(
                    huh.NewOption("Allow", "allow"),
                    huh.NewOption("Allow Always (this session)", "allow_always"),
                    huh.NewOption("Deny", "deny"),
                ).
                Value(&action),
        ),
    ).WithShowHelp(false)

    return m, m.confirmForm.Init()
}
```

**Updated form completion handler (in `Update`):**

The `statusConfirming` branch changes from reading a `bool` to reading a `string` action value. The channel type changes from `chan bool` to `chan PermissionAction`.

```go
if m.status == statusConfirming && m.confirmForm != nil {
    form, cmd := m.confirmForm.Update(msg)
    if f, ok := form.(*huh.Form); ok {
        m.confirmForm = f
        if f.State == huh.StateCompleted {
            action := f.GetString("confirm")
            var pa PermissionAction
            switch action {
            case "allow_always":
                pa = PermissionAllowAlways
            case "deny":
                pa = PermissionDeny
            default:
                pa = PermissionAllow
            }
            m.confirmCh <- pa
            m.status = statusProcessing
            m.confirmForm = nil
            m.pendingTC = nil
        }
    }
    cmds = append(cmds, cmd)
    return m, tea.Batch(cmds...)
}
```

### 2.5 vv/cli: `/permission` Command

The `/permission` command handler lives in `permission.go` alongside the executor logic, keeping permission concerns self-contained. It is invoked from `handleCommand` in `memory.go`.

**File:** `vv/cli/memory.go` (add routing)

```go
if parts[0] == "/permission" {
    return m.handlePermissionCommand(parts[1:])
}
```

**File:** `vv/cli/permission.go` (add handler method)

```go
func (m *model) handlePermissionCommand(args []string) tea.Cmd {
    ps := m.app.permissionState
    if ps == nil {
        return m.printSystem("Permission mode is not available.")
    }

    if len(args) == 0 {
        return m.printSystem(fmt.Sprintf("Current permission mode: %s", ps.Mode()))
    }

    mode := configs.PermissionMode(args[0])
    if !configs.IsValidPermissionMode(mode) {
        return m.printSystem(fmt.Sprintf(
            "Invalid permission mode: %q. Valid modes: default, accept-edits, auto, plan", args[0]))
    }

    ps.SetMode(mode)
    return m.printSystem(fmt.Sprintf("Permission mode changed to %s.", mode))
}
```

### 2.6 vv/cli: App Changes

**File:** `vv/cli/cli.go`

Add `permissionState *PermissionState` field to `App` and a list of `permissionExecutor` instances for wiring the confirmFn. Use a functional option for injection.

```go
type App struct {
    // ... existing fields ...
    permissionState *PermissionState           // NEW: shared state for /permission command
    permissionExecs []*permissionExecutor       // NEW: all executor instances for confirmFn wiring
}

// WithPermissionState sets the shared permission state on the App.
func WithPermissionState(state *PermissionState, execs []*permissionExecutor) func(*App) {
    return func(a *App) {
        a.permissionState = state
        a.permissionExecs = execs
    }
}
```

**Updated `wireConfirmFn`:**

This method completes the wiring that was previously left as a no-op placeholder. It connects all `permissionExecutor` instances to the TUI confirmation dialog.

```go
func (a *App) wireConfirmFn() {
    if len(a.permissionExecs) == 0 {
        return
    }

    fn := func(ctx context.Context, toolName, args string) (PermissionAction, error) {
        a.program.Send(confirmRequestMsg{toolName: toolName, arguments: args})
        select {
        case action := <-a.confirmCh:
            return action, nil
        case <-ctx.Done():
            return PermissionDeny, ctx.Err()
        }
    }

    for _, pe := range a.permissionExecs {
        pe.confirmFn = fn
    }
}
```

Where `a.confirmCh` changes type from `chan bool` to `chan PermissionAction`.

**Updated `handleCtrlC` (statusConfirming branch):**

```go
case statusConfirming:
    select {
    case m.confirmCh <- PermissionDeny:
    default:
    }
    // ... rest unchanged
```

**Updated `newModel`:**

```go
confirmCh: make(chan PermissionAction, 1),
```

### 2.7 vv/main.go: Wiring Changes

```go
// Replace:
//   confirmTools := cfg.CLI.ConfirmTools
// With permission mode setup:

permissionMode := cfg.CLI.PermissionMode
// --permission-mode flag override (parsed earlier)
if *permissionModeFlag != "" {
    permissionMode = configs.PermissionMode(*permissionModeFlag)
}

// Deprecation warning (already logged in Load, but re-check for flag override path).
if len(cfg.CLI.ConfirmTools) > 0 {
    slog.Warn("vv: confirm_tools is deprecated; use permission_mode instead")
}
```

Create shared state and collect executor instances:

```go
permissionState := cli.NewPermissionState(permissionMode)
var permissionExecs []*cli.PermissionExecutor

initResult, err := setup.Init(cfg, &setup.Options{
    WrapToolRegistry: func(r *tool.Registry) tool.ToolRegistry {
        pe := cli.WrapRegistryWithPermission(r, permissionState)
        permissionExecs = append(permissionExecs, pe)
        return pe
    },
    UserInteractor: interactor,
    AskUserTimeout: askUserTimeout,
})
if err != nil {
    slog.Error("vv: init", "error", err)
    os.Exit(1)
}

// Pass shared permission state and executor list to App.
app := cli.New(initResult.SetupResult.Dispatcher, cfg, initResult.PersistentMem,
    cliInteractor, initResult.Compactor,
    cli.WithPermissionState(permissionState, permissionExecs))
```

Note: `permissionExecutor` must be exported as `PermissionExecutor` for use in `main.go`, or the `WithPermissionState` option should accept `[]*permissionExecutor` via a different mechanism. The cleanest approach is to keep `permissionExecutor` unexported and have `WrapRegistryWithPermission` return a `tool.ToolRegistry` while `main.go` only passes the `*PermissionState`. The `App` can then discover executors through the state, or `WrapRegistryWithPermission` can register each executor with the state:

```go
// Alternative: PermissionState tracks its executors internally.
func (s *PermissionState) RegisterExecutor(pe *permissionExecutor) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.executors = append(s.executors, pe)
}

func (s *PermissionState) SetConfirmFn(fn func(ctx context.Context, toolName, args string) (PermissionAction, error)) {
    s.mu.Lock()
    defer s.mu.Unlock()
    for _, pe := range s.executors {
        pe.confirmFn = fn
    }
}
```

This keeps `permissionExecutor` unexported and `main.go` only deals with `*PermissionState`:

```go
permissionState := cli.NewPermissionState(permissionMode)

initResult, err := setup.Init(cfg, &setup.Options{
    WrapToolRegistry: func(r *tool.Registry) tool.ToolRegistry {
        return cli.WrapRegistryWithPermission(r, permissionState)
    },
    // ...
})

app := cli.New(initResult.SetupResult.Dispatcher, cfg, initResult.PersistentMem,
    cliInteractor, initResult.Compactor,
    cli.WithPermissionState(permissionState))
```

And in `wireConfirmFn`:

```go
func (a *App) wireConfirmFn() {
    if a.permissionState == nil {
        return
    }
    a.permissionState.SetConfirmFn(func(ctx context.Context, toolName, args string) (PermissionAction, error) {
        a.program.Send(confirmRequestMsg{toolName: toolName, arguments: args})
        select {
        case action := <-a.confirmCh:
            return action, nil
        case <-ctx.Done():
            return PermissionDeny, ctx.Err()
        }
    })
}
```

## 3. Data Models / Schemas

### 3.1 ToolDef (vage/schema/tooldef.go)

| Field | Type | JSON | Change |
|-------|------|------|--------|
| ReadOnly | bool | `read_only,omitempty` | NEW |

### 3.2 CLIConfig (vv/configs/config.go)

| Field | Type | YAML | Change |
|-------|------|------|--------|
| PermissionMode | PermissionMode | `permission_mode,omitempty` | NEW |
| ConfirmTools | []string | `confirm_tools,omitempty` | DEPRECATED (add omitempty) |

### 3.3 PermissionMode Enum (vv/configs/config.go)

| Value | Description |
|-------|-------------|
| `default` | Read-only auto-approved; write/execute require confirmation |
| `accept-edits` | Read-only + write/edit auto-approved; bash requires confirmation |
| `auto` | All tools auto-approved |
| `plan` | Read-only only; write/execute rejected |

### 3.4 PermissionAction Enum (vv/cli/permission.go)

| Value | Description |
|-------|-------------|
| `PermissionAllow` | Approve this invocation only |
| `PermissionAllowAlways` | Approve for remainder of session |
| `PermissionDeny` | Reject this invocation |

### 3.5 PermissionState (vv/cli/permission.go)

| Field | Type | Description |
|-------|------|-------------|
| mode | PermissionMode | Active permission mode |
| sessionAllowed | map[string]bool | Tools approved via "Allow Always" |
| executors | []*permissionExecutor | All executor instances (for confirmFn wiring) |

Cleared when: session ends, or `/permission <mode>` changes the mode.

## 4. API Contracts

No HTTP API changes. Permission modes are CLI-only (per requirement A3/PERM-09). The HTTP mode path in `main.go` does not use `WrapToolRegistry` and is unaffected. The `-p` (single prompt) mode also does not use `WrapToolRegistry`, so all tools execute without permission checks.

### CLI Flag Contract

```
--permission-mode <mode>    Tool permission mode: default, accept-edits, auto, plan
```

### Environment Variable Contract

```
VV_PERMISSION_MODE=<mode>   Overrides yaml permission_mode
```

### YAML Configuration Contract

```yaml
cli:
  permission_mode: default   # default | accept-edits | auto | plan
  # confirm_tools: [...]     # DEPRECATED - use permission_mode
```

### `/permission` Command Contract

```
/permission                 Show current permission mode
/permission <mode>          Switch to <mode>; clears session-allowed tools
```

## 5. Implementation Plan

### Phase 1: Schema Extension (vage module)

**Task 1.1:** Add `ReadOnly` field to `schema.ToolDef`
- File: `vage/schema/tooldef.go`
- Add `ReadOnly bool` field with JSON tag `read_only,omitempty`

**Task 1.2:** Set `ReadOnly: true` on read-only tools
- Files: `vage/tool/read/read_tool.go`, `vage/tool/glob/glob_tool.go`, `vage/tool/grep/grep_tool.go`, `vage/tool/askuser/askuser.go`
- In each tool's `ToolDef()` method, add `ReadOnly: true` to the returned `schema.ToolDef`

**Task 1.3:** Unit tests for ToolDef ReadOnly
- Verify each tool's `ToolDef()` returns the expected `ReadOnly` value

### Phase 2: Configuration (vv module)

**Task 2.1:** Add `PermissionMode` type and validation to `configs`
- File: `vv/configs/config.go`
- Add `PermissionMode` type, constants, `IsValidPermissionMode()` function, `ValidPermissionModes` slice
- Add `PermissionMode` field to `CLIConfig`
- Add `omitempty` to `ConfirmTools` yaml tag

**Task 2.2:** Add env var, default handling, and validation to `Load`
- File: `vv/configs/config.go`
- Read `VV_PERMISSION_MODE` env var
- Set default to `"default"` in `applyDefaults`
- Validate mode value after `applyDefaults`; return error on invalid
- Log deprecation warning if `ConfirmTools` is non-empty

**Task 2.3:** Add `--permission-mode` CLI flag
- File: `vv/main.go`
- Add flag parsing; apply override after config load

**Task 2.4:** Unit tests for config loading
- Test default value, env var override, invalid value error, deprecation warning

### Phase 3: Permission Executor (vv module)

**Task 3.1:** Implement `PermissionState` and `permissionExecutor`
- File: `vv/cli/permission.go` (new file, replaces `confirm.go`)
- Implement `PermissionState` struct with `Mode`, `SetMode`, `SetConfirmFn`, `IsSessionAllowed`, `AddSessionAllowed` methods
- Implement `permissionExecutor` struct with `Execute` method
- Implement `WrapRegistryWithPermission` constructor (registers executor with state)
- Follow the decision flowchart from the requirement

**Task 3.2:** Add `/permission` command handler to `permission.go`
- Implement `handlePermissionCommand` method on `model`

**Task 3.3:** Remove old `confirmingExecutor`
- Delete `vv/cli/confirm.go` contents
- The old `WrapRegistry` function is removed; callers updated

**Task 3.4:** Unit tests for permission executor
- File: `vv/cli/permission_test.go` (replaces `confirm_test.go`)
- Test cases for each mode (default, accept-edits, auto, plan)
- Test session-allowed ("Allow Always") behavior
- Test mode switching clears session-allowed set
- Test shared state across multiple executor instances
- Test delegation (Register, Get, List, Merge, Unregister)
- Test context cancellation during confirmation

### Phase 4: CLI Integration (vv module)

**Task 4.1:** Update confirmation dialog to three options
- File: `vv/cli/cli.go`
- Change `huh.Confirm` to `huh.Select[string]` with Allow/Allow Always/Deny
- Change `confirmCh` from `chan bool` to `chan PermissionAction`
- Update form completion handler to read string action

**Task 4.2:** Wire `PermissionState` into `App`
- Files: `vv/cli/cli.go`, `vv/main.go`
- Add `permissionState` field to `App`
- Add `WithPermissionState` functional option
- Complete `wireConfirmFn` implementation (replaces no-op placeholder) using `state.SetConfirmFn`
- Update `main.go` to create shared `PermissionState` and pass to `App`

**Task 4.3:** Add `/permission` routing in `handleCommand`
- File: `vv/cli/memory.go`
- Add routing for `/permission` to `handlePermissionCommand`

**Task 4.4:** Update `handleCtrlC` for new channel type
- File: `vv/cli/cli.go`
- Send `PermissionDeny` instead of `false` on Ctrl+C during confirmation

### Phase 5: Cleanup

**Task 5.1:** Remove old `confirm.go` and `confirm_test.go` files
- Delete `vv/cli/confirm.go` (replaced by `permission.go`)
- Delete `vv/cli/confirm_test.go` (replaced by `permission_test.go`)

**Task 5.2:** Update `vv/CLAUDE.md` documentation
- Update CLI section to reference permission modes instead of confirm_tools
- Update configuration section with new fields

**Task 5.3:** Run full test suite
- `cd vage && make build`
- `cd vv && make build`

## 6. Integration Test Plan

Integration tests go in `vv/integrations/cli_tests/` (if the directory exists) or as unit tests with mocks.

### 6.1 Permission Mode Decision Logic (unit tests, `vv/cli/permission_test.go`)

| Test Case | Mode | Tool | ReadOnly | Session Allowed | Expected |
|-----------|------|------|----------|-----------------|----------|
| Auto approves all | auto | bash | false | no | APPROVE |
| Default approves read-only | default | read | true | no | APPROVE |
| Default confirms write | default | write | false | no | CONFIRM |
| Default confirms bash | default | bash | false | no | CONFIRM |
| Accept-edits approves read-only | accept-edits | grep | true | no | APPROVE |
| Accept-edits approves write | accept-edits | write | false | no | APPROVE |
| Accept-edits approves edit | accept-edits | edit | false | no | APPROVE |
| Accept-edits confirms bash | accept-edits | bash | false | no | CONFIRM |
| Plan approves read-only | plan | read | true | no | APPROVE |
| Plan rejects write | plan | write | false | no | REJECT with message |
| Plan rejects bash | plan | bash | false | no | REJECT with message |
| Session allowed bypasses confirm | default | bash | false | yes | APPROVE |
| Allow Always adds to session | default | bash | false | no | APPROVE + session |
| Deny rejects | default | bash | false | no | REJECT |
| Mode switch clears session | - | - | - | yes | session cleared |
| Shared state across executors | default | bash | false | yes (via other executor) | APPROVE |

### 6.2 Configuration Loading (unit tests, `vv/configs/config_test.go`)

| Test Case | Expected |
|-----------|----------|
| Default permission_mode when omitted | "default" |
| YAML permission_mode loaded correctly | value from YAML |
| VV_PERMISSION_MODE env var overrides YAML | env value used |
| Invalid permission_mode returns error | error returned |
| Deprecated confirm_tools logs warning | warning logged |

### 6.3 CLI Flag Override (test in main_test.go or manually)

| Test Case | Expected |
|-----------|----------|
| --permission-mode flag overrides env and YAML | flag value used |
| Invalid flag value produces error | error at startup |

### 6.4 `/permission` Command (unit test with mock)

| Test Case | Expected |
|-----------|----------|
| `/permission` shows current mode | system message with mode |
| `/permission auto` switches mode | mode changed, session cleared |
| `/permission invalid` shows error | error message with valid modes |

### 6.5 ToolDef ReadOnly (unit tests per tool package)

| Test Case | Expected |
|-----------|----------|
| read ToolDef().ReadOnly | true |
| glob ToolDef().ReadOnly | true |
| grep ToolDef().ReadOnly | true |
| ask_user ToolDef().ReadOnly | true |
| write ToolDef().ReadOnly | false |
| edit ToolDef().ReadOnly | false |
| bash ToolDef().ReadOnly | false |
