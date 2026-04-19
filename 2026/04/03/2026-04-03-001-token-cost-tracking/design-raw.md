# Technical Design: Token Usage & Cost Tracking

## Architecture Overview

This feature adds token usage extraction and cost estimation across the three-module stack: `aimodel` -> `vage` -> `vv`. The design leverages existing data flows -- token usage is already extracted from API responses and propagated via `LLMCallEndData` events. The primary additions are: (1) a `CacheReadTokens` field on `aimodel.Usage`, (2) a cost tracker in `vv`, (3) a pricing configuration, and (4) surface changes in CLI and HTTP responses.

### Data Flow

```
aimodel.Usage (with CacheReadTokens)
    |
    v
largemodel.MetricsMiddleware -> schema.LLMCallEndData (with CacheReadTokens)
    |
    v
CLI: model.handleStreamEvent accumulates into CostTracker
HTTP: service handler aggregates from RunResponse.Usage / stream events
    |
    v
CLI status bar / HTTP response payload
```

### Key Insight

The existing `MetricsMiddleware` already reads `resp.Usage` from aimodel responses (both sync and stream) and dispatches `EventLLMCallEnd` with token counts. The CLI already accumulates `totalPromptTokens` and `totalCompletionTokens` from these events. This design extends that pipeline with cache-read tokens and cost calculation, rather than building a parallel mechanism.

---

## Component Design

### 1. aimodel: Add CacheReadTokens to Usage

**File:** `aimodel/schema.go`

Add a `CacheReadTokens` field to the existing `Usage` struct:

```go
type Usage struct {
    PromptTokens     int `json:"prompt_tokens"`
    CompletionTokens int `json:"completion_tokens"`
    TotalTokens      int `json:"total_tokens"`
    CacheReadTokens  int `json:"cache_read_tokens,omitempty"`
}
```

Update `Usage.Add()` to include `CacheReadTokens`.

**File:** `aimodel/anthropic.go`

Update `fromAnthropicResponse` to populate `CacheReadTokens`:

```go
Usage: Usage{
    PromptTokens:     ar.Usage.totalInputTokens(),
    CompletionTokens: ar.Usage.OutputTokens,
    TotalTokens:      ar.Usage.totalInputTokens() + ar.Usage.OutputTokens,
    CacheReadTokens:  ar.Usage.CacheReadInputTokens,
},
```

**File:** `aimodel/anthropic_stream.go`

Update the `message_delta` handler to populate `CacheReadTokens` from `inputTokens` context. The `message_start` event already captures `CacheReadInputTokens`; store it and set it on the final chunk's Usage.

**File:** `aimodel/openai_chat.go` / `aimodel/openai_stream.go`

OpenAI's response includes `usage.prompt_tokens_details.cached_tokens` in some models. For now, leave this as 0 for OpenAI -- it can be added later when the OpenAI response schema is extended. The `CacheReadTokens` field uses `omitempty` so it does not appear for providers that do not support it.

### 2. vage/schema: Add CacheReadTokens to LLMCallEndData

**File:** `vage/schema/event.go`

```go
type LLMCallEndData struct {
    Model            string `json:"model"`
    Duration         int64  `json:"duration_ms"`
    PromptTokens     int    `json:"prompt_tokens"`
    CompletionTokens int    `json:"completion_tokens"`
    TotalTokens      int    `json:"total_tokens"`
    CacheReadTokens  int    `json:"cache_read_tokens,omitempty"`
    Stream           bool   `json:"stream"`
}
```

**File:** `vage/largemodel/metrics.go`

Pass `CacheReadTokens` from `resp.Usage.CacheReadTokens` into `LLMCallEndData` for both sync and stream paths.

### 3. vv/traces/costtraces: New Package

**File:** `vv/traces/costtraces/tracker.go`

A simple, concurrency-safe accumulator. One instance per CLI session or HTTP request.

