# Technical Design: Token Usage & Cost Tracking

## Architecture Overview

This feature adds token usage extraction and cost estimation across the three-module stack: `aimodel` -> `vage` -> `vv`. The design leverages existing data flows -- token usage is already extracted from API responses and propagated via `LLMCallEndData` events. The primary additions are: (1) a `CacheReadTokens` field on `aimodel.Usage`, (2) a cost tracker in `vv`, (3) a pricing configuration, and (4) surface changes in CLI and HTTP responses.

### Data Flow

```
aimodel.Usage (with CacheReadTokens)
    |
    v
Two emission paths:
  A) largemodel.MetricsMiddleware -> EventLLMCallEnd (sync path)
  B) TaskAgent.runStream() -> EventLLMCallEnd (stream path, emitted directly)
    |
    v
CLI: model.handleStreamEvent accumulates into CostTracker
HTTP: vv/httpapis wraps vage service to add cost data
    |
    v
CLI status bar / HTTP response payload
```

### Key Insight

The existing `MetricsMiddleware` already reads `resp.Usage` from aimodel responses (sync path) and dispatches `EventLLMCallEnd` with token counts. The streaming path in `TaskAgent.runStream()` also emits `EventLLMCallEnd` directly after reading `stream.Usage()`. The CLI already accumulates `totalPromptTokens` and `totalCompletionTokens` from these events. This design extends that pipeline with cache-read tokens and cost calculation, rather than building a parallel mechanism.

**Important:** Both emission paths must be updated to include `CacheReadTokens`.

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

Update `anthropicRecvFunc` to store `CacheReadInputTokens` from `message_start` and propagate it to the final chunk's `Usage`:

```go
var (
    msgID           string
    model           string
    inputTokens     int
    cacheReadTokens int  // NEW: store separately from inputTokens
    // ...
)

// In message_start handler:
inputTokens = ms.Message.Usage.totalInputTokens()
cacheReadTokens = ms.Message.Usage.CacheReadInputTokens

// In message_delta handler, when constructing Usage:
chunk.Usage = &Usage{
    PromptTokens:     inputTokens,
    CompletionTokens: outputTokens,
    TotalTokens:      inputTokens + outputTokens,
    CacheReadTokens:  cacheReadTokens,
}
```

**File:** `aimodel/openai_chat.go`

Extract OpenAI's `usage.prompt_tokens_details.cached_tokens` into `CacheReadTokens`. Add a private struct for the nested details:

```go
// promptTokensDetails captures OpenAI's nested token details.
type promptTokensDetails struct {
    CachedTokens int `json:"cached_tokens"`
}
```

Extend the response parsing to read this field. OpenAI includes `prompt_tokens_details` in `ChatResponse.Usage`; since `Usage` is a flat struct, add an intermediary parse step in the OpenAI sync path that extracts `cached_tokens` and sets `CacheReadTokens`. For OpenAI streaming, the final usage chunk similarly includes this field.

**File:** `aimodel/openai_stream.go`

In the OpenAI stream recv function, the final chunk with `Usage` may include `prompt_tokens_details.cached_tokens`. Extend the chunk parsing to extract and set `CacheReadTokens` on the `Usage` struct.

### 2. vage/schema: Add CacheReadTokens to Event Data Types

**File:** `vage/schema/event.go`

Add `CacheReadTokens` to `LLMCallEndData`, `SubAgentEndData`, and `PhaseEndData`:

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

type PhaseEndData struct {
    Phase            string `json:"phase"`
    Duration         int64  `json:"duration_ms"`
    Summary          string `json:"summary,omitempty"`
    ToolCalls        int    `json:"tool_calls,omitempty"`
    PromptTokens     int    `json:"prompt_tokens,omitempty"`
    CompletionTokens int    `json:"completion_tokens,omitempty"`
    CacheReadTokens  int    `json:"cache_read_tokens,omitempty"`
}
```

**File:** `vage/largemodel/metrics.go`

Pass `CacheReadTokens` from `resp.Usage.CacheReadTokens` into `LLMCallEndData` for both sync and stream paths:

```go
// Sync path (line 73-79):
m.dispatch(ctx, schema.NewEvent(schema.EventLLMCallEnd, "", "", schema.LLMCallEndData{
    Model:            req.Model,
    Duration:         duration,
    PromptTokens:     resp.Usage.PromptTokens,
    CompletionTokens: resp.Usage.CompletionTokens,
    TotalTokens:      resp.Usage.TotalTokens,
    CacheReadTokens:  resp.Usage.CacheReadTokens,
}))

