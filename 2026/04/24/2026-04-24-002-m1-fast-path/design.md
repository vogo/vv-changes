# M1 Fast-Path: Technical Design

## 1. Overview

Add a deterministic, zero-LLM heuristic filter in front of `Dispatcher.Run` / `Dispatcher.RunStream`. When the filter matches, the request is dispatched directly to a pre-configured sub-agent and the intent-recognition phase is skipped.

The change is **additive** — all new state lives behind a `WithFastPath` option with a sensible default; existing call sites keep working unchanged.

## 2. File Layout

| File | Change |
|------|--------|
| `vv/dispatches/fastpath.go` | **New** — `FastPathConfig`, default patterns, `fastPathClassify`, `runFastPath`, `runFastPathStream` |
| `vv/dispatches/fastpath_test.go` | **New** — unit tests for the classifier and short-circuit wiring |
| `vv/dispatches/dispatch.go` | Add `fastPath FastPathConfig` field to `Dispatcher`; add `WithFastPath(cfg FastPathConfig) Option`; call `d.fastPathClassify(req)` at top of `Run` and `RunStream`; default-initialize `fastPath` in `New` |
| `vv/configs/config.go` | Add `FastPath FastPathConfig` field to `OrchestrateConfig`; add public `FastPathConfig` type (yaml-driven) |
| `vv/setup/setup.go` | Translate `cfg.Orchestrate.FastPath` → `dispatches.FastPathConfig` and pass via `dispatches.WithFastPath` |
| `vv/setup/setup_test.go` | No forced changes; existing `nil` Options path continues to work |

## 3. Key Types

### 3.1 `dispatches.FastPathConfig`

Lives in the `dispatches` package (not `configs`) because the regex compilation is a package-internal concern. The YAML side holds a mirror struct with `[]string` patterns; `setup.go` compiles them and passes the ready-to-use dispatches-level struct in.

```go
// in vv/dispatches/fastpath.go

// FastPathRule matches a user prompt and routes it to a sub-agent.
type FastPathRule struct {
    Category string         // "greeting" | "tool_trigger"
    Pattern  *regexp.Regexp // compiled regex applied to the last user message
    Agent    string         // sub-agent ID to dispatch to
}

// FastPathConfig configures the heuristic short-circuit.
type FastPathConfig struct {
    Enabled  bool
    MaxChars int            // rune count cap for the last user message
    Rules    []FastPathRule // matched top-to-bottom; first hit wins
}
```

### 3.2 `configs.FastPathConfig` (yaml-facing)

```go
// in vv/configs/config.go

// FastPathConfig controls the dispatcher's heuristic short-circuit.
type FastPathConfig struct {
    Enabled             *bool    `yaml:"enabled,omitempty"`              // default true
    MaxChars            int      `yaml:"max_chars,omitempty"`            // default 60
    GreetingPatterns    []string `yaml:"greeting_patterns,omitempty"`    // nil = defaults; [] = disabled
    ToolTriggerPatterns []string `yaml:"tool_trigger_patterns,omitempty"`// nil = defaults; [] = disabled
}

// IsEnabled returns true unless the user explicitly set enabled: false.
func (c FastPathConfig) IsEnabled() bool {
    return c.Enabled == nil || *c.Enabled
}
```

`*bool` for `Enabled` so that `nil` means "default true" — mirrors `MCPCredentialFilterConfig.Enabled` (`configs/config.go:144`).

## 4. Default Patterns

```go
// in vv/dispatches/fastpath.go

var defaultGreetingPatterns = []string{
    `(?i)^(hi|hello|hey|thanks|thank you|bye|goodbye)(\b|$)`,
    `^(你好|您好|在吗|哈喽|再见)`,
}

var defaultToolTriggerPatterns = []string{
    `(?i)^(calc|echo|date|pwd|ls|whoami|uptime)(\s|$)`,
}

const defaultMaxChars = 60
```

Rules mapping:
- Every greeting pattern → `Agent: "chat"`.
- Every tool-trigger pattern → `Agent: "coder"`.

## 5. Classifier Algorithm

