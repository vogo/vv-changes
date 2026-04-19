# Requirement: Orchestrator Context Exploration & Planning

## Background
vv's main agent (orchestrator) needs to behave more intelligently when handling user questions. Currently it routes directly to sub-agents without first understanding the problem context.

## Requirements

### 1. Problem Understanding Phase
The main agent should first analyze the user's question to determine:
- Whether the question is related to the current project
- What kind of context is needed to properly answer/handle the request

### 2. Project Exploration (like Claude Code's Explorer sub-agent)
If the question is project-related, the orchestrator should:
- Explore the current project file structure
- Based on the question, search and read relevant files to build context
- Use a pattern similar to Claude Code's Explorer sub-agent approach:
  - Glob for file patterns
  - Grep for keywords/code references
  - Read relevant files to understand code structure
- Build a context summary of findings before proceeding

### 3. Task Planning (like Claude Code's Plan sub-agent)
After understanding the context, the orchestrator should:
- Plan what tasks need to be done
- Break down complex requests into discrete steps
- Determine which sub-agents or tools to use for each step
- The main agent is responsible for orchestrating and scheduling the execution

### 4. Orchestration Flow
The enhanced flow should be:
1. Receive user input
2. Analyze if project exploration is needed
3. If yes: explore project files, build context
4. Plan the approach/tasks
5. Execute through appropriate sub-agents or directly

## Reference
- Claude Code's Explorer sub-agent: fast agent specialized for exploring codebases using glob patterns, grep searches, and file reads
- Claude Code's Plan sub-agent: software architect agent for designing implementation plans with step-by-step approaches

## Constraints
- The orchestrator is responsible for planning and scheduling (not a separate plan agent)
- Exploration should be efficient - only explore what's needed for the current question
- The existing agent routing (coder, chat, researcher, reviewer) should still work
- Changes should be within the vv agent framework (vage)
