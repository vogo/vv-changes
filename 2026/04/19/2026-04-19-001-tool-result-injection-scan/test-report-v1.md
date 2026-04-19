# Integration Test Report — v1

**Session:** 2026-04-19-001-tool-result-injection-scan
**Date:** 2026-04-19
**Scope:** Integration tests for the `ToolResultInjectionGuard` end-to-end wiring in `TaskAgent`.
**Test file:** `vage/integrations/guard_tests/tool_result_guard_test.go` (NEW)
**Command:** `cd vage && go test ./integrations/guard_tests/ -v -run ToolResultGuard`
**Result:** **ALL PASS** (12 test functions, 14 including subtests). No failures, no skips.

---

## 1. Test Functions Written

| # | Test function | Scenario | Status |
|---|---------------|----------|--------|
| 1 | `TestIntegration_ToolResultGuard_EndToEnd_Block` | action=block: poisoned tool output → second LLM request sees error-tool-message; raw phrase removed. | PASS |
| 2 | `TestIntegration_ToolResultGuard_EndToEnd_Rewrite` | action=rewrite: content wrapped in `<vage:untrusted source="tool:fetch">…</vage:untrusted>` with WARNING banner; original text preserved inside. | PASS |
| 3 | `TestIntegration_ToolResultGuard_EndToEnd_Log` | action=log: content reaches model untouched; hook observer sees 1 `EventGuardCheck` with `action=log`, correct `tool_name`, `tool_call_id`, `severity`, `snippet`, and `rule_hits`. | PASS |
| 4 | `TestIntegration_ToolResultGuard_HighSeverity_EscalatesToBlock` | ChatML marker (`<|im_start|>`, severity=high) forces Block even when `Action=Log`; event has `action=block`, `severity=high`, rule_hits contains `chatml_marker`. | PASS |
| 5 | `TestIntegration_ToolResultGuard_Stream_EventOrder` | Streaming (`RunStream`) emits event order `ToolCallStart → ToolCallEnd → GuardCheck → ToolResult`; `ToolResult.Data.Result` carries `IsError=true` and not the raw poison. | PASS |
| 6 | `TestIntegration_ToolResultGuard_NoGuard_ZeroImpact` | Agent configured without `WithToolResultGuards` → tool output reaches model byte-for-byte unchanged (even for content that would be high-severity hit). | PASS |
| 7 | `TestIntegration_ToolResultGuard_NonTextContent_PassThrough` | Tool returning only an `image/png` `ContentPart` with `Data` bytes and no text → no scan, no event. | PASS |
| 8 | `TestIntegration_ToolResultGuard_EmptyText_PassThrough` | Tool returning empty text → no scan, no event. | PASS |
| 9 | `TestIntegration_ToolResultGuard_CleanContent_NoEvent` | Clean tool output → silent pass, zero events emitted. | PASS |
| 10 | `TestIntegration_ToolResultGuard_CustomPatterns` (2 subtests) | User-supplied `[]SeveredPatternRule` completely replaces defaults: default phrase does not fire; custom sigil does fire with correct severity. | PASS |
| 11 | `TestIntegration_ToolResultGuard_MaxScanBytes_Truncation` | 1 MB text with `MaxScanBytes=4096`: hits prefix rule + `__truncated` marker observed in `rule_hits`. | PASS |
| 12 | `TestIntegration_ToolResultGuard_Truncation_NoHit` | 1 MB clean text with `MaxScanBytes=4096`: event is still emitted with only `__truncated` in `rule_hits` for observability. | PASS |

---

## 2. Acceptance Criteria Coverage

