# Code Review — P1-8 · Prompt caching hints

Reviewer: Claude (Opus 4.7, 1M context)
Date: 2026-04-23
Scope: aimodel / vage / vv diffs for P1-8.

## Summary

Design lands cleanly in all three modules. The aimodel translation correctly
produces the "exactly 2 markers" contract under vv's default config (AC-1.1),
`json:"-"` prevents any leakage into the OpenAI body (AC-2.1 verified by test
and by inspection of every `json.Marshal` site), and the pointer/env-override
plumbing for `VV_AGENTS_PROMPT_CACHING` is nil-safe.

One real correctness issue was found and fixed: the vage `tool.Registry.List()`
method emitted tools in non-deterministic map-range order, which would bust
the Anthropic prompt-cache prefix on every call. Everything else is polish.

## Findings

### 1. [BLOCKING — fixed] Non-deterministic tool order breaks cache stability

**Location:** `vage/tool/registry.go` — `Registry.List()`

The registry stored tools in `map[string]*entry` and `List()` iterated the
map with `for _, e := range r.entries`. Go randomizes map iteration, so the
ordering of the tool slice — and hence the ordering of the Anthropic
`tools` array, and hence the marked "last" tool — was different on every
call.

Anthropic's prompt-cache key is the byte-exact prefix up to the marked
block. If tool ordering shuffles between two requests within the 5-minute
TTL, the second request is a cache miss and the whole point of the
feature is lost on the tools surface. It also means the breakpoint
lands on a *random* tool rather than a stable one — so even the content
hashed for caching shifts turn-to-turn.

The dev's `markPromptCacheBreakpoints` helper is correct given a stable
tool order; the bug is that `prepareAITools` → `tool.Registry.List()`
doesn't provide one.

**Fix applied:** sort `defs` by `Name` in `List()` before returning. Added
a regression test (`TestRegistry_List_DeterministicOrder`) that registers
four tools in reverse order and asserts 20 successive `List()` calls all
return the name-sorted order.

This is a surgical, backwards-compatible change — no existing test asserted
a specific order; all passed after the fix. No API change.

### 2. [OK] Exactly-2-markers contract (AC-1.1)

Traced through `toAnthropicRequest`:

- System path: one `Message` with `RoleSystem` and `CacheBreakpoint=true`
  sets `anyCacheableSystem=true`, which both forces `useBlocks=true`
  (block-array form) AND marks only the last element of `systemBlocks`
  (`systemBlocks[len-1].CacheControl = ephemeralCache()`). For two
  system messages both flagged, only the tail block gets the marker —
  correct for "cache up to and including tail" semantics. → 1 marker.
- Tool loop: `for _, t := range req.Tools` only attaches `CacheControl`
  when `t.CacheBreakpoint` is true. `markPromptCacheBreakpoints` only
  flags the last tool. → 1 marker.
- Total under vv default: 2 markers. `TestToAnthropicRequest_BothBreakpoints`
  exercises exactly this and counts `"cache_control":` occurrences.

Edge cases considered:

- No system prompt at all: helper's reverse loop never finds one, skips
  the mark; tools still get a marker → 1 total. Suboptimal but non-erroring.
- No tools registered: `len(tools) > 0` guard prevents an out-of-bounds
  panic; system marker still fires → 1 total.
- System message with multimodal `parts`: the block-array path runs; all
  text parts get appended to `systemBlocks`; only the last gets the
  marker. Covered by `TestToAnthropicRequest_SystemWithPartsAndCache`.

### 3. [OK] OpenAI no-leakage (AC-2.1)

Both `Message.CacheBreakpoint` and `Tool.CacheBreakpoint` carry `json:"-"`.
Greped every `json.Marshal` call in aimodel:

- `openai_chat.go:34` marshals `req` (ChatRequest) — the `json:"-"` tag
  prevents the field from serialising.
