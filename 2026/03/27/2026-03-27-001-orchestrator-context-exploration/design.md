# Design: Orchestrator Context Exploration and Planning

## 1. Architecture Overview

The orchestrator's request processing pipeline today is a two-step flow:

```
User Request -> classifyTask (LLM call) -> dispatch (direct/plan)
```

This design inserts an exploration phase between receiving the request and classifying it, producing a three-step flow:

```
User Request -> exploreContext (tool-calling LLM loop) -> classifyTask (LLM call, enriched) -> dispatch (direct/plan, enriched)
```

The exploration phase is an internal orchestrator capability, not a separate agent. It reuses the existing `taskagent` pattern -- an LLM with read-only tools in a bounded ReAct loop -- but runs it as a private helper within the orchestrator, not as a dispatched sub-agent.

The key architectural decision is to use a **single combined LLM session** for exploration and classification. The LLM is given the user's question, read-only tools, and instructions to first explore the codebase and then produce the classification JSON. This avoids two separate LLM calls (and their prompt overhead) while keeping the logic simple. The exploration naturally terminates when the LLM produces the classification JSON instead of another tool call.

### High-Level Flow

```
                     OrchestratorAgent
                     -----------------
                     |               |
                     v               |
               hasWorkingDir?        |
                /        \           |
              yes         no --------+---> classifyTask (existing, unchanged)
              |                      |
              v                      |
        explorerAgent.Run()          |     (internal taskagent with read-only tools)
              |                      |     (bounded: maxIterations=10, token budget)
              v                      |
        ExplorationResult            |     { contextSummary, classifyResult }
              |                      |
              v                      |
        classifyResult != nil? ------+---> use it directly (skip separate classify call)
              |                      |
             yes                     |
              |                      |
              v                      |
        enrichRequest (with context) |
              |                      |
              v                      |
        dispatch (direct/plan)       |
```

When the explorer LLM produces both a context summary and the classification JSON in the same session, the orchestrator skips the separate `classifyTask` call entirely. If the explorer fails or returns no classification, the orchestrator falls through to the existing `classifyTask` path, optionally passing any partial context summary it gathered.

## 2. Component Design

### 2.1 OrchestratorAgent Changes

The `OrchestratorAgent` struct gains two new fields:

```go
type OrchestratorAgent struct {
    agent.Base
    llm            aimodel.ChatCompleter
    model          string
    subAgents      map[string]agent.Agent
    planGen        *taskagent.Agent
    maxConcurrency int
    fallbackAgent  agent.Agent
    workingDir     string

    // New fields
    explorer          *taskagent.Agent   // internal exploration agent (nil if no workingDir)
    explorationConfig ExplorationConfig  // bounds for exploration
}
```

The `explorer` is a `taskagent.Agent` configured with:
- The read-only tool registry (glob, grep, read)
- A dedicated system prompt for exploration + classification
- A bounded iteration count and token budget from `ExplorationConfig`

### 2.2 ExplorationConfig

```go
// ExplorationConfig controls the bounds of the exploration phase.
type ExplorationConfig struct {
    MaxIterations  int // Maximum tool-call iterations for exploration. Default: 10.
    RunTokenBudget int // Maximum tokens for the exploration phase. Default: 0 (use agent default).
}
```

Defaults are applied in the constructor. These values can eventually be made configurable via `config.AgentsConfig` or `config.Config`, but the initial implementation uses hardcoded sensible defaults.

### 2.3 Explorer System Prompt

A new constant `ExplorerSystemPrompt` is added to `orchestrator.go`. This prompt instructs the LLM to:

