# M1 Fast-Path: Heuristic Short-Circuit in Dispatcher

## 1. Background & Objectives

The `vv` Dispatcher (`vv/dispatches/`) currently forces every request through a mandatory 3-phase pipeline: **intent recognition (LLM) → execution (LLM) → optional summarization**. Trivial inputs such as `hello`, `thanks`, `calc 5^6` pay at least **2 serial LLM round-trips** — the first is pure routing overhead that always ends in `{"mode":"direct","agent":"chat"}` or similar. In pathological cases the intent LLM can falsely set `needs_exploration=true` and spin up the explorer for up to 15 additional LLM iterations.

M1 is the **first milestone** of the broader plan in `doc/design/enhance-prompt-understand-and-solve.md`. It is a deliberately small, zero-LLM optimization that short-circuits obviously-trivial inputs to a concrete sub-agent without running intent recognition at all.

### Objectives

1. Eliminate the intent LLM call for trivial greetings / small talk / single-word tool triggers.
2. Preserve today's behavior for every input that does not match the heuristic whitelist.
3. Make the heuristic **configurable and off-switchable** so it can be tuned or disabled in production.
4. Keep the change **surgical**: one new source file, minimal touch in `dispatch.go`, pure additions in `configs/`.

### Non-Objectives (explicitly out of scope for M1)

- No changes to the intent prompt, explorer, planner, or DAG execution.
- No change to chat/coder/reviewer/researcher agent implementations.
- No introduction of function-calling, small router models, or unified Primary Assistant — those are M2/M3/M4.
- No persistent cache for responses.

## 2. User Stories & Acceptance Criteria

### US-1 Greetings bypass the intent LLM

**As a** vv CLI/HTTP user
**When I** type a short greeting (`hello`, `hi`, `hey`, `你好`, `在吗`, `thanks`, `bye`, ...)
**Then** the Dispatcher must route my request directly to the `chat` sub-agent **without** performing intent recognition.

Acceptance criteria:
- AC-1.1 Given a single-message request `{"role":"user","content":"hello"}` with no prior tool calls, `Dispatcher.Run` emits **0 intent LLM calls** and **1 chat LLM call**. Verified via a mock `aimodel.ChatCompleter` that counts calls by system prompt.
- AC-1.2 `Dispatcher.RunStream` emits a `phase_start/phase_end` pair with phase `"fast_path"` **instead of** `"intent"` when the fast-path fires, and the execute phase still runs normally afterwards.
- AC-1.3 The heuristic matches each pattern case-insensitively with an anchored prefix (`^`) and a word boundary (`\b`), so `hello world` matches but `helloworld` does not (because `world` starts with a letter — we require a non-letter after the trigger word, or end-of-string).
- AC-1.4 When the input matches a greeting pattern but its total length exceeds the configured `max_chars` (default 60), the fast-path does **not** fire.
- AC-1.5 Supported greetings out of the box (case-insensitive): `hi`, `hello`, `hey`, `thanks`, `thank you`, `bye`, `goodbye`, `你好`, `您好`, `在吗`, `哈喽`.

### US-2 Simple shell-like triggers go to the coder agent

**As a** vv user
**When I** type a command that obviously maps to a shell tool (`calc 5^6`, `date`, `pwd`, `ls`, `echo foo`)
**Then** the Dispatcher must route my request directly to the `coder` sub-agent so its `bash` tool can handle it.

Acceptance criteria:
- AC-2.1 The default tool-trigger pattern set: `^(calc|echo|date|pwd|ls|whoami|uptime)(\s|$)`.
- AC-2.2 Match → direct dispatch to `coder` without an intent LLM call. Same stream-phase accounting as AC-1.2.
- AC-2.3 If the matched `coder` is unavailable in `subAgents` (e.g. running with a trimmed registry), the fast-path **declines** (returns miss) and the normal intent flow runs.

### US-3 Fast-path respects conversation context

**As a** vv user in the middle of a multi-turn task
**When I** send a short "thanks" after the assistant has been doing tool calls
**Then** the fast-path must **not** hijack the routing — the normal intent flow should run so the follow-up stays in context.

Acceptance criteria:
- AC-3.1 The fast-path only fires when `len(req.Messages) <= 2` (one user message, optionally one leading system/note from enrichment). If there are more messages, no fast-path.
- AC-3.2 The fast-path only fires when no message in `req.Messages` contains tool calls or tool results. Scan `msg.Content.Parts`/`msg.ToolCalls` — if any entry is non-empty for a non-user role, decline.
- AC-3.3 The fast-path only inspects the **last** user message; older context is not matched.

### US-4 Configuration knobs

**As an** operator
**I want** to enable/disable the fast-path, tune the max-char threshold, and add/remove patterns via `~/.vv/vv.yaml`.