- All other Marshal sites are in `anthropic.go` / `anthropic_chat.go`,
  which explicitly construct Anthropic-side structs; they never marshal
  a canonical Message or Tool.
- Third-party content `MarshalJSON` on `Content` marshals only
  `c.text` / `c.parts`, not the enclosing Message.

`TestChatRequest_OpenAIShape_NoCacheControl` asserts no occurrence of the
string `cache_control` OR `CacheBreakpoint` in the marshalled body.

### 4. [OK] Nil-safe `*bool` resolver (focus 5)

`EffectivePromptCaching` guards both `c == nil` and `c.PromptCaching == nil`
before dereferencing — no panic path.

The env override (`VV_AGENTS_PROMPT_CACHING`) correctly does `&b` (pointer
to stack-local bool) — that's valid Go, the bool is heap-promoted by the
escape analysis once its address is taken. `TestAgentsConfig_EffectivePromptCaching_NilReceiver`
exercises the nil-receiver branch.

### 5. [OK] Slice aliasing across ReAct iterations (focus 6)

`Run` does `messages := rc.br.messages` — slice header copy sharing the
same backing array. `markPromptCacheBreakpoints` mutates
`messages[i].CacheBreakpoint`, which is visible through every alias.
Inside the loop, `messages = append(messages, ...)` may allocate a new
backing array on capacity growth, but element values (including the
now-true `CacheBreakpoint` on index 0) are copied into the new array by
the append semantics, so the marker survives.

`RunStream` is the same pattern: mark `br.messages` pre-loop, then
`runStreamLoop` does `messages := br.messages` with the marker already
baked in.

### 6. [OK] Concurrency under P1-7 parallel tool dispatch (focus 9)

The mark step runs exactly once per `Run` / `RunStream` call, before the
ReAct loop. Tool results are appended serially after the parallel batch
completes (`executeToolBatch` returns only after all goroutines join).
No concurrent reader/writer on the `CacheBreakpoint` fields.

### 7. [OK] Cost model (focus 8)

`vv/traces/costtraces/tracker.go:72-76` computes
`nonCachedInput = InputTokens - CacheReadTokens` and bills cached reads
at `CachePerMTokens` — separate from writes. Anthropic cache-write tokens
flow in via `InputTokens + CacheCreationInputTokens + CacheReadInputTokens`
(see `anthropicUsage.totalInputTokens`), so cache-write tokens are in
`PromptTokens` and billed at full input price (the 25% write surcharge
is the only understatement — explicitly listed as out-of-scope follow-up).

No double-counting. No misattribution of cache-writes as cache-reads.

### 8. [OK] Tests & lint

Before changes: aimodel 6/6, vage 47/47, vv 36/36 — all clean.
After changes: same counts, plus 1 new regression test
(`TestRegistry_List_DeterministicOrder`). All three modules pass tests
and lint.

## Files changed during review

- `vage/tool/registry.go` — sort `List()` output by name for cache-stable
  tool ordering; docstring explains why.
- `vage/tool/registry_test.go` — added `TestRegistry_List_DeterministicOrder`.

## Not changed (considered, rejected)

- Sorting `aiTools` inside `prepareAITools` rather than `List()` — would
  paper over the underlying nondeterminism, and every other caller of
  `List()` (skill discovery, MCP tool hand-off, debug output) would keep
  the shuffle bug. Root-cause fix is cheaper.
- Promoting `PromptCaching` to per-agent override — design explicitly
  rejected it (symmetric across tool-using agents). Fine.
- Adding a dedicated integration test in `vv/integrations/setup_tests/`
  for the `ChatRequest.Messages[0].CacheBreakpoint == true` end-to-end
  assertion — the taskagent unit test `TestAgent_Run_PromptCachingDefault`
  already proves the flag flows from constructor → outbound request via
  a recording mock, and the aimodel tests cover the translator side.
  The design doc §5.4 mentions this as a vv integration test; treating
  it as follow-up, not a blocker — full coverage exists via the
  composition of the two unit suites.
