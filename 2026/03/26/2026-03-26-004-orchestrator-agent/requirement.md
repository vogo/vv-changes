# Replace Router Agent with Orchestrator Agent

## Background & Objectives

The current vv architecture uses a Router Agent as the main entry point. The Router performs simple LLM-based intent classification and dispatches the entire request to a single sub-agent. For complex tasks, it relies on routing to the separate Planner Agent, which then decomposes and orchestrates sub-tasks.

This design has limitations:
1. **The Router is a shallow dispatcher** -- it does not understand the task, it only classifies intent. If a task spans multiple agent capabilities (e.g., research first, then code), the Router must pick one agent or route to the Planner as a separate hop.
2. **Two-level indirection** -- complex tasks go Router -> Planner -> sub-agents, adding latency and complexity.
3. **No result aggregation at the top level** -- the Router simply forwards the sub-agent's response; it cannot synthesize or summarize results from multiple agents.
4. **Working directory not captured** -- the current working directory is not explicitly captured and communicated to agents at startup, which limits context awareness.

**Objectives:**
- Replace the Router Agent with an Orchestrator Agent that serves as the main agent
- The Orchestrator understands user instructions, decides whether to handle directly or decompose into sub-tasks
- The Orchestrator dispatches sub-agents for sub-tasks and aggregates results into a coherent response
- Capture the current working directory when vv starts and make it available to the Orchestrator and sub-agents
- Absorb the Planner Agent's orchestration responsibilities into the Orchestrator, eliminating the separate Planner as a routable agent

## User Stories & Acceptance Criteria

### US-01: Orchestrator as Main Agent
**As a** user, **I want** the main agent to understand my instruction and intelligently decide how to fulfill it, **so that** I get coherent, complete responses without needing to know which specialized agent to use.

**Acceptance Criteria:**
1. When the user sends a message, it is received by the Orchestrator Agent (not a Router)
2. The Orchestrator analyzes the instruction to understand intent and complexity
3. For simple tasks (e.g., a general question), the Orchestrator handles it directly or dispatches to a single sub-agent
4. For complex tasks requiring multiple capabilities, the Orchestrator decomposes the task into sub-tasks, dispatches them to appropriate sub-agents, and aggregates results
5. The user receives a single coherent response regardless of how many sub-agents were involved

### US-02: Task Decomposition and Dispatch
**As a** user, **I want** complex tasks to be automatically broken down and assigned to the right specialized agents, **so that** I can issue high-level instructions without manual coordination.

**Acceptance Criteria:**
1. The Orchestrator can decompose a complex instruction into ordered sub-tasks
2. Each sub-task is assigned to the most appropriate sub-agent (coder, researcher, reviewer, or chat)
3. Sub-tasks with dependencies are executed in the correct order
4. Independent sub-tasks may be executed in parallel (up to configured limits)
5. If a sub-task fails, the Orchestrator decides whether to retry, skip, or report the failure

### US-03: Result Aggregation
**As a** user, **I want** the Orchestrator to combine results from multiple sub-agents into a single response, **so that** I receive a unified, coherent answer.

**Acceptance Criteria:**
1. After all sub-tasks complete, the Orchestrator synthesizes results into a single response
2. The response includes relevant outputs from all completed sub-tasks
3. If some sub-tasks failed, the response indicates what succeeded and what failed
4. The aggregated response is presented in a natural, readable format

### US-04: Capture Current Working Directory
**As a** user, **I want** vv to automatically detect my current working directory when it starts, **so that** agents have proper context about where I am working.

**Acceptance Criteria:**
1. When vv starts, it captures the current working directory of the invoking shell
2. The working directory is stored and made available to the Orchestrator and all sub-agents
3. The bash tool uses this captured working directory as its default working directory
4. The working directory is included in agent context so agents understand the user's project location

### US-05: Simple Task Direct Handling
**As a** user, **I want** simple tasks to be handled efficiently without unnecessary decomposition overhead, **so that** response times remain fast for straightforward requests.

**Acceptance Criteria:**
1. For tasks that clearly map to a single agent capability (e.g., "read this file", "what is X?"), the Orchestrator dispatches directly to that agent without creating a formal plan
2. The Orchestrator uses its understanding of the task to decide between direct dispatch and decomposition
3. Direct dispatch has comparable latency to the current Router-based dispatch

## Scope Boundaries

