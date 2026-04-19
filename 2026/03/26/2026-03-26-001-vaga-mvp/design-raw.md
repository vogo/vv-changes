# vv MVP - Technical Design

## Architecture Overview

vv is a single-binary Go application that wires together vage framework components into a production-ready AI agent service. The architecture follows a layered initialization pattern:

```
main.go (CLI entry point)
  |
  +-- config.go        (YAML + env var configuration)
  +-- tools.go         (tool registration)
  +-- agents.go        (agent + router creation)
  +-- main.go          (HTTP service startup + graceful shutdown)
```

Request flow:

```
HTTP Request --> service.Service --> routeragent.Agent
                                        |
                              +---------+---------+
                              |                   |
                        coder (TaskAgent)   chat (TaskAgent)
                        [bash,read,write,   [no tools]
                         edit,glob,grep]
```

All components live in a single Go package (`main`) at `vv/`. This keeps the MVP simple -- no internal sub-packages needed at this scale.

## Component Design

### 1. Configuration (`config.go`)

A single `Config` struct loaded from YAML, with environment variable overrides.

```go
// Config holds all vv application configuration.
type Config struct {
    LLM    LLMConfig    `yaml:"llm"`
    Server ServerConfig `yaml:"server"`
    Tools  ToolsConfig  `yaml:"tools"`
    Agents AgentsConfig `yaml:"agents"`
}

type LLMConfig struct {
    Provider string `yaml:"provider"` // "openai" or "anthropic"
    Model    string `yaml:"model"`    // e.g. "gpt-4o", "claude-sonnet-4-20250514"
    APIKey   string `yaml:"api_key"`  // overridden by VV_LLM_API_KEY
    BaseURL  string `yaml:"base_url"` // overridden by VV_LLM_BASE_URL
}

type ServerConfig struct {
    Addr string `yaml:"addr"` // default ":8080"
}

type ToolsConfig struct {
    BashTimeout    int    `yaml:"bash_timeout"`     // seconds, default 30
    BashWorkingDir string `yaml:"bash_working_dir"` // default ""
}

type AgentsConfig struct {
    MaxIterations  int `yaml:"max_iterations"`   // default 10
    RunTokenBudget int `yaml:"run_token_budget"` // default 0 (unlimited)
}
```

**Loading strategy:**
1. Read YAML file (default `vv.yaml`, overridden by `-config` flag).
2. Apply environment variable overrides: `VV_LLM_API_KEY`, `VV_LLM_BASE_URL`, `VV_LLM_MODEL`, `VV_LLM_PROVIDER`, `VV_SERVER_ADDR`.
3. Validate: API key must be non-empty.
4. Apply defaults for unset values.

The config loader uses `gopkg.in/yaml.v3` (already a dependency of vage) and `os.Getenv` for environment overrides. No new dependencies required.

### 2. Tool Registration (`tools.go`)

Uses the existing `tool.Registry` and each tool package's `Register()` function.

```go
func registerTools(cfg ToolsConfig) (*tool.Registry, error) {
    reg := tool.NewRegistry()

    // bash
    bashOpts := []bashtool.Option{}
    if cfg.BashTimeout > 0 {
        bashOpts = append(bashOpts, bashtool.WithTimeout(time.Duration(cfg.BashTimeout)*time.Second))
    }
    if cfg.BashWorkingDir != "" {
        bashOpts = append(bashOpts, bashtool.WithWorkingDir(cfg.BashWorkingDir))
    }
    if err := bashtool.Register(reg, bashOpts...); err != nil {
        return nil, err
    }

    // read, write, edit, glob, grep -- each with Register(reg)
    if err := readtool.Register(reg); err != nil { return nil, err }
    if err := writetool.Register(reg); err != nil { return nil, err }
    if err := edittool.Register(reg); err != nil { return nil, err }
    if err := globtool.Register(reg); err != nil { return nil, err }
    if err := greptool.Register(reg); err != nil { return nil, err }

    return reg, nil
}
```

### 3. LLM Client Creation

