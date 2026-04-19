# RouterAgent Enhancement, Coder TaskAgent, Planner WorkflowAgent, Three-Level Memory

## Background & Objectives

vv currently has a basic routing mechanism with two sub-agents (coder and chat). The routing is simple LLM-based intent classification. The coder agent operates as a ReAct-loop task agent with file tools, and the chat agent handles general conversation. There is no support for complex multi-step task planning, no workflow orchestration, and no memory persistence beyond the current CLI session's in-memory conversation.

This change introduces four enhancements:

1. **RouterAgent as the Front Door**: Formalize the router agent as the single entry point for all user input. Expand routing targets to include new sub-agents (planner, researcher, reviewer) in addition to the existing coder and chat agents.
2. **Coder as the Primary TaskAgent**: Reinforce the coder agent as the primary workhorse -- a ReAct-loop agent with full file tool access (read, write, edit, glob, grep, bash). No structural change needed; this is already the current design.
3. **Planner as a WorkflowAgent**: Introduce a planner agent that decomposes complex multi-step tasks into a DAG (Directed Acyclic Graph) of sub-tasks and orchestrates their execution via vage's orchestrate engine. The planner delegates individual sub-tasks to other agents (primarily coder).
4. **Three-Level Memory**: Introduce a memory architecture with three tiers -- working memory (per request), session memory (per conversation), and persistent store (long-term knowledge) -- to enable agents to retain and recall context across different scopes.

**Key Goals:**
- Enable intelligent routing to a richer set of specialized agents
- Support complex multi-step task decomposition and orchestration
- Provide memory persistence so agents can retain knowledge within a session and across sessions
- Maintain backward compatibility with the existing CLI and HTTP interfaces

## User Stories & Acceptance Criteria

### US-1: Expanded Intent Routing

**As a** user,
**I want** my requests to be automatically routed to the most appropriate specialized agent,
**So that** I get the best possible response regardless of task complexity.

**Acceptance Criteria:**
- The router agent classifies user intent and dispatches to one of: coder, planner, researcher, reviewer, or chat
- Coding tasks (file editing, debugging, implementation) route to the coder agent
- Complex multi-step tasks (project setup, large refactors, multi-file changes) route to the planner agent
- Research tasks (codebase exploration, documentation lookup) route to the researcher agent
- Review tasks (code review, design review) route to the reviewer agent
- General conversation routes to the chat agent
- Fallback to chat agent when classification is uncertain

### US-2: Multi-Step Task Planning and Execution

**As a** user,
**I want** complex tasks to be automatically decomposed into steps and executed in the right order,
**So that** I can request large changes without manually breaking them down.

**Acceptance Criteria:**
- When a complex task is routed to the planner, the planner analyzes the task and produces a plan as a DAG of sub-tasks
- Each sub-task specifies the agent to execute it (typically coder) and the task description
- Sub-tasks with no dependencies can execute in parallel
- Sub-tasks with dependencies execute only after their dependencies complete
- The planner reports progress as sub-tasks complete
- If a sub-task fails, the planner can retry or adapt the plan
- The final result aggregates outputs from all sub-tasks into a coherent response

### US-3: Working Memory (Per-Request Context)

**As a** user,
**I want** the agent to maintain context within a single request's execution,
**So that** intermediate results from tool calls and sub-agent responses inform subsequent steps.

