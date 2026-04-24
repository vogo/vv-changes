# Requirement — P1-8 · Prompt caching hints

## 1. Background & objective

vv already reads `cache_read_input_tokens` (Anthropic) and `prompt_tokens_details.cached_tokens` (OpenAI) from responses — so the cost-accounting side is ready for prompt-cache savings. But no code path sends the **request-side** `cache_control: {type: "ephemeral"}` hint that Anthropic requires to actually populate the cache. Every ReAct iteration re-bills the full system prompt and tool array at normal input-token prices.

**Objective:** emit `cache_control` hints on the two stable, per-session surfaces — the system prompt and the tool array — in Anthropic requests. This is a low-risk change with a large cost win (70%+ savings on cached input tokens after the first turn of a session) and no schema changes for the OpenAI path (OpenAI prefix-caches automatically).

## 2. Scope

### In-scope

1. `aimodel/schema.go`: add `CacheBreakpoint bool` to `Message` and to `Tool`. Opt-in, zero-value disabled, JSON-omitted when false.
2. `aimodel/anthropic.go`: in `toAnthropicRequest` and `toAnthropicMessage`, attach `cache_control: {type: "ephemeral"}` to the last content block of a message (or the tool object) whose `CacheBreakpoint` is true. System-message breakpoints flow into the `system` field's block-array form.
3. `aimodel/openai_chat.go`: OpenAI translation ignores the flag. No output schema change — the `CacheBreakpoint` bool is struct-local and doesn't serialise (it's a Go field with no JSON tag needed on the request side since `ChatRequest.Messages[].CacheBreakpoint` doesn't get sent — we only emit it in the Anthropic translation path).
4. `vage/agent/taskagent/task.go`: `WithPromptCaching(bool)` option (default ON — matches the industry norm). When enabled, after `buildInitialMessages` mark the last system message `CacheBreakpoint=true`, and mark the last tool in the aiTools slice `CacheBreakpoint=true`.
5. `vv/configs/AgentsConfig.PromptCaching *bool` (default true; pointer so users can disable by writing `false`), env override `VV_AGENTS_PROMPT_CACHING`.
6. `vv/registries/FactoryOptions.PromptCaching bool`, threaded through `setup.Init`, consumed by the four tool-using agent factories (coder / researcher / reviewer / explorer). chat and planner skip it (chat has no tools, planner is single-iteration no-tool).
7. Unit tests for `aimodel` (Anthropic translation emits `cache_control`; OpenAI ignores) + `taskagent` (feature on/off behaviour) + `vv/configs` (default + env override).
8. One integration test in vv asserting that when `PromptCaching=true`, the outbound Anthropic request body contains exactly 2 `cache_control` markers.

### Explicitly out-of-scope

- 1-hour TTL (`ephemeral` default 5-minute is right for agent sessions).
- Cache markers on per-turn user/assistant/tool messages (low hit rate, high write cost).
- Dynamic breakpoint placement based on token counts.
- Client-side cache-hit verification / validation.
- Pricing recalc for cached writes — `vv/traces/costtraces` can follow up separately.
- Gemini-style `CreateCache` handle API (different model, out of scope).

## 3. User stories & acceptance criteria

### US-1 · Automatic caching for Anthropic

**As** an operator running vv with an Anthropic model, **I want** prompt-cache markers to land on my system prompt and tool definitions **so that** subsequent ReAct iterations within a session see `cache_read_input_tokens > 0` and I pay ~10× less for repeat input tokens.

**AC-1.1** — With default config (`agents.prompt_caching` unset) and protocol `anthropic`, the outbound HTTP request body contains exactly 2 occurrences of `"cache_control":{"type":"ephemeral"}`: one at the end of the system message blocks, one on the last tool definition.
**AC-1.2** — The ChatResponse's `Usage.CacheReadTokens` from a second ReAct iteration in the same session is > 0 (tested against an Anthropic-style mock server that echoes `cache_read_input_tokens` on the second call).
**AC-1.3** — When the system message is text-only (no blocks), the translation still emits the block-array form with a single `{type:"text", text:"...", cache_control:{...}}` entry — matches Anthropic's required shape.

### US-2 · No-op on OpenAI

**As** an operator running vv with OpenAI (or an OpenAI-compatible endpoint), **I want** the cache hint to be silently ignored **so that** (a) providers that reject unknown fields don't barf, and (b) behaviour is unchanged relative to pre-P1-8.

**AC-2.1** — With `protocol: openai`, the outbound request JSON contains **no** `cache_control` field anywhere in `messages` or `tools`.
**AC-2.2** — All existing OpenAI-path tests pass with no changes.

### US-3 · Opt-out

**As** an operator on a provider that misbehaves, **I want** a single config knob to disable cache markers.

**AC-3.1** — `agents.prompt_caching: false` (YAML) or `VV_AGENTS_PROMPT_CACHING=false` (env) causes no `cache_control` fields in the outbound Anthropic request body.
**AC-3.2** — The Go `*bool` default means the absence of any config yields `true`; an explicit `false` disables.

### US-4 · No regression

**AC-4.1** — All existing aimodel, vage, and vv tests pass (47 / 47 + 36 / 36).
**AC-4.2** — `make lint` returns 0 issues in all three modules.
**AC-4.3** — `CGO_ENABLED=0 go build ./...` still works in vv.

### US-5 · Clean opt-in API for aimodel consumers

**AC-5.1** — `Message.CacheBreakpoint` and `Tool.CacheBreakpoint` are exported Go fields with godoc. Third parties using `aimodel` directly can mark messages for caching without touching internal Anthropic plumbing.

## 4. Success criteria (verifiable)

- `cd aimodel && make test && make lint` passes.
- `cd vage && make test && make lint` passes.
- `cd vv && make test && make lint` passes.
- `doc/prd/feature-todo.md` P1-8 row moved to `feature-implement.md`.
- Commits pushed on aimodel, vage, vv, and doc/prd.

## 5. Assumptions surfaced

| # | Decision | Rationale |
|---|----------|-----------|
| A1 | Default ON for vv | Claude Code / Cursor default; zero-risk for OpenAI; cache-write surcharge is dwarfed by repeat-read savings on any session >1 turn. |
| A2 | Mark only system + last tool | Two breakpoints cover >90% of the savings; well under Anthropic's 4-breakpoint cap. |
| A3 | `*bool` with nil-default-true | Lets users explicitly disable with `false` while the zero-value stays enabled. |
| A4 | OpenAI path drops the hint silently | Avoids provider-specific "unknown field" errors. |
| A5 | No per-agent override | The four tool-using agents are symmetric in this regard; no reason to diverge. |

## 6. Inconsistencies noted

None with existing docs. The feature-todo row and design align cleanly.
