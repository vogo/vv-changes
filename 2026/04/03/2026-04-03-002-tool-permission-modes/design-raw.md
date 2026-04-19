# Design: Tool Permission Modes

## 1. Architecture Overview

This feature replaces the static `confirm_tools` list with a mode-based permission system that classifies tools by their read/write nature and applies per-mode policies. The changes span two modules:

- **vage** (schema layer): Add `ReadOnly` field to `ToolDef` so each tool self-declares its nature.
- **vv** (application layer): New permission mode config, new permission executor replacing `confirmingExecutor`, updated CLI confirmation dialog, `/permission` runtime command.

The design preserves the existing decorator pattern: the permission executor wraps a `tool.ToolRegistry` and intercepts `Execute()` calls, identical to how `confirmingExecutor` works today. No changes to the agent framework, orchestration, or HTTP layer are required.

```
                     ToolRegistry (inner)
                           |
                  permissionExecutor (wraps inner)
                     |          |
              [mode logic]   [session_allowed set]
                     |
           Approve / Reject / Show confirmation dialog
```

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
```

**CLIConfig changes:**

```go
type CLIConfig struct {
    ConfirmTools   []string       `yaml:"confirm_tools,omitempty"`   // DEPRECATED
    PermissionMode PermissionMode `yaml:"permission_mode,omitempty"` // NEW
}
```

**Load function additions:**

1. Read `VV_PERMISSION_MODE` env var override.
2. In `applyDefaults`, set `PermissionMode` to `"default"` if empty.
3. Validate `PermissionMode` against `ValidPermissionModes`; return error on invalid value.
4. If `ConfirmTools` is non-empty and `PermissionMode` is set, log a deprecation warning.

**main.go additions:**

1. Add `--permission-mode` flag.
2. Apply flag override after config load (flag > env > yaml > default).

### 2.3 vv/cli: Permission Executor

Replace `confirmingExecutor` with `permissionExecutor` in a new file `vv/cli/permission.go`.

**File:** `vv/cli/permission.go`

```go
// PermissionAction represents the user's response to a confirmation dialog.
type PermissionAction int

const (
    PermissionAllow       PermissionAction = iota // approve this invocation only
    PermissionAllowAlways                         // approve this and future invocations in session
    PermissionDeny                                // reject this invocation
)

// permissionExecutor wraps a tool.ToolRegistry, intercepting Execute() calls
// based on the active permission mode and session-allowed tools.
type permissionExecutor struct {
    tool.ToolRegistry
    mode              configs.PermissionMode
    sessionAllowed    map[string]bool
    confirmFn         func(ctx context.Context, toolName, args string) (PermissionAction, error)
    mu                sync.Mutex // protects mode and sessionAllowed
}
```

**Key methods:**

```go
// Execute intercepts tool calls and applies permission logic.
func (p *permissionExecutor) Execute(ctx context.Context, name, args string) (schema.ToolResult, error)

// SetMode updates the permission mode and clears session-allowed tools.
func (p *permissionExecutor) SetMode(mode configs.PermissionMode)