```go
package costtraces

import "sync"

// Usage holds accumulated token usage.
type Usage struct {
    InputTokens      int     `json:"input_tokens"`
    OutputTokens     int     `json:"output_tokens"`
    CacheReadTokens  int     `json:"cache_read_tokens"`
    TotalTokens      int     `json:"total_tokens"`
    EstimatedCostUSD *float64 `json:"estimated_cost_usd"`
    CallCount        int     `json:"call_count"`
}

// Pricing defines cost rates for a model (USD per million tokens).
type Pricing struct {
    InputPerMTokens  float64 `json:"input_per_m_tokens" yaml:"input_per_m_tokens"`
    OutputPerMTokens float64 `json:"output_per_m_tokens" yaml:"output_per_m_tokens"`
    CachePerMTokens  float64 `json:"cache_per_m_tokens,omitempty" yaml:"cache_per_m_tokens,omitempty"`
}

// Tracker accumulates token usage and estimates cost.
type Tracker struct {
    mu      sync.Mutex
    usage   Usage
    model   string
    pricing *Pricing // nil if no pricing available
}

// New creates a Tracker for the given model with optional pricing.
func New(model string, pricing *Pricing) *Tracker {
    return &Tracker{model: model, pricing: pricing}
}

// Add records tokens from a single LLM call.
func (t *Tracker) Add(promptTokens, completionTokens, cacheReadTokens int) {
    t.mu.Lock()
    defer t.mu.Unlock()

    t.usage.InputTokens += promptTokens
    t.usage.OutputTokens += completionTokens
    t.usage.CacheReadTokens += cacheReadTokens
    t.usage.TotalTokens += promptTokens + completionTokens
    t.usage.CallCount++

    if t.pricing != nil {
        cost := float64(t.usage.InputTokens)/1_000_000*t.pricing.InputPerMTokens +
            float64(t.usage.OutputTokens)/1_000_000*t.pricing.OutputPerMTokens +
            float64(t.usage.CacheReadTokens)/1_000_000*t.pricing.CachePerMTokens
        t.usage.EstimatedCostUSD = &cost
    }
}

// Snapshot returns a copy of the current usage.
func (t *Tracker) Snapshot() Usage {
    t.mu.Lock()
    defer t.mu.Unlock()
    u := t.usage
    if t.usage.EstimatedCostUSD != nil {
        v := *t.usage.EstimatedCostUSD
        u.EstimatedCostUSD = &v
    }
    return u
}

// Model returns the model name.
func (t *Tracker) Model() string { return t.model }

// PricingAvailable returns whether pricing is configured.
func (t *Tracker) PricingAvailable() bool { return t.pricing != nil }
```

**File:** `vv/traces/costtraces/defaults.go`

Default pricing table and lookup function:

```go
package costtraces

// DefaultPricing maps model name patterns to pricing.
var DefaultPricing = map[string]Pricing{
    "claude-opus-4":   {InputPerMTokens: 15.0, OutputPerMTokens: 75.0, CachePerMTokens: 1.5},
    "claude-sonnet-4": {InputPerMTokens: 3.0, OutputPerMTokens: 15.0, CachePerMTokens: 0.3},
    "gpt-4o":          {InputPerMTokens: 2.5, OutputPerMTokens: 10.0},
    "gpt-4o-mini":     {InputPerMTokens: 0.15, OutputPerMTokens: 0.6},
    "gpt-4.1":         {InputPerMTokens: 2.0, OutputPerMTokens: 8.0},
    "gpt-4.1-mini":    {InputPerMTokens: 0.4, OutputPerMTokens: 1.6},
}

// LookupPricing finds pricing for a model name.
// It tries exact match first, then prefix match (e.g., "claude-sonnet-4-20250514"
// matches "claude-sonnet-4").
func LookupPricing(model string, custom map[string]Pricing) *Pricing {
    // Check custom overrides first (exact then prefix).
    if p, ok := exactOrPrefixMatch(model, custom); ok {
        return p
    }
    // Fall back to defaults.
    if p, ok := exactOrPrefixMatch(model, DefaultPricing); ok {
        return p
    }
    return nil
}
```

### 4. vv/configs: Add ModelPricing Configuration

