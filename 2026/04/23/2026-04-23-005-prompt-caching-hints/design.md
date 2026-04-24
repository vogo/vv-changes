# Design — P1-8 · Prompt caching hints

## 1. Approach

Thin, two-layer change:

- **aimodel layer** owns the on-wire `cache_control` translation. Adds a struct-local `CacheBreakpoint bool` field on `Message` and `Tool` — never serialised on the canonical JSON side; consumed only by the Anthropic translator.
- **taskagent layer** decides which blocks to mark. When `WithPromptCaching(true)` (default), it sets `CacheBreakpoint=true` on the system message and on the last tool in the request. vv wires the toggle from `cfg.Agents.PromptCaching`.

Zero changes to the OpenAI translation path; OpenAI's automatic prefix caching needs no hint. OpenAI-compatible providers that reject unknown fields never see `cache_control`.

## 2. aimodel changes

### 2.1 Schema additions

```go
// In aimodel/schema.go

type Message struct {
    Role       Role       `json:"role"`
    Content    Content    `json:"content"`
    Thinking   string     `json:"reasoning_content,omitempty"`
    ToolCallID string     `json:"tool_call_id,omitempty"`
    ToolCalls  []ToolCall `json:"tool_calls,omitempty"`

    // CacheBreakpoint, when true, tells protocol backends that support
    // explicit prompt caching (Anthropic) to emit a cache boundary at the
    // end of this message's content blocks. OpenAI-compatible backends
    // silently ignore this field — OpenAI caches 1024-token+ prefixes
    // automatically without request-side hints.
    //
    // Struct-local: never serialised on the canonical (OpenAI-shape)
    // request body. Only the Anthropic translator reads it.
    CacheBreakpoint bool `json:"-"`
}

type Tool struct {
    Type     string             `json:"type"`
    Function FunctionDefinition `json:"function"`

    // CacheBreakpoint, when true, marks this tool as the end of a cache
    // prefix for Anthropic requests. Anthropic caches all tools up to and
    // including the marked one. OpenAI ignores the field.
    CacheBreakpoint bool `json:"-"`
}
```

`json:"-"` keeps the canonical OpenAI shape untouched and guarantees no accidental leakage to providers that don't understand the field.

### 2.2 Anthropic translation — helper type

Extend `anthropicContentBlock` with an optional cache-control field:

```go
type anthropicContentBlock struct {
    Type          string                  `json:"type"`
    Text          string                  `json:"text,omitempty"`
    // ... existing fields unchanged ...
    CacheControl  *anthropicCacheControl  `json:"cache_control,omitempty"`
}

type anthropicCacheControl struct {
    Type string `json:"type"` // always "ephemeral" in this PR
}
```

`anthropicTool` gets the same trailing field:

```go
type anthropicTool struct {
    Name         string                 `json:"name"`
    Description  string                 `json:"description,omitempty"`
    InputSchema  any                    `json:"input_schema"`
    CacheControl *anthropicCacheControl `json:"cache_control,omitempty"`
}
```

Helper:

```go
// ephemeralCache returns the canonical 5-min ephemeral marker.
func ephemeralCache() *anthropicCacheControl {
    return &anthropicCacheControl{Type: "ephemeral"}
}
```

### 2.3 Where the marker gets attached

**System prompt** — `toAnthropicRequest` currently emits the system either as a raw JSON string or as an array of content blocks. When **any** system `Message` has `CacheBreakpoint=true`, we force the block-array form and attach `cache_control` to the **last** block of the system content. This is a minor shape change but Anthropic accepts both forms; the block-array form is what the Messages API recommends anyway.

**Regular message content** — `toAnthropicMessage` builds `blocks []anthropicContentBlock` for tool-call / multimodal messages. When `m.CacheBreakpoint` is true, attach `cache_control` to the **last** block before marshaling. For plain-text messages (`m.Content.Text()` path), promote to the block-array form with a single `{type:"text", text:"..."}` block carrying `cache_control`. The empty-text case is irrelevant — a message with no content and no tool calls isn't meaningful.

**Tool array** — the for-range in `toAnthropicRequest` that appends to `ar.Tools` already has per-`Tool` iteration; set `CacheControl = ephemeralCache()` when `t.CacheBreakpoint` is true.

### 2.4 OpenAI translation

No change. `CacheBreakpoint` has `json:"-"` so it never serialises on the canonical request body sent to OpenAI endpoints. We explicitly do **not** translate it — OpenAI's automatic prefix caching handles this transparently.

## 3. taskagent changes

### 3.1 Option

