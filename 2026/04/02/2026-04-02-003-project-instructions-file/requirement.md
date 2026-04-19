# Requirement: Project Instructions File (VV.md)

## Background & Objectives

### Background

vv currently relies solely on global configuration (`~/.vv/vv.yaml` + environment variables) for all settings. There is no mechanism for users to provide project-specific instructions that customize agent behavior per project. Each project may have unique conventions, architecture patterns, coding standards, and domain context that agents need to understand to be effective.

Comparable tools (e.g., Claude Code with `CLAUDE.md`) have demonstrated the value of project-level instruction files that are automatically loaded and injected into agent system prompts, allowing the AI to adapt its behavior to project-specific needs.

### Objectives

1. Allow users to place a `VV.md` file in their project's working directory to provide project-specific instructions to all agents.
2. Automatically detect and load this file at startup, injecting its content into agent system prompts.
3. Maintain full backward compatibility -- when no `VV.md` exists, behavior is unchanged.

## User Stories

### US-1: Project Instructions via VV.md

**As a** developer using vv in a project directory,
**I want to** place a `VV.md` file in my project root containing project-specific instructions,
**So that** all agents understand my project's conventions, architecture, and coding standards without me repeating them in every conversation.

**Acceptance Criteria:**
- When `VV.md` exists in the working directory, its content is loaded at startup.
- The content is injected into the system prompt of all agents (coder, researcher, reviewer, chat, explorer, planner).
- The injected content is clearly delimited (e.g., wrapped in a `# Project Instructions` section) so agents understand it is user-provided project context.
- The instructions take high priority -- agents should follow project instructions over their default behaviors when applicable.

### US-2: Graceful Absence

**As a** developer using vv in a directory without a VV.md file,
**I want** vv to start normally without any warnings or errors,
**So that** the feature is non-intrusive for projects that don't use it.

**Acceptance Criteria:**
- When `VV.md` does not exist in the working directory, startup proceeds without error or warning.
- All agent behavior remains identical to the current behavior.
- No empty "Project Instructions" section appears in system prompts.

### US-3: Markdown Content Support

**As a** developer writing a VV.md file,
**I want to** use Markdown formatting (headings, lists, code blocks, etc.),
**So that** I can structure my instructions clearly for agent consumption.

**Acceptance Criteria:**
- The file is read as plain text (Markdown) and injected verbatim.
- No special parsing or transformation is applied to the content.
- The full file content is included regardless of size.

## Scope

### In Scope

- Reading `VV.md` from the working directory (`cfg.Tools.BashWorkingDir`) at startup
- Storing the loaded content in the `Config` model as a new attribute
- Injecting the content into all agent system prompts during agent creation
- Wrapping the content with a clear delimiter (e.g., `# Project Instructions` heading)
- Silent skip when the file does not exist

### Out of Scope

- Global-level instructions file (`~/.vv/VV.md`) -- future enhancement
- Multi-level/hierarchical instruction files (parent directories, subdirectories)
- Hot-reloading of VV.md during a running session
- File size limits or content validation
- Alternative file names or formats (e.g., `.vv.md`, `vv.yaml` instructions section)
- VV.md file creation/editing commands within vv
- Caching boundaries for static vs. dynamic prompt content (API-level optimization)

## Involved Roles

| Role | Impact |
|------|--------|
| User (Developer) | Creates and maintains `VV.md` files in project directories |
| Operator | No configuration changes needed; feature is automatic |
| All Agents (Coder, Researcher, Reviewer, Chat, Explorer, Planner) | Receive project instructions in their system prompts |

## Involved Models

### Configuration Model

**Change**: Add a new attribute `project_instructions` (type: text, optional).

| Attribute | Description | Type | Required | Notes |
|-----------|-------------|------|----------|-------|
| project_instructions | Content loaded from the project's VV.md file | text | No | Empty string when no VV.md exists; populated at startup after working directory capture |

