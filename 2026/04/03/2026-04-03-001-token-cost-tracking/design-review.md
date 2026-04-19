# Design Review: Token Usage & Cost Tracking

## Overall Assessment

The design is well-structured, correctly identifies the existing data flow pipeline, and proposes minimal, targeted changes. The "Key Insight" about extending the existing `MetricsMiddleware` -> `EventLLMCallEnd` pipeline rather than building a parallel mechanism is sound. However, several assumptions are inaccurate or incomplete based on code inspection, and there are structural issues that need correction.

## Issues Found

### CRITICAL: Cost calculation bug in Tracker.Add()

The design recalculates cost from cumulative totals on every `Add()` call. While the result is mathematically correct, using `float64(t.usage.InputTokens)/1_000_000*t.pricing.InputPerMTokens` for the running total means the cost calculation is correct. However, for cache-read tokens, the design does not account for the **double-counting** inherent in Anthropic's `totalInputTokens()` method.

Looking at `anthropic.go` line 109-110:
```go
func (u anthropicUsage) totalInputTokens() int {
    return u.InputTokens + u.CacheCreationInputTokens + u.CacheReadInputTokens
}
```

The `PromptTokens` field in `aimodel.Usage` is already set to `totalInputTokens()`, which **includes** `CacheReadInputTokens`. So if the cost tracker charges `InputPerMTokens` for all `PromptTokens` AND separately charges `CachePerMTokens` for `CacheReadTokens`, it will overcharge for cached tokens.

**Fix:** The cost calculation must subtract cache-read tokens from the input token count before applying the input rate, or the `CachePerMTokens` rate must represent the **differential** cost (cache rate minus input rate). The cleaner approach: charge `(InputTokens - CacheReadTokens) * inputRate + CacheReadTokens * cacheRate`. Document this clearly.

### CRITICAL: TaskAgent stream path emits EventLLMCallEnd directly, bypassing MetricsMiddleware

In `task.go` lines 929-935, the streaming path in `TaskAgent.runStream()` emits `EventLLMCallEnd` directly (not via MetricsMiddleware). This means `CacheReadTokens` must be propagated in **two** places:
1. `MetricsMiddleware.Wrap()` for the sync path
2. `TaskAgent.runStream()` for the stream path (where it reads `stream.Usage()` and manually constructs `LLMCallEndData`)

The design mentions only MetricsMiddleware. The task agent's direct emission must also be updated.

### HIGH: The design's "Final decision" on Pricing struct placement is problematic

The design goes through three different proposals for where to define the `Pricing`/`ModelPricingEntry` struct, and settles on defining it in `configs` with `costtraces` importing `configs`. This creates an awkward dependency: the `costtraces` package (domain logic) depends on `configs` (infrastructure). This violates the typical dependency direction.

**Fix:** Define `Pricing` in `costtraces` (it is the domain owner) and have `configs` define its own `ModelPricingEntry` YAML struct. Convert between them at the wiring point (e.g., in `main.go` or `setup`). This is a trivial conversion for a small struct and keeps both packages independent.

### HIGH: Design proposes modifying vage/service/handler.go for usage accumulation, but this belongs in vv

The `vage/service/handler.go` is a framework-level handler. The design proposes accumulating `EventLLMCallEnd` events in the streaming goroutine and injecting a synthetic `usage` SSE event. But:

1. The streaming handler (`handleStream`) receives `schema.Event` objects from `RunStream.Recv()` -- it does not decode event data types. It just marshals and forwards.
2. Adding cost/pricing logic here would leak vv-specific concerns into the vage framework.
3. The `RunResponse.Usage` field already carries `*aimodel.Usage` for the sync path. For streaming, the design should accumulate at the vv HTTP layer (in `httpapis/http.go`), not in vage.

**Fix:** Either:
- (Preferred) Have `vv/httpapis` wrap the streaming handler to intercept events and emit a usage summary. This keeps vage pricing-agnostic.
- Or add a generic usage-accumulation hook in the vage service handler that does not involve pricing (just raw token counts), and let vv add cost on top.

### MEDIUM: Anthropic streaming CacheReadTokens is available from message_start, not message_delta

The design says to "capture `CacheReadInputTokens` from `message_start` and set it on the final chunk's Usage." Looking at the actual code, `message_start` already captures `inputTokens = ms.Message.Usage.totalInputTokens()` (line 100), which **includes** cache-read tokens in the total. The `CacheReadInputTokens` value from `message_start` is available as `ms.Message.Usage.CacheReadInputTokens` but is not stored separately.

**Fix:** In `anthropicRecvFunc`, store `ms.Message.Usage.CacheReadInputTokens` in a separate variable (e.g., `cacheReadTokens`), and set it on the final chunk's Usage in the `message_delta` handler alongside `inputTokens`. This is straightforward but needs explicit mention of the variable.

### MEDIUM: OpenAI cached tokens should be extracted now, not deferred

The design defers OpenAI `cached_tokens` extraction. However, OpenAI has supported `usage.prompt_tokens_details.cached_tokens` since GPT-4o and GPT-4 Turbo models (and it is standard in the API). Since this feature is specifically about cost tracking and cache-read tokens affect cost, it is worth extracting this now. The OpenAI response already includes this data; ignoring it means OpenAI users get no cache cost benefit.

