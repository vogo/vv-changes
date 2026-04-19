# Requirement: Tool Permission Modes

## 1. Background and Objectives

### Background

The vv agent application currently uses a simple `confirm_tools` configuration (a list of tool names) to determine which tool calls require user confirmation in CLI mode. This mechanism has several limitations:

- It does not distinguish between tools based on their read/write nature. Safe read-only tools (read, glob, grep) may still trigger confirmation if listed.
- There is no concept of "permission modes" -- users cannot switch between different security postures.
- Every invocation of a listed tool requires separate confirmation, even within the same session, causing confirmation fatigue.
- There is no way to restrict the agent to read-only operations (e.g., for code review scenarios).

### Objectives

1. Introduce predefined **permission modes** that automatically classify tool calls based on their read/write/execute nature, replacing manual per-tool confirmation lists.
2. Add a **read-only attribute** to each tool definition so the system can distinguish safe (read-only) tools from potentially dangerous (write/execute) tools.
3. Provide **session-level "Allow Always"** functionality so that once a user approves a specific tool, it does not prompt again for the remainder of the session.
4. Enable a **plan (read-only) mode** that restricts the agent to read-only tools, suitable for code review or exploration scenarios.

## 2. Assumptions

Since this is an automated analysis without interactive clarification, the following assumptions are made:

- **A1**: The `ask_user` tool is classified as read-only because it does not modify the filesystem or execute commands; it only solicits input from the user.
- **A2**: The existing `confirm_tools` configuration will be **deprecated** in favor of `permission_mode`. If both are present, `permission_mode` takes precedence and `confirm_tools` is ignored. A deprecation warning will be shown.
- **A3**: The permission mode feature applies to **CLI mode only**. In HTTP mode, tools are executed without user confirmation (as is the current behavior), since there is no interactive confirmation UI.
- **A4**: The `/permission` runtime command is a CLI-only command, consistent with other CLI commands like `/exit`, `/help`, `/compact`.
- **A5**: Permission modes apply uniformly across all agents dispatched within a session. The mode does not vary per sub-agent.
- **A6**: In "plan" mode, if the orchestrator's plan step requires a write tool, the step is rejected (tool call returns an error to the agent), and the agent must adapt.

## 3. User Stories

### US-01: Use Default Permission Mode

**As a** User (CLI),
**I want** read-only tools to be auto-approved and write/execute tools to require my confirmation,
**So that** I am not interrupted by confirmations for safe operations like file reads and searches.

**Acceptance Criteria:**
- When `permission_mode` is `default`, tools classified as read-only (read, glob, grep, ask_user) execute without confirmation prompts.
- When `permission_mode` is `default`, tools classified as write (write, edit) or execute (bash) show a confirmation dialog before executing.
- The `default` mode is the default value when no `permission_mode` is configured.

### US-02: Use Accept-Edits Permission Mode

**As a** User (CLI) who trusts file edit operations,
**I want** both read-only tools and file edit tools to be auto-approved, with only bash requiring confirmation,
**So that** I can work more efficiently when coding tasks primarily involve file modifications.

**Acceptance Criteria:**
- When `permission_mode` is `accept-edits`, read-only tools (read, glob, grep, ask_user) and file modification tools (write, edit) execute without confirmation.
- When `permission_mode` is `accept-edits`, the bash tool still shows a confirmation dialog.

### US-03: Use Auto Permission Mode

**As a** User (CLI) who fully trusts the agent,
**I want** all tools to be auto-approved without any confirmation prompts,
**So that** I can use the agent without any interruptions.

**Acceptance Criteria:**
- When `permission_mode` is `auto`, all tool calls execute without confirmation prompts.
- No confirmation dialogs are shown regardless of tool type.

### US-04: Use Plan (Read-Only) Permission Mode

**As a** User (CLI) who wants the agent to only observe and analyze without making changes,
**I want** the agent restricted to read-only tools with write/execute tools rejected,
**So that** I can safely use the agent for code review, exploration, or auditing without risk of modifications.

