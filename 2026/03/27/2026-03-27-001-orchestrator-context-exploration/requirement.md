# Requirement: Orchestrator Context Exploration and Planning

## 1. Background

The vv orchestrator agent currently receives a user request and immediately classifies it into either a direct single-agent dispatch or a multi-step plan. The classification is based solely on the text of the user's message, with no awareness of the project's actual file structure, codebase conventions, or relevant source code context.

This means the orchestrator makes routing and planning decisions "blind" -- it cannot distinguish between a question about the current project vs. a general programming question, and it cannot build the contextual understanding needed to produce accurate plans or select the right sub-agent for project-specific work.

By contrast, tools like Claude Code first explore the codebase (using glob, grep, and file reads) to build context before deciding on an approach. This exploration phase produces significantly better task decomposition and more targeted sub-agent instructions.

## 2. Objectives

- Enable the orchestrator to determine whether a user request is related to the current working directory / project.
- When project-related, allow the orchestrator to explore the codebase and build a context summary before routing or planning.
- Improve the quality of task plans by grounding them in actual project structure and code.
- Keep the orchestrator as the single point of control for both exploration and planning (no separate "explorer agent" or "plan agent" as top-level peers).

## 3. User Stories

### US-1: Project-Relevance Detection

**As** a user working in a project directory,
**I want** the orchestrator to recognize when my question is about the current project,
**so that** it can gather project context before attempting to answer or delegate.

**Acceptance Criteria:**
- The orchestrator determines whether the user's request is project-related or general.
- General questions (e.g., "explain what a goroutine is") skip exploration and route directly as they do today.
- Project-related questions (e.g., "add a new sub-agent for testing") trigger the exploration phase.

### US-2: Project Exploration

**As** a user asking a project-related question,
**I want** the orchestrator to explore my project files and understand the relevant code,
**so that** subsequent actions are grounded in the actual codebase.

**Acceptance Criteria:**
- The orchestrator uses read-only tools (glob, grep, read) to explore the working directory.
- Exploration is scoped to what is relevant for the current question -- it does not exhaustively scan the entire project.
- The exploration produces a context summary (key files, structures, patterns discovered) that is passed downstream.
- Exploration has a bounded cost: a maximum number of tool-call iterations or token budget to prevent runaway exploration.

### US-3: Context-Informed Planning and Routing

**As** a user with a complex project-related request,
**I want** the orchestrator to plan tasks based on its exploration findings,
**so that** sub-agents receive well-scoped instructions with relevant context.

**Acceptance Criteria:**
- The classification/planning LLM call receives the exploration context summary as input.
- For direct dispatch, the sub-agent request is enriched with the exploration summary.
- For plan mode, each step's description references specific files, functions, or patterns discovered during exploration.
- The plan quality is improved: steps are grounded in actual project structure rather than assumptions.

### US-4: Backward Compatibility

**As** a user who sends non-project questions or uses the chat agent,
**I want** the existing routing behavior to remain unchanged,
**so that** general conversations and simple tasks are not slowed down by unnecessary exploration.

**Acceptance Criteria:**
- Non-project requests skip the exploration phase entirely.
- The existing sub-agents (coder, researcher, reviewer, chat) continue to work without modification.
- The fallback behavior (classification failure -> chat agent) is preserved.
- Streaming support (RunStream) continues to work for all paths.

## 4. Scope

### In-Scope

- Adding an exploration phase to the orchestrator's processing pipeline (between receiving the request and classifying/planning).
- Giving the orchestrator access to read-only tools (glob, grep, read) for exploration.
- Modifying the orchestrator's system prompt and classification logic to incorporate exploration context.
- Enriching sub-agent requests with exploration findings.
- Bounding exploration cost (iteration limit, token budget).

### Out-of-Scope

- Creating a separate standalone "explorer agent" as a new top-level peer agent. The exploration capability lives within the orchestrator's own pipeline.
- Modifying the existing sub-agents (coder, researcher, reviewer, chat) -- their prompts, tools, and behavior remain as-is.
- Persistent caching of exploration results across sessions.
- Changes to the vage framework itself (agent, orchestrate, tool packages). All changes should be within the vv application layer.
- UI/CLI changes -- the user interaction model remains the same.

## 5. Key Components Affected

### 5.1 OrchestratorAgent (`vv/agents/orchestrator.go`)

The primary component to change. The orchestrator's `Run` and `RunStream` methods need a new phase inserted before `classifyTask`:

1. **Relevance check**: Determine if the request is project-related (can be part of the same LLM call or a lightweight heuristic).
2. **Exploration phase**: If project-related, use read-only tools to explore the codebase and produce a context summary.
3. **Enhanced classification**: Pass the context summary into the classification/planning prompt so the LLM makes better routing and planning decisions.

The orchestrator currently has no tool registries. It will need access to a read-only tool registry (glob, grep, read) for the exploration phase.

### 5.2 OrchestratorAgent Constructor and Agent Creation (`vv/agents/agents.go`)

The `NewOrchestratorAgent` constructor and the `Create` function need to accept and wire a read-only tool registry into the orchestrator.

### 5.3 Orchestrator System Prompt (`OrchestratorSystemPrompt` in `orchestrator.go`)

The system prompt needs to be updated to:
- Instruct the LLM on when and how to request exploration.
- Accept and reason about exploration context in the classification/planning phase.

### 5.4 Request Enrichment (`enrichRequest` in `orchestrator.go`)

The `enrichRequest` method currently only prepends the working directory. It should also prepend the exploration context summary so sub-agents have full context.

### 5.5 Tool Registration (`vv/tools/tools.go`)

No new tools are needed. The existing `RegisterReadOnly` function already provides the glob, grep, and read tools needed for exploration. This registry just needs to be passed to the orchestrator.

## 6. Integration Design Notes

### 6.1 Exploration as an Internal Orchestrator Phase

The exploration is not a sub-agent dispatch. It is an internal phase of the orchestrator itself, similar to how the classification LLM call is internal today. The orchestrator uses tools directly (via the taskagent pattern or a lightweight tool-calling loop) to gather context before making its routing decision.

### 6.2 Two-Phase LLM Interaction

The orchestrator's interaction with the LLM becomes two phases:
- **Phase 1 (Exploration)**: The LLM is given the user's question and read-only tools. It explores the codebase and produces a context summary. This phase is bounded by an iteration limit.
- **Phase 2 (Classification/Planning)**: The existing classification call, now enriched with the exploration context. This phase produces the same `ClassifyResult` structure as today.

Alternatively, these could be combined into a single multi-turn tool-calling session where the LLM first explores and then produces the classification JSON. The choice is an implementation detail.

### 6.3 Exploration Scope Control

To keep exploration efficient:
- A maximum iteration count (e.g., 5-10 tool calls) bounds the exploration.
- The system prompt should instruct the LLM to focus on files and patterns directly relevant to the user's question.
- A token budget for the exploration phase prevents excessive cost.

### 6.4 Context Summary Format

The exploration phase should produce a structured text summary containing:
- Project type and key configuration files found.
- Relevant source files and their roles.
- Key code structures (types, functions, interfaces) related to the question.
- Any patterns or conventions observed.

This summary is prepended to the classification prompt and to sub-agent requests.

## 7. Non-Functional Requirements

- **Latency**: The exploration phase adds latency. For non-project questions, this overhead must be zero (skip exploration entirely). For project questions, exploration should complete within a reasonable bound (controlled by iteration/token limits).
- **Cost**: Exploration uses LLM tokens. The iteration and token budgets should be configurable.
- **Testability**: The exploration phase should be testable in isolation, with mock tool registries providing canned file contents.
