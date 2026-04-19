# Requirement: vv `--debug` Flag

## 1. Background and Objectives

### Background
vv is a CLI/HTTP AI agent application built on the vage framework. In daily use and during troubleshooting, developers and operators frequently need to inspect the exact prompts sent to the LLM and the raw responses received, as well as the arguments passed to tool invocations and the results returned. Today, this information is hidden behind normal user-facing output (streaming text, status bar, tool confirmation dialogs), making it difficult to diagnose unexpected agent behavior, prompt regressions, tool integration bugs, or cost anomalies.

### Objectives
Introduce a top-level `--debug` command-line flag for the `vv` binary that, when enabled, exposes detailed model and tool I/O for the entire run. The flag must:

- Be available in both run modes (CLI interactive and HTTP service).
- Not alter default (non-debug) output, behavior, or performance in any noticeable way.
- Apply uniformly to all agents (orchestrator, coder, researcher, reviewer, chat, and any dynamically created sub-agents).
- Work consistently with existing features (permission modes, streaming, context compression, cost tracking).

## 2. Assumptions

Because this is a non-interactive analysis run, the following assumptions are made and must be validated by design:

1. The flag is a process-level switch passed at startup (`vv --debug ...`); it cannot be toggled at runtime via a `/debug` command in the MVP. (A runtime toggle may be a future enhancement.)
2. Debug output is written to stderr in CLI mode so it does not interleave with the bubbletea-rendered TUI on stdout, and to the server log stream in HTTP mode. The HTTP API response body itself is NOT changed in debug mode (no extra fields, no payload bloat).
3. Debug output is plain text / structured log lines (one logical record per LLM call or tool call), not a new wire-protocol surface. It is intended for human inspection and ad-hoc grep, not for programmatic consumption.
4. Sensitive material (API keys, bearer tokens, raw env values) is never written to debug output. Prompts and tool I/O may contain user content; users are responsible for the trust boundary of their terminal/log sink, but vv must redact known secret-bearing config fields.
5. "Full prompts" means the complete request payload sent to the LLM at the application boundary: system prompt, message history (after compression/truncation as actually sent), tool schemas, model parameters. "Full responses" means the assistant message content, tool calls, finish reason, and token usage block.
6. "Tool input/output" means the JSON arguments dispatched to each tool and the textual / JSON result returned to the agent, including error results. Built-in tools (bash, read, write, edit, glob, grep, ask_user) and any MCP/agent-as-tool calls are all in scope.
7. The flag also enables an `INFO`/`DEBUG`-level lift on relevant component loggers (orchestrator decisions, plan steps, compression events) so the surrounding context of each I/O record is interpretable. Loggers unrelated to model/tool I/O remain at their default level.
8. Streaming LLM responses are reconstructed and emitted as a single debug record after the stream completes (plus, optionally, a marker when streaming starts), to avoid drowning the terminal in chunk-level noise.
9. Environment variable `VV_DEBUG=true` is honored as an equivalent to `--debug`, consistent with existing `VV_*` config override conventions.
10. Cost / token usage continues to be reported through existing channels (CLI status bar, HTTP response). Debug output complements but does not replace it.

## 3. User Stories

### US-1: Developer inspects exact prompt sent to the model (CLI)
**As a** developer using `vv` in CLI mode
**I want** to start `vv --debug` and see, for every LLM call, the exact system prompt, message history, tool schemas, and model parameters that were sent
**So that** I can diagnose unexpected agent answers and verify that compression/memory injection produced the prompt I expected.

**Acceptance criteria:**
- Running `vv --debug` starts the CLI normally; the TUI is unaffected.
- For every LLM call made by any agent, a debug record is emitted to stderr containing: agent name, model id, full system prompt, full ordered message list (role + content + tool_calls + tool_results), tool schemas offered, temperature/max_tokens/other params, and a request id correlating it to the matching response record.
- Without `--debug`, none of the above is emitted.

### US-2: Developer inspects raw model response (CLI)
**As a** developer
**I want** to see the full model response for each call when `--debug` is on
**So that** I can verify tool-call selection, finish reason, and token accounting.

**Acceptance criteria:**
- For every completed LLM call, a debug record is emitted containing: matching request id, assistant text content, tool_calls (name + arguments JSON), finish reason, input/output/cache-read token counts, latency.
- For streaming calls, a single consolidated record is emitted on stream completion. An optional "stream started" marker is acceptable.
- Errors from the LLM (HTTP errors, parse errors, context overflow) produce a debug record with the error type and message, correlated to the request id.

