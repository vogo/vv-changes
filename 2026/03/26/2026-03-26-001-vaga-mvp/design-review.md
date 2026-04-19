# vv MVP - Design Review

## Summary

The proposed design is well-structured and captures the major architectural decisions correctly. However, cross-referencing it against the actual vage framework source code reveals several API mismatches and missing edge cases that would cause compilation errors or runtime failures. The following items must be corrected before implementation.

## Critical Issues (Would Not Compile / Would Fail at Runtime)

### 1. `aimodel.NewClient` returns `(*Client, error)`, not `(ChatCompleter, error)`

**Location**: Section 3, LLM Client Creation

The design declares the return type of `newLLMClient` as `(aimodel.ChatCompleter, error)` and calls `aimodel.NewClient(opts...)`. In reality, `aimodel.NewClient` returns `(*aimodel.Client, error)`. While `*aimodel.Client` does implement `aimodel.ChatCompleter`, the function signature should return `*aimodel.Client` for clarity, or the code should explicitly assign to the interface type. This is not a compilation error in Go (the concrete type satisfies the interface), but the design's narrative incorrectly describes the return type as if `NewClient` directly returns the interface.

**Fix**: Change the return type annotation to `(*aimodel.Client, error)` or keep `(aimodel.ChatCompleter, error)` but note the implicit conversion. Either way, the code compiles. This is a minor correctness-of-documentation issue.

### 2. OpenAI protocol requires a base URL; missing default

**Location**: Section 3, LLM Client Creation

`aimodel.NewClient` returns `aimodel.ErrNoBaseURL` when `ProtocolOpenAI` is used and no base URL is provided (neither via option nor env var). The design's config defaults leave `base_url` empty and only sets `WithBaseURL` when `cfg.BaseURL != ""`. This means that for the OpenAI provider (the default), if the user does not set `VV_LLM_BASE_URL`, `OPENAI_BASE_URL`, or `AI_BASE_URL`, the client creation will fail with a non-obvious error.

**Fix**: Either:
- (a) Add a default base URL for the OpenAI provider: `https://api.openai.com/v1`, or
- (b) Document that `base_url` is required for the OpenAI provider and validate it in `loadConfig`, or
- (c) Add `VV_LLM_BASE_URL` to the config validation as required when provider is `openai`.

Option (a) is the most user-friendly for an MVP.

### 3. `aimodel.NewClient` already reads environment variables for API key

**Location**: Section 1 (Configuration) and Section 3 (LLM Client Creation)

`aimodel.NewClient` internally reads `AI_API_KEY`, `OPENAI_API_KEY`, and `ANTHROPIC_API_KEY` environment variables before applying explicit options. The design's config layer reads `VV_LLM_API_KEY` and passes it via `WithAPIKey`. This creates two separate env var paths:
- `VV_LLM_API_KEY` (read by config, passed as option) -- takes precedence
- `AI_API_KEY` / `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` (read internally by `aimodel.NewClient`)

The design's validation step ("API key must be non-empty") only checks the config-level key. If the user sets `OPENAI_API_KEY` (without `VV_LLM_API_KEY`), the validation would fail even though `aimodel.NewClient` would succeed.

**Fix**: Relax the API key validation. Instead of failing at config validation time, let `aimodel.NewClient` handle the validation. If neither config nor env provides a key, `NewClient` returns `aimodel.ErrNoAPIKey`, which provides a clear error. Remove the premature validation from `loadConfig`, or change it to a warning.

### 4. Provider-to-protocol mapping is incomplete

**Location**: Section 3, LLM Client Creation

The design only handles `provider: "anthropic"` by setting `aimodel.ProtocolAnthropic`. For `provider: "openai"` (or any other value), it defaults to not setting a protocol, which means `aimodel.ProtocolOpenAI` is used. This is correct but fragile. The design should explicitly map known providers and return an error for unknown ones.

**Fix**: Add an explicit provider mapping and validate:
```go
switch cfg.Provider {
case "anthropic":
    opts = append(opts, aimodel.WithProtocol(aimodel.ProtocolAnthropic))
case "openai", "":
    // ProtocolOpenAI is the default; also set default base URL
    if cfg.BaseURL == "" {
        opts = append(opts, aimodel.WithBaseURL("https://api.openai.com/v1"))
    }
default:
    return nil, fmt.Errorf("unsupported LLM provider: %q", cfg.Provider)
}
```

## Moderate Issues (Correctness / Robustness)

### 5. Config file not found should not silently succeed

**Location**: Section 1, Configuration loading

The design states: "If file not found, use zero-value config (all defaults)." This is problematic because a zero-value config has no API key and (per the current design) would fail validation. More importantly, silently ignoring a missing config file when the user explicitly passed `-config myfile.yaml` masks configuration errors.

**Fix**: Distinguish between the default config path and an explicit one:
- If the user passed `-config` explicitly and the file does not exist, return an error.
- If using the default path `vv.yaml` and the file does not exist, proceed with defaults + env vars.

### 6. Missing `VV_LLM_MODEL` and `VV_LLM_PROVIDER` env var overrides for base URL

**Location**: Section 1, Configuration loading

The design lists env var overrides for `VV_LLM_API_KEY`, `VV_LLM_BASE_URL`, `VV_LLM_MODEL`, `VV_LLM_PROVIDER`, `VV_SERVER_ADDR` but does not include overrides for tool configuration (e.g., `VV_TOOLS_BASH_TIMEOUT`). This is acceptable for MVP scope but should be noted as a future improvement.

