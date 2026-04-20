# Test Report v1 — vv MCP Server Integration (P1-1)

- **Session dir**: `changes/2026/04/20/2026-04-20-001-vv-mcp-server-integration/`
- **File under test**: `vv/integrations/mcp_tests/mcp_server_extra_test.go` (new), plus existing `vv/integrations/mcp_tests/mcp_server_test.go`.
- **Scope**: integration-level coverage of AC-1.4, AC-2.3 (block + inbound), AC-3.3, AC-3.4 for the new `vv/mcps` package.

## 1. New integration tests added

All tests live in `vv/integrations/mcp_tests/mcp_server_extra_test.go` and reuse the existing helpers (`echoAgent`, `stubLookup`, `startServer`) from `mcp_server_test.go`. Each test case carries a comment describing the scenario it verifies.

| # | Test | Scenario |
|---|------|----------|
| 1 | `TestMCPServer_CredentialFilterBlocksOutbound` | `MCPCredentialFilterConfig{Action: "block"}` + agent reply containing an AWS key → tool result has `IsError=true` and body `"blocked by mcp credential filter (server-out): aws_access_key"`. |
| 2 | `TestMCPServer_CredentialFilterRedactsInbound` | Default (`redact`) config; client sends `{"input":"... AKIAIOSFODNN7EXAMPLE ..."}`. `applyInboundScan` must rewrite the map before the agent sees it, so the echoed output no longer contains the raw credential. |
| 3 | `TestMCPServer_MultipleSequentialCalls` | Three sequential `tools/call` invocations against the same exposed agent on a single MCP session; each reply must equal `echo: <its own input>` with no cross-talk. Covers AC-1.4 (no state leak across calls). |
| 4 | `TestMCPServer_StdioGracefulShutdown` | `srv.Serve(ctx, inMemoryTransport)` in a goroutine; ctx cancel → Serve returns in <2s. |
| 5 | `TestMCPServer_HTTPBearerAuthRejectsMissingHeader` | `startHTTPServer` replicates `serveHTTP`'s stack (StreamableHTTPHandler + bearerAuth + `/healthz`). POST `/` with no header → 401; POST `/` with `Authorization: Bearer wrong` → 401. |
| 6 | `TestMCPServer_HTTPBearerAuthAcceptsCorrectHeader` | Same stack; POST `/` with the correct `Authorization: Bearer <token>` must not return 401 (auth layer passes through to MCP handler). |
| 7 | `TestMCPServer_HTTPGracefulShutdown` | GET `/healthz` proves server is live (204); `httpSrv.Shutdown` returns in <2s; post-shutdown GET fails. Covers AC-3.4 for HTTP transport. |

### Helpers added (test-local, not source)
- `bearerAuthTestMiddleware(token, next)` — byte-for-byte port of `mcps.bearerAuth`. Needed because the production helper is unexported. Documented with a "keep in sync" comment.
- `startHTTPServer(t, cfg, token)` — probes a free `127.0.0.1:0` port, re-binds it, assembles the same `mux + auth + /healthz + StreamableHTTPHandler` stack that `mcps.serveHTTP` builds, runs `httpSrv.Serve` in a goroutine, and returns `(baseURL, shutdown)`. Does not call `mcps.Serve` directly because `mcps.Serve` requires `*setup.InitResult` whose inner `*setup.Result` has unexported fields and no public constructor.

## 2. Run result

### `go test ./integrations/mcp_tests/ -count=1 -v`

