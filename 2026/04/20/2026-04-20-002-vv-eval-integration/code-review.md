# Code Review — P1-2 vv Eval Integration

Reviewer: opus-4-7. Scope: files added/modified by the developer phase.

Summary: the developer phase is a faithful implementation of the design. One
High-severity correctness issue in the batch runner was fixed; a Medium
consistency issue in `RunCLI` (dataset parse errors not surfaced in the JSON
report) was also fixed. Remaining items are Nits that do not warrant changes
in this pass.

Test/lint state after my edits:
- `go test ./eval/ ./httpapis/ ./configs/` → pass
- `go build ./...` → pass
- `make lint` → 0 issues

---

## Findings

### High — `RunBatch` report counts can drift from `TotalCases` (FIXED)

File: `vv/eval/runner.go`

The original loop aborted on `ctx.Err()` with `break`, leaving `results[i] = nil`
for unvisited cases. The aggregation loop skipped `nil` slots with `continue`,
so the report's `PassedCases + FailedCases + ErrorCases` was strictly less than
`TotalCases` whenever the parent context was cancelled mid-batch. HTTP/CI
callers relying on those counters to detect pass/fail would see a silent gap.

There was also a latent issue that goroutines took the semaphore slot
(blocking) even when the parent was already cancelled, forcing the batch to
drain serially before exiting.

**Applied** two fixes:
1. Every launched goroutine now writes a result (either an error when the
   parent is cancelled, or the evaluated result). Aggregation still defensively
   counts any `nil` slot as an error so invariants hold even if future code
   regresses the happy path.
2. The goroutine `select`s on `sem <- struct{}{}` and `<-ctx.Done()` so a
   cancellation short-circuits waiting workers instead of blocking behind the
   semaphore.

Added `TestRunBatch_ParentCancelled` to lock in `passed + failed + error == total`
and verify every case is marked as an error under cancellation.

### Medium — CLI report drops per-line parse errors from JSON output (FIXED)

File: `vv/eval/runner.go` (`RunCLI`)

`RunCLI` already folded `len(loadErrs)` into `report.TotalCases` /
`report.ErrorCases` but did not push the errors into `report.Results`. As a
consequence, a user running `-eval -eval-out out.json` would see the right
aggregate counts but nothing in `results[]` explaining which lines failed —
they'd have to cross-reference stderr. The HTTP handler (`handleEvalRun`)
already appends each malformed case to `report.Results`, so the two surfaces
were inconsistent.

**Applied** a fix that appends each `LoadError` to `report.Results` with
`CaseID: "line-N"` and `Error: <err>`, matching the HTTP handler behavior and
making `-eval-out` self-contained.

### Medium — HTTP handler accepts cases before the evaluator is built

File: `vv/httpapis/eval.go`

When `cfg.Eval` is misconfigured (e.g. `contains` with no keywords), the
handler decodes every case (allocating and parsing) before `Build` runs and
returns 500. The ordering is wasteful but not incorrect — the handler test
(`TestHandleEvalRun_EvaluatorBuildError`) covers the behavior, and the
overhead is bounded by request body size which we already read.

**Deferred** as a micro-optimization. Moving `Build` ahead of decoding would
require passing the uninspected body through differently; not worth the churn.

### Medium — Dataset loader has a 1 MiB hard line cap

File: `vv/eval/dataset.go`

`scanner.Buffer(buf, 1024*1024)` is generous but still a hard ceiling. A real
chat-transcript dataset with long tool outputs could exceed 1 MiB. The failure
mode is a `bufio.Scanner: token too long` error propagated as a scan failure —
the whole dataset is abandoned rather than one line being skipped.

**Deferred** — design §12 explicitly bounds MVP scope to "1000 cases", and
scaling beyond that is out of scope. If this bites, the fix is either a bigger
buffer or switching to `bufio.Reader.ReadBytes('\n')` with no cap.

### Nit — `decodeRunResponse` trusts empty input silently

File: `vv/eval/dataset.go`

If `expected` is the JSON literal `""` (empty string), the loader returns a
`RunResponse` with a single empty assistant message. Harmless in practice; the
user meant "I want this evaluator to check my agent returns nothing" rather
than "I left this blank". Since Expected is optional and consumers (e.g. the
evaluators that use it) tolerate empty content, this is acceptable.

**Deferred** — no evaluator currently treats an empty expected message
specially, and the alternative (rejecting empty) would make round-tripping
`expected:""` awkward.

### Nit — `PrintSummary` format versus design doc

File: `vv/eval/report.go`

The design sketched `Avg Score  : 0.82`; the code prints `%.3f` (e.g. `0.820`).
Tests check the `%.3f` format. Minor cosmetic drift; neither spec is
machine-consumed.

**Deferred** — no functional impact; locking the format in tests is already
sufficient.

### Nit — `runner.go` defensive branch is unreachable from `DecodeCaseLine` callers

File: `vv/eval/runner.go` (`evaluateCase`)

```go
if c.Input == nil && c.Actual == nil {
    return &vageeval.EvalResult{CaseID: c.ID, Error: "case has no input and no actual"}
}
```

This can't fire for cases produced by `DecodeCaseLine` (which rejects missing
input) but is a legitimate guard for programmatic callers that construct
`EvalCase` directly. Leaving as-is.

**Deferred** — defensive code with no cost.

### Nit — `evaluator_test.go` does not exercise Composite with llm_judge

The evaluator unit tests cover latency, cost, contains, and composite of
latency+cost, but do not exercise the llm_judge branch beyond the
"requires LLM client" error path. Since llm_judge just wraps
`eval.NewLLMJudgeEval` with effective-model resolution, the branch is short
and relies on vage's own tests for correctness.

**Deferred** — coverage cost/benefit is low; the tester phase can add an
LLM-backed integration test if needed.

### Nit — `main.go` eval path ignores the error swallowed after `RunCLI`

File: `vv/main.go`

```go
exitCode, evalErr := vveval.RunCLI(...)
if evalErr != nil {
    fmt.Fprintf(os.Stderr, "vv: %s\n", evalErr)
}
os.Exit(exitCode)
```

If `RunCLI` returns a non-nil error, we log it and still exit with whatever
`exitCode` it returned. `RunCLI`'s contract is to return exit code 1 on any
error, so this is correct in practice; I'd prefer an explicit `os.Exit(1)` in
the error branch just to avoid coupling to `RunCLI`'s internals.

**Deferred** — design sanctioned this pattern (see §4 in `design.md`), and the
tester phase can tighten the contract if needed.

---

## Files Touched By This Review

- `vv/eval/runner.go` — bounded worker loop honors context cancellation; dataset
  parse errors flow into `report.Results`.
- `vv/eval/runner_test.go` — new `TestRunBatch_ParentCancelled` locking in the
  passed+failed+error == total invariant.

No files outside the developer's diff were modified.
