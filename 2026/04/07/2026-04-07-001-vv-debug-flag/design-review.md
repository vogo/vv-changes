# Design Review: vv `--debug` Flag

Reviewer perspective: pragmatic technical architect. Verified against current code in
`vv/setup/setup.go`, `vv/cli/prompt.go`, `vv/httpapis/http.go`, `vage/largemodel/middleware.go`,
`vage/tool/registry.go`, and `aimodel/stream.go`.

## Verified

- `largemodel.Middleware` / `largemodel.Chain` exist as described — drop-in for `aimodel.ChatCompleter` wrapping. OK.
- `tool.ToolRegistry` and `Registry.Execute` are the single chokepoint for local + external (MCP) calls. OK.
- `setup.New` already wraps registries via `opts.WrapToolRegistry` then `tool.NewTruncatingToolRegistry`. The decorator can slot in.
- `aimodel.Stream` exposes `Recv()` / `Close()` / `Usage()` and supports `WrapStream(s, onClose)` — but it does NOT expose deltas through a callback. To capture deltas the middleware must replace the stream with a wrapper that intercepts each `Recv()`. The simple `WrapStream(onClose)` only fires a callback at close and only carries `*Usage` — content is NOT available there. The design glosses over this.

## Critical Issues

### C-1. Interactive CLI mode WILL corrupt the TUI when writing to stderr
Design §3 / R-4 claim "bubbletea owns stdout; debug records go to os.Stderr which bubbletea does not capture."
This is wrong in practice. Bubble Tea writes ANSI cursor/clear sequences to the terminal device (typically `/dev/tty`, often stdout). On a normal terminal, stderr is the SAME tty, so any line written to stderr while the TUI is running will be drawn over the TUI viewport on the next frame and corrupt rendering. Bubble Tea does not "capture" stderr but the user does see scrambled output.

The non-interactive `-p` mode (`vv/cli/prompt.go`) already writes status lines to stderr and is fine because there is no TUI. So the design works for `-p` and HTTP, but not for the interactive TUI.

**Fix:** in interactive TUI mode, debug output must go somewhere that does not share the TTY. Options (pick one, MVP-friendly):
1. Auto-redirect to a file when interactive TUI + `--debug`: e.g., `~/.vv/debug-<pid>.log`, log path printed once at startup on the TUI status line. Simple, predictable, no TUI corruption.
2. Require the user to redirect stderr themselves (`vv --debug 2> debug.log`) and detect with `term.IsTerminal(int(os.Stderr.Fd()))`: if stderr is still a tty AND interactive mode AND `--debug`, refuse to start with a clear message OR auto-fall-back to the file sink in (1).
3. Send to `slog.Default()` with a file handler installed in interactive mode.

Recommend (1) with a printed banner. Keeps zero behavioral change when off, no requirement to remember shell redirection, no TUI corruption. The `-p` mode keeps stderr as today.

### C-2. Streaming consolidation needs a real wrapper, not `WrapStream`
`aimodel.WrapStream` only fires `onClose(*Usage)` — it has no access to the chunk stream itself. To produce a consolidated record the middleware must return a `*aimodel.Stream` whose `recv` is replaced (or, more cleanly, return a brand-new `Stream` constructed via an internal helper). Since `Stream`'s `recv` field is unexported, the middleware cannot construct one outside the `aimodel` package.

**Fix:** add a tiny constructor in `aimodel` (e.g., `NewStreamFromFunc(recv func() (*StreamChunk, error), onClose func()) *Stream`), OR have the middleware return its own type that satisfies the interface the caller actually uses. The latter is impossible because `taskagent` consumes `*aimodel.Stream` directly (concrete type), not an interface. So this requires either:
- Small `aimodel` PR to add an exported constructor / interceptor hook, OR
- Move the streaming-aggregation logic upstream of `aimodel.Stream` by providing an `OnRecvChunk` callback added to `Stream`.

The design must call this out and pick one. Recommend a small additive change in `aimodel`: `func InterceptStream(s *Stream, onChunk func(*StreamChunk), onDone func(err error)) *Stream` that swaps `s.recv` internally. Additive, no API break, no behavior change.

