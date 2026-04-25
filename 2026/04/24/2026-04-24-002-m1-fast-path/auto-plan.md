# Complexity Assessment — M1 Fast-Path

**improver**: **skip** — localized 1 new file + thin wiring in 3 existing files; no new architectural pattern; reuses existing option/config/slog conventions; design is a senior-level pattern from the parent design doc.

**reviewer**: **skip** — change is additive, gated behind a config option defaulting to behavior-preserving defaults; new code has its own unit tests; touches no security/concurrency path; diff roughly in the 200–400 line range but mechanically follows existing patterns (`WithXxx` option, `MCPCredentialFilterConfig` mirror-struct style).

**tester**: **include** — backend routing logic is touched; AC-1.1, AC-2.2, AC-3.2, AC-5.2 each need a runnable assertion; unit tests in `fastpath_test.go` + `dispatch_test.go` counts count as M1 integration-level coverage (no live LLM needed). Sub-agent `tester` will lift these into a proper runnable test-report.

**Effective pipeline**: `analyst → designer → developer → tester → documenter`.
