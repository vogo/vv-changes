# Requirement: TUI Tool Execution Logging

## Problem
Currently the TUI (terminal user interface) does not show tool execution logs in the command line. When agents execute tools like read/write/update files, the user cannot see these operations in the TUI output.

## Desired Behavior
- View logs in TUI about tool execution, such as read/write/update files
- Tool execution logs should be visible alongside the existing phase/agent output
- The logging should integrate with the current TUI display format shown in the example log output

## Reference: Current TUI Output Format
```
vv · openai · doubao-1-5-pro-32k-250115
  ~/workspaces/github/vogo/vagents/vv

● Phase 1/3: Explore
● Explore phase complete.
● Phase 2/3: Plan
● Plan phase complete.
● Phase 3/3: Dispatch
● chat
Agent: [response text]
● chat
  └ Done (11s)
● Dispatch phase complete.
```

## Expected Enhancement
Tool execution logs should appear in the TUI, for example:
```
● Phase 1/3: Explore
  ○ read: cli/cli.go
  ○ read: agents/orchestrator.go
  ○ glob: **/*.go
● Explore phase complete.
```