**Acceptance Criteria:**
- Each agent request execution has its own working memory scope
- Tool call results, intermediate reasoning, and sub-task outputs are stored in working memory
- Working memory is accessible to the agent throughout the request lifecycle
- Working memory is discarded when the request completes
- For the planner, working memory spans the entire workflow execution (all sub-tasks share the planner's working memory)

### US-4: Session Memory (Per-Conversation Context)

**As a** user,
**I want** the agent to remember what we discussed earlier in the conversation,
**So that** I do not have to repeat context in follow-up messages.

**Acceptance Criteria:**
- Conversation history is stored as session memory and passed to agents as context
- Session memory persists for the duration of the CLI session or HTTP conversation
- Session memory includes summaries of previous exchanges when the full history exceeds the token budget
- Key facts, decisions, and file references from previous exchanges are retained in session memory
- Session memory is cleared when the session ends

### US-5: Persistent Memory (Long-Term Knowledge)

**As a** user,
**I want** the agent to remember important facts about my project across sessions,
**So that** I do not have to re-explain project conventions, architecture, and preferences each time.

**Acceptance Criteria:**
- Agents can store key-value knowledge entries in persistent memory (e.g., project conventions, architecture notes, user preferences)
- Persistent memory is loaded at session start and available to all agents
- Persistent memory survives across CLI sessions and application restarts
- Persistent memory is stored as local files in a configurable directory
- Users can view, edit, and delete persistent memory entries via CLI commands (/memory list, /memory show, /memory delete)
- Persistent memory entries have a namespace (e.g., "project", "user", "conventions") for organization

### US-6: Researcher Agent

**As a** user,
**I want** research-oriented requests to be handled by a specialized researcher agent,
**So that** codebase exploration and information gathering tasks are handled efficiently.

**Acceptance Criteria:**
- The researcher agent has access to read-only tools (read, glob, grep) but not write, edit, or bash
- The researcher can explore the codebase, read documentation, and synthesize findings
- The researcher's findings can be consumed by other agents (e.g., planner delegates research sub-tasks)

### US-7: Reviewer Agent

**As a** user,
**I want** review requests to be handled by a specialized reviewer agent,
**So that** code and design reviews follow consistent standards.

**Acceptance Criteria:**
- The reviewer agent has access to read-only tools (read, glob, grep) and optionally bash (for running tests/linters)
- The reviewer can analyze code, identify issues, and provide structured feedback
- Review results include categorized findings (bugs, style issues, suggestions)

## Scope Boundaries

### In-Scope
- Expanded router agent with five routing targets (coder, planner, researcher, reviewer, chat)
- Planner agent as a workflow agent using DAG-based task decomposition
- Three-level memory architecture (working, session, persistent)
- Researcher agent with read-only tool access
- Reviewer agent with read-only tool access (plus optional bash)
- Persistent memory file storage and CLI commands
- Session memory with summarization for token budget management

### Out-of-Scope
- Distributed agent execution across multiple processes or machines
- User-defined custom agents or plugins
- Memory encryption or access control (all memory is local to the user)
- Memory sharing between different users
- Web UI for memory management
- Automatic memory cleanup or retention policies (manual management only for now)
- Real-time collaboration between agents (agents communicate only through the planner's orchestration)

## System Roles

### Existing Roles Affected

| Role | Impact |
|------|--------|
| Router Agent | Expanded to route to five sub-agents instead of two |
| Coder Agent | No structural change; remains the primary task agent |
| Chat Agent | No change; continues handling general conversation |
| User | Can now interact with planner, researcher, and reviewer agents; can manage persistent memory via CLI commands |
| Operator | New configuration options for memory storage directory and planner settings |

### New Roles

| Role | Definition | Permissions |
|------|------------|-------------|
| Planner Agent | A workflow agent that decomposes complex tasks into sub-task DAGs and orchestrates execution | Can invoke other agents as sub-tasks; no direct tool access |
| Researcher Agent | A read-only task agent specialized for codebase exploration and information gathering | Read-only tools: read, glob, grep |
| Reviewer Agent | A task agent specialized for code and design review | Read-only tools (read, glob, grep) plus optional bash for running tests/linters |

## Models & States

### Agent Model (Updated)

New agent type values needed:
- "workflow" -- for the planner agent (DAG-based orchestration)

New attributes:
| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| sub_agents | Agents this agent can delegate to | reference to Agent list | No | For workflow agents (planner) |
| plan_prompt | Prompt template for plan generation | text | No | For workflow agents |

### Memory Entry Model (New)

Represents a single entry in persistent memory.

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| key | Unique identifier for the memory entry | text | Yes | Scoped within namespace |
| namespace | Category of the memory entry | text | Yes | e.g., "project", "user", "conventions" |
| content | The stored knowledge | text | Yes | Free-form text |
| created_at | When the entry was created | datetime | Yes | |
| updated_at | When the entry was last modified | datetime | Yes | |

### Session Memory Model (New)

Represents the session-level memory for a conversation.

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| session_id | Reference to the session | text | Yes | Links to CLI Session or HTTP conversation |
| facts | Key facts extracted from the conversation | text list | Yes | Maintained by summarization |
| summary | Condensed summary of conversation history | text | No | Generated when history exceeds token budget |
| token_count | Estimated token count of the session memory | number | Yes | Used for budget management |

### Task Plan Model (New)

Represents a planner-generated execution plan.

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| plan_id | Unique plan identifier | text | Yes | |
| goal | The high-level goal being planned | text | Yes | From user request |
| steps | Ordered list of plan steps | reference to Plan Step list | Yes | |
| status | Current plan status | enum (Plan Status) | Yes | |

**States:** pending, executing, completed, failed, cancelled

### Plan Step Model (New)

Represents a single step in an execution plan.

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| step_id | Unique step identifier | text | Yes | |
| description | What this step should accomplish | text | Yes | |
| agent | Agent to execute this step | reference to Agent | Yes | |
| dependencies | Steps that must complete before this step | reference to Plan Step list | No | Empty means no dependencies |
| status | Current step status | enum (Plan Step Status) | Yes | |
| result | Output from step execution | text | No | Populated after execution |

**States:** pending, running, completed, failed, skipped

## Business Processes & Rules

### New Processes

| Process | Description |
|---------|-------------|
| Plan Generation | Planner analyzes a complex task and produces a DAG of sub-tasks |
| Plan Execution | Planner orchestrates sub-task execution respecting dependencies |
| Session Memory Management | Maintain and summarize session-level memory as conversations progress |
| Persistent Memory Management | Store, retrieve, and manage long-term knowledge entries |

### Updated Processes

| Process | Change |
|---------|--------|
| Routing | Expanded route definitions to include planner, researcher, and reviewer agents |
| CLI Message Processing | Integrate session memory into agent context; support /memory commands |
| Application Startup | Load persistent memory at startup; configure memory storage directory |

### Business Rules

| Rule ID | Rule Name | Description |
|---------|-----------|-------------|
| ROUTE-05 | Planner Routing | Requests involving multi-step tasks, project setup, large refactors, or tasks requiring coordination across multiple files/systems route to the planner |
| ROUTE-06 | Researcher Routing | Requests focused on exploring, understanding, or documenting code route to the researcher |
| ROUTE-07 | Reviewer Routing | Requests asking for code review, design review, or quality assessment route to the reviewer |
| PLAN-01 | DAG Validation | A generated plan must form a valid DAG (no cycles) |
| PLAN-02 | Parallel Execution | Steps with no unmet dependencies may execute in parallel |
| PLAN-03 | Failure Handling | If a step fails, the planner may retry, skip dependent steps, or re-plan |
| MEM-01 | Working Memory Scope | Working memory is scoped to a single request execution and discarded afterward |
| MEM-02 | Session Memory Summarization | When session memory exceeds 80% of the token budget, older exchanges are summarized to free tokens |
| MEM-03 | Persistent Memory Loading | Persistent memory entries are loaded into agent context at the start of each session |
| MEM-04 | Persistent Memory Storage | Persistent memory is stored as individual files in a configurable directory |
| MEM-05 | Memory Namespace | Each persistent memory entry belongs to exactly one namespace |

## Applications & Pages

### CLI Application (Updated)

New CLI commands:
| Command | Description |
|---------|-------------|
| /memory list [namespace] | List persistent memory entries, optionally filtered by namespace |
| /memory show <key> | Display the content of a persistent memory entry |
| /memory delete <key> | Delete a persistent memory entry |
| /memory set <namespace> <key> | Create or update a persistent memory entry (opens editor) |

### HTTP API Application (Updated)

New endpoints:
| Endpoint | Method | Description |
|----------|--------|-------------|
| /v1/memory | GET | List persistent memory entries |
| /v1/memory/{namespace}/{key} | GET | Get a specific memory entry |
| /v1/memory/{namespace}/{key} | PUT | Create or update a memory entry |
| /v1/memory/{namespace}/{key} | DELETE | Delete a memory entry |

## Non-functional Requirements

- **Data Scale**: Persistent memory is expected to hold dozens to low hundreds of entries per project. No need for database-level performance optimization; file-based storage is sufficient.
- **Performance Expectations**: Planner plan generation should complete within a single LLM call (a few seconds). Sub-task orchestration latency is dominated by individual agent execution times. Memory loading at session start should add no more than 100ms overhead.
- **Data Security & Privacy**: Persistent memory files are stored locally with the user's filesystem permissions. No encryption is applied (same security model as configuration files). Memory content may contain project-specific information and should not be transmitted externally.
- **Compatibility**: Memory storage uses plain text files and works on all platforms. No new external dependencies required.

## Assumptions

1. **vage framework support**: The vage framework already provides (or will provide) a workflow agent type with DAG orchestration capabilities that the planner can use.
2. **Researcher and reviewer are task agents**: Both use the same ReAct-loop mechanism as the coder, differing only in system prompt and tool access.
3. **Persistent memory directory**: Defaults to `~/.vv/memory/` but is configurable via the configuration file.
4. **Session memory is in-memory only**: Session memory is not persisted to disk. Only persistent memory survives across sessions.
5. **No automatic memory extraction**: For the initial implementation, persistent memory is explicitly managed by the user via commands. Automatic extraction of key facts from conversations is a future enhancement.
6. **Planner delegates, does not execute tools directly**: The planner agent does not have direct tool access. It can only delegate to other agents that have tools.
7. **Single-user context**: Memory (all three levels) is single-user. There is no multi-user memory isolation.