1. Determine whether the user's request is project-related by checking if the working directory exists and is relevant.
2. If project-related, explore the codebase using the available read-only tools (glob, grep, read), focusing on files and patterns relevant to the user's question.
3. After exploration (or immediately if the question is not project-related), produce a response in a structured format containing:
   - An optional context summary (free-form text describing discovered project structure, relevant files, key types/functions, conventions).
   - The classification JSON (same schema as today's `ClassifyResult`).

The response format uses a simple delimiter-based structure:

```
<context_summary>
... free-form text about project structure, relevant files, key code ...
</context_summary>
<classification>
{"mode": "direct", "agent": "coder"}
</classification>
```

If the question is not project-related, the LLM skips exploration and produces:

```
<context_summary>
</context_summary>
<classification>
{"mode": "direct", "agent": "chat"}
</classification>
```

The prompt text:

```go
const ExplorerSystemPrompt = `You are an orchestrator agent that first explores a project codebase, then routes the user's request to the appropriate sub-agent.

## Working Directory
The user is working in: {{.WorkingDir}}

## Available Tools
- **glob**: Find files by name pattern (e.g., "**/*.go", "src/**/*.ts").
- **grep**: Search file contents using regular expressions.
- **read**: Read file contents.

## Available Agents for Routing
- "coder": Reads, writes, edits files, runs commands, searches codebases, debugs
- "researcher": Explores codebases, reads documentation, gathers information (read-only)
- "reviewer": Reviews code for correctness, style, performance, security
- "chat": General conversation, questions, explanations, brainstorming

## Your Task

### Step 1: Determine Project Relevance
Decide whether the user's request is about the project in the working directory or is a general question.
- General questions (e.g., "explain what a goroutine is", "what is the difference between TCP and UDP") do NOT need exploration.
- Project-specific questions (e.g., "add a new endpoint", "refactor the auth module", "how does the orchestrator work") DO need exploration.

### Step 2: Explore (if project-related)
If the request is project-related, use the tools to understand the relevant parts of the codebase:
- Start broad: use glob to find relevant files, grep to find key patterns.
- Narrow down: read specific files that are most relevant to the user's question.
- Focus only on what is needed for the question -- do not exhaustively scan the entire project.
- Build understanding of: project structure, relevant source files, key types/functions/interfaces, conventions.

### Step 3: Produce Output
After exploration (or immediately for general questions), respond with EXACTLY this format:

<context_summary>
(For project-related questions: describe the project structure, relevant files, key code constructs, and patterns you discovered. For general questions: leave this empty.)
</context_summary>
<classification>
(A JSON object -- same rules as below)
</classification>

### Classification Rules
For simple tasks (single agent): {"mode": "direct", "agent": "<agent_id>"}
For complex tasks (multi-step): {"mode": "plan", "plan": {"goal": "...", "steps": [{"id": "step_1", "description": "...", "agent": "coder", "depends_on": []}]}}

1. Use "direct" mode for tasks that clearly map to one agent capability.
2. Use "plan" mode only when the task genuinely requires multiple distinct steps across different capabilities.
3. For plan steps, use "depends_on" to specify ordering. Steps without dependencies run in parallel.
4. Keep plans focused: typically 2-5 steps.
5. Default to "coder" for ambiguous coding tasks, "chat" for general questions.
6. For project-related requests, reference specific files, functions, or patterns in step descriptions.
`
```

### 2.4 ExplorationResult

A new struct to hold the output of the exploration phase:

```go
// ExplorationResult holds the output of the exploration phase.
type ExplorationResult struct {
    ContextSummary string         // Free-form summary of project context discovered.
    Classify       *ClassifyResult // Classification produced during exploration (may be nil).
    Usage          *aimodel.Usage  // Token usage from the exploration phase.
}
```

### 2.5 New Method: exploreAndClassify

A new private method on `OrchestratorAgent`:

```go
// exploreAndClassify runs the exploration phase if an explorer is configured.
// It returns the exploration result. If no explorer is configured or the working
// directory is empty, it returns nil (signaling the caller to use the existing
// classifyTask path).
func (o *OrchestratorAgent) exploreAndClassify(ctx context.Context, req *schema.RunRequest) (*ExplorationResult, error)
```

Implementation:
1. If `o.explorer == nil`, return `nil, nil`.
2. Build a `schema.RunRequest` with the user's messages.
3. Call `o.explorer.Run(ctx, explorerReq)`.
4. Parse the explorer's response to extract `<context_summary>` and `<classification>` sections.
5. Parse the classification JSON into a `ClassifyResult` and validate it.
6. Return an `ExplorationResult` with the context summary, classification, and usage.

### 2.6 Response Parsing

A new helper function to parse the explorer's structured response:

```go
// parseExplorationResponse extracts the context summary and classification JSON
// from the explorer's structured response.
func parseExplorationResponse(text string) (contextSummary string, classifyJSON string, err error)
```

This uses simple string search for the `<context_summary>` and `<classification>` delimiters. If the delimiters are not found, the function falls back to treating the entire text as a potential classification JSON (backward-compatible with the existing `extractJSON` logic).

### 2.7 Modified Run / RunStream

Both `Run` and `RunStream` are updated to call `exploreAndClassify` first:

```go
func (o *OrchestratorAgent) Run(ctx context.Context, req *schema.RunRequest) (*schema.RunResponse, error) {
    // Phase 1: Explore and classify (if applicable).
    exploration, err := o.exploreAndClassify(ctx, req)
    if err != nil {
        slog.Warn("orchestrator: exploration failed, falling back to classify-only", "error", err)
    }

    var result *ClassifyResult
    var totalUsage *aimodel.Usage

    if exploration != nil && exploration.Classify != nil {
        // Exploration produced a valid classification.
        result = exploration.Classify
        totalUsage = exploration.Usage
    } else {
        // Fall back to the existing classification path.
        var classifyUsage *aimodel.Usage
        result, classifyUsage, err = o.classifyTask(ctx, req)
        if err != nil {
            slog.Warn("orchestrator: classification failed, falling back to chat", "error", err)
            return o.fallbackRun(ctx, req, aggregateUsage(explorationUsage(exploration), nil))
        }
        totalUsage = aggregateUsage(explorationUsage(exploration), classifyUsage)
    }

    // Enrich the request with exploration context.
    enrichedReq := o.enrichRequestWithContext(req, contextFromExploration(exploration))

    switch result.Mode {
    case "direct":
        return o.runDirect(ctx, enrichedReq, result, totalUsage)
    case "plan":
        return o.runPlan(ctx, enrichedReq, result.Plan, totalUsage)
    default:
        slog.Warn("orchestrator: unknown mode, falling back to chat", "mode", result.Mode)
        return o.fallbackRun(ctx, enrichedReq, totalUsage)
    }
}
```

The `RunStream` method follows the same pattern.

### 2.8 Modified enrichRequest

The existing `enrichRequest` method is extended (or a new variant `enrichRequestWithContext` is created) to include the exploration context summary:

```go
// enrichRequestWithContext prepends working directory and exploration context to a request.
func (o *OrchestratorAgent) enrichRequestWithContext(req *schema.RunRequest, contextSummary string) *schema.RunRequest {
    if o.workingDir == "" && contextSummary == "" {
        return req
    }
    var msgs []schema.Message
    if o.workingDir != "" {
        msgs = append(msgs, schema.NewUserMessage(
            fmt.Sprintf("Working directory: %s", o.workingDir),
        ))
    }
    if contextSummary != "" {
        msgs = append(msgs, schema.NewUserMessage(
            fmt.Sprintf("Project context:\n%s", contextSummary),
        ))
    }
    msgs = append(msgs, req.Messages...)
    return &schema.RunRequest{
        Messages:  msgs,
        SessionID: req.SessionID,
        Options:   req.Options,
        Metadata:  req.Metadata,
    }
}
```

The original `enrichRequest` method is preserved for backward compatibility but delegates to `enrichRequestWithContext` with an empty context summary. Plan step `InputMapper` functions are also updated to include the context summary.

### 2.9 Constructor and Agent Creation Changes

#### NewOrchestratorAgent

The constructor gains two new parameters:

```go
func NewOrchestratorAgent(
    cfg agent.Config,
    llm aimodel.ChatCompleter,
    model string,
    subAgents map[string]agent.Agent,
    planGen *taskagent.Agent,
    maxConcurrency int,
    fallback agent.Agent,
    workingDir string,
    readOnlyReg tool.ToolRegistry, // NEW
    explorationCfg ExplorationConfig, // NEW
) *OrchestratorAgent
```

Inside the constructor, if `workingDir != ""` and `readOnlyReg != nil`, it creates the internal `explorer` taskagent:

```go
var explorer *taskagent.Agent
if workingDir != "" && readOnlyReg != nil {
    maxIter := explorationCfg.MaxIterations
    if maxIter == 0 {
        maxIter = 10
    }
    opts := []taskagent.Option{
        taskagent.WithChatCompleter(llm),
        taskagent.WithModel(model),
        taskagent.WithToolRegistry(readOnlyReg),
        taskagent.WithSystemPrompt(prompt.StringPrompt(
            strings.Replace(ExplorerSystemPrompt, "{{.WorkingDir}}", workingDir, 1),
        )),
        taskagent.WithMaxIterations(maxIter),
    }
    if explorationCfg.RunTokenBudget > 0 {
        opts = append(opts, taskagent.WithRunTokenBudget(explorationCfg.RunTokenBudget))
    }
    explorer = taskagent.New(
        agent.Config{
            ID:          "orchestrator-explorer",
            Name:        "Orchestrator Explorer",
            Description: "Internal exploration phase for the orchestrator",
        },
        opts...,
    )
}
```

#### agents.Create

The `Create` function in `agents.go` is updated to pass the `readOnlyReg` and a default `ExplorationConfig` to `NewOrchestratorAgent`:

```go
orchestratorAgent := NewOrchestratorAgent(
    agent.Config{...},
    llm,
    cfg.LLM.Model,
    subAgents,
    planGen,
    cfg.Memory.MaxConcurrency,
    chatAgent,
    cfg.Tools.BashWorkingDir,
    readOnlyReg,                    // already available as a parameter
    ExplorationConfig{
        MaxIterations:  10,
        RunTokenBudget: 0,          // uses agent default
    },
)
```

## 3. Data Models / Schemas

### 3.1 ExplorationConfig

```go
type ExplorationConfig struct {
    MaxIterations  int // Max tool-call iterations. Default: 10.
    RunTokenBudget int // Max tokens for exploration. Default: 0 (unlimited).
}
```

### 3.2 ExplorationResult

```go
type ExplorationResult struct {
    ContextSummary string          // Free-form project context summary.
    Classify       *ClassifyResult // Parsed classification (may be nil on parse failure).
    Usage          *aimodel.Usage  // Token usage from exploration.
}
```

### 3.3 ClassifyResult (unchanged)

```go
type ClassifyResult struct {
    Mode  string `json:"mode"`
    Agent string `json:"agent"`
    Plan  *Plan  `json:"plan"`
}
```

### 3.4 Explorer Response Format

The explorer LLM produces a response with two delimited sections:

```
<context_summary>
... free text ...
</context_summary>
<classification>
{"mode": "direct", "agent": "coder"}
</classification>
```

This is parsed by `parseExplorationResponse()`. The format is chosen for simplicity and robustness -- the LLM naturally produces XML-like delimiters reliably when instructed.

## 4. Implementation Plan

### Task 1: Add ExplorationConfig and ExplorationResult types

**File:** `vv/agents/orchestrator.go`

- Add the `ExplorationConfig` struct.
- Add the `ExplorationResult` struct.
- Add the `ExplorerSystemPrompt` constant.

**Estimated effort:** Small. Pure type definitions and a string constant.

### Task 2: Add response parsing helper

**File:** `vv/agents/orchestrator.go`

- Implement `parseExplorationResponse(text string) (contextSummary, classifyJSON string, err error)`.
- Add unit tests in `orchestrator_test.go` for the parsing function (various formats, missing delimiters, empty sections, malformed input).

**Estimated effort:** Small. String parsing with tests.

### Task 3: Update OrchestratorAgent struct and constructor

**File:** `vv/agents/orchestrator.go`

- Add `explorer *taskagent.Agent` and `explorationConfig ExplorationConfig` fields to `OrchestratorAgent`.
- Update `NewOrchestratorAgent` signature to accept `readOnlyReg tool.ToolRegistry` and `explorationCfg ExplorationConfig`.
- Create the internal `explorer` taskagent in the constructor when `workingDir` and `readOnlyReg` are both non-empty/non-nil.

**File:** `vv/agents/agents.go`

- Update the `Create` function to pass `readOnlyReg` and a default `ExplorationConfig` to `NewOrchestratorAgent`.

**File:** `vv/agents/orchestrator_test.go`

- Update all existing test cases that call `NewOrchestratorAgent` to pass the two new parameters (nil registry and zero-value config for tests that do not need exploration).

**Estimated effort:** Medium. Constructor changes plus updating all test call sites.

### Task 4: Implement exploreAndClassify

**File:** `vv/agents/orchestrator.go`

- Implement `exploreAndClassify(ctx, req) (*ExplorationResult, error)`.
- Calls `o.explorer.Run()`, parses the response with `parseExplorationResponse`, validates the classification.
- Returns `nil, nil` when no explorer is configured.

**Estimated effort:** Medium. Core logic with error handling.

### Task 5: Implement enrichRequestWithContext

**File:** `vv/agents/orchestrator.go`

- Add `enrichRequestWithContext(req, contextSummary) *schema.RunRequest`.
- Refactor `enrichRequest` to delegate to `enrichRequestWithContext` with empty context.
- Update `buildNodes` to pass context summary into plan step `InputMapper` closures.

**Estimated effort:** Small. Refactoring existing method.

### Task 6: Update Run and RunStream to use exploration

**File:** `vv/agents/orchestrator.go`

- Modify `Run` to call `exploreAndClassify` first, then either use the exploration classification or fall back to `classifyTask`.
- Modify `RunStream` with the same logic.
- Use `enrichRequestWithContext` instead of `enrichRequest` for dispatch.
- Aggregate usage from exploration + classification/dispatch.

**Estimated effort:** Medium. Core integration point, needs careful error handling and fallback logic.

### Task 7: Unit tests for new behavior

**File:** `vv/agents/orchestrator_test.go`

- Test `exploreAndClassify` returns nil when no explorer is configured.
- Test `exploreAndClassify` with a mock that simulates tool calls and produces a valid exploration response.
- Test `enrichRequestWithContext` with various combinations of workingDir and contextSummary.
- Test the full `Run` path with exploration enabled (mock explorer produces classification).
- Test the full `Run` path where exploration fails and falls back to `classifyTask`.
- Test that non-project requests (no workingDir) skip exploration entirely.

**Estimated effort:** Medium. Several test scenarios with mock setup.

### Task Order and Dependencies

```
Task 1 (types)
    |
    v
Task 2 (parsing)       Task 3 (constructor + agents.go)
    |                       |
    v                       v
Task 4 (exploreAndClassify)
    |
    v
Task 5 (enrichRequest refactor)
    |
    v
Task 6 (Run/RunStream integration)
    |
    v
Task 7 (unit tests -- can be written alongside each task)
```

## 5. Integration Test Plan

Integration tests validate the end-to-end behavior with a real or mock LLM. They should be placed in the existing `integrations/` directory pattern.

### Test 1: Project-Related Question Triggers Exploration

**Setup:** Configure an orchestrator with a real working directory pointing to a small test project (e.g., a temporary directory with a few Go files). Use a mock `ChatCompleter` that:
- On the first call (exploration), responds with tool calls to glob/read, then produces the structured response with a context summary and classification.
- On sub-agent dispatch, returns a canned response.

**Verify:**
- The explorer received tool calls (glob, grep, or read were invoked).
- The sub-agent received a request containing the exploration context summary.
- The final response includes aggregated usage from both exploration and sub-agent execution.

### Test 2: General Question Skips Exploration

**Setup:** Configure an orchestrator with no working directory (empty string).

**Verify:**
- The explorer is nil, so `exploreAndClassify` returns nil immediately.
- The request proceeds to `classifyTask` directly.
- No tool calls are made for exploration.

### Test 3: Exploration Failure Falls Back to Classification

**Setup:** Configure an orchestrator with an explorer that returns an error (e.g., LLM returns an error or produces unparseable output).

**Verify:**
- The orchestrator logs a warning about exploration failure.
- The request falls through to `classifyTask`.
- The final response is still produced correctly.

### Test 4: Exploration with Plan Mode

**Setup:** Configure a mock explorer that returns a plan classification with context summary. The plan references specific files mentioned in the context summary.

**Verify:**
- The plan steps receive the context summary in their input.
- The DAG executes correctly with enriched inputs.

### Test 5: Exploration Respects Iteration Bounds

**Setup:** Configure an explorer with `MaxIterations: 3`. Use a mock LLM that always requests tool calls.

**Verify:**
- The exploration phase terminates after 3 iterations.
- The `StopReason` is `max_iterations_exceeded`.
- The orchestrator handles the partial result gracefully (falls back to `classifyTask` or uses any partial context gathered).

### Test 6: Streaming Path with Exploration

**Setup:** Same as Test 1 but using `RunStream` instead of `Run`.

**Verify:**
- The exploration phase completes before streaming begins.
- The sub-agent stream is proxied correctly.
- Events are emitted in the correct order.

### Test 7: Backward Compatibility

**Setup:** Create an orchestrator with the new constructor signature but pass `nil` for `readOnlyReg`.

**Verify:**
- The orchestrator behaves identically to the current implementation.
- All existing test cases continue to pass.
- No exploration phase is triggered.
