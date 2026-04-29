# P2-10 WebSearch 工具 — 设计

> 范围：实现 `vage/tool/websearch/` 包并接入 vv 的 read-capable agent，提供 Tavily + Brave 两个 Provider。原则：复用 webfetch 包的工程模式（envelope / Option / Register），不引入新抽象层。

## 1. 总体结构

```
vage/tool/websearch/
├── websearch_tool.go     // Tool struct, ToolDef, Handler, Register, Option
├── envelope.go           // searchEnvelope, errorResult, jsonResult
├── provider.go           // Provider 接口
├── tavily.go             // tavilyProvider 实现
├── brave.go              // braveProvider 实现
└── websearch_tool_test.go

vv/configs/config.go      // + ToolsConfig.WebSearch + env overrides
vv/registries/tool_access.go // + 条件注册 web_search 到 CapRead
vv/tools/tools.go         // + 三个工厂条件注册 web_search
vv/agents/*.go            // + 4 个 prompt 加一行说明
vv/debugs/toolregistry.go // + web_search 入 builtin + read-only set
```

不新增 vv 顶级目录。所有跨包 API 保持最小：vv 只依赖 `websearch.New(...)` / `websearch.Register(...)` / `websearch.WithXxx(...)`。

## 2. 接口与数据结构

### 2.1 Provider 接口

```go
// provider.go
package websearch

import "context"

type Provider interface {
    Name() string
    Search(ctx context.Context, query string, maxResults int) ([]Result, error)
}

type Result struct {
    URL         string `json:"url"`
    Title       string `json:"title,omitempty"`
    Snippet     string `json:"snippet,omitempty"`
    PublishedAt string `json:"published_at,omitempty"`
}
```

- 单方法接口；不引入 multi-result strategy / ranker / merger 抽象。
- `Search` 返回 nil + 空切片表示「无错但 0 结果」；返回 error 由 Tool 转译为 envelope 错误码。
- 本地 sentinel 错误：`ErrInvalidAPIKey`、`ErrProviderHTTP`，便于 Tool 翻译。

### 2.2 Envelope

```go
// envelope.go
type searchEnvelope struct {
    Query       string   `json:"query"`
    Provider    string   `json:"provider"`
    Results     []Result `json:"results"`
    RetrievedAt string   `json:"retrieved_at"`
    Warnings    []string `json:"warnings,omitempty"`
    ErrorCode   string   `json:"error_code,omitempty"`  // empty on success
    Message     string   `json:"message,omitempty"`
    StatusCode  int      `json:"status_code,omitempty"` // upstream HTTP status when relevant
}
```

错误码集合（与 webfetch 风格一致）：
| code | 触发场景 |
|------|----------|
| `invalid_arguments` | JSON unmarshal 失败 / 字段越界 |
| `empty_query` | query 去空白后为空 |
| `query_too_long` | len > 1024 字符 |
| `provider_error` | upstream 4xx/5xx；附 `status_code` |
| `parse_failed` | upstream JSON 字段不可解析 |
| `timeout` | `context.DeadlineExceeded` 或 net 超时 |
| `request_failed` | 其它网络错误 |
| `invalid_api_key` | upstream 401/403 |

成功且 0 结果时 envelope `IsError=false`，warnings 含 `"no_results"`。

### 2.3 Tool 与 Option

```go
// websearch_tool.go
type Tool struct {
    provider        Provider
    httpClient      *http.Client
    defaultMax      int  // 5
    hardMaxResults  int  // 20
    timeout         time.Duration  // 10s
    userAgent       string
}

type Option func(*Tool)

func WithProvider(p Provider) Option              // required path
func WithHTTPClient(c *http.Client) Option
func WithTimeout(d time.Duration) Option
func WithDefaultMaxResults(n int) Option
func WithHardMaxResults(n int) Option
func WithUserAgent(s string) Option
```

注：`Provider` 通过显式 Option 注入 —— Tool 本身不感知具体后端。`New(opts...)` 在 `provider == nil` 时返回 nil（vv 接线层据此跳过注册）。

### 2.4 ToolDef

```go
ToolName = "web_search"
Description = "Search the public web by keyword and return a list of result URLs with titles and snippets. The result URLs can be passed to web_fetch for full content."
Parameters: {
    type: "object",
    properties: {
        query:       { type: "string", description: "Search keywords (required, max 1024 chars)" },
        max_results: { type: "integer", description: "Optional cap on result count (default 5, max 20)" },
        topic:       { type: "string", description: "Optional Tavily topic ('general' | 'news'); ignored by other providers" }
    },
    required: ["query"],
    additionalProperties: false,
}
ReadOnly = true   // 与 web_fetch 一致；不写工作区
Source   = ToolSourceLocal
```

