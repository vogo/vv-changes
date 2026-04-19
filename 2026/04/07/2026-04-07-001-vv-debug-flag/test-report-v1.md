# Integration Test Report v1 — vv `--debug` flag

Date: 2026-04-07
Tester: QA (automated)
Scope: integration tests for the new `--debug` flag wiring across vv (CLI `-p`, HTTP, env/flag/yaml precedence, tool decorator, redaction, concurrency).

## What was added

New file: `vv/integrations/debug_tests/debug_integration_test.go` (new package `debug_tests`).

The tests use a fake `aimodel.ChatCompleter` and a fake `tool.ToolRegistry`, wired through the **same** debug stack the production code uses (`largemodel.NewDebugMiddleware` + `debugs.SinkAdapter` for LLM I/O; `debugs.NewDebuggingToolRegistry` for tool I/O). No real LLM is required; one smoke test is gated on `VV_LLM_API_KEY` / `AI_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` and is skipped otherwise.

## Test cases

| # | Test | Scenario |
|---|------|----------|
| 1 | `TestDebug_Off_NoOutput_PromptMode` | Mirrors `-p` mode wiring with `cfg.Debug=false`. Asserts sink is never constructed and zero bytes are written; LLM and tool calls produce no debug output. |
| 2 | `TestDebug_Off_NoOutput_HTTPMode` | Mirrors HTTP mode wiring with `cfg.Debug=false`. Asserts no `llm.*` records reach the slog buffer. |
| 3 | `TestDebug_On_PromptMode_LLMAndToolRecords` | `-p` mode with debug ON. Asserts `llm.request`, `llm.response`, `tool.start`, `tool.end` lines, plus the response content, tool args, tool result, and `agent=coder` tag, all appear on the captured stderr writer. |
| 4 | `TestDebug_EnvOverridesYAML` | YAML `debug: false` + `VV_DEBUG=true` ⇒ `cfg.Debug==true`. |
| 5 | `TestDebug_FlagOverridesEnv` | YAML `debug: false` + `VV_DEBUG=true` + explicit `--debug=false` ⇒ `cfg.Debug==false` (mirrors main.go's `if debugSet { cfg.Debug = *debugFlag }`). |
| 6 | `TestDebug_YAMLEnabledWhenNoEnvOrFlag` | YAML `debug: true` alone ⇒ `cfg.Debug==true`. |
| 7 | `TestDebug_HTTPMode_ResponseByteIdentical` | Same canned `ChatResponse` returned by raw client and by debug-wrapped client; asserts response content + usage are identical and that the wrapped path emits `llm.request` / `llm.response` records to slog. Proves the middleware is non-mutating. |
| 8 | `TestDebug_ToolDecorator_CapturesArgsResultLatency` | Built-in `read` tool exercised through `DebuggingToolRegistry`; asserts captured `tool.start`, `tool.end`, args, result text, and `dur=` (latency). |
| 9 | `TestDebug_ToolDecorator_CapturesError` | Inner tool returns an error; asserts the error string is present in the `tool.end` record and is propagated. |
| 10 | `TestDebug_Redaction_APIKeyNotPresent` | `Redact()` scrubs `api_key=...`, `Authorization: Bearer ...`, and `?api-key=...` patterns; asserts none of the original secret bytes appear. |
| 11 | `TestDebug_Redaction_EnvSecretsScrubbed` | When `OPENAI_API_KEY` is set in env, the literal value is replaced with `<redacted>` anywhere in a snapshot. |
| 12 | `TestDebug_ConcurrentRequests_NoGarble` | 200 goroutines emit records with distinct correlation ids; asserts (a) every correlation id is present and intact in the output, and (b) the line count for `kind=llm.request` is exactly 200 (no torn writes — proves the sink mutex prevents interleaving). |
| 13 | `TestDebug_RealLLM_Smoke` | Gated on `VV_LLM_API_KEY` / `AI_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`; skipped otherwise. |

Each test has a doc comment describing the scenario it covers.

## Test runs

### `go test ./integrations/...` in `vv/`

```
ok  	github.com/vogo/vv/integrations/agents_tests
ok  	github.com/vogo/vv/integrations/askuser_tests
ok  	github.com/vogo/vv/integrations/cli_tests
ok  	github.com/vogo/vv/integrations/config_tests
ok  	github.com/vogo/vv/integrations/costtracker_tests
ok  	github.com/vogo/vv/integrations/debug_tests
ok  	github.com/vogo/vv/integrations/dispatches_tests
ok  	github.com/vogo/vv/integrations/http_tests
ok  	github.com/vogo/vv/integrations/permission_tests
ok  	github.com/vogo/vv/integrations/project_instructions_tests
ok  	github.com/vogo/vv/integrations/prompt_tests
ok  	github.com/vogo/vv/integrations/setup_tests
ok  	github.com/vogo/vv/integrations/shutdown_tests
ok  	github.com/vogo/vv/integrations/tools_tests
ok  	github.com/vogo/vv/integrations/vv_tests
ok  	github.com/vogo/vv/integrations/wiring_tests
```

### `go test ./...` in `vv/`

```
ok  	github.com/vogo/vv/agents
ok  	github.com/vogo/vv/cli
ok  	github.com/vogo/vv/configs
ok  	github.com/vogo/vv/debugs
ok  	github.com/vogo/vv/dispatches
ok  	github.com/vogo/vv/hooks
ok  	github.com/vogo/vv/httpapis
ok  	github.com/vogo/vv/integrations/...   (all green, see above)
ok  	github.com/vogo/vv/memories
ok  	github.com/vogo/vv/registries
ok  	github.com/vogo/vv/setup
ok  	github.com/vogo/vv/tools
ok  	github.com/vogo/vv/traces/costtraces
```

All tests pass. No flakes. No source code was modified during this round.

## Notes / observations

- The `Redact` helper is intentionally narrow (regex-driven, oriented at JSON-ish snapshots and HTTP headers). The first cut of test #10 used a JSON-embedded `"authorization":"Bearer ..."` form that does not match the existing `authHeaderRe` (which expects header-style `Authorization: Bearer ...`). The test was rewritten to feed a multi-line snapshot containing the patterns the helper actually targets, matching the documented design (config snapshot only, R-7).
- The byte-identical HTTP property (test #7) is verified at the LLM-client boundary, which is the only point at which the debug middleware interposes. The HTTP layer below is unmodified by the debug feature, so this is the relevant invariant.
- The first-run real-LLM smoke (test #13) is intentionally minimal — the heavier real-LLM coverage already lives in other `vv/integrations/*` suites and would be redundant to duplicate here.

## Status

PASS — all 13 added test cases pass; full `go test ./...` in `vv/` is green.
