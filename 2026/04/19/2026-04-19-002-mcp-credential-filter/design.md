# 技术设计：MCP 凭据过滤中间件（P0-4）

## 1. 设计原则

- **与 P0-3 并列、互不吃对方**：P0-3 `ToolResultInjectionGuard` 作用在 `taskagent` 层扫 jailbreak 指令；P0-4 作用在 MCP Client/Server 边界扫凭据泄漏。两个 guard 在同一条工具返回链上顺序串联（先 MCP 层去凭据，再 taskagent 层查注入），不共用接口、不共享配置。
- **专项化、低代码量**：MCP 是第三方 I/O 面，有比全局 Guard 更强的约束空间。扫描器是 `vage/security/credscrub/` 里的一个独立包，纯函数式（无事件、无 slog），MCP Client/Server 侧负责事件和日志。
- **四点注入**：MCP Client 的 `CallTool` 出站 args、入站 result；MCP Server 的 `RegisterTool`/`RegisterAgent` handler 入站 args、出站 result。全部共用一个 `Scanner`。
- **零配置零破坏**：没有 `WithCredentialScanner` 时，MCP Client/Server 行为等同现在。
- **默认 redact，不默认 block**：业界经验 (gitleaks/TruffleHog) 表明规则类检测误报率非零。默认 redact 既减伤害又让调用继续；`block` 留给运维显式选择。
- **不做熵值检测**：v1 只做正则 + 关键字上下文 + JSON 敏感字段名。

## 2. 顶层架构

```
 ┌──────────────────────────── MCP Client ────────────────────────────┐
 │                                                                    │
 │  args (JSON string)                                                │
 │      │                                                             │
 │      ▼                                                             │
 │  ┌──────────────────────────────────────────┐                      │
 │  │ Scanner.ScanJSON(argsBytes)              │  outbound            │
 │  │   action=log/redact/block                │                      │
 │  └───────────────┬──────────────────────────┘                      │
 │          pass    │  redact    block                                │
 │          │       │     │                                           │
 │          │   redacted  │                                           │
 │          │    JSON  ErrorResult("blocked by mcp credential filter")│
 │          ▼       ▼                                                 │
 │    s.CallTool(ctx, argsObj)                                        │
 │          │                                                         │
 │          ▼                                                         │
 │    result.Content (text parts)                                     │
 │          │                                                         │
 │          ▼                                                         │
 │  ┌──────────────────────────────────────────┐                      │
 │  │ Scanner.ScanText(partText)               │  inbound             │
 │  └───────────────┬──────────────────────────┘                      │
 │   pass/redact    │      block → ErrorResult                        │
 │          ▼                                                         │
 │    return schema.ToolResult                                        │
 └────────────────────────────────────────────────────────────────────┘
```

MCP Server 镜像对称：`ScanJSONMap(args)` on inbound handler args, `ScanText(partText)` on outbound handler result.

四处调用共享一个 `*Scanner` 实例（无可变状态、可并发）。

## 3. 模块改动清单

### 3.1 新增包 `vage/security/credscrub/`

```
vage/security/credscrub/
├── credscrub.go       # Scanner 结构 + 构造 + Scan 入口
├── patterns.go        # 默认规则集
├── credscrub_test.go  # 单元测试
```

#### 3.1.1 `credscrub.go`

