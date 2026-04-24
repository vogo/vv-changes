# Raw requirement — P1-7 · Parallel tool calling

Source: `doc/prd/feature-todo.md` row P1-7.

> **🆕 P1-7 · 并行工具调用支持** · 能力接通（主流对齐）· 依赖：无 · 难度：中 · 复用：`aimodel/schema.go`（已有 `ToolCalls []`）、`vage/agent/taskagent/task.go`
>
> LLM 本已可在一次响应中返回多个 tool_call；当前 TaskAgent 按数组顺序串行执行。改为 errgroup 并发派发，显著提升多文件读取/批量搜索类任务吞吐；需统一超时/错误聚合 + 事件顺序保留；风险低，ROI 极高。

## Why this matters

A single `assistantMsg.ToolCalls` batch from a modern LLM typically has 2–6 calls for tasks like "show me these five files" or "grep for these three patterns across the repo." With the current serial loop in `vage/agent/taskagent/task.go:883-910` (sync path) and `1137-1179` (stream path), wall-clock latency is the sum of each tool's duration. File I/O and grep calls are mostly I/O-bound, so most of that time is wasted waiting.

## Market & industry survey

| Product | Parallelism | Concurrency cap | Event ordering | Notes |
|---------|-------------|-----------------|----------------|-------|
| Claude Code | Default parallel | ~10 internal default | Start events emitted up-front; End/Result events as they land | Documented default; mutating tools (Write, Edit, Bash) still parallel — model is trusted not to issue conflicting calls |
| Cursor (v0.40+) | Default parallel | 4 (configurable) | Stream: interleaved | Flag to serialize on edit/write tools |
| Cline (v3+) | Default parallel | User-tunable | Per-tool spinner in UI | Same: trusts the model |
| Aider | Serial | N/A | Strict serial | Conservative — avoids write races |
| Codex / OpenAI Responses API | Parallel by default | Model-driven | SSE | Supports `parallel_tool_calls: false` per request |
| Gemini API | Parallel by default | Up to 8 | Unordered results | `disable_parallel_tool_use` available |
| Anthropic Messages API | Parallel by default | Up to 10 per turn | Tool results returned in any order | `disable_parallel_tool_use` option |
| Continue.dev | Parallel | 4 | Strict | Errors aggregated but don't short-circuit |
| LangChain `create_react_agent` | Parallel via asyncio.gather | Unbounded (!) | Pure interleaved | Documented risk of resource exhaustion on many calls |

## Common implementation patterns

1. **Bounded worker pool** (semaphore or errgroup.SetLimit). Claude Code, Cursor, Cline, Continue.
2. **Start events dispatched up-front in order**, then results handled by goroutines, then End/Result events dispatched in original tool-calls-array order. Keeps per-call event ordering stable while fanning out execution.
3. **Tool-result messages re-ordered to match `ToolCalls` slice order** before appending to the conversation, regardless of goroutine finish order. The LLM doesn't actually care about order (tool_call_id routes results), but deterministic ordering simplifies testing and context-compression logic.
4. **Errors captured into `ToolResult` content, not propagated up.** Matches current behavior — the existing `executeToolCall` already returns a `schema.ToolResult` with an error message string on failure. One tool failing doesn't block siblings.
5. **Opt-out flag** — most projects let the user serialize with a flag or per-tool annotation. For conservative mode or for tools like `bash` with shared-state risk.
6. **Context cancellation** — if the run's ctx is cancelled, in-flight goroutines observe it through their own `executeToolCall(ctx, ...)` call. Goroutines don't spawn new work after cancellation; results in-flight continue but are discarded.
7. **Single hook dispatch goroutine for streaming** — only the main goroutine calls `send()` for the streamed event channel. Parallel tool execution happens on side goroutines; results are collected and then the main goroutine serially emits End/Result events. This avoids needing a mutex around the stream channel.

## What to watch out for

1. **Event ordering contract** — users (and tests) expect `ToolCallStart_i` before `ToolCallEnd_i` before `ToolResult_i`. If we dispatch events from worker goroutines, inter-call interleaving becomes nondeterministic and breaks any test that asserts event sequence.
2. **`Duration` field** — must measure each tool's own wall-clock (`time.Since(start_i)`), not the elapsed time between two `ToolCallEnd` emissions — otherwise `Duration` inflates to the full batch duration for all but the first.
3. **Guard side-effects** — `runToolResultGuards` may mutate the result (redact, block). It must not run in parallel with other guards if any guard holds shared state; easier to run guards serially after all tools finish.
4. **Budget middleware** — the post-call budget check currently runs once per assistant message (before the tool loop). That stays correct with parallel execution.
5. **Write tools can race.** Example: LLM returns parallel calls `write("a.txt", ...)` and `edit("a.txt", ...)`. Physical file write order is nondeterministic. Our scope decision: **trust the LLM not to issue conflicting calls** (mirrors Claude Code, Cursor, Cline). Documented.
6. **Bash tool** — one `bash` process per call is fine in parallel if the tool's own implementation is goroutine-safe. `vage/tool/bash` already uses per-call `exec.Cmd` with its own sandboxed working dir. Low risk.
7. **Default concurrency cap** — too high burns file descriptors and CPU on grep; too low leaves throughput on the table. Industry converges on **4**.
8. **Memory usage** — parallel tool results all exist simultaneously instead of being consumed one-at-a-time. For typical 2–6 tool calls with O(KB) results, this is fine. The `ToolOutputMaxTokens` truncation already caps per-result size.
9. **Streaming path event interleaving** — since the stream channel is consumed serially and we emit End events in the original order, the downstream consumer sees a deterministic stream. ✓
10. **Test determinism** — existing tests assert event sequence; need to keep that deterministic. Emitting End events in `ToolCalls` slice order (not goroutine-completion order) preserves determinism.
11. **Tool-result ordering in `messages` slice** — matters for the *next* LLM call: the transcript replay should match what the LLM would expect. Claude/OpenAI routes by `tool_call_id`, so order doesn't matter for correctness — but deterministic order matters for prompt caching (hash stability).
12. **Context cancellation mid-batch** — each worker calls `executeToolCall(ctx, tc)` and the ctx is cancelled; the tool's own implementation should respect that. We capture the cancellation-error result, keep the slice complete, and let the outer loop return.

## Questions that settle on autonomous defaults

- **Default concurrency cap:** **4** (matches Cursor / Continue).
- **Concurrency knob location:** `taskagent.WithMaxParallelToolCalls(n)` + `cfg.Agents.MaxParallelToolCalls` in vv. `n<=1` → serial (preserves old behavior). `n==0` → use default 4.
- **Opt-out mechanism:** setting `MaxParallelToolCalls: 1` is the documented way to serialize. No per-tool metadata in this PR.
- **Env override:** `VV_AGENTS_MAX_PARALLEL_TOOL_CALLS`.
- **Event ordering:** Start events up-front in original order, then fan-out execution, then End/guard/Result events in original order once all done. **Test parity with current deterministic ordering is preserved.**
- **Errors:** one tool's error does not cancel siblings — captured into `ToolResult`, same as today. errgroup.WithContext would cancel; we instead use a bounded `sync.WaitGroup` + channel semaphore to avoid cascading cancellation.
- **Scope for this PR:** TaskAgent's two tool loops (sync + stream). No changes to `executeToolCall` itself, no changes to guards, no changes to `aimodel/` or `vv/dispatches/`. Surgical.