```go
// In vage/agent/taskagent/task.go

type Agent struct {
    // ... existing fields ...
    promptCaching bool
}

const defaultPromptCaching = true

// WithPromptCaching enables or disables emission of cache-boundary hints
// on the system message and the last tool definition. Default true; has
// no on-wire effect for OpenAI-compatible backends (they cache prefixes
// automatically without request-side hints).
func WithPromptCaching(on bool) Option {
    return func(a *Agent) { a.promptCaching = on }
}

func New(cfg agent.Config, opts ...Option) *Agent {
    a := &Agent{
        // ...
        promptCaching: defaultPromptCaching,
    }
    for _, o := range opts { o(a) }
    return a
}
```

### 3.2 Marking call sites

Both `Run` and `RunStream` call `a.buildInitialMessages(...)` → `messages []aimodel.Message` and `a.prepareAITools(...)` → `aiTools []aimodel.Tool`. After those two lines (and before the ReAct loop starts), insert:

```go
if a.promptCaching {
    markPromptCacheBreakpoints(messages, aiTools)
}
```

Helper:

```go
// markPromptCacheBreakpoints attaches cache-breakpoint hints to the two
// stable per-session surfaces: the system message (if any) and the last
// tool definition (if any). Called once per Run/RunStream invocation.
func markPromptCacheBreakpoints(messages []aimodel.Message, tools []aimodel.Tool) {
    // Find the last system message and mark it. There may be zero or
    // more system messages; in practice there is exactly one, but the
    // API allows multiple and we want the breakpoint at the tail.
    for i := len(messages) - 1; i >= 0; i-- {
        if messages[i].Role == aimodel.RoleSystem {
            messages[i].CacheBreakpoint = true
            break
        }
    }
    if len(tools) > 0 {
        tools[len(tools)-1].CacheBreakpoint = true
    }
}
```

Setting a bool on a slice element mutates in place (slices are reference types). The marked `messages` and `tools` flow straight into the `ChatRequest` fed to `aimodel.Client.ChatCompletion(...)`, which branches on protocol and (for Anthropic) picks up the markers in `toAnthropicRequest`.

Note on the streaming path: the same mark step runs once before the ReAct loop. Every iteration within the loop shares the same slice; the marker persists correctly since it's Go-side state and not wiped per-turn.

## 4. vv config + wiring

### 4.1 `configs.AgentsConfig`

```go
type AgentsConfig struct {
    // ... existing fields ...

    // PromptCaching, when set, controls emission of prompt-cache markers
    // on the system prompt and tool definitions. Default (nil) is on;
    // set to a pointer to false to disable. No on-wire effect for OpenAI
    // backends.
    PromptCaching *bool `yaml:"prompt_caching"`
}
```

Default applied in `applyDefaults`: if `cfg.Agents.PromptCaching == nil` → leave nil (the resolver below treats nil as true). Only an explicit `false` disables.

`Load(...)` env override:

```go
if v := os.Getenv("VV_AGENTS_PROMPT_CACHING"); v != "" {
    if b, err := strconv.ParseBool(v); err == nil {
        cfg.Agents.PromptCaching = &b
    } else {
        slog.Warn("vv: invalid VV_AGENTS_PROMPT_CACHING, ignoring", "value", v)
    }
}
```

### 4.2 Resolver

```go
// EffectivePromptCaching resolves the nil-default-on pointer.
func (c *AgentsConfig) EffectivePromptCaching() bool {
    if c == nil || c.PromptCaching == nil {
        return true
    }
    return *c.PromptCaching
}
```

### 4.3 Registry + factories

`registries.FactoryOptions.PromptCaching bool`. `setup.Init` passes `cfg.Agents.EffectivePromptCaching()` into the `FactoryOptions` for the four tool-using agent factories (coder / researcher / reviewer / explorer). Each factory appends:

```go
taskagent.WithPromptCaching(opts.PromptCaching),
```

chat and planner skip it — chat is a no-tool single-iteration path where caching doesn't help; planner has no tools at all.

## 5. Tests

### 5.1 `aimodel`

