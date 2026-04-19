# CLI Interactive Mode

## Background & Objectives

vv currently operates exclusively as an HTTP service. Users must rely on external HTTP clients (curl, Postman, or custom applications) to interact with agents. This creates friction for developers who want a quick, direct interaction with the AI agent from their terminal.

The objective is to add a CLI interactive mode that allows users to interact with vv agents directly in the terminal using a rich TUI (Terminal User Interface) built with bubbletea and huh. This provides a first-class developer experience without requiring the HTTP server to be running.

**Key Goals:**
- Allow users to start vv in interactive CLI mode as an alternative to HTTP server mode
- Provide a rich terminal UI with input editing, streaming output rendering, and confirmation dialogs
- Bypass the HTTP layer entirely -- invoke agents directly through in-process function calls
- Support the same agent capabilities (routing, coder, chat) available via HTTP

## User Stories & Acceptance Criteria

### US-1: Start vv in CLI interactive mode

**As a** developer,
**I want to** launch vv in CLI interactive mode from the terminal,
**So that** I can interact with agents without starting the HTTP server.

**Acceptance Criteria:**
- Running `vv` without specifying a mode starts in CLI interactive mode (CLI mode is the default)
- Running `vv --mode cli` explicitly starts in CLI interactive mode
- Running `vv --mode http` starts the HTTP server as before
- Configuration loading (YAML, env vars) works the same in both modes
- The CLI mode displays a welcome message and an input prompt upon startup

### US-2: Send messages to agents via CLI

**As a** developer,
**I want to** type a message in the CLI and have it processed by the appropriate agent,
**So that** I can get AI assistance directly in my terminal.

**Acceptance Criteria:**
- User can type a multi-line message in the input area
- Pressing a submit key (Enter) sends the message to the router agent
- The router dispatches to the appropriate sub-agent (coder or chat) as it does for HTTP requests
- The agent invocation is in-process (no HTTP round-trip)

### US-3: View streaming agent output

**As a** developer,
**I want to** see agent output streamed in real time,
**So that** I can follow the agent's reasoning and progress as it executes.

**Acceptance Criteria:**
- Agent text output is displayed incrementally as it is generated
- Tool calls are visually indicated (e.g., tool name and parameters shown)
- Tool results are displayed when they return
- Markdown content is rendered with appropriate terminal formatting
- The output area scrolls automatically as new content arrives

### US-4: Confirm agent actions

**As a** developer,
**I want to** be prompted for confirmation before the agent performs certain actions,
**So that** I can review and approve potentially impactful operations.

**Acceptance Criteria:**
- When the agent requests a tool call that requires confirmation, the agent execution pauses
- A confirmation dialog (using huh) is displayed with action details
- User can approve or reject the action
- On approval, agent execution resumes; on rejection, the agent receives a rejection signal and adapts
- The set of tools requiring confirmation is configurable

### US-5: Navigate conversation history

**As a** developer,
**I want to** scroll through the conversation history within the current session,
**So that** I can review previous exchanges.

**Acceptance Criteria:**
- The output area supports scrolling up and down through the conversation
- New messages appear at the bottom
- The conversation persists for the duration of the CLI session

### US-6: Exit CLI mode

**As a** developer,
**I want to** exit the CLI interactive mode gracefully,
**So that** any in-progress operations are properly terminated.

**Acceptance Criteria:**
- Pressing Ctrl+C or typing an exit command (e.g., `/exit`, `/quit`) initiates shutdown
- In-progress agent execution is cancelled gracefully
- The terminal is restored to its original state upon exit

## Scope Boundaries

### In-Scope
- CLI interactive mode as an alternative to HTTP server mode
- bubbletea-based TUI with input, output, and confirmation views
- Direct (in-process) agent invocation bypassing the HTTP layer
- Streaming output rendering with markdown formatting
- Tool call confirmation dialogs using huh
- Conversation history within a single session
- Mode selection via CLI flag (default: CLI mode)

### Out-of-Scope
- Persistent conversation history across sessions (future)
- CLI mode and HTTP mode running simultaneously
- Configuration editing from within the CLI
- Custom keybinding configuration
- Plugin or extension system for the CLI
- File attachment or image input in CLI mode
- Multi-session or multi-user CLI usage

## System Roles

### Existing Roles Affected

| Role | Impact |
|------|--------|
| Operator | Now chooses between CLI mode and HTTP mode at startup |
| User | Can interact via CLI in addition to HTTP API |
| Router Agent | Same routing logic, invoked in-process instead of via HTTP |
| Coder Agent | Same behavior, now supports confirmation callback for tool calls |
| Chat Agent | Same behavior, output streamed to CLI |

