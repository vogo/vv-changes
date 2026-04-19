# Change Record: Token Usage & Cost Tracking

## Background & Objectives

Users of vv currently have no visibility into how many tokens are consumed or how much their interactions cost when using the CLI or HTTP API. Competing tools like Claude Code display model name, cumulative session cost, and token counts in their status bar, giving users immediate awareness of API consumption.

The objective is to add real-time token usage extraction, cost estimation, and session-level accumulation to vv, surfacing this data in the CLI status bar and HTTP API responses. This provides cost transparency, enables informed usage decisions, and lays the foundation for future budget-limit and cost-aware routing features.

## User Stories

### US-1: View Token Usage and Cost in CLI Status Bar

**As a** User (CLI mode),
**I want to** see the current model name, cumulative session cost, and total token count in the CLI status bar,
**So that** I can monitor my API consumption in real time during an interactive session.

**Acceptance Criteria:**
1. The CLI status bar displays the configured model name (short form, e.g., "opus-4" derived from the full model identifier).
2. The CLI status bar displays the cumulative estimated cost in USD, formatted as "$X.XX" (or "$X.XXX" for sub-cent precision when total is below $1.00).
3. The CLI status bar displays the cumulative total token count (input + output), formatted with "k" suffix for thousands (e.g., "12.3k tokens").
4. All three values update after each LLM call completes (including calls made by sub-agents during orchestration).
5. On session start, cost and token values display as "$0.00" and "0 tokens".
6. The status bar format is: `vv | {status} | {agent} | {model} | {cost} | {tokens}`.

### US-2: View Token Usage and Cost in HTTP API Responses

**As a** User (HTTP API mode),
**I want to** receive token usage and estimated cost in every agent execution response,
**So that** I can track consumption programmatically and integrate cost monitoring into my workflows.

**Acceptance Criteria:**
1. The synchronous run endpoint (`POST /v1/agents/{id}/run`) response includes a `usage` object with `input_tokens`, `output_tokens`, `cache_read_tokens`, `total_tokens`, and `estimated_cost_usd` fields.
2. The streaming endpoint (`POST /v1/agents/{id}/stream`) emits a final `usage` event (before the `agent_end` event) containing the same usage fields.
3. The async task result (`GET /v1/tasks/{taskID}`) includes the same `usage` object when the task is completed.
4. Token counts aggregate across all LLM calls within the request, including orchestrator and sub-agent calls.
5. `estimated_cost_usd` is a floating-point number representing USD.

### US-3: Configure Model Pricing

**As an** Operator,
**I want to** configure pricing rates for the models I use,
**So that** cost estimates reflect my actual pricing (which may differ from defaults due to volume discounts or custom agreements).

**Acceptance Criteria:**
1. A `model_pricing` section in the configuration file allows specifying per-model pricing with `input_per_m_tokens`, `output_per_m_tokens`, and `cache_per_m_tokens` rates (USD per million tokens).
2. Default pricing is provided for common models: Claude Opus 4, Claude Sonnet 4, GPT-4o, GPT-4o-mini.
3. If a model has no configured pricing, cost displays as "N/A" in the CLI and `estimated_cost_usd` is `null` in the HTTP response (token counts are still reported).
4. Pricing configuration can be overridden via `VV_MODEL_PRICING` environment variable (JSON-encoded map).

### US-4: View Per-Sub-Agent Token Usage in Orchestrated Tasks

**As a** User,
**I want to** see token usage broken down by sub-agent in orchestrated (multi-step) tasks,
**So that** I can understand which parts of a complex task consume the most tokens and cost.

**Acceptance Criteria:**
1. In CLI mode, the existing `sub_agent_end` display already shows token usage (e.g., "Done (12 tool uses . 45.2k tokens . 2m 30s)"). This continues to work and now reflects actual extracted token counts instead of estimates.
2. In HTTP streaming mode, the `sub_agent_end` event includes a `usage` object with per-sub-agent token counts and cost.
3. In HTTP synchronous mode, the response includes a `step_usage` array (for plan-mode orchestration) with per-step token breakdowns.

