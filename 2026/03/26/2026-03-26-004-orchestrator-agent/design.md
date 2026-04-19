# Technical Design: Replace Router Agent with Orchestrator Agent

## 1. Architecture Overview

The Orchestrator Agent replaces both the Router Agent and the standalone Planner Agent with a single agent that serves as the main entry point for all user requests. It absorbs the Planner's plan-generation and DAG-execution capabilities while also handling simple tasks via direct dispatch -- eliminating the two-level indirection (Router -> Planner -> sub-agents).

### Current Architecture

```
User -> CLI/HTTP -> Router Agent --[LLM classify]--> sub-agent (coder/planner/researcher/reviewer/chat)
                                                          |
                                                     Planner Agent --[LLM plan]--> DAG execution --> sub-agents
```

### Target Architecture

```
User -> CLI/HTTP -> Orchestrator Agent --[LLM classify+plan]--> direct dispatch OR DAG execution --> sub-agents
```

### Key Design Decisions

1. **Single LLM call for classification + optional planning.** The Orchestrator uses one raw LLM call (via `aimodel.ChatCompleter` directly, not a `taskagent.Agent`) that returns a structured JSON response indicating either a direct dispatch (single agent) or a multi-step plan. This avoids both the two-hop latency and the overhead of a full ReAct-loop agent for a simple classification task.

2. **Streaming-aware direct dispatch.** For direct dispatch (single-agent tasks), the Orchestrator's `RunStream` calls the sub-agent's `RunStream` directly and proxies the stream to the caller, preserving real-time text deltas and tool call events. For plan mode, `agent.RunToStream` wraps the synchronous DAG execution since DAG execution is inherently non-streaming.

3. **Reuse existing `orchestrate.ExecuteDAG`.** The Planner already uses the vage framework's DAG engine. The Orchestrator continues to use `orchestrate.ExecuteDAG` for multi-step plans, so no changes to the orchestrate package are needed.

4. **Working directory captured at startup via `os.Getwd()`.** Stored in config and injected into the Orchestrator's system prompt (via simple string replacement) and the bash tool's working directory. The Orchestrator prepends working directory context to sub-agent requests at dispatch time.

5. **The Orchestrator is a custom agent in `vv/agents/`**, not a new agent type in the vage framework. It implements `agent.Agent` and `agent.StreamAgent` directly, keeping vage unchanged.

6. **Fallback to Chat Agent.** Per requirement ORCH-08, the fallback agent is the Chat Agent (not the Coder), since it can handle general conversation without tools when classification or dispatch fails.

## 2. Component Design

### 2.1 New File: `vv/agents/orchestrator.go`

The `OrchestratorAgent` struct replaces both `PlannerAgent` and the `routeragent.Agent` usage.

```go
package agents

// OrchestratorAgent is the main agent that receives all user requests.
// It decides whether to handle directly (single agent dispatch) or
// decompose into a multi-step plan executed as a DAG.
type OrchestratorAgent struct {
    agent.Base
    llm            aimodel.ChatCompleter
    model          string
    subAgents      map[string]agent.Agent  // coder, researcher, reviewer, chat
    planGen        *taskagent.Agent        // LLM agent for plan summarization
    maxConcurrency int
    fallbackAgent  agent.Agent             // chat agent as fallback
    workingDir     string                  // captured CWD
}

// Compile-time interface checks.
var (
    _ agent.Agent       = (*OrchestratorAgent)(nil)
    _ agent.StreamAgent = (*OrchestratorAgent)(nil)
)

// NewOrchestratorAgent creates a new OrchestratorAgent.
func NewOrchestratorAgent(
    cfg agent.Config,
    llm aimodel.ChatCompleter,
    model string,
    subAgents map[string]agent.Agent,
    planGen *taskagent.Agent,
    maxConcurrency int,
    fallback agent.Agent,
    workingDir string,
) *OrchestratorAgent {
    return &OrchestratorAgent{
        Base:           agent.NewBase(cfg),
        llm:            llm,
        model:          model,
        subAgents:      subAgents,
        planGen:        planGen,
        maxConcurrency: maxConcurrency,
        fallbackAgent:  fallback,
        workingDir:     workingDir,
    }
}
```

