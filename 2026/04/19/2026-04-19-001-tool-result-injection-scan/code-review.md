# Code Review: Tool-Result Injection Scanning (P0-3)

Reviewer: reviewer sub-agent, 2026-04-19.

Scope: the files listed in the session's execution plan. Review is against `design.md` / `requirement.md`.

---

## Summary of verdict

One **correctness bug** that directly breaks AC-3.1 / AC-3.2 (log-only action does not actually emit a `guard_check` event or `slog.Warn`). One **minor observability bug** (snippet byte-truncation can produce invalid UTF-8). A third item I briefly flagged as out-of-scope (`vv/go.mod` aimodel bump) was **withdrawn** after verifying it is a required dependency-consistency fix. Everything else is either correct-as-designed, test-level nits, or deferrable follow-ups.

---

## Accepted (applied in-tree)

### A1. Log-only path does not emit an event (AC-3.1, AC-3.2 regression)

`guard.RunGuards` only inspects `result.Action`; when a guard returns `ActionPass` with `Violations` (the ToolResultInjectionGuard's log outcome), `RunGuards` swallows both `Violations` and `GuardName`, returning a bare `Pass()`.

Consequence: in `taskagent.runToolResultGuards`, the `case ActionPass: if len(gres.Violations)==0 { return }` branch always returns without emitting the log event, so the log action is effectively silent — exactly the case AC-3.1/AC-3.2 are written for.

Existing tests do not catch this because `TestAgent_Run_ToolResultGuard_LogPassThrough` only asserts content is unchanged, not that an event was dispatched.

**Fix:** In `guard/chain.go::RunGuards`, accumulate violations on `ActionPass` as well. On exit, if no rewrite occurred but any pass-violations were accumulated, return a Pass-result carrying those violations and (for single-guard chains) a stable `GuardName`. This is a strict addition — every existing caller already ignores `Violations`/`GuardName` on Pass.

Also add a regression test at the taskagent layer that asserts a `guard_check` event with `Action=="log"` is emitted.

### A2. Snippet truncation may produce invalid UTF-8

`buildGuardCheckEvent` does `snippet[:snippetMax]` where `snippetMax=200`. If the 200-th byte falls mid-rune, the JSON-serialized `snippet` field is invalid UTF-8. Low impact (observability only) but cheap to fix.

**Fix:** Walk back to the previous rune boundary before appending `"..."`.

### A3. (Withdrawn) `vv/go.mod` aimodel bump

Initially flagged as unrelated scope creep. Investigation: `vage/go.mod` already requires `aimodel v0.3.0` (pre-existing), so `go mod tidy` correctly resolves `vv` to the same version. The bump is a dependency-consistency fix, not scope creep. **No action.**

---

## Rejected / not a problem

### R1. `maxToolResultSeverity` type-asserts to `*ToolResultInjectionGuard`

Flagged briefly as a narrow coupling. But the severity concept *is* specific to this guard type (other `guard.Guard` implementations don't have a `MaxSeverity` concept), so reflecting on the concrete type is justified. No fix.

### R2. Rewrite chain would re-scan quarantine wrapper

If multiple tool-result guards were configured (`ToolResultInjectionGuard` + user-defined), the wrapped text could hypothetically re-match on the second pass. In practice no callers chain multiple tool-result guards. Deferring until a real use case appears. No fix.

### R3. `DirectionOutput → != DirectionInput` change in `PromptInjectionGuard.Check`

Design §3.1.1 explicitly requires this hardening. In the current two-value direction world, `== DirectionOutput` and `!= DirectionInput` match the same set for `{Input, Output}`. For the new `DirectionToolResult`, the hardened form correctly stays Pass. No backward-compat concern.

### R4. `PatternRule` left unchanged, Severity lives on `SeveredPatternRule`

Design §3.1.1 and §10 explicitly forbid adding a `Severity` field to `PatternRule`. Verified the diff does not touch the struct. Good.

### R5. Go's `regexp` is RE2 — no ReDoS

All 20 default patterns compile at init via `regexp.MustCompile` and execute in linear time under RE2. No catastrophic backtracking risk. No fix.

### R6. `MaxScanBytes` zero-copy slice

`content = content[:g.maxScanBytes]` is a header-level reslice; zero allocation in the happy path. Good.

### R7. `result.Content` slice defensive copy on Rewrite

`runToolResultGuards` makes a new slice before mutating `out.Content[textIdx]`. Original `result.Content` is untouched. Good.

---

## Deferred (noted, not applied now)

### D1. `EventGuardCheck` direction disambiguation

`GuardCheckData` has no `direction` field. If a future version emits this event from input/output guards as well, consumers would not be able to distinguish channels without inspecting tool metadata. Design explicitly defers this to v2. OK for now.

### D2. `configs.ToolResultInjectionConfig` has no unit test

AC-4.2 is implicitly covered by `setup.buildToolResultGuards`, but there is no direct test for `IsEnabled()`, action parsing, or severity parsing defaults. Would be a small addition but marked as tester-phase work.

### D3. Quarantine template not configurable

Design §4.2 is explicit: `QuarantineTmpl` stays private. YAGNI. OK.

### D4. `persona_hijack` pattern at Low severity matches phrases like "act as quickly as possible"

Known false-positive class. Log-only default + severity tier accepts this.

---

## Verification

After applying A1 and A2 (A3 withdrawn):

- `go test ./guard/ ./agent/taskagent/ ./schema/` in vage — pass
- `go test ./configs/ ./setup/ ./registries/` in vv — pass
- Full `go test ./...` in both vage and vv — pass
- `golangci-lint run ./...` in both vage and vv — clean