### New Roles

No new roles are introduced. The CLI user is the existing "User" role interacting through a different interface.

## Models & States

### Configuration Model (Updated)

New attributes added:

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| mode | Application run mode | enum (Run Mode) | No | Default: "cli" |
| confirm_tools | Tools requiring user confirmation | text list | No | Default: empty (no confirmation needed) |

### CLI Session Model (New)

Represents a single CLI interactive session.

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| session_id | Unique session identifier | text | Yes | Generated at startup |
| status | Current session status | enum (CLI Session Status) | Yes | |
| conversation | Ordered list of messages | message list | Yes | User and agent messages |
| active_agent_run | Currently executing agent run | reference | No | Null when idle |

**States:** Idle, Processing, Awaiting Confirmation, Shutting Down

### CLI Message Model (New)

Represents a single message in the CLI conversation.

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| role | Message sender | enum (CLI Message Role) | Yes | user, agent, system |
| content | Message content | text | Yes | |
| timestamp | When the message was created | datetime | Yes | |
| tool_calls | Tool calls made in this message | tool call list | No | For agent messages |

## Business Processes & Rules

### New Processes

| Process | Description |
|---------|-------------|
| CLI Startup | Initialize CLI mode: load config, create agents, launch TUI |
| CLI Message Processing | Handle user input, invoke agent, stream output to TUI |
| CLI Tool Confirmation | Pause agent, display confirmation dialog, resume or reject |
| CLI Shutdown | Gracefully terminate CLI session |

### Updated Processes

| Process | Change |
|---------|--------|
| Application Startup | Branch based on `mode` flag -- start HTTP service or CLI TUI |

### Business Rules

| Rule ID | Rule Name | Description |
|---------|-----------|-------------|
| CLI-01 | Default Mode | CLI interactive mode is the default when no mode is specified |
| CLI-02 | Streaming Output | Agent output must be streamed to the TUI incrementally, not buffered until completion |
| CLI-03 | Confirmation Required | Tools listed in `confirm_tools` configuration must pause for user confirmation before execution |
| CLI-04 | Graceful Cancellation | Ctrl+C during agent execution cancels the current run but does not exit the application; a second Ctrl+C exits |
| CLI-05 | Terminal Restoration | On exit, the terminal must be restored to its pre-TUI state (cursor, alternate screen, etc.) |
| CLI-06 | In-Process Invocation | CLI mode invokes agents directly via function calls, not via HTTP |

## Applications & Pages (CLI Views)

### New Application: CLI TUI

Since this is a terminal application rather than a web application, "pages" are mapped to CLI views/screens.

| View | Description | Module |
|------|-------------|--------|
| Main View | Primary conversation interface with input and output areas | CLI |
| Confirmation Dialog | Modal confirmation for tool calls requiring approval | CLI |

## Non-functional Requirements

- **Performance Expectations**: Input-to-first-token latency should feel immediate (under 200ms for the local TUI layer; LLM latency is separate). Streaming output rendering must keep up with the LLM token generation rate without visible lag.
- **Data Security & Privacy**: Same security model as HTTP mode. No conversation data is persisted to disk. API keys are loaded from configuration and never displayed in the TUI.
- **Compatibility**: Must work on macOS and Linux terminals. Windows terminal support is desirable but not required for the initial release. Requires a terminal supporting ANSI escape codes and alternate screen mode.

## Assumptions

The following assumptions were made in the absence of interactive clarification:

1. **CLI mode is the default**: Since vv is primarily a developer tool, CLI interactive mode is the more natural default. HTTP mode is selected explicitly via `--mode http`.
2. **Single Ctrl+C behavior**: First Ctrl+C cancels the current agent run; second Ctrl+C exits the application. This follows common CLI tool conventions.
3. **No simultaneous modes**: CLI and HTTP modes are mutually exclusive in a single process. Running both simultaneously would add complexity without clear value for the MVP.
4. **Confirmation is opt-in**: By default, no tools require confirmation. The operator configures which tools need confirmation via the `confirm_tools` setting. This avoids interrupting the workflow unnecessarily.
5. **Session-scoped conversation**: Conversation history lives only in memory for the current session. Persistent history is a future enhancement.
6. **Markdown rendering in terminal**: Agent output containing markdown is rendered with basic terminal formatting (bold, italic, code blocks with syntax highlighting where possible). Full HTML rendering is not attempted.
