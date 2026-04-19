# Design Review: Tool Permission Modes

## Summary

The proposed design is well-structured and closely follows the requirement. The architecture choice -- replacing `confirmingExecutor` with `permissionExecutor` using the same decorator pattern -- is sound and minimally invasive. Below are specific issues and improvements identified by reviewing the design against the actual codebase.

---

## Issue 1: WrapToolRegistry is called per-agent, not once globally

**Severity:** High (correctness)

In `setup.go` (lines 96-99), `WrapToolRegistry` is called **once per dispatchable agent** inside the `for _, desc := range reg.Dispatchable()` loop. The proposed design writes:

```go
WrapToolRegistry: func(r *tool.Registry) tool.ToolRegistry {
    pe := cli.WrapRegistryWithPermission(r, permissionMode)
    permissionExec = pe
    return pe
},
```

This means a separate `permissionExecutor` instance is created for **each agent** (coder, researcher, reviewer, chat). The design only captures the last one into `permissionExec`, so `/permission` mode changes would only affect the last agent initialized, while other agents retain their original mode.

**Fix:** The `permissionExecutor` must share a single permission state across all wrapped registries. Options:
- (A) Extract the mutable state (`mode`, `sessionAllowed`) into a shared `*PermissionState` object, and pass that same pointer to every `permissionExecutor` instance.
- (B) Collect all `permissionExecutor` instances and update them all on mode change.

Option (A) is cleaner and is the recommended approach.

---

## Issue 2: `wireConfirmFn` is currently a no-op placeholder

**Severity:** Medium (incomplete wiring)

The current `wireConfirmFn` at `cli.go:110` is an empty placeholder. The actual confirmation wiring between the `confirmingExecutor` and the TUI dialog is **not yet implemented** -- the `confirmCh` channel exists in `model` but nothing writes `confirmRequestMsg` to the program. The design assumes `wireConfirmFn` currently wires a real function, but in reality the existing confirm flow only works because `WrapRegistry` returns the inner registry unchanged when `confirmTools` is empty (which is the default).

**Fix:** The design should explicitly document that it is completing the wiring that was left as a placeholder. The new `wireConfirmFn` implementation must:
1. Iterate over all `permissionExecutor` instances (or set the `confirmFn` on the shared state).
2. Send `confirmRequestMsg` via `a.program.Send()` and wait on the response channel.

This is actually what the design describes in Section 2.6, but it should acknowledge that this is new functionality, not a modification of working code.

---

## Issue 3: `accept-edits` mode hardcodes tool names instead of using a tool attribute

**Severity:** Medium (design consistency)

The design checks `name == "write" || name == "edit"` to decide what `accept-edits` auto-approves. This is fragile -- if a new file-modifying tool is added, the mode logic needs updating. Since we are already adding `ReadOnly` as a tool attribute, a more consistent approach would be to add a tool category or a second attribute.

However, given the requirement explicitly scopes `accept-edits` to "write" and "edit" tools, and the tool set is small and stable, the hardcoded approach is acceptable for now. The fix is to add a code comment explaining this design choice and noting that if tool categories grow, this should be refactored to use tool metadata.

---

## Issue 4: The `/permission` command is placed in `memory.go`

**Severity:** Low (code organization)

The design places the `/permission` command handler in `memory.go`, which currently handles `/memory` and `/compact` commands. The file name `memory.go` is misleading for permission-related code.

**Fix:** Rename the file to `commands.go` (since it already handles multiple command types), or place the `/permission` handler in the new `permission.go` file and call it from `handleCommand`. The second option is simpler and keeps permission logic self-contained.

---

## Issue 5: Missing validation of `PermissionMode` in `applyDefaults`

**Severity:** Low (robustness)

The design says to set the default in `applyDefaults` and validate in `Load`. But `applyDefaults` runs after env var processing, and validation should happen after all overrides are applied (including future CLI flag override). The design should clarify the exact ordering: YAML parse -> env var override -> applyDefaults (set default if empty) -> validate.

**Fix:** Add explicit validation as a separate step after `applyDefaults` in `Load`, not interleaved with it.

---

## Issue 6: `confirmCh` type change needs careful migration

**Severity:** Medium (correctness)

The `confirmCh` channel type changes from `chan bool` to `chan PermissionAction`. Several places reference it:
- `model.confirmCh` field and initialization in `newModel`
- `handleCtrlC` sends `false` (needs to send `PermissionDeny`)
- The form completion handler in `Update`

The design addresses these in Tasks 4.1 and 4.4, but the `handleCtrlC` case has a `select` with `default` that could silently drop the deny if the channel is full. Since the channel is buffered (size 1), this should be fine, but worth noting.

---

## Issue 7: `WrapRegistryWithPermission` default always returns `PermissionAllow`

**Severity:** Low (clarity)

The design's constructor sets a default `confirmFn` that always allows. The comment says "App wires real fn", but in HTTP mode no wiring happens. This is safe because HTTP mode does not call `WrapToolRegistry` at all (per `setup.go`), but the design should document this explicitly.

---

## Issue 8: The design does not address the `-p` (single prompt) mode

**Severity:** Low (completeness)

When `vv` runs in `-p` (non-interactive) mode, `setup.Init` is called without `WrapToolRegistry` (see `main.go:106-113`). The design does not mention this path. Since `-p` mode has no interactive confirmation, it should implicitly use `auto` mode. This is already the case (no wrapping = no permission checks), but worth documenting.

---

## Issue 9: Missing `IsValidPermissionMode` helper function

**Severity:** Low (API completeness)

The design references `configs.IsValidPermissionMode(mode)` in Section 2.5 but does not define it. Add it alongside the `ValidPermissionModes` slice:

```go
func IsValidPermissionMode(m PermissionMode) bool {
    for _, v := range ValidPermissionModes {
        if v == m { return true }
    }
    return false
}
```

---

## Issue 10: Shared state needs to be in `vv/cli` not `vv/configs`

**Severity:** Low (module boundary)

The `PermissionMode` type should live in `vv/configs` (configuration concern), but the runtime state (`sessionAllowed`, mutable `mode`) should live in `vv/cli` (runtime concern). The design already does this correctly -- just confirming the module boundary is right.

---

## Recommended Changes Summary

| # | Issue | Action |
|---|-------|--------|
| 1 | Per-agent wrapping creates multiple executors | Extract shared `PermissionState` struct |
| 2 | `wireConfirmFn` is a no-op | Acknowledge and complete the wiring |
| 3 | Hardcoded tool names in accept-edits | Add explanatory comment; acceptable for now |
| 4 | `/permission` in memory.go | Place handler in `permission.go`, call from `handleCommand` |
| 5 | Validation ordering | Validate after all overrides applied |
| 6 | `confirmCh` type migration | Already addressed; confirm Ctrl+C path |
| 7 | Default confirmFn in HTTP mode | Document that HTTP path skips wrapping entirely |
| 8 | `-p` mode not mentioned | Document as implicitly `auto` (no wrapping) |
| 9 | Missing `IsValidPermissionMode` | Add the helper function definition |
| 10 | Module boundary | Confirmed correct as-is |
