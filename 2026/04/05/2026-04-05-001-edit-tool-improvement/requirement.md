# Requirement: Edit Tool Safety and Compliance Improvements

## 1. Background

The `vage/tool/edit/` package provides a file edit tool used by LLM agents to perform exact string replacements in files. The reference specification (`doc/claude-code-prd/tools/file-edit.md`) documents a set of safety features and constraints that a production-grade edit tool should enforce. A gap analysis between the current implementation (`vage/tool/edit/edit_tool.go`) and the reference reveals several missing or under-specified behaviors.

The shared path validation utility (`vage/tool/toolkit/path.go`) currently validates that paths are absolute and within allowed directories, but does not address platform-specific attack vectors such as UNC paths on Windows.

## 2. Objectives

- Align the edit tool's safety characteristics with the reference specification.
- Increase the default maximum file size to match the reference constant (1 GiB).
- Introduce UNC path blocking to prevent network-based path traversal attacks.
- Add a deny-rules mechanism so that certain files or patterns can be protected from edits.
- Add a file-read prerequisite check so that the edit tool can optionally enforce that a file has been read before being edited (preventing blind edits).
- Improve error messages to include actionable guidance.

## 3. Improvement Items

### 3.1 Increase Default Max File Size Constant

**As an** agent framework consumer,
**I want** the default maximum editable file size to be 1 GiB (1,073,741,824 bytes),
**so that** the tool can handle large files consistent with the reference specification.

**Acceptance Criteria:**
- The constant `defaultMaxEditFileBytes` is changed from `1 * 1024 * 1024` (1 MB) to `1 * 1024 * 1024 * 1024` (1 GiB).
- The `WithMaxFileBytes` option continues to allow callers to override this default.
- The `maxFileBytes` field type is changed from `int` to `int64` to safely represent sizes up to 1 GiB and beyond.
- Existing tests for the `WithMaxFileBytes` option continue to pass.

### 3.2 UNC Path Blocking

**As an** agent framework consumer,
**I want** the edit tool to reject UNC paths (e.g., `\\server\share\file.txt` or `//server/share/file.txt`),
**so that** agents cannot be tricked into editing files on remote network shares.

**Acceptance Criteria:**
- Paths starting with `\\` or `//` are rejected with a clear error message before any file I/O occurs.
- This check is implemented in the shared `toolkit.ValidatePath` function so that the read, write, and edit tools all benefit.
- Unit tests cover both `\\server\share` and `//server/share` path formats.

### 3.3 Deny Rules (File Protection Patterns)

**As an** agent framework consumer,
**I want** to configure a list of deny rules (glob patterns or path prefixes) that prevent edits to matching files,
**so that** sensitive files (e.g., `.env`, credentials, lock files) are protected from agent modifications.

**Acceptance Criteria:**
- A new functional option `WithDenyRules(patterns ...string)` accepts glob patterns (e.g., `*.env`, `**/credentials.json`, `*.lock`).
- When a file path matches any deny rule, the edit is rejected with an error message that names the matched rule.
- Deny rule evaluation happens after path validation and before any file I/O.
- Unit tests cover: exact match, glob wildcard match, no-match pass-through.

### 3.4 File-Read Prerequisite Check

**As an** agent framework consumer,
**I want** the edit tool to optionally enforce that a file has been read (via the read tool) before it can be edited,
**so that** agents do not make blind edits to files they have not inspected.

**Acceptance Criteria:**
- A new interface `ReadTracker` is defined with a method `HasRead(path string) bool`.
- A new functional option `WithReadTracker(tracker ReadTracker)` allows injecting a tracker.
- When a tracker is configured and `HasRead` returns false for the target file, the edit is rejected with a clear error message advising the agent to read the file first.
- When no tracker is configured, the check is skipped (backward compatible).
- The read tool records file reads into the tracker when one is provided.
- Unit tests cover: edit rejected when file not read, edit allowed after file read, edit allowed when no tracker configured.

### 3.5 Write Permission Pre-Check

**As an** agent framework consumer,
**I want** the edit tool to verify write permission on the target file before attempting the edit,
**so that** the error message is clear and specific when the file is read-only, rather than surfacing a generic OS write error.

**Acceptance Criteria:**
- After `os.Stat` succeeds, check that the file mode includes write permission for the current user.
- If the file is not writable, return an error message: `"edit tool: file is not writable: <path>"`.
- Unit tests cover: read-only file returns the expected error.

### 3.6 Improved Error Messages

**As an** agent framework consumer,
**I want** error messages to include actionable guidance,
**so that** the LLM agent can self-correct more effectively.

**Acceptance Criteria:**
- When `old_string` is not found, the error message suggests possible causes (e.g., whitespace mismatch, file may have changed since last read).
- When the file exceeds the maximum size, the error message includes both the file size and the limit.
- When the path is rejected by deny rules, the error message names the matched pattern.
- All error messages consistently use the `"edit tool: "` prefix.

## 4. Scope

### In-Scope

- `vage/tool/edit/edit_tool.go` -- main edit tool implementation.
- `vage/tool/toolkit/path.go` -- shared path validation (UNC blocking).
- `vage/tool/edit/edit_tool_test.go` -- unit tests for the edit tool.
- `vage/tool/toolkit/path_test.go` -- unit tests for path validation (if exists, or new).
- New `ReadTracker` interface definition (location to be determined during design).
- `vage/tool/read/read_tool.go` -- read tool integration for `ReadTracker` recording.

### Out-of-Scope

- Changes to the write tool (`vage/tool/write/`) -- it may benefit from similar improvements, but that is a separate effort.
- "Team memory secret guard" mentioned in the reference -- this is a higher-level concern handled by the `guard/` subsystem, not the edit tool itself.
- Prompt/description text changes for LLM instruction alignment -- this is a prompt engineering concern, not a tool logic concern.
- Changes to the glob, grep, or bash tools.
- Integration tests requiring LLM API keys.

## 5. Involved Components

| Component | Path | Change Type |
|-----------|------|-------------|
| Edit tool | `vage/tool/edit/edit_tool.go` | Modify |
| Edit tool tests | `vage/tool/edit/edit_tool_test.go` | Modify |
| Path validation | `vage/tool/toolkit/path.go` | Modify |
| Path validation tests | `vage/tool/toolkit/path_test.go` | Modify or Create |
| Read tool | `vage/tool/read/read_tool.go` | Modify (ReadTracker recording) |
| ReadTracker interface | `vage/tool/toolkit/` (new file) | Create |
