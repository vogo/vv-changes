# AskUserQuestion Tool — Agent-Initiated User Interaction

## Requirement Description

Add an `ask_user` tool that allows agents to pause execution and ask the user a clarifying question mid-task. The user's response is returned as the tool result, enabling the agent to make informed decisions instead of guessing.

## Gap Analysis (Claude Code vs vv)

**Claude Code**: Has a built-in `AskUserQuestion` tool that agents invoke when they encounter ambiguity. The agent formulates a question, execution pauses, the user sees the question in the terminal and types an answer, and the answer is returned as a tool result. This is heavily used — Claude Code's system prompt explicitly instructs the model to "ask the user" when uncertain rather than guessing.

**vv agent**: No equivalent capability. When an agent is unsure about intent, file paths, approach, or scope, it must guess or proceed with assumptions. This leads to:
- Wasted iterations when the guess is wrong
- Incorrect file modifications that need reverting
- Sub-optimal task decomposition in the orchestrator
- Poor user experience when the agent goes down the wrong path

## Why This Feature

1. **High impact on task success rate**: The single biggest cause of agent failure is acting on wrong assumptions. Asking resolves ambiguity at the source.
2. **Low implementation complexity**: vv already has TUI confirmation dialogs (tool confirmation flow in the CLI). The `ask_user` tool reuses this infrastructure — instead of a yes/no confirmation, it collects free-form text input.
3. **Broadly applicable**: Useful for all agent types (coder, researcher, reviewer, orchestrator).
4. **Cost reduction**: Avoids wasted LLM calls on wrong assumptions.

## Functional Requirements

### Tool Definition

- **Name**: `ask_user`
- **Description**: Ask the user a clarifying question. Use when the task is ambiguous, multiple valid approaches exist, or critical information is missing.
- **Parameters**:
  - `question` (string, required): The question to present to the user.
- **Returns**: The user's text response as a string.

### Behavior

1. When the agent calls `ask_user`, the tool executor pauses the agent loop.
2. **CLI mode**: Display the question prominently in the TUI (similar to tool confirmation, but with a text input field instead of yes/no buttons). Wait for user input. Return the text as the tool result.
3. **HTTP mode**: Return the question as a pending interaction event. The client responds via a callback endpoint. The agent resumes when the response arrives (or times out).
4. The agent receives the user's answer as a tool result and continues execution.

### Constraints

- Timeout: Configurable, default 5 minutes. If the user doesn't respond, return a timeout message (agent should proceed with best guess).
- The tool should be available to all agent types by default.
- In non-interactive mode (`-p` flag), the tool should return a message indicating non-interactive mode (agent must proceed without user input).

## Implementation Suggestions

### 1. Tool Registration (vv/tools/)

Define the `ask_user` tool with JSON Schema parameters. Register it alongside existing tools (bash, read, write, edit, glob, grep).

### 2. Tool Executor

The tool executor needs a callback mechanism to the UI layer:
- Define an interface (e.g., `UserInteractor`) with a method like `AskUser(ctx context.Context, question string) (string, error)`
- CLI mode implements this via the bubbletea TUI
- HTTP mode implements this via a pending-interaction pattern

### 3. CLI Integration

Extend the existing TUI confirmation flow:
- Current: tool confirmation shows tool name + args, user clicks Allow/Deny
- New: `ask_user` shows the question text, user types a free-form response and submits

### 4. Agent System Prompt Update

Add guidance to agent system prompts instructing them to use `ask_user` when:
- The user's instruction is ambiguous
- Multiple valid approaches exist and the choice matters
- A destructive or irreversible action is about to be taken
- Critical information (file paths, variable names, scope) is missing

### 5. Estimated Scope

- New tool definition: ~30 lines
- UserInteractor interface: ~10 lines
- CLI TUI input component: ~50-80 lines (extend existing confirmation dialog)
- HTTP pending-interaction: ~40 lines
- System prompt updates: ~10 lines
- Tests: ~50 lines
- **Total: ~200-250 lines of new code**
