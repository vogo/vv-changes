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

1. **Single LLM call for classification + optional planning.** The Orchestrator uses one LLM call that returns a structured JSON response indicating either a direct dispatch (single agent) or a multi-step plan. This avoids the current two-hop latency (Router LLM call + Planner LLM call).

2. **Reuse existing `orchestrate.ExecuteDAG`.** The Planner already uses the vage framework's DAG engine. The Orchestrator continues to use `orchestrate.ExecuteDAG` for multi-step plans, so no changes to the orchestrate package are needed.

3. **Working directory captured at startup via `os.Getwd()`.** Stored in config and injected into system prompts and the bash tool's working directory.

4. **The Orchestrator is a custom agent in `vv/agents/`**, not a new agent type in the vage framework. It implements `agent.Agent` and `agent.StreamAgent` directly, keeping vage unchanged.

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
    planGen        *taskagent.Agent        // LLM agent for plan generation / summarization
    maxConcurrency int
    fallbackAgent  agent.Agent             // chat agent as fallback
    workingDir     string                  // captured CWD
}
```

**Interface compliance:**
- Implements `agent.Agent` (Run)
- Implements `agent.StreamAgent` (RunStream)

**Run method flow:**

```
Run(ctx, req):
  1. Call classifyTask(ctx, req) -> ClassifyResult
  2. If ClassifyResult.Mode == "direct":
       a. Look up ClassifyResult.Agent in subAgents
       b. Dispatch directly: subAgent.Run(ctx, req)
       c. Return response
  3. If ClassifyResult.Mode == "plan":
       a. Parse Plan from ClassifyResult
       b. Build DAG nodes (reuse buildNodes logic from planner.go)
       c. Execute via orchestrate.ExecuteDAG
       d. Return aggregated result
  4. On any failure: fall back to chat agent
```

**RunStream method:** Uses `agent.RunToStream(ctx, orchestrator, req)` -- same pattern as the current PlannerAgent.

### 2.2 Classification / Planning via Single LLM Call

The `classifyTask` method makes one LLM call with a system prompt that instructs the LLM to return a JSON response.

```go
// ClassifyResult is the structured LLM response.
type ClassifyResult struct {
    Mode  string     `json:"mode"`  // "direct" or "plan"
    Agent string     `json:"agent"` // only for mode="direct": coder/researcher/reviewer/chat
    Plan  *Plan      `json:"plan"`  // only for mode="plan"
}
```

The classification prompt (new constant `OrchestratorSystemPrompt`) instructs the LLM to:
- Analyze the user instruction
- If it maps to a single agent capability, return `{"mode": "direct", "agent": "<id>"}`
- If it requires multiple steps, return `{"mode": "plan", "plan": {"goal": "...", "steps": [...]}}`

This merges the Router's classification and the Planner's plan generation into one call.

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
  - The same `planGen` taskagent (reused for plan generation and summarization)
  - The same sub-agents map (coder, researcher, reviewer)
  - Additionally include `chat` in sub-agents
  - `chatAgent` as fallback
  - Working directory from config
- Remove the deprecated `CreateRouter` function

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

**Existing prompts** (`PlannerSystemPrompt`, `PlanSummaryPrompt`) are retained since the plan generation sub-agent and aggregator still use them internally. They can be removed in a follow-up cleanup if desired.

### 2.5 Updated File: `vv/main.go`

**Changes:**

1. **Capture working directory at startup:**
   ```go
   workingDir, err := os.Getwd()
   if err != nil {
       workingDir = "."
   }
   cfg.Tools.BashWorkingDir = workingDir  // set if not already configured
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

### 2.6 Updated File: `vv/cli/cli.go`

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

### 2.7 Updated File: `vv/config/config.go`

**Changes to `MemoryConfig`:**

Rename `MaxConcurrency` comment to reference the Orchestrator instead of the Planner:

```go
type MemoryConfig struct {
    Dir            string `yaml:"dir"`
    SessionWindow  int    `yaml:"session_window"`
    PersistentLoad bool   `yaml:"persistent_load"`
    MaxConcurrency int    `yaml:"max_concurrency"` // orchestrator DAG concurrency, default 2
}
```

No structural changes needed -- the field stays the same, just the semantics shift.

### 2.8 File Removal: `vv/agents/planner.go`

The `PlannerAgent` struct and its methods are absorbed into the new `OrchestratorAgent`. The following are moved/reused:

| From planner.go | Destination |
|---|---|
| `Plan`, `PlanStep` structs | `orchestrator.go` (same package, reused as-is) |
| `parsePlan`, `extractJSON` | `orchestrator.go` (reused as-is) |
| `buildNodes`, `findTerminalNodes` | `orchestrator.go` (reused as-is) |
| `PlanAggregator` | `orchestrator.go` (reused as-is) |
| `PlannerAgent` struct | Removed |
| `NewPlannerAgent` | Removed |
| `PlannerAgent.Run` | Logic merged into `OrchestratorAgent.Run` |
| `PlannerAgent.RunStream` | Replaced by `OrchestratorAgent.RunStream` |
| `PlannerAgent.fallbackRun` | Adapted into `OrchestratorAgent` fallback logic |

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

**Agent listing changes:**
- Before: router, coder, chat, researcher, reviewer (5 agents)
- After: orchestrator, coder, chat, researcher, reviewer (5 agents)

