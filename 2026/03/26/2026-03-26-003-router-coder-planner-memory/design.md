# Technical Design: RouterAgent Enhancement, Coder TaskAgent, Planner WorkflowAgent, Three-Level Memory

## 1. Architecture Overview

This design introduces four changes to the vv application, building entirely on existing vage framework primitives:

1. **Expanded RouterAgent** -- The existing `routeragent.Agent` is reconfigured with five routes (coder, planner, researcher, reviewer, chat) instead of two. The `routeragent.LLMFunc` route function already supports N routes; only the route table and prompt need updating.

2. **Researcher and Reviewer TaskAgents** -- Two new `taskagent.Agent` instances with restricted tool registries and specialized system prompts. They follow the same construction pattern as the existing coder and chat agents.

3. **Planner WorkflowAgent** -- A two-phase agent: phase 1 uses `taskagent.Agent` (with the LLM) to generate a structured plan (JSON), phase 2 constructs `orchestrate.Node` slices and runs them via `orchestrate.ExecuteDAG`. The planner is implemented as a dedicated struct implementing both `agent.Agent` and `agent.StreamAgent`.

4. **Three-Level Memory Integration** -- Wire the existing `memory.Manager` (working/session/store tiers) into vv. Add a `FileStore` implementation of `memory.Store` for persistent memory. Add `/memory` CLI commands and HTTP endpoints.

### Architectural Principles

- **Reuse framework primitives** -- All agent types, orchestrate engine, memory tiers, and tool registries already exist in vage. This design composes them; it does not create new framework abstractions.
- **Changes are in the vv application layer** -- The vage framework receives no new types. `FileStore` is placed in vv because it is application-specific filesystem persistence.
- **Backward compatible** -- Existing CLI and HTTP interfaces continue to work. The router now has more routes, but the interface is identical.

### Component Diagram

```
User Input
    |
    v
+-------------------+
|   RouterAgent     |  routeragent.Agent (LLMFunc with 5 routes)
+-------------------+
    |  |  |  |  |
    v  v  v  v  v
+------+ +--------+ +----------+ +----------+ +------+
|Coder | |Planner | |Researcher| |Reviewer  | |Chat  |
|Agent | |Agent   | |Agent     | |Agent     | |Agent |
+------+ +--------+ +----------+ +----------+ +------+
   |        |            |            |           |
   |     +--+--+         |            |           |
   |     |Plan |         |            |           |
   |     |Gen  |         |            |           |
   |     +--+--+         |            |           |
   |        |            |            |           |
   |     +--v--+         |            |           |
   |     |DAG  |         |            |           |
   |     |Exec |         |            |           |
   |     +-----+         |            |           |
   |                     |            |           |
   +--- Full Tools ------+-- RO Tools-+-- RO+Bash-+-- No Tools
   |
+--v-------------------+
| Memory Manager       |
| Working -> Session   |
|         -> Store     |
| (FileStore backend)  |
+-----------------------+
```

## 2. Component Design

### 2.1 Expanded RouterAgent

**Location:** `vv/agents/agents.go` (update `CreateRouter`)

The router remains a `routeragent.Agent`. The change is purely in the route table and the fallback index.

```go
// Updated route table (index order matters for LLMFunc)
routes := []routeragent.Route{
    {Agent: coder,     Description: "Handles code-related tasks: reading, writing, editing files, running commands, debugging"},
    {Agent: planner,   Description: "Handles complex multi-step tasks: project setup, large refactors, multi-file coordinated changes"},
    {Agent: researcher,Description: "Handles research tasks: codebase exploration, documentation lookup, information gathering"},
    {Agent: reviewer,  Description: "Handles review tasks: code review, design review, quality assessment"},
    {Agent: chat,      Description: "Handles general conversation, questions, explanations, brainstorming"},
}
// fallback index = 4 (chat)
```

The `LLMFunc` route function already builds a numbered prompt from route descriptions and parses the LLM's integer response. No changes needed in vage.

**Important:** The router's `RunStream` uses `agent.RunToStream` which wraps the entire `Run` call, losing sub-agent streaming events. The CLI works around this by calling `selectAgent` to get the routed agent and calling `RunStream` on it directly. This pattern is preserved for all new agents -- the CLI always streams from the selected sub-agent, not from the router.

### 2.2 Researcher Agent

**Location:** `vv/agents/agents.go` (new creation in `Create` function)

A standard `taskagent.Agent` with:
- **System prompt:** Research-focused instructions (explore, read, summarize).
- **Tool registry:** A subset registry containing only `read`, `glob`, `grep`. NOT wrapped with `WrapRegistry` (no confirmation needed for read-only tools).
- **Max iterations:** Same as coder (configurable).