**State Change**: During Application Startup, after the working directory is captured, the `project_instructions` attribute transitions from empty to populated (if `VV.md` exists).

### Agent Model

No structural changes. Agents already have system prompts. The project instructions content will be appended to each agent's existing system prompt during creation.

## Involved Processes

### Application Startup (Modified)

A new step is inserted between the existing "Capture Working Directory" (Step 2) and "Load Configuration" (Step 3), or immediately after configuration is fully resolved and the working directory is known:

**New Step: Load Project Instructions**
- **Executing Role**: System
- **Description**: After the working directory is determined, attempt to read `VV.md` from the working directory. If the file exists, store its content in `Config.ProjectInstructions`. If the file does not exist, silently skip.
- **Input**: Working directory path (from `cfg.Tools.BashWorkingDir`)
- **Output**: Project instructions content (string, may be empty)
- **Model State Changes**: `Configuration.project_instructions` populated
- **Error Handling**: File not found is silently ignored. File read errors (permission denied, I/O error) should log a warning and proceed with empty instructions.

### Agent Creation (Modified)

All agent factory functions must incorporate project instructions into their system prompts:

- **When project_instructions is non-empty**: Append the content to the agent's base system prompt, wrapped with a delimiter section:
  ```
  <base system prompt>

  # Project Instructions

  IMPORTANT: The following are project-specific instructions provided by the user. These instructions should be followed and take precedence over default behaviors when applicable.

  <VV.md content>
  ```
- **When project_instructions is empty**: No change to the existing system prompt.

This affects the factory functions for all six agents: coder, researcher, reviewer, chat, explorer, planner.

## Business Rules

| Rule ID | Rule Name | Description | Applicable Scenario |
|---------|-----------|-------------|---------------------|
| PROJ-01 | File Name Convention | The project instructions file must be named `VV.md` (case-sensitive) | Startup file detection |
| PROJ-02 | File Location | The file is looked up only in the resolved working directory (`cfg.Tools.BashWorkingDir`) | Startup file detection |
| PROJ-03 | Silent Absence | When VV.md does not exist, no error or warning is produced and no project instructions section is injected | Startup, Agent creation |
| PROJ-04 | Verbatim Injection | File content is injected as-is without parsing, filtering, or transformation | Agent creation |
| PROJ-05 | High Priority | Project instructions are marked as high-priority context that should override default agent behaviors when applicable | Agent system prompt assembly |
| PROJ-06 | Universal Application | Project instructions are injected into ALL agents, not selectively | Agent creation |
| PROJ-07 | Read Error Graceful Handling | If VV.md exists but cannot be read (permissions, I/O error), log a warning and proceed with empty instructions | Startup file detection |

## Involved Applications & Pages

### CLI Mode
- No UI changes. The feature is transparent to the CLI TUI.

### HTTP Mode
- No API changes. The feature is transparent to the HTTP API.

## Implementation Hints

These are observations from codebase analysis, not prescriptive implementation details:

1. **Loading point**: `setup.Init()` in `vv/setup/setup.go` captures the working directory. Project instructions loading should occur after `cfg.Tools.BashWorkingDir` is resolved, using `filepath.Join(cfg.Tools.BashWorkingDir, "VV.md")`.

2. **Storage**: The `configs.Config` struct or the `registries.FactoryOptions` struct can carry the project instructions string.

3. **Injection pattern**: The existing `PersistentMemoryPrompt` in `vv/agents/memory_prompt.go` demonstrates the pattern of appending dynamic content to base system prompts. A similar approach can append project instructions.

4. **Affected files** (estimated 3-5):
   - `vv/configs/config.go` -- add `ProjectInstructions` field to Config or a new struct
   - `vv/setup/setup.go` -- load VV.md after working dir capture
   - `vv/registries/registry.go` -- add `ProjectInstructions` to `FactoryOptions`
   - `vv/agents/*.go` -- each agent factory incorporates project instructions into system prompt

5. **No new dependencies**: Uses only `os.ReadFile` and string concatenation.
