# Design — P1-7 · Parallel tool calling in TaskAgent

## 1. Approach

Factor the two existing serial tool-call loops (sync `Run` and streaming `RunStream`) into a single helper `executeToolBatch(...)` that:

1. Dispatches all `EventToolCallStart` events up-front in `ToolCalls[i]` order.
2. Fans out `executeToolCall(ctx, tc)` across a bounded worker pool (cap = `maxParallel`, default 4).
3. After all workers complete, dispatches `EventToolCallEnd` → guard run → `EventToolResult` (stream only) → tool-result message append, strictly in `ToolCalls[i]` order.

Single-call batches take a fast path: no goroutines, no semaphore, zero overhead — identical to the current serial code.

The helper's public contract differs between sync and stream because the sync path dispatches events via `a.dispatch(ctx, ev)` (which may be buffered internally) while the stream path must call `send(ev)` and propagate an error. We express that as a small `eventSink` callback — one function, two adapters.

**Rejected alternatives (and why):**
- `golang.org/x/sync/errgroup` with `SetLimit` — natural fit, but `errgroup.WithContext` cancels siblings on first error. That's not what we want; one tool error doesn't abort the batch. Plain `sync.WaitGroup` + channel semaphore is simpler and exactly right.
- Emitting End events from worker goroutines — destroys deterministic ordering, breaks existing tests.
- Per-tool opt-out metadata (`tool.ToolDef.Parallel bool`) — unscoped, and the model already controls which tools it batches.

## 2. Code shape

### 2.1 New helper in `vage/agent/taskagent/task.go`

```go
// executeToolBatch runs the tools requested by assistantMsg.ToolCalls with
// bounded concurrency, preserving deterministic event and message ordering.
// For 0 or 1 call it degenerates to the serial path with no goroutines.
//
// eventSink is invoked serially from the calling goroutine for every event
// (Start / End / Guard / Result) in ToolCalls order. It returns an error only
// for the streaming path (where send() can fail); the sync path's sink
// always returns nil.
//
// Returns the tool messages to append (in ToolCalls order) and any sink
// error (stream path only).
func (a *Agent) executeToolBatch(
    ctx context.Context,
    rc *runContext,
    toolCalls []aimodel.ToolCall,
    emitResultEvent bool, // true for stream path
    eventSink func(schema.Event) error,
) ([]aimodel.Message, error) {
    n := len(toolCalls)

    // 1. Start events in order.
    starts := make([]time.Time, n)
    for i, tc := range toolCalls {
        if err := eventSink(schema.NewEvent(schema.EventToolCallStart, agentID, rc.sessionID, schema.ToolCallStartData{
            ToolCallID: tc.ID,
            ToolName:   tc.Function.Name,
            Arguments:  tc.Function.Arguments,
        })); err != nil {
            return nil, err
        }
        starts[i] = time.Now()
    }

    // 2. Execute — fast-path single call, bounded fan-out for many.
    results := make([]schema.ToolResult, n)
    durations := make([]time.Duration, n)
    if n <= 1 || a.maxParallelToolCalls <= 1 {
        for i, tc := range toolCalls {
            results[i] = a.executeToolCall(ctx, tc)
            durations[i] = time.Since(starts[i])
        }
    } else {
        cap := a.maxParallelToolCalls
        if cap > n { cap = n }
        sem := make(chan struct{}, cap)
        var wg sync.WaitGroup
        for i, tc := range toolCalls {
            wg.Add(1)
            sem <- struct{}{}
            go func(i int, tc aimodel.ToolCall) {
                defer wg.Done()
                defer func() { <-sem }()
                results[i] = a.executeToolCall(ctx, tc)
                durations[i] = time.Since(starts[i])
            }(i, tc)
        }
        wg.Wait()
    }

    // 3. End events + guards + (stream) Result event + tool messages, in order.
    toolMsgs := make([]aimodel.Message, 0, n)
    for i, tc := range toolCalls {
        if err := eventSink(schema.NewEvent(schema.EventToolCallEnd, agentID, rc.sessionID, schema.ToolCallEndData{
            ToolCallID: tc.ID,
            ToolName:   tc.Function.Name,
            Duration:   durations[i].Milliseconds(),
        })); err != nil {
            return nil, err
        }

        res, guardEvt := a.runToolResultGuards(ctx, rc, tc, results[i])
        if guardEvt != nil {
            if err := eventSink(*guardEvt); err != nil {
                return nil, err
            }
        }

        if emitResultEvent {
            if err := eventSink(schema.NewEvent(schema.EventToolResult, agentID, rc.sessionID, schema.ToolResultData{
                ToolCallID: tc.ID,
                ToolName:   tc.Function.Name,
                Result:     res,
            })); err != nil {
                return nil, err
            }
        }

        toolMsgs = append(toolMsgs, aimodel.Message{
            Role:       aimodel.RoleTool,
            ToolCallID: res.ToolCallID,
            Content:    aimodel.NewTextContent(toolResultText(res)),
        })
    }

    return toolMsgs, nil
}
```

