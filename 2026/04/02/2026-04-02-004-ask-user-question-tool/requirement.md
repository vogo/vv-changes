# Requirement: Add `ask_user` Tool for Agent-Initiated User Interaction

## Background and Objectives

Currently, when an agent encounters ambiguity during task execution -- unclear intent, multiple valid approaches, missing critical information -- it must guess or proceed with assumptions. This leads to wasted LLM iterations, incorrect modifications that need reverting, sub-optimal task decomposition, and poor user experience when the agent goes down the wrong path.

The vv CLI already supports a tool confirmation flow (approve/reject dialogs via huh) that pauses agent execution and waits for user input. However, this mechanism is limited to binary yes/no decisions about tool calls. There is no way for the agent to ask a free-form question and receive an open-ended text response.

The objective is to add an `ask_user` tool that enables agents to pause execution and ask the user a clarifying question mid-task. The user's text response is returned as the tool result, allowing the agent to make informed decisions based on actual user intent rather than guessing. This capability is broadly applicable across all agent types (coder, researcher, reviewer, orchestrator) and both interface modes (CLI and HTTP).

## User Stories

### US-1: Agent Asks Clarifying Question in CLI Mode

**As a** developer using the CLI TUI,
**I want** agents to ask me clarifying questions when they encounter ambiguity,
**So that** the agent acts on my actual intent instead of guessing, reducing wasted iterations and incorrect results.

**Acceptance Criteria:**
- When an agent calls the `ask_user` tool, the CLI TUI displays the question prominently in a dialog.
- The dialog includes a text input field where the user can type a free-form response.
- The user submits their response, and it is returned to the agent as the tool result.
- The agent resumes execution using the user's answer to inform its next steps.
- The session status transitions to `awaiting_user_input` while waiting and back to `processing` when the user responds.

### US-2: Agent Asks Clarifying Question in HTTP Mode

**As a** system integrating with vv's HTTP API,
**I want** agents to surface clarifying questions through the API so the client can relay them to the end user,
**So that** HTTP-mode agents can also resolve ambiguity interactively rather than guessing.

**Acceptance Criteria:**
- When an agent calls the `ask_user` tool during an HTTP streaming or async request, the question is emitted as a pending interaction event in the response stream.
- The client can submit the user's answer via a callback endpoint.
- The agent resumes execution once the response is received.
- If the response is not received within the configured timeout, a timeout message is returned as the tool result and the agent proceeds with its best judgment.

### US-3: Graceful Degradation in Non-Interactive Mode

**As a** developer using `vv -p <prompt>` for scripted/automated execution,
**I want** the `ask_user` tool to gracefully degrade when there is no interactive user,
**So that** automated pipelines do not hang waiting for input that will never come.

**Acceptance Criteria:**
- In non-interactive mode (`-p` flag), the `ask_user` tool returns a message such as "Running in non-interactive mode. No user available to answer questions. Proceed with your best judgment."
- The agent receives this response as the tool result and continues execution without pausing.
- No error is raised; the non-interactive fallback is a normal tool result.

### US-4: Configurable Timeout

**As an** operator,
**I want** to configure the timeout for user responses to `ask_user`,
**So that** I can control how long agents wait before proceeding with their best guess.

**Acceptance Criteria:**
- A configuration option `ask_user_timeout` is available in the YAML config file, with a default of 5 minutes.
- If the user does not respond within the timeout, a timeout message is returned as the tool result (e.g., "User did not respond within 5 minutes. Proceed with your best judgment.").
- The agent receives the timeout message and continues execution.

### US-5: Agent System Prompt Guidance

**As an** agent (via system prompt),
**I want** clear guidance on when to use the `ask_user` tool,
**So that** I ask questions at the right moments without over-asking or under-asking.

**Acceptance Criteria:**
- Agent system prompts include instructions to use `ask_user` when:
  - The user's instruction is ambiguous and multiple interpretations exist.
  - Multiple valid approaches exist and the choice significantly affects the outcome.
  - A destructive or irreversible action is about to be taken and the intent is unclear.
  - Critical information (file paths, variable names, scope, conventions) is missing and cannot be reasonably inferred.
- Agent system prompts also include guidance to NOT use `ask_user` when:
  - The answer can be reasonably inferred from context.
  - The question is trivial or would interrupt flow unnecessarily.
  - The agent has already asked a question in the current turn (avoid question chains).

## Scope

### In Scope

