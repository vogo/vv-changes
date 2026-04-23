# Test Report v1 — P1-5 Structured JSONL Conversation Trace Logging

Date: 2026-04-23
Scope: Integration tests for `vv/traces/tracelog` + `setup.Init` wiring.

## Tests Written

All new integration tests live in
`vv/integrations/traces_tests/tracelog_tests/`. Each test is annotated in
source with the User Story (US-N) it verifies.

| File | Test | US Coverage | Purpose |
|------|------|-------------|---------|
| `tracelog_helper_test.go` | (helpers only — `stubCompleter`, `decodeEvents`, `makeTraceConfig`, `initWithStubAgent`, `shutdownWithTimeout`, `projectTraceDir`) | — | Shared fixtures for the package; mirrors `budget_tests/budget_helper_test.go` conventions. |
| `tracelog_enabled_test.go` | `TestIntegration_Enabled_EndToEnd_FullPipeline` | **US-1, US-5, US-6** | Full `setup.Init` → TaskAgent.Run → shutdown flush → JSONL-on-disk. Asserts file under `<tempDir>/<projectHash>/<sid>.jsonl`; every line decodes to `{type, session_id, timestamp, data}`; first event is `agent_start`; last is `agent_end` or `error`; `initResult.SetupResult.HookManager != nil`; `initResult.Shutdown != nil`. |
| `tracelog_enabled_test.go` | `TestIntegration_Enabled_MultiSessionRouting` | **US-1, US-5** | Two sequential agent runs with distinct `SessionID` → two separate JSONL files, each routed to the correct file, each starting with `agent_start`. Supports the P2-14 session-resume contract. |
| `tracelog_enabled_test.go` | `TestIntegration_Enabled_ShutdownIsIdempotent` | **US-4** | Two consecutive `initResult.Shutdown(ctx)` calls do not panic, deadlock, or error — matches `main.go`'s defer-plus-signal usage. |
| `tracelog_async_test.go` | `TestIntegration_Enabled_AsyncDoesNotBlock` | **US-3** | 50 back-to-back agent runs under a 5-second deadline — asserts the AsyncHook never blocks the dispatch path even under burst pressure. Explicitly tolerates event drops (per design §US-3). |
| `tracelog_disabled_test.go` | `TestIntegration_Disabled_NoHookNoFiles` | **US-2** | `cfg.Trace.Enabled` unset → `initResult.SetupResult.HookManager == nil`; `initResult.Shutdown` is still non-nil and a safe no-op (including against an already-cancelled context); no files materialise under the would-be project trace dir even after a full agent Run. |

## Totals

| Metric | Value |
|--------|-------|
| Tests written | 5 (in 3 new test files + 1 helper file) |
| Test count in `tracelog_tests` package | 5 |
| Pass | 5 |
| Fail | 0 |
| Full `vv` suite `make test` | **PASS** (all packages) |
| `golangci-lint run ./...` | **0 issues** |

Single-package run:

```
$ cd vv && go test ./integrations/traces_tests/tracelog_tests/ -v -count=1
=== RUN   TestIntegration_Enabled_AsyncDoesNotBlock
--- PASS (0.02s)
=== RUN   TestIntegration_Disabled_NoHookNoFiles
--- PASS (0.00s)
=== RUN   TestIntegration_Enabled_EndToEnd_FullPipeline
--- PASS (0.01s)
=== RUN   TestIntegration_Enabled_MultiSessionRouting
--- PASS (0.01s)
=== RUN   TestIntegration_Enabled_ShutdownIsIdempotent
--- PASS (0.01s)
PASS
ok      github.com/vogo/vv/integrations/traces_tests/tracelog_tests     0.743s
```

## Coverage for `vv/traces/tracelog`

```
$ go test -cover ./traces/tracelog/
ok  github.com/vogo/vv/traces/tracelog  coverage: 75.7% of statements
```