```
=== RUN   TestMCPServer_CredentialFilterBlocksOutbound
--- PASS: TestMCPServer_CredentialFilterBlocksOutbound (0.00s)
=== RUN   TestMCPServer_CredentialFilterRedactsInbound
--- PASS: TestMCPServer_CredentialFilterRedactsInbound (0.00s)
=== RUN   TestMCPServer_MultipleSequentialCalls
--- PASS: TestMCPServer_MultipleSequentialCalls (0.00s)
=== RUN   TestMCPServer_StdioGracefulShutdown
--- PASS: TestMCPServer_StdioGracefulShutdown (0.05s)
=== RUN   TestMCPServer_HTTPBearerAuthRejectsMissingHeader
--- PASS: TestMCPServer_HTTPBearerAuthRejectsMissingHeader (0.00s)
=== RUN   TestMCPServer_HTTPBearerAuthAcceptsCorrectHeader
--- PASS: TestMCPServer_HTTPBearerAuthAcceptsCorrectHeader (0.00s)
=== RUN   TestMCPServer_HTTPGracefulShutdown
--- PASS: TestMCPServer_HTTPGracefulShutdown (0.00s)
=== RUN   TestMCPServer_ListsExposedAgents
--- PASS: TestMCPServer_ListsExposedAgents (0.00s)
=== RUN   TestMCPServer_CallCoderReturnsText
--- PASS: TestMCPServer_CallCoderReturnsText (0.00s)
=== RUN   TestMCPServer_Whitelist
--- PASS: TestMCPServer_Whitelist (0.00s)
=== RUN   TestMCPServer_ExposeDispatcher
--- PASS: TestMCPServer_ExposeDispatcher (0.00s)
=== RUN   TestMCPServer_CredentialFilterRedactsOutbound
--- PASS: TestMCPServer_CredentialFilterRedactsOutbound (0.00s)
=== RUN   TestMCPServer_UnknownAgentInWhitelist
--- PASS: TestMCPServer_UnknownAgentInWhitelist (0.00s)
PASS
ok  	github.com/vogo/vv/integrations/mcp_tests	0.668s
```

- 13/13 tests pass (6 pre-existing + 7 new). Total wall time ~0.7s.
- Observed `slog` lines confirm both the block path and inbound/outbound scan directions fire as designed (`direction=mcp_server_outbound action=block`, `direction=mcp_server_inbound action=redact`).

### `go test ./... -count=1` (regression sweep, full vv module)

All 30 packages pass. No regressions introduced by the new integration file:

```
ok  	github.com/vogo/vv/agents	0.5s
ok  	github.com/vogo/vv/cli	0.7s
ok  	github.com/vogo/vv/configs	0.8s
ok  	github.com/vogo/vv/debugs	0.8s
ok  	github.com/vogo/vv/dispatches	1.2s
ok  	github.com/vogo/vv/hooks	1.7s
ok  	github.com/vogo/vv/httpapis	2.4s
ok  	github.com/vogo/vv/integrations/agents_tests	2.6s
ok  	github.com/vogo/vv/integrations/askuser_tests	3.5s
ok  	github.com/vogo/vv/integrations/cli_tests	3.4s
ok  	github.com/vogo/vv/integrations/config_tests	4.2s
ok  	github.com/vogo/vv/integrations/costtracker_tests	4.7s
ok  	github.com/vogo/vv/integrations/debug_tests	4.7s
ok  	github.com/vogo/vv/integrations/dispatches_tests	4.6s
ok  	github.com/vogo/vv/integrations/http_tests	4.5s
ok  	github.com/vogo/vv/integrations/mcp_tests	4.4s
ok  	github.com/vogo/vv/integrations/permission_tests	4.0s
ok  	github.com/vogo/vv/integrations/project_instructions_tests	4.3s
ok  	github.com/vogo/vv/integrations/prompt_tests	62.9s
ok  	github.com/vogo/vv/integrations/setup_tests	3.7s
ok  	github.com/vogo/vv/integrations/shutdown_tests	3.4s
ok  	github.com/vogo/vv/integrations/tools_tests	3.6s
ok  	github.com/vogo/vv/integrations/vv_tests	59.3s
ok  	github.com/vogo/vv/integrations/wiring_tests	3.7s
ok  	github.com/vogo/vv/mcps	3.7s
ok  	github.com/vogo/vv/memories	3.9s
ok  	github.com/vogo/vv/registries	3.2s
ok  	github.com/vogo/vv/setup	3.2s
ok  	github.com/vogo/vv/tools	3.8s
ok  	github.com/vogo/vv/traces/costtraces	3.3s
```

## 3. Acceptance-criterion coverage map

