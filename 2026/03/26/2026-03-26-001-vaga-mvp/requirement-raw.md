# Build vv — A Full-Ability AI Agent (MVP)

## Scope: Working MVP

Implement a working MVP with: config + coder agent + chat agent + router + HTTP server.

## What is vv?

vv (Vage Agent Application) is a production-ready, full-ability AI agent built on the vage framework. It wires together all vage subsystems into a single deployable agent that can:

- Converse naturally with multi-turn memory
- Use tools (bash, read/write/edit files, glob, grep, and MCP remote tools)
- Route tasks to specialized sub-agents
- Orchestrate multi-step workflows via DAG
- Guard inputs/outputs with security guardrails
- Serve over HTTP (sync, streaming, async)
- Evaluate its own quality

## Architecture

```
vv/
├── main.go              # Entry point: CLI + HTTP server
├── config.go            # Configuration (env, flags, YAML)
├── app.go               # Application wiring: assembles all vage components
├── agents/
│   ├── coder.go         # Code-focused TaskAgent (read/write/edit/bash/grep/glob)
│   ├── planner.go       # Planning agent (breaks tasks into DAG workflows)
│   ├── researcher.go    # Research agent (web search, file analysis)
│   ├── reviewer.go      # Code review agent (quality, security checks)
│   └── chat.go          # General chat agent (conversational)
├── tools/
│   ├── register.go      # Registers all built-in tools into a Registry
│   └── web.go           # Web search/fetch tool (if needed)
├── prompts/
│   ├── system.go        # System prompts for each agent role
│   └── templates.go     # Reusable prompt templates
├── router.go            # RouterAgent: dispatches to coder/planner/researcher/chat
├── workflow.go          # Pre-defined workflows (e.g., "implement feature" DAG)
├── guard.go             # Guardrail chain setup (injection, PII, content filter)
├── memory.go            # Memory setup (working + session + persistent store)
├── hooks.go             # Observability hooks (logging, metrics)
└── CLAUDE.md            # Dev guide
```

## Key Design Decisions

1. RouterAgent as the front door — All user input enters a RouterAgent that classifies intent and dispatches to the right sub-agent (coder, planner, researcher, reviewer, or chat).
2. Coder as the primary TaskAgent — A ReAct-loop agent with all file tools (read, write, edit, glob, grep, bash). This is the workhorse.
3. Planner uses WorkflowAgent — For complex multi-step tasks, the planner decomposes into a DAG and orchestrates via vage's orchestrate engine.
4. Three-level memory — Working memory per request, session memory per conversation, persistent store for long-term knowledge.
5. Guard chain on all inputs/outputs — Prompt injection detection, content filtering, length limits.
6. LLM middleware stack — Logging → Circuit breaker → Rate limiting → Retry → Timeout → Cache → Metrics.
7. HTTP service — Exposes all agents via REST with sync/streaming/async modes.
8. MCP support — Can consume external MCP tool servers and expose itself as an MCP server.
9. Configuration-driven — Model provider, API keys, tool permissions, guardrail settings, memory backends all via config file + env vars.

## Implementation Phases (MVP Focus: Phases 1-6)

| Phase | What | Priority |
|-------|------|----------|
| 1 | config.go + app.go — Core wiring, config loading | Must |
| 2 | tools/register.go — Register all built-in tools | Must |
| 3 | prompts/system.go — System prompts per agent role | Must |
| 4 | agents/coder.go + agents/chat.go — The two essential agents | Must |
| 5 | router.go — RouterAgent dispatching | Must |
| 6 | main.go — CLI entry point + HTTP server | Must |
| 7 | guard.go + memory.go + hooks.go — Security, memory, observability | Should |
| 8 | agents/planner.go + workflow.go — DAG orchestration | Should |
| 9 | agents/researcher.go + agents/reviewer.go — Specialized agents | Nice |
| 10 | MCP server mode, eval harness, skill discovery | Nice |
