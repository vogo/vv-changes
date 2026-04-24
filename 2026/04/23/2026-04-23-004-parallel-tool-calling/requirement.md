# Requirement — P1-7 · Parallel tool calling in TaskAgent

## 1. Background & objective

When an LLM returns an assistant message with multiple tool calls in a single response, `vage/agent/taskagent/Agent` currently walks the `ToolCalls` slice serially — each call's full duration (`ToolCallStart → execute → ToolCallEnd → guards → append`) must complete before the next starts. For read-heavy tasks (`read` on 5 files, `grep` across 3 patterns, parallel `glob`s) wall-clock latency is the sum of individual durations, even though each tool is I/O-bound and would benefit from concurrent execution.

**Objective:** execute the tool calls in a single assistant response concurrently, with a small bounded concurrency cap, while preserving (a) the deterministic event ordering today's tests and downstream consumers rely on, (b) the existing error-in-ToolResult pattern (one failure does not short-circuit siblings), and (c) byte-identical behaviour for single-tool responses.

## 2. Scope

### In-scope

1. Parallel dispatch of tool calls in both the sync `Run` path (task.go ~883-910) and the streaming `RunStream` path (~1137-1179) in `vage/agent/taskagent/task.go`.
2. New option `taskagent.WithMaxParallelToolCalls(n)`:
   - `n <= 1` → serial (old behavior; escape hatch).
   - `n >= 2` → bounded parallel, cap = `n`.
   - unset → default `4`.
3. `cfg.Agents.MaxParallelToolCalls int` YAML / env knob in vv; plumbed through `registries.FactoryOptions.MaxParallelToolCalls` and the existing six agent factories.
4. Env override `VV_AGENTS_MAX_PARALLEL_TOOL_CALLS`.
5. Unit tests in vage covering: single tool (serial path unchanged), parallel with N=4 calls, cap respected, error-in-one-result doesn't block siblings, event ordering deterministic (Start_1 < Start_2 < End_1 < End_2 when order matches), serial mode still works (`WithMaxParallelToolCalls(1)`).
6. Integration test in vv that a `Dispatcher`-hosted agent executing two independent reads completes in ≈ max(t_1, t_2), not t_1 + t_2 (tolerance-checked).

### Out-of-scope (documented)

- Per-tool opt-out metadata (e.g., "never parallelize Write"). The LLM is trusted not to issue racing writes — matches Claude Code / Cursor / Cline.
- Cross-assistant-message parallelism (tool calls from different turns). Still strictly serial across turns.
- Cancellation-on-first-error semantics. Errors stay captured in `ToolResult`; siblings continue.
- Changes to `executeToolCall`, `runToolResultGuards`, guard execution order, or budget / memory plumbing.
- `aimodel/` / `vv/dispatches/` changes.

## 3. User stories & acceptance criteria

### US-1 · Throughput for parallelizable workloads

**As** a developer running an agent that issues a batch of independent reads/greps, **I want** those tool calls to execute concurrently **so that** the wall-clock latency approaches `max(duration_i)` rather than `sum(duration_i)`.

**AC-1.1** — With default config and an assistant message requesting two 200 ms simulated-latency tool calls, total tool-phase wall time is `< 300 ms` (vs. `> 400 ms` serial). Verified via unit test with a controllable mock tool.

**AC-1.2** — With 8 calls and `MaxParallelToolCalls = 4`, the batch finishes in roughly `2 × (slowest-call-time)` (caps respected).

### US-2 · Deterministic event ordering

**As** an operator reading event traces or running the integration tests, **I want** event ordering for tool calls to remain deterministic **so that** existing test assertions (and any downstream consumer of `EventToolCallStart` / `EventToolCallEnd` / `EventToolResult`) keep working.

**AC-2.1** — `EventToolCallStart` is emitted for each tool call in `ToolCalls[i]` order before any call executes.
**AC-2.2** — `EventToolCallEnd`, `EventGuardCheck` (if any), and `EventToolResult` (stream path only) for each call are emitted in `ToolCalls[i]` order after **all** calls complete.
**AC-2.3** — The `Duration` field on `EventToolCallEnd` reflects the individual call's wall-clock time (measured per-goroutine), not the batch duration.

### US-3 · Error isolation

**As** an agent developer, **I want** one tool call failing (or timing out) to not block its siblings **so that** the remaining results still land in the conversation transcript.

**AC-3.1** — When call N=3 returns a tool error, calls 1, 2, 4 still execute, their results appear in the transcript in index order, and call 3's error-shape `ToolResult` is appended in the correct slot.
**AC-3.2** — The assistant-message's `ToolCalls` and the appended tool-result messages have a 1:1 mapping in original order — no missing slots, no duplicates.

### US-4 · Opt-out / backward compatibility

**As** an integrator with tools whose side-effects I want serialized, **I want** a one-line config to serialize tool execution **so that** I can opt out of parallelism.

**AC-4.1** — Setting `agents.max_parallel_tool_calls: 1` (YAML) or `VV_AGENTS_MAX_PARALLEL_TOOL_CALLS=1` (env) produces byte-identical behaviour to pre-P1-7 (verified by keeping existing TaskAgent tests passing unchanged on this setting).

**AC-4.2** — When an assistant message has exactly one tool call, no goroutines are spawned — the path is the same as the current serial code.

### US-5 · No regression

**As** a maintainer, **I want** the rest of TaskAgent's behaviour (budget middleware, guards, compactor integration, memory promotion, iteration counting) to be unchanged.

**AC-5.1** — All existing `vage/agent/taskagent/*_test.go` tests pass.
**AC-5.2** — All existing `vv/` integration tests pass (37/37 packages after P1-6 → must still be 37/37 after P1-7).
**AC-5.3** — `make lint` returns 0 issues in vage and vv.

## 4. Success criteria

- `cd vage && make test` passes.
- `cd vv && make test` passes.
- Both module lints clean.
- `doc/prd/feature-todo.md` row P1-7 moved to `feature-implement.md` with the session directory referenced.
- Two commits pushed (vage + doc/prd). `vv` only if the wiring needed changes.

## 5. Assumptions

| # | Decision | Why |
|---|----------|-----|
| A1 | Default `MaxParallelToolCalls = 4` | Cursor / Continue default; balances throughput vs. FD pressure. |
| A2 | `n <= 1` serializes | Clean escape hatch without adding a second bool flag. |
| A3 | Start events dispatched up-front in order; End/Result events after all done in order | Event determinism > per-goroutine event freshness. Tests rely on this. |
| A4 | Errors captured in `ToolResult` as today | Matches current behaviour; no new error plumbing. |
| A5 | No per-tool opt-out metadata | Follows Claude Code / Cursor; surgical scope for this PR. |
| A6 | `Duration` measured per-goroutine | Accurate attribution. |

## 6. Inconsistencies noted

None against existing docs. The feature-todo row and design are internally consistent.
