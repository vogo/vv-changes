# Raw requirement — P1-8 · Prompt caching hints

Source row: `doc/prd/feature-todo.md` P1-8.

> **🆕 P1-8 · Prompt Caching 主动标记** · 能力接通（成本优化）· 依赖：无 · 难度：低 · 复用：`aimodel/anthropic_chat.go`、`aimodel/openai_chat.go`、`vage/skill/`、`vv/configs/project_instructions.go`
>
> 已能从响应提取 cache-read token 但未在请求侧下发 `cache_control: {type: "ephemeral"}`；在 System Prompt、VV.md、已激活 Skill、长工具定义上打标记即可；Anthropic 实测可省 70%+ 成本；无新增依赖。

## Why this matters

Every TaskAgent ReAct iteration re-sends the system prompt (including VV.md + skill instructions) and the full tool definitions. For a 10-turn session, that's 10× identical system tokens billed at full price. Anthropic offers a prompt-cache with a write surcharge (+25% for 5-min, +100% for 1-hour) and a massive read discount (-90% on cache hits). OpenAI's API caches identical prefixes automatically for 1024-token+ requests with no request-side marker. Neither side costs us anything to add; Anthropic requires an explicit `cache_control: {type: "ephemeral"}` hint on the content block where the cache boundary should fall.

Current state (as of P1-7):
- `aimodel.Usage.CacheReadTokens` already exists and is populated from Anthropic's `cache_read_input_tokens` and OpenAI's `prompt_tokens_details.cached_tokens`.
- No code path sends `cache_control` on the request side.
- `vv/traces/costtraces` already knows about cached input pricing, so the downstream metrics side is ready.

## Market & industry survey

| Product | Cache markers | Default on | Breakpoint placement | Notes |
|---------|---------------|------------|----------------------|-------|
| Claude Code | Native SDK cache_control | Yes | System + last tool + periodic message blocks | Aggressive; 2–4 breakpoints per turn |
| Cursor (agent mode) | Yes (Anthropic via API key) | Yes | System + tools + project context | Marks stable regions only |
| Cline | Yes | Yes (when Anthropic selected) | System prompt + tools | Simple approach |
| Continue.dev | Yes | Optional | System + tools, configurable | Documents as opt-in for Anthropic; silent no-op for OpenAI |
| Aider | Yes | Yes for Anthropic models | System + repo map | Re-validates cache every edit |
| OpenAI direct | N/A (auto) | Auto | 1024-token prefix auto-cache | No request-side marker; caches for ~5–10 min |
| Gemini | Context caching API | Opt-in | Named cache handles | Different model: explicit CreateCache then reference; heavier lift |
| Anthropic direct | Manual | Opt-in | Up to 4 breakpoints | `cache_control: {type: "ephemeral"}` on any text/tool/image/document block |

Common patterns:
1. **Mark the LAST stable block**, not every block. Anthropic caches everything **up to and including** the marked block. So one marker at the end of the system prompt caches the whole system; one at the end of the tool array caches all tools.
2. **System + tools are the universal breakpoints**. Everything after (user/assistant turn messages) is per-turn and rarely worth caching.
3. **OpenAI path: no-op**. OpenAI's prefix caching is automatic; markers are ignored.
4. **Short prompts are free-passed** by Anthropic (it silently no-ops if the cached portion is below the minimum threshold — 1024 tokens for Sonnet 4 / 2048 for Haiku).
5. **Cache TTL**: `ephemeral` = 5 min by default; `"cache_control": {"type": "ephemeral", "ttl": "1h"}` → 1h with +100% write surcharge. 5-min default is right for agent sessions.

## Implementation gotchas

1. **Tool-array breakpoint is position-sensitive.** Marking the last tool caches all tools in order. If tools are registered in non-deterministic order across turns, the cache key changes and hit rate drops. `vv`'s `registries.Registry` already iterates in sorted order for dispatchable agents — verify per-agent factories also emit a stable tool list.
2. **System prompt is dynamic when skills are injected.** `vage/skill` appends skill instructions at agent start. Within one session these are stable; across sessions they may change. Still worth caching: hits only count within a session anyway.
3. **`VV.md` / `ProjectInstructions` is stable per-cwd.** Baked into the system prompt at startup. Good candidate.
4. **Don't mark user/assistant turn messages.** They change every turn; marking them writes to cache and never reads. Pure waste.
5. **Don't break the translation path.** `aimodel/anthropic.go:toAnthropicRequest` currently builds system as either a raw JSON string OR an array of content blocks. Cache markers require the block-array form. Adding a marker forces us to emit blocks even when there's just one system string. Safe; it's the same shape Anthropic accepts.
6. **Per-block `cache_control` must live on the block, not the message.** Anthropic schema is `{ "type": "text", "text": "...", "cache_control": {"type": "ephemeral"} }`. Attach on the **last** block of the target message.
7. **Opt-out for weird providers.** Some OpenAI-compatible providers barf on unknown fields. Keep the hint purely a Go-side struct flag that only gets emitted when the Anthropic backend translates; OpenAI backend drops it.
8. **No version-specific models.** Cache breakpoint syntax has been identical since Anthropic's 2024-07-31 release. No model-gating needed.
9. **Cost observability.** The response already carries `cache_read_input_tokens`; cost trackers already read it. Nothing new on the observability side, just that the numbers will now start being non-zero.
10. **4-breakpoint cap.** Anthropic rejects requests with >4 cache_control markers. We plan to emit 2 (system + tools), well within bounds.
11. **Idempotent cache key.** Any invisible whitespace change in the system prompt breaks the cache. Keep the prompt build path deterministic.
12. **Streaming path.** Cache metrics come through `message_start` / `message_delta` usage events; both already parsed. No additional streaming work.

## Autonomous decisions (no user input sought)

| # | Decision | Why |
|---|----------|-----|
| A1 | Add `CacheBreakpoint bool` field on `aimodel.Message` and `aimodel.Tool` | Minimal API surface; marker is canonical on the schema side, translation is per-backend. |
| A2 | Anthropic backend translates; OpenAI backend ignores (no error, no-op) | OpenAI auto-caches; emitting `cache_control` on OpenAI-compatible endpoints is the spec's reserved extension space but some providers barf. Silent drop is safest. |
| A3 | In taskagent: when `WithPromptCaching(true)` (default), mark the system message's `CacheBreakpoint=true` and the last tool's `CacheBreakpoint=true` | Covers 90% of the savings; dead simple. |
| A4 | vv default: on (`cfg.Agents.PromptCaching: true`); env `VV_AGENTS_PROMPT_CACHING`; single knob covers all agent factories | Matches Claude Code / Cursor default. Opt-out for weird local providers is one line of YAML. |
| A5 | No auto-detection of protocol | The marker is always set when enabled; OpenAI backend drops silently — same result whether or not we branch on protocol. |
| A6 | Don't mark user/assistant/tool messages | Lower hit rate, extra write cost. |
| A7 | Use default `ephemeral` TTL (5 min); no 1-hour option in this PR | Matches typical agent session cadence. |

## User's implicit asks (from the /loop prompt)

1. Research similar products → done.
2. Research industry practices / implementation approaches → done.
3. Note cautions → done.
4. Reference these in the design.
5. After implementation, move item from `feature-todo.md` → `feature-implement.md`.
6. Run `/gitpush` to commit & push.