### 2.5 Provider 实现要点

#### Tavily (`tavily.go`)

- Endpoint: `POST https://api.tavily.com/search`
- Body 字段：`api_key, query, max_results, search_depth="basic", include_raw_content=false, include_answer=false, topic`
- Response 字段：`results[].url`, `results[].title`, `results[].content` (→ snippet)
- 401/403 → `ErrInvalidAPIKey`；4xx/5xx → `ErrProviderHTTP{Status: ...}`
- 选项：`tavilyProvider{apiKey string, httpClient *http.Client, baseURL string}`；baseURL 仅 test 用

#### Brave Search (`brave.go`)

- Endpoint: `GET https://api.search.brave.com/res/v1/web/search?q=...&count=...`
- Header: `X-Subscription-Token: <api_key>`
- Response 字段：`web.results[].url`, `web.results[].title`, `web.results[].description` (→ snippet)
- 401/403 → `ErrInvalidAPIKey`；429 → `ErrProviderHTTP{Status: 429}`（rate-limited 由 LLM Middleware 兜底重试，本工具不重试）
- topic 字段被忽略

两个 provider 内部都不读环境变量，全部通过构造参数注入；这样 unit test 可以用 `httptest.Server` 替换 baseURL。

## 3. vv 接线

### 3.1 Config 扩展（`vv/configs/config.go`）

```go
type ToolsConfig struct {
    BashTimeout    int             `yaml:"bash_timeout"`
    BashWorkingDir string          `yaml:"bash_working_dir"`
    AllowedDirs    *[]string       `yaml:"allowed_dirs,omitempty"`
    BashRules      BashRulesConfig `yaml:"bash_rules,omitempty"`
    WebSearch      WebSearchConfig `yaml:"web_search,omitempty"`
}

type WebSearchConfig struct {
    Provider       string `yaml:"provider,omitempty"`         // "" | "tavily" | "brave"
    APIKey         string `yaml:"api_key,omitempty"`          // never log
    TimeoutSeconds int    `yaml:"timeout_seconds,omitempty"`  // 0 → 10
    MaxResults     int    `yaml:"max_results,omitempty"`      // 0 → 5; cap 20
}

func (w WebSearchConfig) IsEnabled() bool {
    return strings.TrimSpace(w.APIKey) != "" &&
        normalizedWebSearchProvider(w.Provider) != ""
}

func normalizedWebSearchProvider(p string) string {
    switch strings.ToLower(strings.TrimSpace(p)) {
    case "tavily": return "tavily"
    case "brave":  return "brave"
    default:       return ""
    }
}
```

Env overrides（在 `Load`）：
- `VV_WEB_SEARCH_PROVIDER` → `cfg.Tools.WebSearch.Provider`
- `VV_WEB_SEARCH_API_KEY`  → `cfg.Tools.WebSearch.APIKey`
- `VV_WEB_SEARCH_TIMEOUT_SECONDS` → `cfg.Tools.WebSearch.TimeoutSeconds`
- `VV_WEB_SEARCH_MAX_RESULTS` → `cfg.Tools.WebSearch.MaxResults`

无效 provider（非 `""` 也非 `tavily/brave`）：`slog.Warn`，视同未配置（fail-soft，与 trace/eval 默认错误处理一致）。

### 3.2 单一构造函数（`vv/tools/websearch_factory.go`，新文件）

为避免 `tools.go` / `tool_access.go` 内同样的 30 行重复，集中 web_search 构造逻辑：

```go
package tools

// MaybeRegisterWebSearch registers web_search on reg if cfg has a usable
// provider + API key. Otherwise it is a no-op (zero-cost path). Returns the
// resolved provider name for logging, or "" when skipped.
func MaybeRegisterWebSearch(reg *tool.Registry, cfg configs.ToolsConfig) (string, error)
```

实现：
1. 若 `!cfg.WebSearch.IsEnabled()` → return `"", nil`
2. 根据 `cfg.WebSearch.Provider` 构造 Tavily / Brave provider
3. `tool.New(WithProvider(...), WithTimeout(...), WithDefaultMaxResults(...), WithHardMaxResults(20))`
4. `websearch.Register(reg, opts...)` (Register 内部判 nil 跳过)
5. 返回 provider 名