```go
// Read-only tool registry (not wrapped with confirmation)
readOnlyReg, err := tools.RegisterReadOnly(cfg.Tools)

researcherAgent := taskagent.New(
    agent.Config{ID: "researcher", Name: "Researcher Agent", ...},
    taskagent.WithChatCompleter(llm),
    taskagent.WithModel(cfg.LLM.Model),
    taskagent.WithToolRegistry(readOnlyReg),
    taskagent.WithSystemPrompt(prompt.StringPrompt(ResearcherSystemPrompt)),
    taskagent.WithMaxIterations(cfg.Agents.MaxIterations),
)
```

### 2.3 Reviewer Agent

**Location:** `vv/agents/agents.go`

Same as researcher but with `bash` added (for running tests/linters) and a review-focused system prompt. The bash tool within the review registry is NOT wrapped with confirmation since it is scoped to test/lint commands.

```go
reviewReg, err := tools.RegisterReviewTools(cfg.Tools) // read + glob + grep + bash

reviewerAgent := taskagent.New(
    agent.Config{ID: "reviewer", Name: "Reviewer Agent", ...},
    taskagent.WithChatCompleter(llm),
    taskagent.WithModel(cfg.LLM.Model),
    taskagent.WithToolRegistry(reviewReg),
    taskagent.WithSystemPrompt(prompt.StringPrompt(ReviewerSystemPrompt)),
    taskagent.WithMaxIterations(cfg.Agents.MaxIterations),
)
```

### 2.4 Tool Registry Helpers

**Location:** `vv/tools/tools.go` (add new functions)

```go
// RegisterReadOnly creates a registry with read, glob, grep only.
func RegisterReadOnly(cfg config.ToolsConfig) (*tool.Registry, error)

// RegisterReviewTools creates a registry with read, glob, grep, bash.
func RegisterReviewTools(cfg config.ToolsConfig) (*tool.Registry, error)
```

These reuse the same tool registration functions (e.g., `readtool.Register`, `greptool.Register`) but selectively register only the desired tools. They follow the exact error-wrapping pattern from the existing `Register` function.

### 2.5 Planner Agent

**Location:** `vv/agents/planner.go` (new file)

The planner is the most complex new component. It is implemented as a dedicated struct that implements both `agent.Agent` and `agent.StreamAgent` (required because the CLI casts routed agents to `StreamAgent`).

**Phase 1 -- Plan Generation:**
Uses a `taskagent.Agent` (the "plan generator") with no tools, a specialized system prompt that instructs the LLM to output a JSON plan, and `maxIterations=1`.

**Phase 2 -- Plan Execution:**
Parses the JSON plan into `orchestrate.Node` slices, constructs a DAG, and runs it via `orchestrate.ExecuteDAG`. Each node's `Runner` is one of the sub-agents (typically coder).

```go
type PlannerAgent struct {
    agent.Base
    planGen       *taskagent.Agent           // LLM for plan generation
    subAgents     map[string]agent.Agent     // available delegation targets
    maxConcurrency int                       // configurable, default from config
}

// Compile-time checks.
var (
    _ agent.Agent       = (*PlannerAgent)(nil)
    _ agent.StreamAgent = (*PlannerAgent)(nil)
)

func (p *PlannerAgent) Run(ctx context.Context, req *schema.RunRequest) (*schema.RunResponse, error) {
    // Phase 1: Generate plan
    planResp, err := p.planGen.Run(ctx, planRequest(req))
    plan, err := parsePlan(planResp)

    // Phase 2: Build and execute DAG
    nodes := p.buildNodes(plan, req)
    dagCfg := orchestrate.DAGConfig{
        MaxConcurrency: p.maxConcurrency,
        ErrorStrategy:  orchestrate.Skip,
        Aggregator:     NewPlanAggregator(p.planGen),
    }
    result, err := orchestrate.ExecuteDAG(ctx, dagCfg, nodes, req)

    return result.FinalOutput, err
}

// RunStream wraps Run with lifecycle events for streaming compatibility.
func (p *PlannerAgent) RunStream(ctx context.Context, req *schema.RunRequest) (*schema.RunStream, error) {
    return agent.RunToStream(ctx, p, req), nil
}
```

#### Node Input Mapping

Each DAG node must set an `InputMapper` to pass the step description to the sub-agent as a user message, incorporating upstream results as context:

```go
func (p *PlannerAgent) buildNodes(plan *Plan, req *schema.RunRequest) []orchestrate.Node {
    nodes := make([]orchestrate.Node, len(plan.Steps))

    for i, step := range plan.Steps {
        stepCopy := step // capture for closure
        subAgent := p.subAgents[step.Agent]

        nodes[i] = orchestrate.Node{
            ID:     step.ID,
            Runner: subAgent,
            Deps:   step.DependsOn,
            InputMapper: func(upstream map[string]*schema.RunResponse) (*schema.RunRequest, error) {
                var msgs []schema.Message
                // Prepend upstream context if available
                for _, depID := range stepCopy.DependsOn {
                    if resp, ok := upstream[depID]; ok && resp != nil {
                        for _, m := range resp.Messages {
                            msgs = append(msgs, m)
                        }
                    }
                }
                // Add the step description as the user message
                msgs = append(msgs, schema.NewUserMessage(stepCopy.Description))
                return &schema.RunRequest{
                    Messages:  msgs,
                    SessionID: req.SessionID,
                }, nil
            },
        }
    }

    return nodes
}
```

#### DAG Connectivity Constraint

The `orchestrate.ValidateDAG` function requires all nodes to form a single connected component. Plans with independent (disconnected) sub-tasks would fail validation. To handle this, the `buildNodes` method adds a synthetic terminal "summary" node that depends on all leaf nodes (nodes with no downstream dependents), ensuring connectivity:

```go
// After building all plan step nodes, add a summary node
terminalIDs := findTerminalNodes(nodes)
if len(terminalIDs) > 1 {
    summaryNode := orchestrate.Node{
        ID:     "summary",
        Runner: p.planGen, // use the plan generator for summarization
        Deps:   terminalIDs,
        InputMapper: func(upstream map[string]*schema.RunResponse) (*schema.RunRequest, error) {
            // Combine all upstream results into a summarization request
            ...
        },
    }
    nodes = append(nodes, summaryNode)
}
```

#### Plan JSON Schema

The LLM is prompted to produce a JSON plan. The schema is kept minimal:

```json
{
  "goal": "string -- restated goal",
  "steps": [
    {
      "id": "step_1",
      "description": "What this step accomplishes",
      "agent": "coder",
      "depends_on": []
    },
    {
      "id": "step_2",
      "description": "...",
      "agent": "coder",
      "depends_on": ["step_1"]
    }
  ]
}
```

#### Plan Generation Prompt

The planner's system prompt instructs the LLM to:
1. Analyze the user's request and break it into atomic sub-tasks.
2. Identify dependencies between sub-tasks (what must complete before what).
3. Output a valid JSON plan following the schema above.
4. Use only the available agent IDs: `coder`, `researcher`, `reviewer`.
5. Ensure the plan forms a connected graph -- every independent sub-task should have at least one dependency link to another sub-task, or be part of a chain.

#### Plan Aggregation

After DAG execution, a custom `Aggregator` synthesizes the sub-task outputs into a coherent response. The planner LLM itself is called with a "summarize the completed work" prompt.

```go
type PlanAggregator struct {
    summarizer *taskagent.Agent
}

func (a *PlanAggregator) Aggregate(ctx context.Context, results map[string]*schema.RunResponse) (*schema.RunResponse, error) {
    // Build summary prompt from all step results
    // Call summarizer LLM to produce coherent response
}
```

#### Error Handling

- If plan generation fails (invalid JSON, empty plan), fall back to routing the original request to the coder agent. When falling back, log a warning and prepend context to the coder's request: "Note: task planning failed, executing as a single task."
- If a step fails during DAG execution, the `ErrorStrategy: Skip` allows the DAG to continue. The aggregator notes the failure in the summary.
- Re-planning on failure is deferred to a future iteration to keep scope manageable.

#### Known Limitation: Streaming

The planner's `RunStream` uses `agent.RunToStream`, which wraps the entire `Run` into AgentStart/AgentEnd events. Users do not see streaming text deltas during plan generation or sub-task execution. This is acceptable for the initial implementation. A future iteration could emit custom progress events during plan execution.

### 2.6 Three-Level Memory

#### 2.6.1 FileStore (New in vv)

**Location:** `vv/memory/filestore.go` (new file in a new `vv/memory` package)

The `FileStore` is placed in vv (not vage) because it is application-specific filesystem persistence. The vage framework provides the `Store` interface and the in-memory `MapStore`; `FileStore` is a concrete backend at the same abstraction level and belongs in the application layer.