### C-3. HTTP request-id propagation does not exist today
Design §3.2 says "extract from existing request-id middleware; if none, add a tiny one that puts a UUID into ctx". `vv/httpapis/http.go` currently uses `slog.Default()` only and has no request-id middleware. The design must include adding one (small) and writing it into ctx via `debugs.WithRequestID`. Otherwise records will lack correlation in HTTP mode.

Also: `vage/service` (which serves the actual agent endpoints) does its own context handling. The middleware needs to wrap the `vage/service` mux, not be added inside vv only — verify in `vage/service/service.go` that ctx propagates from the http handler to the agent's `Run`/`RunStream` (it does, via standard `r.Context()`). So a `func(http.Handler) http.Handler` wrapper added in `vv/httpapis/http.go` around the `service.Service` ServeHTTP is sufficient.

### C-4. Agent name in ctx is "best-effort" — but uniform coverage is a stated rule
R-3 says no agent is silently exempt and every record should be interpretable. Leaving `AgentName` empty for sub-agent LLM calls makes the "uniform coverage" claim weaker. The fix is one line per agent dispatch site (`dispatches/execute.go`) that wraps ctx with `debugs.WithAgentName(ctx, desc.ID)` before calling `agent.Run`. The design lists this as "non-blocking"; promote it to a required step. Same for the explorer/planner/plan-gen/intent calls — they're all dispatched from the dispatcher and easy to tag.

## Notable Gaps

### G-1. Compactor's own LLM call is not covered
`setup.Init` builds a `compactSummarizer` that calls `llmClient.ChatCompletion` directly (line 317). If the debug middleware is wrapped onto `llmClient` BEFORE this closure is constructed, this is automatically covered. The design says "wrap llmClient … before constructing compactor and New" — good — but make this explicit in the implementation steps and add a unit test that triggers compaction and asserts a debug record appears with `agent.name = "compactor"` (or similar).

### G-2. Dispatcher's intent / planner / plan-gen LLM calls are covered, but not the explorer's tool calls
`setup.New` wraps the explorer's tool registry only with `NewTruncatingToolRegistry`, not with `opts.WrapToolRegistry`. Today the explorer is therefore exempt from CLI permission prompts AND would be exempt from the debug decorator unless the design explicitly wraps the explorer registry too. The design §6 mentions "Apply the same wrap to the explorer registry" — make sure this is in the implementation steps as a code change to `setup.New`, not just a comment.

### G-3. Redaction of `Authorization` headers and base URL query strings
`Params` map in the LLM record may carry the underlying http client config if the snapshot is broad. Today `aimodel.ChatRequest` does not include the API key (it's set at client level), so the request payload is safe. However if the design ever logs an "LLM client config snapshot" (mentioned in §4.4), make sure it goes through the redactor. Add a single test that constructs a client with a real-looking key and asserts the key never appears in any emitted bytes. Also redact `?api-key=...` query strings if present in `BaseURL`.

### G-4. `RegisterIfAbsent` for ask_user happens AFTER `WrapToolRegistry`?
Looking at `setup.New` lines 90-100, `RegisterIfAbsent` is called on `toolReg` (the underlying `*tool.Registry`) BEFORE `WrapToolRegistry` runs. Good — the decorator wraps a registry that already includes ask_user. The debug decorator will see ask_user calls. Confirmed OK; document this ordering invariant in the design so a future refactor doesn't accidentally invert it.

### G-5. Concurrency / record ordering
Multiple agents may run in parallel under the DAG (`MaxConcurrency` > 1). The `Sink.Emit` must be protected by a mutex (design says "thread-safe via mutex" — good) AND each record's correlation id must be sufficient to pair request/response across interleaving. Use a monotonic counter or UUID, not a timestamp. The design says "NewCorrelationID()" — specify it returns a UUIDv4 or `atomic.Uint64`.

### G-6. Streaming error path
Design §6.1 has `TestDebugOn_LLMError` for non-streaming. Add `TestDebugOn_StreamError` for streaming (Recv returns non-EOF error mid-stream). The consolidated record must still be emitted with whatever was accumulated plus the error.

### G-7. `Close()` without draining
If the consumer calls `stream.Close()` before EOF (early cancel, ctx cancel), the consolidation logic must still emit a record. The cleanest hook is in the wrapper's `Close` path, not only on EOF in `Recv`. Both paths must emit at most once (use a `sync.Once`).

### G-8. Tool result truncation timing
R-8 says debug shows what the agent saw, i.e., post-truncation. The design correctly inserts the debug decorator INSIDE (closer to the agent than) the truncating wrap when it says "between truncation and permission wrapper". But re-reading §3 carefully: "WrapToolRegistry chains: truncate -> debugtool -> permission -> base". Truncate is outermost (closest to agent), debug is between truncate and permission. So `Execute` flows: agent → truncate → debug → permission → base. Good — debug sees the result AFTER permission/base have produced it but BEFORE truncate clips it. **That contradicts R-8** ("exactly what the agent saw"). To honor R-8 the debug decorator must be OUTSIDE the truncating wrap (closer to the agent). Fix the chain to: agent → debug → truncate → permission → base. Update §3 ASCII diagram and §6 step 6 accordingly.

### G-9. Default-OFF byte-identical guarantee for HTTP
Test `TestHTTP_DebugOn_LogHasRecords` asserts byte-identical response bodies between debug on/off. With a stub LLM and deterministic tools this is achievable, but only if no debug code path mutates ctx in a way that's observable to handlers. Adding `WithRequestID` to ctx for ALL requests (even when debug off) is fine (ctx mutation isn't observable in body). Adding it ONLY when debug on is also fine. Pick "always add" so the http middleware is unconditional and the only conditional is the sink emission. Simpler.