### In Scope
- New Orchestrator Agent type that replaces the Router Agent as the main agent
- Task understanding and complexity assessment logic within the Orchestrator
- Task decomposition capability (absorbing Planner's plan generation)
- Sub-agent dispatch and result aggregation (absorbing Planner's plan execution)
- Capturing current working directory at startup
- Updating the Application Startup process to create an Orchestrator instead of a Router
- Updating CLI Message Processing to invoke the Orchestrator instead of the Router
- Updating the Agent model to support the orchestrator type
- Removing the Router Agent type from the system
- Merging the Planner Agent's orchestration capabilities into the Orchestrator

### Out of Scope
- Changes to sub-agent capabilities (coder, researcher, reviewer, chat remain unchanged)
- Changes to the tool system
- Changes to memory management
- Changes to the HTTP service interface
- Changes to CLI TUI rendering
- New sub-agent types
- User-configurable orchestration strategies

## System Roles

| Role | Impact |
|------|--------|
| Orchestrator Agent (new) | New role replacing both Router Agent and Planner Agent; understands tasks, decomposes, dispatches, aggregates |
| Router Agent | Removed -- replaced by Orchestrator Agent |
| Planner Agent | Removed as a standalone routable agent -- orchestration capabilities absorbed into Orchestrator |
| Coder Agent | Unchanged -- receives tasks from Orchestrator instead of Router/Planner |
| Researcher Agent | Unchanged -- receives tasks from Orchestrator instead of Router/Planner |
| Reviewer Agent | Unchanged -- receives tasks from Orchestrator instead of Router/Planner |
| Chat Agent | Unchanged -- receives tasks from Orchestrator instead of Router/Planner |
| User | Unchanged -- interacts via CLI or HTTP as before |
| Operator | Minor change -- configuration for planner settings now applies to Orchestrator |

## Models & States

| Model | Change Type | Description |
|-------|-------------|-------------|
| Agent | Updated | Add "orchestrator" agent type; remove "router" type; remove "workflow" type (merged into orchestrator); add `working_dir` attribute |
| Agent Type (dictionary) | Updated | Replace "router" and "workflow" values with "orchestrator" |
| Configuration | Updated | Add `working_dir` attribute (auto-detected); rename planner config fields to orchestrator config fields |
| Task Plan | Updated | Now created by Orchestrator Agent instead of Planner Agent |
| Plan Step | Unchanged | Structure remains the same; steps are now managed by Orchestrator |

## Business Processes & Rules

| Process | Change Type | Description |
|---------|-------------|-------------|
| Application Startup | Updated | Capture working directory; create Orchestrator instead of Router; remove separate Planner creation |
| Orchestration (new) | New | Replaces both the Routing process and the Plan Generation/Execution processes; single process for task understanding, optional decomposition, dispatch, and aggregation |
| Routing | Removed | Replaced by the Orchestration process |
| Plan Generation | Removed | Absorbed into the Orchestration process |
| Plan Execution | Removed | Absorbed into the Orchestration process |
| CLI Message Processing | Updated | Invoke Orchestrator Agent instead of Router Agent |
| CLI Startup | Minor update | Pass working directory to Orchestrator context |

### Key Business Rules

| Rule ID | Rule Name | Description |
|---------|-----------|-------------|
| ORCH-01 | Task Understanding | The Orchestrator must analyze the user instruction to understand intent, scope, and complexity before deciding on an execution strategy |
| ORCH-02 | Complexity Assessment | The Orchestrator classifies tasks as simple (single-agent) or complex (multi-agent decomposition required) |
| ORCH-03 | Direct Dispatch | Simple tasks are dispatched directly to the most appropriate sub-agent without formal plan creation |
| ORCH-04 | Task Decomposition | Complex tasks are decomposed into a DAG of sub-tasks with agent assignments and dependencies |
| ORCH-05 | Parallel Execution | Independent sub-tasks may execute in parallel, up to the configured maximum |
| ORCH-06 | Result Aggregation | The Orchestrator synthesizes results from all sub-tasks into a single coherent response |
| ORCH-07 | Failure Handling | On sub-task failure, the Orchestrator may retry, skip dependents, re-plan, or report failure |
| ORCH-08 | Fallback to Chat | If the Orchestrator cannot understand the instruction or all dispatch attempts fail, fall back to the Chat Agent |
| ORCH-09 | Working Directory Context | The captured working directory is included in the Orchestrator's context and passed to sub-agents |
| ORCH-10 | Token Usage Aggregation | Token usage from the Orchestrator's own LLM calls is aggregated with sub-agent usage in the response |

## Applications & Pages

Not applicable -- vv is a CLI tool, not a web/mobile application.

## Non-functional Requirements

- **Performance Expectations**: Simple task dispatch (direct routing) should complete the Orchestrator's decision within 1-2 seconds. Complex task decomposition should complete plan generation within 5 seconds.
- **Data Security & Privacy**: No changes to current security model.
- **Compatibility**: No changes to supported platforms.