```go
// FileStore persists key-value pairs as individual JSON files in a directory.
// Keys are sanitized to safe filenames. The store is not safe for concurrent
// use; wrap with PersistentMemory (which uses syncMemory) for thread safety.
type FileStore struct {
    dir string
}

// fileRecord is the JSON-serialized format of a stored entry.
type fileRecord struct {
    Key       string    `json:"key"`
    Value     string    `json:"value"`     // stored as string content
    Namespace string    `json:"namespace"` // extracted from key prefix
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    TTL       int64     `json:"ttl"`
}
```

File layout:
```
~/.vv/memory/
    project/           # namespace directory
        conventions.json
        architecture.json
    user/
        preferences.json
    conventions/
        go_style.json
```

Key encoding: `namespace:key` maps to `<dir>/<namespace>/<key>.json`. The namespace is the prefix before the first `:`.

Interface compliance:
```go
var _ memory.Store = (*FileStore)(nil)

func NewFileStore(dir string) (*FileStore, error)
func (s *FileStore) Get(ctx context.Context, key string) (any, bool, error)
func (s *FileStore) Set(ctx context.Context, key string, value any, ttl int64) error
func (s *FileStore) Delete(ctx context.Context, key string) error
func (s *FileStore) List(ctx context.Context, prefix string) ([]memory.StoreEntry, error)
func (s *FileStore) Clear(ctx context.Context) error
```

Note: `FileStore` imports `memory.Store` and `memory.StoreEntry` from vage. The import direction is `vv/memory` -> `vage/memory`, which is correct (application depends on framework).

#### 2.6.2 Memory Manager Wiring

**Location:** `vv/agents/agents.go` and `vv/main.go`

At application startup:

```go
// Create persistent memory with FileStore backend
fileStore, _ := vvmemory.NewFileStore(cfg.Memory.Dir) // default ~/.vv/memory/
persistentMem := memory.NewPersistentMemoryWithStore(fileStore)

// Create session memory (in-memory, per session)
sessionMem := memory.NewSessionMemory("router", sessionID)

// Create memory manager
memMgr := memory.NewManager(
    memory.WithSession(sessionMem),
    memory.WithStore(persistentMem),
    memory.WithPromoter(memory.PromoteAll()),
    memory.WithCompressor(memory.NewSlidingWindowCompressor(cfg.Memory.SessionWindow)),
)

// Pass to agents that need memory
coderAgent := taskagent.New(..., taskagent.WithMemory(memMgr))
```

#### 2.6.3 Working Memory

Already implemented in vage. The `taskagent.Agent.storeAndPromoteMessages` method creates a `WorkingMemory` per run, stores request/response messages, and promotes them to session memory via the manager. No changes needed.

#### 2.6.4 Session Memory -- CLI Integration

**Important design decision:** The CLI currently maintains conversation history in `App.history` and passes the full history in `RunRequest.Messages`. The `taskagent.Agent.buildInitialMessages` also loads history from the memory manager's session tier. If both are populated, messages would be duplicated.

**Chosen approach:** Wire the memory manager and change the CLI to pass only the latest user message in `RunRequest.Messages` (not the full history). The memory manager's session tier provides historical context via the `buildInitialMessages` path. This enables the sliding window compressor to manage context length automatically.

Changes to `cli.go`:
- Remove `App.history` field.
- In `handleSubmit`, pass only the new user message in `req.Messages` (a single-element slice).
- In `handleStreamDone`, do NOT append to history (the memory manager handles this via `storeAndPromoteMessages`).

#### 2.6.5 Persistent Memory

Loaded at each request and injected into agent context as a system prompt suffix. The persistent memory entries are formatted as a knowledge block appended to the system prompt.

**Dynamic injection approach:** Because persistent memory may change during a session (via `/memory set` commands), it must not be baked into the agent at creation time. Instead, use a dynamic `prompt.PromptTemplate` that loads persistent memory on each `Render` call:

```go
type PersistentMemoryPrompt struct {
    basePrompt string
    store      memory.Memory // the persistent memory instance
}

func (p *PersistentMemoryPrompt) Render(ctx context.Context, data any) (string, error) {
    entries, err := p.store.List(ctx, "")
    if err != nil {
        return p.basePrompt, nil // degrade gracefully
    }
    if len(entries) == 0 {
        return p.basePrompt, nil
    }
    return p.basePrompt + "\n\n" + formatPersistentMemory(entries), nil
}
```

### 2.7 Configuration Changes

**Location:** `vv/config/config.go`

