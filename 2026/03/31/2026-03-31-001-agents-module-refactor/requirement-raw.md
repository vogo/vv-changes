# vv/agents 模块重构方案

## 一、现状问题

| 问题 | 具体表现 |
|------|----------|
| Orchestrator 臃肿 | 单文件 ~1200 行，混合分发、DAG构建、动态Agent创建、流式处理、结果聚合 |
| Agent 注册分散 | 添加新 Agent 需改 agents.go(工厂)、planner.go(提示词)、orchestrator.go(验证map) 三处 |
| 工具权限粗粒度 | 只有 full / read-only / none 三级，reviewer 绕过权限体系走单独的 reviewReg |
| InputMapper 不可测 | DAG 节点的输入构造是内联闭包，无法单元测试 |
| 动态 Agent 无生命周期 | 创建即弃，无 hook、无复用、无可观测性 |
| DAG 错误静默跳过 | ErrorStrategy: Skip，下游步骤无法感知上游失败 |
| 配置错位 | MaxConcurrency 放在 MemoryConfig 里 |

## 二、重构后目录结构

```
vv/
├── dispatch/           # 编排调度（从 orchestrator.go 拆出）
│   ├── dispatch.go     # Dispatcher 入口（原 OrchestratorAgent）
│   ├── dispatcher.go   # 请求分发：explore → classify → route
│   ├── dag.go          # Plan → DAG 节点构建 + 执行
│   ├── input.go        # StepInput 消息构造（纯函数，可测试）
│   └── stream.go       # 流式事件转发 + streamingDAGHandler
│
├── registry/           # Agent 注册与发现
│   ├── registries.go     # AgentRegistry + AgentDescriptor + AgentFactory
│   └── tool_access.go  # ToolProfile / ToolCapability 权限模型
│
├── agents/             # Agent 定义（每个文件只管自己的 prompt + 注册）
│   ├── coder.go
│   ├── researcher.go
│   ├── reviewer.go
│   ├── chat.go
│   ├── explorer.go
│   ├── planner.go
│   └── memory_prompt.go
│
├── lifecycle/          # Agent 生命周期 Hook
│   └── hooks.go        # OnBeforeRun / OnAfterRun 接口 + 内置实现
│
├── setup/              # 组装层（原 agents.go 的工厂职责）
│   └── setup.go        # 读 config → 注册 agents → 构造 Dispatcher
│
├── config/             # 不变
├── tools/              # 不变
├── memory/             # 不变
└── cli/                # 不变
```

## 三、依赖关系

```
main.go
  └→ setup/
     ├→ config/
     ├→ registry/    ← 无外部依赖（仅 vage/agent 接口）
     ├→ agents/      → registry/
     ├→ dispatch/    → registry/, lifecycle/, vage/orchestrate
     ├→ lifecycle/   ← 仅 vage/schema
     ├→ tools/
     └→ memory/
```

无循环依赖，每层只向下依赖。

## 四、各子包详细设计

### 4.1 registry/ — Agent 注册表 + 工具权限

**registry/tool_access.go**

```go
package registry

type ToolCapability string

const (
    CapRead    ToolCapability = "read"
    CapWrite   ToolCapability = "write"
    CapExecute ToolCapability = "execute"   // bash
    CapSearch  ToolCapability = "search"    // glob, grep
)

type ToolProfile struct {
    Name         string
    Capabilities []ToolCapability
}

var (
    ProfileFull     = ToolProfile{"full", []ToolCapability{CapRead, CapWrite, CapExecute, CapSearch}}
    ProfileReadOnly = ToolProfile{"read-only", []ToolCapability{CapRead, CapSearch}}
    ProfileReview   = ToolProfile{"review", []ToolCapability{CapRead, CapSearch, CapExecute}}
    ProfileNone     = ToolProfile{"none", nil}
)

// 根据 Capability 从完整工具集中筛选出子集
func (p ToolProfile) BuildRegistry(allTools tool.ToolRegistry) tool.ToolRegistry
```

