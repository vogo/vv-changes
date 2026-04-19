# Design Review: Tool-Result Injection Scan (P0-3)

Reviewer: senior staff engineer, zero prior context. Grounded in `vage/guard/*.go`, `vage/agent/taskagent/task.go`, `vage/schema/*.go`, `vage/hook/manager.go`, `vv/registries/registry.go`.

## Summary (Accepted / Rejected / Deferred)

| # | Item | Verdict | One-liner |
|---|------|---------|-----------|
| A | Reuse existing `guard.Guard` + `RunGuards` chain | Accepted | Correct instinct — do not invent a parallel type. |
| B | Scan only at `executeToolCall` return site (once, both branches) | Accepted | Right placement; no memory replay re-scanning. |
| C | 256 KB hard truncation before scan | Accepted | Protects against regex backtracking on MB-scale tool output. |
| D | Default rule pack with severity tiers | Accepted | Sensible; Unicode-tag + ChatML as High is standard industry practice. |
| E | `Severity` field added to existing `PatternRule` (inline) | **Rejected — simpler alt** | Breaks cohesion of existing `PIIGuard` / `PromptInjectionGuard` that don't care. Use a separate `SeveredPatternRule` or keep severity in a sibling slice inside the new guard. See §1. |
| F | New `EventGuardCheck` schema event type | **Accepted with scope cut** | The event is genuinely new (not in code despite CLAUDE.md claim). Keep it — but trim fields and do **not** pass `*hook.Manager` into the guard. See §2. |
| G | `Hook *hook.Manager` field on `ToolResultInjectionConfig` | **Rejected** | Guards do not and should not talk to hooks. `taskagent` already dispatches `EventToolCallStart/End/Result` — it should dispatch `EventGuardCheck` too, keeping `guard/` hook-free. See §3. |
| H | `rewrite` default = quarantine wrap (`<vage:untrusted …>`) | **Accepted, with caveats** | Right call. Document known weaknesses: wrapper itself pattern-matchable, ~40–80 byte bloat. Use stable lowercase tag, no emoji. See §4. |
| I | vv default action = `log` | **Accepted** | Correct shadow-deploy posture for P0 v1. "Defeats the point" argument is wrong: High-severity rules still block via `BlockOnSeverity=High`. Document the two-tier behaviour. See §5. |
| J | `Message.ToolCallID` / `ToolName` fields added to `guard.Message` | **Rejected — use Metadata** | Bloats a struct used by many guards that don't care. `Metadata map[string]any` already exists and is the natural home. See §6. |
| K | `DirectionToolResult` as third `Direction` constant | **Accepted** | It is a direction, and `RunGuards` chain participation is desirable (future `PIIGuard` on tool results). Alternative (separate type) fragments the abstraction. See §7. |
| L | `QuarantineTmpl` config knob | **Rejected** | v1 YAGNI. Hardcode the template as an unexported `const`; expose later if asked. See §8. |
| M | `BlockOnSeverity` with zero-value = disabled | Accepted with tweak | Fine — but default it to `SeverityHigh` at *config-construction time*, not implicit zero-sentinel, so the behaviour is clear from reading the default config. |
| N | 20-rule regex pack incl. Unicode tag / Bidi override | Accepted | Good coverage; keep. |
| O | `result.IsError == true` short-circuits scan | Accepted | Correct — don't re-accuse a tool error message. |
| P | Per-tool severity thresholds (`per_tool: {bash: …}`) | Deferred | Requirement says "global default" — v2. |
| Q | LLM-Judge / Prompt-Guard-86M | Deferred | Explicitly out of scope. |
| R | Memory replay re-scan | Deferred (and correctly not done) | Ingest-time scan only. |

Net: 3 rejections reduce surface area (inline severity field, hook injection into guard, ToolCallID/ToolName fields). 1 defer reduces config surface (`QuarantineTmpl`). All requirement ACs remain satisfied.

---

## 1. `PatternRule.Severity` — reject inline field

**What design says**: add `Severity` field to existing `PatternRule`, zero value = `SeverityMedium`.

**Why this is wrong**: `PatternRule` is a shared primitive in `vage/guard/guard.go:114-119`, used by `PromptInjectionGuard` and — per the comment — `PIIGuard`. Those guards don't need severity; they do hit-or-miss decisions. Adding a field they ignore:
- Pollutes the shared struct with a concept only one guard cares about.
- The "zero = Medium" convention is a footgun: a future user building a rule list without reading the docs gets implicit Medium, and severity only matters once it interacts with `BlockOnSeverity`. Silent promotion is confusing.
- `DefaultInjectionPatterns()` callers would semantically shift: their rules would be treated as "Medium" in any future severity-aware code, without ever being audited.

