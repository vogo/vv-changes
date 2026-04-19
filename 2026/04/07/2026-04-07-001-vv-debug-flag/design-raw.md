# Technical Design: vv `--debug` Flag

## 1. Goals & Constraints (recap)

- New `--debug` CLI flag, `VV_DEBUG` env, and `debug: bool` YAML field. Precedence: CLI > env > YAML > default(false).
- Default OFF, zero behavioral side effects when off.
- Capture full LLM I/O (system prompt, message history, tool schemas, params, response content, tool_calls, finish reason, token usage, latency) and full tool I/O (args, result, error, latency, source, agent).
- Works in CLI mode (write to stderr) and HTTP mode (write to server log).
- Streaming LLM responses are consolidated and emitted on stream completion.
- Secret redaction for known config fields; correlation IDs link request → LLM calls → tool calls.
- Uniform across all agents (coder, researcher, reviewer, chat, explorer, planner, dynamic sub-agents) and all tool sources (built-in, MCP, agent-as-tool, bash).
- Implementation MUST be non-invasive: no edits to individual agents. Insertion points are the existing `vage/largemodel` middleware chain (LLM I/O) and a new tool-registry decorator (tool I/O).

## 2. Architecture Overview

```
                +-----------------------------------------------+
                |                vv main / setup                |
                |                                               |
   --debug ---->|  cfg.Debug = resolve(flag, env, yaml)         |
                |       |                                       |
                |       v                                       |
                |   debugsink.New(writer)                       |
                |     - writer = os.Stderr (cli) or             |
                |                slog server log (http)         |
                |     - secret-redaction                        |
                |     - correlation-id helpers                  |
                |       |                                       |
                |       +--> setup.Init(cfg, opts, debugSink)   |
                |              |                                |
                |              |  if cfg.Debug:                 |
                |              |    llmClient =                 |
                |              |      largemodel.Chain(         |
                |              |        rawClient,              |
                |              |        debugmw.New(sink) )     |
                |              |                                |
                |              |    WrapToolRegistry chains:    |
                |              |      truncate -> debugtool ->  |
                |              |        permission -> base      |
                |              v                                |
                |         Dispatcher / sub-agents (UNCHANGED)   |
                +-----------------------------------------------+

   request ctx ── correlation_id ──> LLM debug record ──> sink
                                  └─> tool debug record ─> sink

   CLI mode:  sink writes to os.Stderr
   HTTP mode: sink writes via slog (server logger) tagged with http.request_id
```

Insertion points:
- **LLM I/O**: a new `vage/largemodel/debug.go` middleware (`DebugMiddleware`) added to the chain in `setup.Init` only when `cfg.Debug` is true. Captures `ChatRequest` / `ChatResponse` / streaming and emits one record per call (with start/done correlation).
- **Tool I/O**: a new `vv/debugs/toolregistry.go` decorator implementing `tool.ToolRegistry`. Wraps `Execute()` and emits one record per invocation. Inserted in the wrapping chain in `setup.New` (and the explorer wrap), so it works for built-in, MCP, agent-as-tool, and bash uniformly (they all flow through the registry).
- Both insertions happen behind a single `if cfg.Debug` check in `setup.go`, so when the flag is off the chain and decorator are absent and the binary is byte-identical to today.

## 3. Component Design

### 3.1 New files

| File | Purpose |
|------|---------|
| `vv/debugs/sink.go` | `Sink` struct: holds a `io.Writer` (CLI: stderr) or `*slog.Logger` (HTTP), provides `Emit(record)` and `NewCorrelationID()`. Plain-text formatter, one logical record per line group. Thread-safe via mutex. |
| `vv/debugs/redact.go` | Helpers to scrub known secret-bearing config fields (`api_key`, `Authorization`, env vars listed below). Used only on the LLM client config snapshot, NOT on prompt content. |
| `vv/debugs/context.go` | `WithCorrelationID(ctx, id)` / `CorrelationIDFromContext(ctx)` helpers; `WithRequestID(ctx, id)` for HTTP request id. Both consumed by `debugmw` and `debugtool`. |
| `vage/largemodel/debug.go` | `DebugMiddleware` implementing `largemodel.Middleware`. On `ChatCompletion`: emits a `request` record before calling `next` and a `response` record after. On `ChatCompletionStream`: wraps the returned `*aimodel.Stream` to accumulate deltas and emits a single consolidated record on stream end (or error). All emission goes through a `Sink`-shaped interface (defined in `largemodel` to avoid `vv` import). |
| `vage/largemodel/debug_test.go` | Unit tests for capture, streaming consolidation, error path. |
| `vv/debugs/toolregistry.go` | `DebuggingToolRegistry` decorator implementing `tool.ToolRegistry`. Wraps `Execute()`: records start (with agent name from ctx if available, args), invokes inner, records end (result string, error, latency). Delegates all other methods. |
| `vv/debugs/toolregistry_test.go` | Unit tests including success, error, MCP-style external caller passthrough. |
| `vv/debugs/sink_test.go` | Unit tests for redaction and formatting. |