```go
package credscrub

import (
    "encoding/json"
    "regexp"
    "strings"
)

// Action is the action taken when credentials are detected.
type Action string

const (
    ActionLog    Action = "log"    // record only, content unchanged
    ActionRedact Action = "redact" // replace with [REDACTED:<type>]
    ActionBlock  Action = "block"  // reject with error
)

// Rule is a single credential pattern.
type Rule struct {
    Name    string         // stable identifier, e.g. "aws_access_key"
    Pattern *regexp.Regexp // compiled at Scanner construction
    // Keywords: if non-empty, at least one keyword must appear within
    // KeywordWindow bytes before the match. Lowercase compared.
    Keywords      []string
    KeywordWindow int // 0 = 64 (default)
}

// FieldRule triggers when a JSON key (case-insensitive) matches Pattern.
// The whole value at that key is redacted as <Type>.
type FieldRule struct {
    Name    string
    Type    string // placeholder tag, e.g. "secret"
    Pattern *regexp.Regexp
}

// Config configures a Scanner.
type Config struct {
    Rules        []Rule      // if empty, DefaultRules() is used
    FieldRules   []FieldRule // if empty, DefaultFieldRules() is used
    Allowlist    []*regexp.Regexp // skip matches that equal an allowlist hit
    Action       Action      // default: ActionRedact
    MaxScanBytes int         // default: 256 * 1024
}

// Scanner scans strings / JSON for credentials.
type Scanner struct {
    rules        []Rule
    fieldRules   []FieldRule
    allow        []*regexp.Regexp
    action       Action
    maxScanBytes int
}

// NewScanner builds a Scanner. Safe for concurrent use.
func NewScanner(cfg Config) *Scanner { ... }

// Hit describes a single credential match.
type Hit struct {
    Rule  string // rule name
    Type  string // placeholder type, e.g. "aws_access_key"
    Start int    // byte offset in the input (ScanText) or -1 for field hits
    End   int
    // Masked is a de-identified preview such as "AKIA****" — never the
    // full credential. Used for logs/events.
    Masked string
}

// ScanResult is the outcome of a scan.
type ScanResult struct {
    Hits      []Hit
    Action    Action  // the effective action (caller's configured Action)
    Truncated bool    // input was larger than MaxScanBytes
}

// ScanText scans a plain string for matches.
func (s *Scanner) ScanText(text string) ScanResult { ... }

// RedactText returns text with each hit replaced by "[REDACTED:<type>]".
// If no hits, returns text unchanged.
func (s *Scanner) RedactText(text string, hits []Hit) string { ... }

// ScanJSON parses raw JSON bytes (args payload) and returns hits plus
// a redacted-JSON byte slice if any field/value matched. The structure
// is preserved; values are replaced in-place. If raw is not valid
// JSON, ScanJSON falls back to ScanText + RedactText on raw.
func (s *Scanner) ScanJSON(raw []byte) (ScanResult, []byte /*redacted*/, error) { ... }

// ScanJSONMap scans a pre-decoded JSON map (MCP Server handler side).
// It mutates the map in place when action=redact is requested via caller.
// Returns scan result; the caller decides log/redact/block.
func (s *Scanner) ScanJSONMap(m map[string]any) ScanResult { ... }

// Action returns the configured default action.
func (s *Scanner) Action() Action { return s.action }

// Enabled returns false if the scanner is nil (helper for call sites).
func (s *Scanner) Enabled() bool { return s != nil }
```

**Key behaviors:**
- `ScanText` walks `rules`; each rule uses `FindAllStringIndex`; keyword context checked via lowercased substring look-behind in a window (default 64 bytes). Allowlist checked against matched substring; if any allowlist regex matches the match text, drop it.
- `ScanJSON` / `ScanJSONMap` walks the decoded tree:
  - At each map entry, check each `FieldRule.Pattern` against the key (lowercase). If it matches, the value (if string) is marked as a hit with `Type = fieldRule.Type`.
  - For string values, also run `ScanText` on the string content.
  - For non-string values (numbers, bools, nulls) skip.
  - For nested maps/slices, recurse.
- `RedactText` returns a new string with each hit's `[Start:End]` replaced by `[REDACTED:<type>]`. Overlapping hits deduplicated (keep first).
- `Masked`: first 4 chars + `****`, ellipsizing credentials below 8 chars to `****`. Never logs the full value.
- Compiled regex lists stored once, no per-call allocation.

#### 3.1.2 `patterns.go`

Provides `DefaultRules()` and `DefaultFieldRules()` and `DefaultAllowlist()`.