### US-3: Developer inspects tool call arguments and results
**As a** developer
**I want** to see, for every tool call, the arguments passed in and the result returned
**So that** I can debug tool integration issues and understand why an agent took an action.

**Acceptance criteria:**
- For every tool invocation (bash, read, write, edit, glob, grep, ask_user, MCP, agent-as-tool), a debug record is emitted containing: invoking agent, tool name, source (builtin/mcp/agent), full arguments JSON, full result string (subject to the same truncation policy as the actual agent input — i.e., debug shows what the agent saw), success/error, latency.
- Read-only vs. write classification is included for clarity.
- Tool errors include the error message.
- Without `--debug`, tool execution proceeds with only the existing tool-confirmation UX.

### US-4: Operator inspects model and tool I/O in HTTP mode
**As an** operator running `vv` in HTTP service mode
**I want** to start the server with `--debug` (or `VV_DEBUG=true`) and have detailed model/tool I/O written to the server log
**So that** I can troubleshoot production behavior without changing the API contract for clients.

**Acceptance criteria:**
- `vv --mode http --debug` starts the HTTP service normally on the configured address.
- For every sync, streaming, and async request, all LLM calls and tool calls performed during request handling produce debug records in the server log, each annotated with the originating HTTP request id (or async task id).
- The HTTP response body is byte-identical to non-debug mode for the same inputs (modulo non-deterministic LLM output). No new fields, no additional SSE events.
- The `/v1/health` endpoint, agent listing, tool listing, and memory CRUD endpoints are unaffected.

### US-5: Debug flag is off by default and inert
**As a** user who does not pass `--debug`
**I want** vv to behave exactly as it does today
**So that** debug instrumentation has zero observable cost in normal use.

**Acceptance criteria:**
- Default invocations of `vv` (no flag, `VV_DEBUG` unset) produce no additional output.
- No additional log files are created.
- No measurable latency regression on tool calls or LLM calls in non-debug mode.

### US-6: Sensitive configuration is redacted in debug output
**As a** user enabling `--debug`
**I want** API keys and other known secrets to be redacted in debug output
**So that** I can paste debug logs into bug reports without leaking credentials.

