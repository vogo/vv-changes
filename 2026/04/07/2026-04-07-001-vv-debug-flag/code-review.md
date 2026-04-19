# Code Review: vv `--debug` Flag

Reviewed against `design.md` (revised).

## Files Reviewed

- `aimodel/intercept.go`, `aimodel/intercept_test.go`
- `vage/largemodel/debug.go`, `vage/largemodel/debug_test.go`
- `vv/debugs/{sink,format,redact,context,env,toolregistry,debugs_test}.go`
- `vv/configs/config.go` (+ tests)
- `vv/setup/setup.go`
- `vv/dispatches/execute.go`, `vv/dispatches/intent.go`
- `vv/httpapis/http.go`, `vv/httpapis/requestid.go`
- `vv/main.go`

## Summary

Implementation closely follows the design. Default-off path is clean: no
middleware, no decorator, no goroutines. Two insertion points
(`largemodel.DebugMiddleware` and `debugs.DebuggingToolRegistry`) plus the
additive `aimodel.InterceptStream` helper match the architecture in §2-3.
Decorator order is correct: debug is outermost (after truncation) so records
reflect what the agent actually saw.

A handful of correctness, race-safety, and hygiene issues were found and
fixed in-place. Details below.

## Findings

### High

**H-1. Race in streaming debug accumulator** *(fixed)*
`vage/largemodel/debug.go` `chatCompletionStream` shared `contentBuf`,
`toolCalls`, `finish`, `lastUsage` between `onChunk` (called from the
consumer's `Recv` goroutine) and `onDone` (called from either `Recv` on
error/EOF, or `Close` on early termination). `aimodel.Stream` only guarantees
that `Recv` and `Close` are safe to call concurrently — meaning `onDone` from
`Close` may run while `onChunk` from `Recv` is still touching the
accumulator. Added a `sync.Mutex` guarding all accumulator reads/writes.

**H-2. `isEOF` brittle string comparison** *(fixed)*
`isEOF(err) = err.Error() == "EOF"` would miss any wrapped error and
mis-handle implementations that override `Error()`. Replaced with
`errors.Is(err, io.EOF)`.

### Medium

**M-1. File sink never closed** *(fixed)*
`debugs.Sink.Close()` exists but `vv/main.go` never invokes it. Added
`defer debugSink.Close()` after construction. Note: paths that exit via
`os.Exit` still skip the defer (Go semantics) — acceptable since the OS
reclaims the fd; structured shutdown would require larger refactoring and is
out of scope for this change.

**M-2. No streaming-path test for `DebugMiddleware`**
`vage/largemodel/debug_test.go` exercises only the non-streaming branch and
the noop sink. The streaming consolidation (chunk accumulation, EOF, early
close, mid-stream error) is exercised only indirectly via
`aimodel/intercept_test.go`. A streaming test in `largemodel` is not added
because `aimodel.Stream` has no public constructor outside the `aimodel`
package. Recommendation: expose a small test helper (e.g.
`aimodel.NewTestStream`) in a follow-up so the consolidation logic can be
tested at the largemodel boundary.

**M-3. Source classification heuristic flags agent-as-tool as `mcp`**
`debugs.classifySource` falls through to `"mcp"` for any non-builtin name,
which mis-labels agent-as-tool calls. Design accepts this as heuristic.
Consider plumbing `schema.ToolDef.Source` (if available) through `Get(name)`
in a follow-up; left as-is per design scope.

### Low / hygiene

**L-1. `gofumpt` whitespace in `requestFields` map literal** *(fixed)*
Aligned column padding caused gofumpt diff. Trimmed to canonical spacing.

**L-2. `largemodel.NewCorrelationID` is dead exported code**
`vage/largemodel/debug.go` exports `NewCorrelationID` but the only call
sites use `Sink.NewCorrelationID()` (the `vv/debugs.SinkAdapter` route).
Harmless; left in place since the design names it as part of the
`DebugSink` contract surface.

**L-3. `httpapis/requestid.go` shim wrappers**
`cryptoRandRead` / `hexEncode` are one-line indirections over `crypto/rand`
and `encoding/hex`. Cosmetic; no functional impact.

**L-4. `formatRecord` prints `Fields` map with `%v`**
Multi-line slices render as opaque Go syntax. Acceptable for debug; design
does not require structured rendering.

**L-5. Tool decorator captures `Args` without redaction**
Matches design R-7 (no auto-redaction of tool/prompt content). Documented in
the design; flagged here for future awareness if a tool ever takes raw
secrets in args.

**L-6. `Sink.Close` returns `error` but slog/writer backends have nothing
to flush.** Only the file backend benefits. Fine.

### Verified OK (no issues)

- Default-off behavior: when `cfg.Debug=false`, sink is `nil`, middleware is
  not constructed, tool decorator returns the inner registry unchanged
  (`NewDebuggingToolRegistry` short-circuits).
- Always-on http request-id middleware (G-design) does not depend on
  `cfg.Debug` and adds a single UUID alloc per request.
- Decorator order in `setup.New`: truncate → debug (outermost), correct per
  R-8.
- LLM client wrapping happens in `Init` BEFORE compactor closure and BEFORE
  `setup.New`, covering compactor + intent + plan-gen + sub-agent calls.
- `WithAgentName` is set at every dispatch site listed in the design
  (direct, stream-direct, intent, explorer, planner, classify-direct).
- `aimodel.InterceptStream` correctly uses `sync.Once` to guarantee
  `onDone` fires exactly once across EOF, mid-stream error, and Close.
- Nil safety: `Sink.Emit` short-circuits on `nil` receiver/backend/record;
  `SinkAdapter.Emit` short-circuits on nil sink; tool decorator returns
  inner unchanged when sink is nil; `largemodel.NewDebugMiddleware(nil)`
  installs `NoopSink`.
- Concurrent emit test (`TestSink_ConcurrentEmit`) passes 50-goroutine
  contention with the mutex-guarded backend.
- Redaction covers `api_key`, `Authorization`, query-string `api-key=`, and
  the four env-var values (matches §4.4).

## Fixes Applied

1. `vage/largemodel/debug.go`: added `sync.Mutex` around streaming
   accumulator (H-1).
2. `vage/largemodel/debug.go`: switched `isEOF` to `errors.Is(err, io.EOF)`
   (H-2).
3. `vage/largemodel/debug.go`: gofumpt-clean map literal (L-1).
4. `vv/main.go`: added `defer debugSink.Close()` after construction (M-1).

## Test & Lint Status

See report in surrounding session.