### G-10. `vv/debugs` import from `vage/largemodel`
Design correctly defines a `DebugSink` interface inside `largemodel` to avoid an import cycle. Good. Make sure the interface has zero `vv`-specific types (the design's `map[string]any` is fine). Also add a no-op default sink so test code in `largemodel` doesn't need to import anything.

### G-11. Help text output channel
`flag.Usage` writes to `flag.CommandLine.Output()`, which defaults to stderr. Confirm `-h` output documents `--debug`, `VV_DEBUG`, and the destination differences (TUI: file under ~/.vv; -p: stderr; HTTP: server log).

## Minor

- M-1: `KindLLMResponse` and `KindLLMStreamDone` are redundant — pick one and use a `Streamed bool` field. Reduces consumer branching.
- M-2: `Record.Params map[string]any` — prefer typed fields (`Temperature *float64`, `MaxTokens *int`) for the 3-4 known knobs and a `map[string]any` for the rest, to avoid stringification surprises.
- M-3: `Record` struct is large; pass by pointer to `Sink.Emit`.
- M-4: Add `VV_DEBUG_FILE=path` env to override the auto file path (interactive mode), useful for ops who want a known location.
- M-5: Document that `make build` (vage then vv) is required after the `aimodel` constructor change (C-2) because of local replace directives.
- M-6: Doc test: assert `vv --help` output contains the literal substring `--debug` and `VV_DEBUG`.
- M-7: Ensure `slog.Default()` in HTTP mode uses a handler that does not add color/ansi to log output (so log files are clean).

## Summary of required design changes

1. **Interactive TUI sink** — change destination from stderr to a file (or slog with file handler) when stdout is a TTY and TUI is active. Keep stderr only for `-p` mode and HTTP server log for HTTP mode.
2. **Stream interception** — needs a small additive change in `aimodel` (`InterceptStream` or exported constructor). Document and include this as a sub-task.
3. **HTTP request-id middleware** — add it explicitly; not optional.
4. **Decorator order** — debug must be OUTSIDE the truncating wrap to honor R-8 ("what the agent saw"). Update the chain order.
5. **Agent name propagation** — promote from "best-effort" to a required one-line edit at each dispatch site.
6. **Explorer registry wrap** — explicitly wrap explorer's registry with the debug decorator.
7. **Compactor coverage test** — explicit test case.
8. **Streaming early-close + error tests** — explicit test cases with `sync.Once` semantics.

These are all small, contained changes and preserve the "default OFF zero side effects" rule and the "byte-identical HTTP response" rule.
