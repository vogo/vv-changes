# Code Review: `ask_user` Tool for Agent-Initiated User Interaction

## Summary

The implementation is well-structured and follows existing codebase patterns closely. The layered architecture (interface in `vage`, implementations in `vv`) is clean, and the test coverage is solid. Below are the issues found, organized by severity.

---

## Issues Found

### 1. [BUG / Medium] HTTP mode never wires `HTTPInteractor` into agents

**Files:** `vv/main.go` (lines 134-135, 156)

In HTTP mode, `main.go` sets `interactor = askuser.NonInteractiveInteractor{}` (line 135), then passes it to `setup.Init`. The `InteractionStore` is created later (line 156) and passed to `Serve`, but no `HTTPInteractor` is ever constructed or wired into the agents. As a result:

- All agents in HTTP mode receive the `NonInteractiveInteractor`, so `ask_user` always returns the fallback message.
- The `POST /v1/interactions/{interactionID}/respond` endpoint exists but no agent ever creates interactions, making it dead code.
- The `HTTPInteractor` struct and its tests work correctly in isolation but are never used in production.

The design doc acknowledges that the sync endpoint should use `NonInteractiveInteractor`, but streaming endpoints should use `HTTPInteractor`. The current code does not differentiate. This is intentional per the design doc's note that `emitFn` is "set per-request for streaming", but the wiring is missing.

**Status:** Not fixed -- this requires architectural decisions about per-request interactor injection that go beyond a code review fix. Flagged for follow-up.

### 2. [BUG / Low] Race condition in `handleInteractionRespond` error classification

**File:** `vv/httpapis/http.go` (lines 206-217)

When `store.Respond()` fails, the handler calls `store.Get()` to determine whether to return 404 or 409. Between these two calls, the cleanup goroutine could remove the interaction, causing an "already responded" error to be misclassified as 404. 

**Fix applied:** Use `strings.Contains` on the error message to distinguish error types atomically, avoiding the TOCTOU race.

### 3. [QUALITY / Low] Unnecessary channel allocation in `newModel`

**File:** `vv/cli/cli.go` (line 166)

`askUserCh: make(chan string, 1)` is created in `newModel`, but `handleAskUserRequest` (line 747) immediately overwrites it with `m.app.interactor.respCh`. The initial channel is never used.

**Fix applied:** Removed the initial allocation; the field is set when needed.

### 4. [QUALITY / Low] `InteractionStore.timeout` is unexported but accessed from `HTTPInteractor`

**File:** `vv/httpapis/askuser.go` (line 30)

`h.store.timeout` is accessed directly from `HTTPInteractor.AskUser`. Since both types are in the same package, this compiles fine, but it couples `HTTPInteractor` to `InteractionStore`'s internal field. This is acceptable for same-package access in Go but noted for awareness.

**Status:** No change needed -- same-package access is idiomatic Go.

### 5. [QUALITY / Positive] Well-designed test coverage

The test suite is thorough:
- `askuser_test.go` covers valid input, empty question, invalid JSON, timeout, non-interactive, tool def, registration, and duplicate registration.
- `interactions_test.go` covers CRUD operations, double-respond, not-found, cleanup, and HTTPInteractor with both success and timeout paths.

---

## Changes Applied

1. Fixed race condition in `handleInteractionRespond` by using error string inspection instead of a second store lookup.
2. Removed unnecessary channel allocation in `newModel`.
