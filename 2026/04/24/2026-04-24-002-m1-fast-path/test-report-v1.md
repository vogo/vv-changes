# M1 Fast-Path — Integration Test Report (v1)

## Summary: **Pass**

All 7 newly-added integration tests pass on the first green run. The pre-existing integration suite (13 tests), the `dispatches` unit suite (90 tests), the `configs` suite (82 tests), and the `setup` suite (13 tests) remain green — no regressions.

Total runtime across the four invoked packages: **~2.9s** (integration 0.318s, dispatches 0.380s, configs 0.973s, setup 1.427s; numbers from `-count=1` runs).

## Deliverables

- **New file:** `vv/integrations/dispatches_tests/dispatches_tests/fastpath_integration_test.go`
  - Reuses `newIntegrationRegistry`, `sequentialMockLLM`, `callTrackingAgent`, `stubStreamAgent`, `stubAgent`, and `makeSubAgents` from `dispatches_helper_test.go` (principle: no duplicate helpers).
  - Introduces one local helper `newFastPathIntegrationDispatcher(t, fp, chat, coder, llm)` — a thin factory wrapping `dispatches.New` for the shared four-agent setup.
  - Uses string literal `"fast_path"` / `"intent"` for stream phase matching because `fastPathPhase` is an unexported constant in the `dispatches` package (integration tests cannot import unexported symbols).

- **No source files modified** — only tests added. Source under `vv/dispatches/`, `vv/configs/`, and `vv/setup/` was left untouched.

## New integration test cases

Each case carries a comment describing the scenario it verifies. Mapping to design acceptance criteria:

| Test | ACs Covered | Scenario |
|------|-------------|----------|
| `TestFastPathIntegration_GreetingShortCircuitsToChat` | AC-1.1 | `"hello"` → 0 intent LLM calls; chat agent invoked; coder not invoked; response body equals chat output. |
| `TestFastPathIntegration_StreamEmitsFastPathPhase` | AC-1.2 / AC-5.2 | `RunStream("hello")` emits `phase_start{fast_path}` + `phase_end{fast_path, ToolCalls=0, PromptTokens=0, CompletionTokens=0}`; does NOT emit `phase_start{intent}`; LLM call count 0. |
| `TestFastPathIntegration_LongGreetingFallsBackToIntent` | AC-1.4 | ~82-char greeting starting with "hello" exceeds 60-rune cap → normal intent flow; exactly 1 intent LLM call; chat dispatched through intent. |
| `TestFastPathIntegration_ToolTriggerRoutesToCoder` | AC-2.1 / AC-2.2 | `"calc 5^6"` → 0 intent LLM calls; coder agent invoked (chat not called); coder's "15625" reply returned. |
| `TestFastPathIntegration_MultiTurnDeclinesFastPath` | AC-3.1 | 3-message conversation (user, assistant, user:"thanks") → fast-path declines; 1 intent LLM call; chat dispatched via intent. |
| `TestFastPathIntegration_ToolHistoryDeclinesFastPath` | AC-3.2 | Prior assistant message with `ToolCalls` + user:"thanks" → fast-path declines; 1 intent LLM call. |
| `TestFastPathIntegration_DisabledFastPathRunsIntent` | AC-4.2 | `WithFastPath(DisabledFastPathConfig())` + `"hello"` → 1 intent LLM call; chat dispatched via intent path; response matches intent-path output. |

Coverage posture: every AC listed in the tester-phase brief has at least one dedicated integration test. The unit-level fast-path suite in `vv/dispatches/fastpath_test.go` already exercises classifier internals (case-insensitivity, Chinese greetings, word-boundary, empty input, agent-missing decline, Default/Disabled config), so the integration layer focuses on end-to-end behavior (LLM call counts, sub-agent invocation, stream event shape).

## Full test inventory

### `./integrations/dispatches_tests/...` — 20 test funcs, 0.318s (all PASS)

Pre-existing (13):
- `TestIntegration_SimpleIntentFastPath`
- `TestIntegration_CodeRequestWithExploration`
- `TestIntegration_ComplexTaskPlanning`
- `TestIntegration_EndToEnd_ExplorerPlusDirect`
- `TestIntegration_DepthPropagationThroughExecute`
- `TestIntegration_ReplanOnStepFailure`
- `TestIntegration_ReplanCountLimit`
- `TestIntegration_RecursionDepthEnforcement` (3 subtests: `depth_0_full_loop`, `depth_at_max_skips_intent`, `depth_above_max`)
- `TestIntegration_StreamingPhaseEvents`
- `TestIntegration_StreamingWithSummarizePhase`
- `TestIntegration_SummaryPolicy_Auto_CLI`
- `TestIntegration_SummaryPolicy_Auto_HTTP`
- `TestIntegration_SummaryPolicy_AlwaysNever` (2 subtests: `always_summarizes_in_cli`, `never_skips_in_http`)

Newly added (7):
- `TestFastPathIntegration_GreetingShortCircuitsToChat`
- `TestFastPathIntegration_StreamEmitsFastPathPhase`
- `TestFastPathIntegration_LongGreetingFallsBackToIntent`
- `TestFastPathIntegration_ToolTriggerRoutesToCoder`
- `TestFastPathIntegration_MultiTurnDeclinesFastPath`
- `TestFastPathIntegration_ToolHistoryDeclinesFastPath`
- `TestFastPathIntegration_DisabledFastPathRunsIntent`

### Regression — `./dispatches/` — 90 tests, 0.380s — **all PASS**
Includes the full fast-path unit suite (`TestFastPathClassify_*`, `TestDispatcher_Run_FastPath*`, `TestDispatcher_RunStream_FastPathEmitsPhaseEvents`, `TestDefaultFastPathConfig_CompilesAllPatterns`).

### Regression — `./configs/` — 82 tests, 0.973s — **all PASS**
Includes fast-path YAML loading tests (Enabled default, explicit `enabled: false`, invalid regex dropped with warning).

### Regression — `./setup/` — 13 tests, 1.427s — **all PASS**
Includes `buildFastPath` translation coverage.

## Notes for the next phase

- Fast-path slog.Info events are emitted during the greeting / tool-trigger tests — visible in test logs as `dispatcher: fast-path hit` records. This confirms the observability requirement from design §9.
- No flakiness observed; all tests use deterministic mocks and no sleeps.
- No source changes were required; the implementation from the developer phase satisfies every acceptance criterion exercised here.