### 3.2 Edited files

| File | Change |
|------|--------|
| `vv/main.go` | Add `--debug` flag (`flag.Bool`). After loading config, apply precedence: if `--debug` was visited use that; else if `VV_DEBUG` set in env use that; else keep `cfg.Debug` from YAML. Update `flag.Usage` to document `--debug` and `VV_DEBUG`, and the stderr/server-log destinations. Construct a `debugs.Sink` (stderr for CLI, slog-backed for HTTP) when `cfg.Debug`, pass into `setup.Init` via `setup.Options.DebugSink`. |
| `vv/configs/config.go` | Add `Debug bool \`yaml:"debug"\`` to `Config`. Read `VV_DEBUG` env var in `Load()` (parse with `strconv.ParseBool`, log warning on parse error, leave default). No change to `applyDefaults` (zero-value false is correct default). |
| `vv/configs/config_test.go` | Add tests for env override, YAML parse, default. |
| `vv/setup/setup.go` | Extend `Options` with `DebugSink *debugs.Sink`. In `Init`: if `cfg.Debug && opts.DebugSink != nil`, wrap `llmClient` via `largemodel.Chain(rawClient, debugmw.New(sinkAdapter))` before passing to `New`. In `New`: if debug sink present, insert `debugs.NewDebuggingToolRegistry(sink)` into the per-agent registry wrapping chain (between truncation and permission wrapper) for both dispatchable agents and the explorer. |
| `vv/cli/prompt.go` and `vv/cli/app.go` (or wherever bubbletea is initialized) | No code change required: bubbletea owns stdout; debug records go to `os.Stderr` which bubbletea does not capture. Add a doc comment confirming this contract. |
| `vv/httpapis/serve.go` (or main HTTP entrypoint) | When constructing the server slog logger, pass it into the debug sink so each record carries the HTTP request id from context (extract from existing request-id middleware; if none, add a tiny one that puts a UUID into ctx via `debugs.WithRequestID`). |

### 3.3 Why these insertion points

- `largemodel` middleware is the single chokepoint for every LLM call from every agent, including dispatcher's own intent/planner/summarizer calls. One wrap covers all six pre-built agents and any dynamic sub-agent that uses the same `llmClient` (which they all do — see `factoryOpts.LLM`).
- The tool registry decorator covers every tool source uniformly because vage routes all tool execution (local handler, MCP `ExternalToolCaller`, agent-as-tool which is registered as a normal tool, and bash) through `Registry.Execute`. Wrapping the registry catches all of them.
- No edits to `taskagent`'s ReAct loop, no edits to any agent factory, no edits to MCP code.

## 4. Data Models / Schemas

### 4.1 Configuration extension

```go
// vv/configs/config.go
type Config struct {
    // ... existing fields ...
    Debug bool `yaml:"debug"` // when true, emit detailed LLM and tool I/O. CLI > env (VV_DEBUG) > YAML > false.
}
```

### 4.2 Debug record structs (internal, in `vv/debugs`)

```go
type Kind string
const (
    KindLLMRequest    Kind = "llm.request"
    KindLLMResponse   Kind = "llm.response"
    KindLLMStreamDone Kind = "llm.stream.done"
    KindLLMError      Kind = "llm.error"
    KindToolStart     Kind = "tool.start"
    KindToolEnd       Kind = "tool.end"
)

type Record struct {
    Kind          Kind
    CorrelationID string        // ties llm.request <-> llm.response, tool.start <-> tool.end
    HTTPRequestID string        // empty in CLI mode
    AgentName     string        // best-effort; from ctx if present
    Timestamp     time.Time
    Duration      time.Duration // populated on .response/.end records

    // LLM-specific
    Model       string
    Params      map[string]any  // temperature, max_tokens, top_p, etc.
    Messages    []MessageView   // role, content, tool_calls, tool_call_id
    Tools       []ToolSchemaView
    Content     string          // assistant content (response)
    ToolCalls   []ToolCallView  // tool_calls in response
    FinishReason string
    Usage       *UsageView      // prompt, completion, total, cache_read

    // Tool-specific
    ToolName    string
    ToolSource  string          // "builtin" | "mcp" | "agent" | "bash"
    Args        string          // raw JSON
    Result      string          // exactly what the agent saw (post-truncation)
    ReadOnly    bool

    Err string                  // populated on error/end records
}
```

