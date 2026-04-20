# Code Review — P1-1 vv MCP Server Integration

Scope: `vv/configs/config.go`, `vv/configs/mcp_server_test.go`, `vv/mcps/*`, `vv/integrations/mcp_tests/mcp_server_test.go`, `vv/main.go`, `vv/go.{mod,sum}`.

Baseline before review: `go test ./... -count=1` → all packages pass; `golangci-lint run ./...` → 0 issues.

---

## Findings

### 1. `isLoopbackAddr` treats bare-port addresses (`":8080"`) as loopback — critical

**File**: `vv/configs/config.go` lines 436–452 (`isLoopbackAddr`) and `vv/mcps/transport.go` lines 65–81 (`IsLoopbackAddr`).

`net.SplitHostPort(":8080")` returns `host=""`, and the current code classifies empty-host as loopback. But `net.Listen("tcp", ":8080")` binds to **all interfaces** (wildcard), so a user config of

```yaml
mcp:
  server:
    transport: http
    addr: ":8080"
```

passes the non-loopback/no-token guard in `ValidateMCPServer` and ends up binding a public, unauthenticated HTTP MCP server. The design explicitly calls this out (`§3.5`: *"非 loopback 且 AuthToken 为空 → 返回启动错误 (defense-in-depth，避免裸 bind 0.0.0.0)"*).

**Severity**: critical (security). **Decision**: applied — an empty host is treated as non-loopback in both `configs.isLoopbackAddr` and `mcps.IsLoopbackAddr`; the test case `"": true` is replaced with `"": false` and `":8080": false` is added. The bare-string fallback (e.g. `"localhost"` with no port) still resolves correctly through the hostname branch.

### 2. `ResolveTransport` does not re-enforce the non-loopback auth guard — major

**File**: `vv/mcps/transport.go`.

`ValidateMCPServer` in `configs.Load` is the only guard today. Any caller that constructs `*configs.Config` directly (the integration tests do) bypasses it. The design asked for defense-in-depth layering. The pure-config path in `mcps.Serve` also relies on `ValidateMCPServer` having been run.

**Severity**: major (defence-in-depth). **Decision**: applied — `ResolveTransport` re-runs the `!IsLoopbackAddr(addr) && AuthToken==""` check. Cheap and keeps `mcps` usable stand-alone. Covered with a new `TestResolveTransport_HTTPNonLoopbackRequiresToken` unit test.

### 3. `serveHTTP` leaks the shutdown goroutine on non-Shutdown Serve returns — minor

**File**: `vv/mcps/serve.go` lines 164–167.

```go
go func() { <-ctx.Done(); _ = httpSrv.Shutdown(context.Background()) }()
if err := httpSrv.Serve(ln); err != nil && !errors.Is(err, http.ErrServerClosed) {
    return fmt.Errorf(...)
}
```

If `Serve` exits with an unexpected error (e.g. `accept tcp: ...: use of closed network connection` from external listener close) the goroutine survives until `ctx` is eventually canceled. In a long-lived parent context this is an accumulating leak.

**Severity**: minor. **Decision**: applied — wrap the shutdown goroutine so it returns promptly once `Serve` returns. Uses a `done` channel closed in `defer` after `Serve` returns.

### 4. Flag help and `--mode` usage string still reference only `cli`/`http` — minor

**File**: `vv/main.go` lines 28, 41.

```go
modeFlag := flag.String("mode", "", "run mode: cli or http (default: cli)")
...
fmt.Fprintf(os.Stderr, "  VV_MODE             Run mode (cli or http)\n")
```

The feature adds a third mode but the `--help` output does not surface it.

**Severity**: minor (docs/UX). **Decision**: applied — updated both strings to include `mcp`.

### 5. `BuildServer` returns four values — minor

**File**: `vv/mcps/serve.go` lines 91–127.

Four-tuple returns are a mild smell, but the signature is justified: callers (logger, tests) need to know what was exposed before `Serve` blocks. `toolNames(exposed, exposedDispatcher)` is the only consumer of the middle two values, and the tests make it free to inspect registration without cracking open the underlying `*mcp.Server`.

**Severity**: trivial. **Decision**: deferred — not worth introducing a wrapper struct for two call sites (`serve()` and three tests). Leaving as-is.