Uses `aimodel.NewClient` directly. The `aimodel.Client` implements `aimodel.ChatCompleter`, which is what `taskagent` and `routeragent.LLMFunc` require.

```go
func newLLMClient(cfg LLMConfig) (aimodel.ChatCompleter, error) {
    opts := []aimodel.Option{
        aimodel.WithAPIKey(cfg.APIKey),
        aimodel.WithDefaultModel(cfg.Model),
    }
    if cfg.BaseURL != "" {
        opts = append(opts, aimodel.WithBaseURL(cfg.BaseURL))
    }
    if cfg.Provider == "anthropic" {
        opts = append(opts, aimodel.WithProtocol(aimodel.ProtocolAnthropic))
    }
    return aimodel.NewClient(opts...)
}
```

No `largemodel.Model` middleware wrapper is used in the MVP (observability is out of scope). The raw `aimodel.Client` is sufficient.

### 4. Agent Creation (`agents.go`)

#### Coder Agent

A `taskagent.Agent` with a code-focused system prompt and access to all tools.

```go
coderAgent := taskagent.New(
    agent.Config{
        ID:          "coder",
        Name:        "Coder Agent",
        Description: "Performs coding tasks: reads, writes, edits files, runs commands, and searches codebases",
    },
    taskagent.WithChatCompleter(llmClient),
    taskagent.WithModel(cfg.LLM.Model),
    taskagent.WithToolRegistry(toolRegistry),
    taskagent.WithSystemPrompt(prompt.StringPrompt(coderSystemPrompt)),
    taskagent.WithMaxIterations(cfg.Agents.MaxIterations),
    taskagent.WithRunTokenBudget(cfg.Agents.RunTokenBudget),
)
```

#### Chat Agent

A `taskagent.Agent` with a conversational system prompt and no tool registries.

```go
chatAgent := taskagent.New(
    agent.Config{
        ID:          "chat",
        Name:        "Chat Agent",
        Description: "Handles general conversation, questions, and non-coding tasks",
    },
    taskagent.WithChatCompleter(llmClient),
    taskagent.WithModel(cfg.LLM.Model),
    taskagent.WithSystemPrompt(prompt.StringPrompt(chatSystemPrompt)),
    taskagent.WithMaxIterations(1), // no tool loop needed
)
```

Setting `MaxIterations(1)` ensures the chat agent makes exactly one LLM call (no ReAct loop since it has no tools).

#### Router Agent

A `routeragent.Agent` with `LLMFunc` routing.

```go
router := routeragent.New(
    agent.Config{
        ID:          "router",
        Name:        "Router Agent",
        Description: "Routes requests to the appropriate specialized agent",
    },
    []routeragent.Route{
        {Agent: coderAgent, Description: "Handles code-related tasks: reading files, writing code, editing files, running commands, searching codebases, debugging, and software engineering tasks"},
        {Agent: chatAgent, Description: "Handles general conversation, questions, explanations, brainstorming, and non-coding tasks"},
    },
    routeragent.WithFunc(routeragent.LLMFunc(llmClient, cfg.LLM.Model, 1)), // fallback=1 (chat)
)
```

The fallback index is `1` (chat agent), so if the LLM routing fails or returns ambiguous results, requests default to the chat agent.

### 5. System Prompts

Defined as Go string constants in `prompts.go`:

**Coder system prompt** (key elements):
- Role: expert software engineer
- Available tools: bash, read, write, edit, glob, grep
- Instructions: think step-by-step, read before editing, verify changes
- Guidelines: prefer minimal changes, explain reasoning

**Chat system prompt** (key elements):
- Role: helpful, knowledgeable assistant
- No tool access
- Instructions: provide accurate, clear responses
- Be concise but thorough

### 6. HTTP Service & CLI (`main.go`)