## Scope Boundaries

### In Scope

- Extract token usage (input, output, cache read) from LLM API responses in the aimodel module
- Propagate token usage from aimodel through vage's largemodel middleware to the agent layer
- Accumulate token usage per session in a cost tracker
- Calculate estimated cost using a configurable pricing table
- Display model name, cumulative cost, and cumulative tokens in the CLI status bar
- Include token usage and cost in HTTP API responses (sync, stream, async)
- Include per-sub-agent usage in orchestration results
- Add `model_pricing` configuration with sensible defaults
- Update the existing `sub_agent_end` display to use actual token counts

### Out of Scope

- Token budget enforcement or spending limits (future enhancement)
- Context window utilization display (percentage of context used) -- related but separate concern
- Historical cost tracking across sessions (persistent cost logs)
- Per-tool token usage breakdown (tokens consumed by tool call overhead)
- Real-time cost alerts or notifications when thresholds are exceeded
- Currency conversion (USD only)
- Billing integration or payment processing

## Involved Roles

| Role | Involvement |
|------|-------------|
| User (CLI) | Views token usage and cost in the CLI status bar; sees per-sub-agent usage in orchestration summaries |
| User (HTTP API) | Receives token usage and cost in API response payloads |
| Operator | Configures model pricing rates in the configuration file or via environment variable |
| Orchestrator Agent | Aggregates token usage from sub-agents and its own LLM calls |
| All Sub-Agents (Coder, Researcher, Reviewer, Chat) | Their LLM calls contribute to token usage tracking |

## Involved Models & State Changes

### New Model: Session Cost Tracker

Accumulates token usage and cost across all LLM calls within a session (CLI) or request (HTTP).

| Attribute | Description | Type | Required |
|-----------|-------------|------|----------|
| total_input_tokens | Cumulative input tokens across all LLM calls | number | Yes |
| total_output_tokens | Cumulative output tokens across all LLM calls | number | Yes |
| total_cache_read_tokens | Cumulative cache-read input tokens | number | Yes |
| total_cost_usd | Cumulative estimated cost in USD | number (float) | Yes |
| call_count | Number of LLM calls made | number | Yes |
| model_name | Name of the primary model being used | text | Yes |
| pricing_available | Whether pricing is configured for the model | boolean | Yes |

**Lifecycle:** Created when a CLI session starts or an HTTP request begins. Updated after every LLM call. Destroyed when the session ends or the HTTP response is sent.

### New Model: Token Usage (per LLM call)

Captures token usage from a single LLM API response.

| Attribute | Description | Type | Required |
|-----------|-------------|------|----------|
| input_tokens | Number of input/prompt tokens consumed | number | Yes |
| output_tokens | Number of output/completion tokens generated | number | Yes |
| cache_read_tokens | Number of cached input tokens (0 if unsupported) | number | Yes |

### New Model: Model Pricing

Defines the cost rates for a specific model.

| Attribute | Description | Type | Required |
|-----------|-------------|------|----------|
| model_pattern | Model identifier or pattern to match | text | Yes |
| input_per_m_tokens | Cost in USD per million input tokens | number (float) | Yes |
| output_per_m_tokens | Cost in USD per million output tokens | number (float) | Yes |
| cache_per_m_tokens | Cost in USD per million cached input tokens | number (float) | No |

### Modified Model: Configuration

Add the following attribute:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| model_pricing | Per-model pricing rates | map of model name to Model Pricing | No | Override via `VV_MODEL_PRICING` env var; defaults provided for common models |

### Modified Model: CLI Session

Add the following attribute:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| cost_tracker | Session-level cost tracker | reference to Session Cost Tracker | Yes | Created at session startup; updated after each LLM call |

### Modified Model: Async Task

Add the following attribute:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| usage | Aggregated token usage and cost for the task | reference to Token Usage + cost | No | Populated on completion |

## Involved Business Processes

### Modified Process: CLI Message Processing

