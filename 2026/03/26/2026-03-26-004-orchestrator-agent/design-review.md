# Design Review: Replace Router Agent with Orchestrator Agent

## Overview

The proposed design is well-structured and correctly identifies the key architectural changes needed. The following review provides specific improvements organized by severity.

---

## Critical Issues

### CR-01: RunStream Should Delegate to Sub-Agent Streaming, Not Wrap Run

**Problem:** The design specifies `RunStream` uses `agent.RunToStream(ctx, orchestrator, req)`, which wraps the synchronous `Run` method. This means the entire orchestration (including sub-agent work) runs as a single blocking call, and the TUI receives no streaming events (no text deltas, no tool call events) until the entire operation completes. The user sees a blank spinner for the full duration.

In the current architecture, the Router's `RunStream` calls `agent.RunToStream` which calls `Run`, which delegates to the selected sub-agent's `Run`. This works because the Router is a thin wrapper. But the Orchestrator's `Run` for plan mode will take much longer (multiple sub-agent calls), making the lack of streaming far more noticeable.

For direct dispatch mode, the Orchestrator should call `subAgent.RunStream()` directly and proxy the stream to the caller, preserving real-time text deltas and tool call events. For plan mode, `agent.RunToStream` is acceptable initially since DAG execution is inherently non-streaming.

**Suggestion:** Split `RunStream` into two paths:
- **Direct mode:** Call `subAgent.RunStream(ctx, req)` and return the stream directly. This preserves the current streaming UX.
- **Plan mode:** Use `agent.RunToStream(ctx, orchestrator, req)` as a fallback since DAG execution does not natively stream.

### CR-02: Classification LLM Call Uses Wrong Abstraction

**Problem:** The design says `classifyTask` calls the LLM via a `planGen *taskagent.Agent`. But `taskagent.Agent` is a full ReAct-loop agent with tool calling, memory management, and streaming infrastructure. Using it for a simple JSON classification call adds unnecessary overhead and complexity.

The classification call is a single-shot LLM request that returns structured JSON. It should use the raw `aimodel.ChatCompleter` (the `llm` field) directly, not a `taskagent.Agent`.

**Suggestion:** Use `llm.ChatComplete(ctx, &aimodel.ChatRequest{...})` directly in `classifyTask`. This is simpler, faster, and avoids the ReAct loop overhead. The `planGen` taskagent is still useful for the plan summarization/aggregation step (where it acts as a proper LLM agent), but not for classification.

### CR-03: The `chat` Agent Is Missing from `subAgents` in Direct Dispatch

**Problem:** The design's `OrchestratorSystemPrompt` lists `"chat"` as an available agent for direct dispatch, and `ClassifyResult.Agent` can be `"chat"`. However, in the existing `planner.go`, the `subAgents` map only includes `coder`, `researcher`, and `reviewer` -- `chat` is not in the map. The design mentions including `chat` in `subAgents` in section 2.3, but the `parsePlan` validation (section 2.8) only allows `coder`, `researcher`, `reviewer`. These are inconsistent.

**Suggestion:** Ensure the `subAgents` map includes `chat`, and update the plan step validation to allow `chat` as a valid agent for both direct dispatch and plan steps. The validation logic from `parsePlan` needs updating when moved to `orchestrator.go`.

---

## Significant Improvements

### SI-01: Separate Classification Logic from Plan Parsing

**Problem:** The design merges classification and plan parsing into one `classifyTask` method that returns a `ClassifyResult`. But `parsePlan` (from planner.go) is separate and parses a `Plan` from a `RunResponse`. These are two different parsing contexts -- the classification response is a new JSON schema, while `parsePlan` handles the existing plan-only JSON.

**Suggestion:** Create two clear methods:
1. `classifyTask(ctx, req) -> ClassifyResult` -- makes the LLM call and parses the classification JSON (which may include an inline plan).
2. Keep `parsePlan` as a utility for parsing the plan portion only, called from within `classifyTask` when `mode == "plan"`.

### SI-02: Include Original User Message in Sub-Agent Requests for Direct Dispatch

**Problem:** For direct dispatch, the design passes `req` (the full conversation history) directly to the sub-agent. This is correct. But for plan mode, step descriptions replace the user message entirely. The sub-agent loses the original user context, which may lead to less accurate execution.

**Suggestion:** For plan step execution, include both the step description and a summary of the original user request. Modify the `InputMapper` to prepend a system-level context message: "Original user request: {goal}" before the step description. This is a small change to `buildNodes`.

### SI-03: Working Directory Should Be Injected into All Agent System Prompts

**Problem:** The design injects the working directory into the `OrchestratorSystemPrompt` via template substitution (`{{.WorkingDir}}`). But sub-agents (coder, researcher, reviewer) do not receive this information in their system prompts. They only get it indirectly through the bash tool's `BashWorkingDir` setting.

**Suggestion:** Add the working directory to all agent system prompts (coder, researcher, reviewer). This gives agents context about the user's project location even when they are not using bash. Consider adding it to the `CoderSystemPrompt`, `ResearcherSystemPrompt`, and `ReviewerSystemPrompt` as a template variable, or have the Orchestrator prepend it as a context message when dispatching.

The simpler approach: have the Orchestrator prepend a system message "Working directory: /path/to/dir" to the request messages when dispatching to sub-agents, rather than modifying all system prompt templates.

### SI-04: Prompt Template Engine Mismatch