**Interface compliance:**
- Implements `agent.Agent` (Run)
- Implements `agent.StreamAgent` (RunStream)

**Run method flow:**

```
Run(ctx, req):
  1. Call classifyTask(ctx, req) -> ClassifyResult, classifyUsage
  2. If ClassifyResult.Mode == "direct":
       a. Look up ClassifyResult.Agent in subAgents
       b. Prepend working directory context message to request
       c. Dispatch directly: subAgent.Run(ctx, enrichedReq)
       d. Aggregate classifyUsage with sub-agent usage
       e. Return response
  3. If ClassifyResult.Mode == "plan":
       a. Parse Plan from ClassifyResult
       b. Build DAG nodes (reuse buildNodes logic from planner.go)
       c. Execute via orchestrate.ExecuteDAG
       d. Aggregate classifyUsage with DAG usage
       e. Return aggregated result
  4. On any failure: fall back to chat agent
```

**RunStream method:** Two-path implementation for optimal streaming:

```
RunStream(ctx, req):
  1. Call classifyTask(ctx, req) -> ClassifyResult, classifyUsage
  2. If ClassifyResult.Mode == "direct":
       a. Look up ClassifyResult.Agent in subAgents
       b. Cast to agent.StreamAgent
       c. Prepend working directory context message to request
       d. Return subAgent.RunStream(ctx, enrichedReq) -- proxies stream directly
  3. If ClassifyResult.Mode == "plan":
       a. Fall back to agent.RunToStream(ctx, orchestrator, req) -- wraps synchronous DAG
  4. On any failure: fall back to chat agent's RunStream
```

This ensures that direct dispatch (the common case) preserves real-time streaming of text deltas and tool call events to the TUI, while plan mode uses the synchronous wrapper (acceptable since DAG execution is inherently non-streaming).

### 2.2 Classification via Raw LLM Call

The `classifyTask` method makes one direct LLM call using `llm.ChatComplete` (the raw `aimodel.ChatCompleter` interface), not a `taskagent.Agent`. This avoids the overhead of a full ReAct loop for what is a simple single-shot JSON classification.

```go
// classifyTask makes a single LLM call to classify the user's request.
// Returns the classification result and the token usage from the LLM call.
func (o *OrchestratorAgent) classifyTask(ctx context.Context, req *schema.RunRequest) (*ClassifyResult, *aimodel.Usage, error) {
    systemPrompt := strings.Replace(OrchestratorSystemPrompt, "{{.WorkingDir}}", o.workingDir, 1)

    chatReq := &aimodel.ChatRequest{
        Model: o.model,
        Messages: append(
            []aimodel.Message{{Role: aimodel.RoleSystem, Content: aimodel.NewTextContent(systemPrompt)}},
            convertToAIModelMessages(req.Messages)...,
        ),
    }

    resp, err := o.llm.ChatComplete(ctx, chatReq)
    if err != nil {
        return nil, nil, err
    }

    // Parse JSON from response text.
    text := resp.Content.Text()
    jsonStr := extractJSON(text)

    var result ClassifyResult
    if err := json.Unmarshal([]byte(jsonStr), &result); err != nil {
        return nil, resp.Usage, fmt.Errorf("parse classification JSON: %w", err)
    }

    // Validate the classification result.
    if err := result.validate(o.subAgents); err != nil {
        return nil, resp.Usage, err
    }

    return &result, resp.Usage, nil
}
```

**ClassifyResult struct:**