| # | Rule name | Regex | Keywords (require one within 64B) | Type placeholder |
|---|-----------|-------|-----------------------------------|------------------|
| 1 | aws_access_key | `\b(AKIA\|ASIA\|AROA\|AGPA\|AIDA\|ANPA\|ANVA\|ABIA\|ACCA)[0-9A-Z]{16}\b` | — | aws_access_key |
| 2 | aws_secret_key | `\b[0-9a-zA-Z/+]{40}\b` | `aws_secret`, `secret_access_key`, `aws_secret_access_key` | aws_secret_key |
| 3 | github_token | `\b(ghp\|gho\|ghu\|ghs\|ghr)_[A-Za-z0-9]{36,255}\b` | — | github_token |
| 4 | slack_token | `\bxox[baprsAPE]-[0-9A-Za-z-]{10,48}\b` | — | slack_token |
| 5 | jwt | `\beyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\b` | — | jwt |
| 6 | pem_private_key | `-----BEGIN (?:RSA\|DSA\|EC\|OPENSSH\|PGP\|ENCRYPTED\|)\s*PRIVATE KEY-----` | — | pem_private_key |
| 7 | stripe_key | `\bsk_(?:live\|test)_[0-9a-zA-Z]{24,}\b` | — | stripe_key |
| 8 | google_api_key | `\bAIza[0-9A-Za-z_-]{35}\b` | — | google_api_key |
| 9 | openai_key | `\bsk-(?:proj-)?[A-Za-z0-9_-]{20,}\b` | — | openai_key |
| 10 | bearer_token | `(?i)bearer\s+[A-Za-z0-9._~+/=-]{16,}` | — | bearer_token |
| 11 | generic_high_entropy_api_key | `\b[A-Za-z0-9]{32,64}\b` | `api_key`, `apikey`, `access_token`, `secret` | generic_api_key |

Sensitive field-name rules (case-insensitive key match):

| FieldRule name | Key pattern | Type |
|----------------|-------------|------|
| field_password | `(?i)^(password\|passwd\|pwd)$` | password |
| field_secret | `(?i)^.*(secret\|credentials?)$` | secret |
| field_token | `(?i)^.*(token\|auth[_\-]?token)$` | token |
| field_api_key | `(?i)^(api[_\-]?key\|access[_\-]?key\|access[_\-]?token\|auth[_\-]?header)$` | api_key |
| field_authorization | `(?i)^(authorization\|x-api-key\|proxy-authorization)$` | authorization |

Default allowlist (drop match if it equals one of these):

| Allow | Regex | 用途 |
|-------|-------|------|
| uuid_v4 | `(?i)^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$` | 避免 UUID 被当作 40 char secret |
| git_sha | `^[0-9a-f]{40}$` | git 提交哈希 |

Rule #2 (aws_secret_key) and #11 (generic_high_entropy_api_key) are **keyword-gated** — they only fire when a surrounding keyword is present within 64 bytes. This is explicitly borrowed from TruffleHog's design and prevents false positives on generic base64 strings.

### 3.2 `vage/mcp/client/client.go`

**Changes**:

1. Add field to `Client`:
   ```go
   scanner *credscrub.Scanner
   onScan  func(ctx context.Context, ev ScanEvent) // optional callback
   ```
2. Add option type and constructor options (functional options for non-breaking evolution):

   ```go
   type Option func(*Client)

   func WithCredentialScanner(s *credscrub.Scanner) Option { ... }
   func WithScanCallback(cb func(context.Context, ScanEvent)) Option { ... }

   // NewClient signature stays the same; new variadic NewClientWith(uri, opts...):
   func NewClient(serverURI string, opts ...Option) *Client
   ```

   `NewClient(uri)` (no opts) remains backwards compatible.

3. In `CallTool` (client.go L124):
   - After `json.Unmarshal(args) → argsObj`, if `c.scanner != nil`:
     - Run `result, redacted, _ := c.scanner.ScanJSON([]byte(args))`.
     - If `len(result.Hits) > 0`:
       - Emit `ScanEvent{Direction: "mcp_outbound", ToolName: name, Hits: result.Hits, Action: action}` via `onScan` callback (if set).
       - Dispatch depending on `result.Action`:
         - `ActionLog`: keep `argsObj` as-is.
         - `ActionRedact`: re-decode `redacted` into `argsObj`.
         - `ActionBlock`: return `schema.ErrorResult("", "blocked by mcp credential filter (outbound): "+typesSummary(result.Hits))`, no call made.
   - After `s.CallTool(...) → result`, if `c.scanner != nil`:
     - For each text `ContentPart`, `sr := c.scanner.ScanText(part.Text)`; if hits:
       - Emit event.
       - `ActionLog`: leave as-is.
       - `ActionRedact`: `part.Text = scanner.RedactText(part.Text, sr.Hits)`.
       - `ActionBlock`: return `schema.ErrorResult("", "blocked by mcp credential filter (inbound): "+typesSummary(sr.Hits))`.

4. `ScanEvent` struct (exported from `vage/mcp/client` for the callback type):
   ```go
   type ScanEvent struct {
       Direction string // "mcp_outbound" | "mcp_inbound"
       ServerURI string
       ToolName  string
       Hits      []credscrub.Hit // Masked, never plaintext
       Action    credscrub.Action
   }
   ```