**registry/registries.go**

```go
package registry

type AgentDescriptor struct {
    ID          string
    DisplayName string
    Description string          // 给 planner 自动生成提示词用
    ToolProfile ToolProfile
    Factory     AgentFactory
}

type AgentFactory func(opts FactoryOptions) (agent.Agent, error)

type FactoryOptions struct {
    LLM           aimodel.ChatCompleter
    Model         string
    ToolRegistry  tool.ToolRegistry
    MaxIterations int
    Memory        memory.Manager
}

type Registry struct {
    mu     sync.RWMutex
    agents map[string]AgentDescriptor
}

func New() *Registry
func (r *Registry) Register(d AgentDescriptor)
func (r *Registry) Get(id string) (AgentDescriptor, bool)
func (r *Registry) All() []AgentDescriptor
func (r *Registry) ValidateRef(id string) bool

// 自动生成 planner 可用 agent 列表片段，不再手动维护
func (r *Registry) PlannerAgentList() string
```

### 4.2 agents/ — 纯 Agent 定义

每个文件只包含：系统提示词 + 注册函数。不包含编排逻辑。

```go
// agents/coder.go
package agents

func RegisterCoder(r *registries.Registry, mem memory.Manager) {
    r.Register(registries.AgentDescriptor{
        ID:          "coder",
        DisplayName: "Coder",
        Description: "Full tool access, code modification and creation",
        ToolProfile: registries.ProfileFull,
        Factory: func(opts registries.FactoryOptions) (agent.Agent, error) {
            return taskagent.New("coder", "Coder",
                taskagent.WithSystemPrompt(coderSystemPrompt(mem)),
                taskagent.WithToolRegistry(opts.ToolRegistry),
                taskagent.WithMaxIterations(opts.MaxIterations),
            ), nil
        },
    })
}

// agents/reviewer.go
package agents

func RegisterReviewer(r *registries.Registry) {
    r.Register(registries.AgentDescriptor{
        ID:          "reviewer",
        DisplayName: "Reviewer",
        Description: "Code review with read access and bash for running tests/linters",
        ToolProfile: registries.ProfileReview,   // 统一权限模型，不再特殊处理
        Factory: func(opts registries.FactoryOptions) (agent.Agent, error) {
            return taskagent.New("reviewer", "Reviewer",
                taskagent.WithSystemPrompt(reviewerSystemPrompt),
                taskagent.WithToolRegistry(opts.ToolRegistry),
                taskagent.WithMaxIterations(opts.MaxIterations),
            ), nil
        },
    })
}
```

### 4.3 dispatch/ — 编排调度

**dispatch/dispatch.go — 入口**

```go
package dispatch

type Dispatcher struct {
    agent.Base
    registry       *registries.Registry
    planner        agent.Agent
    explorer       agent.Agent
    summarizer     *taskagent.Agent
    hooks          []lifecycle.Hook
    maxConcurrency int
    workingDir     string
}

func New(reg *registries.Registry, opts Options) (*Dispatcher, error)

func (d *Dispatcher) Run(ctx context.Context, req *schema.RunRequest) (*schema.RunResponse, error) {
    // 1. explore
    ctx, summary := d.explore(ctx, req)
    // 2. classify
    result, err := d.classify(ctx, req, summary)
    // 3. route
    if result.Mode == "direct" {
        return d.runDirect(ctx, req, result)
    }
    return d.runPlan(ctx, req, result.Plan, summary)
}

func (d *Dispatcher) RunStream(ctx context.Context, req *schema.RunRequest) (*schema.RunStream, error)
```

**dispatch/dispatcher.go — 分发逻辑**