### 6. `echoAgent.Run` returns `schema.NewUserMessage(reply)` in integration tests — trivial

**File**: `vv/integrations/mcp_tests/mcp_server_test.go` line 62.

Semantically the agent's reply should be an assistant message, not user. Because `vage/mcp/server` only reads `resp.Messages[0].Content.Text()`, the role is not observed, so behaviour is identical. Still semantically wrong for future readers.

**Severity**: trivial. **Decision**: deferred — purely cosmetic in a test that does not assert role, and the noise of fixing it is not proportional to the value.

### 7. credscrub integration — reviewed, no change needed

`BuildServer` calls `configs.BuildMCPCredentialScanner(cfg.Security.MCPCredentialFilter)` exactly once and hands it to `vage/mcp/server` via `WithCredentialScanner`. The vage server applies inbound scanning on raw request args (`ScanJSONMap`) and outbound scanning on the text reply (`ScanText`) — no double scan. The direction is correct: `server_inbound` for handler args, `server_outbound` for handler outputs. The callback only logs the `Masked`/summary fields; plaintext is never logged.

**Severity**: none. **Decision**: reviewed — OK.

### 8. Thread safety — reviewed, no change needed

`vage/mcp/server.Server.RegisterAgent` takes `s.mu.Lock()` internally; the underlying go-sdk `mcp.Server.AddTool` is also safe to call pre-Serve. All registrations in `BuildServer` run synchronously on a single goroutine before `Serve` starts, so there is no concurrent registration window.

### 9. MCP mode lifecycle — reviewed, no change needed

`main.go` isolates MCP from CLI cleanly: `UserInteractor = NonInteractiveInteractor{}` when mode is http or mcp; `WrapToolRegistry` is skipped when mode is mcp; `permissionState.SetNonInteractive(...)` is set for both http and mcp. `permissionState` still constructed for `mcp` mode but never used (no wrapping) — harmless. The `-p` flag guard (`-p incompatible with mcp mode`) is in place.

One cosmetic note: `permissionState` is constructed and `SetPathGuardian` called even when unused for `mcp`. Could short-circuit this but the state is tiny and not returning early keeps the mainline flat. **Decision**: deferred.

### 10. Config defaults — reviewed, no change needed

Existing `vv.yaml` files that don't mention `mcp:` load as a zero-value `MCPConfig`, which validates to `transport=stdio` (no addr/token). Mode still defaults to `cli`, so users who never pass `--mode mcp` see identical behaviour. No breaking change.

---

### 11. `mcp_server_extra_test.go` tripped errcheck on `resp.Body.Close()` — minor

**File**: `vv/integrations/mcp_tests/mcp_server_extra_test.go` lines 268, 284, 313, 335.

Four `resp.Body.Close()` calls were un-guarded and triggered the `errcheck` linter. Not part of the originally listed candidate files, but surfaces when lint runs the whole module.

**Severity**: minor. **Decision**: applied — wrapped in `defer func() { _ = resp.Body.Close() }()` / `_ = resp.Body.Close()` as appropriate.

## Summary of applied changes

1. `vv/configs/config.go` — `isLoopbackAddr("")` now returns `false`; doc comment updated.
2. `vv/configs/mcp_server_test.go` — added `TestValidateMCPServer_BarePortRequiresToken`.
3. `vv/mcps/transport.go` — `IsLoopbackAddr("")` now returns `false`; `ResolveTransport` re-enforces the non-loopback + auth guard.
4. `vv/mcps/transport_test.go` — updated loopback table (`""` and `":8080"` now false); added `TestResolveTransport_HTTPNonLoopbackRequiresToken`.
5. `vv/mcps/serve.go` — shutdown goroutine guarded with a `done` channel so it exits on any `Serve` return.
6. `vv/main.go` — `--mode` flag description and `VV_MODE` help now include `mcp`.
7. `vv/integrations/mcp_tests/mcp_server_extra_test.go` — errcheck-clean `resp.Body.Close()` calls.

## Tests after changes

- `go test ./... -count=1` → all packages pass (30/30).
- `golangci-lint run ./...` → 0 issues.
- `go build ./...` → clean.
