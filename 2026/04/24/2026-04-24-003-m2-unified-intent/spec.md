# M2 — Unified Intent (tool-calling merge of intent+answer)

> Session: 2026-04-24-003-m2-unified-intent
> Design: `doc/design/enhance-prompt-understand-and-solve.md` §3.2 / M2
> Owner: vv dispatches

## Core Goal (loop anchor)

When `configs.orchestrate.unified_intent=true` and the model chooses `answer_directly`, the dispatcher must produce the user-facing reply with **exactly one LLM call** (vs. the current 2: intent + chat sub-agent). Routing (`delegate_to`) and planning (`plan_task`) keep the existing 2-call shape.

Non-goals:
- Do not touch the fast path (M1) — it still runs first.
- Do not change explorer/reassess paths in this milestone; if `answer_directly` is not chosen, fall through to the current intent-classification contract (via the same call's `delegate_to`/`plan_task` tool output).
- Do not introduce a separate router model (that's M3).

## Done Contract

1. Config flag `orchestrate.unified_intent` (default `false`) loads from `~/.vv/vv.yaml` and is wired into the Dispatcher via a functional option.
2. With the flag on and no explorer/planner agent set, `recognizeIntent` goes through a new `recognizeIntentUnified` that sends tools `[answer_directly, delegate_to, plan_task]` in a single `aimodel.ChatRequest`.
3. `IntentResult` gains `Mode="answered"` + `Answer string`; when this mode is returned, `Dispatcher.Run`/`RunStream` skips `executeTask` and returns the answer directly.
4. Event phases emitted: `unified_intent` (start/end of the call), `unified_answer_direct` | `delegated_to` | `planned` (path taken — used by observability).
5. With the flag off, behavior is byte-identical to pre-M2 (verified by existing tests passing unchanged).
6. Unit tests (`unified_intent_test.go`) cover: each of the 3 tool choices, malformed tool call fallback, flag-off path, and LLM error propagation.
7. Integration tests (`vv/integrations/dispatches_tests/dispatches_tests/unified_intent_integration_test.go`) assert: (a) `answer_directly` → 1 sub-agent-free run, (b) `delegate_to` → same observable events as current direct flow, (c) `plan_task` → same DAG events as current plan flow, (d) flag-off runs old intent path.
8. `make test` and `make lint` pass in `vv/` and the integration sub-module.

**Evidence of done:** green `go test ./...` in `vv/` + the new integration test module; a one-run smoke test showing LLMCall count = 1 for an `answer_directly` path vs 2 for current behavior.

## Facts (from exploration)

- Current pipeline: `fastPathClassify` → `recognizeIntent` (LLM #1) → `executeTask` → sub-agent (LLM #2). `dispatch.go:224+`.
- `aimodel.ChatRequest` already supports `Tools []Tool` and `ToolChoice any`; `Message.ToolCalls` and `FinishReasonToolCalls` are canonical. `aimodel/schema.go:73`, `:213`, `:244`.
- `IntentResult` lives at `vv/dispatches/types.go:77`. Adding `Answer string` is additive/JSON-tag-safe.
- Intent LLM call today: `vv/dispatches/intent.go:103 recognizeIntentDirect` — raw JSON parse of the assistant text, no tools passed.
- Chat system prompt: `vv/agents/chat.go` `ChatSystemPrompt` — short, generic; safe to inline into the unified prompt's `answer_directly` guidance.
- M1 pattern to mirror: `OrchestrateConfig.FastPath` (`vv/configs/config.go:54–91`) + `WithFastPath` option + `fastpath.go` module + `fastpath_integration_test.go`. I'll follow the same shape.
- `recognizeIntent` has a planner-agent branch (`recognizeIntentViaPlanner`). Unified mode is mutually exclusive with that branch (the planner-agent path is an existing complex-scenario flow). When planner is set, unified mode skips itself — keep M2 scoped.

## Plan (sequenced)

1. **Config** — add `UnifiedIntent bool` to `OrchestrateConfig` in `vv/configs/config.go`; default false; YAML key `unified_intent`.
2. **Types** — extend `IntentResult` at `vv/dispatches/types.go` with `Answer string` and document `Mode="answered"`. Update `(IntentResult).validate` to treat `"answered"` as valid when `Answer != ""`.
3. **New file** `vv/dispatches/unified_intent.go`:
   - Tool schemas (3 tools) with Go struct for each parameter.
   - `recognizeIntentUnified(ctx, req) (*IntentResult, *aimodel.Usage, error)`.
   - Builds `ChatRequest` with `Tools: [...]`, `ToolChoice: "auto"`.
   - Parses `resp.Choices[0].Message.ToolCalls[0]`: maps to `IntentResult` (`answered`/`direct`/`plan`).
   - On no tool call (model returned plain text) → treat as `answered` with the text.
   - On parse error → fall back to `fallbackIntent()` (mirrors current fallback).
4. **Prompt** — `unifiedIntentSystemPromptTemplate` constant in same file. Much shorter than current intent prompt: lists agents, explains three tools, embeds `agents.ChatSystemPrompt` content for the `answer_directly` arm. Reuses `{{.AgentList}}` and `{{.WorkingDir}}` substitutions.
5. **Dispatcher wiring** — `dispatch.go`:
   - Add `unifiedIntent bool` field + `WithUnifiedIntent(bool)` option.
   - In `recognizeIntent`/`recognizeIntentStream`: if flag on AND `plannerAgent == nil` AND `llm != nil`, call `recognizeIntentUnified` before falling through to `recognizeIntentDirect`.
   - In `Run`/`RunStream`: when `intent.Mode == "answered"`, emit `unified_answer_direct` phase event + return a `RunResult` carrying `intent.Answer`; skip `executeTask`/`summarize`.
   - For `direct`/`plan`, emit `delegated_to`/`planned` phase-end tag so observability can count paths. (Piggyback on existing `EventPhaseEnd` with a `Reason` field or a new sub-event — choose whichever minimizes schema churn; confirm in code.)
6. **Setup wiring** — `vv/setup/` (or wherever Dispatcher is assembled) passes `cfg.Orchestrate.UnifiedIntent` into `WithUnifiedIntent`.
7. **Unit tests** — `vv/dispatches/unified_intent_test.go` with mocked `aimodel.Client` returning each tool-call shape + malformed + text-only.
8. **Integration tests** — `vv/integrations/dispatches_tests/dispatches_tests/unified_intent_integration_test.go` mirroring `fastpath_integration_test.go` style (mock LLM, assert event sequence + LLM call count).
9. **Validate** — `make test && make lint` in both `vv/` and `vv/integrations/dispatches_tests/`.
10. **Reverse Sync** — write `result.md` with actual file list, test output, and any divergence from this plan.

## Risks

- **Tool-calling compatibility**: some backends may return text instead of tool calls. Mitigated by the "treat plain text as answered" branch.
- **Prompt regression**: the unified prompt is new; if badly worded the model may over-use `plan_task`. Mitigated by mock-based tests; real-model eval is out of scope for M2 (called out as M4's concern in design §6.4).
- **Schema churn**: if `IntentResult.Mode="answered"` leaks to HTTP clients that don't know the value, they may 400. Compat posture: only emit `answered` when the flag is on, and `answered` only exists when unified mode is active — HTTP consumers that enable the flag opt into the schema.

## Checkpoint — awaiting approval before implementation

- Current understanding: above.
- Core goal: single-LLM-call answer path when flag on and LLM chooses `answer_directly`.
- Next 3 actions: (1) extend config + types; (2) add `unified_intent.go` with prompt + parser; (3) wire Dispatcher and write tests.
- Risks: backend tool-calling variance, prompt quality, schema leakage — mitigations above.
- Validation: unit + integration tests asserting exact LLM call counts per path; `make test`/`make lint` green.

**Open questions for the user (non-blocking defaults in brackets):**
1. Config key location: `orchestrate.unified_intent` (alongside `fast_path`) or a new `dispatcher` section? [default: `orchestrate.unified_intent`, matches M1.]
2. Event shape: add a new `EventPhaseEnd` *reason* field, or emit a new `EventTag` for path labels? [default: reuse existing `EventPhaseStart`/`End` with Phase="unified_intent" and pass path label via `Detail`/`Summary` — no new event type.]
3. Should `recognizeIntentViaPlanner` also gain a unified variant? [default: **no** — out of scope for M2, noted as future work.]