Acceptance criteria:
- AC-4.1 New YAML section under `orchestrate`:
  ```yaml
  orchestrate:
    fast_path:
      enabled: true                # default true
      max_chars: 60                # default 60
      greeting_patterns: [...]     # optional: override defaults
      tool_trigger_patterns: [...] # optional: override defaults
  ```
- AC-4.2 When `enabled: false`, the fast-path is bypassed entirely — behavior is identical to today's main branch.
- AC-4.3 If either pattern slice is nil, the built-in default set is used. If it's an empty slice `[]`, that category is fully disabled (no matches for that category).
- AC-4.4 Invalid regex patterns in the YAML log a warning and are silently dropped; they do not fail config loading (consistent with `MCPCredentialFilterConfig.ExtraPatterns` convention, `configs/config.go:152`).
- AC-4.5 `dispatches.New` receives the parsed options via a new `WithFastPath(FastPathConfig)` option. If never called, the fast-path defaults to **enabled** with default patterns (backwards compatible: today's config files automatically benefit).

### US-5 Observability

**As an** operator watching traces/logs
**I want** to know when the fast-path fires and why.

Acceptance criteria:
- AC-5.1 Each fast-path hit produces an `slog.Info` line with fields: `fast_path_hit=true`, `category=greeting|tool_trigger`, `agent=<id>`, `matched_pattern=<regex>`. No user content is logged (PII safety).
- AC-5.2 Streaming mode emits `phase_start("fast_path")` + `phase_end("fast_path", summary="Fast path -> <agent>")` events so CLI/HTTP dashboards can display the hit.
- AC-5.3 The stream-phase `PhaseEndData` for fast-path sets `PromptTokens=0`, `CompletionTokens=0`, `ToolCalls=0` — makes the savings visible.

## 3. In-Scope / Out-of-Scope

### In-Scope

- `vv/configs/config.go` — add `FastPathConfig` struct nested under `OrchestrateConfig`.
- `vv/dispatches/` — new file `fastpath.go` holding the matcher + config types that belong to the package; `dispatch.go` wiring only.
- `vv/dispatches/dispatch.go` — add `WithFastPath` option + call `fastPath(req)` at the top of `Run`/`RunStream`.
- `vv/setup/setup.go` — wire `cfg.Orchestrate.FastPath` into `dispatches.WithFastPath`.
- Unit tests for the matcher and for Dispatcher's short-circuit wiring.

### Out-of-Scope

- Changing chat's `MaxIterations=1`.
- Adding new event types to `vage/schema/event.go`. We reuse `EventPhaseStart/End` with `Phase="fast_path"`.
- HTTP API schema changes.
- Integration tests against real LLMs (pure unit-level mocks are sufficient for M1).

## 4. Affected Roles / Models / Processes / Applications

| Dimension | Touched? | Notes |
|-----------|----------|-------|
| Roles | No new role | Orchestrator Agent behavior expanded with a pre-intent filter |
| Models | No new model | `FastPathConfig` is a config object, not a domain model |
| Processes | Yes — **Orchestration** procedure gets a new pre-step | `doc/prd/procedures/core/orchestration/procedure-orchestration.md` will need a note inserted between Step 1 and Step 2 describing the heuristic short-circuit |
| Applications | Neither CLI nor HTTP changes behavior visibly — fast-path is transparent to API consumers | Document visibility via `phase_start("fast_path")` event only |

## 5. Verifiable Success Criteria

| # | Criterion | How to verify |
|---|-----------|---------------|
| SC-1 | "hello" triggers 0 intent LLM calls | Unit test with counting mock `ChatCompleter` |
| SC-2 | Fast-path disables via config | Unit test flipping `Enabled=false` and observing intent LLM call resumes |
| SC-3 | Long greeting (>60 chars) does **not** trigger fast-path | Unit test |
| SC-4 | Multi-turn context does **not** trigger fast-path | Unit test with 3-message history |
| SC-5 | Missing `coder` agent → tool-trigger fast-path declines, intent flow runs | Unit test with subAgents missing `coder` |
| SC-6 | `make test` in `vv/` stays green | `cd vv && make test` |

## 6. Open Questions & Assumptions

- **Q1**: Should fast-path bypass `summarize` entirely? **Assumption**: Yes — since no plan runs, `shouldSummarize` naturally returns false for `SummaryAuto` in CLI mode. For HTTP + `SummaryAuto`, summarization would still trigger today; for M1 we **preserve** that behavior (fast-path only eliminates intent, not summarize). This can be revisited in M2.
- **Q2**: Should we remember a session's "sticky agent" as hinted by the raw requirement rule #3 ("历史上下文检测: 对话历史中已经确定 agent → 后续同 session 短消息继续沿用")? **Decision**: **Defer to M2**. Session-sticky routing requires persistent state that the Dispatcher currently doesn't own; adding it now would violate the "surgical" constraint. M1 handles only single-shot heuristic matching.
- **Q3**: Character length vs. rune length — `max_chars` applies to UTF-8 rune count (not bytes) so `"你好"` is 2, not 6. Important for CJK greeting support.