```go
// ClassifyResult is the structured LLM response.
type ClassifyResult struct {
    Mode  string `json:"mode"`  // "direct" or "plan"
    Agent string `json:"agent"` // only for mode="direct": coder/researcher/reviewer/chat
    Plan  *Plan  `json:"plan"`  // only for mode="plan"
}

// validate checks that the classification result references valid agents.
func (cr *ClassifyResult) validate(subAgents map[string]agent.Agent) error {
    switch cr.Mode {
    case "direct":
        if _, ok := subAgents[cr.Agent]; !ok {
            return fmt.Errorf("unknown agent %q in direct dispatch", cr.Agent)
        }
    case "plan":
        if cr.Plan == nil || len(cr.Plan.Steps) == 0 {
            return fmt.Errorf("plan mode but no steps provided")
        }
        for _, step := range cr.Plan.Steps {
            if _, ok := subAgents[step.Agent]; !ok {
                return fmt.Errorf("unknown agent %q in plan step %q", step.Agent, step.ID)
            }
        }
    default:
        return fmt.Errorf("unknown classification mode %q", cr.Mode)
    }
    return nil
}
```

Note: Validation checks against the `subAgents` map (which includes `chat`), ensuring all four agents are valid for both direct dispatch and plan steps.

### 2.3 Updated File: `vv/agents/agents.go`

**Changes to `Agents` struct:**

```go
type Agents struct {
    Coder        *taskagent.Agent
    Chat         *taskagent.Agent
    Researcher   *taskagent.Agent
    Reviewer     *taskagent.Agent
    Orchestrator *OrchestratorAgent  // replaces Router and Planner
}
```

**Changes to `Create` function:**
- Remove `PlannerAgent` creation
- Remove `routeragent.Agent` creation
- Create `OrchestratorAgent` with:
  - The raw `llm` client for classification calls
  - The `planGen` taskagent (reused for plan summarization/aggregation only)
  - The sub-agents map including all four agents: coder, researcher, reviewer, **and chat**
  - `chatAgent` as fallback (not coder)
  - Working directory from `cfg.Tools.BashWorkingDir`
- Remove the deprecated `CreateRouter` function
- Remove `routeragent` import

```go
subAgents := map[string]agent.Agent{
    "coder":      coderAgent,
    "researcher": researcherAgent,
    "reviewer":   reviewerAgent,
    "chat":       chatAgent,
}

orchestratorAgent := NewOrchestratorAgent(
    agent.Config{
        ID:          "orchestrator",
        Name:        "Orchestrator Agent",
        Description: "Orchestrates user requests: classifies, dispatches, and aggregates",
    },
    llm,
    cfg.LLM.Model,
    subAgents,
    planGen,
    cfg.Memory.MaxConcurrency,
    chatAgent, // fallback is chat, not coder
    cfg.Tools.BashWorkingDir,
)
```

### 2.4 Updated File: `vv/agents/prompts.go`

**New constant:** `OrchestratorSystemPrompt`

```go
const OrchestratorSystemPrompt = `You are an orchestrator agent. You receive user instructions and decide how to fulfill them.

## Working Directory
The user is working in: {{.WorkingDir}}

## Available Agents
- "coder": Reads, writes, edits files, runs commands, searches codebases, debugs
- "researcher": Explores codebases, reads documentation, gathers information (read-only)
- "reviewer": Reviews code for correctness, style, performance, security
- "chat": General conversation, questions, explanations, brainstorming

## Response Format
You MUST respond with ONLY a JSON object. No other text.

### For simple tasks (single agent):
{"mode": "direct", "agent": "<agent_id>"}

### For complex tasks (multi-step):
{"mode": "plan", "plan": {"goal": "...", "steps": [{"id": "step_1", "description": "...", "agent": "coder", "depends_on": []}]}}

## Rules
1. Use "direct" mode for tasks that clearly map to one agent capability.
2. Use "plan" mode only when the task genuinely requires multiple distinct steps across different capabilities.
3. For plan steps, use "depends_on" to specify ordering. Steps without dependencies run in parallel.
4. Keep plans focused: typically 2-5 steps.
5. Default to "coder" for ambiguous coding tasks, "chat" for general questions.
`
```

Note: The `{{.WorkingDir}}` placeholder is rendered via `strings.Replace` (not Go `text/template`) to stay consistent with the plain-string prompt pattern used throughout the codebase.

**Remove `PlannerSystemPrompt`** -- it has no consumers after the Planner is deleted.

**Retain `PlanSummaryPrompt`** -- the `PlanAggregator` continues to use it.

### 2.5 Working Directory Context Injection