```go
func main() {
    // Parse flags
    configPath := flag.String("config", "vv.yaml", "config file path")
    listenAddr := flag.String("addr", "", "listen address (overrides config)")
    flag.Parse()

    // Load config
    cfg, err := loadConfig(*configPath)
    // ... handle error

    if *listenAddr != "" {
        cfg.Server.Addr = *listenAddr
    }

    // Create LLM client
    llmClient, err := newLLMClient(cfg.LLM)

    // Register tools
    toolRegistry, err := registerTools(cfg.Tools)

    // Create agents
    coderAgent, chatAgent := createAgents(cfg, llmClient, toolRegistry)
    router := createRouter(cfg, llmClient, coderAgent, chatAgent)

    // Create and start service
    svc := service.New(
        service.Config{Addr: cfg.Server.Addr},
        service.WithToolRegistry(toolRegistry),
    )
    svc.RegisterAgent(router)
    svc.RegisterAgent(coderAgent)
    svc.RegisterAgent(chatAgent)

    // Graceful shutdown
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    slog.Info("vv: starting", "addr", cfg.Server.Addr)
    if err := svc.Start(ctx); err != nil {
        slog.Error("vv: server error", "error", err)
        os.Exit(1)
    }
}
```

All three agents (router, coder, chat) are registered so they can be accessed individually via the HTTP API. The router is the primary entry point, but users can also call the coder or chat agents directly via `/v1/agents/coder/run`, `/v1/agents/chat/run`, etc.

## Data Models / Schemas

No new data models are needed. The MVP uses existing vage framework types exclusively:

| Concept | Type | Package |
|---------|------|---------|
| Run request | `schema.RunRequest` | `github.com/vogo/vage/schema` |
| Run response | `schema.RunResponse` | `github.com/vogo/vage/schema` |
| Stream events | `schema.Event`, `schema.RunStream` | `github.com/vogo/vage/schema` |
| Tool definitions | `schema.ToolDef` | `github.com/vogo/vage/schema` |
| Tool results | `schema.ToolResult` | `github.com/vogo/vage/schema` |
| Messages | `schema.Message` | `github.com/vogo/vage/schema` |
| Application config | `Config` (new, local) | `main` (vv) |

## API Contracts

Inherited entirely from `service.Service`. No custom endpoints.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/health` | GET | Returns `{"status": "ok"}` |
| `/v1/agents` | GET | Lists router, coder, chat agents |
| `/v1/agents/{id}` | GET | Agent details (id, name, description) |
| `/v1/agents/{id}/run` | POST | Synchronous execution. Body: `RunRequest` JSON. Response: `RunResponse` JSON |
| `/v1/agents/{id}/stream` | POST | SSE streaming. Body: `RunRequest` JSON. Response: SSE events |
| `/v1/agents/{id}/async` | POST | Async execution. Body: `RunRequest` JSON. Response: `{"task_id": "..."}` |
| `/v1/tools` | GET | Lists registered tools |
| `/v1/tasks/{taskID}` | GET | Async task status |
| `/v1/tasks/{taskID}/cancel` | POST | Cancel async task |

**Example request:**
```json
POST /v1/agents/router/run
{
  "messages": [
    {"role": "user", "content": {"type": "text", "text": "Read the main.go file and add error handling"}}
  ]
}
```

## Configuration File Format

```yaml
# vv.yaml
llm:
  provider: "openai"        # "openai" or "anthropic"
  model: "gpt-4o"           # model name
  api_key: ""               # prefer VV_LLM_API_KEY env var
  base_url: ""              # optional, prefer VV_LLM_BASE_URL env var

server:
  addr: ":8080"

tools:
  bash_timeout: 30          # seconds
  bash_working_dir: ""      # empty = current directory

agents:
  max_iterations: 10
  run_token_budget: 0       # 0 = unlimited
```

## File Structure

```
vv/
  main.go       -- CLI entry point, flag parsing, signal handling, service startup
  config.go     -- Config struct, loadConfig(), env var overrides, validation
  tools.go      -- registerTools() function
  agents.go     -- createAgents(), createRouter() functions
  prompts.go    -- system prompt string constants
  go.mod        -- module: github.com/vogo/vv, depends on github.com/vogo/vage