// Stream path (onClose callback, line 107-122):
if usage != nil {
    data.PromptTokens = usage.PromptTokens
    data.CompletionTokens = usage.CompletionTokens
    data.TotalTokens = usage.TotalTokens
    data.CacheReadTokens = usage.CacheReadTokens
}
```

**File:** `vage/agent/taskagent/task.go`

The stream path in `runStream()` emits `EventLLMCallEnd` directly (lines 929-935). Update to include `CacheReadTokens`:

```go
_ = send(schema.NewEvent(schema.EventLLMCallEnd, agentID, rc.sessionID, schema.LLMCallEndData{
    Model:            chatReq.Model,
    PromptTokens:     streamUsage.PromptTokens,
    CompletionTokens: streamUsage.CompletionTokens,
    TotalTokens:      streamUsage.TotalTokens,
    CacheReadTokens:  streamUsage.CacheReadTokens,
    Stream:           true,
}))
```

### 3. vv/traces/costtraces: New Package

**File:** `vv/traces/costtraces/tracker.go`

A simple, concurrency-safe accumulator. One instance per CLI session or HTTP request.

```go
package costtraces

import "sync"

// Pricing defines cost rates for a model (USD per million tokens).
type Pricing struct {
    InputPerMTokens  float64 `json:"input_per_m_tokens" yaml:"input_per_m_tokens"`
    OutputPerMTokens float64 `json:"output_per_m_tokens" yaml:"output_per_m_tokens"`
    CachePerMTokens  float64 `json:"cache_per_m_tokens,omitempty" yaml:"cache_per_m_tokens,omitempty"`
}