When dispatching to sub-agents (both direct mode and plan steps), the Orchestrator prepends a working directory context message to the request. This gives sub-agents awareness of the user's project location without modifying their system prompt templates.

```go
// enrichRequest prepends working directory context to a request for sub-agent dispatch.
func (o *OrchestratorAgent) enrichRequest(req *schema.RunRequest) *schema.RunRequest {
    if o.workingDir == "" {
        return req
    }
    msgs := make([]schema.Message, 0, len(req.Messages)+1)
    msgs = append(msgs, schema.NewUserMessage(
        fmt.Sprintf("Working directory: %s", o.workingDir),
    ))
    msgs = append(msgs, req.Messages...)
    return &schema.RunRequest{
        Messages:  msgs,
        SessionID: req.SessionID,
        Options:   req.Options,
        Metadata:  req.Metadata,
    }
}
```

For plan steps, the `buildNodes` InputMapper includes the original user goal as context:

```go
InputMapper: func(upstream map[string]*schema.RunResponse) (*schema.RunRequest, error) {
    var msgs []schema.Message
    // Add working directory context.
    if o.workingDir != "" {
        msgs = append(msgs, schema.NewUserMessage(
            fmt.Sprintf("Working directory: %s", o.workingDir),
        ))
    }
    // Add original user goal for context.
    msgs = append(msgs, schema.NewUserMessage(
        fmt.Sprintf("Original request: %s", plan.Goal),
    ))
    // Prepend upstream context if available.
    for _, depID := range stepCopy.DependsOn {
        if resp, ok := upstream[depID]; ok && resp != nil {
            msgs = append(msgs, resp.Messages...)
        }
    }
    // Add the step description as the task.
    msgs = append(msgs, schema.NewUserMessage(stepCopy.Description))
    return &schema.RunRequest{
        Messages:  msgs,
        SessionID: req.SessionID,
    }, nil
},
```

### 2.6 Updated File: `vv/main.go`

**Changes:**

1. **Capture working directory at startup** (before agent creation):
   ```go
   // After config load, before agent creation:
   if cfg.Tools.BashWorkingDir == "" {
       workingDir, err := os.Getwd()
       if err != nil {
           workingDir = "."
       }
       cfg.Tools.BashWorkingDir = workingDir
   }
   ```

2. **CLI mode:** Replace the `routeFn` / `routes` pattern with direct use of the Orchestrator:
   ```go
   // Before:
   routeFn := routeragent.LLMFunc(llmClient, cfg.LLM.Model, 4)
   routes := []routeragent.Route{...}
   app := vvcli.New(routeFn, routes, cfg, persistentMem)

   // After:
   app := vvcli.New(allAgents.Orchestrator, cfg, persistentMem)
   ```

3. **HTTP mode:** Register the Orchestrator instead of the Router:
   ```go
   svc.RegisterAgent(allAgents.Orchestrator)
   ```

4. **Remove imports** for `routeragent` from main.go.

### 2.7 Updated File: `vv/cli/cli.go`

**Changes to `App` struct:**

```go
type App struct {
    orchestrator  agent.StreamAgent  // replaces routeFn + routes
    cfg           *config.Config
    sessionID     string
    history       []schema.Message
    messages      []DisplayMessage
    program       *tea.Program
    persistentMem memory.Memory
}
```

**Changes to `New` function:**

```go
func New(
    orchestrator agent.StreamAgent,
    cfg *config.Config,
    persistentMem memory.Memory,
) *App {
    return &App{
        orchestrator:  orchestrator,
        cfg:           cfg,
        persistentMem: persistentMem,
    }
}
```

**Remove `selectAgent` method.** The `invokeAgent` method directly calls the orchestrator:

```go
func (m *model) invokeAgent(ctx context.Context, _ string) tea.Cmd {
    return func() tea.Msg {
        req := &schema.RunRequest{
            Messages:  m.app.history,
            SessionID: m.app.sessionID,
        }

        stream, err := m.app.orchestrator.RunStream(ctx, req)
        if err != nil {
            return streamDoneMsg{err: err}
        }

        go func() {
            // ... same stream consumption logic ...
        }()

        return nil
    }
}
```