**Acceptance Criteria:**
- When `permission_mode` is `plan`, read-only tools execute without confirmation.
- When `permission_mode` is `plan`, non-read-only tools (write, edit, bash) are automatically rejected without showing a confirmation dialog.
- The agent receives a rejection message indicating the tool is not permitted in the current permission mode.

### US-05: Allow Always (Session-Level Tool Approval)

**As a** User (CLI) who has approved a specific tool once,
**I want** the option to approve it for the rest of my session,
**So that** I do not have to approve the same tool repeatedly during a coding task.

**Acceptance Criteria:**
- The confirmation dialog presents three options: "Allow" (approve this invocation only), "Allow Always" (approve this and all future invocations of this tool in the current session), and "Deny" (reject this invocation).
- When "Allow Always" is selected, subsequent invocations of that tool within the same session execute without showing a confirmation dialog.
- The session-allowed set is cleared when the CLI session ends (application exit or restart).

### US-06: Configure Permission Mode via YAML

**As an** Operator,
**I want** to set the default permission mode in the YAML configuration file,
**So that** the permission policy is pre-configured for my users/environment.

**Acceptance Criteria:**
- The `permission_mode` field is accepted under the `cli` section of `vv.yaml`.
- Valid values are: `default`, `accept-edits`, `auto`, `plan`.
- If omitted, the system uses `default` as the fallback.
- Invalid values produce a clear error message at startup.

### US-07: Configure Permission Mode via Environment Variable

**As an** Operator or User,
**I want** to override the permission mode via an environment variable,
**So that** I can switch modes without editing the configuration file.

**Acceptance Criteria:**
- The environment variable `VV_PERMISSION_MODE` overrides the YAML `permission_mode` setting.
- Valid values are: `default`, `accept-edits`, `auto`, `plan`.

### US-08: Configure Permission Mode via CLI Flag

**As a** User,
**I want** to specify the permission mode when launching vv from the command line,
**So that** I can choose a mode per-session without changing persistent configuration.

**Acceptance Criteria:**
- The `--permission-mode` CLI flag is accepted.
- The flag takes precedence over the environment variable and YAML configuration.
- Valid values are: `default`, `accept-edits`, `auto`, `plan`.

### US-09: Switch Permission Mode at Runtime

**As a** User (CLI),
**I want** to switch the permission mode during a session using a command,
**So that** I can adjust the security posture mid-conversation without restarting.

**Acceptance Criteria:**
- The `/permission <mode>` command is available in the CLI.
- Valid arguments are: `default`, `accept-edits`, `auto`, `plan`.
- When issued without arguments, it displays the current permission mode.
- Switching modes clears the session-allowed tool set.
- A system message confirms the mode change.

## 4. Scope Boundaries

### In Scope

- Tool read-only attribute classification for all built-in tools (bash, read, write, edit, glob, grep, ask_user)
- Four permission modes: `default`, `accept-edits`, `auto`, `plan`
- Permission decision logic replacing the current `confirm_tools` mechanism
- Session-level "Allow Always" with three-option confirmation dialog
- Configuration via YAML, environment variable, and CLI flag
- Runtime mode switching via `/permission` command
- Deprecation of `confirm_tools` with backward compatibility (warn if used)

### Out of Scope

- Per-tool allowlist/blocklist overrides within a permission mode
- Permission modes for MCP or agent-as-tool invocations (future consideration)
- Permission persistence across sessions (e.g., "Always allow write for this project")
- Hook-based pre/post tool execution checks (a more advanced security layer)
- Permission modes for HTTP API mode (no interactive confirmation UI exists)
- Tool-level argument inspection for finer-grained permission decisions (e.g., allow bash only for certain commands)

## 5. Involved System Roles

| Role | Involvement |
|------|-------------|
| Operator | Configures `permission_mode` in YAML; may set `VV_PERMISSION_MODE` env var |
| User (CLI) | Selects permission mode via CLI flag; uses `/permission` command; interacts with updated confirmation dialog (Allow / Allow Always / Deny) |
| System | Evaluates permission decision logic on each tool call; maintains session-allowed set |