```go
type Config struct {
    // ... existing fields ...
    Memory MemoryConfig `yaml:"memory"`
}

type MemoryConfig struct {
    Dir             string `yaml:"dir"`              // default ~/.vv/memory/
    SessionWindow   int    `yaml:"session_window"`   // sliding window size, default 50
    PersistentLoad  bool   `yaml:"persistent_load"`  // load at startup, default true
    MaxConcurrency  int    `yaml:"max_concurrency"`  // planner DAG concurrency, default 2
}
```

Defaults applied in `applyDefaults`:
```go
func applyDefaults(cfg *Config) {
    // ... existing defaults ...

    if cfg.Memory.SessionWindow == 0 {
        cfg.Memory.SessionWindow = 50
    }
    if cfg.Memory.Dir == "" {
        cfg.Memory.Dir = filepath.Join(DefaultDir(), "memory")
    }
    if cfg.Memory.MaxConcurrency == 0 {
        cfg.Memory.MaxConcurrency = 2
    }
    // PersistentLoad defaults to true (zero value is false, handle explicitly)
}
```

Default `vv.yaml` addition:
```yaml
memory:
    dir: ""               # empty = ~/.vv/memory/
    session_window: 50
    persistent_load: true
    max_concurrency: 2
```

### 2.8 CLI Memory Commands

**Location:** `vv/cli/cli.go` (extend command handling) and `vv/cli/memory.go` (new file)

New slash commands handled in the `handleSubmit` method after the exit check but before routing to agents:

| Command | Action |
|---------|--------|
| `/memory list [namespace]` | List entries from `FileStore`, optionally filtered by namespace prefix |
| `/memory show <namespace>:<key>` | Read and display a single entry |
| `/memory set <namespace>:<key>` | Prompt for content (multiline input mode), then write to `FileStore` |
| `/memory delete <namespace>:<key>` | Delete entry from `FileStore` |

These commands operate directly on the `PersistentMemory` wrapper and display results as system messages in the TUI. They do not invoke any agent.

Implementation approach: Add a `handleCommand` method to the CLI model that checks for `/memory` prefix before routing to agents:

```go
func (m *model) handleCommand(input string) (handled bool) {
    parts := strings.Fields(input)
    if len(parts) == 0 || parts[0] != "/memory" {
        return false
    }
    if len(parts) < 2 {
        m.appendSystemMessage("Usage: /memory list|show|set|delete ...")
        return true
    }
    switch parts[1] {
    case "list": ...
    case "show": ...
    case "set":  ...
    case "delete": ...
    default:
        m.appendSystemMessage("Unknown memory command: " + parts[1])
    }
    return true
}
```

Updated `handleSubmit`:
```go
func (m *model) handleSubmit() (tea.Model, tea.Cmd) {
    input := strings.TrimSpace(m.textarea.Value())
    if input == "" {
        return m, nil
    }
    if isExitCommand(input) {
        return m, tea.Quit
    }
    // Handle /memory commands before agent routing
    if m.handleCommand(input) {
        m.textarea.Reset()
        return m, nil
    }
    // ... existing agent routing logic ...
}
```

### 2.9 CLI App Struct Changes

**Location:** `vv/cli/cli.go`

The `App` struct currently holds `coder` and `chat` fields, but these are only used to pass into the constructor and are not referenced elsewhere. The routing is fully handled by `routeFn` + `routes`. The struct is updated to:

```go
type App struct {
    routeFn        routeragent.RouteFunc
    routes         []routeragent.Route
    cfg            *config.Config
    sessionID      string
    messages       []DisplayMessage
    program        *tea.Program
    persistentMem  memory.Memory // for /memory commands
}
```

The `coder` and `chat` fields are removed. The `New` function signature is updated:

```go
func New(
    routeFn routeragent.RouteFunc,
    routes []routeragent.Route,
    cfg *config.Config,
    persistentMem memory.Memory,
) *App
```

### 2.10 HTTP Memory Endpoints

**Location:** `vv/main.go` (register routes on the mux)

New endpoints added directly in `main.go` as closures over the `PersistentMemory` instance. Since the vage `service.Service.buildMux` is not extensible from the outside, the HTTP memory endpoints are registered on a custom mux that wraps the service mux:

```go
// In HTTP mode:
svc := service.New(...)
svcHandler := svc.Handler()

mux := http.NewServeMux()
// Delegate all /v1/agents, /v1/tools, etc. to the service handler
mux.Handle("/", svcHandler)
// Add memory endpoints
mux.HandleFunc("GET /v1/memory", handleListMemory(persistentMem))
mux.HandleFunc("GET /v1/memory/{namespace}/{key}", handleGetMemory(persistentMem))
mux.HandleFunc("PUT /v1/memory/{namespace}/{key}", handleSetMemory(persistentMem))
mux.HandleFunc("DELETE /v1/memory/{namespace}/{key}", handleDeleteMemory(persistentMem))
```

