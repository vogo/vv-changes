# Code Review: Tool Permission Modes

## Files Reviewed

### vage module
- `vage/schema/tooldef.go` -- Added `ReadOnly` field
- `vage/tool/read/read_tool.go` -- Set `ReadOnly: true`
- `vage/tool/glob/glob_tool.go` -- Set `ReadOnly: true`
- `vage/tool/grep/grep_tool.go` -- Set `ReadOnly: true`
- `vage/tool/askuser/askuser.go` -- Set `ReadOnly: true`

### vv module
- `vv/configs/config.go` -- `PermissionMode` type, validation, `CLIConfig` extension, env var override, defaults
- `vv/configs/config_test.go` -- Tests for permission mode loading, env override, invalid mode, accept-edits
- `vv/cli/permission.go` -- `PermissionState`, `permissionExecutor`, `WrapRegistryWithPermission`, `/permission` command handler
- `vv/cli/permission_test.go` -- Tests for all modes, session-allowed, shared state, delegation, context cancellation
- `vv/cli/cli.go` -- `App` struct changes, `WithPermissionState`, `wireConfirmFn`, confirmation dialog (3-option huh.Select), `confirmCh` type change, Ctrl+C handling
- `vv/cli/memory.go` -- `/permission` routing
- `vv/cli/messages.go` -- `confirmRequestMsg` (unchanged, already correct)
- `vv/main.go` -- `--permission-mode` flag, `PermissionState` creation, wiring

## Verdict

The implementation is solid and faithfully follows the design document. The code is well-structured, well-tested, and the permission logic is correct. All four modes (default, accept-edits, auto, plan) work as specified. The shared `PermissionState` pattern cleanly solves the multi-agent consistency requirement.

## Issues Found

### Issue 1: wireConfirmFn called before program is assigned (Low -- fragile ordering)

**File:** `vv/cli/cli.go`, lines 105 and 111

`wireConfirmFn()` is called on line 105, but `a.program` is assigned on line 111. The closure inside `SetConfirmFn` captures `a` by pointer and accesses `a.program.Send(...)`. This works at runtime because the closure is only invoked after the Bubble Tea event loop starts (i.e., after `a.program` is set). However, the ordering is fragile -- a future refactor that triggers a tool call earlier (e.g., during initialization) would cause a nil pointer dereference.

**Recommendation:** Move `wireConfirmFn()` after `a.program = p` for defensive correctness.

**Severity:** Low. Currently safe due to execution ordering, but fragile.

**Fix applied:** Yes -- moved `wireConfirmFn()` after `a.program = p`.

### Issue 2: Missing `RegisterIfAbsent` on mockRegistry (Informational)

**File:** `vv/cli/permission_test.go`

The `mockRegistry` implements the `ToolRegistry` interface with `Register` but the production code uses `RegisterIfAbsent` on `*tool.Registry`. The test mock only needs the interface methods (`Register`, `Unregister`, `Get`, `List`, `Merge`, `Execute`), so this is fine. The compile-time check `var _ tool.ToolRegistry = (*mockRegistry)(nil)` on line 61 validates interface compliance.

**Severity:** Informational. No action needed.

### Issue 3: `action` variable scoping in handleConfirmRequest (Informational)

**File:** `vv/cli/cli.go`, line 857

The `action` variable is declared but used only as a binding target for `huh.NewSelect`. The variable is read later via `f.GetString("confirm")` rather than through the pointer binding. This is the standard huh pattern and works correctly, but the `Value(&action)` binding is technically redundant since `GetString` is used to retrieve the value. This is consistent with how huh forms work (the binding and `GetString` both work), so no issue.

**Severity:** Informational. No action needed.

## Positive Observations

1. **Clean separation of concerns**: `PermissionState` is properly extracted as shared mutable state, and `permissionExecutor` is kept unexported while `PermissionState` is the public API surface.

2. **Thorough test coverage**: The permission_test.go covers all modes, session-allowed behavior, AllowAlways, Deny, error handling, context cancellation, shared state across executors, delegation, and the WrapRegistryWithPermission constructor. Config tests cover default, YAML, env var override, invalid mode, and accept-edits.

3. **Correct `IsValidPermissionMode` using `slices.Contains`**: Cleaner than the loop shown in the design doc.

4. **Proper mutex usage**: All `PermissionState` methods consistently acquire the lock, and `SetConfirmFn` properly iterates under the lock.

5. **Backward compatibility**: The deprecated `confirm_tools` field is preserved with `omitempty` and a deprecation warning is logged.

6. **Config precedence chain**: Flag > env > YAML > default is correctly implemented across `main.go` and `configs.Load`.