**Fix:** Add a `PromptTokensDetails` struct to handle OpenAI's nested `usage.prompt_tokens_details` in `schema.go`, and extract `cached_tokens` into `CacheReadTokens`. This is a small addition that significantly improves accuracy for OpenAI users.

### MEDIUM: Status bar format differs from requirement

The requirement (US-1, AC6) specifies: `vv | {status} | {agent} | {model} | {cost} | {tokens}`. The existing `headerView()` renders `vv . {provider} . {model}` (using dot separators). The design proposes adding a new `statusBarView()` method separate from `headerView()`, but the requirement seems to want a single unified status bar, not two separate displays.

**Fix:** Clarify whether this is an additional status bar (rendered on every frame in `View()`) or a replacement of the existing header. Given the Bubble Tea inline mode (no alt screen), a persistent status bar in `View()` is the right approach. The header is printed once at startup; the status bar should appear in the `View()` output. The design should be explicit about this distinction.

### MEDIUM: Missing CacheReadTokens in PhaseEndData

The design adds `CacheReadTokens` to `LLMCallEndData` and `SubAgentEndData` but does not mention `PhaseEndData`, which also carries `PromptTokens` and `CompletionTokens`. For consistency and correct per-phase cost tracking, `PhaseEndData` should also include `CacheReadTokens`.

### LOW: Prefix matching in LookupPricing is fragile

The design proposes prefix matching (e.g., `"claude-sonnet-4-20250514"` matches `"claude-sonnet-4"`). This can produce false matches: `"gpt-4o-mini"` would match `"gpt-4o"` since `"gpt-4o"` is a prefix of `"gpt-4o-mini"`. The matching should prefer the longest prefix match.

**Fix:** Sort pricing keys by length (longest first) when doing prefix matching, so `"gpt-4o-mini"` matches before `"gpt-4o"`.

### LOW: Usage event placement in HTTP stream

The design says "emit a `usage` event... before the `agent_end` event." But the streaming handler does not control event ordering -- it forwards events from `RunStream.Recv()`. The usage event would need to be emitted after all events are consumed (after the stream ends), which means it comes **after** `agent_end`, not before. This is actually fine semantically (it is a summary), but the design text should match reality.

### LOW: Missing `estimated_cost_usd` in HTTP sync response

The design acknowledges that cost estimation in HTTP responses is "handled by the vv HTTP layer wrapping the response with pricing lookup." But it does not describe this wrapping mechanism. For the sync path, `handleRun` in `vage/service/handler.go` returns `RunResponse` directly. The vv layer would need to intercept this or post-process the response. The design should describe this concretely.

### LOW: Thread safety note for status bar rendering

The `View()` method in Bubble Tea is called from the main goroutine, and `costTracker.Snapshot()` acquires a mutex. This is fine, but worth noting that `Snapshot()` should be fast (no allocations beyond the float64 copy) to avoid blocking the TUI render loop.

## Simplification Opportunities

1. **Remove the indecisive Pricing struct discussion.** The design spends three paragraphs going back and forth on where to put the struct. Pick one approach and state it clearly.

2. **Consolidate the cost tracker and existing CLI token accumulation.** The CLI `model` struct already tracks `totalPromptTokens` and `totalCompletionTokens`. After adding the cost tracker, these become redundant for the task completion line. Consider whether the cost tracker can replace them entirely, simplifying the code.

3. **Consider using `aimodel.Usage` directly in the cost tracker** instead of defining a new `costtraces.Usage` type. The only addition is `EstimatedCostUSD` and `CallCount`. These could be separate fields on the Tracker, with `Snapshot()` returning an embedded `aimodel.Usage` plus cost. This reduces type proliferation.

## Summary of Required Changes

| # | Severity | Issue | Fix |
|---|----------|-------|-----|
| 1 | CRITICAL | Cache-read double-counting in cost calculation | Subtract cache-read from input before applying input rate |
| 2 | CRITICAL | TaskAgent stream path emits LLMCallEnd directly | Update task.go stream path alongside MetricsMiddleware |
| 3 | HIGH | Pricing struct placement oscillation | Define in costtraces, convert at wiring |
| 4 | HIGH | Usage accumulation in vage service handler | Move to vv/httpapis layer |
| 5 | MEDIUM | Anthropic stream CacheReadTokens storage | Explicit variable in anthropicRecvFunc |
| 6 | MEDIUM | OpenAI cached tokens deferred unnecessarily | Extract prompt_tokens_details.cached_tokens now |
| 7 | MEDIUM | Status bar vs header confusion | Clarify as separate persistent status in View() |
| 8 | MEDIUM | Missing CacheReadTokens in PhaseEndData | Add field for consistency |
| 9 | LOW | Prefix matching false positives | Use longest-prefix-first matching |
| 10 | LOW | Usage event timing in stream | Correct to "after agent_end" |
| 11 | LOW | Missing HTTP sync cost wrapping mechanism | Describe concrete approach |
| 12 | LOW | Redundant CLI token tracking fields | Note consolidation opportunity |