### 2.2 Call sites

Sync `Run` loop (around line 883):

```go
sink := func(ev schema.Event) error {
    a.dispatch(ctx, ev)
    return nil
}
toolMsgs, _ := a.executeToolBatch(ctx, rc, assistantMsg.ToolCalls, false, sink)
messages = append(messages, toolMsgs...)
```

Streaming `RunStream` loop (around line 1137):

```go
toolMsgs, err := a.executeToolBatch(ctx, rc, accumulated.ToolCalls, true, send)
if err != nil {
    return err
}
messages = append(messages, toolMsgs...)
```

### 2.3 Config fields (vage option + vv config)

In `vage/agent/taskagent/task.go`:
```go
type Agent struct {
    ...
    maxParallelToolCalls int  // 0 → default 4; 1 → serial; N → cap
}

const defaultMaxParallelToolCalls = 4

func WithMaxParallelToolCalls(n int) Option {
    return func(a *Agent) {
        if n < 0 { n = 0 }
        a.maxParallelToolCalls = n
    }
}
```

Set the default in `New`:
```go
a := &Agent{
    ...
    maxParallelToolCalls: defaultMaxParallelToolCalls,
}
```

In `vv/configs/config.go` `AgentsConfig`:
```go
MaxParallelToolCalls int `yaml:"max_parallel_tool_calls"` // default 4; <=1 serializes
```

Default applied in `applyDefaults`: if zero → 4. Env override `VV_AGENTS_MAX_PARALLEL_TOOL_CALLS`.

In `vv/registries/registry.go` `FactoryOptions`:
```go
MaxParallelToolCalls int
```

In `vv/setup/setup.go`, the `factoryOpts` builder passes `cfg.Agents.MaxParallelToolCalls` through. Each of the six agent factories under `vv/agents/*.go` appends `taskagent.WithMaxParallelToolCalls(opts.MaxParallelToolCalls)` to its option list.

## 3. Concurrency correctness notes