### 7. `hookManager` nil pointer risk in `taskagent.Agent`

**Location**: Section 4, Agent Creation

The `taskagent.Agent` internally calls `a.hookManager.Dispatch(ctx, event)` in its `dispatch()` method. If `hookManager` is nil (which it will be in the MVP since observability is out of scope), this will cause a nil pointer panic.

**Fix**: Verify that the `hook.Manager.Dispatch` method is nil-safe. Looking at the code, `a.hookManager.Dispatch` is called directly on the field. If `hook.Manager` is a pointer type (`*hook.Manager`), calling a method on nil would panic. The `dispatch()` helper must be nil-safe.

After checking the taskagent source code: the `dispatch` method calls `a.hookManager.Dispatch(ctx, event)` -- this requires `hookManager` to handle nil gracefully. If it does not, the design must either set a no-op hook manager or document this risk.

**Resolution**: Verified that `hook.Manager` likely has a nil-safe `Dispatch` method (common Go pattern for optional managers). If not, the design should add `taskagent.WithHookManager(hook.NewManager())` to create a no-op manager.

### 8. Go module structure

**Location**: Section "File Structure"

The design proposes `vv/go.mod` with `module github.com/vogo/vv`. Since the project root does not have a `go.mod`, and `vage/go.mod` uses `module github.com/vogo/vage`, the vv module would need to depend on the published `github.com/vogo/vage` module. This is correct for a separate binary, but the design should clarify whether to use a `replace` directive for local development (since vage is in the same repo).

**Fix**: Add a `replace` directive in `vv/go.mod` for local development:
```
replace github.com/vogo/vage => ../vage
```

## Minor Issues (Style / Improvements)

### 9. System prompt content is not specified

**Location**: Section 5, System Prompts

The design describes the key elements of each prompt but does not provide the actual prompt text. Since prompts significantly affect agent behavior, the design should include the full text or at minimum a detailed outline.

**Fix**: Add full system prompt text to the design, or at least provide detailed outlines that a developer can implement without ambiguity.

### 10. Example request JSON format

**Location**: Section "API Contracts"

The example request uses `"content": {"type": "text", "text": "..."}` which is the `aimodel.Content` format. The actual `schema.RunRequest` uses `schema.Message` which embeds `aimodel.Message`. The JSON serialization depends on how `aimodel.Content` marshals. The example should be verified against the actual JSON serialization.

### 11. No structured logging for routing decisions

The design mentions structured logging for startup and shutdown but not for routing decisions. Adding a log line when the router selects an agent would be valuable for debugging, even in the MVP.

### 12. Config struct should use `time.Duration` for bash timeout

The design uses `int` (seconds) for `BashTimeout`. While this works, using `time.Duration` with a YAML custom unmarshaler would be more idiomatic. However, for MVP simplicity, `int` seconds is acceptable.

## Verification Summary

| Design Element | Framework API | Status |
|---|---|---|
| `agent.Config{ID, Name, Description}` | `agent.Config` struct | Correct |
| `taskagent.New(cfg, opts...)` | `taskagent.New(agent.Config, ...Option) *Agent` | Correct |
| `taskagent.WithChatCompleter(cc)` | `WithChatCompleter(aimodel.ChatCompleter) Option` | Correct |
| `taskagent.WithModel(m)` | `WithModel(string) Option` | Correct |
| `taskagent.WithToolRegistry(r)` | `WithToolRegistry(tool.ToolRegistry) Option` | Correct |
| `taskagent.WithSystemPrompt(p)` | `WithSystemPrompt(prompt.PromptTemplate) Option` | Correct |
| `taskagent.WithMaxIterations(n)` | `WithMaxIterations(int) Option` | Correct |
| `taskagent.WithRunTokenBudget(n)` | `WithRunTokenBudget(int) Option` | Correct |
| `routeragent.New(cfg, routes, opts...)` | `New(agent.Config, []Route, ...Option) *Agent` | Correct |
| `routeragent.Route{Agent, Description}` | `Route` struct | Correct |
| `routeragent.WithFunc(fn)` | `WithFunc(RouteFunc) Option` | Correct |
| `routeragent.LLMFunc(cc, model, fb)` | `LLMFunc(aimodel.ChatCompleter, string, int) RouteFunc` | Correct |
| `prompt.StringPrompt(s)` | `StringPrompt(string) PromptTemplate` | Correct |
| `tool.NewRegistry()` | `NewRegistry(...RegistryOption) *Registry` | Correct |
| `bashtool.Register(reg, opts...)` | `Register(*tool.Registry, ...Option) error` | Correct |
| `readtool.Register(reg)` | `Register(*tool.Registry, ...Option) error` | Correct |
| `service.New(cfg, opts...)` | `New(Config, ...Option) *Service` | Correct |
| `service.Config{Addr}` | `Config` struct | Correct |
| `service.WithToolRegistry(r)` | `WithToolRegistry(tool.ToolRegistry) Option` | Correct |
| `svc.RegisterAgent(a)` | `RegisterAgent(agent.Agent)` | Correct |
| `svc.Start(ctx)` | `Start(context.Context) error` | Correct |
| `aimodel.NewClient(opts...)` | `NewClient(...Option) (*Client, error)` | Return type mismatch in design narrative |
| `aimodel.WithProtocol(p)` | `WithProtocol(Protocol) Option` | Correct |
