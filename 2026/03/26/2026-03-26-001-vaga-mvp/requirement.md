# vv MVP - Structured Requirement

## Background & Objectives

vv (Vage Agent Application) is a production-ready AI agent application built on the vage framework. The objective of this MVP is to deliver a minimal but fully functional agent system that can:

- Accept user input via HTTP endpoints
- Route requests to the appropriate specialized agent (coder or chat)
- Execute code-related tasks using file/bash tools (coder agent)
- Handle general conversation (chat agent)
- Serve all capabilities over HTTP with sync, streaming, and async modes

The MVP establishes the foundational wiring that all future agents, workflows, and capabilities will build upon.

## User Stories & Acceptance Criteria

### US-1: Configuration Loading
**As** an operator, **I want** vv to load its configuration from a YAML file and environment variables, **so that** I can deploy it with different LLM providers and settings without code changes.

**Acceptance Criteria:**
- Configuration loads from a YAML file (default path: `vv.yaml`)
- Environment variables override YAML values (e.g., `VV_LLM_API_KEY`)
- Configuration includes: LLM provider/model/API key, server listen address, tool permissions, max iterations, token budgets
- Missing required configuration (API key) results in a clear error at startup

### US-2: Tool Registration
**As** a coder agent, **I need** access to file manipulation and shell tools, **so that** I can read, write, edit, search, and execute commands on the user's behalf.

**Acceptance Criteria:**
- The following tools are registered: bash, read, write, edit, glob, grep
- Each tool is instantiated and registered with the shared tool registry
- Tools are configurable (e.g., bash working directory, bash timeout)

### US-3: Coder Agent
**As** a user, **I want** to ask the agent to perform coding tasks, **so that** it can read files, write code, run commands, and search the codebase.

**Acceptance Criteria:**
- Coder agent is a TaskAgent with a code-focused system prompt
- Coder agent has access to all registered tools (bash, read, write, edit, glob, grep)
- Coder agent uses the ReAct loop to iteratively solve coding tasks
- Configurable max iterations and token budget

### US-4: Chat Agent
**As** a user, **I want** to have general conversations with the agent, **so that** it can answer questions and assist with non-coding tasks.

**Acceptance Criteria:**
- Chat agent is a TaskAgent with a conversational system prompt
- Chat agent has no tool access (pure conversational)
- Chat agent provides helpful, accurate responses

### US-5: Router Agent
**As** the system, **I need** to route incoming requests to the correct sub-agent, **so that** coding tasks go to the coder and general questions go to the chat agent.

**Acceptance Criteria:**
- Router agent uses LLM-based routing (via `routeragent.LLMFunc`) to classify intent
- Falls back to the chat agent when routing is uncertain
- Routes are: coder (code/file tasks), chat (general conversation)

### US-6: HTTP Server
**As** a user or external system, **I want** to interact with vv over HTTP, **so that** I can send requests and receive responses via a standard API.

**Acceptance Criteria:**
- HTTP server starts on the configured address (default `:8080`)
- The router agent is registered as the primary agent in the service
- Supports sync (`POST /v1/agents/{id}/run`), streaming (`POST /v1/agents/{id}/stream`), and async (`POST /v1/agents/{id}/async`) execution modes
- Health endpoint (`GET /v1/health`) returns status
- Agent listing (`GET /v1/agents`) returns registered agents

### US-7: CLI Entry Point
**As** an operator, **I want** a single binary entry point, **so that** I can start vv from the command line.

**Acceptance Criteria:**
- `main.go` parses CLI flags (config file path, listen address)
- Initializes configuration, tools, agents, router, and HTTP service
- Starts the HTTP server and handles graceful shutdown on SIGINT/SIGTERM

## Scope Boundaries

### In Scope (MVP)
- Configuration loading (YAML + env vars)
- Tool registration (bash, read, write, edit, glob, grep)
- System prompts for coder and chat agents
- Coder agent (TaskAgent with tools)
- Chat agent (TaskAgent without tools)
- Router agent (LLM-based routing to coder/chat)
- HTTP server (sync, streaming, async)
- CLI entry point with graceful shutdown

### Out of Scope
- Memory management (working, session, persistent)
- Security guardrails (prompt injection, content filter, PII)
- Observability hooks (logging, metrics)
- Planner agent and DAG workflow orchestration
- Researcher and reviewer agents
- MCP server mode
- Evaluation harness
- Skill discovery and management
- Authentication and authorization on HTTP endpoints
- Web search tool

## System Roles

| Role | Description |
|------|-------------|
| Operator | Deploys and configures vv; provides API keys and settings |
| User | Sends requests via HTTP API; receives agent responses |
| Router Agent | Classifies user intent and dispatches to sub-agents |
| Coder Agent | Handles code-related tasks using file/bash tools |
| Chat Agent | Handles general conversational requests |

## Models & States

| Model | Description | States |
|-------|-------------|--------|
| Configuration | Application configuration loaded at startup | N/A (static after load) |
| Agent | An agent instance registered in the service | Registered |
| Tool | A tool registered in the tool registry | Registered |
| HTTP Request | An incoming HTTP request | Received -> Processing -> Completed / Failed |
| Async Task | A background agent execution | Pending -> Running -> Completed / Failed / Cancelled |

## Business Processes & Rules

| Process | Description |
|---------|-------------|
| Application Startup | Load config -> Create LLM client -> Register tools -> Create agents -> Create router -> Start HTTP server |
| Synchronous Request | Receive HTTP request -> Decode -> Route to agent -> Execute -> Return response |
| Streaming Request | Receive HTTP request -> Decode -> Route to agent -> Stream events via SSE -> Complete |
| Async Request | Receive HTTP request -> Decode -> Create task -> Execute in background -> Return task ID |
| Routing | Extract user message -> LLM classifies intent -> Select agent -> Delegate execution |

## Applications & Pages

This is an API-only application (no UI pages). The HTTP API serves as the application interface.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/health` | GET | Health check |
| `/v1/agents` | GET | List registered agents |
| `/v1/agents/{id}` | GET | Get agent details |
| `/v1/agents/{id}/run` | POST | Synchronous agent execution |
| `/v1/agents/{id}/stream` | POST | Streaming agent execution (SSE) |
| `/v1/agents/{id}/async` | POST | Asynchronous agent execution |
| `/v1/tools` | GET | List registered tools |
| `/v1/tasks/{taskID}` | GET | Get async task status |
| `/v1/tasks/{taskID}/cancel` | POST | Cancel async task |

## Non-functional Requirements

- **Performance**: Sync requests should add less than 100ms overhead on top of LLM response time
- **Reliability**: Graceful shutdown must complete in-flight requests before exiting
- **Configuration**: All secrets (API keys) must be configurable via environment variables, never hardcoded
- **Logging**: Structured logging (slog) for startup, request handling, and errors