- New `ask_user` tool definition with `question` parameter and string return value.
- A `UserInteractor` interface (or equivalent callback mechanism) that the tool executor invokes to collect user input.
- CLI mode implementation: a TUI dialog (extending the existing huh-based confirmation pattern) with a text input field for free-form responses.
- HTTP mode implementation: a pending interaction event emitted in the stream, with a callback endpoint for the client to submit the user's answer.
- Non-interactive mode (`-p` flag) fallback: return a non-interactive message without pausing.
- Configurable timeout with a default of 5 minutes.
- New CLI session status value `awaiting_user_input` for the period while waiting for the user's answer.
- System prompt updates for all agent types to guide appropriate usage of `ask_user`.
- The `ask_user` tool is available to all agent types by default (coder, researcher, reviewer, chat, orchestrator, and dynamically created agents).

### Out of Scope

- Multiple-choice or structured input formats (future enhancement -- the MVP is free-form text only).
- File upload or image input via the `ask_user` tool.
- Automatic detection of when to ask (the LLM decides based on system prompt guidance).
- Rate limiting on how many times an agent can call `ask_user` per request (rely on system prompt guidance).
- Persisting question-answer history as distinct from normal conversation memory.
- Changes to the existing tool confirmation dialog (it remains separate for tool approval).

## Involved System Roles

| Role | Involvement |
|------|-------------|
| User | Receives and answers clarifying questions from agents in CLI TUI or via HTTP client |
| Operator | Configures `ask_user_timeout` in the YAML config |
| Orchestrator Agent | Can invoke `ask_user` during task understanding and decomposition to clarify scope or approach |
| Coder Agent | Can invoke `ask_user` when coding tasks are ambiguous (which file, which approach, scope of changes) |
| Researcher Agent | Can invoke `ask_user` when research scope or focus is unclear |
| Reviewer Agent | Can invoke `ask_user` when review criteria or focus areas are unclear |
| Chat Agent | Can invoke `ask_user` when the user's question is ambiguous |

## Involved Models and State Changes

### New Tool: `ask_user`

Added to the Tool model's MVP tools table:

| Tool Name | Description | Key Parameters |
|-----------|-------------|----------------|
| ask_user | Ask the user a clarifying question when the task is ambiguous or critical information is missing | question |

- **Source**: `local`
- **Available to**: All agent types by default.

### Modified Model: CLI Session Status (Dictionary)

A new status value is added:

| Value | Label | Description | Sort Order | Notes |
|-------|-------|-------------|------------|-------|
| awaiting_user_input | Awaiting User Input | Paused for user response to an agent question | 3 | Text input dialog is displayed |

This is distinct from `awaiting_confirmation` (which is for tool approval). The sort order places it alongside `awaiting_confirmation` as both are "paused waiting for user" states.

### Modified Model: Configuration

A new optional configuration attribute:

| Attribute | Description | Type | Required | Default | Notes |
|-----------|-------------|------|----------|---------|-------|
| ask_user_timeout | How long to wait for a user response to `ask_user` | duration | No | 5m | In non-interactive mode, this is ignored (immediate fallback) |

## Involved Business Processes

### New Process: CLI User Question

Analogous to the existing CLI Tool Confirmation process but with a text input instead of approve/reject buttons.

**Steps:**

1. **Pause Agent Execution**: When the agent calls the `ask_user` tool, pause the agent's execution pipeline. Transition session status to `awaiting_user_input`.
2. **Display Question Dialog**: Display a modal dialog (using huh) showing the agent's question with a text input field for the user's response.
3. **Await User Response**: The user reads the question, types their answer, and submits. If the configured timeout elapses, auto-submit a timeout message.
4. **Resume Agent Execution**: Return the user's text response (or timeout message) as the tool result. Transition session status back to `processing`. The agent continues execution with the answer.

**Business Rules:**

| Rule ID | Rule Name | Rule Description | Applicable Scenario |
|---------|-----------|------------------|---------------------|
| ASKUSR-01 | Display Question | The question dialog must display the agent's question clearly with enough space for reading | Step 2 |
| ASKUSR-02 | Free-Form Input | The text input field accepts any text; there are no validation constraints on the user's response | Step 3 |
| ASKUSR-03 | Timeout Fallback | If the user does not respond within `ask_user_timeout`, return a timeout message as the tool result | Step 3 |
| ASKUSR-04 | Non-Interactive Fallback | In non-interactive mode (`-p` flag), skip the dialog entirely and return a non-interactive fallback message as the tool result immediately | Step 1 |
| ASKUSR-05 | Empty Response | If the user submits an empty response, treat it as a valid response (return empty string); the agent decides how to handle it | Step 3 |