**Simpler alternative**: keep `PatternRule` unchanged. In `tool_result.go`, introduce a sibling type that composes it:

```go
// SeveredPatternRule is a PatternRule with a severity tier, used only by
// tool-result scanning.
type SeveredPatternRule struct {
    PatternRule
    Severity Severity
}
```

`ToolResultInjectionConfig.Patterns` becomes `[]SeveredPatternRule`. `DefaultToolResultInjectionPatterns()` returns `[]SeveredPatternRule`. Existing `PromptInjectionGuard` and its default pack are untouched. Zero backwards-compat risk. No migration.

If you want to let callers mix-and-match, provide a one-liner helper:

```go
func Sev(p PatternRule, s Severity) SeveredPatternRule { return SeveredPatternRule{p, s} }
```

## 2. New `EventGuardCheck` — keep, but trim

Checked `vage/hook/` and the event constants in `vage/schema/event.go`. There is **no existing** `EventGuardCheck`, despite what the parent `CLAUDE.md` implies. So the question "reuse or new" is moot — it's new either way. Keep it.

Trim `GuardCheckData`:
- Drop `Direction` field. This event is only emitted for tool-result scanning today; if that changes, add it then. Requirement lists `agent_id, session_id, tool_call_id, tool_name, rules, action, snippet` — `Direction` is not in the ask.
- Drop `Severity` string serialization — include it only if the guard has a tiered severity; the max-severity label is valuable for dashboards, so keep it, but use the `Severity.String()` method consistently.
- Keep `Snippet` capped at 200 bytes with explicit truncation marker (`…`), as requirement AC-3.1 says.

## 3. Where the hook lives — `taskagent` dispatches, not the guard

The design has `ToolResultInjectionConfig.Hook *hook.Manager` so the guard can fire events itself. Rejected.

**Why this is wrong**: the whole `guard` package today is hook-free. `runInputGuards` / `runOutputGuards` don't dispatch events, they just produce `*Result` and let the caller react. The `taskagent` already owns a `*hook.Manager` (field `hookManager` on `Agent`) and already dispatches `EventToolCallStart/End/Result` inline at the tool-execution site. Adding a hook-manager reference into the guard config inverts the existing contract and creates a second way to wire hooks (confusion).

**Simpler alternative**: guard stays pure. `taskagent.runToolResultGuards` inspects the `*guard.Result` after `RunGuards` returns, and dispatches `EventGuardCheck` itself with the data it already knows (`tc.ID`, `tc.Function.Name`, `agentID`, `rc.sessionID`, the `result.Violations`, and the action it chose). This is literally how the existing event dispatches at lines 711 and 720 already work.

Side benefit: the `slog.Warn` and the `EventGuardCheck` dispatch sit next to each other in `taskagent`, not split between `guard/` and `taskagent/`.

## 4. Quarantine wrap — accepted, but note the failure modes

The design's choice of `<vage:untrusted source="tool:<name>">…</vage:untrusted>` with a leading system reminder is the right default (aligns with Microsoft spotlighting research, Anthropic guidance, industry norm).

Known downsides that should be explicit in §11 "limitations":
1. **The wrapper itself is pattern-matchable.** An attacker who knows the exact tag string can include `</vage:untrusted>` in their payload to close the quarantine early. Mitigation: use a random per-scan nonce in the tag (`<vage:untrusted:a1b2c3>…</vage:untrusted:a1b2c3>`), or at minimum strip any occurrence of the closing tag from the original text before wrapping. v1 recommendation: strip the literal closing tag from the inner text.
2. **Prompt bloat.** ~80 bytes of wrapper + ~400 bytes of system reminder per rewrite. Acceptable on a 256KB budget but contributes to token usage. Note it; don't optimize.
3. **Emoji in template.** The current design template starts with `⚠️ `. Drop the emoji — non-ASCII in system-generated text occasionally trips up downstream tokenization/logging and provides zero safety value. Use plain `WARNING:`.

## 5. vv default = `log` — correct, with one clarification

Shadowed log-only is the industry-standard first ship and the right answer. The counterargument ("defeats the point of P0 security") is not right once you notice: `BlockOnSeverity=SeverityHigh` is enabled by default, so High-severity rules (ChatML markers, Unicode tag chars, Bidi override, Markdown image exfil, exfil command combos) **still block** even when `Action=log`. This gives:

- Log-only for fuzzy Low/Medium-severity string matches (the genuinely noisy ones prone to false positives on security docs, CVE writeups, tutorials).
- Block for high-precision structural/typographic attacks.

Design §5 kind of says this but it's easy to miss. The new design.md should state plainly: **"v1 default = log *for Low/Medium*, block *for High* via `BlockOnSeverity=SeverityHigh`. This is a two-tier default, not a single action."**