| Endpoint | Method | Handler |
|----------|--------|---------|
| `/v1/memory` | GET | List all entries, optional `?namespace=` query param |
| `/v1/memory/{namespace}/{key}` | GET | Get single entry |
| `/v1/memory/{namespace}/{key}` | PUT | Create/update entry (body: `{"content": "..."}`) |
| `/v1/memory/{namespace}/{key}` | DELETE | Delete entry |

## 3. Data Models

### 3.1 Plan (Internal to Planner)

```go
// vv/agents/planner.go

// Plan represents a parsed execution plan from the LLM.
type Plan struct {
    Goal  string     `json:"goal"`
    Steps []PlanStep `json:"steps"`
}

// PlanStep represents a single step in the plan.
type PlanStep struct {
    ID          string   `json:"id"`
    Description string   `json:"description"`
    Agent       string   `json:"agent"`
    DependsOn   []string `json:"depends_on"`
}
```

### 3.2 Persistent Memory Entry (File Format)

```go
// vv/memory/filestore.go

type fileRecord struct {
    Key       string    `json:"key"`
    Value     string    `json:"value"`
    Namespace string    `json:"namespace"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

### 3.3 Memory HTTP API Models

```go
// vv/main.go (or a separate vv/httpapi package if needed)

// Request body for PUT /v1/memory/{namespace}/{key}
type MemorySetRequest struct {
    Content string `json:"content"`
}

// Response for GET /v1/memory
type MemoryListResponse struct {
    Entries []MemoryEntryResponse `json:"entries"`
}

// Response for GET /v1/memory/{namespace}/{key}
type MemoryEntryResponse struct {
    Namespace string    `json:"namespace"`
    Key       string    `json:"key"`
    Content   string    `json:"content"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

## 4. API Contracts

### 4.1 Agent Interface (Unchanged)

All new agents implement the existing `agent.Agent` and `agent.StreamAgent` interfaces. No new interfaces are introduced. The planner explicitly implements both interfaces (unlike `CustomAgent` which only implements `Agent`).

### 4.2 Memory Store Interface (Existing)

The `FileStore` implements the existing `memory.Store` interface from vage. No new interfaces.

### 4.3 HTTP Memory API

**GET /v1/memory?namespace={ns}**
```
Response 200:
{
  "entries": [
    {"namespace": "project", "key": "conventions", "content": "...", "created_at": "...", "updated_at": "..."},
    ...
  ]
}
```

**GET /v1/memory/{namespace}/{key}**
```
Response 200:
{"namespace": "project", "key": "conventions", "content": "...", "created_at": "...", "updated_at": "..."}

Response 404:
{"code": "not_found", "message": "memory entry not found"}
```

**PUT /v1/memory/{namespace}/{key}**
```
Request body:
{"content": "Go projects use gofumpt for formatting..."}

Response 200:
{"namespace": "project", "key": "conventions", "content": "...", "created_at": "...", "updated_at": "..."}
```

**DELETE /v1/memory/{namespace}/{key}**
```
Response 200:
{"status": "deleted"}

Response 404:
{"code": "not_found", "message": "memory entry not found"}
```

## 5. Implementation Plan

### Task 1: FileStore for Persistent Memory (vv)

**Files:**
- `vv/memory/filestore.go` -- New `FileStore` implementation
- `vv/memory/filestore_test.go` -- Unit tests

**Details:**
- Implement `memory.Store` interface backed by filesystem
- Key format: `namespace:key` -> file path `<dir>/<namespace>/<key>.json`
- Sanitize filenames (replace unsafe characters)
- Create namespace directories on demand
- JSON serialization with `fileRecord` struct

**Depends on:** Nothing

### Task 2: Tool Registry Helpers (vv)

**Files:**
- `vv/tools/tools.go` -- Add `RegisterReadOnly` and `RegisterReviewTools` functions

**Details:**
- `RegisterReadOnly`: registers `read`, `glob`, `grep` only
- `RegisterReviewTools`: registers `read`, `glob`, `grep`, `bash`
- Reuse existing tool registration functions
- Follow the exact error-wrapping pattern from existing `Register` function
- These registries are NOT wrapped with `WrapRegistry` (confirmation not needed for read-only/review tools)

**Depends on:** Nothing

### Task 3: Researcher and Reviewer Agents (vv)