### New Process: HTTP User Question

**Steps:**

1. **Emit Pending Interaction Event**: When the agent calls `ask_user` during an HTTP request, emit a `pending_interaction` event in the SSE stream (or set a pending status on the async task) containing the question text and an interaction ID.
2. **Await Client Response**: The server waits for the client to POST the user's answer to a callback endpoint (`POST /v1/interactions/{interactionID}/respond`). If the configured timeout elapses, proceed with a timeout message.
3. **Resume Agent Execution**: Return the client's response (or timeout message) as the tool result. The agent continues execution.

**Business Rules:**

| Rule ID | Rule Name | Rule Description | Applicable Scenario |
|---------|-----------|------------------|---------------------|
| ASKHTTP-01 | Interaction ID | Each `ask_user` invocation generates a unique interaction ID included in the pending interaction event | Step 1 |
| ASKHTTP-02 | Callback Endpoint | The client submits the user's answer via `POST /v1/interactions/{interactionID}/respond` with a JSON body containing the response text | Step 2 |
| ASKHTTP-03 | Timeout | If no response is received within `ask_user_timeout`, return a timeout message as the tool result | Step 2 |
| ASKHTTP-04 | Single Response | Each interaction ID accepts only one response; subsequent submissions return a 409 Conflict | Step 2 |

### Modified Process: CLI Message Processing

Step 4 (Stream Agent Output) gains a new event type:

- **ask_user**: When the stream emits an `ask_user` event, pause the stream and trigger the CLI User Question process. After the user responds, resume streaming.

This parallels the existing Step 5 (Handle Tool Confirmation) behavior.

### Modified Process: Application Startup

The tool registration phase gains a new tool:

- Register the `ask_user` tool in the shared tool registry.
- The `ask_user` tool's executor is wired to the appropriate `UserInteractor` implementation based on the run mode (CLI, HTTP, or non-interactive).

## Involved Applications and Pages

### CLI Application

**New Page: User Question Dialog**

A modal dialog overlaid on the main conversation view when an agent calls the `ask_user` tool.

| Section | Content | Interactions |
|---------|---------|-------------|
| Dialog Header | Title: "Agent Question"; Agent name that is asking | Display only |
| Question Area | The agent's question text, word-wrapped for readability | Display only |
| Response Input | Multi-line text input field for the user's answer | Type response, submit with Enter or submit button |
| Action Buttons | Submit button | Submit the response; dialog dismisses and agent resumes |

**View States:**

| State | Display |
|-------|---------|
| Displayed | Dialog is visible with question and input field. Main view is visible but inactive behind it. |
| Dismissed | Dialog closes after user submits response. Main view becomes active again. |

### HTTP API Application

**New Endpoint: Submit Interaction Response**

| Endpoint | Method | Description |
|----------|--------|-------------|
| /v1/interactions/{interactionID}/respond | POST | Submit the user's response to a pending `ask_user` interaction |

**Request Body:**
```json
{
  "response": "The user's text response"
}
```

**Response:**
- 200 OK: Response accepted, agent resumes.
- 404 Not Found: Interaction ID does not exist or has expired.
- 409 Conflict: Response already submitted for this interaction.

**Modified Streaming Events:**

A new SSE event type `pending_interaction` is added to streaming responses:

```json
{
  "type": "pending_interaction",
  "interaction_id": "unique-id",
  "question": "The agent's question text",
  "timeout_seconds": 300
}
```

## Assumptions

- The `ask_user` tool does not require inclusion in the `confirm_tools` list -- it always pauses for user input by design. It is not subject to the tool confirmation flow.
- All agent types (coder, researcher, reviewer, chat, orchestrator) have access to `ask_user` by default. It is not gated by tool access level since it is a meta-tool for communication, not a system-affecting operation.
- In CLI mode, the TUI text input field supports basic editing (backspace, cursor movement) consistent with huh's text input capabilities.
- The `ask_user` tool is distinct from the existing tool confirmation mechanism. Tool confirmation asks "should this tool call proceed?" (binary). `ask_user` asks an open-ended question (free-form text). They use separate UI dialogs and separate session status values.
- System prompt guidance is advisory; the LLM may still over-ask or under-ask. Tuning the guidance is an iterative process outside the scope of this requirement.
- The HTTP pending interaction mechanism works with both streaming (SSE) and async modes. For synchronous mode, `ask_user` is not supported and returns a fallback message similar to non-interactive mode, since synchronous HTTP requests cannot support mid-execution interaction.