**Problem:** The `OrchestratorSystemPrompt` uses Go template syntax (`{{.WorkingDir}}`), but the existing system prompts (`CoderSystemPrompt`, etc.) are plain string constants used with `prompt.StringPrompt()`. The design does not specify how the template rendering happens for the Orchestrator prompt.

**Suggestion:** Use `fmt.Sprintf` or `strings.Replace` for the simple working directory substitution rather than introducing Go `text/template` machinery. This keeps the approach consistent with the rest of the codebase. For example:
```go
systemPrompt := strings.Replace(OrchestratorSystemPrompt, "{{.WorkingDir}}", workingDir, 1)
```

### SI-05: Fallback Agent Should Be `chat`, Not `coder`

**Problem:** The existing `PlannerAgent` uses `coderAgent` as its fallback (line 147 in `agents.go`). The design retains this pattern for the Orchestrator. But the requirement (ORCH-08) states: "If the Orchestrator cannot understand the instruction or all dispatch attempts fail, fall back to the Chat Agent." The fallback should be the chat agent, not the coder.

**Suggestion:** Change the fallback agent from `coderAgent` to `chatAgent` to align with the requirement. The chat agent is the correct fallback for unclassifiable requests because it can handle general conversation without tools.

---

## Minor Improvements

### MI-01: Remove Redundant `PlannerSystemPrompt` and `PlanSummaryPrompt`

**Problem:** The design says these prompts "are retained since the plan generation sub-agent and aggregator still use them internally." But the Orchestrator's classification prompt (`OrchestratorSystemPrompt`) replaces the plan generation role, and the `PlanAggregator` uses `PlanSummaryPrompt` for aggregation. After the migration, `PlannerSystemPrompt` is no longer used.

**Suggestion:** Remove `PlannerSystemPrompt` in this change (it will have no consumers after the Planner is deleted). Keep `PlanSummaryPrompt` since the `PlanAggregator` still uses it.

### MI-02: Add `NewOrchestratorAgent` Constructor Function

**Problem:** The design defines the `OrchestratorAgent` struct but does not show a constructor function. The existing `PlannerAgent` has `NewPlannerAgent`.

**Suggestion:** Add a `NewOrchestratorAgent` constructor that mirrors the pattern from `NewPlannerAgent`, with compile-time interface checks:
```go
var (
    _ agent.Agent       = (*OrchestratorAgent)(nil)
    _ agent.StreamAgent = (*OrchestratorAgent)(nil)
)
```

### MI-03: Token Usage Aggregation for Classification Call

**Problem:** The design mentions token usage aggregation (ORCH-10) but does not specify how the classification LLM call's token usage is captured and added to the final response.

**Suggestion:** After the `llm.ChatComplete` call in `classifyTask`, extract `usage` from the LLM response and store it. In `Run`, aggregate this with the sub-agent response's usage before returning. The `aimodel.ChatResponse` likely includes usage information.

### MI-04: Config Comment Update Is Underspecified

**Problem:** Task 8 says "Update MaxConcurrency comment." But `MaxConcurrency` lives in `MemoryConfig`, which is an odd location for orchestration concurrency. The design does not address this structural concern.

**Suggestion:** While a structural move (e.g., to a new `OrchestratorConfig`) is out of scope, add a TODO comment noting the field's placement is legacy and should be moved in a future cleanup.

### MI-05: HTTP API Breaking Change Not Addressed

**Problem:** Section 4.2 notes that `/v1/agents/router` becomes `/v1/agents/orchestrator`. This is a breaking change for any HTTP API consumers. The design does not mention backward compatibility.

**Suggestion:** Document this as a known breaking change in the implementation plan. If backward compatibility is needed, add a redirect or alias. If not, note it explicitly so the change is intentional and communicated.

### MI-06: Task Ordering in Implementation Plan

**Problem:** Task 3 (Capture Working Directory) is listed after Task 2 (Update agents.go), but agents.go's Create function reads `cfg.Tools.BashWorkingDir` which must already be populated. The working directory capture should happen before agent creation.

**Suggestion:** Merge Task 3 into Task 5 (Update main.go Wiring), since both modify main.go and the working directory capture must happen before `agents.Create()` is called. The implementation order becomes: Task 1, Task 2, Task 4, Task 5 (includes working directory), Task 6, Task 7, Task 8.

---

## Summary of Changes

| ID | Severity | Section | Summary |
|----|----------|---------|---------|
| CR-01 | Critical | 2.1 | RunStream should proxy sub-agent stream in direct mode |
| CR-02 | Critical | 2.2 | Use raw LLM client for classification, not taskagent |
| CR-03 | Critical | 2.1, 2.8 | Include chat agent in subAgents map and validation |
| SI-01 | Significant | 2.2 | Separate classification from plan parsing cleanly |
| SI-02 | Significant | 2.1 | Include original user context in plan step requests |
| SI-03 | Significant | 2.4 | Inject working directory into sub-agent context |
| SI-04 | Significant | 2.4 | Use simple string replacement for prompt template |
| SI-05 | Significant | 2.1 | Change fallback from coder to chat agent |
| MI-01 | Minor | 2.4 | Remove unused PlannerSystemPrompt |
| MI-02 | Minor | 2.1 | Add constructor and compile-time checks |
| MI-03 | Minor | 2.2 | Capture classification call token usage |
| MI-04 | Minor | 2.7 | Add TODO for MaxConcurrency placement |
| MI-05 | Minor | 4.2 | Document HTTP API breaking change |
| MI-06 | Minor | 5 | Fix task ordering in implementation plan |