| AC | Coverage | Test(s) | Notes |
|----|----------|---------|-------|
| AC-1.1 (scan between executeToolCall and toolMsg append) | **Covered** | 1, 2, 3, 4 (non-stream) + 5 (stream) | Asserts the second LLM request's tool message is post-scan; second chat request exists → scan happened before append. |
| AC-1.2 (block / rewrite / log actions) | **Covered** | 1 (block), 2 (rewrite), 3 (log) | Each action exercised end-to-end with mock ChatCompleter. |
| AC-1.3 (Run and RunStream both scan) | **Covered** | 1–4 (Run), 5 (RunStream) | Streaming path verified through SSE test server + event order assertions. |
| AC-1.4 (no guard = zero impact) | **Covered** | 6 | Content with a would-be High-severity hit still passes through when no guard installed. |
| AC-2.1 (default rule pack w/ severities) | **Covered** | 1, 3, 4 | `ignore_instructions` (Low) + `broad_ignore` (Medium) + `chatml_marker` (High) all observed through default pack. |
| AC-2.2 (user-supplied patterns override defaults) | **Covered** | 10 | Verifies both negative (default phrase doesn't fire) and positive (custom sigil fires). |
| AC-2.3 (PromptInjectionGuard unchanged) | **Not re-verified here** | (existing `guard_test.go` already exercises input-side PromptInjectionGuard) | No regression expected; out of scope for this test file. |
| AC-3.1 (EventGuardCheck payload: guard_name, tool_call_id, tool_name, action, rule_hits, severity, snippet) | **Covered** | 3, 4, 10 | Every field asserted on at least one test (tool_call_id in #3, severity in #3/#4/#10, snippet non-empty in #3). |
| AC-3.2 (slog.Warn on hit) | **Partial** | Visible in test logs (e.g. `WARN vage: tool result guard hit guard=tool_result_injection …`) | Not asserted programmatically — no slog capture handler used here. Visible through stderr during tests. |
| AC-3.3 (silent pass emits no events) | **Covered** | 9 (clean content), 7, 8 (non-text/empty) | Explicit `len(events) == 0` assertions. |
| AC-4.1 (vv wires ToolResultGuards into coder/researcher/reviewer) | **Not covered here** | — | Out of scope for `vage/integrations`; should be covered in `vv/integrations/`. |
| AC-4.2 (vv YAML config exposure) | **Not covered here** | — | Same as above. |
| AC-4.3 (chat agent unchanged) | **Not covered here** | — | Same as above. |
| AC-5.1 (p99 < 2 ms on 50 KB) | **Not covered here** | — | Performance target belongs in benchmark (`*_bench_test.go`); integration tests measure correctness, not latency. |
| AC-5.2 (MaxScanBytes truncation + `__truncated` marker) | **Covered** | 11 (with-hit), 12 (clean) | Both paths exercised. |
| AC-5.3 (non-text ContentPart passthrough) | **Covered** | 7 (image), 8 (empty text) | No event emitted, no content mutation. |

---

## 3. Coverage Gaps (and why)

- **AC-2.3 (PromptInjectionGuard regression)** — intentionally not re-tested in this file. The existing suite `vage/integrations/guard_tests/guard_test.go` already exercises `PromptInjectionGuard` on the input direction and would flag any behavior change. No new test added to avoid duplication.
- **AC-3.2 (slog.Warn payload)** — the warn log is visible at stderr during tests (see run output) but not programmatically asserted. Doing so requires a custom `slog.Handler`; reserved for a follow-up unit test (not integration-level noise).
- **AC-4.x (vv wiring and YAML)** — belongs in `vv/integrations/`, not `vage/integrations/`. The vv side is a separate Go module (`github.com/vogo/vv`) with its own factory/registry tests.
- **AC-5.1 (perf target)** — benchmark, not integration test. See design §9.4 (`vage/guard/tool_result_bench_test.go`).

Nothing was silently dropped. All gaps are explicit and outside the intended scope of this file.

---

## 4. Test Conventions

- Mock `ChatCompleter` copied locally (renamed helpers with `TR` suffix to avoid future collisions if this file is ever merged into a package with shared helpers). The pattern mirrors `vage/agent/taskagent/task_test.go`.
- Each test asserts only what the AC demands; no over-specification of log formatting or internal guard order.
- Event-capture is concurrency-safe via `sync.Mutex` (copied from `tool_result_guard_test.go` hook pattern).
- Streaming test uses `sseStreamServer`-style `httptest.Server` following the existing `task_test.go` conventions.

---

## 5. Run Output (tail)

```
=== RUN   TestIntegration_ToolResultGuard_EndToEnd_Block
--- PASS: TestIntegration_ToolResultGuard_EndToEnd_Block (0.00s)
=== RUN   TestIntegration_ToolResultGuard_EndToEnd_Rewrite
--- PASS: TestIntegration_ToolResultGuard_EndToEnd_Rewrite (0.00s)
=== RUN   TestIntegration_ToolResultGuard_EndToEnd_Log
--- PASS: TestIntegration_ToolResultGuard_EndToEnd_Log (0.00s)
=== RUN   TestIntegration_ToolResultGuard_HighSeverity_EscalatesToBlock
--- PASS: TestIntegration_ToolResultGuard_HighSeverity_EscalatesToBlock (0.00s)
=== RUN   TestIntegration_ToolResultGuard_Stream_EventOrder
--- PASS: TestIntegration_ToolResultGuard_Stream_EventOrder (0.00s)
=== RUN   TestIntegration_ToolResultGuard_NoGuard_ZeroImpact
--- PASS: TestIntegration_ToolResultGuard_NoGuard_ZeroImpact (0.00s)
=== RUN   TestIntegration_ToolResultGuard_NonTextContent_PassThrough
--- PASS: TestIntegration_ToolResultGuard_NonTextContent_PassThrough (0.00s)
=== RUN   TestIntegration_ToolResultGuard_EmptyText_PassThrough
--- PASS: TestIntegration_ToolResultGuard_EmptyText_PassThrough (0.00s)
=== RUN   TestIntegration_ToolResultGuard_CleanContent_NoEvent
--- PASS: TestIntegration_ToolResultGuard_CleanContent_NoEvent (0.00s)
=== RUN   TestIntegration_ToolResultGuard_CustomPatterns
    --- PASS: TestIntegration_ToolResultGuard_CustomPatterns/default_phrase_does_not_match_custom_patterns (0.00s)
    --- PASS: TestIntegration_ToolResultGuard_CustomPatterns/custom_phrase_matches_custom_patterns (0.00s)
--- PASS: TestIntegration_ToolResultGuard_CustomPatterns (0.00s)
=== RUN   TestIntegration_ToolResultGuard_MaxScanBytes_Truncation
--- PASS: TestIntegration_ToolResultGuard_MaxScanBytes_Truncation (0.00s)
=== RUN   TestIntegration_ToolResultGuard_Truncation_NoHit
--- PASS: TestIntegration_ToolResultGuard_Truncation_NoHit (0.00s)
PASS
ok  	github.com/vogo/vage/integrations/guard_tests	0.494s
```

---

## 6. Implementation-level observations (informational, not bugs)

1. The phrase `"ignore previous instructions"` matches **both** `ignore_instructions` (Low) and `broad_ignore` (Medium) in `DefaultToolResultInjectionPatterns()`. My first draft asserted `severity=low` for the Log test; actual behavior correctly returns `severity=medium` (max across hits). Test updated to reflect the true max-severity semantic; this is the intended behavior per design §4/§6.
2. `TruncationMarker` (`"__truncated"`) is correctly excluded from severity computation (`ToolResultInjectionGuard.MaxSeverity` skips it). The truncation-only event (`TestIntegration_ToolResultGuard_Truncation_NoHit`) has empty severity in the log, matching design §3.1.3 "observation only".
3. The quarantine wrapper uses `%q` format (`"fetch"` with quotes) in the WARNING line and `"tool:fetch"` in the source attribute. Tests assert the stable tag form only.

No source-code issues surfaced. All tests pass against the current implementation on the current branch.