// Mode returns the current permission mode.
func (p *permissionExecutor) Mode() configs.PermissionMode
```

**Execute logic (matches requirement flowchart):**

```go
func (p *permissionExecutor) Execute(ctx context.Context, name, args string) (schema.ToolResult, error) {
    p.mu.Lock()
    mode := p.mode
    allowed := p.sessionAllowed[name]
    p.mu.Unlock()

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
    if mode == configs.PermissionModeAcceptEdits && (name == "write" || name == "edit") {
        return p.ToolRegistry.Execute(ctx, name, args)
    }

    // 6. Check session-allowed set.
    if allowed {
        return p.ToolRegistry.Execute(ctx, name, args)
    }

    // 7. Show confirmation dialog.
    action, err := p.confirmFn(ctx, name, args)
    if err != nil {
        return schema.ErrorResult("", err.Error()), nil
    }

    switch action {
    case PermissionAllowAlways:
        p.mu.Lock()
        p.sessionAllowed[name] = true
        p.mu.Unlock()
        return p.ToolRegistry.Execute(ctx, name, args)
    case PermissionAllow:
        return p.ToolRegistry.Execute(ctx, name, args)
    default:
        return schema.ErrorResult("", "Tool call rejected by user"), nil
    }
}
```

**Constructor and WrapRegistry replacement:**

```go
// WrapRegistryWithPermission wraps a tool.ToolRegistry with permission logic.
// Replaces the old WrapRegistry function.
func WrapRegistryWithPermission(
    inner tool.ToolRegistry,
    mode configs.PermissionMode,
) *permissionExecutor {
    return &permissionExecutor{
        ToolRegistry:   inner,
        mode:           mode,
        sessionAllowed: make(map[string]bool),
        confirmFn: func(_ context.Context, _, _ string) (PermissionAction, error) {
            return PermissionAllow, nil // default: allow all; App wires real fn
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

**File:** `vv/cli/memory.go` (rename consideration: this file handles all commands, but keeping it simple)

Add to `handleCommand`:

```go
if parts[0] == "/permission" {
    return m.handlePermissionCommand(parts[1:])
}
```

```go
func (m *model) handlePermissionCommand(args []string) tea.Cmd {
    pe := m.app.permissionExec
    if pe == nil {
        return m.printSystem("Permission mode is not available (HTTP mode).")
    }

    if len(args) == 0 {
        return m.printSystem(fmt.Sprintf("Current permission mode: %s", pe.Mode()))
    }

    mode := configs.PermissionMode(args[0])
    if !configs.IsValidPermissionMode(mode) {
        return m.printSystem(fmt.Sprintf(
            "Invalid permission mode: %q. Valid modes: default, accept-edits, auto, plan", args[0]))
    }

    pe.SetMode(mode)
    return m.printSystem(fmt.Sprintf("Permission mode changed to %s.", mode))
}
```

### 2.6 vv/cli: App Changes

**File:** `vv/cli/cli.go`

Add `permissionExec *permissionExecutor` field to `App`. This reference is set during construction so that `wireConfirmFn` and `/permission` can access it.

```go
type App struct {
    // ... existing fields ...
    permissionExec *permissionExecutor // NEW: for /permission command and confirmFn wiring
}
```

**Updated `wireConfirmFn`:**

```go
func (a *App) wireConfirmFn() {
    if a.permissionExec == nil {
        return
    }
    a.permissionExec.confirmFn = func(ctx context.Context, toolName, args string) (PermissionAction, error) {
        a.program.Send(confirmRequestMsg{toolName: toolName, arguments: args})
        select {
        case action := <-a.confirmCh:
            return action, nil
        case <-ctx.Done():
            return PermissionDeny, ctx.Err()
        }
    }
}
```

Where `a.confirmCh` changes type from `chan bool` to `chan PermissionAction`.

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

// Deprecation warning
if len(cfg.CLI.ConfirmTools) > 0 {
    slog.Warn("vv: confirm_tools is deprecated; use permission_mode instead")
}
```

The `WrapToolRegistry` option changes:

```go
var permissionExec *permissionExecutor

initResult, err := setup.Init(cfg, &setup.Options{
    WrapToolRegistry: func(r *tool.Registry) tool.ToolRegistry {
        pe := cli.WrapRegistryWithPermission(r, permissionMode)
        permissionExec = pe
        return pe
    },
    // ...
})

// Pass permissionExec to App so /permission and wireConfirmFn can use it.
app := cli.New(initResult.SetupResult.Dispatcher, cfg, initResult.PersistentMem,
    cliInteractor, initResult.Compactor,
    cli.WithPermissionExecutor(permissionExec))
```

Use a functional option `WithPermissionExecutor` on `App` to inject the reference cleanly.

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

### 3.5 Session State (vv/cli/permission.go, in permissionExecutor)

| Field | Type | Description |
|-------|------|-------------|
| mode | PermissionMode | Active permission mode |
| sessionAllowed | map[string]bool | Tools approved via "Allow Always" |

Cleared when: session ends, or `/permission <mode>` changes the mode.

## 4. API Contracts

No HTTP API changes. Permission modes are CLI-only (per requirement A3/PERM-09). The HTTP mode path in `main.go` does not use `WrapToolRegistry` and is unaffected.

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
- Add `PermissionMode` type, constants, `IsValidPermissionMode()` function
- Add `PermissionMode` field to `CLIConfig`
- Add `omitempty` to `ConfirmTools` yaml tag

**Task 2.2:** Add env var and default handling to `Load`
- File: `vv/configs/config.go`
- Read `VV_PERMISSION_MODE` env var
- Set default to `"default"` in `applyDefaults`
- Validate mode value; return error on invalid
- Log deprecation warning if `ConfirmTools` is non-empty

**Task 2.3:** Add `--permission-mode` CLI flag
- File: `vv/main.go`
- Add flag parsing; apply override after config load

**Task 2.4:** Unit tests for config loading
- Test default value, env var override, invalid value error, deprecation warning

### Phase 3: Permission Executor (vv module)

**Task 3.1:** Implement `permissionExecutor`
- File: `vv/cli/permission.go` (new file, replaces `confirm.go`)
- Implement `permissionExecutor` struct with `Execute`, `SetMode`, `Mode` methods
- Implement `WrapRegistryWithPermission` constructor
- Follow the decision flowchart from the requirement

**Task 3.2:** Remove old `confirmingExecutor`
- Delete or replace `vv/cli/confirm.go` contents
- The old `WrapRegistry` function is removed; callers updated

**Task 3.3:** Unit tests for permission executor
- File: `vv/cli/permission_test.go` (replaces `confirm_test.go`)
- Test cases for each mode (default, accept-edits, auto, plan)
- Test session-allowed ("Allow Always") behavior
- Test mode switching clears session-allowed set
- Test delegation (Register, Get, List, Merge, Unregister)
- Test context cancellation during confirmation

### Phase 4: CLI Integration (vv module)

**Task 4.1:** Update confirmation dialog to three options
- File: `vv/cli/cli.go`
- Change `huh.Confirm` to `huh.Select[string]` with Allow/Allow Always/Deny
- Change `confirmCh` from `chan bool` to `chan PermissionAction`
- Update form completion handler to read string action

**Task 4.2:** Wire `permissionExecutor` into `App`
- Files: `vv/cli/cli.go`, `vv/main.go`
- Add `permissionExec` field to `App`
- Add `WithPermissionExecutor` functional option
- Update `wireConfirmFn` to wire the three-option confirm function
- Update `main.go` to pass `permissionExec` to `App`

**Task 4.3:** Implement `/permission` command
- File: `vv/cli/memory.go`
- Add `/permission` handler in `handleCommand`
- Show current mode (no args) or switch mode (with arg)

**Task 4.4:** Update `handleCtrlC` for new channel type
- File: `vv/cli/cli.go`
- Send `PermissionDeny` instead of `false` on Ctrl+C during confirmation

### Phase 5: Cleanup

**Task 5.1:** Remove old `confirm.go` file
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
