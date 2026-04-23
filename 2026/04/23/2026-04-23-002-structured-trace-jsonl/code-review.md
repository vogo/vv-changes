# Code Review — P1-5 Structured Trace JSONL (reviewer phase)

Scope: the 11 files listed in the reviewer brief. Design document:
`design.md` (refined). Tests and lint pass (`make test`, `golangci-lint run ./...`).

## Summary

The implementation faithfully follows the refined design. Channel ownership,
rotation logic, session-id sanitization, file permissions, and lifecycle
wiring all match what the design calls out, and I could not find a correctness
or concurrency defect in the hook path. Below are the items I checked and the
few rough edges worth logging.

## Correctness / concurrency — verified clean

- **`JSONLHook.Stop` vs `hook.Manager.Stop`** — Verified
  `vage/hook/manager.go:132-145`: `Manager.Stop` only calls `h.Stop(ctx)` per
  async hook and does *not* touch the channel. `JSONLHook.Stop` owns the
  channel and `close`s it exactly once via `stopOnce`. No double-close.
- **Send-to-closed-channel risk** — `hook.Manager.Dispatch` does a
  `select { case h.EventChan() <- event: default: ... }` send; the `default`
  clause does *not* prevent a panic if the channel is already closed. This is
  safe in practice because `main.shutdownInit` runs *after* the mode loop
  returns (CLI `app.Run`, `httpapis.Serve` with `server.Shutdown`, MCP
  `Serve`), so no TaskAgent is mid-run when `Stop` closes the channel. The
  `-p` / `-eval` paths also call `shutdownInit` only after `cli.RunPrompt` /
  `vveval.RunCLI` have returned. Correct as implemented — worth preserving
  with a comment (done by the existing comments in `shutdownInit` and
  `InitResult.Shutdown`).
- **`Init` error path releases hook cleanly** — when `New(...)` fails after
  `buildHookManager` succeeded, `traceShutdown(context.Background())` is
  invoked; `mgr.Stop → JSONLHook.Stop → close(h.ch) → wg.Wait → Sync+Close`
  runs correctly because the consumer is already started. No goroutine leak.
- **Stop idempotence** — `stopOnce.Do` + test coverage (`Test_StopIsIdempotent`)
  confirms it.
- **File size accounting** — `openSession` seeds `written = info.Size()`, so a
  file already on disk from a prior run does not get double-rotated or
  truncated. `O_CREATE|O_WRONLY|O_APPEND` is the correct flag set.
- **Rotation pre-write check** — `sf.written+len(line) > max` is evaluated
  before write, keeping each file bounded by `max` bytes.
- **Consumer `continue` on error** — marshal / open / write errors `slog.Warn`
  and skip the event rather than returning from the goroutine. Consistent with
  the design's "never drop the whole pipeline" stance.
- **`MaxFileBytes == 0` disables rotation** — `if h.maxFileBytes > 0 && ...`
  guard is correct. `Test_RotationDisabledKeepsSingleFile` exercises this.
- **Session-id path-traversal protection** — `sanitizeSessionID` strips `/`
  (regex `[^a-zA-Z0-9._-]`), so `abc/../etc` → `abc_.._etc`. A bare `..` sid
  becomes literal filename `..jsonl`, not a directory traversal. Test covers
  both.
- **File permissions** — dir `0o700`, file `0o600`. Matches requirement;
  `Test_FilePermissions` asserts.
- **`getHookManager(opts)` after `opts.HookManager` mutation** — `Init`
  assigns `opts.HookManager = hookManager` *before* calling `New(...)`.
  Every downstream `getHookManager(opts)` inside `New` sees the same handle.
  Ordering is correct.

## Minor observations (not fixed — out of scope or intentional)

### 1. `MaxFileBytes` YAML default is 0, which means "no rotation"

Design doc says "default 64 MiB; 0 = disable rotation". The implementation
coerces negative → 64 MiB and leaves 0 as "disabled". Because a YAML-unset
`max_file_bytes` serializes to Go zero (`0`), this means **the out-of-the-box
default is no rotation**, not 64 MiB.

Options:
- (a) Apply 64 MiB default in `applyDefaults(cfg)`, lose the ability to
  explicitly disable rotation via YAML.
- (b) Keep as-is. Users who want 64 MiB must set it.

The design has an inherent inconsistency here (you can't distinguish
YAML-unset from explicit-zero with a plain `int64`). Developer picked (b),
which matches Go idioms. **Not fixing in this phase** — this is a design
clarification for the documenter to capture in the PRD (either document "users
must set `trace.max_file_bytes: 67108864` explicitly" or change the field to
`*int64` in a follow-up). Calling it out so the call doesn't silently drift.

### 2. `buildHookManager` starts the manager eagerly inside itself

The refined design (§Lifecycle) describes `buildHookManager` as returning a
non-started manager and having `setup.Init` call `Start` separately. The
developer inlined `Start` into `buildHookManager` and returns the shutdown
closure directly. Functionally equivalent and arguably cleaner — one less
step for the caller to get wrong. Accepted as an implementation detail
deviation.

### 3. `Init` mutates caller-provided `opts`

`Init` sets `opts.HookManager = hookManager` on the *caller's* Options struct
(after promoting `nil` to a fresh `&Options{}`). All call sites in `main.go`
pass a fresh `&setup.Options{}` per call, so this is effectively a no-op
aliasing. Still, it is a surprise for readers. Not worth fixing: documenting
it in the field comment on `Options.HookManager` would be overreach — the
field is already marked `optional: pre-built event bus`. If `Init` grows a
second caller, revisit.

### 4. Two `vv` instances on the same project share a file

Because `ProjectHash` is deterministic on cwd and session ids can
collide across processes (e.g. both use `"default"`), concurrent writers with
`O_APPEND` would interleave lines. JSONL lines are written atomically at the
`write(2)` layer on POSIX for sizes below `PIPE_BUF` (4096 bytes), so lines
stay intact, but rotation accounting diverges between processes. Out of scope
for P1-5 (cross-process coordination is the P1-6 SQLite story).

### 5. `Start(ctx)` does not use `ctx`

`JSONLHook.Start` takes a ctx by interface contract but ignores it — the
consumer exits only on channel close. Matches the design
(`"The ctx is not retained..."`) and `hook.Manager.Start` does not rely on
ctx cancellation to terminate child goroutines either. Intentional; no
action.

## Tests and lint

- `cd vv && make test` — all packages pass. `traces/tracelog` coverage 75.7%
  (the uncovered lines are mostly the "stat failed after open" and
  "close-on-stop-with-error" error branches — acceptable).
- `cd vv && golangci-lint run ./...` — `0 issues.`
- `Test_FileRotation` stressed with `-count=10` — stable, no flake.
- No `vv/vv` binary leftover.

## Decision

No code changes applied. The implementation matches the design, tests cover
the checklist items, and the observations above are either intentional
deviations, design-level questions best handled by the documenter, or
out-of-scope follow-ups.