### 3.3 `vage/mcp/server/server.go`

Symmetric changes:

1. Add fields `scanner *credscrub.Scanner`, `onScan func(...)`.
2. New options `WithCredentialScanner`, `WithScanCallback`. `NewServer()` remains backwards compatible.
3. `RegisterTool` handler (server.go L142–L170):
   - After `json.Unmarshal(req.Params.Arguments, &args) → args map`, if `s.scanner != nil`:
     - `sr := s.scanner.ScanJSONMap(args)` (mutates string leaves when needed).
     - On hits + action=redact: mutation already done in ScanJSONMap returning redacted values replaced in place.
     - On hits + action=block: return `&mcp.CallToolResult{IsError:true, Content:[]mcp.Content{&mcp.TextContent{Text: "blocked by mcp credential filter (server-in)"}}}` and callback fired.
     - On log: just fire callback.
   - After `reg.Handler(ctx, args) → result`, iterate each `result.Content[i].Text`:
     - `sr := s.scanner.ScanText(text)` → same 3-way branch; redact mutates `result.Content[i].Text`.
4. `RegisterAgent` handler at server.go L93–L128: apply scanner at the same two points (inbound args unmarshal, outbound agent text).

Implementation lifts the scan/branch logic into one unexported helper per direction: `applyInboundScan(ctx, name, m) (map[string]any, *mcp.CallToolResult)` where second return non-nil means "block, return this error".

### 3.4 `vage/schema/event.go`

**No change**. MCP layer events are a new concept (mcp-specific), and the existing `EventGuardCheck` is shaped around tool-result guards. Reusing `EventGuardCheck` would overload semantics (the user could not distinguish P0-3 from P0-4 events). Instead we add:

```go
const EventMCPCredentialDetected = "mcp_credential_detected"

// MCPCredentialDetectedData carries a summary of a credential scan outcome.
// Plaintext credentials never appear in this data — only Masked previews.
type MCPCredentialDetectedData struct {
    Direction string   `json:"direction"`      // "mcp_outbound" | "mcp_inbound" | "mcp_server_inbound" | "mcp_server_outbound"
    ServerURI string   `json:"server_uri,omitempty"`
    ToolName  string   `json:"tool_name,omitempty"`
    Action    string   `json:"action"`         // "log" | "redact" | "block"
    HitTypes  []string `json:"hit_types"`      // deduped, sorted credential types
    HitCount  int      `json:"hit_count"`
    Masked    []string `json:"masked,omitempty"` // e.g. ["AKIA****", "sk-****"]
}

func (MCPCredentialDetectedData) eventData() {}
```

### 3.5 `vv` 接入

1. **`vv/configs/config.go`** — extend `SecurityConfig`:

   ```go
   type SecurityConfig struct {
       ToolResultInjection  ToolResultInjectionConfig  `yaml:"tool_result_injection,omitempty"`
       MCPCredentialFilter  MCPCredentialFilterConfig  `yaml:"mcp_credential_filter,omitempty"`
   }

   type MCPCredentialFilterConfig struct {
       Enabled       *bool    `yaml:"enabled,omitempty"`        // default true
       Action        string   `yaml:"action,omitempty"`         // "log" | "redact" | "block"; default "redact"
       MaxScanBytes  int      `yaml:"max_scan_bytes,omitempty"` // default 256 * 1024
       ExtraPatterns []string `yaml:"extra_patterns,omitempty"` // user regex additions
       Allowlist     []string `yaml:"allowlist,omitempty"`      // user allowlist regex additions
   }

   func (m MCPCredentialFilterConfig) IsEnabled() bool {
       return m.Enabled == nil || *m.Enabled
   }
   ```

   Env overrides in `Load()`:
   - `VV_MCP_CREDFILTER_ENABLED` (bool)
   - `VV_MCP_CREDFILTER_ACTION` (string)

2. **`vv/setup/setup.go`** — build a `*credscrub.Scanner` from `cfg.Security.MCPCredentialFilter` when enabled; expose via `setup.Init` return value or a new field on `registries.FactoryOptions` (follow P0-3 precedent).