- **Races on `results[i]` / `durations[i]`** — each goroutine writes to its own unique index; no overlap. Reads only after `wg.Wait()`. No mutex needed.
- **`a.executeToolCall` reentrancy** — the method reads `a.toolRegistry`, which is immutable after `New`. Tool handlers themselves are user-supplied; per the tool.Registry contract they must be goroutine-safe (the MCP client, bash tool, read/write tools already are).
- **Hook manager** — `a.dispatch` and the stream `send` are only called from the main goroutine (in step 3). Parallel workers do not touch the hook manager.
- **Guards** — `runToolResultGuards` is called sequentially in step 3 from the main goroutine, never concurrently. Matches today's behaviour.
- **Context cancellation** — in-flight `executeToolCall(ctx, tc)` observes ctx cancellation and returns an error-result. Workers that had not yet acquired the semaphore still do so (it's buffered), run briefly, and exit. Total extra work is bounded by `maxParallel` calls.

## 4. Observability

`EventToolCallStart` remains first, `EventToolCallEnd` carries the per-call `Duration`, guard and result events follow. Nothing new. Tests asserting event sequence continue to work because the sequence is identical when tools complete in order.

Log noise: none added; the existing `slog.Warn` calls inside `executeToolCall` and guards still fire per-call.

## 5. Testing plan

### 5.1 Unit — `vage/agent/taskagent/task_test.go`

Add:

- `TestParallelToolCalls_WallClockIsMax` — fake ChatCompleter returns an assistant message with 2 tool calls; each tool sleeps 200 ms. Assert total elapsed < 300 ms and both results appear in order. Covers AC-1.1.
- `TestParallelToolCalls_RespectsConcurrencyCap` — 8 tool calls, each sleeps 100 ms, `WithMaxParallelToolCalls(4)`. Assert elapsed between 180 ms and 280 ms (two waves). Covers AC-1.2.
- `TestParallelToolCalls_EventOrderingDeterministic` — 4 tool calls with randomised sleep durations; collect events via a capture hook; assert Start events come before any End events, and End/Result events appear in `ToolCalls[i]` order. Covers AC-2.1, AC-2.2.
- `TestParallelToolCalls_DurationIsPerCall` — tool 0 sleeps 50 ms, tool 1 sleeps 150 ms; assert `EventToolCallEnd.Duration` ≈ 50 ms for call 0 and ≈ 150 ms for call 1. Covers AC-2.3.
- `TestParallelToolCalls_ErrorInOneDoesNotBlockSiblings` — tool 1 returns `IsError=true`; assert tools 0 and 2 still produce normal results and the error result lands at index 1. Covers AC-3.1, AC-3.2.
- `TestSerialToolCalls_WhenCapIsOne` — `WithMaxParallelToolCalls(1)` with 3 tool calls; assert total elapsed ≈ sum (not max). Covers AC-4.1.
- `TestSingleToolCall_NoGoroutines` — one call: event sequence matches current serial code; test survives with `race` detector clean.

Reuse the existing `mockChatCompleter` and any `mockTool` pattern from `task_test.go`.

### 5.2 Unit — `vv/configs/config_test.go` (or a new small file)

- `TestAgentsConfig_MaxParallelToolCallsDefault` — loads a config with no `max_parallel_tool_calls` set; defaults to 4.
- `TestAgentsConfig_MaxParallelToolCallsEnvOverride` — env `VV_AGENTS_MAX_PARALLEL_TOOL_CALLS=2` beats YAML.

### 5.3 Integration — `vv/integrations/agents_tests/agents_tests/`

One test that builds a coder agent via `setup.New`, feeds a mock assistant response with two `read` tool calls against real temp files, and asserts both results appear in order. This is throughput-neutral but confirms end-to-end wiring through the factory.

## 6. Files touched

| File | Change |
|------|--------|
| `vage/agent/taskagent/task.go` | Add `maxParallelToolCalls`, `WithMaxParallelToolCalls`, default const, new `executeToolBatch`, replace the two inline tool loops with helper calls. |
| `vage/agent/taskagent/task_test.go` | 7 new unit tests. |
| `vv/configs/config.go` | Add `AgentsConfig.MaxParallelToolCalls` + default-applying code + env override. |
| `vv/configs/*_test.go` | 2 small test additions. |
| `vv/registries/registry.go` | Add `FactoryOptions.MaxParallelToolCalls`. |
| `vv/agents/{coder,researcher,reviewer,chat,explorer,planner}.go` | Pass `taskagent.WithMaxParallelToolCalls(opts.MaxParallelToolCalls)` in each factory. |
| `vv/setup/setup.go` | Thread `cfg.Agents.MaxParallelToolCalls` into `FactoryOptions`. |
| `vv/integrations/agents_tests/agents_tests/` | One integration test. |

## 7. Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Goroutine leak on never-completing tool | Tool's own ctx-respect is the contract; we do not spawn watchdogs here. Matches existing code. |
| FD pressure at high cap | Cap defaults to 4; users raise at their own risk. |
| Downstream code expected End-before-Start interleaving | Current code emits End right after its own tool's work, before the next Start — our new ordering (all Start first, then all End) is a small shift. Search the repo for callers of `schema.EventToolCallStart` / `EventToolCallEnd` and confirm no logic depends on interleave. |
| Tests with tight timing on mocks flake on CI | Tolerance bands ±50 ms in tests; avoid `== expected`. |

## 8. Follow-ups explicitly deferred

- Per-tool `Parallel bool` flag for mutating tools. Ship when a user hits an actual race.
- Adaptive concurrency (AIMD) — over-engineered for a batch of 2-6 calls.
- `parallel_tool_calls: false` knob at the `aimodel` request level — orthogonal feature, belongs with P1-8 cache-control work.