**Changes:**
- **Step 4 (Stream Agent Output)**: After processing LLM-related events, update the session's `cost_tracker` with extracted token usage. The status bar re-renders automatically to reflect new totals.
- **Step 4 (sub_agent_end event)**: The token usage in the summary now comes from actual extracted counts rather than estimates.
- **New business rule CLIMSG-15: Cost Tracker Update**: After each LLM call completes (detected via the agent event stream), the session's cost tracker is updated with the extracted token usage, the cost is calculated using the pricing table, and the status bar is re-rendered.

### Modified Process: Synchronous Request

**Changes:**
- After agent execution completes, populate the `usage` field in the response with aggregated token usage and cost from all LLM calls during the request.

### Modified Process: Streaming Request

**Changes:**
- Emit a `usage` event containing aggregated token usage and cost before the `agent_end` event.

### Modified Process: Async Request

**Changes:**
- When the task completes, store aggregated token usage and cost in the task result.

### Modified Process: Orchestration

**Changes:**
- The orchestrator already aggregates token usage (rule ORCH-10). This now uses actual extracted token counts from the aimodel module instead of estimates.
- Per-sub-agent usage is captured and included in sub_agent_end events and step_usage arrays.

### Modified Process: Application Startup

**Changes:**
- Load `model_pricing` from configuration file.
- Merge default pricing with user-configured pricing (user config overrides defaults).
- Parse `VV_MODEL_PRICING` environment variable if present and merge.

## Involved Applications & Pages

### Modified Page: CLI Main Conversation View

**Section 1: Status Bar changes:**
- Add model name display (short form derived from `llm_model` config, e.g., "claude-sonnet-4-20250514" -> "sonnet-4")
- Add cumulative cost display (e.g., "$0.42", or "N/A" if pricing unavailable)
- Add cumulative token count display (e.g., "12.3k tokens")
- Updated format: `vv | {status} | {agent} | {model} | {cost} | {tokens}`
- All cost/token values start at zero and update after each LLM call

### Modified Endpoint: Run Agent (POST /v1/agents/{id}/run)

**Response changes:**
- The existing `usage` field is enhanced with additional fields:

| Field | Type | Description |
|-------|------|-------------|
| usage.input_tokens | number | Total input tokens across all LLM calls |
| usage.output_tokens | number | Total output tokens across all LLM calls |
| usage.cache_read_tokens | number | Total cached input tokens |
| usage.total_tokens | number | Sum of input + output tokens |
| usage.estimated_cost_usd | number or null | Estimated cost in USD; null if pricing unavailable |
| usage.call_count | number | Number of LLM calls made |
| step_usage | array | Per-step usage breakdown (plan mode only) |

### Modified Endpoint: Stream Agent (POST /v1/agents/{id}/stream)

**New event type:**

| Event Type | Data | Description |
|------------|------|-------------|
| usage | UsageData | Aggregated token usage and cost; emitted before agent_end |

**Modified event:**

| Event Type | Change |
|------------|--------|
| sub_agent_end | AgentEndData now includes a `usage` sub-object with per-sub-agent token counts and cost |

### Modified Endpoint: Get Async Task (GET /v1/tasks/{taskID})

**Response changes when status is "Completed":**
- The `result` object includes a `usage` field with the same structure as the synchronous run endpoint.

## Cross-Module Impact

### aimodel module
- Extract token usage from API responses (OpenAI: `usage.prompt_tokens`, `usage.completion_tokens`; Anthropic: `usage.input_tokens`, `usage.output_tokens`, `usage.cache_read_input_tokens`)
- Return Token Usage as part of the LLM call result or via a callback mechanism
- This is the lowest-level change and all downstream modules depend on it

### vage module
- The `largemodel` middleware chain must propagate token usage from aimodel responses up to the agent layer
- Agent execution results must include aggregated token usage
- The orchestrate package must aggregate usage across sub-agent executions

### vv module
- Consumes token usage from vage agent results
- Manages the Session Cost Tracker
- Renders usage in CLI status bar
- Includes usage in HTTP response payloads
- Loads and provides model pricing configuration