Also: requirement AC-1.4 says "未配置扫描器时行为与现状完全一致". Clarify that "未配置" means "disable via config or not setting `enabled=true`", since in vv the default is `enabled=true`. There's no regression for vage framework users who never wire `WithToolResultGuards`, since the slice is empty and the guard method no-ops.

## 6. `Message.ToolCallID` / `ToolName` — use `Metadata`

Design adds two fields to `guard.Message` specifically for this one guard. Rejected.

`guard.Message` already has `Metadata map[string]any` exactly for this case. Pass these via metadata keys `"tool_call_id"` and `"tool_name"`. The guard (and any future guard that doesn't care) stays decoupled from a struct field it doesn't read. The `taskagent.runToolResultGuards` already has `tc` in scope when it builds the event — it doesn't need the guard to echo them back; it reads them from its own closure.

If you worry about magic-string keys, export two named constants in `guard/`:

```go
const (
    MetaToolCallID = "tool_call_id"
    MetaToolName   = "tool_name"
)
```

This is how `runInputGuards` already passes `req.Metadata` through (line 464), so it matches the existing pattern.

## 7. `DirectionToolResult` as third constant — accept

Two options considered: (a) add a third `Direction` value, (b) create a separate guard type that does not participate in `RunGuards`.

(a) is right. The chain-level semantics (`Pass` / `Block` / `Rewrite`, sequential composition, `BlockedError`) are exactly the same shape as input guards. A future user who wants to stack a PII guard (or a regex denylist, or a secrets-redactor) on top of tool results should compose them in the same chain. Making tool-result a separate type forces users to understand two parallel APIs for no benefit.

One minor lint: the existing `PromptInjectionGuard.Check` uses `if msg.Direction == DirectionOutput { Pass }`. When `DirectionToolResult=2` is added, this guard will happily scan tool-result messages if someone mistakenly wires it there, because `DirectionToolResult != DirectionOutput`. Fix by inverting the check to `if msg.Direction != DirectionInput { Pass }` — strictly an improvement, backwards compatible with today's two-value world.

## 8. `QuarantineTmpl` — drop from config

YAGNI. v1 hardcodes the template as a package-level `const` in `tool_result.go`. If a user asks for customization in v2, promote it then. Exposing it now means committing to the template shape as API surface and writing docs for it.

Tests that need a specific marker for assertion can reference the `const` directly.

## 9. Other small items worth flagging

- **Rule #15 (`credential_leak`) is Medium but actually high-FP**: any config dump, `.env.example`, test fixture, or CI pipeline log triggers it. Recommend dropping to Low, *or* removing from v1 and letting it live behind a flag. Industry experience: this rule alone is >50% of false-positive volume in log-only deployments.
- **Rule #7 (`broad_ignore`)** and rule #1 (`ignore_instructions`) overlap; the existing Low severity one from `DefaultInjectionPatterns` will always fire alongside the new Medium one. De-dup or let both fire (then the max-severity logic correctly picks Medium).
- **`IsError=true` skip**: clarify this is a belt-and-braces optimization — if an attacker controls an upstream tool and returns `IsError=false` with malicious content, scan still runs. Only genuinely internal errors get skipped.
- **Stream-branch event ordering**: Design says `ToolCallEnd → GuardCheck → ToolResult`. The design's `runToolResultGuards` runs **between** `ToolCallEnd` and `ToolResult` — so the stream client sees the scanned/rewritten result in `ToolResult`. Good. Make the implementation's event ordering explicit in the new design.
- **Content-type gate for non-text**: design says "only scan text ContentPart". Verify this also means JSON content parts (`p.Type == "json"`) are *not* scanned — a `json` part is often machine-structured data where string matches are more likely to be false positives. Scan only `p.Type == "text"`. (Design currently says this; just making it explicit.)

## 10. What the main agent should know before implementation

1. The guard package stays hook-free. Event dispatch lives in `taskagent.runToolResultGuards`.
2. `PatternRule` is **not** modified. A new `SeveredPatternRule` type is added in `tool_result.go`.
3. `guard.Message` is **not** modified. Tool-call ID and tool name flow through `Metadata`.
4. `ToolResultInjectionConfig` has no `Hook` field and no `QuarantineTmpl` field.
5. vv default: `enabled=true, action=log, block_on_severity=high`. The last part is load-bearing — without it, log-only is indeed too passive for P0.
6. `PromptInjectionGuard.Check` gets a one-line hardening tweak: `if msg.Direction != DirectionInput { Pass }` instead of `== DirectionOutput`.

All requirement acceptance criteria (AC-1.* through AC-5.*) remain satisfied by the revised design.
