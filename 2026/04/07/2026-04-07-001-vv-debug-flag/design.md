# Technical Design: vv `--debug` Flag (revised)

> Supersedes `design-raw.md`. Incorporates `design-review.md`.

## 1. Goals & Constraints

- New `--debug` CLI flag, `VV_DEBUG` env, and `debug: bool` YAML field. Precedence: CLI > env > YAML > default(false).
- **Default OFF, zero behavioral side effects when off.** When off, no middleware is added, no decorator is added, no extra goroutine is created, no log file is opened.
- Capture full LLM I/O (system prompt, message history, tool schemas, params, response content, tool_calls, finish reason, token usage, latency) and full tool I/O (args, result, error, latency, source, agent).
- Works in three run modes:
  - **CLI interactive (Bubble Tea TUI)** — debug records go to a file (`~/.vv/debug-<pid>.log` or `$VV_DEBUG_FILE`); a one-line banner on the status bar shows the path. **Never written to the tty** because Bubble Tea owns the terminal device and any stderr write would corrupt the TUI.
  - **CLI non-interactive (`-p`)** — debug records go to `os.Stderr`, alongside the existing `[tool]/[phase]/[agent]` status lines.
  - **HTTP service** — debug records go to `slog.Default()` (server log), tagged with the http request id (and async task id where applicable).