**File:** `vv/configs/config.go`

Add to `Config`:

```go
type Config struct {
    // ... existing fields ...
    ModelPricing map[string]costtraces.Pricing `yaml:"model_pricing,omitempty"`
}
```

In `Load()`, add env var override:

```go
if v := os.Getenv("VV_MODEL_PRICING"); v != "" {
    var mp map[string]costtraces.Pricing
    if err := json.Unmarshal([]byte(v), &mp); err == nil {
        if cfg.ModelPricing == nil {
            cfg.ModelPricing = mp
        } else {
            for k, v := range mp {
                cfg.ModelPricing[k] = v
            }
        }
    }
}
```

Note: To avoid a circular import (`configs` -> `costtraces`), the `Pricing` struct will be defined in `costtraces` and imported by `configs`. Alternatively, `Pricing` can be defined directly in `configs` and `costtraces` imports it. Given the dependency flow (`vv/configs` is a leaf package used by `vv/cli` and `vv/traces/costtraces`), the cleanest approach is to define `Pricing` in `costtraces` and import it from `configs`. Since both are in the same module (`vv`), there is no cross-module issue. However, if `configs` is loaded before `costtraces`, a simpler approach is to define the pricing struct inline in `configs` and have `costtraces` accept it as a parameter. We will use the latter to keep `configs` as a leaf:

```go
// In configs/config.go
type ModelPricingEntry struct {
    InputPerMTokens  float64 `json:"input_per_m_tokens" yaml:"input_per_m_tokens"`
    OutputPerMTokens float64 `json:"output_per_m_tokens" yaml:"output_per_m_tokens"`
    CachePerMTokens  float64 `json:"cache_per_m_tokens,omitempty" yaml:"cache_per_m_tokens,omitempty"`
}

type Config struct {
    // ... existing fields ...
    ModelPricing map[string]ModelPricingEntry `yaml:"model_pricing,omitempty"`
}
```

Then `costtraces.Pricing` is identical and converted at setup time. Or simpler: `costtraces` accepts `configs.ModelPricingEntry` directly. Given this is a small struct, the pragmatic choice is to define it once in `configs` and have `costtraces` use it directly via import.

**Final decision:** Define `ModelPricingEntry` in `configs`. The `costtraces` package imports `configs` for the type. The `Tracker` accepts `*configs.ModelPricingEntry`.

### 5. vv/cli: Cost Tracker Integration and Status Bar

**File:** `vv/cli/cli.go`

Add `costTracker` to `App`:

```go
type App struct {
    // ... existing fields ...
    costTracker *costtraces.Tracker
}
```

Initialize in `New()` or `Run()`:

```go
pricing := costtraces.LookupPricing(cfg.LLM.Model, cfg.ModelPricing)
a.costTracker = costtraces.New(cfg.LLM.Model, pricing)
```

Update `handleStreamEvent` for `EventLLMCallEnd`:

```go
case schema.EventLLMCallEnd:
    if data, ok := event.Data.(schema.LLMCallEndData); ok {
        m.totalPromptTokens += data.PromptTokens
        m.totalCompletionTokens += data.CompletionTokens
        m.subAgentPromptTokens += data.PromptTokens
        m.subAgentCompletionTokens += data.CompletionTokens
        // Update cost tracker.
        m.app.costTracker.Add(data.PromptTokens, data.CompletionTokens, data.CacheReadTokens)
    }
```

**Status Bar Changes:**

Update `headerView()` to include model short name, cost, and tokens:

```go
func (m *model) headerView() string {
    // ... existing dir logic ...
    shortModel := shortModelName(m.app.cfg.LLM.Model)
    providerModel := fmt.Sprintf("vv | %s | %s", m.app.cfg.LLM.Provider, shortModel)
    // ... render ...
}
```

Add a `statusBarView()` method rendered in `View()`:

