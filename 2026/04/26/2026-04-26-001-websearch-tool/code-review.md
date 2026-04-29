# P2-10 WebSearch — Code Review

Reviewer: Claude Opus 4.7 (1M ctx)
Date: 2026-04-26

## Files reviewed

vage:
- `vage/tool/websearch/provider.go`
- `vage/tool/websearch/envelope.go`
- `vage/tool/websearch/websearch_tool.go`
- `vage/tool/websearch/tavily.go`
- `vage/tool/websearch/brave.go`
- `vage/tool/websearch/util.go`
- `vage/tool/websearch/websearch_tool_test.go`

vv:
- `vv/configs/config.go` (web_search additions, env overrides, soft-warn)
- `vv/configs/web_search_test.go`
- `vv/registries/tool_access.go` + `tool_access_test.go`
- `vv/tools/tools.go` + `tools_test.go`
- `vv/tools/websearch_factory.go`
- `vv/agents/{coder,researcher,reviewer,primary}.go`
- `vv/debugs/toolregistry.go`

## Audit method

Cross-referenced each file against the design (`design.md`) and AC matrix
(`requirement.md`). Walked the call graph from LLM tool call → Handler →
Provider → HTTP → envelope. Tracked every place the api_key, raw query,
Provider error string, and HTTPError body could leak. Looked specifically for:

- Typed-nil interface traps when wiring providers.
- Pre-flight argument validation (must run before any HTTP call).
- API key handling (header / body only; never envelope, log, or error string).
- Error translation correctness (timeout vs canceled vs HTTPError vs parse).
- Body / snippet / query / topic size caps.
- Test coverage of AC bullets.

## Findings & decisions

### F1 — `query_too_long` envelope drops the echoed query (BUG)

