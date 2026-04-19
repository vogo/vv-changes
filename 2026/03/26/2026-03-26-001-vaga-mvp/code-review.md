# Code Review: vv MVP

## Summary

The implementation is well-structured, closely follows the design document, and correctly uses vage framework APIs throughout. All 25 unit tests pass. `go build` and `go vet` produce no errors or warnings.

## Files Reviewed

| File | Lines | Purpose |
|------|-------|---------|
| `vv/config.go` | 139 | Config structs, YAML loading, env var overrides, LLM client creation |
| `vv/config_test.go` | 300 | 10 tests covering YAML parsing, env overrides, defaults, provider behavior |
| `vv/tools.go` | 60 | Tool registration with bashtool, readtool, writetool, edittool, globtool, greptool |
| `vv/tools_test.go` | 72 | 3 tests verifying tool registration count, custom options, default config |
| `vv/agents.go` | 57 | Agent/router creation wiring |
| `vv/agents_test.go` | 273 | 6 tests including mock LLM, stub agents, router routing verification |
| `vv/prompts.go` | 27 | System prompt constants for coder and chat agents |
| `vv/prompts_test.go` | 33 | 4 tests verifying prompt content |
| `vv/main.go` | 90 | CLI entry point, flag parsing, signal handling, service startup |
| `vv/go.mod` | 11 | Module definition with local replace directive |

## Verdict: PASS

No blocking issues found. The code is production-quality for an MVP. Two minor improvements applied (documented below).

## Findings

### Issue 1: Redundant BaseURL fallback in newLLMClient (Severity: Low - Informational)

**Location**: `config.go`, lines 129-133

`loadConfig()` already sets `cfg.LLM.BaseURL = "https://api.openai.com/v1"` for OpenAI/empty providers (line 99-101). The identical fallback in `newLLMClient()` (lines 129-133) is therefore unreachable when called from `main()`.

However, since `newLLMClient` is a standalone function that tests call directly (without going through `loadConfig`), having this defensive fallback is actually good practice. It makes `newLLMClient` self-contained.

**Action**: Added a clarifying comment to explain the intentional redundancy. No logic change needed.

### Issue 2: API key in Config struct lacks redaction guidance (Severity: Low)

**Location**: `config.go`, line 23

The `LLMConfig.APIKey` field could be inadvertently exposed if the struct is logged or serialized. The design document specifies "redact API key" in logging. The current `main.go` correctly avoids logging it (only logs provider and model), but there is no structural protection.

**Action**: Added a comment on the APIKey field noting it should never be logged. A `fmt.Stringer` implementation for `LLMConfig` is overkill at MVP stage but recommended for production.

### Positive Observations

1. **Correct vage API usage**: All framework APIs (`taskagent.New`, `routeragent.New`, `routeragent.LLMFunc`, `service.New`, `tool.NewRegistry`, `prompt.StringPrompt`, tool `Register` functions) are used with correct signatures and semantics.

2. **Proper error handling**: Every error return is checked and wrapped with context (e.g., `fmt.Errorf("register bash tool: %w", err)`).

3. **Clean flag handling**: The `flag.Visit` technique to detect explicit `-config` usage is idiomatic Go.

4. **Good test coverage**: Tests cover happy paths, edge cases (missing files, env overrides, empty config), and integration between components (router routing tests with mock LLM).

5. **Correct ChatCompleter interface usage**: The `mockChatCompleter` in tests properly implements both `ChatCompletion` and `ChatCompletionStream` methods from `aimodel.ChatCompleter`.

6. **Appropriate MaxIterations(1) for chat agent**: Prevents unnecessary ReAct loops for the tool-less chat agent.

7. **Signal handling**: Uses `signal.NotifyContext` with proper `defer stop()` cleanup.

8. **Service wiring**: All three agents registered individually, enabling direct access via `/v1/agents/{id}/run`.