```go
package dispatch

func (d *Dispatcher) explore(ctx context.Context, req *schema.RunRequest) (string, *aimodel.Usage, error)
func (d *Dispatcher) classify(ctx context.Context, req *schema.RunRequest, context string) (*ClassifyResult, error)

type ClassifyResult struct {
    Mode  string   // "direct" | "plan"
    Agent string   // direct 模式的目标 agent
    Plan  *Plan    // plan 模式的执行计划
}

type Plan struct {
    Goal  string
    Steps []PlanStep
}

type PlanStep struct {
    ID          string
    Description string
    Agent       string
    DependsOn   []string
    DynamicSpec *DynamicAgentSpec
}

func (c *ClassifyResult) Validate(reg *registries.Registry) error
```

**dispatch/dag.go — DAG 构建与执行**

```go
package dispatch

func (d *Dispatcher) buildNodes(plan *Plan, workDir, contextSummary, goal string) ([]orchestrate.Node, error)
func (d *Dispatcher) buildDynamicAgent(step PlanStep) (agent.Agent, error)
func (d *Dispatcher) runPlan(ctx context.Context, req *schema.RunRequest, plan *Plan, summary string) (*schema.RunResponse, error)
```

**dispatch/input.go — 消息构造（纯函数，可测试）**

```go
package dispatch

type StepInput struct {
    WorkingDir      string
    ContextSummary  string
    OriginalGoal    string
    StepDescription string
    Upstream        map[string]StepResult
}

type StepResult struct {
    Output string
    Status StepStatus
}

type StepStatus int
const (
    StepCompleted StepStatus = iota
    StepFailed
    StepSkipped
)

func (s *StepInput) BuildMessages() []schema.Message
func (s *StepInput) HasFailedDependency() bool
func BuildInputMapper(workDir, ctx, goal string, step PlanStep) orchestrate.InputMapper
```

**dispatch/stream.go — 流式处理**

```go
package dispatch

type StreamHandler struct {
    send      func(schema.Event) error
    agentID   string
    sessionID string
    steps     map[string]stepMeta
    mu        sync.Mutex
}

func NewStreamHandler(send func(schema.Event) error, agentID, sessionID string) *StreamHandler
func (h *StreamHandler) OnNodeStart(nodeID string, agent, desc string)
func (h *StreamHandler) OnNodeEnd(nodeID string, tokens int)
func (h *StreamHandler) ForwardSubAgentStream(nodeID string, stream *schema.RunStream) (*schema.RunResponse, error)
```

### 4.4 lifecycle/ — 生命周期 Hook

```go
package lifecycle

type Hook interface {
    OnBeforeRun(ctx context.Context, agentID string, req *schema.RunRequest) error
    OnAfterRun(ctx context.Context, agentID string, resp *schema.RunResponse, err error)
}

type LoggingHook struct{ Logger *slog.Logger }
type MetricsHook struct{ Collector MetricsCollector }
type RateLimitHook struct{ Limiter *rate.Limiter }

func Chain(hooks ...Hook) Hook
```

### 4.5 setup/ — 组装层

```go
package setup

func New(cfg *config.Config) (*dispatch.Dispatcher, error) {
    // 1. 基础设施
    llm := config.NewLLMClient(cfg)
    mem := memory.NewFileStore(cfg.Memory.Dir)
    reg := registries.New()

    // 2. 注册所有 agent
    agents.RegisterCoder(reg, mem)
    agents.RegisterResearcher(reg, mem)
    agents.RegisterReviewer(reg)
    agents.RegisterChat(reg)
    agents.RegisterExplorer(reg)
    agents.RegisterPlanner(reg)

    // 3. 构造 hooks
    hooks := []lifecycle.Hook{
        &lifecycle.LoggingHook{Logger: slog.Default()},
    }

    // 4. 构造 dispatcher
    return dispatch.New(reg, dispatch.Options{
        LLM:            llm,
        Model:          cfg.LLM.Model,
        MaxConcurrency: cfg.Orchestrate.MaxConcurrency,
        Hooks:          hooks,
        WorkingDir:     cfg.WorkingDir,
    })
}
```