`websearch_tool.go:183` constructs the error with `errorResult(query[:0], ...)`
which is a zero-length slice → empty string. The envelope's `query` field
ends up `""`, so the LLM gets a useless echo and cannot self-correct ("you
asked for `foo bar baz...`, retry with a shorter query").

**Decision: FIX.** Echo a runic-truncated prefix of the input so the LLM can
recognise its own request without bloating the envelope.

### F2 — Typed-nil interface trap in `buildWebSearchProvider` (LATENT)

`buildWebSearchProvider` returns `websearch.Provider` (interface) by directly
returning `websearch.NewTavily(cfg.APIKey)` / `NewBrave(...)`. Those
constructors return a concrete `*TavilyProvider` / `*BraveProvider` that may
be nil when the api key is empty. Assigning a nil concrete pointer to an
interface produces a non-nil interface with a nil concrete value — the
`if provider == nil` guard in `MaybeRegisterWebSearch` will not catch it,
and downstream `provider.Name()` will panic.

In practice the `IsEnabled()` gate guarantees the api key is non-empty, so
the constructors will always return non-nil here. But this is a footgun for
future maintainers (e.g. if `IsEnabled` semantics change, or if a future
provider has stricter validation than just "key non-empty"). The defensive
fix is one line.

**Decision: FIX.** Convert through concrete typed locals and explicitly
return `nil` from `buildWebSearchProvider` if the constructor returned a nil
pointer, so the caller's `provider == nil` check actually catches it.

### F3 — Topic string is forwarded to Tavily without bounds (MINOR)

The LLM-supplied `topic` field is trimmed but otherwise unbounded. Tavily
accepts only `"general"` / `"news"`; any other value causes Tavily to reject
with 4xx and surfaces as `provider_error`. That's correct behaviour but
wastes a round-trip and lets the LLM smuggle large strings through the
context value into the request body.

**Decision: FIX (defensive).** Cap topic length pre-flight (64 chars is
plenty for any realistic enum value); over-cap → drop topic and log a
warning (ignore silently is fine since topic is optional).

### F4 — API key leakage surface review (NO-CHANGE)

Walked every spot the api_key passes through:

- Tavily: api_key is in JSON request body (`tavilyRequest.APIKey`). Provider
  errors wrap `httpClient.Do` which produces `*url.Error` containing the
  endpoint URL but **not** the body. `HTTPError.Body` carries upstream
  response body only (truncated to 1 MiB), never the request.
- Brave: api_key is in `X-Subscription-Token` header. Same headers are not
  echoed by `*url.Error`.
- Envelope: never includes api_key.
- `slog.Warn` lines in config: log `provider`, raw provider value, and
  `timeout_seconds` / `max_results`. Never `api_key`. ✓

**Decision: NO CHANGE** — also added a regression test (T1 below).

### F5 — Pre-flight validation runs before HTTP (CORRECT)

- empty query: returns before provider.Search.
- query > 1024 runes: returns before provider.Search.
- max_results clamp: applied before provider.Search.
- bad JSON args: returns before provider.Search.

All four asserted by existing unit tests. ✓

### F6 — Error translation order in `translateProviderError` (CORRECT)

Order: `ErrInvalidAPIKey` → `*HTTPError` → `DeadlineExceeded` → `Canceled` →
`parseError` → fall-through `request_failed`. `errors.Is` / `errors.As`
unwrap %w chains, so a Tavily/Brave error wrapping `context.DeadlineExceeded`
is correctly mapped to `timeout`. parseError wraps json decode errors only,
which never wrap context errors, so order is safe.

### F7 — SSRF / open-redirect surface (NO CONCERN)

`web_search` itself never follows URLs returned by the provider; it only
echoes `url` strings into the envelope. The downstream `web_fetch` retains
its own `blockPrivateAddrControl`. AC-3.2 holds.

The provider HTTP calls themselves go to fixed endpoints
(`api.tavily.com`, `api.search.brave.com`); the LLM cannot redirect them.
Test endpoints are overridable via `WithTavilyEndpoint` / `WithBraveEndpoint`
but those are package-private to vv wiring (vv only calls `NewTavily(key)`
and `NewBrave(key)` without endpoint options).

### F8 — Body size caps (CORRECT)

Both providers wrap response reads in `io.LimitReader(_, 1 MiB)` for both
the success path and the non-2xx path. Snippet truncation (`280` runes per
result) caps per-result size after parse.

### F9 — Concurrency (NO CONCERN)

`Tool` is immutable post-construction. No shared mutable state. Each
`Search` call gets its own request and uses the per-call timeout context.

### F10 — `RegisterIfAbsent` vs `Register` choice (CORRECT)

`websearch.Register` calls `RegisterIfAbsent`, matching the webfetch pattern
and avoiding TOCTOU on a hot-reload path.

## Test coverage gaps vs AC

| AC bullet | Covered? |
|-----------|----------|
| AC-1.1 default 5 / cap 20 / clamp warning | ✓ TestWebSearch_Success, TestWebSearch_MaxResultsClamped |
| AC-1.2 Provider order preserved | implicit (slice order) |
| AC-1.3 0 results → empty results + no_results warning | ✓ TestWebSearch_NoResults |
| AC-2.1 missing key → unregistered | ✓ default cfg path in tools_test.go (count=7) |
| AC-2.2 unknown provider warns + skips | ✓ TestRegister_WebSearchUnknownProvider + TestLoad_WebSearchUnknownProviderDisabled |
| AC-2.3 cross-provider key mismatch | covered by IsEnabled requiring both |
| AC-2.4 env override priority | ✓ TestLoad_WebSearchEnvOverride |
| AC-3.1 url passthrough | partial — added test asserting URL echo (T2) |
| AC-4.1 query > 1024 → invalid_arguments | ✓ TestWebSearch_QueryTooLong (added query echo assertion) |
| AC-4.3 4xx/5xx → provider_error + status_code | ✓ TestWebSearch_HTTPError, TestTavilyProvider_ServerError |
| AC-4.4 timeout → error_code:timeout | ✓ TestWebSearch_Timeout |
| AC-5.2 envelope contains no api_key | added regression test (T1) |

## Tests added

- **T1** `TestWebSearch_EnvelopeOmitsAPIKey` — drives the Handler with a
  stub provider whose Name embeds a sentinel; asserts the marshaled
  envelope text contains neither the api key string nor any of the upstream
  request fields.
- **T2** `TestWebSearch_QueryTooLongEchoesTruncatedQuery` — extends the
  existing query-too-long case to assert the envelope echoes a truncated
  prefix (post F1 fix).
- **T3** `TestBuildWebSearchProvider_NilSafe` (vv) — exercises the typed-nil
  defence in the factory.

## Changes applied (summary)

1. `websearch_tool.go`: query echo on `query_too_long` truncates to a safe
   prefix instead of `query[:0]` (F1).
2. `websearch_tool.go`: pre-flight cap on `topic` length (F3).
3. `vv/tools/websearch_factory.go`: typed-nil defence on
   `buildWebSearchProvider` return (F2).
4. `vage/tool/websearch/websearch_tool_test.go`: added T1, extended T2.
5. `vv/tools/tools_test.go`: added T3.

## Result

All targeted unit tests + golangci-lint stayed green after each change.