```go
// in vv/dispatches/fastpath.go

// fastPathResult is the outcome of fast-path classification.
type fastPathResult struct {
    Hit      bool
    Agent    string
    Category string
    Pattern  string
}

// fastPathClassify decides whether the request qualifies for the short-circuit.
func (d *Dispatcher) fastPathClassify(req *schema.RunRequest) fastPathResult {
    if !d.fastPath.Enabled || len(d.fastPath.Rules) == 0 {
        return fastPathResult{}
    }

    // Multi-turn context guard.
    if len(req.Messages) > 2 {
        return fastPathResult{}
    }

    // Tool-history guard: any assistant tool call or tool-role message disqualifies.
    for _, m := range req.Messages {
        if m.Role == aimodel.RoleTool || len(m.ToolCalls) > 0 {
            return fastPathResult{}
        }
    }

    // Find the last user message.
    last, ok := lastUserMessage(req.Messages)
    if !ok {
        return fastPathResult{}
    }

    text := strings.TrimSpace(last.Content.Text())
    if text == "" {
        return fastPathResult{}
    }

    if utf8.RuneCountInString(text) > d.fastPath.MaxChars {
        return fastPathResult{}
    }

    for _, rule := range d.fastPath.Rules {
        if !rule.Pattern.MatchString(text) {
            continue
        }
        if _, ok := d.subAgents[rule.Agent]; !ok {
            // Requested target is not wired in this registry — decline gracefully.
            continue
        }
        return fastPathResult{
            Hit:      true,
            Agent:    rule.Agent,
            Category: rule.Category,
            Pattern:  rule.Pattern.String(),
        }
    }

    return fastPathResult{}
}
```

Helper `lastUserMessage` scans `req.Messages` in reverse for the first `RoleUser` entry.

## 6. Dispatcher Wiring

### 6.1 `Dispatcher` struct addition (`dispatch.go`)

```go
type Dispatcher struct {
    // ... existing fields ...
    fastPath FastPathConfig
}
```

`New` initializes it to `DefaultFastPathConfig()` before applying options.

### 6.2 New option

```go
// WithFastPath configures the heuristic short-circuit filter.
func WithFastPath(cfg FastPathConfig) Option {
    return func(d *Dispatcher) {
        d.fastPath = cfg
    }
}
```

### 6.3 `DefaultFastPathConfig`

```go
// DefaultFastPathConfig returns the built-in defaults used when no explicit
// configuration is supplied.
func DefaultFastPathConfig() FastPathConfig {
    rules := make([]FastPathRule, 0, len(defaultGreetingPatterns)+len(defaultToolTriggerPatterns))
    for _, p := range defaultGreetingPatterns {
        rules = append(rules, FastPathRule{Category: "greeting", Pattern: regexp.MustCompile(p), Agent: "chat"})
    }
    for _, p := range defaultToolTriggerPatterns {
        rules = append(rules, FastPathRule{Category: "tool_trigger", Pattern: regexp.MustCompile(p), Agent: "coder"})
    }
    return FastPathConfig{Enabled: true, MaxChars: defaultMaxChars, Rules: rules}
}
```

### 6.4 `Run` short-circuit

```go
func (d *Dispatcher) Run(ctx context.Context, req *schema.RunRequest) (*schema.RunResponse, error) {
    depth := DepthFrom(ctx)

    if depth >= d.maxRecursionDepth {
        return d.fallbackRun(ctx, req, nil)
    }

    // NEW: heuristic short-circuit.
    if hit := d.fastPathClassify(req); hit.Hit {
        return d.runFastPath(ctx, req, hit)
    }

    // ... existing intent/execute/summarize pipeline ...
}
```

`runFastPath` dispatches straight to `d.subAgents[hit.Agent]` and returns — bypassing both intent and summarize for CLI, preserving current summarize-on-HTTP behavior via `shouldSummarize` after execution. See §6.6.

### 6.5 `RunStream` short-circuit

Before emitting the `phase_start("intent")` event:

```go
if hit := d.fastPathClassify(req); hit.Hit {
    return d.runFastPathStream(ctx, send, req, hit)
}
```

`runFastPathStream`:
1. Emits `EventPhaseStart{Phase: "fast_path", PhaseIndex: 1}`.
2. Emits `EventPhaseEnd{Phase: "fast_path", Summary: "Fast path -> "+agent, Duration: elapsed, PromptTokens:0, CompletionTokens:0, ToolCalls:0}`.
3. Calls `d.forwardSubAgentStream(ctx, send, subAgent, req, hit.Agent, "", sessionID)`.
4. Runs optional summarize phase (reusing existing code path) when `d.shouldSummarize(req)` is true.

### 6.6 `runFastPath` (non-stream)

```go
func (d *Dispatcher) runFastPath(ctx context.Context, req *schema.RunRequest, hit fastPathResult) (*schema.RunResponse, error) {
    slog.Info("dispatcher: fast-path hit",
        "category", hit.Category,
        "agent", hit.Agent,
        "matched_pattern", hit.Pattern,
    )

    sub, ok := d.subAgents[hit.Agent]
    if !ok {
        return d.fallbackRun(ctx, req, nil)
    }

    ctx = debugs.WithAgentName(ctx, hit.Agent)
    resp, err := d.runWithHooks(ctx, hit.Agent, req, func() (*schema.RunResponse, error) {
        return sub.Run(ctx, req)
    })
    if err != nil {
        return nil, fmt.Errorf("fast-path sub-agent %q failed: %w", hit.Agent, err)
    }

    if d.shouldSummarize(req) && len(resp.Messages) > 0 {
        summaryResp, serr := d.summarize(ctx, req, []*schema.RunResponse{resp})
        if serr == nil {
            summaryResp.Usage = aggregateUsage(resp.Usage, summaryResp.Usage)
            resp = summaryResp
        }
    }

    return resp, nil
}
```