// Usage holds accumulated token usage and cost.
type Usage struct {
    InputTokens      int      `json:"input_tokens"`
    OutputTokens     int      `json:"output_tokens"`
    CacheReadTokens  int      `json:"cache_read_tokens"`
    TotalTokens      int      `json:"total_tokens"`
    EstimatedCostUSD *float64 `json:"estimated_cost_usd"`
    CallCount        int      `json:"call_count"`
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
//
// IMPORTANT: promptTokens (from aimodel.Usage.PromptTokens) includes
// cache-read tokens for Anthropic (via totalInputTokens()). The cost
// calculation separates them: non-cached input tokens are charged at
// the input rate, cache-read tokens at the (lower) cache rate.
func (t *Tracker) Add(promptTokens, completionTokens, cacheReadTokens int) {
    t.mu.Lock()
    defer t.mu.Unlock()

    t.usage.InputTokens += promptTokens
    t.usage.OutputTokens += completionTokens
    t.usage.CacheReadTokens += cacheReadTokens
    t.usage.TotalTokens += promptTokens + completionTokens
    t.usage.CallCount++

    if t.pricing != nil {
        // Subtract cache-read tokens from input count to avoid double-charging.
        // PromptTokens includes cache-read tokens (Anthropic's totalInputTokens()),
        // so we charge: (input - cached) * inputRate + cached * cacheRate + output * outputRate.
        nonCachedInput := t.usage.InputTokens - t.usage.CacheReadTokens
        cost := float64(nonCachedInput)/1_000_000*t.pricing.InputPerMTokens +
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

import "sort"

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
// It tries exact match first, then longest-prefix match (e.g., "claude-sonnet-4-20250514"
// matches "claude-sonnet-4", and "gpt-4o-mini" matches "gpt-4o-mini" not "gpt-4o").
func LookupPricing(model string, custom map[string]Pricing) *Pricing {
    // Check custom overrides first (exact then longest prefix).
    if p, ok := exactOrLongestPrefixMatch(model, custom); ok {
        return p
    }
    // Fall back to defaults.
    if p, ok := exactOrLongestPrefixMatch(model, DefaultPricing); ok {
        return p
    }
    return nil
}

// exactOrLongestPrefixMatch tries exact match, then longest prefix match.
func exactOrLongestPrefixMatch(model string, pricing map[string]Pricing) (*Pricing, bool) {
    // Exact match.
    if p, ok := pricing[model]; ok {
        return &p, true
    }

    // Collect and sort keys by length (longest first) for correct prefix matching.
    keys := make([]string, 0, len(pricing))
    for k := range pricing {
        keys = append(keys, k)
    }
    sort.Slice(keys, func(i, j int) bool {
        return len(keys[i]) > len(keys[j])
    })

    for _, k := range keys {
        if len(model) > len(k) && model[:len(k)] == k {
            p := pricing[k]
            return &p, true
        }
    }

    return nil, false
}
```

### 4. vv/configs: Add ModelPricing Configuration

**File:** `vv/configs/config.go`

Add `ModelPricing` to `Config`. The pricing entry struct is defined here as a simple YAML-friendly type, independent of `costtraces.Pricing`:

```go
// ModelPricingEntry defines cost rates for a model (USD per million tokens).
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

In `Load()`, add env var override:

```go
if v := os.Getenv("VV_MODEL_PRICING"); v != "" {
    var mp map[string]ModelPricingEntry
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

**Conversion at wiring (in setup/main):** Convert `configs.ModelPricingEntry` to `costtraces.Pricing` when initializing the tracker. This is a trivial field-by-field copy for a 3-field struct, keeping `configs` and `costtraces` independent.

### 5. vv/cli: Cost Tracker Integration and Status Bar

**File:** `vv/cli/cli.go`

Add `costTracker` to `App`:

```go
type App struct {
    // ... existing fields ...
    costTracker *costtraces.Tracker
}
```

Initialize in `New()`:

```go
pricing := costtraces.LookupPricing(cfg.LLM.Model, convertPricing(cfg.ModelPricing))
a.costTracker = costtraces.New(cfg.LLM.Model, pricing)
```

Update `handleStreamEvent` for `EventLLMCallEnd` to feed the cost tracker:

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

Note: The existing `totalPromptTokens` / `totalCompletionTokens` fields are retained for per-task stats (reset each turn). The cost tracker accumulates across the entire session. These serve different purposes and both are needed.

**Status Bar:**

Add a persistent `statusBarView()` rendered in `View()`. This is distinct from `headerView()`, which is printed once at session start. The status bar appears on every frame between the streaming output and the input area:

```go
func (m *model) statusBarView() string {
    snap := m.app.costTracker.Snapshot()
    status := "idle"
    if m.status == statusProcessing {
        status = "working"
    }
    model := shortModelName(m.app.costTracker.Model())
    cost := formatCost(snap.EstimatedCostUSD)
    tokens := formatCompactTokens(snap.TotalTokens) + " tokens"

    parts := []string{"vv", status, model, cost, tokens}
    return dimStyle.Render(strings.Join(parts, " | "))
}
```

The status bar is rendered in `View()` just above the input area:

```go
func (m *model) View() string {
    var sb strings.Builder
    // ... existing output and status line ...

    // Persistent status bar.
    sb.WriteString(m.statusBarView())
    sb.WriteString("\n")

    // Input area.
    sb.WriteString(m.textarea.View())
    return sb.String()
}
```

Note: The agent name field from the requirement (`{agent}`) is omitted from the status bar because the active agent changes frequently during orchestration and the current agent name is not readily available in the CLI model. If needed later, it can be tracked via `EventSubAgentStart`/`EventSubAgentEnd` events.

**File:** `vv/cli/render.go`

Add helper functions:

```go
// shortModelName derives a short display name from a full model identifier.
// e.g., "claude-sonnet-4-20250514" -> "sonnet-4", "gpt-4o-2024-08-06" -> "gpt-4o"
func shortModelName(model string) string {
    // Strip date suffixes like "-20250514" (8-digit date at end).
    // Strip "claude-" prefix for Anthropic models.
    // Keep as-is for short names.
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

### 6. vv/httpapis: Usage and Cost in HTTP Responses

The usage data in HTTP responses is handled at the **vv layer** (not in vage's service handler), keeping the vage framework pricing-agnostic.

**Sync path (POST /v1/agents/{id}/run):**

The `RunResponse.Usage` field already carries `*aimodel.Usage` with `CacheReadTokens` (after the aimodel changes). For `estimated_cost_usd`, the vv HTTP layer wraps the response at the `httpapis.Serve()` level. Two options:

Option A (simpler): Add a response-wrapping middleware in `httpapis` that intercepts the JSON response for `/v1/agents/*/run` endpoints, reads `usage`, looks up pricing, and injects `estimated_cost_usd`. This avoids modifying the vage service handler.

Option B (cleaner): Use vage's service handler as-is for the raw response. Add a separate vv-specific endpoint or middleware that enriches the response with cost data. Since `httpapis` already builds a custom `http.ServeMux` that wraps `svc.Handler()`, it can intercept specific routes.

**Recommended: Option A.** Implement a `costEnrichMiddleware` in `httpapis` that wraps the service handler:

```go
func costEnrichMiddleware(next http.Handler, pricingLookup func(string) *costtraces.Pricing) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // For run endpoints, capture the response, enrich with cost, re-write.
        // For other endpoints, pass through.
    })
}
```

**Stream path (POST /v1/agents/{id}/stream):**

The SSE stream handler in `vage/service/handler.go` forwards events from `RunStream.Recv()`. To add a usage summary event:

- The vv HTTP layer can wrap the SSE output with a response writer that accumulates token counts from `llm_call_end` events in the JSON stream, and emits a final `usage` SSE event after all events have been forwarded (i.e., after the stream goroutine closes the events channel).
- This `usage` event is emitted **after** `agent_end`, as a post-stream summary.

```
event: agent_end
data: {...}

event: usage
data: {"prompt_tokens":1500,"completion_tokens":800,"total_tokens":2300,"cache_read_tokens":200,"estimated_cost_usd":0.042}
```

**Async path (GET /v1/tasks/{taskID}):**

No schema change needed. The `response.usage` field in the task result already contains `RunResponse.Usage`, which now includes `cache_read_tokens`. Cost enrichment can be applied in the same middleware.

### 7. SubAgentEndData Usage Enhancement

The `CacheReadTokens` field added to `SubAgentEndData` (Section 2) enables per-sub-agent cache tracking. The CLI already uses `SubAgentEndData` fields for sub-agent summary display via `buildStatsLine`. To include cache-read tokens in the display:

**File:** `vv/cli/render.go`

Extend `execStats` with `CacheReadTokens`:

```go
type execStats struct {
    ToolCalls        int
    DurationMs       int64
    PromptTokens     int
    CompletionTokens int
    CacheReadTokens  int
}
```

Update `buildStatsLine` to optionally show cache hits when significant:

```go
if s.CacheReadTokens > 0 {
    parts = append(parts, fmt.Sprintf("cache %s", formatCompactTokens(s.CacheReadTokens)))
}
```

Update all call sites constructing `execStats` to include `CacheReadTokens` from the relevant event data.

---

## Data Models

### aimodel.Usage (modified)

| Field | Type | JSON | Change |
|-------|------|------|--------|
| PromptTokens | int | `prompt_tokens` | existing |
| CompletionTokens | int | `completion_tokens` | existing |
| TotalTokens | int | `total_tokens` | existing |
| CacheReadTokens | int | `cache_read_tokens,omitempty` | **new** |

Note: `PromptTokens` includes cache-read tokens for Anthropic (from `totalInputTokens()`). Cost calculations must account for this to avoid double-charging.

### configs.ModelPricingEntry (new)

| Field | Type | JSON/YAML | Required |
|-------|------|-----------|----------|
| InputPerMTokens | float64 | `input_per_m_tokens` | yes |
| OutputPerMTokens | float64 | `output_per_m_tokens` | yes |
| CachePerMTokens | float64 | `cache_per_m_tokens` | no |

### costtraces.Pricing (new)

Identical fields to `configs.ModelPricingEntry`. Defined separately in `costtraces` to keep packages independent. Converted at wiring time.

### costtraces.Usage (new)

| Field | Type | JSON | Description |
|-------|------|------|-------------|
| InputTokens | int | `input_tokens` | cumulative input tokens (includes cache-read) |
| OutputTokens | int | `output_tokens` | cumulative output tokens |
| CacheReadTokens | int | `cache_read_tokens` | cumulative cache-read tokens |
| TotalTokens | int | `total_tokens` | input + output |
| EstimatedCostUSD | *float64 | `estimated_cost_usd` | nil if no pricing |
| CallCount | int | `call_count` | number of LLM calls |

---

## API Contracts

### Modified: POST /v1/agents/{id}/run Response

The existing `usage` field in `RunResponse` gains `cache_read_tokens`. The `estimated_cost_usd` field is added by the vv HTTP middleware (not part of the vage schema):

```json
{
  "messages": [...],
  "usage": {
    "prompt_tokens": 1500,
    "completion_tokens": 800,
    "total_tokens": 2300,
    "cache_read_tokens": 200
  },
  "estimated_cost_usd": 0.042,
  "duration_ms": 5420
}
```

### Modified: POST /v1/agents/{id}/stream

New SSE event emitted after the stream ends (after `agent_end`):

```
event: usage
data: {"prompt_tokens":1500,"completion_tokens":800,"total_tokens":2300,"cache_read_tokens":200,"estimated_cost_usd":0.042}
```

### Modified: GET /v1/tasks/{taskID}

No schema change needed. The `response.usage` field in the task result already contains `RunResponse.Usage` which now includes `cache_read_tokens`. Cost enrichment applied by middleware.

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

### Task 1: Add CacheReadTokens to aimodel.Usage (both protocols)

**Module:** aimodel
**Files:** `schema.go`, `anthropic.go`, `anthropic_stream.go`, `openai_chat.go`, `openai_stream.go`
**Effort:** Small-Medium

1. Add `CacheReadTokens int` field with `json:"cache_read_tokens,omitempty"` to `Usage`.
2. Update `Usage.Add()` to accumulate `CacheReadTokens`.
3. **Anthropic sync:** In `fromAnthropicResponse()`, set `CacheReadTokens: ar.Usage.CacheReadInputTokens`.
4. **Anthropic stream:** In `anthropicRecvFunc()`, add `cacheReadTokens` variable. Store `ms.Message.Usage.CacheReadInputTokens` from `message_start`. Set it on the Usage in `message_delta`.
5. **OpenAI sync:** Add `promptTokensDetails` struct. After parsing `ChatResponse`, extract `cached_tokens` from `usage.prompt_tokens_details` and set `CacheReadTokens`.
6. **OpenAI stream:** Similarly extract `cached_tokens` from the final usage chunk.
7. Add/update unit tests in `anthropic_test.go`, `anthropic_stream_test.go`, `openai_chat_test.go`, `openai_stream_test.go`, and `schema_test.go`.

### Task 2: Propagate CacheReadTokens through vage

**Module:** vage
**Files:** `schema/event.go`, `largemodel/metrics.go`, `agent/taskagent/task.go`
**Effort:** Small

1. Add `CacheReadTokens int` field to `LLMCallEndData`, `SubAgentEndData`, and `PhaseEndData`.
2. In `MetricsMiddleware.Wrap()`, pass `resp.Usage.CacheReadTokens` to `LLMCallEndData` for both sync and stream (onClose callback) paths.
3. In `TaskAgent.runStream()` (task.go ~line 929), add `CacheReadTokens: streamUsage.CacheReadTokens` to the directly-emitted `LLMCallEndData`.
4. Update unit tests in `largemodel/metrics_test.go`, `schema/event_test.go`, `agent/taskagent/task_test.go`.

### Task 3: Create costtraces package in vv

**Module:** vv
**Files:** new `costtraces/tracker.go`, `costtraces/defaults.go`, `costtraces/tracker_test.go`
**Effort:** Small

1. Implement `Pricing` struct, `Tracker` with `New()`, `Add()`, `Snapshot()`, `Model()`, `PricingAvailable()`.
2. Implement `DefaultPricing` map and `LookupPricing()` with exact and longest-prefix matching.
3. Write unit tests covering: accumulation, cost calculation with cache-read double-counting prevention, nil pricing, prefix matching (including the `gpt-4o` vs `gpt-4o-mini` disambiguation).

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

1. Add `costTracker` to `App`. Initialize in `New()` using `LookupPricing` with converted pricing.
2. Update `handleStreamEvent` for `EventLLMCallEnd` to call `costTracker.Add()`.
3. Add `shortModelName()` and `formatCost()` helpers to `render.go`.
4. Add `statusBarView()` to the model, render it in `View()` above the input area.
5. Extend `execStats` with `CacheReadTokens` and update `buildStatsLine`.
6. Update existing CLI tests.

### Task 6: Add cost enrichment to HTTP responses

**Module:** vv
**Files:** `httpapis/http.go`, new `httpapis/cost.go`
**Effort:** Medium

1. Implement `costEnrichMiddleware` that wraps the vage service handler.
2. For sync run responses: parse response JSON, look up pricing, inject `estimated_cost_usd`, re-write.
3. For streaming responses: wrap response writer to accumulate token counts from `llm_call_end` events, emit final `usage` event after stream ends.
4. Wire middleware in `Serve()`.
5. Add tests.

### Task Order

```
Task 1 (aimodel: CacheReadTokens for both protocols)
  |
  v
Task 2 (vage: propagate CacheReadTokens through events + task agent)
  |
  +---> Task 3 (vv: costtraces package)  --+
  |                                          |
  +---> Task 4 (vv: config)            -----+
  |                                          |
  |                                          v
  |                                    Task 5 (vv: CLI integration)
  |
  +---> Task 6 (vv: HTTP cost enrichment)
```

Tasks 3, 4, and 6 can run in parallel after Task 2. Task 5 depends on Tasks 3 and 4.

---

## Integration Test Plan

### Test 1: End-to-End Token Extraction (aimodel)

- Send a chat completion request to a mock Anthropic server that returns known `cache_read_input_tokens`.
- Verify `ChatResponse.Usage.CacheReadTokens` is correctly populated.
- Send a streaming request and verify `Stream.Usage().CacheReadTokens` after consuming all chunks.
- Repeat for a mock OpenAI server returning `usage.prompt_tokens_details.cached_tokens`.

### Test 2: Metrics Middleware Propagation (vage)

- Create a `MetricsMiddleware` wrapping a mock `ChatCompleter` that returns fixed usage with `CacheReadTokens`.
- Verify that dispatched `LLMCallEndData` events include `CacheReadTokens`.
- Test both sync and stream paths.

### Test 3: TaskAgent Direct Emission (vage)

- Run a `TaskAgent` with streaming enabled and a mock completer that returns `CacheReadTokens` in stream usage.
- Consume the `RunStream` and verify `EventLLMCallEnd` events include `CacheReadTokens`.

### Test 4: Cost Tracker Accumulation (vv)

- Create a `Tracker` with known pricing.
- Call `Add()` multiple times with different token counts including cache-read tokens.
- Verify `Snapshot()` returns correct cumulative totals and cost.
- Verify cache-read tokens are not double-charged (cost of 1000 input tokens with 200 cached should be: 800 * inputRate + 200 * cacheRate, not 1000 * inputRate + 200 * cacheRate).
- Test with nil pricing: `EstimatedCostUSD` should be nil.

### Test 5: Pricing Lookup (vv)

- Verify exact match: `"gpt-4o"` matches `"gpt-4o"`.
- Verify longest prefix match: `"gpt-4o-mini"` matches `"gpt-4o-mini"` (not `"gpt-4o"`).
- Verify prefix match: `"claude-sonnet-4-20250514"` matches `"claude-sonnet-4"`.
- Verify custom overrides take precedence over defaults.
- Verify unknown model returns nil pricing.

### Test 6: CLI Status Bar Rendering (vv)

- Create a CLI model with a cost tracker.
- Simulate `EventLLMCallEnd` events.
- Verify the status bar output contains model name, cost, and token count in the expected format.

### Test 7: HTTP Cost Enrichment (vv)

- Start a test HTTP server with the cost enrichment middleware.
- Send a sync run request and verify `estimated_cost_usd` appears in the response.
- Send a stream request and verify a `usage` event is emitted after the stream ends with correct totals.

### Test 8: Configuration Loading (vv)

- Write a vv.yaml with `model_pricing` section.
- Load config and verify pricing entries are parsed correctly.
- Set `VV_MODEL_PRICING` env var and verify merge behavior (env overrides YAML for same model).
