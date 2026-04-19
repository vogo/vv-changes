# Token & Cost Tracking Display

## Requirement Summary

Add real-time token usage and estimated cost tracking to the vv agent, displayed in the CLI status bar and available via HTTP API. This is a core feature in Claude Code's status line (showing model name, cost, tokens) that vv currently lacks entirely.

## Gap Analysis

### Claude Code

Claude Code tracks and displays in its status bar:
- **Model name** (e.g., Opus 4.6)
- **Cumulative session cost** (e.g., $0.42)
- **Token counts** (input/output tokens consumed)
- Cost is computed per API call based on model pricing and accumulated across the session
- Token budget tracking with nudge messages when approaching budget limits

This information is always visible in the terminal status line, giving users immediate awareness of their API consumption.

### vv Agent (Current State)

- vv has a status bar showing: app name, session status, active agent, processing spinner
- **No token counting** - token usage from LLM responses is not tracked
- **No cost estimation** - no pricing model or cost accumulation
- **No budget awareness** - no way to know how much context/budget has been used

### Impact

Without token/cost tracking, users have no visibility into:
- How much an interaction costs
- How many tokens they have consumed
- Whether they are approaching model context limits
- Relative cost of different operations (e.g., a plan with 5 sub-agents vs. a simple query)

## Proposed Design

### Data Model

```go
// TokenUsage tracks token consumption for a single LLM call
type TokenUsage struct {
    InputTokens  int
    OutputTokens int
    CacheHits    int // cached input tokens (if supported)
}

// SessionCostTracker accumulates usage across a session
type SessionCostTracker struct {
    TotalInputTokens  int
    TotalOutputTokens int
    TotalCacheHits    int
    TotalCost         float64 // estimated USD
    CallCount         int
}
```

### Token Extraction

The aimodel module already receives token usage in API responses:
- OpenAI-compatible: `usage.prompt_tokens`, `usage.completion_tokens`
- Anthropic: `usage.input_tokens`, `usage.output_tokens`, `usage.cache_read_input_tokens`

Add a callback or return value to propagate these counts back to the caller after each LLM call.

### Cost Calculation

Maintain a simple pricing table (configurable, with sensible defaults for common models):

```go
var DefaultPricing = map[string]ModelPricing{
    "claude-opus-4":   {InputPerMToken: 15.0, OutputPerMToken: 75.0, CachePerMToken: 1.875},
    "claude-sonnet-4": {InputPerMToken: 3.0, OutputPerMToken: 15.0, CachePerMToken: 0.375},
    // OpenAI models, etc.
}
```

### Display

**CLI Status Bar** - Extend current status bar to show:
```
vv | processing | coder | $0.42 | 12.3k tokens
```

**HTTP API** - Include in response metadata:
```json
{
  "usage": {
    "input_tokens": 8500,
    "output_tokens": 3800,
    "estimated_cost_usd": 0.42
  }
}
```

### Scope

1. Extract token usage from LLM API responses in aimodel module
2. Accumulate per-session in a cost tracker
3. Display in CLI status bar (model + cost + tokens)
4. Include in HTTP API response metadata
5. Support configurable pricing table (with defaults)

## Implementation Complexity

**Low** - This is primarily:
- Reading fields already present in API responses (token counts)
- Simple arithmetic (multiply by pricing rates)
- Updating an existing status bar component
- Adding a few fields to HTTP response structs

No new tools, no new agent types, no complex orchestration changes. The data is already available in LLM responses; it just needs to be captured, accumulated, and displayed.

## Priority Justification

- **High user value**: Cost visibility is one of the most frequently requested features for LLM-based tools
- **Low implementation effort**: Approximately 1-2 days of work across aimodel + vv modules
- **No dependencies**: Does not require any other feature to be implemented first
- **Foundation for future features**: Token tracking enables future budget limits, context window monitoring, and cost-aware routing