3. **MCP client/server construction sites**: MCP wiring currently lives in tools/MCP integration modules. For vv, the scanner must flow into any place `mcp.client.NewClient` or `mcp.server.NewServer` is called. As of today vv does not wire MCP servers at startup (that's P1-1). But because we want the feature to be blocker-ready for P1-1, we expose the scanner as part of `setup.Init` output. **Scope note**: If no MCP wiring exists in vv today, vv wiring in this ticket is just (a) building the scanner and (b) making it retrievable from setup. Applying it to MCP clients is done when MCP clients are instantiated — currently in integration tests and user code. For completeness we still emit the config + constructor, satisfying AC-4.

## 4. 数据结构细节

### 4.1 `Hit.Masked` 生成

```go
func maskCredential(s string) string {
    if len(s) < 8 {
        return "****"
    }
    return s[:4] + "****"
}
```

Short, stable for de-duplication, reveals no more than the prefix (attacker already knows AWS keys start with `AKIA`).

### 4.2 Redact 占位符格式

`[REDACTED:<type>]` — fixed literal `[REDACTED:` prefix, credential type name, `]` suffix. The type name is drawn from the rule's `Type` field. The choice of a single fixed format (rather than preserving length) matches Sentry / OpenTelemetry practice and is simple.

**为什么不保留长度**：业界经验（Nuclei、OWASP）显示保留长度容易被侧信道推断；固定短占位符反而更安全。requirement AC-2.3 原文"保持原长度位置不变"修订为"保持语法位置有效（在 JSON 值上、在文本原地）但长度可变"。此偏离在 v1 可接受。

### 4.3 JSON 结构化 redact 语义

`ScanJSON` 走递归：
- map: 对每个 key 先看 FieldRule；若匹配且 value 是 string，整个 value 变 `[REDACTED:<fieldType>]`；若匹配但 value 是非 string（数组/对象），递归——避免把结构搞坏。
- map/value string: 同时跑 `ScanText` 以捕获"字段名看似正常但值里嵌入了凭据"。
- slice: 递归每个元素。
- 其余透传。

返回值是 `json.Marshal(modified)` 的新字节。

### 4.4 Scanner 线程安全

`Scanner` 只读字段 + 无可变状态。多 MCP Client 并发调用 `ScanText` / `ScanJSON` 不需锁。Config 在 `NewScanner` 时拷贝与编译。

## 5. 与现有能力的关系 / 层级

```
[MCP Server 外部调用方]
         │
         ▼
┌────────────────────┐
│ mcp.Server handler │  ← P0-4 scanner at inbound args + outbound result
└────────────────────┘

[外部 MCP Server]
         │
         ▼
┌────────────────────┐
│ mcp.Client.CallTool│  ← P0-4 scanner at outbound args + inbound result
└────────────────────┘
         │
         ▼
┌────────────────────┐
│ tool.Registry      │
└────────────────────┘
         │
         ▼
┌────────────────────┐
│ taskagent.runToolResultGuards │ ← P0-3 ToolResultInjectionGuard
└────────────────────┘
         │
         ▼
     messages → LLM
```

两层防御不冲突：P0-4 把**凭据**拿掉，P0-3 把**指令注入**拿掉。共存无副作用。

## 6. 测试计划（v1）

### 6.1 `vage/security/credscrub/credscrub_test.go`

- 每种默认规则正例 + 反例：
  - AWS: `AKIAIOSFODNN7EXAMPLE` 命中 → aws_access_key；`AKIA12345` 不命中（长度）。
  - GitHub: `ghp_1234567890abcdef1234567890abcdef1234` 命中。
  - Slack: `xoxb-12345-67890-abcdef` 命中。
  - JWT: 三段 base64 结构命中；两段结构不命中。
  - PEM: `-----BEGIN RSA PRIVATE KEY-----` + `-----BEGIN PRIVATE KEY-----` (PKCS#8) 都命中。
  - Stripe: `sk_live_abc...`, `sk_test_xyz...` 命中。
  - Google: `AIzaSyA1234567890123456789012345678901234` 命中。
  - OpenAI: `sk-proj-abc...`, `sk-abc...` 命中。
  - Bearer: `Authorization: Bearer eyJ...` 命中。
- 关键字上下文：
  - `aws_secret_access_key=abc...40-char-base64...` 命中 aws_secret_key。
  - 单独一个 40-char base64 文本（无关键字）不命中。
- 默认 allowlist：
  - UUID `550e8400-e29b-41d4-a716-446655440000` 不命中。
  - 40-hex git sha 不命中。
- FieldRule JSON：
  - `{"password":"xyz"}` → redacted `{"password":"[REDACTED:password]"}`。
  - `{"authorization":"Bearer eyJ..."}` → redacted authorization（整值）。
  - `{"username":"alice"}` → 原样。
- `RedactText` 多命中合并、不越界、不改周围文本。
- `MaxScanBytes` 截断：超长文本只扫前 N 字节，`Truncated=true`。
- `Masked` 输出不含原文超过 4 字符前缀。
- allowlist 优先级：命中 UUID 正则但 UUID 形态 → 被 allowlist drop。

### 6.2 `vage/mcp/client/client_test.go`（扩展）

- `NewClient(uri)` 无 scanner → `CallTool` 原路径行为不变。
- `NewClient(uri, WithCredentialScanner(s))`：
  - outbound `action=redact`: args 里 `{"aws_secret":"AKIA..."}` → 服务端收到 `[REDACTED:...]`（fake transport 验证）。
  - outbound `action=block`: 返回 `IsError=true` 的 ToolResult，未触发底层 `s.CallTool`。
  - outbound `action=log`: 原样转发，scan callback 收到事件。
  - inbound `action=redact`: 服务端返回含 `AKIAIOSFODNN7EXAMPLE` 文本 → Client 返回值已 redact。
  - inbound `action=block`: Client 返回 `IsError=true`。
  - callback 收到 `ScanEvent` 且 `Hits[i].Masked` 不含完整凭据。

### 6.3 `vage/mcp/server/server_test.go`（扩展）

- 无 scanner → 现有行为不变。
- RegisterTool + scanner：
  - handler 入站 args 含凭据 + action=redact → handler 真正拿到的 `args map` 字段值已 redact。
  - handler 入站 + action=block → handler 不被调用；返回 `IsError`。
  - handler 出站 result 含凭据 + action=redact → 外部看到的是 `[REDACTED:...]`。
- RegisterAgent + scanner：Agent 返回文本含凭据 + action=redact → 外部看到已 redact。

### 6.4 `vage/integrations/mcp_tests/`（集成）

- 端到端：一个本地 MCP Server 注册一个回显工具；Client 带 scanner 连上；`CallTool("echo", {"authorization":"Bearer AKIA..."})` → Server handler 收到的 args / Client 看到的 result 都是 redact 后的。

### 6.5 Benchmark

- `ScanText` 50KB 文本 20 规则 → 测 `ns/op` 与 `allocs/op`。
- `ScanJSON` 同规模 → 测递归开销。

## 7. 可观测性

- MCP Client / Server 侧 `onScan` 回调接收 `ScanEvent`；vv 将回调绑定为同时做：
  - `slog.Warn("vage: mcp credential detected", "direction", ..., "tool", ..., "hit_types", ..., "action", ..., "masked", ...)`（不含明文）。
  - 通过 `hook.Manager.Dispatch` 投递 `EventMCPCredentialDetected`（复用现有 hook 基础设施；dispatcher 在 vv 胶水层拼装）。

**关键约束**：`ScanEvent.Hits[*].Masked` 保证是"前 4 字符 + `****`"之类的短前缀，永远不含完整凭据。测试显式断言。

## 8. 性能

- 默认 11 条 regex + 5 条 field rule + 2 条 allowlist。
- `regexp.MustCompile` at Scanner construction, O(1) at scan time。
- 典型 50KB 文本扫描预估 p50 < 0.5ms，p99 < 2ms（与 P0-3 同量级）。
- `MaxScanBytes` 裁剪为 `text[:max]`，零拷贝 substring。
- 无命中时 `ScanResult.Hits == nil`，无分配。

## 9. 向后兼容

- `vage/mcp/client.NewClient(uri)` 签名改为 `NewClient(uri, opts ...Option)` — variadic 新增，无历史调用受损（Go 允许零个可变参数）。
- `vage/mcp/server.NewServer()` → `NewServer(opts ...Option)` 同上。
- 未启用 scanner 路径完全与当前一致（nil check 早退）。
- `vv/configs.SecurityConfig` 新增字段，YAML 里缺失时零值→ `IsEnabled` 默认 true，但如果未传入 scanner 也不会有变化；保持"配置存在但没有 MCP 使用点"的安全默认。

## 10. 已知局限（v1）

1. **规则不能覆盖全部凭据格式**。专有凭据（内部 API token、厂商特定 SDK key）需用户自行加 `extra_patterns`。
2. **未做熵值兜底**。Generic high-entropy 关键字规则（#11）覆盖了一部分，但 "无关键字裸密钥串" 会漏。交代给文档。
3. **不扫 binary ContentPart**。Image/File 不扫，原样透传。
4. **JSON 数值里藏凭据不检测**。e.g. `{"token": 123}`。FieldRule 仅对 string value 生效；数字型凭据罕见，v1 不处理。
5. **HTTP header 形式的凭据**: 在 JSON 值里出现 `"Authorization: Bearer ..."` 会被 Bearer 正则命中；在 HTTP 传输层 header 里的凭据不在本 middleware 扫描范围（那是 MCP transport 层的事）。
6. **占位符不可逆**：redact 后原始凭据不可恢复。符合需求。
7. **不回验证**：不去 AWS STS / GitHub `/user` 验证凭据是否真实（避免网络副作用）。

## 11. 验收对齐（AC 映射）

| AC | 本设计满足点 |
|----|------------|
| AC-1.1 | `client.CallTool` 在 `json.Unmarshal` 后、`s.CallTool` 前调用 `scanner.ScanJSON` |
| AC-1.2 | `RedactText` / `ScanJSON` 的 redact 分支替换为 `[REDACTED:<type>]`，保 JSON 结构 |
| AC-1.3 | `action=block` 在 Client 侧返回 `schema.ErrorResult("", "blocked by mcp credential filter (...): <types>")`，不调用远端 |
| AC-1.4 | `action=log` → `onScan` 回调触发，原请求透传 |
| AC-1.5 | `DefaultRules()` 含 AWS/GitHub/Slack/JWT/PEM/Stripe/Google/Bearer/OpenAI + generic keyword-gated + PEM PKCS#8 |
| AC-1.6 | 零命中 → `ScanResult.Hits` 为 nil，请求原样透传 |
| AC-2.1 | `client.CallTool` 在 `s.CallTool` 返回后对每个 text `ContentPart` 跑 `ScanText` |
| AC-2.2 | 三档 action 对称实现 |
| AC-2.3 | 占位符格式固定 `[REDACTED:<type>]`，原地替换；不修改周围文本（长度变化是已知偏离，设计注释中说明） |
| AC-2.4 | `MaxScanBytes` 裁剪，`Truncated=true` 标记，hit 仅来自裁剪范围 |
| AC-3.1 | `server.RegisterTool` handler 在 unmarshal 后 → 调用 `ScanJSONMap` |
| AC-3.2 | `server.RegisterTool` handler 返回后 → 对每个 text part 跑 `ScanText` |
| AC-3.3 | Client/Server 共用同一 `Scanner` 实例（setup 阶段构造一次，四个挂载点引用） |
| AC-4.1 | `MCPCredentialFilterConfig` 含 enabled/action/max_scan_bytes/extra_patterns/allowlist |
| AC-4.2 | `VV_MCP_CREDFILTER_ENABLED` 与 `VV_MCP_CREDFILTER_ACTION` env 覆盖 |
| AC-4.3 | 默认 `enabled=true`, `action=redact`, `max_scan_bytes=262144` |
| AC-4.4 | `WithCredentialScanner(nil)` 路径或未调用 option → 运行时 nil check 早退，零开销 |
| AC-5.1 | `onScan` callback + `EventMCPCredentialDetected` hook + `slog.Warn` 三路观测 |
| AC-5.2 | `slog.Warn` 含结构化字段 |
| AC-5.3 | `Hit.Masked` = `前4+****`；事件/日志只引用 Masked |
| AC-6.1 | §6.1 单元测试每种凭据正反 |
| AC-6.2 | §6.1 allowlist 测试（UUID / git SHA） |
| AC-6.3 | §6.2/§6.3 端到端测试四个方向 × 三档动作 |
| AC-6.4 | §6.2 首个 case：`nil` scanner 零开销路径 |

## 12. 范围外 / v2

- 熵值检测作为第二信号
- LLM-as-Judge 二次审查
- 凭据"真实性"回验证
- 非文本 ContentPart 扫描（image OCR, binary file header scan）
- 可逆化的加密存储（让 Agent 后续操作能恢复）
- MCP transport 层（HTTP header / WebSocket frame）的凭据扫描
- P1-1（vv 集成 MCP Server 模式）实际挂载点的配线（本 ticket 仅预留构造器 + 配置）