**Remove imports** for `routeragent` from cli.go.

### 2.8 Updated File: `vv/config/config.go`

**Changes to `MemoryConfig`:**

Rename `MaxConcurrency` comment to reference the Orchestrator instead of the Planner:

```go
type MemoryConfig struct {
    Dir            string `yaml:"dir"`
    SessionWindow  int    `yaml:"session_window"`
    PersistentLoad bool   `yaml:"persistent_load"`
    MaxConcurrency int    `yaml:"max_concurrency"` // orchestrator DAG concurrency, default 2
    // TODO: MaxConcurrency is orchestration config, not memory config.
    // Move to a dedicated OrchestratorConfig in a future cleanup.
}
```

No structural changes needed -- the field stays the same, just the semantics shift.

### 2.9 File Removal: `vv/agents/planner.go`

The `PlannerAgent` struct and its methods are absorbed into the new `OrchestratorAgent`. The following are moved/reused:

| From planner.go | Destination |
|---|---|
| `Plan`, `PlanStep` structs | `orchestrator.go` (same package, reused as-is) |
| `parsePlan` | Removed -- validation logic integrated into `ClassifyResult.validate` |
| `extractJSON` | `orchestrator.go` (reused as-is) |
| `buildNodes`, `findTerminalNodes` | `orchestrator.go` (adapted with working directory and goal context injection) |
| `PlanAggregator` | `orchestrator.go` (reused as-is) |
| `PlannerAgent` struct | Removed |
| `NewPlannerAgent` | Removed |
| `PlannerAgent.Run` | Logic merged into `OrchestratorAgent.Run` |
| `PlannerAgent.RunStream` | Replaced by `OrchestratorAgent.RunStream` (two-path implementation) |
| `PlannerAgent.fallbackRun` | Adapted into `OrchestratorAgent` fallback logic (using chat agent) |

The file `planner.go` is deleted and replaced by `orchestrator.go`.

## 3. Data Models / Schemas

### 3.1 ClassifyResult (new)

```go
type ClassifyResult struct {
    Mode  string `json:"mode"`   // "direct" or "plan"
    Agent string `json:"agent"`  // agent ID for direct mode
    Plan  *Plan  `json:"plan"`   // plan for plan mode
}
```

### 3.2 Plan and PlanStep (unchanged, moved)

```go
type Plan struct {
    Goal  string     `json:"goal"`
    Steps []PlanStep `json:"steps"`
}

type PlanStep struct {
    ID          string   `json:"id"`
    Description string   `json:"description"`
    Agent       string   `json:"agent"`
    DependsOn   []string `json:"depends_on"`
}
```

### 3.3 Config changes

No new config fields beyond the semantic rename. The `Tools.BashWorkingDir` field already exists and will be populated at startup if empty.

## 4. API Contracts

### 4.1 Internal Agent Interface (unchanged)

The Orchestrator implements the same `agent.Agent` and `agent.StreamAgent` interfaces. No API changes for the HTTP service layer.

### 4.2 HTTP Service Impact

The HTTP service registers the Orchestrator with ID `"orchestrator"` instead of `"router"`. API consumers using `/v1/agents/router` will need to use `/v1/agents/orchestrator`. The Planner is no longer registered as a routable agent.

**Breaking change:** The agent ID changes from `"router"` to `"orchestrator"`. Any HTTP API consumers referencing `/v1/agents/router` must update their URLs. This is an intentional breaking change aligned with the architectural shift.

**Agent listing changes:**
- Before: router, coder, chat, researcher, reviewer (5 agents)
- After: orchestrator, coder, chat, researcher, reviewer (5 agents)

## 5. Implementation Plan

### Task 1: Create `OrchestratorAgent` and System Prompt

**Files:**
- Create `vv/agents/orchestrator.go`
- Update `vv/agents/prompts.go` (add `OrchestratorSystemPrompt`, remove `PlannerSystemPrompt`)