## 6. Involved Models and State Changes

### 6.1 Tool (model change)

**New attribute:**

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| read_only | Whether this tool only reads/observes without modifying state | boolean | Yes | Determines auto-approval behavior in permission modes |

**Tool classification:**

| Tool | read_only | Rationale |
|------|-----------|-----------|
| read | true | Reads file content only |
| glob | true | Searches file names only |
| grep | true | Searches file content only |
| ask_user | true | Solicits user input only |
| write | false | Creates or overwrites files |
| edit | false | Modifies file content |
| bash | false | Executes arbitrary commands |

### 6.2 Configuration (model change)

**New attribute:**

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| permission_mode | Tool permission mode for CLI mode | enum (Permission Mode) | No | Default: `default`. Under the `cli` section. Override: `VV_PERMISSION_MODE` env var, `--permission-mode` CLI flag |

**Deprecated attribute:**

| Attribute | Status | Migration |
|-----------|--------|-----------|
| confirm_tools | Deprecated | Replaced by `permission_mode`. If both present, `permission_mode` takes precedence and a warning is logged |

### 6.3 CLI Session (model change)

**New attributes:**

| Attribute | Description | Type | Notes |
|-----------|-------------|------|-------|
| permission_mode | Active permission mode for this session | enum (Permission Mode) | Initialized from config; can be changed via `/permission` command |
| session_allowed_tools | Set of tool names approved via "Allow Always" | set of text | Cleared on session end or mode change |

## 7. Involved Dictionaries

### 7.1 Permission Mode (new dictionary)

| Value | Label | Description | Sort Order |
|-------|-------|-------------|------------|
| default | Default | Read-only tools auto-approved; write and execute tools require confirmation | 1 |
| accept-edits | Accept Edits | Read-only and file edit tools auto-approved; bash requires confirmation | 2 |
| auto | Auto | All tools auto-approved without confirmation | 3 |
| plan | Plan | Read-only tools only; write and execute tools are rejected | 4 |

### 7.2 Confirmation Action (new dictionary)

| Value | Label | Description | Sort Order |
|-------|-------|-------------|------------|
| allow | Allow | Approve this tool invocation only | 1 |
| allow_always | Allow Always | Approve this and all future invocations of this tool in the current session | 2 |
| deny | Deny | Reject this tool invocation | 3 |

## 8. Involved Business Processes

### 8.1 Tool Permission Decision (new process)

Replaces the current `confirm_tools` check in CLI Tool Confirmation.

**Process Steps:**

1. **Receive tool call request**: The system intercepts a tool call from an agent.
2. **Check permission mode**:
   - If mode is `auto`: approve immediately.
   - If mode is `plan` and tool is not read-only: reject immediately with message "Tool [name] is not permitted in plan mode (read-only)."
   - If mode is `default` and tool is read-only: approve immediately.
   - If mode is `accept-edits` and tool is read-only: approve immediately.
   - If mode is `accept-edits` and tool is a file edit tool (write, edit): approve immediately.
3. **Check session-allowed set**: If the tool name is in `session_allowed_tools`, approve immediately.
4. **Show confirmation dialog**: Display the updated three-option dialog (Allow / Allow Always / Deny).
5. **Process user decision**:
   - Allow: approve this invocation.
   - Allow Always: approve this invocation and add the tool name to `session_allowed_tools`.
   - Deny: reject this invocation.

**Decision flowchart:**

```
Tool call received
  |
  v
[mode == auto?] --Yes--> APPROVE
  |No
  v
[mode == plan AND NOT read_only?] --Yes--> REJECT ("not permitted in plan mode")
  |No
  v
[tool is read_only?] --Yes--> APPROVE
  |No
  v
[mode == accept-edits AND tool in {write, edit}?] --Yes--> APPROVE
  |No
  v
[tool in session_allowed_tools?] --Yes--> APPROVE
  |No
  v
SHOW CONFIRMATION DIALOG (Allow / Allow Always / Deny)
  |
  +--> Allow --> APPROVE
  +--> Allow Always --> add to session_allowed_tools --> APPROVE
  +--> Deny --> REJECT
```