- `TestToAnthropicRequest_SystemCacheBreakpoint` — system message with `CacheBreakpoint=true`; assert the marshalled `system` field is a JSON array of blocks whose last block carries `"cache_control":{"type":"ephemeral"}`.
- `TestToAnthropicRequest_ToolCacheBreakpoint` — two tools, last with `CacheBreakpoint=true`; assert the second tool's JSON carries `cache_control`.
- `TestToAnthropicRequest_NoCacheBreakpoint` — flag false; assert no `cache_control` in output.
- `TestToAnthropicRequest_BothBreakpoints` — system + tool both flagged; exactly 2 occurrences of the marker.
- `TestOpenAIRequest_CacheBreakpointIgnored` — build a `ChatRequest` with cache flags and marshal (OpenAI's request body is the canonical `ChatRequest` JSON); assert `cache_control` string does NOT appear.

### 5.2 `vage/agent/taskagent`

- `TestMarkPromptCacheBreakpoints_SystemAndTools` — helper marks last system message + last tool; others untouched.
- `TestMarkPromptCacheBreakpoints_NoSystem` — input with no system message: only the last tool gets marked.
- `TestMarkPromptCacheBreakpoints_NoTools` — input with no tools: only the system message gets marked.
- `TestAgent_Run_PromptCachingDefault` — run with no option; assert the outgoing `ChatRequest` has system.CacheBreakpoint=true via a recording ChatCompleter.
- `TestAgent_Run_PromptCachingDisabled` — `WithPromptCaching(false)`; assert no flags set.

### 5.3 `vv/configs`

- `TestAgentsConfig_PromptCachingDefaultOn` — empty YAML → `EffectivePromptCaching() == true`.
- `TestAgentsConfig_PromptCachingExplicitFalse` — YAML `prompt_caching: false` → `EffectivePromptCaching() == false`.
- `TestAgentsConfig_PromptCachingEnvOverride` — env `VV_AGENTS_PROMPT_CACHING=false` overrides YAML `true`.

### 5.4 vv integration test

`vv/integrations/setup_tests/setup_tests/setup_prompt_caching_test.go`:

- Build a coder agent via `setup.New` with a recording mock ChatCompleter.
- Feed a user prompt, have the mock reply `stop`.
- Assert the recorded `ChatRequest.Messages[0].CacheBreakpoint == true` and `Tools[len-1].CacheBreakpoint == true`.
- Re-do with `cfg.Agents.PromptCaching = &false`; assert flags false.

## 6. Files touched

| File | Change |
|------|--------|
| `aimodel/schema.go` | Add `CacheBreakpoint bool` on `Message` and `Tool` (`json:"-"`). |
| `aimodel/anthropic.go` | Add `anthropicCacheControl` struct; extend `anthropicContentBlock` and `anthropicTool` with optional `cache_control`; attach on last block of marked messages + marked tools in `toAnthropicRequest` and `toAnthropicMessage`. |
| `aimodel/anthropic_test.go` or new `aimodel/anthropic_cache_test.go` | 4–5 translation tests. |
| `aimodel/schema_test.go` | One test proving OpenAI JSON body contains no `cache_control`. |
| `vage/agent/taskagent/task.go` | Add `promptCaching` field + `WithPromptCaching` option + `defaultPromptCaching=true` + `markPromptCacheBreakpoints` helper; call the helper in `Run` and `RunStream` after the initial messages and tools are built. |
| `vage/agent/taskagent/task_cache_test.go` (new) | 4–5 helper + end-to-end tests. |
| `vv/configs/config.go` | Add `AgentsConfig.PromptCaching *bool` + env override + `EffectivePromptCaching()`. |
| `vv/configs/prompt_caching_test.go` (new) | 3 default/override tests. |
| `vv/registries/registry.go` | Add `FactoryOptions.PromptCaching bool`. |
| `vv/setup/setup.go` | Thread `cfg.Agents.EffectivePromptCaching()` into `FactoryOptions` for coder / researcher / reviewer / explorer. |
| `vv/agents/{coder,researcher,reviewer,explorer}.go` | Each factory appends `taskagent.WithPromptCaching(opts.PromptCaching)`. |
| `vv/integrations/setup_tests/setup_tests/setup_prompt_caching_test.go` (new) | 2 e2e tests. |

## 7. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Weird OpenAI-compatible provider rejects hidden fields | The field is `json:"-"`; never serialised to OpenAI endpoints. |
| Anthropic rejects >4 cache_control markers | Design emits exactly 2 (system + last tool); well under the cap. |
| Empty tool array path | `len(tools) > 0` guard in the helper; no mark attempted. |
| Cost model mis-attribution | `Usage.CacheReadTokens` is already separated from `PromptTokens`; cost trackers already discount cached tokens. No change needed. |
| Prompt drift breaks cache | Beyond scope; the prompt build path is already deterministic after P1-5. |

## 8. Follow-ups deferred

- 1-hour TTL (`ephemeral` with `ttl: "1h"`). Optional knob if an operator has long-idle sessions.
- Per-turn message breakpoints for conversation history caching.
- Cache-hit rate monitoring / alerting.
- Gemini context-caching API (different model; separate RFC).
- Cache-write surcharge accounting adjustment in cost tracker (currently cached writes are billed at the regular input rate; this is a small overstatement — the Anthropic cache-write is +25%).