**Acceptance criteria:**
- LLM client configuration logged in debug mode (base url, model, provider) does NOT include the API key.
- Environment values for `VV_LLM_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `AI_API_KEY` are never written to debug output.
- Prompt/tool content is NOT auto-redacted (the user owns that data).

### US-7: Help text and discoverability
**As a** new user
**I want** `vv --help` to clearly describe `--debug`
**So that** I know the flag exists and what it does.

**Acceptance criteria:**
- `vv --help` lists `--debug` with a one-line description and notes the `VV_DEBUG` environment variable equivalent.
- `vv --help` notes that debug output goes to stderr (CLI) or server log (HTTP).

## 4. Scope

### In Scope
- New top-level CLI flag `--debug` (boolean) on the `vv` binary.
- New environment variable `VV_DEBUG` honored as equivalent to `--debug`.
- New configuration field on the Configuration model: `debug` (boolean, default false), overridable by env and CLI flag with the existing precedence (CLI > env > YAML > default).
- Debug record emission for every LLM call performed by any agent (orchestrator, coder, researcher, reviewer, chat, dynamically created sub-agents).
- Debug record emission for every tool call performed by any agent, including built-in tools, MCP tools, and agent-as-tool calls.
- Correlation ids linking each LLM request to its response record and each tool call's start/end.
- Redaction of known secret-bearing config fields in debug output.
- Help text update for `vv --help`.
- Equivalent behavior in CLI mode and HTTP mode.

### Out of Scope
- Runtime toggling of debug mode via a CLI command (`/debug on|off`) — future enhancement.
- Per-agent or per-tool debug filtering — future enhancement.
- Structured (JSON) debug log format suitable for machine ingestion — future enhancement; MVP is human-readable text.
- Persistent debug log files / log rotation — users redirect stderr themselves.
- Changes to the HTTP API response schema (no `debug` field, no extra SSE events).
- Automatic redaction of user-provided prompt content or tool I/O.
- Sampling or rate-limiting of debug records.
- Debug instrumentation for non-LLM, non-tool subsystems (memory store internals, config loader internals) beyond what is needed to make I/O records interpretable.
- Web UI / TUI panel for browsing debug records.

## 5. Roles Involved

| Role | Interaction |
|------|------------|
| Developer | Primary consumer; runs `vv --debug` interactively to inspect prompts and tool I/O during agent runs. |
| DevOps / Operator | Enables `--debug` (or `VV_DEBUG`) on the HTTP service to capture model/tool I/O in server logs for production troubleshooting. |
| External Systems | Unaffected; HTTP API contract is unchanged in debug mode. |

No new roles. No permission/authorization changes.

## 6. Models and State Changes

### Configuration (existing model — extended)
Add a new field:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `debug` | boolean | `false` | When true, vv emits detailed LLM and tool I/O records to stderr (CLI mode) or the server log (HTTP mode). Source precedence: CLI flag `--debug` > env `VV_DEBUG` > YAML `debug` > default. |

No state-machine changes. No new dictionaries required (a boolean does not need a dictionary). No changes to existing dictionaries.

### Other models
No changes. In particular: Tool, Agent, Token Usage, Session Cost Tracker, HTTP Request, Async Task, CLI Session, CLI Message — all unchanged.

## 7. Business Processes and Rules

### Affected processes (existing)

1. **Application Startup** (`procedures/core/config/procedure-application-startup.md`) — must read `--debug` flag and `VV_DEBUG` env into the Configuration model and apply the resolved `debug` value to the logging/instrumentation layer before any agent or tool is constructed.
2. **CLI Startup** — when `debug` is true, ensure the bubbletea TUI is configured so that stderr is not captured by the TUI renderer and remains visible to the user (or is otherwise routed to a debug sink that does not corrupt the TUI on stdout).
3. **CLI Message Processing** — every LLM call and every tool call within message processing emits debug records when `debug` is true. Compression events and orchestrator routing decisions emit context records so the I/O records are interpretable.
4. **Synchronous / Streaming / Async Request** (HTTP) — same as above; debug records are tagged with the HTTP request id (or async task id). Response bodies are unchanged.
5. **Orchestration** — orchestrator's own LLM calls (task understanding, plan generation, dispatch decisions) and the LLM/tool calls of dispatched sub-agents are all instrumented uniformly.

### New rules

- **R-1 Default off:** `debug` defaults to false. Absence of the flag and env yields the current behavior exactly.
- **R-2 Precedence:** CLI flag > environment variable > YAML config > default, consistent with other `VV_*` overrides.
- **R-3 Uniform coverage:** when `debug` is true, every LLM call and every tool call performed by any agent (static, dynamic, orchestrator) within the process produces debug records. No agent is silently exempt.
- **R-4 Output isolation:** debug records must not corrupt the CLI TUI rendering on stdout, and must not appear in HTTP response bodies. CLI debug goes to stderr; HTTP debug goes to the server log writer.
- **R-5 Correlation:** each LLM request and its response share a correlation id; each tool call's invocation and completion records share a correlation id; in HTTP mode records also carry the HTTP request id / async task id.
- **R-6 Streaming consolidation:** for streaming LLM calls, the debug response record is emitted once on stream completion and contains the full reconstructed assistant message and tool calls.
- **R-7 Secret redaction:** API keys and known secret-bearing config fields are never present in debug output. User-provided prompt and tool content are not auto-redacted.
- **R-8 No content-length cap beyond agent's view:** debug shows tool results exactly as the agent received them, i.e., after vage's existing tool-output truncation. Debug must not bypass nor introduce its own additional truncation in MVP.
- **R-9 No behavioral side effects:** enabling debug must not change agent decisions, model parameters, retry behavior, compression thresholds, cost calculation, or any user-visible result.

## 8. Applications and Pages

### vv CLI (`applications/cli/application-cli.md`)
- Update launch flags page (or equivalent) to document `--debug` alongside existing `--mode`, `--config`, etc.
- Update help-output description.
- No new pages. The interactive TUI itself is unchanged; debug output appears on stderr beside/under the TUI when the user redirects or runs in a scrollback-friendly terminal.

### vv HTTP API (`applications/api/application-api.md`)
- Document that the service binary accepts `--debug` and `VV_DEBUG`, and that enabling them causes detailed model/tool I/O to appear in the server log without changing any HTTP endpoint contract.
- No changes to endpoint pages. No changes to request/response schemas. No changes to SSE event types.

## 9. Open Questions / Future Enhancements

These are explicitly out of scope for this requirement and are listed for future planning:

- Runtime `/debug` toggle command in the CLI.
- Structured JSON debug log format with a stable schema.
- Per-agent / per-tool debug filtering (e.g., `--debug=tools` or `--debug=llm`).
- Dedicated debug log file with rotation, separate from stderr.
- Optional inclusion of debug records in HTTP responses behind an explicit per-request header (would require API contract change).
- Automatic PII / secret scanning of prompt and tool content.
