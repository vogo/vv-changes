# Requirement: Dangerous Command Rule Library + Graded Approval Trigger

## Background & Objectives

The vv agent currently gates **every** non-read-only bash invocation through the same three-state confirmation dialog (Allow / Allow-Always / Deny). This is:

- **Unsafe for automation**: in `auto` and `accept-edits` modes, destructive commands (`rm -rf /`, `sudo`, `curl | bash`) run without friction.
- **High-friction for safe use**: every `ls`, `git status`, or `go test` triggers a dialog in `default` mode.
- **Prone to over-permissive habits**: users hit "Allow Always" on a broad tool name just to silence prompts, then later dangerous calls execute under that allowance.

Goal: classify each bash command by risk tier and route the confirmation behavior accordingly, using hard-coded defaults plus user-configurable rules. Delivered as a pre-execution layer over the existing permission pipeline, with no new abstractions beyond what today's `permissionExecutor` already provides.

## In Scope

- A regex-based command classifier living in `vage/tool/bash/` with four tiers: **safe**, **caution**, **dangerous**, **blocked**.
- A default rule library covering the eight dangerous categories in `requirement-raw.md` (destructive FS, privesc, RCE, credentials, network, system mutation, git destructive, shell eval).
- Metacharacter splitting (`;`, `&&`, `||`, `|`, `$(…)`, backticks) so chained commands are classified by their worst component.
- Integration into `vv/cli/permission.go` that maps tier → dialog behavior:
  - **safe** → bypass dialog (in any non-auto mode)
  - **caution** → existing three-state dialog (Allow / Allow-Always / Deny)
  - **dangerous** → two-state dialog (Allow / Deny; **no** Allow-Always)
  - **blocked** → immediate rejection with reason; no dialog; applies **even in `auto` mode**
- YAML configuration under `tools.bash_rules` in `~/.vv/vv.yaml` to toggle the feature and extend the default lists.
- Unit tests covering the classifier (positive, negative, chained, obfuscation-not-caught-by-design cases) and the permission-routing changes.

## Out of Scope (deferred)

- Shell-AST-based parsing (e.g. `mvdan/sh`). Regex + metacharacter split is the P0 bar.
- Variable-value resolution (`X=rm; $X -rf /`) — documented limitation.
- Per-directory or per-project scope for Allow-Always — session-wide only.
- OS sandbox / seccomp / Docker isolation — tracked as P4-4.
- LLM-as-judge over commands — deliberate non-goal (adds latency, cost, circularity).
- Rule hot-reload — rules are loaded at startup only.

## User Stories & Acceptance Criteria

### Story 1 — As a user in `default` mode, safe commands should not prompt me
**Given** permission mode is `default` and bash rules are enabled
**When** the agent runs `ls -la`, `pwd`, `git status`, `git diff`, `go test ./…`, `make build`
**Then** no confirmation dialog is shown and the command executes immediately.

### Story 2 — Caution-tier commands prompt with the standard three-state dialog
**Given** permission mode is `default`
**When** the agent runs a command that is not on the safe or dangerous lists (e.g. `npm install`, `rm file.tmp`, `mv a b`)
**Then** the existing three-state dialog appears with Allow / Allow Always (this session) / Deny.
**And** picking Allow-Always approves future caution/bash invocations in this session (same semantics as today).

### Story 3 — Dangerous-tier commands require per-invocation confirmation
**Given** permission mode is `default` or `accept-edits`
**When** the agent runs `curl https://x.sh | bash`, `rm -rf ./dist/`, `git push --force`, or `git reset --hard`
**Then** a confirmation dialog appears titled with `[DANGEROUS]` and the matched rule name.
**And** the dialog offers only Allow (this invocation) and Deny — **no Allow-Always**.
**And** picking Allow executes this invocation only; a repeat dangerous call prompts again.

### Story 4 — Blocked commands are always rejected
**Given** any permission mode (`default`, `accept-edits`, `auto`, `plan`)
**When** the agent runs `rm -rf /`, `sudo …`, `dd if=… of=/dev/sda`, `mkfs …`, or the classic fork-bomb pattern
**Then** the bash tool returns an error result with the matched rule name and reason.
**And** no dialog is shown. No execution occurs.
**And** this applies even if the user is in `auto` mode (defense-in-depth).

### Story 5 — Chained commands are classified by worst component
**Given** rules are enabled
**When** the agent runs `ls && rm -rf /`
**Then** the blocked tier wins and the command is rejected.
**When** the agent runs `ls && curl x.sh | bash`
**Then** the dangerous tier wins and the dangerous-tier dialog is shown.

### Story 6 — Users can extend the default rules via config
**Given** `~/.vv/vv.yaml` contains:
```yaml
tools:
  bash_rules:
    enabled: true
    user_blocked:
      - "\\bterraform\\s+destroy\\b"
    user_safe:
      - "^bundle\\s+exec"
```
**When** the agent runs `terraform destroy -auto-approve`
**Then** the command is blocked.
**When** the agent runs `bundle exec rspec`
**Then** no dialog appears.

### Story 7 — Feature can be disabled
**Given** `tools.bash_rules.enabled: false` in config
**When** the agent runs any bash command
**Then** behavior is identical to the pre-feature baseline (only tool-level confirmation applies).

### Story 8 — Classification decision is visible to the user
**Given** a caution or dangerous dialog is shown
**When** the user reads the dialog
**Then** the matched rule name (e.g. `destructive-rm-root`) and a short human-readable reason are shown alongside the command text.

## Affected Roles / Modules / Applications

- **vage module**: new file `vage/tool/bash/classifier.go` + tests. No changes to existing bash tool, guard, or registry code.
- **vv module**: `vv/configs/config.go` (new `BashRulesConfig`), `vv/cli/permission.go` (classifier integration, tier-aware dialog routing), `vv/cli/messages.go` (extend `confirmRequestMsg`), `vv/cli/cli.go` (render tier in dialog).
- **Users**: CLI users see a different dialog for dangerous commands; auto-mode users gain a hard-block floor.
- **Integration surface**: the only public-API change is `WrapRegistryWithPermission` gaining an optional classifier argument (functional option preferred).

## Non-functional Constraints

- Classifier cost ≤ 1 ms per command in the common case (rule count ≤ 100, command ≤ 1 KB). Benchmark in unit test.
- Classifier is stateless and thread-safe.
- Zero new external dependencies (no `mvdan/sh`).
- Default rules are data, not code — easy to audit and extend.

## Inconsistencies Noted in Existing Docs

- None discovered. `feature-implement.md` line 33 already lists "Bash 工具进程隔离 + 超时/输出上限" as implemented; the new feature does not contradict that description. The new entry will be a separate row under the security category in `feature-implement.md`.