**Work:**
1. Define `OrchestratorAgent` struct with fields: `llm`, `model`, `subAgents`, `planGen`, `maxConcurrency`, `fallbackAgent`, `workingDir`
2. Add `NewOrchestratorAgent` constructor with compile-time interface checks
3. Define `ClassifyResult` struct with `validate` method
4. Implement `classifyTask` method: builds classification prompt via `strings.Replace`, calls `llm.ChatComplete` directly, parses JSON response, captures token usage
5. Implement `enrichRequest` method for working directory context injection
6. Implement `Run` method: classifyTask -> direct dispatch (with enriched request) or DAG execution; aggregate classification usage with sub-agent usage
7. Implement `RunStream` method: two-path -- direct mode proxies sub-agent stream, plan mode uses `agent.RunToStream`
8. Move `Plan`, `PlanStep`, `extractJSON`, `buildNodes` (adapted with goal context), `findTerminalNodes`, `PlanAggregator` from `planner.go` into `orchestrator.go`
9. Update `buildNodes` InputMapper to include working directory and original goal context
10. Add `OrchestratorSystemPrompt` constant to `prompts.go`; remove `PlannerSystemPrompt`
11. Add unit tests in `vv/agents/orchestrator_test.go`

### Task 2: Update `agents.go` -- Replace Agents Struct and Create Function

**Files:**
- Update `vv/agents/agents.go`

**Work:**
1. Replace `Planner *PlannerAgent` and `Router *routeragent.Agent` with `Orchestrator *OrchestratorAgent`
2. Update `Create` function:
   - Remove planner and router creation
   - Create `OrchestratorAgent` with raw `llm` client, sub-agents map including **chat**, and `chatAgent` as fallback
   - Pass working directory (from `cfg.Tools.BashWorkingDir`)
3. Remove `CreateRouter` function
4. Remove `routeragent` import
5. Update `vv/agents/agents_test.go`

### Task 3: Update CLI to Use Orchestrator Directly

**Files:**
- Update `vv/cli/cli.go`

**Work:**
1. Change `App` struct: replace `routeFn`/`routes` with `orchestrator agent.StreamAgent`
2. Update `New` constructor signature
3. Remove `selectAgent` method
4. Update `invokeAgent` to call `orchestrator.RunStream` directly
5. Remove `routeragent` import
6. Update `vv/cli/cli_test.go`

### Task 4: Update `main.go` Wiring (includes Working Directory Capture)

**Files:**
- Update `vv/main.go`

**Work:**
1. After config load but **before agent creation**, call `os.Getwd()` and set `cfg.Tools.BashWorkingDir` if empty
2. CLI mode: pass `allAgents.Orchestrator` to `vvcli.New` (remove routeFn/routes)
3. HTTP mode: register `allAgents.Orchestrator` instead of `allAgents.Router`
4. Remove `routeragent` import from main.go
5. Remove unused Planner-related code

### Task 5: Delete Planner Agent

**Files:**
- Delete `vv/agents/planner.go`
- Delete `vv/agents/planner_test.go`

**Work:**
1. Verify all code from `planner.go` is absorbed into `orchestrator.go`
2. Delete files

### Task 6: Update Integration Tests

**Files:**
- Update `vv/integrations/wiring_test.go`
- Update `vv/integrations/cli_test.go`
- Update `vv/integrations/agents_test.go`

**Work:**
1. Replace references to `Router` with `Orchestrator`
2. Update HTTP agent listing assertions (agent ID "orchestrator" instead of "router")
3. Update CLI test setup to use new `vvcli.New` signature
4. Verify agent count remains 5

### Task 7: Update Config Comment

**Files:**
- Update `vv/config/config.go`

**Work:**
1. Update `MaxConcurrency` comment to say "orchestrator" instead of "planner"
2. Add TODO comment noting the field should be moved out of MemoryConfig

## 6. Integration Test Plan

### Test 1: Orchestrator Direct Dispatch (Simple Task)

**Objective:** Verify that simple tasks are dispatched to a single agent without plan generation, with streaming preserved.

**Setup:** Mock LLM that returns `{"mode": "direct", "agent": "coder"}` for classification, and a mock coder agent implementing StreamAgent.