```

## Implementation Plan

### Task 1: Initialize Go module

Create `vv/go.mod` with module path `github.com/vogo/vv` and dependency on `github.com/vogo/vage`.

### Task 2: Configuration loading (`config.go`)

- Define `Config`, `LLMConfig`, `ServerConfig`, `ToolsConfig`, `AgentsConfig` structs with YAML tags.
- Implement `loadConfig(path string) (*Config, error)`:
  - Read and parse YAML file. If file not found, use zero-value config (all defaults).
  - Apply environment variable overrides (`VV_LLM_API_KEY`, `VV_LLM_BASE_URL`, `VV_LLM_MODEL`, `VV_LLM_PROVIDER`, `VV_SERVER_ADDR`).
  - Apply defaults (server addr `:8080`, max iterations `10`, bash timeout `30`).
  - Validate: API key must be non-empty after all overrides.
- Implement `newLLMClient(cfg LLMConfig) (aimodel.ChatCompleter, error)`.

### Task 3: Tool registration (`tools.go`)

- Implement `registerTools(cfg ToolsConfig) (*tool.Registry, error)`.
- Register bash (with configurable timeout/working dir), read, write, edit, glob, grep tools.

### Task 4: System prompts (`prompts.go`)

- Define `coderSystemPrompt` constant string.
- Define `chatSystemPrompt` constant string.

### Task 5: Agent creation (`agents.go`)

- Implement `createAgents(cfg *Config, llm aimodel.ChatCompleter, reg *tool.Registry) (*taskagent.Agent, *taskagent.Agent)` returning coder and chat agents.
- Implement `createRouter(cfg *Config, llm aimodel.ChatCompleter, coder, chat agent.Agent) *routeragent.Agent`.

### Task 6: CLI entry point (`main.go`)

- Parse `-config` and `-addr` flags.
- Wire everything together: config -> LLM client -> tools -> agents -> router -> service.
- Register all three agents with the service.
- Set up `signal.NotifyContext` for SIGINT/SIGTERM.
- Call `svc.Start(ctx)` and handle errors.
- Add structured logging (`log/slog`) for startup, configuration summary (redact API key), and shutdown.

### Task 7: Build verification

- Ensure `go build ./vv/` compiles cleanly.
- Ensure `go vet ./vv/` passes.

## Integration Test Plan

### Test 1: Configuration loading

- **Test valid YAML**: Load a valid config file, verify all fields parsed correctly.
- **Test env var override**: Set `VV_LLM_API_KEY` env var, verify it overrides YAML value.
- **Test missing API key**: Verify `loadConfig` returns a clear error when no API key is provided.
- **Test defaults**: Load empty YAML, verify defaults applied (addr=`:8080`, max_iterations=10, bash_timeout=30).
- **Test missing file**: Verify graceful handling when config file does not exist (use defaults + env vars).

### Test 2: Tool registration

- **Test all tools registered**: Call `registerTools`, verify registry contains exactly 6 tools: bash, read, write, edit, glob, grep.
- **Test bash options**: Verify custom timeout and working dir are applied.

### Test 3: Agent creation

- **Test coder agent has tools**: Create coder agent, verify `Tools()` returns 6 tool definitions.
- **Test chat agent has no tools**: Create chat agent, verify `Tools()` returns nil/empty.
- **Test router routes correctly**: Use a mock LLM that returns index "0" for code requests and "1" for chat requests, verify routing.

### Test 4: HTTP service end-to-end

- **Test health endpoint**: `GET /v1/health` returns 200 with `{"status": "ok"}`.
- **Test agent listing**: `GET /v1/agents` returns 3 agents (router, coder, chat).
- **Test tool listing**: `GET /v1/tools` returns 6 tools.
- **Test sync run**: `POST /v1/agents/chat/run` with a simple message, verify response structure.
- **Test streaming**: `POST /v1/agents/chat/stream` returns SSE events.
- **Test async**: `POST /v1/agents/chat/async` returns task ID, `GET /v1/tasks/{id}` returns status.

### Test 5: Graceful shutdown

- Start the service, send SIGINT, verify the process exits cleanly without error.