### 8.2 CLI Tool Confirmation (modified process)

The existing CLI Tool Confirmation process is modified as follows:

- **Step 1** changes: Instead of checking `confirm_tools` list, invoke the Tool Permission Decision process (8.1 above). The permission decision may result in immediate approval, immediate rejection, or showing the confirmation dialog.
- **Step 2** changes: The confirmation dialog now shows three options instead of two (Allow / Allow Always / Deny).
- **Step 4** changes: If "Allow Always" was selected, the tool name is added to `session_allowed_tools` before resuming execution.
- **Business rule CLICONF-04** changes: Replaced. The triggering condition is now determined by the permission mode and tool read-only classification, not by a static `confirm_tools` list.

### 8.3 CLI Message Processing (modified process)

- **Step 5** changes: The tool confirmation check uses the new permission decision logic instead of checking `confirm_tools`.

### 8.4 Application Startup (modified process)

- Load `permission_mode` from configuration (YAML, then env var override, then CLI flag override).
- Validate `permission_mode` value; error on invalid values.
- If `confirm_tools` is also present, log a deprecation warning.

### 8.5 Runtime Permission Mode Switch (new process)

1. User enters `/permission <mode>` command.
2. Validate the mode value.
3. Update the session's `permission_mode`.
4. Clear `session_allowed_tools`.
5. Display system message: "Permission mode changed to [mode]."

If no argument is provided:
1. Display system message: "Current permission mode: [mode]."

## 9. Involved Applications and Pages

### 9.1 Confirmation Dialog (modified page)

**Changes:**

- **Section 3 (Action Buttons)**: Replace the two-button layout (Approve / Reject) with a three-option selector:
  - **Allow**: Approve this invocation only (equivalent to current Approve).
  - **Allow Always**: Approve this and all future invocations of this tool in the current session.
  - **Deny**: Reject this invocation (equivalent to current Reject).

- **New Feature: Allow Always**:
  - **Trigger**: User selects "Allow Always."
  - **Business Logic**: The tool name is added to `session_allowed_tools`. Execution resumes with approval.
  - **Feedback**: Dialog dismisses. Future invocations of this tool proceed without confirmation for the rest of the session.

### 9.2 Main Conversation View (modified page)

**Changes:**

- The `/permission` command is added to the list of recognized CLI commands.
- Status bar or help text may indicate the current permission mode (optional UX enhancement).

### 9.3 Configuration File (modified)

**New field:**

```yaml
cli:
  permission_mode: default  # default | accept-edits | auto | plan
  # confirm_tools is deprecated; use permission_mode instead
```

## 10. Business Rules Summary

| Rule ID | Rule Name | Description |
|---------|-----------|-------------|
| PERM-01 | Mode Default | If `permission_mode` is not configured, `default` mode is used |
| PERM-02 | Read-Only Auto-Approve | In `default`, `accept-edits`, and `plan` modes, read-only tools are always auto-approved |
| PERM-03 | Plan Mode Rejection | In `plan` mode, non-read-only tools are rejected with a descriptive error message, not silently dropped |
| PERM-04 | Accept-Edits Scope | In `accept-edits` mode, the tools `write` and `edit` are additionally auto-approved (on top of read-only tools) |
| PERM-05 | Session Allowed Persistence | The `session_allowed_tools` set persists for the duration of the CLI session and is cleared on exit or mode change |
| PERM-06 | Config Precedence | Configuration precedence: CLI flag > environment variable > YAML file > default value |
| PERM-07 | Deprecation Warning | If `confirm_tools` is non-empty and `permission_mode` is also set, log a warning that `confirm_tools` is deprecated |
| PERM-08 | Mode Change Clears Allowlist | Switching permission mode via `/permission` clears the session-allowed tool set to ensure the new mode's policies take full effect |
| PERM-09 | CLI-Only | Permission modes and confirmation dialogs apply only in CLI mode. HTTP mode is unaffected. |