**Steps:**
1. Create OrchestratorAgent with mock sub-agents
2. Send a simple request: "read file main.go"
3. Assert the coder agent received the request
4. Assert no DAG execution occurred
5. Assert response is the coder's response
6. Assert RunStream returns the sub-agent's stream (not a RunToStream wrapper)

### Test 2: Orchestrator Plan Execution (Complex Task)

**Objective:** Verify that complex tasks trigger plan generation and DAG execution.

**Setup:** Mock LLM that returns a plan with 2 steps for classification.

**Steps:**
1. Create OrchestratorAgent with mock sub-agents
2. Send a complex request: "research the codebase, then refactor the config module"
3. Assert plan was parsed with 2 steps
4. Assert both researcher and coder agents were invoked
5. Assert step requests include working directory context and original goal
6. Assert results were aggregated into a single response

### Test 3: Orchestrator Fallback on Classification Failure

**Objective:** Verify fallback to chat agent (not coder) when classification fails.

**Setup:** Mock LLM that returns an error or invalid JSON for classification.

**Steps:**
1. Create OrchestratorAgent with mock sub-agents and chat as fallback
2. Send a request
3. Assert **chat** agent received the request as fallback
4. Assert response is the chat agent's response

### Test 4: Working Directory Capture and Propagation

**Objective:** Verify that the working directory is captured, stored, and propagated to sub-agents.

**Steps:**
1. Capture CWD before test
2. Load config (with empty `BashWorkingDir`)
3. Apply working directory logic from main.go
4. Assert `cfg.Tools.BashWorkingDir` equals the captured CWD
5. Create OrchestratorAgent with the working directory
6. Trigger a direct dispatch and assert the sub-agent request contains the working directory context message

### Test 5: CLI Wiring with Orchestrator

**Objective:** Verify the CLI correctly invokes the Orchestrator.

**Setup:** Create a mock OrchestratorAgent that implements StreamAgent.

**Steps:**
1. Create `vvcli.App` with the mock orchestrator
2. Verify the App struct holds the orchestrator reference
3. (Manual/smoke) Verify invokeAgent calls orchestrator.RunStream

### Test 6: HTTP Service with Orchestrator

**Objective:** Verify the HTTP service registers the Orchestrator correctly.

**Steps:**
1. Create all agents via `agents.Create`
2. Register `allAgents.Orchestrator` with the service
3. Assert `/v1/agents` lists "orchestrator" (not "router")
4. Assert agent count is 5 (orchestrator, coder, chat, researcher, reviewer)

### Test 7: Parallel Step Execution

**Objective:** Verify independent plan steps execute in parallel.

**Setup:** Mock LLM returns a plan with 2 independent steps (no dependencies).

**Steps:**
1. Create OrchestratorAgent with mock sub-agents that record invocation timestamps
2. Send request triggering a plan with parallel steps
3. Assert both steps started within a short window (not sequentially)
4. Assert both results are in the aggregated response

### Test 8: Plan Step Failure Handling

**Objective:** Verify the Orchestrator handles sub-task failures gracefully.

**Setup:** Mock LLM returns a 2-step plan. One sub-agent returns an error.

**Steps:**
1. Create OrchestratorAgent with one failing mock sub-agent
2. Send request
3. Assert the failing step is skipped (Optional=true in DAG nodes)
4. Assert the response includes results from the successful step
5. Assert no panic or unhandled error

### Test 9: Classification Token Usage Aggregation

**Objective:** Verify token usage from the classification LLM call is included in the response.

**Steps:**
1. Create OrchestratorAgent with a mock LLM that reports usage
2. Send a request triggering direct dispatch to a mock sub-agent that also reports usage
3. Assert the final response's usage includes both classification and sub-agent tokens

### Test 10: Chat Agent in Plan Steps

**Objective:** Verify the chat agent can be used as a valid agent in plan steps.

**Setup:** Mock LLM returns a plan with a step assigned to "chat".

**Steps:**
1. Create OrchestratorAgent with all four sub-agents including chat
2. Send request triggering the plan
3. Assert the chat agent is invoked for its assigned step
4. Assert the response includes chat agent output