`tool_access.go::registerCapabilityTools` 与 `tools.go` 三个工厂都调用这个函数；构造时机无差异。

### 3.3 Capability 接线

`vv/registries/tool_access.go`：
```go
case CapRead:
    ...read.Register(reg, ...)
    if err := webfetch.Register(reg); err != nil { return err }
    if _, err := tools.MaybeRegisterWebSearch(reg, cfg); err != nil { return err }
    return nil
```

> 循环依赖检查：`vv/registries` 已 import `vv/configs`；新增 import `vv/tools` 会形成 `tools → registries` 反向？
> 当前 `tools/tools.go` 不 import `registries`，只 import `vv/configs` + `vage/tool/*`。新文件 `tools/websearch_factory.go` 同样只依赖 vage + configs，**`registries` 单向 import `tools` 不会成环**。已确认。

`vv/tools/tools.go` 三个工厂在 `read.Register` 后追加：
```go
if err := webfetch.Register(reg); err != nil { ... }
if _, err := MaybeRegisterWebSearch(reg, cfg); err != nil { ... }
```

### 3.4 Agent Prompt 微调（4 文件）

每个文件追加一行 `**web_search**: Find URLs by keyword. Pair with web_fetch to read full content.` 紧跟在 `**web_fetch**` 行后：
- `vv/agents/coder.go`
- `vv/agents/researcher.go`
- `vv/agents/reviewer.go`
- `vv/agents/primary.go`

### 3.5 Debug 注册元数据（`vv/debugs/toolregistry.go`）

```go
builtinTools  = map[string]bool{..., "web_search": true}
readOnlyTools = map[string]bool{..., "web_search": true}
```

## 4. 错误流与安全

| 输入 / 状况 | 处理 |
|-------------|------|
| `query` 为空 / 仅空白 | `errorResult(empty_query)` |
| `query` > 1024 字符 | `errorResult(query_too_long)` |
| `max_results <= 0` | 用 `defaultMaxResults` |
| `max_results > hardMaxResults` | 夹取到 hardMax + warning `max_results_clamped` |
| Provider 401/403 | envelope `error_code: invalid_api_key, IsError=true` |
| Provider 4xx/5xx | envelope `error_code: provider_error, status_code: N, IsError=true` |
| Provider 返回 0 结果 | `IsError=false`, `warnings: ["no_results"]` |
| `context.DeadlineExceeded` / `errors.Is(err, context.DeadlineExceeded)` | `errorResult(timeout)` |
| 其它网络错误 | `errorResult(request_failed)` |

API Key 仅出现在 HTTP 请求 body / header，不进入 envelope、不被 logging hook 输出。`Provider.Search(...)` 返回 error 时，不把请求体或 key 暴露在 error string。

依赖现有 `ToolResultInjectionGuard`（已默认开启）扫描 envelope 文本，无需新增 guard。

## 5. 测试矩阵

### 5.1 `vage/tool/websearch/`（unit）

| 用例 | 断言 |
|------|------|
| Tavily：200 + 3 结果 | envelope `provider="tavily"`, `len(results)=3`, `IsError=false` |
| Brave：200 + 5 结果 | envelope `provider="brave"`, `IsError=false`, snippet 字段映射 |
| Tavily：401 | `error_code="invalid_api_key", IsError=true` |
| Brave：500 | `error_code="provider_error", status_code=500` |
| Provider 0 结果 | `warnings: ["no_results"]`, `IsError=false` |
| `query` 空白 | `error_code="empty_query"`, 不发 HTTP |
| `query` 1025 字符 | `error_code="query_too_long"`, 不发 HTTP |
| `max_results=50` | 夹取到 20，`warnings: ["max_results_clamped"]` |
| `max_results=0` 默认 5 | provider 收到 5 |
| Timeout（context cancel） | `error_code="timeout"` |
| `Tool.New(provider=nil)` | 返回 nil（用于 vv 跳过注册） |
| `Register(reg, ...)` provider 注入 | 注册可见且 ToolDef 正确 |

测试通过 `httptest.NewServer` 起本地桩；与 webfetch 测试同款 `roundTripFunc` 也支持，但 httptest 更接近真实 path。

### 5.2 `vv/configs/`