## 7. Configuration Wiring

### 7.1 `OrchestrateConfig`

```go
type OrchestrateConfig struct {
    MaxConcurrency    int            `yaml:"max_concurrency"`
    MaxRecursionDepth int            `yaml:"max_recursion_depth"`
    SummaryPolicy     string         `yaml:"summary_policy"`
    Replan            ReplanConfig   `yaml:"replan"`
    FastPath          FastPathConfig `yaml:"fast_path,omitempty"` // NEW
}
```

### 7.2 `setup.go`

Insert **before** the `dispatches.New` call:

```go
fastPath := buildFastPath(cfg.Orchestrate.FastPath)
// ...
dispatches.New(..., dispatches.WithFastPath(fastPath), ...)
```

`buildFastPath` (also in `setup.go`, private helper):
- If `!cfg.IsEnabled()` → return `dispatches.FastPathConfig{Enabled: false}`.
- If `MaxChars == 0` → use default.
- Compile `GreetingPatterns`; fall back to `defaultGreetingPatterns` when the slice is `nil`. For an **empty slice**, skip the category. Invalid regexes are logged via `slog.Warn` and dropped (matches `MCPCredentialFilterConfig.ExtraPatterns` behavior).
- Same for `ToolTriggerPatterns`.

## 8. Test Plan

### 8.1 Unit tests (`fastpath_test.go`)

| Test | Intent |
|------|--------|
| `TestFastPath_GreetingHitsChat` | `"hello"` → hit, agent `chat`. |
| `TestFastPath_CaseInsensitive` | `"Hello"`, `"HI"`, `"Thanks"` all hit. |
| `TestFastPath_ChineseGreetings` | `"你好"`, `"在吗"` hit with rune length 2. |
| `TestFastPath_ToolTriggerHitsCoder` | `"calc 5^6"`, `"date"`, `"pwd"` → hit, agent `coder`. |
| `TestFastPath_LongMessageDeclines` | 65-char greeting → no hit. |
| `TestFastPath_WordBoundary` | `"helloworld"` does not hit (non-word-boundary). |
| `TestFastPath_DeclinesWhenMultiTurn` | `len(Messages) == 3` → no hit. |
| `TestFastPath_DeclinesWhenToolHistoryPresent` | Message with `ToolCalls` → no hit. |
| `TestFastPath_DeclinesWhenDisabled` | `Enabled=false` → no hit even for `"hello"`. |
| `TestFastPath_DeclinesWhenAgentMissing` | `subAgents` missing `coder`; tool trigger declines. |
| `TestFastPath_EmptyTextDeclines` | `""` or whitespace-only → no hit. |

### 8.2 Dispatcher integration test (in `dispatch_test.go` or new file)

- `TestDispatcher_FastPathShortCircuits`: mock `ChatCompleter` counting intent calls; send `"hello"`; assert exactly **zero** intent LLM calls and exactly **one** chat LLM call.
- `TestDispatcher_FastPathStreamEmitsPhase`: stream mode; assert event sequence contains `phase_start{Phase:"fast_path"}` then `phase_end{Phase:"fast_path", ToolCalls:0, PromptTokens:0, CompletionTokens:0}`.

### 8.3 Config parsing test (`configs/config_test.go`)

- Enabled default true when `orchestrate.fast_path` is omitted.
- `enabled: false` disables the feature.
- Invalid regex pattern logs a warning and is dropped.

## 9. Observability

- `slog.Info` on each hit (see §6.6).
- Stream phase events encode zero-cost runs.
- No new metrics emitted in M1 — `phase_tracker` already counts tokens/tool calls; the fast-path phase simply reports zeros.

## 10. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Regex false positives trigger wrong agent | Conservative default set + strict 60-char cap + multi-turn guard |
| Users with custom `ChatCompleter` paths now see an extra code path | `WithFastPath` defaults preserve behavior; `Enabled=false` is a one-line kill switch |
| Invalid user regex breaks startup | Logged and dropped during `buildFastPath`; never fatal |
| Extending to new greetings requires a release | Config exposes `greeting_patterns` override |

## 11. Simplicity Check

- One new file (`fastpath.go`) plus its test.
- `dispatch.go` gains 1 field, 1 option, ~4 new lines in `Run`, ~4 new lines in `RunStream`.
- `configs/config.go` gains one struct and one field.
- `setup.go` gains ~15 lines of translation code.
- Total diff well under ~400 lines including tests.

A senior engineer would call this surgical. No new abstractions, no new events, no new agents, no speculative extensibility.