- Streaming LLM responses are consolidated into a single record at stream completion (or early close / error).
- Secret redaction for known config fields; correlation IDs link request → response and tool start → end.
- Uniform across all agents (coder, researcher, reviewer, chat, explorer, planner, plan-gen, compactor, dispatcher's intent call, dynamic sub-agents) and all tool sources (built-in, MCP, agent-as-tool, bash, ask_user).
- Implementation MUST be non-invasive: no edits to individual agent factories, no edits to vage's ReAct loop. Insertion points are the existing `vage/largemodel` middleware chain (LLM I/O) and a new tool-registry decorator (tool I/O), plus a small additive helper in `aimodel` for stream interception.

## 2. Architecture Overview

```
                +--------------------------------------------------+
                |                vv main / setup                   |
                |                                                  |
   --debug ---->|  cfg.Debug = resolve(flag, env, yaml)            |
                |       |                                          |
                |       v                                          |
                |   sink := debugs.NewSink(mode)                   |
                |     - mode=cli-tui  -> file ~/.vv/debug-<pid>.log|
                |     - mode=cli-prompt -> os.Stderr               |
                |     - mode=http     -> slog.Default()            |
                |       |                                          |
                |       v                                          |
                |   setup.Init(cfg, opts{DebugSink:sink})          |
                |       |                                          |
                |       |  if cfg.Debug && sink != nil:            |
                |       |    llmClient = largemodel.Chain(         |
                |       |        rawClient,                        |
                |       |        debugmw.New(sinkAdapter))         |
                |       |                                          |
                |       |    setup.New wraps each per-agent        |
                |       |    registry chain as:                    |
                |       |      agent -> debug -> truncate ->       |
                |       |               permission -> base         |
                |       |    (debug OUTSIDE truncate so it sees    |
                |       |     what the agent actually got, R-8)    |
                |       v                                          |
                |  Dispatcher / sub-agents (UNCHANGED)             |
                +--------------------------------------------------+

   request ctx ── correlation_id, agent_name, http_request_id
                │
                ├──> LLM debug record  ──> sink
                └──> Tool debug record ──> sink
```

Two insertion points + small support changes:

1. **LLM I/O** — `vage/largemodel/debug.go` defines `DebugSink` interface and `DebugMiddleware`. Wrapped onto `llmClient` in `setup.Init` BEFORE the compactor closure is created and BEFORE `setup.New`, so it covers: every agent's LLM calls, the compactor's summarization call, the dispatcher's intent and plan-gen calls, and any future direct `llmClient` consumer.
2. **Tool I/O** — `vv/debugs/toolregistry.go` decorator implementing `tool.ToolRegistry`. Inserted in the per-agent registry chain in `setup.New`, **outside** `NewTruncatingToolRegistry`, so the debug record reflects the post-truncation result the agent actually consumed.
3. **Streaming** — `aimodel.InterceptStream(s, onChunk, onDone)` (small additive helper, see §3.4) lets the LLM middleware accumulate deltas without forking the stream consumer.
4. **HTTP request id** — small `func(http.Handler) http.Handler` middleware in `vv/httpapis/http.go` that puts a UUID into ctx via `debugs.WithRequestID`. Always installed (not gated on debug flag) to keep the conditional surface minimal and to avoid any chance of body drift between debug on/off.
5. **Agent name in ctx** — one-line `ctx = debugs.WithAgentName(ctx, id)` at each dispatch site in `vv/dispatches/execute.go` (direct dispatch + DAG step dispatch + explorer call + planner call + intent call + plan-gen call). Cheap, always done — when debug is off, the helpers compile to a single context-with-value.

When `cfg.Debug` is false: middleware is not constructed, decorator is not constructed, sink is `nil`. The only residual cost vs. today is the always-on http request-id middleware (one UUID alloc per request) and the agent-name ctx values (one ctx alloc per dispatch). Both negligible and behavior-neutral.

## 3. Component Design

### 3.1 New files

| File | Purpose |
|------|---------|
| `vv/debugs/sink.go` | `Sink` struct with pluggable backend: `fileBackend` (TUI), `writerBackend` (stderr for `-p`), `slogBackend` (HTTP). Provides `Emit(ctx, *Record)` and `NewCorrelationID()`. Mutex-guarded so concurrent agents/tools don't interleave records. Backend is chosen by constructor: `NewFileSink(path)`, `NewWriterSink(io.Writer)`, `NewSlogSink(*slog.Logger)`. |
| `vv/debugs/format.go` | Plain-text formatter. One header line per record (kind, correlation id, agent, duration, http req id), then indented multi-line content. Streaming and non-streaming responses use the same `kind=llm.response` (with a `streamed=true` field) — see review M-1. |
| `vv/debugs/redact.go` | Helpers to scrub known secret-bearing config fields (`api_key`, `Authorization`, query string `api-key=`, env vars listed in §4.4). Used only on the LLM client config snapshot. Prompt and tool content are NOT auto-redacted (per R-7). |
| `vv/debugs/context.go` | `WithCorrelationID/CorrelationIDFromContext`, `WithRequestID/RequestIDFromContext`, `WithAgentName/AgentNameFromContext`. |
| `vv/debugs/toolregistry.go` | `DebuggingToolRegistry` decorator implementing `tool.ToolRegistry`. Wraps `Execute(ctx, name, args)`: emits `tool.start` (correlation id, agent name from ctx, tool name, args, source classification, read-only flag), invokes inner, emits `tool.end` (result string, error, latency). All other methods delegate. |
| `vv/debugs/{*_test.go}` | Unit tests: redaction, formatting, decorator success/error, no-op when sink nil. |
| `vage/largemodel/debug.go` | `DebugSink` interface (zero vv-specific types). `DebugMiddleware` implementing `Middleware`. Non-streaming path: emits request before `next.ChatCompletion`, response (or error) after. Streaming path: calls `next.ChatCompletionStream`, then `aimodel.InterceptStream(s, onChunk, onDone)` to accumulate; emits one consolidated `llm.response` (`streamed=true`) record on `onDone`, gated by `sync.Once` so early `Close()` and EOF cannot double-emit. |
| `vage/largemodel/debug_test.go` | Capture, streaming consolidation (EOF, early close, mid-stream error), error path, no-op when sink nil. |
| `aimodel/intercept.go` | `InterceptStream(s *Stream, onChunk func(*StreamChunk), onDone func(err error)) *Stream` — internally swaps `s.recv` to a wrapping closure that calls `onChunk` for each non-nil chunk and `onDone` exactly once on first non-nil error. Also called from `Close()` path via `WrapStream`-style `onClose` to guarantee `onDone` fires on early close. Additive, no API break. |
| `aimodel/intercept_test.go` | Tests for chunk interception, EOF, early close, error mid-stream, double-close idempotency. |

### 3.2 Edited files

| File | Change |
|------|--------|
| `vv/main.go` | Add `--debug` (`flag.Bool`). Resolve precedence (CLI > env > YAML). Decide sink mode based on resolved run mode and `term.IsTerminal(int(os.Stdout.Fd()))`: HTTP → `NewSlogSink(slog.Default())`; CLI `-p` → `NewWriterSink(os.Stderr)`; CLI interactive (no `-p`) → `NewFileSink(resolvePath())` and print a one-line banner via the bubbletea status bar (or stderr before TUI starts). Update `flag.Usage` text (see §3.5). Pass sink via `setup.Options.DebugSink`. |
| `vv/configs/config.go` | Add `Debug bool \`yaml:"debug"\`` to `Config`. Read `VV_DEBUG` env in `Load()` (`strconv.ParseBool`, warn on parse error). Default false. |
| `vv/configs/config_test.go` | Unit tests: env override, YAML parse, default. |
| `vv/setup/setup.go` | Extend `Options` with `DebugSink debugs.Sink` (interface or pointer). In `Init`, when `cfg.Debug && opts.DebugSink != nil`, wrap `llmClient` with `largemodel.Chain(llmClient, largemodel.NewDebugMiddleware(adapter(opts.DebugSink)))` BEFORE constructing the compactor closure and BEFORE calling `New`, so compactor calls are covered. In `New`, when sink present, insert `debugs.NewDebuggingToolRegistry(finalToolReg, sink)` AFTER `NewTruncatingToolRegistry` (i.e., wraps the truncating wrap; debug is the OUTERMOST tool layer the agent sees — see review G-8, R-8). Apply the same outer wrap to the explorer registry (review G-2). |
| `vv/dispatches/execute.go` and `vv/dispatches/intent.go` | One-line `ctx = debugs.WithAgentName(ctx, id)` at each agent dispatch site: direct dispatch, each DAG step, explorer invocation, planner invocation, intent LLM call, plan-gen call. Helpers degrade to plain `context.WithValue` and add no behavior. |
| `vv/httpapis/http.go` | Wrap the mux/handler with a small middleware that generates a UUID and stores it via `debugs.WithRequestID`. Always installed; not gated on debug. Also pass `slog.Default()` into the sink constructor. |
| `vv/cli/app.go` (or wherever bubbletea is initialized) | Add a doc comment: "do NOT write to os.Stderr while the TUI is active; route diagnostics through the sink which targets a file in interactive mode." No code change to the TUI itself. |

### 3.3 Why these insertion points

- `largemodel.Middleware` is the single chokepoint for every LLM call from every agent (all six pre-built agents + dispatcher's intent/plan-gen + compactor + any dynamic sub-agent — they all share `factoryOpts.LLM`, which is the wrapped client).
- `tool.ToolRegistry.Execute` is the single chokepoint for every tool call regardless of source: built-in handlers, MCP `ExternalToolCaller`, agent-as-tool (registered as a normal tool), bash, ask_user. Wrapping the registry catches all of them with one decorator.
- The aimodel `InterceptStream` helper is the only invasive change but is purely additive and tiny.

### 3.4 `aimodel.InterceptStream` (additive helper)

```go
// aimodel/intercept.go
package aimodel

import "sync"

// InterceptStream installs onChunk and onDone callbacks on s without changing
// the consumer-visible Stream API. onChunk fires for every non-nil chunk
// returned by Recv. onDone fires exactly once: on the first non-nil error
// (including io.EOF) returned by Recv, or on Close, whichever comes first.
//
// InterceptStream is additive: callers using Recv/Close see no behavior
// change. The callbacks must be cheap and must not call Recv/Close.
func InterceptStream(s *Stream, onChunk func(*StreamChunk), onDone func(err error)) *Stream {
    if s == nil {
        if onDone != nil { onDone(nil) }
        return nil
    }
    var once sync.Once
    fire := func(err error) { once.Do(func() { if onDone != nil { onDone(err) } }) }

    inner := s.recv
    s.recv = func() (*StreamChunk, error) {
        chunk, err := inner()
        if chunk != nil && onChunk != nil { onChunk(chunk) }
        if err != nil { fire(err) }
        return chunk, err
    }
    prevOnClose := s.onClose
    s.onClose = func(u *Usage) {
        if prevOnClose != nil { prevOnClose(u) }
        fire(nil)
    }
    return s
}
```

This requires `recv` and `onClose` to be package-internal but they already are. Add a test for: normal EOF, mid-stream error, early `Close()` before EOF, double-close idempotency.

### 3.5 `--help` text

```
  --debug
        Enable detailed LLM and tool I/O debug records. Default: false.
        Equivalent to environment variable VV_DEBUG=true.
        Output destination by mode:
          interactive CLI : ~/.vv/debug-<pid>.log (override with VV_DEBUG_FILE)
          -p (non-interactive CLI) : stderr
          HTTP server     : slog server log
        Does not change HTTP response bodies or TUI output.
```

## 4. Data Models

### 4.1 Configuration

```go
// vv/configs/config.go
type Config struct {
    // ... existing fields ...
    Debug bool `yaml:"debug"` // CLI > env (VV_DEBUG) > YAML > false
}
```

### 4.2 Debug record

```go
// vv/debugs/record.go
type Kind string
const (
    KindLLMRequest Kind = "llm.request"
    KindLLMResponse Kind = "llm.response" // streamed and non-streamed share this kind
    KindLLMError   Kind = "llm.error"
    KindToolStart  Kind = "tool.start"
    KindToolEnd    Kind = "tool.end"
)

type Record struct {
    Kind          Kind
    CorrelationID string
    HTTPRequestID string        // empty in CLI mode
    AgentName     string        // from ctx
    Timestamp     time.Time
    Duration      time.Duration // populated on response/end records

    // LLM fields
    Model        string
    Temperature  *float64
    MaxTokens    *int
    TopP         *float64
    OtherParams  map[string]any
    Messages     []MessageView
    Tools        []ToolSchemaView
    Streamed     bool
    Content      string
    ToolCalls    []ToolCallView
    FinishReason string
    Usage        *UsageView // prompt/completion/total/cache_read

    // Tool fields
    ToolName   string
    ToolSource string // "builtin" | "mcp" | "agent" | "bash"
    Args       string
    Result     string
    ReadOnly   bool

    Err string
}
```

Pass by pointer (`*Record`) to `Sink.Emit` to avoid copying the large struct.

### 4.3 `DebugSink` interface (in `vage/largemodel`)

```go
// vage/largemodel/debug.go
type DebugSink interface {
    Emit(ctx context.Context, kind, correlationID string, fields map[string]any)
    NewCorrelationID() string
}
```

`vv/debugs.Sink` implements this via a small adapter that translates the flat `fields` map into a typed `*debugs.Record`. Keeps `vage/largemodel` free of vv-specific types.

A no-op `NoopSink` is provided in `largemodel` so middleware tests don't need vv.

### 4.4 Redacted fields

Config snapshot only: `api_key`, `Authorization` header, `?api-key=` query string, env vars `VV_LLM_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `AI_API_KEY`. Replaced with `<redacted>`. Prompt and tool content are NEVER auto-redacted (per R-7).

### 4.5 Correlation IDs

`NewCorrelationID()` returns a UUIDv4 string. Independent of timestamps so concurrent records cannot collide.

## 5. Implementation Plan

1. **aimodel additive helper** — add `InterceptStream` and tests. Run `make build` in `aimodel/`.
2. **vv configs** — add `Debug` field, `VV_DEBUG` env override, tests.
3. **debugs package skeleton** — `sink.go`, `format.go`, `redact.go`, `context.go`, `record.go`. Three sink backends (file, writer, slog). Unit tests for redaction and formatting.
4. **largemodel debug middleware** — `DebugSink` interface, `NoopSink`, `DebugMiddleware`. Non-streaming path first. Tests with a fake `aimodel.ChatCompleter`.
5. **largemodel streaming** — extend middleware to call `aimodel.InterceptStream`, accumulate deltas (content + tool calls), emit one consolidated `llm.response` record (`Streamed=true`) on `onDone`. Use `sync.Once` (already inside `InterceptStream`). Tests for EOF, early close, mid-stream error.
6. **debugs tool registry decorator** — `Execute` instrumentation, source classification (lookup `Get(name).Source` if available, else heuristic by name list), tests for success, error, no-op.
7. **vv/setup wiring** — `DebugSink` in `Options`. In `Init`, wrap `llmClient` BEFORE compactor closure and BEFORE `New`. In `New`, when sink present, wrap each per-agent `finalToolReg` AFTER the truncating wrap (so debug is outermost, sees post-truncation result the agent gets). Same for explorer registry.
8. **vv/dispatches** — add `ctx = debugs.WithAgentName(ctx, id)` at each dispatch site (intent, planner, plan-gen, direct, DAG step, explorer).
9. **vv/httpapis** — add request-id middleware (always on); construct `slog`-backed sink in main.go for HTTP mode.
10. **vv/main.go** — add `--debug` flag, resolve precedence, choose sink mode based on `(--mode http) ? slog : (-p ? stderr : file)` and `term.IsTerminal`, print one-line banner before TUI starts when file sink is selected.
11. **Help text** — update `flag.Usage`.
12. **Integration tests** — see §6.
13. **Final pass** — `make build` in `aimodel/`, `vage/`, `vv/`. Verify zero behavioral change with debug off (existing tests pass) and verify records appear with debug on.

## 6. Test Plan

### 6.1 Offline unit/integration (no LLM key)

- **TestDebugOff_NoOutput**: stub LLM + stub tool, `cfg.Debug=false`. One LLM call + one tool call. Assert sink not constructed and no records.
- **TestDebugOn_LLMRecords**: in-memory sink. Assert one `llm.request` + one `llm.response` per call sharing correlation id; request contains full messages and tool schemas; response contains content/finish_reason/usage.
- **TestDebugOn_ToolRecords**: stub LLM emits a tool call. Assert one `tool.start` + one `tool.end` sharing correlation id; args and post-truncation result captured.
- **TestDebugOn_TruncationVisible**: tool returns 100KB; truncation set to 1KB. Assert `tool.end.Result` length ≤ 1KB (proves debug sees what agent sees, R-8).
- **TestDebugOn_StreamingConsolidated**: fake stream emits 5 deltas + finish. Assert one `llm.response` with `Streamed=true`, concatenated content, final usage.
- **TestDebugOn_StreamingEarlyClose**: caller calls `Close()` after 2 deltas. Assert exactly one consolidated record with the partial content and `Err=""` (or context.Canceled if applicable).
- **TestDebugOn_StreamingError**: stream returns non-EOF error after 3 deltas. Assert one record with partial content and `Err` populated.
- **TestDebugOn_LLMError**: non-streaming LLM returns error. Assert one `llm.error` correlated to the request.
- **TestDebugOn_ToolError**: tool handler returns error. Assert `tool.end` with `Err` populated.
- **TestDebugOn_CompactorCovered**: trigger conversation compaction. Assert a `llm.request`/`llm.response` pair appears with `agent.name` indicating compactor (or empty + a marker that disambiguates from sub-agent calls).
- **TestDebugOn_DispatcherIntentCovered**: dispatcher intent call appears in records.
- **TestDebugOn_ExplorerToolsCovered**: explorer-triggered tool calls appear (proves explorer registry is wrapped).
- **TestRedaction_NoAPIKeyInBytes**: build a client with a recognizable API key; format any record that includes config snapshot; assert key bytes never appear.
- **TestSink_ConcurrentEmit**: 100 goroutines emit; assert no interleaving (each record's lines are contiguous).
- **TestNewCorrelationID_Unique**: 10k calls; assert all unique.

### 6.2 CLI `-p` mode

- **TestPrompt_DebugOff_StderrUnchanged**: `vv -p "..."` with stub backend; capture stderr; assert it matches the existing baseline (no `llm.*`/`tool.*` lines).
- **TestPrompt_DebugOn_StderrHasRecords**: `vv -p --debug "..."`; capture stderr; assert presence of `llm.request`, `llm.response`, `tool.start`, `tool.end` lines. Capture stdout; assert byte-identical to non-debug run (with deterministic stub LLM).

### 6.3 CLI interactive TUI mode

- **TestTUI_DebugOn_FileSink**: launch TUI under a pty with `--debug`; assert a file at `~/.vv/debug-<pid>.log` (or `$VV_DEBUG_FILE`) exists and contains records after one round; assert tty bytes contain no `llm.*`/`tool.*` text (proves no TUI corruption).
- **TestTUI_DebugOff_NoFile**: no `--debug`; assert no debug file is created.
- **TestTUI_BannerPrinted**: `--debug` interactive; assert the status bar / startup banner mentions the debug log path.

### 6.4 HTTP mode

- **TestHTTP_DebugOff_LogClean**: server with `cfg.Debug=false`; sync request; capture slog; no `llm.*`/`tool.*` records.
- **TestHTTP_DebugOn_LogHasRecords**: server with `--debug`; sync request with deterministic stub; capture slog; assert correlated llm and tool records each tagged with the http request id.
- **TestHTTP_BodyByteIdentical**: same request body, deterministic stub backend, run with debug off and debug on; assert HTTP response bodies byte-identical.
- **TestHTTP_StreamingByteIdentical**: SSE endpoint; same comparison; assert SSE byte stream identical and exactly one consolidated stream record per LLM call in the log.
- **TestHTTP_AsyncTask**: async endpoint; records carry async task id in addition to http request id.

### 6.5 Coverage matrix

| Scenario | CLI-p off | CLI-p on | TUI off | TUI on | HTTP off | HTTP on |
|---|---|---|---|---|---|---|
| LLM records | none | stderr | none | file | none | slog |
| Tool records | none | stderr | none | file | none | slog |
| Streaming consolidation | n/a | yes | n/a | yes | n/a | yes |
| stdout/response unchanged | baseline | yes | baseline | yes | baseline | yes |
| Secret redaction | n/a | yes | n/a | yes | n/a | yes |
| TUI uncorrupted | n/a | n/a | yes | yes | n/a | n/a |

## 7. Risks & Notes

- **Performance off**: zero — middleware/decorator not constructed. Residual: one UUID alloc per http request (always-on request-id middleware) and one ctx.WithValue per agent dispatch. Negligible, behavior-neutral.
- **Performance on**: one map alloc + stringification per LLM/tool call, plus one chunk callback per stream delta. Acceptable for debug mode.
- **MCP `ExternalToolCaller`**: funnels through `Registry.Execute`, caught by the decorator. Source classification is heuristic (by tool name list); good enough for human reading.
- **Agent name propagation**: required edits at dispatch sites in `vv/dispatches/execute.go` and `intent.go`. Single line each. Helpers degrade to plain `context.WithValue`.
- **Decorator order (R-8)**: debug is OUTSIDE truncation so it sees the truncated bytes the agent received. The chain from outermost (closest to agent) to innermost (closest to base) is: `debug → truncate → permission → base`.
- **Streaming**: requires the additive `aimodel.InterceptStream` helper. Without it the middleware cannot intercept deltas while still returning `*aimodel.Stream` to the consumer.
- **Interactive TUI**: writing debug to stderr would corrupt the TUI because Bubble Tea draws on the same tty. Use a file sink in this mode; print a startup banner with the path.
- **HTTP request id**: middleware always installed (not gated on debug) so the only conditional is sink emission. Eliminates any chance of body drift between debug on/off.
- **Out of scope** (deferred): runtime `/debug` toggle, structured JSON output, per-agent filtering, log rotation, redacting user prompt content.

---

## Summary

Adds a single `Debug bool` to `Config` with CLI > env > YAML precedence and inserts debug capture at exactly two existing chokepoints — `largemodel.DebugMiddleware` for all LLM I/O and `vv/debugs.DebuggingToolRegistry` for all tool I/O — plus three small support changes: `aimodel.InterceptStream` (additive helper for stream consolidation), an always-on http request-id middleware, and one-line agent-name ctx tagging at each dispatch site. Sink destination is mode-aware: file for the interactive TUI (so the tty is never written to and the TUI is never corrupted), stderr for `-p` non-interactive mode, slog for HTTP. Decorator order places debug OUTSIDE truncation so records reflect exactly what the agent saw. Streaming uses `sync.Once` to guarantee exactly one consolidated record on EOF, early close, or error. Default OFF means no middleware/decorator constructed, byte-identical HTTP responses, and no TUI corruption. No edits to any agent or to vage's ReAct loop.
