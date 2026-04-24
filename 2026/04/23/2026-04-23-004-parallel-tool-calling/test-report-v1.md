# Test Report v1 — P1-7 Parallel Tool Calling

Session dir: `changes/2026/04/23/2026-04-23-004-parallel-tool-calling/`

## Summary

| Check | Result |
|-------|--------|
| `go test ./...` in `vage/` | 47 packages OK, 0 FAIL |
| `go test ./...` in `vv/` | 36 packages OK, 0 FAIL |
| `go test -race ./agent/taskagent/...` | PASS |
| `make lint` in `vage/` | 0 issues |
| `make lint` in `vv/` | 0 issues |

All integration tests pass. No error report needed.

## Coverage audit

Every requirement AC is covered. The pre-existing unit tests (9 in
`task_parallel_test.go`, 3 in `configs/agents_parallel_test.go`) already
covered AC-1.1, 1.2, 2.1, 2.2, 2.3, 3.1, 3.2, 4.1, 4.2, 5.1 for the sync
`Run` path and the config wiring. Two gaps remained, which this phase filled:

| AC | Gap | Filled by |
|----|-----|-----------|
| AC-5.2 | End-to-end wiring through `setup.New` → `registries.FactoryOptions` → coder factory → `taskagent.WithMaxParallelToolCalls` | `TestIntegration_SetupNew_ParallelToolCalls_DefaultCap`, `TestIntegration_SetupNew_ParallelToolCalls_SerialCap` |
| AC-2.1 / AC-2.2 in `RunStream` | Event-ordering test only exercised `Run`; streaming path uses the same helper but a different event sink (`send()` instead of hook-only `dispatch`) | `TestParallelToolCalls_StreamOrdering` |

## Tests added

### `vv/integrations/setup_tests/setup_tests/setup_parallel_tools_test.go`

- `TestIntegration_SetupNew_ParallelToolCalls_DefaultCap` (AC-5.2) — builds
  a coder agent via `setup.New`, feeds a mocked assistant message with two
  `read` tool calls against real temp files, asserts both tool-result
  messages appear in `ToolCalls` order in the follow-up LLM request with
  correct `ToolCallID`s and file-content bodies. Runs with
  `MaxParallelToolCalls: 0` (framework default = 4).
- `TestIntegration_SetupNew_ParallelToolCalls_SerialCap` (AC-4.1, AC-5.2)
  — same scaffold as above but with `MaxParallelToolCalls: 1`, confirming
  the serial escape hatch flows through the full `setup.New` → registry
  → factory → TaskAgent pipeline and produces identical transcript shape.

Local helpers `queuedMockCompleter`, `twoReadToolCallResponse`,
`stopTextResponse`, and `runCoderWithTwoReads` were added in-file because
the shared setup-tests `mockChatCompleter` only returns a single response
(this test needs two LLM turns — tool-call turn + stop turn).

### `vage/agent/taskagent/task_parallel_stream_test.go`

- `TestParallelToolCalls_StreamOrdering` (AC-2.1, AC-2.2 for `RunStream`)
  — three concurrent `wait` tool calls with randomised sleep times
  (90 / 10 / 45 ms) so workers finish out-of-order. Uses the existing
  `sseStreamServer` scaffolding and a new `multiToolCallChunks` helper that
  emits one OpenAI-compatible streaming chunk sequence carrying multiple
  `tool_calls` entries. Asserts Start → Start → Start precede all End /
  Result events, and End_i / Result_i arrive in `ToolCalls[i]` order.

This confirms the shared `executeToolBatch` helper preserves ordering
through the stream path's `send()` sink, not only through the sync path's
hook-based `dispatch`. The helper is structurally identical between the
two call sites, but the explicit stream test guards against any future
divergence in the `emitResultEvent=true` branch.

## Tests that failed

None.

## Notes

- The race detector remained clean on the new tests
  (`go test -race ./agent/taskagent/...` passes in 2.5 s).
- The new `setup_parallel_tools_test.go` required `tmp` to be added to
  `cfg.Tools.AllowedDirs` so the real `read` tool's `PathGuard` would
  permit reads of the temp-file fixtures.
- The `read` tool's argument is `file_path`, not `path`; the mocked
  assistant message uses the correct shape.