### 4.3 Sink interface (abstracted in `largemodel` to avoid import cycle)

```go
// vage/largemodel/debug.go
type DebugSink interface {
    Emit(ctx context.Context, kind, correlationID string, fields map[string]any)
    NewCorrelationID() string
}
```

`vv/debugs.Sink` satisfies this interface; flat fields keep `largemodel` package free of vv-specific types.

### 4.4 Redacted fields (config snapshot only)

`api_key`, `Authorization`, env vars `VV_LLM_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `AI_API_KEY`. Replaced with `"<redacted>"`. Prompt and tool content are NOT auto-redacted (per requirement R-7).

## 5. Implementation Plan (small ordered tasks)

1. **configs**: add `Debug` field, env override (`VV_DEBUG`), unit tests. (`vv/configs/config.go`, `_test.go`)
2. **debugs package skeleton**: create `vv/debugs/{sink.go,redact.go,context.go}` with `Sink`, `Record`, formatter, redaction helpers, ctx helpers. Unit tests for redaction and formatting.
3. **largemodel debug middleware**: create `vage/largemodel/debug.go` with `DebugSink` interface and `DebugMiddleware`. Implement non-streaming path first; emit request + response records keyed by a correlation id. Unit tests with a fake sink and a fake `aimodel.ChatCompleter`.
4. **largemodel debug streaming**: extend middleware to wrap `*aimodel.Stream`, accumulate deltas, emit single consolidated `llm.stream.done` record on EOF or error. Unit tests using a fake stream.
5. **debugging tool registry**: create `vv/debugs/toolregistry.go` decorator. Implement `Execute` instrumentation, delegate the rest. Detect tool source by querying inner registry's def metadata where possible (fallback: classify by name list — bash/read/write/edit/glob/grep/ask_user → builtin; otherwise mcp/agent). Unit tests for success, error, no-op when sink nil.
6. **setup wiring**: add `DebugSink` to `setup.Options`. In `Init`, when `cfg.Debug` and sink present, wrap `llmClient` with `largemodel.Chain(llmClient, largemodel.NewDebugMiddleware(sink))` before constructing `compactor` and `New`. In `New`, when sink present, insert `debugs.NewDebuggingToolRegistry(finalToolReg, sink)` after the truncating wrap (so the debug record reflects what the agent actually saw). Apply the same wrap to the explorer registry.
7. **main.go wiring**: add `--debug` flag with `flag.Bool`. Resolve precedence (CLI > env > YAML). Construct `debugs.NewSinkStderr()` for CLI mode and `debugs.NewSinkSlog(slog.Default())` for HTTP mode. Pass via `setup.Options.DebugSink`. Update `flag.Usage` text.
8. **HTTP request id propagation**: in `vv/httpapis`, ensure each handler puts an http request id (already in slog or generate UUID) into ctx via `debugs.WithRequestID`. Sink reads it from ctx when emitting.
9. **CLI verification**: confirm bubbletea init does not redirect stderr; add a doc comment in `vv/cli/app.go` noting the contract. No code change unless redirection is found.
10. **Help text & docs**: update `flag.Usage` strings. Add a brief section to `vv/CLAUDE.md` (only if user asks; default no doc churn) — skip for MVP.
11. **Integration tests**: see §6.
12. **Final pass**: run `make build` in `vage/` then `vv/`. Verify no behavioral change with debug off (existing tests pass), and verify records appear with debug on.

## 6. Integration Test Plan

Place under `vv/integrations/debug_tests/` (gated on `VV_LLM_API_KEY`, mirroring existing pattern; for fully offline tests use a stub `aimodel.ChatCompleter` in unit tests instead).

### 6.1 Offline (unit/integration without LLM key)

- **TestDebugOff_NoOutput**: Construct dispatcher via `setup.New` with a stub LLM and stub tool, `cfg.Debug=false`. Run a simple message that triggers one LLM call and one tool call. Assert sink receives zero records (or sink not constructed).
- **TestDebugOn_LLMRecords**: Same, with `cfg.Debug=true` and an in-memory sink. Assert exactly one `llm.request` and one `llm.response` record per LLM call, sharing a correlation id, and that the request record contains the full message slice and tool schemas while the response contains content/finish_reason/usage.
- **TestDebugOn_ToolRecords**: Trigger a tool call via the stub LLM (returning a tool_call). Assert one `tool.start` and one `tool.end` record sharing a correlation id, with args and post-truncation result captured.
- **TestDebugOn_StreamingConsolidated**: Use a fake stream that emits 5 deltas + finish. Assert exactly one consolidated `llm.stream.done` record with the concatenated content and final usage.
- **TestDebugOn_LLMError**: Stub LLM returns an error. Assert one `llm.error` record correlated to the request record.
- **TestDebugOn_ToolError**: Tool handler returns error. Assert `tool.end` record with `Err` populated.
- **TestRedaction_ConfigSnapshot**: When middleware logs LLM client config (base url, model, provider), assert the API key is not present in the emitted record bytes.

### 6.2 CLI mode

- **TestCLI_DebugOff_StderrEmpty**: `vv -p "echo hi"` with stub backend; capture stderr; assert no debug records emitted (existing diagnostics still allowed but no `llm.*`/`tool.*` lines).
- **TestCLI_DebugOn_StderrHasRecords**: `vv --debug -p "list files"`; capture stderr; assert presence of `llm.request`, `llm.response`, and at least one `tool.start`/`tool.end` line. Capture stdout; assert it remains the user-facing output unchanged from the non-debug run (modulo LLM nondeterminism — use stub).
- **TestCLI_TUINotCorrupted**: Smoke test that bubbletea TUI initialization succeeds with `--debug`; assert stdout is not contaminated with debug record prefixes.

### 6.3 HTTP mode

- **TestHTTP_DebugOff_LogClean**: Start server with `cfg.Debug=false`, perform sync request, capture server slog output, assert no `llm.*`/`tool.*` records.
- **TestHTTP_DebugOn_LogHasRecords**: Start server with `--debug`, perform sync request; assert log contains correlated llm and tool records each tagged with the http request id. Compare HTTP response body bytes against the same request run with debug off (using deterministic stub LLM); assert byte-identical.
- **TestHTTP_StreamingResponse**: Streaming endpoint returns same SSE stream bytes with `--debug` on vs off (stub LLM); server log contains exactly one consolidated stream record per LLM call.
- **TestHTTP_AsyncTask**: Async endpoint records carry the async task id in addition to (or instead of) the http request id.

### 6.4 Coverage matrix

| Scenario | CLI off | CLI on | HTTP off | HTTP on |
|---|---|---|---|---|
| LLM call records | none | yes | none | yes |
| Tool call records | none | yes | none | yes |
| Streaming consolidation | n/a | yes | n/a | yes |
| Stdout/response unchanged | baseline | yes | baseline | yes |
| Secret redaction | n/a | yes | n/a | yes |

## 7. Risks & Notes

- **Performance when off**: zero — middleware and decorator are not constructed at all. No conditional checks in hot paths.
- **Performance when on**: one extra map allocation and one stringification per LLM/tool call; acceptable for a debug mode.
- **MCP `ExternalToolCaller`**: still funnels through `Registry.Execute`, so the decorator catches it. The "source" classification is heuristic (by tool def metadata or name), which is fine for human-readable output.
- **Agent name in ctx**: vage doesn't currently propagate agent name into context. Best-effort: `debugs.WithAgentName(ctx, name)` is set by the dispatcher for dispatched sub-agents (one small edit in `vv/dispatches/execute.go`); otherwise the field is left empty. This is non-blocking.
- **Out of scope**: structured JSON output, runtime toggle, per-agent filtering, log rotation — explicitly deferred.

---

## Summary

Design adds a single `Debug bool` to `Config` with CLI > env > YAML precedence, and inserts debug capture at exactly two existing chokepoints: a new `largemodel.DebugMiddleware` (for all LLM I/O across every agent and the dispatcher's own LLM calls) and a new `vv/debugs.DebuggingToolRegistry` decorator wrapping the per-agent tool registry (for all tool I/O regardless of source — built-in, MCP, agent-as-tool, bash). Records share correlation ids, are routed to stderr in CLI mode and to the server slog in HTTP mode, are byte-zero-cost when the flag is off, and require no edits to any agent or to vage's ReAct loop. Streaming is consolidated into a single record at stream completion. Secret redaction is applied only to LLM client config snapshots; user-provided prompt/tool content is left intact per requirement.