```go
func (m *model) statusBarView() string {
    snap := m.app.costTracker.Snapshot()
    status := "idle"
    if m.status == statusProcessing {
        status = "working"
    }
    agent := "chat" // or derive from current agent
    model := shortModelName(m.app.costTracker.Model())
    cost := formatCost(snap.EstimatedCostUSD)
    tokens := formatCompactTokens(snap.TotalTokens) + " tokens"

    parts := []string{"vv", status, agent, model, cost, tokens}
    return dimStyle.Render(strings.Join(parts, " | "))
}
```

The status bar is rendered in the `View()` method between the streaming output and the spinner/textarea.

**File:** `vv/cli/render.go`

Add helper functions:

```go
// shortModelName derives a short display name from a full model identifier.
// e.g., "claude-sonnet-4-20250514" -> "sonnet-4", "gpt-4o-2024-08-06" -> "gpt-4o"
func shortModelName(model string) string {
    // Strip date suffixes like "-20250514"
    // Strip "claude-" prefix for Anthropic models
    // Keep as-is for short names
}

// formatCost formats a USD cost for display.
func formatCost(cost *float64) string {
    if cost == nil {
        return "N/A"
    }
    if *cost < 1.0 {
        return fmt.Sprintf("$%.3f", *cost)
    }
    return fmt.Sprintf("$%.2f", *cost)
}
```

### 6. vv/httpapis and vage/service: Usage in HTTP Responses

The existing `RunResponse` already has a `Usage *aimodel.Usage` field. The `handleRun` endpoint returns `RunResponse` directly, so once the agent properly populates `Usage`, it flows through automatically.

For the streaming endpoint, add a `usage` event emission. After the stream ends and before closing, emit a final usage summary. This requires the stream consumer in `handleStream` to accumulate usage from `EventLLMCallEnd` events and emit a synthetic `usage` event.

**File:** `vage/service/handler.go`

Modify `handleStream` to accumulate usage:

```go
// In the event consumption goroutine:
var totalUsage aimodel.Usage
// When processing events:
if e.Type == schema.EventLLMCallEnd {
    if data, ok := e.Data.(schema.LLMCallEndData); ok {
        totalUsage.PromptTokens += data.PromptTokens
        totalUsage.CompletionTokens += data.CompletionTokens
        totalUsage.TotalTokens += data.TotalTokens
        totalUsage.CacheReadTokens += data.CacheReadTokens
    }
}
// After stream ends (before closing events channel):
usageData, _ := json.Marshal(totalUsage)
events <- sseItem{event: "usage", data: usageData}
```

For async tasks, usage comes from `RunResponse.Usage` which is already stored in `Task.Response`.

### 7. SubAgentEndData Usage Enhancement

**File:** `vage/schema/event.go`

Add `CacheReadTokens` to `SubAgentEndData`:

```go
type SubAgentEndData struct {
    AgentName        string `json:"agent_name"`
    StepID           string `json:"step_id,omitempty"`
    Duration         int64  `json:"duration_ms"`
    ToolCalls        int    `json:"tool_calls"`
    TokensUsed       int    `json:"tokens_used"`
    PromptTokens     int    `json:"prompt_tokens,omitempty"`
    CompletionTokens int    `json:"completion_tokens,omitempty"`
    CacheReadTokens  int    `json:"cache_read_tokens,omitempty"`
}
```

The CLI already uses these fields for sub-agent summary display. No additional CLI changes needed for sub-agent stats beyond what is already rendered by `buildStatsLine`.

---

## Data Models

### aimodel.Usage (modified)

| Field | Type | JSON | Change |
|-------|------|------|--------|
| PromptTokens | int | `prompt_tokens` | existing |
| CompletionTokens | int | `completion_tokens` | existing |
| TotalTokens | int | `total_tokens` | existing |
| CacheReadTokens | int | `cache_read_tokens,omitempty` | **new** |

### configs.ModelPricingEntry (new)

| Field | Type | JSON/YAML | Required |
|-------|------|-----------|----------|
| InputPerMTokens | float64 | `input_per_m_tokens` | yes |
| OutputPerMTokens | float64 | `output_per_m_tokens` | yes |
| CachePerMTokens | float64 | `cache_per_m_tokens` | no |

