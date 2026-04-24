# Code Review — P1-7 Parallel Tool Calling

Reviewed against: `design.md` (same session dir).

## Files reviewed

**vage**
- `vage/agent/taskagent/task.go` — added `maxParallelToolCalls` field, `defaultMaxParallelToolCalls` const, `WithMaxParallelToolCalls` option, new `executeToolBatch` helper; replaced two serial tool-dispatch loops.
- `vage/agent/taskagent/task_parallel_test.go` — 9 new unit tests.

**vv**
- `vv/configs/config.go` — `AgentsConfig.MaxParallelToolCalls` + default (4) + `VV_AGENTS_MAX_PARALLEL_TOOL_CALLS` env override.
- `vv/configs/agents_parallel_test.go` — 3 tests (default, YAML, env override).
- `vv/registries/registry.go` — `FactoryOptions.MaxParallelToolCalls` field.
- `vv/setup/setup.go` — threaded value into both `factoryOpts` and the explorer factory call.
- `vv/agents/{coder,explorer,researcher,reviewer}.go` — each factory passes `WithMaxParallelToolCalls`.

## Verification

| Check | Result |
|-------|--------|
| `go test ./...` in `vage/` | 47 packages ok, 0 fail |
| `go test ./...` in `vv/` | 36 packages ok, 0 fail |
| `go test -race ./agent/taskagent/...` | PASS (new parallel tests race-clean) |
| `make lint` in `vage/` | 0 issues |
| `make lint` in `vv/` | 0 issues |

## Blocking issues

**None.** The implementation matches the design 1-for-1 and is correct under `-race`. Tight focus-area analysis:

1. **Per-goroutine writes to `results[i]`/`durations[i]`** — each worker writes a disjoint slice slot; reads happen after `wg.Wait()` (happens-before via `wg`). Race detector confirms clean.
2. **Semaphore ordering** — `wg.Add(1)` precedes `sem <- struct{}{}` inside the for-loop, main blocks on the channel send when `cap < n`; worker's deferred `<-sem` runs before `wg.Done()` (LIFO), so the main goroutine always unblocks before the worker finishes. No deadlock path.
3. **Context cancellation** — workers that already started observe ctx via `executeToolCall`; late-scheduled workers run briefly and exit. No leak bounded by `maxParallel` extra calls (matches design §3).
4. **Guard ordering** — `runToolResultGuards` runs serially in step 3, preserving prior semantics. Guard-mutation of `rc` is therefore race-free.
5. **Message order** — `toolMsgs` is appended inside the index-ordered step-3 loop over `toolCalls`, so tool-result messages are 1:1 with `ToolCalls[i]` order regardless of worker finish order. Verified by `TestParallelToolCalls_ErrorInOneDoesNotBlockSiblings`.
6. **`agentID` parameter** — both callers pass the outer `agentID := a.ID()` variable; verified by grep.
7. **Fast-path equivalence at `n == 1`** — Start → work → End → Guard → (Result) → append is byte-identical to the pre-P1-7 code. For `n > 1, cap == 1` the event interleave shifts to Start_0…Start_{n-1} then End_0, Guard_0, (Result_0), …, End_{n-1} — design §7 explicitly notes this and no existing consumer depends on the old interleave (grep of `EventToolCallStart|End` across the repo confirms).
8. **Sync path sink error** — closure always returns nil; `toolMsgs, _ := …` is safe. Left as-is to keep diff surgical.

## Non-blocking improvements applied

None. The diff is tight, the helper is well-commented, and the tests cover AC-1.1, 1.2, 2.1, 2.2, 2.3, 3.1, 3.2, 4.1 and the option defaults/clamping. Applying further polish would be opportunistic rework outside the P1-7 scope.

## Non-blocking improvements NOT applied (with reason)

1. **Tighten the timing tolerances in `TestParallelToolCalls_RespectsConcurrencyCap` (180–320 ms range).** Current ±40 % band is reasonable for CI; tightening risks flake without adding signal. The `maxInFlight` atomic already gives a deterministic cap check.
2. **Surface the ignored error from `executeToolBatch` in the sync `Run` path.** The sync sink always returns `nil` by construction, so an `if err != nil` branch would be dead code. If a future maintainer makes the sink fallible they will notice via the compiler (error would become actually non-nil). Defensive check rejected as clutter.
3. **Hoist `parallelCap` resolution into a method.** Only one real call site; extracting for one test (`resolveParallelCap` is a private test-only helper that duplicates the two lines of clamping) would grow the public surface. The duplication is one `if` statement — accepted.
4. **Convert channel-semaphore to `golang.org/x/sync/semaphore` or `errgroup.SetLimit`.** Design §1 rejects `errgroup` explicitly because `WithContext` cancels siblings on first error, which would regress AC-3.1. The plain channel semaphore is the correct primitive here.
5. **Emit events from worker goroutines to reduce step-3 latency.** Design §1 rejects this — destroys deterministic ordering, breaks the existing streaming ordering test.

## Questions / follow-ups

- **`cap` shadowing builtin.** The design sample uses `cap := …` which shadows the builtin `cap()`. The implementation renames it to `parallelCap` — good; nothing to fix.
- **Single-assistant-message scope.** Parallelism is within a single assistant message's `ToolCalls`. Cross-iteration parallelism is out of scope; the LLM issues tool batches, then waits for replies. Confirmed with design §1.
- **Tool goroutine-safety contract.** Tool handlers must be re-entrant under `maxParallelToolCalls > 1`. Existing handlers (`bash`, `read`, `write`, `edit`, `glob`, `grep`, MCP client) already are. Future custom tool contributors should be warned in tool docs — consider a note in `vage/.doc/tool.md` as a follow-up (not in this PR's scope).
- **Chat / planner factories deliberately skipped.** Confirmed: `chat` runs with a single iteration and no ToolCalls batch; `planner` has no tools at all. Omitting `WithMaxParallelToolCalls` is correct and keeps the chat flow byte-identical to pre-P1-7.

## Conclusion

Ship it. The implementation is minimal, correct, deterministic where it must be, race-free, and faithful to the design. Tests cover the advertised acceptance criteria. Both modules build, test, and lint clean.
