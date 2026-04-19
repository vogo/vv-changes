## Dynamic Sub-Agent Support for Vaga Main Agent

### Background & Objectives

Currently, the vv Orchestrator Agent dispatches tasks exclusively to a fixed set of statically-configured sub-agents (coder, researcher, reviewer, chat) that are created at application startup. While the orchestrator can decompose complex tasks into DAG plans, each plan step must reference one of these pre-registered agent types. This limits flexibility: the orchestrator cannot tailor an agent's system prompt, tool set, or model parameters to the specific needs of a sub-task at runtime.

**Objectives:**
- Enable the Orchestrator Agent to dynamically define, configure, and instantiate sub-agents at runtime based on the requirements of each sub-task
- Allow prompt-driven sub-task definitions where the LLM determines what kind of agent is needed and what parameters it should have
- Maintain backward compatibility with existing static sub-agent dispatch for simple tasks

### User Stories & Acceptance Criteria

**US-1: Prompt-Driven Sub-Task Definition**
- As the Orchestrator Agent, when I receive a complex user request, I can analyze the prompt and define sub-tasks that each specify a custom agent configuration (system prompt, tool requirements, model parameters) rather than just selecting from a fixed agent pool.
- **Acceptance Criteria:**
  - The planner can produce sub-task definitions that include agent configuration parameters (system prompt, tool set, model)
  - Sub-task definitions are validated for completeness (must have at least a description and a base agent type)
  - The orchestrator falls back to static agent selection when no custom configuration is needed

**US-2: Dynamic Parameter Preparation**
- As the Orchestrator Agent, for each sub-task I can dynamically determine and prepare the parameters needed for the sub-agent, including selecting appropriate tools, crafting a specialized system prompt, and choosing the right model.
- **Acceptance Criteria:**
  - Parameters are derived from the task context (user request, project context, upstream step results)
  - Tool selection is constrained to tools available in the system's tool registries
  - Model selection defaults to the configured model if not explicitly specified

**US-3: Build and Run Dynamic Sub-Agent**
- As the Orchestrator Agent, I can build a new sub-agent instance with the dynamically prepared parameters and execute it as part of the plan.
- **Acceptance Criteria:**
  - Dynamic sub-agents are created as ephemeral task agents that exist only for the duration of their step execution
  - Dynamic sub-agents have access to tool registries based on the specified tool access level (full, read-only, or none)
  - Results from dynamic sub-agents are collected and aggregated the same way as static sub-agent results
  - Token usage from dynamic sub-agents is tracked and aggregated into the overall response

**US-4: Mixed Static and Dynamic Dispatch**
- As the Orchestrator Agent, I can create plans that mix static sub-agent dispatch (for well-known task types) and dynamic sub-agent creation (for novel or specialized tasks) within the same plan.
- **Acceptance Criteria:**
  - Plan steps can reference either a static agent ID or a dynamic agent specification
  - Static and dynamic steps can have dependencies on each other
  - The DAG execution engine handles both step types uniformly

### Scope Boundaries

**In Scope:**
- Dynamic sub-agent specification format within plan steps
- Dynamic agent building from specifications (system prompt, tool access level, model)
- Tool access level selection (full, read-only, none) for dynamic agents
- Integration with existing DAG execution and result aggregation
- Validation of dynamic agent specifications
- Backward compatibility with existing static dispatch

**Out of Scope:**
- User-defined custom agent types or plugins (this is orchestrator-internal only)
- Persistent agent definitions (dynamic agents are ephemeral per-step)
- Dynamic tool creation (agents use existing registered tools)
- New tool types or tool registries
- Changes to the CLI or HTTP interfaces (this is transparent to end users)
- Dynamic model provider configuration (uses existing configured providers)

### System Roles

| Role | Impact |
|------|--------|
| Orchestrator Agent | Extended to support dynamic sub-agent specifications in plans, building dynamic agents, and executing them |
| Planner Agent | Extended to produce plan steps with optional dynamic agent specifications |
| Coder Agent | No change (remains a static sub-agent) |
| Researcher Agent | No change (remains a static sub-agent) |
| Reviewer Agent | No change (remains a static sub-agent) |
| Chat Agent | No change (remains a static sub-agent) |

### Models & States

| Model | Change Type | Description |
|-------|-------------|-------------|
| Plan Step | Updated | Add optional dynamic agent specification attributes |
| Dynamic Agent Spec | New | Specification for dynamically creating a sub-agent |
| Agent | No change | Existing model unchanged; dynamic agents are ephemeral and not registered |
| Task Plan | No change | Existing model unchanged |

### Business Processes & Rules

| Process | Change Type | Description |
|---------|-------------|-------------|
| Orchestration | Updated | Extended to handle dynamic agent specs in plan steps: build agent from spec, execute, collect results |

**New Business Rules:**
- ORCH-16: Dynamic Agent Building - When a plan step includes a dynamic agent specification, the Orchestrator must build an ephemeral task agent with the specified parameters before executing the step
- ORCH-17: Tool Access Validation - Dynamic agent tool access level must be one of: full, read-only, none; the system validates that the requested tools are available
- ORCH-18: Dynamic Agent Lifecycle - Dynamic agents are ephemeral; they are created before step execution and discarded after the step completes
- ORCH-19: Specification Fallback - If a plan step specifies both a static agent ID and a dynamic agent spec, the dynamic spec takes precedence
- ORCH-20: Base Type Constraint - Dynamic agent specifications must declare a base type (coder, researcher, reviewer, chat) that determines the default tool access level

### Applications & Pages

No application or page changes. This feature is internal to the orchestrator and transparent to end users interacting via CLI or HTTP API.

### Non-functional Requirements

- **Data Scale**: Dynamic agents are ephemeral and do not persist; no storage impact. A single plan may have up to the existing max steps limit of dynamic agents.
- **Performance Expectations**: Dynamic agent creation should add negligible overhead (sub-millisecond) compared to agent execution time. The overall orchestration latency is dominated by LLM calls, not agent instantiation.
- **Data Security & Privacy**: Dynamic agents inherit the security context of the orchestrator. Tool access levels constrain what dynamic agents can do. Dynamic agent system prompts are generated by the planner LLM and are not user-controllable.
- **Compatibility**: No breaking changes to existing APIs or configurations. Existing plans without dynamic specs continue to work unchanged.

### Assumptions

1. The planner LLM is capable of generating structured dynamic agent specifications as part of plan output when instructed to do so via an updated planner system prompt.
2. Dynamic agents only need access to existing tool registries (full, read-only, or none); there is no need to create custom tool sets per agent.
3. Dynamic agents do not need session memory or persistent memory -- they operate with working memory only for the duration of their step.
4. The existing `taskagent.New()` API in the vage framework is sufficient for building dynamic agents at runtime.