### costtraces.Usage (new)

| Field | Type | JSON | Description |
|-------|------|------|-------------|
| InputTokens | int | `input_tokens` | cumulative input tokens |
| OutputTokens | int | `output_tokens` | cumulative output tokens |
| CacheReadTokens | int | `cache_read_tokens` | cumulative cache-read tokens |
| TotalTokens | int | `total_tokens` | input + output |
| EstimatedCostUSD | *float64 | `estimated_cost_usd` | nil if no pricing |
| CallCount | int | `call_count` | number of LLM calls |

---

## API Contracts

### Modified: POST /v1/agents/{id}/run Response

The existing `usage` field in `RunResponse` gains `cache_read_tokens`. A new `estimated_cost_usd` can be added at the HTTP handler level by wrapping the response (not modifying `schema.RunResponse`).

```json
{
  "messages": [...],
  "usage": {
    "prompt_tokens": 1500,
    "completion_tokens": 800,
    "total_tokens": 2300,
    "cache_read_tokens": 200
  },
  "duration_ms": 5420
}
```

Cost estimation in HTTP responses is handled by the vv HTTP layer wrapping the response with pricing lookup, keeping vage module pricing-agnostic.

### Modified: POST /v1/agents/{id}/stream

New SSE event emitted before stream close:

```
event: usage
data: {"prompt_tokens":1500,"completion_tokens":800,"total_tokens":2300,"cache_read_tokens":200}
```

### Modified: GET /v1/tasks/{taskID}

No schema change needed. The `response.usage` field in the task result already contains `RunResponse.Usage` which now includes `cache_read_tokens`.

---

## Configuration

### vv.yaml additions

```yaml
model_pricing:
  claude-opus-4:
    input_per_m_tokens: 15.0
    output_per_m_tokens: 75.0
    cache_per_m_tokens: 1.5
  claude-sonnet-4:
    input_per_m_tokens: 3.0
    output_per_m_tokens: 15.0
    cache_per_m_tokens: 0.3
  gpt-4o:
    input_per_m_tokens: 2.5
    output_per_m_tokens: 10.0
```

### Environment variable

`VV_MODEL_PRICING` -- JSON-encoded map, merged on top of YAML config:

```bash
export VV_MODEL_PRICING='{"my-model":{"input_per_m_tokens":1.0,"output_per_m_tokens":5.0}}'
```

---

## Implementation Plan

### Task 1: Add CacheReadTokens to aimodel.Usage

**Module:** aimodel
**Files:** `schema.go`, `anthropic.go`, `anthropic_stream.go`
**Effort:** Small

1. Add `CacheReadTokens int` field with `json:"cache_read_tokens,omitempty"` to `Usage`.
2. Update `Usage.Add()` to accumulate `CacheReadTokens`.
3. In `fromAnthropicResponse()`, set `CacheReadTokens: ar.Usage.CacheReadInputTokens`.
4. In `anthropicRecvFunc()`, capture `CacheReadInputTokens` from `message_start` and set it on the final chunk's `Usage` alongside `PromptTokens`.
5. Add/update unit tests in `anthropic_test.go` and `anthropic_stream_test.go`.

### Task 2: Propagate CacheReadTokens through vage

**Module:** vage
**Files:** `schema/event.go`, `largemodel/metrics.go`
**Effort:** Small

1. Add `CacheReadTokens int` field to `LLMCallEndData` and `SubAgentEndData`.
2. In `MetricsMiddleware.Wrap()`, pass `resp.Usage.CacheReadTokens` to `LLMCallEndData` for both sync and stream handlers.
3. Update unit tests in `largemodel/metrics_test.go` and `schema/event_test.go`.

### Task 3: Create costtraces package in vv

**Module:** vv
**Files:** new `costtraces/tracker.go`, `costtraces/defaults.go`, `costtraces/tracker_test.go`
**Effort:** Small

1. Implement `Tracker` with `New()`, `Add()`, `Snapshot()`, `Model()`, `PricingAvailable()`.
2. Implement `DefaultPricing` map and `LookupPricing()` with exact and prefix matching.
3. Write unit tests covering: accumulation, cost calculation, nil pricing, prefix matching.