## 5. Implementation Plan

### Task 1: Create `OrchestratorAgent` and System Prompt

**Files:**
- Create `vv/agents/orchestrator.go`
- Update `vv/agents/prompts.go` (add `OrchestratorSystemPrompt`)

**Work:**
1. Define `OrchestratorAgent` struct with fields: `llm`, `model`, `subAgents`, `planGen`, `maxConcurrency`, `fallbackAgent`, `workingDir`
2. Define `ClassifyResult` struct
3. Implement `classifyTask` method: builds classification prompt, calls LLM, parses JSON response
4. Implement `Run` method: classifyTask -> direct dispatch or DAG execution
5. Implement `RunStream` method: wraps Run via `agent.RunToStream`
6. Move `Plan`, `PlanStep`, `parsePlan`, `extractJSON`, `buildNodes`, `findTerminalNodes`, `PlanAggregator` from `planner.go` into `orchestrator.go`
7. Add `OrchestratorSystemPrompt` constant to `prompts.go`
8. Add unit tests in `vv/agents/orchestrator_test.go`

### Task 2: Update `agents.go` -- Replace Agents Struct and Create Function

**Files:**
- Update `vv/agents/agents.go`

**Work:**
1. Replace `Planner *PlannerAgent` and `Router *routeragent.Agent` with `Orchestrator *OrchestratorAgent`
2. Update `Create` function:
   - Remove planner and router creation
   - Create `OrchestratorAgent` with sub-agents map including chat
   - Pass working directory (from `cfg.Tools.BashWorkingDir`)
3. Remove `CreateRouter` function
4. Remove `routeragent` import
5. Update `vv/agents/agents_test.go`

### Task 3: Capture Working Directory at Startup

**Files:**
- Update `vv/main.go`

**Work:**
1. After config load, call `os.Getwd()` and set `cfg.Tools.BashWorkingDir` if empty
2. This ensures the bash tool and all agents know the CWD

### Task 4: Update CLI to Use Orchestrator Directly

**Files:**
- Update `vv/cli/cli.go`

**Work:**
1. Change `App` struct: replace `routeFn`/`routes` with `orchestrator agent.StreamAgent`
2. Update `New` constructor signature
3. Remove `selectAgent` method
4. Update `invokeAgent` to call `orchestrator.RunStream` directly
5. Remove `routeragent` import
6. Update `vv/cli/cli_test.go`

### Task 5: Update `main.go` Wiring

**Files:**
- Update `vv/main.go`

**Work:**
1. CLI mode: pass `allAgents.Orchestrator` to `vvcli.New` (remove routeFn/routes)
2. HTTP mode: register `allAgents.Orchestrator` instead of `allAgents.Router`
3. Remove `routeragent` import from main.go
4. Remove unused Planner-related code

### Task 6: Delete Planner Agent

**Files:**
- Delete `vv/agents/planner.go`
- Delete `vv/agents/planner_test.go`

**Work:**
1. Verify all code from `planner.go` is absorbed into `orchestrator.go`
2. Delete files

### Task 7: Update Integration Tests

**Files:**
- Update `vv/integrations/wiring_test.go`
- Update `vv/integrations/cli_test.go`
- Update `vv/integrations/agents_test.go`

**Work:**
1. Replace references to `Router` with `Orchestrator`
2. Update HTTP agent listing assertions (agent ID "orchestrator" instead of "router")
3. Update CLI test setup to use new `vvcli.New` signature
4. Verify agent count remains 5

### Task 8: Update Config Comment (minor)

**Files:**
- Update `vv/config/config.go`

**Work:**
1. Update `MaxConcurrency` comment to say "orchestrator" instead of "planner"

## 6. Integration Test Plan

### Test 1: Orchestrator Direct Dispatch (Simple Task)

**Objective:** Verify that simple tasks are dispatched to a single agent without plan generation.

**Setup:** Mock LLM that returns `{"mode": "direct", "agent": "coder"}` for classification, and a mock coder agent.

**Steps:**
1. Create OrchestratorAgent with mock sub-agents
2. Send a simple request: "read file main.go"
3. Assert the coder agent received the request
4. Assert no DAG execution occurred
5. Assert response is the coder's response

### Test 2: Orchestrator Plan Execution (Complex Task)

**Objective:** Verify that complex tasks trigger plan generation and DAG execution.

**Setup:** Mock LLM that returns a plan with 2 steps for classification.

**Steps:**
1. Create OrchestratorAgent with mock sub-agents
2. Send a complex request: "research the codebase, then refactor the config module"
3. Assert plan was parsed with 2 steps
4. Assert both researcher and coder agents were invoked
5. Assert results were aggregated into a single response

### Test 3: Orchestrator Fallback on Classification Failure

**Objective:** Verify fallback to chat agent when classification fails.

**Setup:** Mock LLM that returns an error or invalid JSON for classification.

**Steps:**
1. Create OrchestratorAgent with mock sub-agents
2. Send a request
3. Assert chat agent received the request as fallback
4. Assert response is the chat agent's response

### Test 4: Working Directory Capture

**Objective:** Verify that the working directory is captured and propagated.

**Steps:**
1. Capture CWD before test
2. Load config (with empty `BashWorkingDir`)
3. Apply working directory logic from main.go
4. Assert `cfg.Tools.BashWorkingDir` equals the captured CWD

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
