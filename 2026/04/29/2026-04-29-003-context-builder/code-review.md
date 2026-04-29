# Context Builder — Code Review

Reviewer: Claude (code-review pass)
Scope: `vage/context/` (new), `vage/agent/taskagent/task.go`, `vage/schema/event.go`
Reference: `design.md` in this folder.

## Summary

The implementation is faithful to the design. Files are licensed, public/private boundaries match the spec, BuildReport / Source contracts line up with `§2`–`§5`, and the TaskAgent integration preserves the original message order and `sessionMsgCount` semantics (verified by reading `loadAndCompressSessionHistory` against `SessionMemorySource.Fetch` line by line).

I found one correctness issue worth fixing now (context-cancellation masking in `Build` Pass 2) and a handful of nice-to-haves that I left as notes.

## Critical (applied)

### 1. Pass 2 fail-open swallows `context.Canceled` / `context.DeadlineExceeded`

`DefaultBuilder.Build`'s Pass 2 catches *all* source errors as fail-open. If the user cancels the context (or it deadlines) and a source returns `ctx.Err()`, the builder logs a warning, marks the source `Status="error"`, and *keeps iterating* later sources — wasting work and silently masking cancellation.

The Build also does not re-check `ctx.Err()` between optional sources, so a cancelled run keeps fetching until the source list is exhausted.

**Fix applied to `vage/context/builder.go`:**

- Added `if err := ctx.Err(); err != nil { return ... }` at the top of every Pass-2 iteration so cancellation short-circuits.
- After a source error, re-check `ctx.Err()`; if cancelled, bubble that out instead of treating it as fail-open.

The fix is surgical — Pass 1 already bubbles errors, and `runSource` itself does not change.

## Nice to have (skipped, noted only)

These are intentional or borderline calls that I'd have done differently but are not regressions and don't violate the design.

- **`runSource` Tokens backfill drops source's "I emitted messages but explicitly want 0 tokens"** (e.g., a future virtual source). The condition `if rep.Tokens == 0 && len(res.Messages) > 0` overwrites that with the estimator. In practice no current source does this and the design says "Tokens left at zero — Builder fills via estimator" (sources_system.go:84) so the behaviour is documented.
- **Mixed token accounting in `trimByTokens`**: when a source set its own `rep.Tokens` (knowing better than the heuristic), the builder uses that value to *decide* whether to trim, but recomputes per-message tokens via the *estimator* during trimming. So the post-trim `rep.Tokens` no longer matches the source's accounting basis. Marginal; documented.
- **`SessionStateSource.Fetch` partial-failure**: if some keys error and the rest are missing, the source reports `Skipped` — failures get hidden. The design says GetState failures are fail-open per-key, so this is consistent, just a bit lossy in the audit trail.
- **No `source_test.go`**: design lists it but only `builder_test.go` and per-source tests exist. The builder tests cover the helpers (`isMustInclude`, `fromBuildInput`) implicitly.
- **`EventContextBuilt` dispatches even on zero-source / cancelled-with-no-output Build**: design accepts this as audit overhead. OK.
- **`Vars` is plumbed but only `SystemPromptSource` reads it** — by design.
- **Builder does not defensively copy slices** between sources or in `BuildResult.Messages`. All current sources produce fresh allocations (`schema.ToAIModelMessages`, `append([]aimodel.Message(nil), …)`), so there is no aliasing today. Worth a doc comment if future sources are added.

## Concurrency

`DefaultBuilder` post-construction state (`name`, `sources`, `estimator`, `hooks`) is read-only; per-call state (`slots`, `emitted`, `reports`, `remaining`) is local. Safe for concurrent `Build` calls across distinct invocations. Sources rely on their backing stores being concurrency-safe (`memory.Manager`, `session.SessionStateStore`) — same expectation as today.

## License headers

All 13 new `.go` files in `vage/context/` carry the Apache 2.0 header. `make license-check` passes.

## Test status (post-fix)

- `go build ./...` — clean
- `go test ./context/ ./agent/taskagent/ ./schema/` — pass
- `go test ./...` — pass (full module)
- `make lint` — 0 issues
- `make license-check` — pass