The 75.7% statement coverage is produced by the pre-existing unit tests
(`tracelog_test.go`) — the 5 new integration tests do not run inside the
`tracelog` package and therefore do not contribute to its per-package
coverage number. Uncovered statements are mostly the `slog.Warn` fallback
branches inside the consumer goroutine (marshal error / write error /
rotate-error paths), which are defensively unreachable in the happy-path
tests.

## Acceptance Criteria Mapping

| User Story | Acceptance Summary | Test(s) |
|------------|--------------------|---------|
| **US-1** (enabled → JSONL on disk) | `~/.vv/traces/<projectHash>/<sid>.jsonl` exists; each line is valid JSON; first line `agent_start`, last `agent_end` or `error`; key fields `type/timestamp/session_id/data` present. | `TestIntegration_Enabled_EndToEnd_FullPipeline` (primary), `TestIntegration_Enabled_MultiSessionRouting` (secondary). |
| **US-2** (disabled → zero cost) | No `hook.Manager` installed; no consumer goroutine; no files; `HookManager == nil`. | `TestIntegration_Disabled_NoHookNoFiles`. |
| **US-3** (non-blocking dispatch) | AsyncHook never blocks agent path even under burst; drops are acceptable per design. | `TestIntegration_Enabled_AsyncDoesNotBlock`. |
| **US-4** (graceful flush / crash-safe JSONL) | `Stop(ctx)` drains and flushes; append-only line writes remain valid JSON even on partial lines. | `TestIntegration_Enabled_ShutdownIsIdempotent` (drain + idempotency); `TestIntegration_Enabled_EndToEnd_FullPipeline` (events visible on disk only after Shutdown flush). The append-only "partial last line" behaviour is asserted by the existing unit test `Test_FileRotation` + the read-back `decodeEvents` helper which rejects any malformed line. |
| **US-5** (event coverage for SFT rebuild) | Minimum `agent_start` + `agent_end`; full event firehose retained in file order. | `TestIntegration_Enabled_EndToEnd_FullPipeline` asserts both bookends; unit-level `Test_HappyPath` / `Test_MultipleSessionsGetSeparateFiles` cover the full event envelope fields. |
| **US-6** (CLI / HTTP / MCP uniform coverage) | Hook is wired in the common `setup.Init` path; three modes share the Init entry point. | `TestIntegration_Enabled_EndToEnd_FullPipeline` asserts `initResult.SetupResult.HookManager != nil` after `setup.Init` — the single invariant that guarantees all three modes pick up trace logging because all three share the exact same `setup.Init` entry. |

## Uncovered Acceptance Criteria

None of the six User Stories are uncovered. A few sub-bullets are
exercised only indirectly:

- **US-3 drop warning** (`slog.Warn` when the AsyncHook channel is full):
  the design explicitly permits drops; reliably forcing a drop from an
  integration test would require stalling the consumer goroutine or
  slamming the channel beyond `BufferSize=1024`. The integration
  `TestIntegration_Enabled_AsyncDoesNotBlock` covers the load-tolerance
  half (no block); the drop-warning half is left to `vage/hook`'s own
  manager tests — they are the owner of that `select { default: drop }`
  code path, not `vv/traces/tracelog`.
- **US-4 SIGKILL "partial last line"**: no integration test kills the
  process mid-write (the Go test harness cannot do this without external
  orchestration). Append-only semantics are structurally guaranteed by
  `os.O_APPEND` in `openSession` and by line-by-line `Write(line)` in the
  consumer — both are inspected by code review.
- **US-6 HTTP / MCP bindings**: tested indirectly via the single
  `setup.Init` invariant. A direct HTTP-mode e2e would require standing
  up `httpapis.Serve` with a stub LLM — that belongs to a broader
  httpapi test, not this feature. The current tests verify the wiring
  step all three modes share.

## Notes for Developer / Documenter

- **No source changes were made.** Only tests + this report were
  produced.
- `vv/vv` binary absent after tests (no build step invoked).
- New helper `tracelog_helper_test.go` matches the sibling
  `budget_helper_test.go` style so future tests in this package have a
  consistent entry point.
