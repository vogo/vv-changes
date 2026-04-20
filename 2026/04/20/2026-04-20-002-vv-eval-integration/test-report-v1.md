# Test Report v1 — P1-2 vv Eval Integration

Tester: opus-4-7. Scope: integration tests that exercise wiring the unit
suite cannot catch. Source code was NOT modified — only tests under
`vv/integrations/eval_tests/` were added.

## Files added

- `vv/integrations/eval_tests/http_eval_test.go` — 9 tests that boot the
  real `httpapis.Serve` on a loopback port, stub the dispatcher, and
  exercise `POST /v1/eval/run` end-to-end.
- `vv/integrations/eval_tests/cli_eval_test.go` — 7 tests that drive the
  CLI-side pipeline (LoadJSONL → Build → RunBatch → WriteReportJSON)
  plus config-loader behavior (`VV_EVAL_ENABLED`, unknown evaluator
  rejection, default posture).

## Coverage by acceptance criterion

| AC | Criterion | Test(s) |
|----|-----------|---------|
| AC-1.1 | CLI batch runs and produces a report | `TestIntegration_CLIEval_DatasetToReportRoundTrip` |
| AC-1.2 | `-eval-out` writes full JSON | `TestIntegration_CLIEval_DatasetToReportRoundTrip` (round-trips via `WriteReportJSON` + `json.Unmarshal`) |
| AC-1.3 | Per-line parse failures counted, not fatal | `TestIntegration_CLIEval_MalformedLineCounted`, `TestIntegration_CLIEval_MissingDatasetFile` |
| AC-1.4 | `-eval` / `-p` / `--mode http\|mcp` exclusivity | Exercised in `main.go` directly; config-level parts covered by `TestIntegration_CLIEval_EnvVarEnablesHTTPRoute` and the default-posture test. End-to-end CLI-flag parsing is a pure `main.go` flow; kept out because `main()` can't be unit-invoked without a test binary harness — noted below. |
| AC-1.5 | `-eval-concurrency` / `-eval-timeout-ms` overrides | `TestIntegration_CLIEval_ConcurrencyOverrideIdempotent`, `TestIntegration_CLIEval_EnvVarEnablesHTTPRoute` (verifies defaults 1 / 60000) |
| AC-1.6 | Evaluator combo from config; unknown names rejected | `TestIntegration_CLIEval_UnknownEvaluatorRejected` |
| AC-2.1 | HTTP accepts `cases` array | `TestIntegration_HTTPEval_HappyPath`, `TestIntegration_HTTPEval_MessagesObjectInput` |
| AC-2.2 | Response is `EvalReport` JSON with counts | `TestIntegration_HTTPEval_HappyPath` (decodes and checks totals) |
| AC-2.3 | Empty body / empty cases → 400; malformed case → error (not abort) | `TestIntegration_HTTPEval_InvalidBody`, `TestIntegration_HTTPEval_EmptyCases`, `TestIntegration_HTTPEval_MalformedCaseIsErrorNotAbort`, `TestIntegration_HTTPEval_EvaluatorBuildError` |
| AC-2.4 | Coexists with memory endpoints; runs behind same middleware | `TestIntegration_HTTPEval_CoexistsWithMemoryEndpoints`, `TestIntegration_HTTPEval_ServedThroughMiddleware` |
| AC-2.5 | `eval.enabled=false` → 404 (route not mounted) | `TestIntegration_HTTPEval_Disabled404` |
| AC-3.1 | `eval.enabled` default false; `VV_EVAL_ENABLED` flips it | `TestIntegration_CLIEval_DefaultsDisabledForHTTP`, `TestIntegration_CLIEval_EnvVarEnablesHTTPRoute` |
| AC-3.2 | Default evaluators = `latency, cost` (no LLM) | `TestIntegration_CLIEval_EnvVarEnablesHTTPRoute` |
| AC-3.3 | `llm_judge` explicitly opt-in | Covered at unit level (`httpapis/eval_test.go::TestHandleEvalRun_EvaluatorBuildError` + `vv/eval/evaluator_test.go`); the integration surface is the same `Build` call tested via `TestIntegration_HTTPEval_EvaluatorBuildError`. |
| AC-3.4 | Latency / cost default thresholds | `TestIntegration_CLIEval_EnvVarEnablesHTTPRoute` |
| AC-3.5 | Missing `criteria` is fine (evaluators ignore) | Implicitly covered by every happy-path test (cases omit `criteria` and still pass). |