| AC | Description | Integration test(s) |
|----|-------------|---------------------|
| AC-1.2 | `tools/list` exposes coder/researcher/reviewer (+ optional dispatcher) | `TestMCPServer_ListsExposedAgents` (existing) |
| AC-1.3 | `tools/call <agent>` routes through `agent.Run` and returns text | `TestMCPServer_CallCoderReturnsText` (existing) |
| AC-1.4 | Multiple sequential `tools/call` on one MCP session do not leak state | **`TestMCPServer_MultipleSequentialCalls` (new)** |
| AC-2.1 | Baseline safety (guardrails/path guardian/credscrub defaults on) | Implicit via AC-2.3 tests + vv unit tests in `configs/` |
| AC-2.2 | `ask_user` non-interactive in MCP mode | Out of scope for this test pass — `setup.Init` wiring; covered by `integrations/askuser_tests/` |
| AC-2.3 | credscrub log/redact/block on inbound and outbound | `TestMCPServer_CredentialFilterRedactsOutbound` (existing, outbound/redact), **`TestMCPServer_CredentialFilterBlocksOutbound` (new, outbound/block)**, **`TestMCPServer_CredentialFilterRedactsInbound` (new, inbound/redact)** |
| AC-2.4 | Permission mode forced non-interactive; dangerous bash blocked | Out of scope for this test pass — enforced via `setup.Init` not constructing `permissionState` when `cfg.Mode == "mcp"`; covered by existing `integrations/permission_tests/` unit tests. |
| AC-3.1 | `--mode mcp` defaults to stdio | `mcps/transport_test.go::TestResolveTransport_Defaults` (existing unit) |
| AC-3.2 | HTTP transport + DNS rebinding protection | `mcps/transport_test.go::TestResolveTransport_HTTPLoopback` + `_HTTPNonLoopbackRequiresToken` (existing unit); DNS rebinding itself is enforced inside the go-sdk's `StreamableHTTPHandler` |
| AC-3.3 | HTTP Bearer auth: wrong/missing → 401, correct → 2xx | **`TestMCPServer_HTTPBearerAuthRejectsMissingHeader` (new)**, **`TestMCPServer_HTTPBearerAuthAcceptsCorrectHeader` (new)** |
| AC-3.4 | Graceful shutdown <2s (both transports) | **`TestMCPServer_StdioGracefulShutdown` (new)**, **`TestMCPServer_HTTPGracefulShutdown` (new)** |
| AC-4.1 | `mcp.server.agents` whitelist filters exposure | `TestMCPServer_Whitelist` (existing); `TestMCPServer_UnknownAgentInWhitelist` (existing) |
| AC-4.2 | `expose_dispatcher` surfaces dispatcher tool | `TestMCPServer_ExposeDispatcher` (existing) |
| AC-4.3 | Env overrides (`VV_MCP_TRANSPORT`, `VV_MCP_ADDR`, `VV_MCP_AUTH_TOKEN`) | `configs/mcp_server_test.go` (existing unit) |
| AC-5.1 | Startup log includes transport/addr/tools/auth/session_timeout | Inspected in `mcps/serve.go::serve`; exercised implicitly by every new test (slog output observed in `-v`) |
| AC-5.2 | Per-call slog | Out of scope — vage-internal hooks |
| AC-5.3 | Credential hits fire `ScanCallback` | Observed in `-v` output of the three credscrub tests (`WARN vv: mcp credential scanner hit ...`) |

### Acceptance criteria not newly covered

- **AC-2.2 (`ask_user` non-interactive)** — requires driving through `setup.Init` with a MCP mode flag to install `NonInteractiveInteractor`, which is orthogonal to the `mcps` package surface. Already covered in `integrations/askuser_tests/`.
- **AC-2.4 (permission mode disabled in MCP)** — same; enforced at `main.go` / `setup.Init` layer, not in `mcps`.

These could be expanded in a follow-up test pass that spins up `setup.Init` with `cfg.Mode = "mcp"` and then invokes `mcps.Serve`, but that requires a real LLM (or a stub) and exceeds the P1-1 integration-test scope listed in `design.md §9`.

## 4. Status

- All 7 new integration tests pass.
- Full-module `go test ./...` passes (no regressions).
- No `integrations-error-*.md` file produced (no failures to triage).
- Developer loop does **not** need to re-engage.
