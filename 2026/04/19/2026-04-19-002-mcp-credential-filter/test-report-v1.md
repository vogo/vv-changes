# Test Report v1 — MCP Credential Filter (P0-4)

## Summary

Integration tests authored for the MCP credential-filter middleware.
All target acceptance criteria (US-1, US-2, US-3, AC-5.3, AC-2.4, AC-4.4)
are exercised via the MCP Client/Server pair over real in-memory
transports.

- **File added**: `vage/integrations/mcp_tests/credential_filter_test.go`
- **Files modified**: none (pure test addition)
- **Tests authored**: 14
- **Tests passing**: 14 / 14
- **Lint**: 0 issues

## What Was Built

A single test file with its own helper type `credFilterFixture` that
wires up an in-memory `mcp.NewInMemoryTransports()` pair, registers an
"echo" tool whose handler the test supplies, and tracks whether the
handler was actually invoked via an `atomic.Bool`. All tests are
top-level functions (no `t.Run` subtests) so each appears in go-test
output with a descriptive name.

### Test Matrix

| # | Test | AC mapped | Verifies |
|---|------|-----------|----------|
| 1 | `TestCredentialFilter_ClientOutboundRedact` | AC-1.2 | Client redact → handler sees `[REDACTED:...]`, not plaintext |
| 2 | `TestCredentialFilter_ClientOutboundBlock` | AC-1.3 | Client block → `IsError=true`, handler NOT invoked |
| 3 | `TestCredentialFilter_ClientOutboundLog` | AC-1.4 | Client log → args passthrough, callback fires |
| 4 | `TestCredentialFilter_ClientInboundRedact` | AC-2.1, AC-2.2 | Inbound redact → plaintext never reaches caller |
| 5 | `TestCredentialFilter_ClientInboundBlock` | AC-2.2 | Inbound block → `IsError=true`, no plaintext in error |
| 6 | `TestCredentialFilter_ServerInboundRedact` | AC-3.1 | Server redact → handler sees redacted args |
| 7 | `TestCredentialFilter_ServerInboundBlock` | AC-3.1 | Server block → handler NOT invoked, client gets error |
| 8 | `TestCredentialFilter_ServerOutboundRedact` | AC-3.2 | Server redact → client sees redacted text |
| 9 | `TestCredentialFilter_ServerOutboundBlock` | AC-3.2 | Server block → client sees error result |
| 10 | `TestCredentialFilter_ClientCombinedOutboundInbound` | AC-3.3 | Both directions redact in one CallTool; callback fires per direction |
| 11 | `TestCredentialFilter_CallbackMaskedOnly` | AC-5.3 | `Hits[*].Masked` never contains full plaintext |
| 12 | `TestCredentialFilter_TruncatedPropagation` | AC-2.4 | `MaxScanBytes:32` + credential past cap → `Truncated=true`, no hit beyond cap |
| 13 | `TestCredentialFilter_DefaultPassthrough` | AC-4.4 | No scanner → zero-effect: args and result unchanged |
| 14 | `TestCredentialFilter_ServerCallbackReceivesEvents` | AC-5.1 (server) | Server-side callback fires with masked hits |

Credential test vectors used (documented non-secret strings):
- `AKIAIOSFODNN7EXAMPLE` (AWS access-key example from AWS docs)
- `AKIAABCDEFGHIJKLMNOP` (second AWS-shaped key for combined test)
- `ghp_1234567890abcdefghij1234567890abcdefgh` (GitHub PAT shape)
- `AIzaSyA1234567890123456789012345678901234` (Google API key shape)

## Commands Run

```
cd vage && go test ./integrations/mcp_tests/ -v
cd vage && make lint
```

### Test Output (credential filter subset)

```
=== RUN   TestCredentialFilter_ClientOutboundRedact
--- PASS: TestCredentialFilter_ClientOutboundRedact (0.00s)
=== RUN   TestCredentialFilter_ClientOutboundBlock
--- PASS: TestCredentialFilter_ClientOutboundBlock (0.00s)
=== RUN   TestCredentialFilter_ClientOutboundLog
--- PASS: TestCredentialFilter_ClientOutboundLog (0.00s)
=== RUN   TestCredentialFilter_ClientInboundRedact
--- PASS: TestCredentialFilter_ClientInboundRedact (0.00s)
=== RUN   TestCredentialFilter_ClientInboundBlock
--- PASS: TestCredentialFilter_ClientInboundBlock (0.00s)
=== RUN   TestCredentialFilter_ServerInboundRedact
--- PASS: TestCredentialFilter_ServerInboundRedact (0.00s)
=== RUN   TestCredentialFilter_ServerInboundBlock
--- PASS: TestCredentialFilter_ServerInboundBlock (0.00s)
=== RUN   TestCredentialFilter_ServerOutboundRedact
--- PASS: TestCredentialFilter_ServerOutboundRedact (0.00s)
=== RUN   TestCredentialFilter_ServerOutboundBlock
--- PASS: TestCredentialFilter_ServerOutboundBlock (0.00s)
=== RUN   TestCredentialFilter_ClientCombinedOutboundInbound
--- PASS: TestCredentialFilter_ClientCombinedOutboundInbound (0.00s)
=== RUN   TestCredentialFilter_CallbackMaskedOnly
--- PASS: TestCredentialFilter_CallbackMaskedOnly (0.00s)
=== RUN   TestCredentialFilter_TruncatedPropagation
--- PASS: TestCredentialFilter_TruncatedPropagation (0.00s)
=== RUN   TestCredentialFilter_DefaultPassthrough
--- PASS: TestCredentialFilter_DefaultPassthrough (0.00s)
=== RUN   TestCredentialFilter_ServerCallbackReceivesEvents
--- PASS: TestCredentialFilter_ServerCallbackReceivesEvents (0.00s)
PASS
ok  	github.com/vogo/vage/integrations/mcp_tests	0.744s
```

### Full Suite

```
ok  	github.com/vogo/vage/integrations/mcp_tests	0.274s
```
(18 tests total — 14 new + 4 pre-existing — all pass.)

### Lint

```
golangci-lint run
0 issues.
```

## Notes / Observations

- Each test case carries a doc comment that states the scenario being
  verified, per the tester brief.
- Behavior of `ActionLog` on the outbound path was confirmed: even though
  `authorization` is a FieldRule, log mode does not mutate the map — the
  server handler receives the original value and the callback still fires.
  This matches `ScanJSONMap`'s `mutateMap := s.action == ActionRedact`
  guard. Encoded as the assertion in `TestCredentialFilter_ClientOutboundLog`.
- `TestCredentialFilter_TruncatedPropagation` covers the review's flagged
  "no test asserts `ScanEvent.Truncated` propagates" gap.
- `TestCredentialFilter_ClientCombinedOutboundInbound` covers the review's
  flagged "no test for combined outbound+inbound hits in a single call" gap.
- No source modifications were made; all findings are test-side only.