### Task 4: Add ModelPricing to vv config

**Module:** vv
**Files:** `configs/config.go`
**Effort:** Small

1. Add `ModelPricingEntry` struct and `ModelPricing` field to `Config`.
2. Add `VV_MODEL_PRICING` env var parsing in `Load()`.
3. Update config tests.

### Task 5: Integrate cost tracker into CLI

**Module:** vv
**Files:** `cli/cli.go`, `cli/render.go`
**Effort:** Medium

1. Add `costTracker` to `App`. Initialize in `New()` using `LookupPricing`.
2. Update `handleStreamEvent` for `EventLLMCallEnd` to call `costTracker.Add()`.
3. Add `shortModelName()` and `formatCost()` helpers to `render.go`.
4. Add `statusBarView()` to the model, render it in `View()`.
5. Update `headerView()` to include the short model name.
6. Update existing CLI tests.

### Task 6: Add usage event to HTTP streaming

**Module:** vage
**Files:** `service/handler.go`
**Effort:** Small

1. In `handleStream`, accumulate usage from `EventLLMCallEnd` events in the consumer goroutine.
2. After the stream consumer loop ends, emit a `usage` SSE event with the accumulated totals before closing the events channel.
3. Update service tests.

### Task Order

Tasks 1 -> 2 -> 3 (depends on 1 only for type awareness) -> 4 -> 5 (depends on 3, 4) -> 6 (depends on 2).

Tasks 3 and 4 can run in parallel after Task 2.

```
Task 1 (aimodel: CacheReadTokens)
  |
  v
Task 2 (vage: propagate CacheReadTokens)
  |
  +---> Task 3 (vv: costtraces package)  --+
  |                                          |
  +---> Task 4 (vv: config)            -----+
  |                                          |
  +---> Task 6 (vage: HTTP usage event)      v
                                       Task 5 (vv: CLI integration)
```

---

## Integration Test Plan

### Test 1: End-to-End Token Extraction (aimodel)

- Send a chat completion request to a mock Anthropic server that returns known `cache_read_input_tokens`.
- Verify `ChatResponse.Usage.CacheReadTokens` is correctly populated.
- Send a streaming request and verify `Stream.Usage().CacheReadTokens` after consuming all chunks.

### Test 2: Metrics Middleware Propagation (vage)

- Create a `MetricsMiddleware` wrapping a mock `ChatCompleter` that returns fixed usage.
- Verify that dispatched `LLMCallEndData` events include `CacheReadTokens`.
- Test both sync and stream paths.

### Test 3: Cost Tracker Accumulation (vv)

- Create a `Tracker` with known pricing.
- Call `Add()` multiple times with different token counts.
- Verify `Snapshot()` returns correct cumulative totals and cost.
- Test with nil pricing: `EstimatedCostUSD` should be nil.

### Test 4: CLI Status Bar Rendering (vv)

- Create a CLI model with a cost tracker.
- Simulate `EventLLMCallEnd` events.
- Verify the status bar output contains model name, cost, and token count in the expected format.

### Test 5: HTTP Streaming Usage Event (vage)

- Start a test HTTP server with a mock streaming agent.
- Send a stream request and consume SSE events.
- Verify a `usage` event is emitted with correct aggregated token counts.
- Verify it appears after the last `text_delta` and before the connection closes.

### Test 6: Configuration Loading (vv)

- Write a vv.yaml with `model_pricing` section.
- Load config and verify pricing entries are parsed correctly.
- Set `VV_MODEL_PRICING` env var and verify merge behavior (env overrides YAML for same model).

### Test 7: Pricing Lookup (vv)

- Verify exact match: `"gpt-4o"` matches `"gpt-4o"`.
- Verify prefix match: `"claude-sonnet-4-20250514"` matches `"claude-sonnet-4"`.
- Verify custom overrides take precedence over defaults.
- Verify unknown model returns nil pricing.