| 用例 | 断言 |
|------|------|
| YAML `web_search: {provider: tavily, api_key: x}` | `IsEnabled=true` |
| YAML 缺 api_key | `IsEnabled=false` |
| YAML provider=`unknown` | `slog.Warn` 触发；`IsEnabled=false` |
| env `VV_WEB_SEARCH_API_KEY` | 覆盖 YAML |
| env `VV_WEB_SEARCH_MAX_RESULTS=12` | 解析成 int |

### 5.3 `vv/registries/tool_access_test.go` 与 `vv/tools/tools_test.go`

现有断言强行列出工具名集合：
```go
assertToolNames(t, reg.List(), "bash", "edit", "glob", "grep", "read", "web_fetch", "write")
```

修改策略：**不在已有 case 里加 `web_search`**（保持「缺配置零成本」语义被测试到），新增并列 case：
- `TestProfile_BuildRegistry_WithWebSearchEnabled`：注入带 key 的 cfg → 列表多 `web_search`。
- `TestRegister_AllRegistered_WithWebSearchEnabled` 同理。

### 5.4 `vv/integrations/setup_tests/`

新增 case：用 `t.Setenv("VV_WEB_SEARCH_PROVIDER", "tavily"); t.Setenv("VV_WEB_SEARCH_API_KEY", "test-key")`，断言 coder/researcher/reviewer/primary 的工具列表包含 `web_search`。该用例不发出真实 HTTP 请求（只构造 ToolDef）。

### 5.5 `vv/debugs/`

更新 `debugs_test.go` 中 builtin/readOnly 集合断言（如有显式列表对比）。

## 6. 影响清单（落地用）

新增：
- `vage/tool/websearch/{provider.go, tavily.go, brave.go, envelope.go, websearch_tool.go, websearch_tool_test.go}`
- `vv/tools/websearch_factory.go`
- `vv/configs/config_websearch_test.go`（专项）
- `vv/integrations/setup_tests/setup_tests/setup_websearch_test.go`

修改：
- `vv/configs/config.go`：扩 `ToolsConfig`、加 4 个 env 解析、加 `WebSearchConfig.IsEnabled` / `normalizedWebSearchProvider`。
- `vv/registries/tool_access.go`：CapRead 分支末尾 hook MaybeRegisterWebSearch。
- `vv/tools/tools.go`：3 个工厂在 read 之后 hook MaybeRegisterWebSearch。
- `vv/agents/{coder,researcher,reviewer,primary}.go`：prompt 加一行。
- `vv/debugs/toolregistry.go`：补 `web_search` 进 builtin + read-only。
- `vv/registries/tool_access_test.go`：新增 enabled-case；保持原 case 不变。
- `vv/tools/tools_test.go`：同上。
- `vv/agents/*_test.go`：4 个 prompt 测试若校验"工具列表包含 web_fetch"则一并加 web_search 期望（按现有断言形态而定，缺省可不动）。
- `vv/integrations/tools_tests/tools_tests/tools_test.go` & `setup_tests/.../setup_agents_test.go`：与 unit 同步策略。

## 7. 集成测试计划

| 场景 | 步骤 | 通过条件 |
|------|------|----------|
| WebSearch 缺 key 不出现在工具列表 | `setup.New` cfg 无 web_search → 列出 coder 工具 | `web_search` 不在列表 |
| WebSearch 配 Tavily key 出现在工具列表 | `setup.New` cfg = tavily+key → 列出 4 个 agent | `web_search` 在 4 个 agent 列表 |
| Tool 调用：Tavily 桩 200 | provider 用 httptest.Server URL，Handler 触发，断言 envelope | provider="tavily", 3 results |
| Tool 调用：Brave 桩 401 | 同上 | error_code="invalid_api_key" |
| Tool 调用：query 越界 | 不触发 provider | error_code="query_too_long" |

## 8. 复杂度自评

- 单一新增工具，遵循 webfetch 已有套路；Provider 接口 1 个方法、2 个实现各 ~80 LOC。
- 不改 vage 框架核心；不改 Dispatcher / Primary tool 逻辑；不改持久层。
- 配置 YAML 兼容（缺省即零成本路径）。
- 测试覆盖与 webfetch 同密度；总改动估 ~600 LOC（含测试）。

## 9. 推迟项（V2+）
- Serper / Exa Provider
- 结果级别缓存（短 TTL）
- include_raw_content 一次性内嵌正文
- 多 Provider 并发查询 + 去重重排
- per-tool 频率限制（依赖 P2-9 Hooks 落地）