## Run results

```
$ go test ./integrations/eval_tests/... -v
=== RUN   TestIntegration_CLIEval_DatasetToReportRoundTrip          PASS
=== RUN   TestIntegration_CLIEval_MalformedLineCounted              PASS
=== RUN   TestIntegration_CLIEval_MissingDatasetFile                PASS
=== RUN   TestIntegration_CLIEval_EnvVarEnablesHTTPRoute            PASS
=== RUN   TestIntegration_CLIEval_DefaultsDisabledForHTTP           PASS
=== RUN   TestIntegration_CLIEval_UnknownEvaluatorRejected          PASS
=== RUN   TestIntegration_CLIEval_ConcurrencyOverrideIdempotent     PASS
=== RUN   TestIntegration_HTTPEval_HappyPath                        PASS
=== RUN   TestIntegration_HTTPEval_Disabled404                      PASS
=== RUN   TestIntegration_HTTPEval_InvalidBody                      PASS
=== RUN   TestIntegration_HTTPEval_EmptyCases                       PASS
=== RUN   TestIntegration_HTTPEval_MalformedCaseIsErrorNotAbort     PASS
=== RUN   TestIntegration_HTTPEval_EvaluatorBuildError              PASS
=== RUN   TestIntegration_HTTPEval_CoexistsWithMemoryEndpoints      PASS
=== RUN   TestIntegration_HTTPEval_MessagesObjectInput              PASS
=== RUN   TestIntegration_HTTPEval_ServedThroughMiddleware          PASS
PASS
ok  	github.com/vogo/vv/integrations/eval_tests	0.925s
```

Full `go test ./...` for the vv module: **all packages pass**, no
regressions. `golangci-lint run ./integrations/eval_tests/...` reports 0
issues.

## Notes for the developer

- AC-1.4 exclusivity between `-eval` and `-p` / `--mode http|mcp` is
  enforced in `main.go` via `os.Exit(1)` branches. These branches have
  no seam to unit-test without spawning the compiled binary; the design
  acknowledges this and the behavior is enforced by simple string
  comparisons. Consider extracting the exclusivity check into a pure
  function if a future story wants table-driven coverage.
- `RunCLI` itself isn't invoked end-to-end from tests because the
  function takes a `*setup.InitResult` with a concrete `*dispatches.Dispatcher`
  that can't be stood up without a live LLM client. Instead, the CLI
  tests exercise `RunCLI`'s internal pipeline (LoadJSONL → Build →
  RunBatch → WriteReportJSON) against the same stub dispatcher used in
  HTTP tests. If the reviewer later wants a thin seam for full-pipeline
  coverage, consider extracting a `RunCLIWithAgent(ctx, cfg, agent, ...)`
  variant.
- All tests avoid real LLM traffic (no `VV_LLM_API_KEY` required). The
  `llm_judge` evaluator path is exercised only via its error-path
  (evaluator-build failure); that matches the reviewer's "Nit — composite
  with llm_judge not exercised" note, and layered unit tests in
  `vage/eval/llm_judge_test.go` cover the judge's core behavior.
- No source files were modified. Every fix called out in
  `code-review.md` was already in place before tests were written and is
  protected by the new tests (see `TestIntegration_HTTPEval_MalformedCaseIsErrorNotAbort`
  locking in the "counts sum to total" invariant under the HTTP path).