**Files:**
- `vv/agents/agents.go` -- Expand `Create` to return researcher and reviewer
- `vv/agents/prompts.go` -- Add `ResearcherSystemPrompt` and `ReviewerSystemPrompt`

**Details:**
- Create researcher `taskagent.Agent` with read-only tool registry
- Create reviewer `taskagent.Agent` with review tool registry
- Define specialized system prompts

**Depends on:** Task 2

### Task 4: Planner Agent (vv)

**Files:**
- `vv/agents/planner.go` -- New `PlannerAgent` implementation
- `vv/agents/planner_test.go` -- Unit tests
- `vv/agents/prompts.go` -- Add `PlannerSystemPrompt` and `PlanSummaryPrompt`

**Details:**
- Implement `PlannerAgent` struct implementing both `agent.Agent` and `agent.StreamAgent`
- Two-phase approach: plan generation (taskagent.Run) + DAG execution (orchestrate.ExecuteDAG)
- Each DAG node sets `InputMapper` to pass step description as user message with upstream context
- Handle DAG connectivity: add synthetic summary node when plan has multiple terminal nodes
- Define plan JSON schema and parsing
- Implement `PlanAggregator` for result summarization
- Handle plan generation failures with coder fallback (with warning log)
- MaxConcurrency from `cfg.Memory.MaxConcurrency`

**Depends on:** Task 3 (needs sub-agents to delegate to)

### Task 5: Expanded Router (vv)

**Files:**
- `vv/agents/agents.go` -- Update `CreateRouter` with 5 routes
- `vv/main.go` -- Update agent creation and wiring

**Details:**
- Update route table: coder(0), planner(1), researcher(2), reviewer(3), chat(4)
- Update fallback index to 4 (chat)
- Update `Create` function signature to return all agents

**Depends on:** Task 3, Task 4

### Task 6: Memory Manager Wiring (vv)

**Files:**
- `vv/config/config.go` -- Add `MemoryConfig`, update `applyDefaults`
- `vv/main.go` -- Wire memory manager at startup
- `vv/agents/agents.go` -- Pass memory manager to task agents
- `vv/agents/prompts.go` -- Add `PersistentMemoryPrompt` dynamic template

**Details:**
- Add `MemoryConfig` to `Config` struct with defaults in `applyDefaults`
- Create `FileStore` + `PersistentMemory` at startup
- Create `SessionMemory` per session
- Wire `memory.Manager` into coder, researcher, reviewer agents
- Implement `PersistentMemoryPrompt` for dynamic persistent memory injection
- Wrap each agent's base system prompt with persistent memory template

**Depends on:** Task 1, Task 5

### Task 7: CLI Session Memory Integration (vv)

**Files:**
- `vv/cli/cli.go` -- Remove `App.history`, pass only latest message in RunRequest

**Details:**
- Remove `App.history` field from `App` struct
- Update `handleSubmit` to pass only the new user message in `req.Messages`
- Update `handleStreamDone` to not append to history (memory manager handles this)
- Remove `coder` and `chat` fields from `App` (use routes only)
- Update `New` signature to accept `persistentMem` for `/memory` commands

**Depends on:** Task 6

### Task 8: CLI Memory Commands (vv)

**Files:**
- `vv/cli/cli.go` -- Add command parsing before agent routing
- `vv/cli/memory.go` -- New file with memory command handlers

**Details:**
- Parse `/memory list`, `/memory show`, `/memory set`, `/memory delete` commands
- Implement handlers that operate on `PersistentMemory`
- Display results as system messages in the TUI
- For `/memory set`, use a multiline input mode (reuse textarea)
- Handle commands in `handleSubmit` after exit check, before agent routing

**Depends on:** Task 7

### Task 9: HTTP Memory Endpoints (vv)

**Files:**
- `vv/main.go` -- Register memory HTTP routes in HTTP mode

**Details:**
- Implement GET/PUT/DELETE handlers for memory entries as closures in `main.go`
- Wrap the service handler with a custom mux that includes memory routes
- Reuse `PersistentMemory` instance from startup

**Depends on:** Task 6

### Task 10: System Prompts with Persistent Memory (vv)

**Files:**
- `vv/agents/agents.go` -- Use `PersistentMemoryPrompt` for all agents with tools
- `vv/agents/prompts.go` -- Add memory context formatting

**Details:**
- Wrap each agent's system prompt with `PersistentMemoryPrompt`
- Format persistent memory entries as a structured knowledge block
- Keep the injected block under a reasonable size (truncate if needed)
- Dynamic loading on each `Render` call, not baked at agent creation time

**Depends on:** Task 6

