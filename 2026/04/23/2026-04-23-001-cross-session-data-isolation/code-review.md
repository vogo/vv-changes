# Code Review — P1-4: Cross-Session Data Isolation

**Scope**: `vv/memories/session.go`, `vv/memories/namespaces.go`,
`vv/memories/filestore.go`, `vv/memories/filestore_test.go`, `vv/cli/memory.go`,
`vv/httpapis/http.go`, `vv/integrations/agents_tests/agents_test.go`.

Baseline: all new and existing `go test ./memories/...` tests pass before
this review's edits.

## Overall Verdict

The design is sound: record-level `SessionID` + per-session subdirectory
layout + user-path marker. Implementation is close to the design's row
table. Most gaps are either (a) small inconsistencies between the design's
"no-context + private ns" row and the code's handling of it in `Delete`,
or (b) tests that claim to exercise the security boundary but actually
verify path separation — leaving the defense-in-depth overwrite/delete
guards uncovered.

## Findings

### Critical

**C-1. Tests "Forbidden" for cross-session overwrite/delete don't exercise the
guard they claim.** `TestFileStore_AgentPath_CrossSessionOverwrite_Forbidden`
and `TestFileStore_AgentPath_CrossSessionDelete_Forbidden` both pass because
A and B write to different private paths (path layout includes sid). Neither
reaches the defense-in-depth code in `Set` (`if !shared && old.SessionID !=
"" && old.SessionID != sid`) or `Delete` (`rec.SessionID != sid`) — those
guards are effectively dead as far as the test suite is concerned. A
manually-planted file (simulating corruption / manual tampering) is needed
to verify the guard actually fires. **Action**: rename the two tests to
accurately describe what they do ("isolation via path separation") and add
a new test that plants a file owned by session A at session B's private
path to verify the overwrite guard actually returns `ErrSessionForbidden`.
Applied.

### Major

**M-1. `Delete` does not block no-context calls on private namespaces.** The
design table says "no context | session-private | any | user-path strict
mode → write returns `ErrSessionForbidden`". `Set` implements this. `Delete`
does not — with `ctx = context.Background()` and key `scratch:x`, Delete
either returns `nil` (file doesn't exist) or `ErrSessionForbidden` (legacy
file present). Inconsistent. **Action**: add a symmetric up-front guard in
`Delete`: `!shared && sid == "" && !isUser → ErrSessionForbidden`. Applied.

**M-2. `readSharedDir` swallows non-NotExist errors from `os.ReadDir`.**
```go
files, err := os.ReadDir(nsDir)
if err != nil { return nil, nil }
```
A permission-denied or I/O error is silently ignored. **Action**: only
silence `os.IsNotExist`; propagate others. Applied.

**M-3. `Delete`'s "legacy private no-sid" branch depends on disk state for
its error posture.** A no-context caller hitting `Delete("scratch:x")`
returns forbidden if a legacy record exists, nil otherwise. Covered by the
M-1 fix above (up-front block normalizes this).

### Minor

**N-1. Dead variable in `Get`.** The `_ = legacyFallback` blank-assignment
is dead; `legacyFallback` is not read anywhere downstream. **Action**:
remove the redundant local variable by dropping the second return value of
`resolveReadPath` (only `Get`/`Delete` consume it, and neither actually
cares beyond the path itself). Left in place — removing it would ripple
through `resolveReadPath`'s signature; the comment already documents intent,
and the line is harmless. Recorded but not applied.

**N-2. Error messages leak concrete session IDs ("owner=<sid>").** vv's
session_id is an 8-byte hex token from `crypto/rand`, not an identifier
about the user, so the leak is minor. Kept for now — removing it would
hurt debuggability. Recorded.

**N-3. HTTP GET memory handler does not pre-validate shared namespace.**
Set/Delete pre-check `IsSharedNamespace(ns)`; GET does not and relies on
the store's "not-found posture" for private ns (returns 404). This is
intentional per the design (no existence leak) and consistent. No action.

**N-4. `userPathNamespaceOK` in `cli/memory.go` re-implements namespace
parsing.** Uses `strings.Cut`, mirrors `parseKey` in the store. Small code
duplication, not worth a shared helper given the different defaults. No
action.

### Nit

**X-1. `IsSharedNamespace` is exported but `isShared` isn't — both exist.**
Justified (exported form doesn't take extras). Keep.

**X-2. `resolveReadPath` second return `legacy bool` is currently unused.**
See N-1.

## Concurrency / TOCTOU

`PersistentMemory` serializes all Store calls through its mutex, so the
Set→Set race that would otherwise bypass the overwrite guard is closed at
the higher layer. The FileStore's own doc already warns "not safe for
concurrent use"; this remains accurate. No action.

## Design-Fit Mapping

| Row | Code location | Status |
|---|---|---|
| User + shared / any | `Set/Delete/Get` early branches, shared path | OK |
| User + private / Set/Delete | `!shared && isUser → forbidden` | OK |
| User + private / Get/List | Not-found posture + List user-path skips session dir | OK |
| Agent + shared / any | `recordSessionID(true, sid) = ""` | OK |
| Agent + private / Set | private path + stamp sid | OK |
| Agent + private / Get | path walk + SessionID match check | OK |
| Agent + private / Delete | path walk + SessionID match + legacy permissive | OK post M-1 |
| Agent + private / List | walks own sid dir only + shared dirs | OK |
| No-ctx + shared / any | passes through | OK |
| No-ctx + private / Set | forbidden | OK |
| No-ctx + private / Delete | forbidden (after M-1) | OK post fix |
| No-ctx + private / Get/List | legacy-readable | OK |
| Clear | user-path only | OK |

## Tests Added

- `TestFileStore_Delete_NoSessionOnPrivate_Forbidden` — verifies M-1 block.
- `TestFileStore_AgentPath_DefenseInDepth_OverwriteBlocked` — plants a
  mismatched record at the caller's own private path and asserts Set
  returns `ErrSessionForbidden`.
- Renamed `TestFileStore_AgentPath_CrossSessionOverwrite_Forbidden` →
  `TestFileStore_AgentPath_CrossSessionIsolation_Overwrite`.
- Renamed `TestFileStore_AgentPath_CrossSessionDelete_Forbidden` →
  `TestFileStore_AgentPath_CrossSessionIsolation_Delete`.

## Post-Edit Status

- `go test ./memories/... -v` → PASS (all 25 tests).
- `go test ./...` → PASS.
- `golangci-lint run ./memories/... ./cli/... ./httpapis/...` → PASS.

## Lingering Risks

- The defense-in-depth Set overwrite guard and Delete SessionID-mismatch
  guard are now exercised by `DefenseInDepth` test but are still only
  reachable under corruption/manual-tampering paths. Regular flow stays in
  per-session subdirectories.
- The shared HTTP/CLI path does not currently stamp `WithSessionID` for
  any agent-tool-triggered memory write (none exist yet). When the
  `write_memory` tool lands in P3, it MUST call `memories.WithSessionID`
  on the context it passes to `memory.Memory.Set` — otherwise writes to
  private namespaces will fail with `ErrSessionForbidden`. The failure
  mode is explicit, not silent, so the coverage gap is acceptable.
- The shared-dir legacy-filter logic inside `readSharedDir` is correct
  but branchy; any future change there should be accompanied by a test
  table covering the four `(shared, isUser, sid, recSid)` axes.