## 6. Integration Test Plan

### Test 1: Router Dispatches to All Five Agents

**Location:** `vv/integrations/agents_test.go`

- Send a coding request -> verify it routes to coder (agent ID in response)
- Send a "set up a new Go project with tests and CI" request -> verify it routes to planner
- Send a "what does the orchestrate package do?" request -> verify it routes to researcher
- Send a "review this function for bugs" request -> verify it routes to reviewer
- Send a "what is the capital of France?" request -> verify it routes to chat
- Send an ambiguous request -> verify it falls back to chat

### Test 2: Researcher Agent Uses Read-Only Tools

**Location:** `vv/integrations/agents_test.go`

- Send a research request to the researcher agent directly
- Verify it can use `read`, `glob`, `grep` tools
- Verify it does NOT have access to `write`, `edit`, `bash` tools (check tool list)

### Test 3: Reviewer Agent Has Correct Tools

**Location:** `vv/integrations/agents_test.go`

- Send a review request to the reviewer agent directly
- Verify it can use `read`, `glob`, `grep`, `bash` tools
- Verify it does NOT have access to `write`, `edit` tools

### Test 4: Planner Generates and Executes a Plan

**Location:** `vv/integrations/agents_test.go`

- Send a multi-step task to the planner
- Verify the planner generates a valid plan (JSON parseable)
- Verify the plan contains steps with agent assignments
- Verify the plan respects declared dependencies (forms a valid DAG)
- Verify the aggregated response includes outputs from completed steps

### Test 5: Planner Handles Failure Gracefully

**Location:** `vv/integrations/agents_test.go`

- Send a task that will cause a step to fail (e.g., edit a non-existent file)
- Verify the planner continues executing other steps (Skip strategy)
- Verify the final response notes the failure

### Test 6: Planner Fallback to Coder

**Location:** `vv/integrations/agents_test.go`

- Mock/configure plan generation to return invalid JSON
- Verify the planner falls back to the coder agent
- Verify the fallback includes a warning in logs

### Test 7: FileStore CRUD

**Location:** `vv/memory/filestore_test.go` (unit)

- Set a key-value pair -> verify file is created on disk
- Get the key -> verify value matches
- List with prefix -> verify filtering works
- Delete the key -> verify file is removed
- List after delete -> verify empty
- Clear -> verify all files removed

### Test 8: Session Memory via Memory Manager

**Location:** `vv/integrations/cli_test.go`

- Send message 1 to the agent (only latest message in RunRequest)
- Send message 2 that references message 1
- Verify the agent's response to message 2 has context from message 1 (via session memory)
- Verify session memory contains both exchanges

### Test 9: Persistent Memory Load at Startup

**Location:** `vv/integrations/config_test.go`

- Pre-populate `~/.vv/memory/` with test entries
- Start the application
- Send a request that should trigger use of persistent memory
- Verify the agent's system prompt includes the persistent memory content

### Test 10: CLI Memory Commands

**Location:** `vv/integrations/cli_test.go`

- Execute `/memory set project:conventions` with content
- Execute `/memory list` -> verify entry appears
- Execute `/memory show project:conventions` -> verify content matches
- Execute `/memory delete project:conventions`
- Execute `/memory list` -> verify entry is gone

### Test 11: HTTP Memory API

**Location:** `vv/integrations/http_test.go`

- PUT `/v1/memory/project/conventions` with content -> verify 200
- GET `/v1/memory` -> verify entry in list
- GET `/v1/memory/project/conventions` -> verify content matches
- DELETE `/v1/memory/project/conventions` -> verify 200
- GET `/v1/memory/project/conventions` -> verify 404

### Test 12: End-to-End Router -> Planner -> Coder Flow

**Location:** `vv/integrations/agents_test.go`

- Send a complex multi-step request through the router
- Verify the router selects the planner
- Verify the planner decomposes the task into steps
- Verify steps are delegated to the coder agent
- Verify the final response aggregates step outputs
- Verify session memory contains the full exchange

### Test 13: Memory Compressor Integration

**Location:** `vv/integrations/agents_test.go`

- Send enough messages to exceed the sliding window size (50)
- Verify older messages are compressed/dropped
- Verify the agent still responds with context from recent messages
- Verify session memory token count stays within budget

### Test 14: PlannerAgent Implements StreamAgent

**Location:** `vv/agents/planner_test.go`

- Compile-time check: `var _ agent.StreamAgent = (*PlannerAgent)(nil)`
- Call `RunStream` on the planner
- Verify it returns a valid `RunStream` that emits AgentStart and AgentEnd events
