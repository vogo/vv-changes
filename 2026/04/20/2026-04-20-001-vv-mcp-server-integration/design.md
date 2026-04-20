# Design: vv 集成 MCP Server 模式 (P1-1)

> Scope: 把 `vage/mcp/server` 作为 vv 的第三种运行模式（与 cli / http 并列）接通出来，暴露 dispatchable agents 为 MCP tools，复用既有安全基线。

## 1. 设计原则

- **最小新增代码**：server 协议由 `vage/mcp/server.Server` + 官方 `go-sdk` 全部承担；vv 侧只做"配置→组装→生命周期管理"的胶水层。
- **对称 cli/http/mcp**：三模式互斥（一个 `vv` 进程只跑一种），复用同一个 `setup.Init` 构建出来的 LLM、memory、agents、guards。
- **安全默认**：stdio 默认无 auth（进程隔离），HTTP 默认只允许 loopback；非 loopback bind 必须有 `auth_token`，否则启动失败。
- **沿用既有治理**：credscrub / tool-result injection scan / bash path guardian / allowed_dirs / dangerous bash 分类 **与 HTTP 模式等价**。
- **零 CLI 交互假设**：`ask_user` 用 `NonInteractiveInteractor`；tool registry 不应用 CLI permission 包装。

## 2. 顶层结构

```
┌─── main.go ───┐
│   --mode mcp  │────► setup.Init (LLM, memory, agents — same as http)
└───────┬───────┘            │
        │                    ▼
        │           ┌──────────────────────┐
        │           │ mcps.Serve(ctx, cfg, │
        │           │   initResult)        │
        │           └─────────┬────────────┘
        │                     │
        │   transport=stdio   │   transport=http
        │           ▼                   ▼
        │  mcp.StdioTransport    mcp.StreamableHTTPHandler
        │      │                        │
        │      └────────┐      ┌────────┘
        │               ▼      ▼
        │         ┌──────────────────┐
        │         │ vage/mcp/server  │ ← RegisterAgent(coder/researcher/…)
        │         │   .Server        │   + optional RegisterAgent(dispatcher)
        │         │  + credscrub     │   + WithCredentialScanner (reuse)
        │         └──────────────────┘
```

新增包：`vv/mcps/`（与 `vv/httpapis/` 对称）。文件：
- `mcps/serve.go` — 入口 `Serve(ctx, cfg, initResult) error`
- `mcps/transport.go` — stdio / streamable HTTP 构造 + loopback/auth 预校验
- `mcps/register.go` — 按配置把 agents 注册到 MCP server
- `mcps/auth.go` — Bearer token middleware（仅 HTTP transport 使用）
- `mcps/serve_test.go`、`mcps/transport_test.go` — 单元测试

## 3. 启动入口与配置

### 3.1 Flag
`main.go` 保持当前 `--mode` 的结构，`"mcp"` 成为第三个合法值：

```go
// main.go 修改点（伪代码）
switch cfg.Mode {
case "http":   ... // existing
case "mcp":
    if err := mcps.Serve(ctx, cfg, initResult); err != nil { ... }
default: /* cli */
}
```

`--mode mcp` 下强制：
- `UserInteractor = askuser.NonInteractiveInteractor{}`（不要 `cli.NewCLIInteractor()`）
- 不构造 `permissionState`，不走 `cli.WrapRegistryWithPermission`
- `promptSet` 情况下直接报错"-p incompatible with mcp mode"（与 HTTP 行为一致）

### 3.2 YAML（新增 `mcp.server` 段）

```yaml
mcp:
  server:
    transport: stdio            # stdio | http
    addr: "127.0.0.1:7801"      # 仅 transport=http 生效，默认 loopback
    auth_token: ""              # 仅 transport=http 生效；监听非 loopback 时必填
    agents: []                  # 空 = 暴露所有 dispatchable agents；非空 = 白名单过滤
    expose_dispatcher: false    # true 时额外暴露 dispatcher 为顶层工具
    session_timeout: 0          # 秒；0 = 不超时 (stdio 时无效)
  credential_filter: { … }      # 保持现有 mcp_credential_filter 的语义（见 §3.3）
```

> 兼容性：原 YAML 键 `security.mcp_credential_filter` 保持不变，不做 breaking change。新 server 段独立在顶层 `mcp.server`，旧 `vv.yaml` 不感知新字段即可继续工作（默认 transport=stdio, agents=all, expose_dispatcher=false）。

### 3.3 代码级配置结构

```go
// vv/configs/config.go 新增
type MCPConfig struct {
    Server MCPServerConfig `yaml:"server,omitempty"`
}

type MCPServerConfig struct {
    Transport        string   `yaml:"transport,omitempty"`        // "stdio" (default) | "http"
    Addr             string   `yaml:"addr,omitempty"`             // default "127.0.0.1:7801"
    AuthToken        string   `yaml:"auth_token,omitempty"`       // required for non-loopback
    Agents           []string `yaml:"agents,omitempty"`           // empty = all dispatchable
    ExposeDispatcher bool     `yaml:"expose_dispatcher,omitempty"`
    SessionTimeout   int      `yaml:"session_timeout,omitempty"`  // seconds
}
```

追加到 `Config` struct：`MCP MCPConfig \`yaml:"mcp,omitempty"\``。**不动**现有 `Security.MCPCredentialFilter` 字段。

### 3.4 Env 覆盖
```
VV_MCP_TRANSPORT     → cfg.MCP.Server.Transport
VV_MCP_ADDR          → cfg.MCP.Server.Addr
VV_MCP_AUTH_TOKEN    → cfg.MCP.Server.AuthToken
```

### 3.5 校验（在 `configs.Load` applyDefaults 末尾）
- `Transport` 归一为小写；未知值 → 报错 `invalid mcp.server.transport`。
- `Transport=http` 时：若 `Addr` 空 → 默认 `"127.0.0.1:7801"`。
- `Transport=http` 且 `Addr` 解析出的 host **非 loopback** 且 `AuthToken` 为空 → 返回启动错误（defense-in-depth，避免裸 bind 0.0.0.0）。
- `SessionTimeout < 0` → 视为 0。

校验代码集中在 `mcps/transport.go` 的 `ResolveTransport(cfg)`，方便单测。

## 4. 启动流程 `mcps.Serve`

```go
func Serve(ctx context.Context, cfg *configs.Config, init *setup.InitResult) error {
    // 1. 构建 MCP server（官方 go-sdk + vage 封装）
    scanner := configs.BuildMCPCredentialScanner(cfg.Security.MCPCredentialFilter)
    srv := mcpserver.NewServer(
        mcpserver.WithCredentialScanner(scanner),
        mcpserver.WithScanCallback(buildScanCallback(slog.Default())),
    )

    // 2. 注册 agents
    exposed, err := selectAgents(init.SetupResult, cfg.MCP.Server.Agents)
    if err != nil { return err }
    for _, a := range exposed {
        if err := srv.RegisterAgent(a); err != nil { return err }
    }
    if cfg.MCP.Server.ExposeDispatcher {
        _ = srv.RegisterAgent(init.SetupResult.Dispatcher)
    }

    slog.Info("vv: mcp server ready",
        "transport", cfg.MCP.Server.Transport,
        "exposed_tools", toolNames(exposed, cfg.MCP.Server.ExposeDispatcher),
        "auth", cfg.MCP.Server.AuthToken != "")

    // 3. 起 transport
    t, err := ResolveTransport(cfg.MCP.Server)
    if err != nil { return err }

    switch t.Kind {
    case TransportStdio:
        return srv.Serve(ctx, &mcp.StdioTransport{})
    case TransportHTTP:
        return serveHTTP(ctx, srv, t, cfg.MCP.Server)
    }
}
```

### 4.1 `selectAgents`
- 空白名单 → `init.SetupResult.Agents()`（已按 ID 排序、只含 dispatchable）。
- 非空 → 逐 ID 查 `init.SetupResult.Agent(id)`；未知 ID 返回 `fmt.Errorf("mcp.server.agents[%d]=%q not registered", i, id)`。

（`setup.Result` 已有 `Agents()` 与 `Agent(id)` 两个方法，不需要新增 API。）

### 4.2 stdio transport
直接 `srv.Serve(ctx, &mcp.StdioTransport{})`。父进程（Claude Desktop 等）拉起即服务；stdin/stdout 关闭时 Serve 返回。

### 4.3 Streamable HTTP transport
```go
func serveHTTP(ctx context.Context, srv *mcpserver.Server, t *Transport, cfg configs.MCPServerConfig) error {
    opts := &mcp.StreamableHTTPOptions{
        SessionTimeout: time.Duration(cfg.SessionTimeout) * time.Second,
        // DisableLocalhostProtection: false  ← 使用默认 (开启)
    }
    handler := mcp.NewStreamableHTTPHandler(func(_ *http.Request) *mcp.Server {
        return srv.Server()
    }, opts)

    mux := http.NewServeMux()
    if cfg.AuthToken != "" {
        mux.Handle("/", bearerAuth(cfg.AuthToken, handler))
    } else {
        mux.Handle("/", handler)
    }
    mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, _ *http.Request) { w.WriteHeader(204) })

    ln, err := net.Listen("tcp", cfg.Addr)
    if err != nil { return fmt.Errorf("listen %s: %w", cfg.Addr, err) }
    srv := &http.Server{Handler: mux, ReadHeaderTimeout: 10 * time.Second}

    go func() { <-ctx.Done(); _ = srv.Shutdown(context.Background()) }()
    if err := srv.Serve(ln); err != nil && err != http.ErrServerClosed { return err }
    return nil
}
```

### 4.4 `bearerAuth` 中间件
```go
func bearerAuth(token string, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        got := r.Header.Get("Authorization")
        expected := "Bearer " + token
        if subtle.ConstantTimeCompare([]byte(got), []byte(expected)) != 1 {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

使用 `crypto/subtle.ConstantTimeCompare` 避免 timing 侧信道（虽然对 shared secret 场景影响小，但几乎零成本，值得做）。

### 4.5 loopback 判定
```go
func isLoopbackAddr(addr string) bool {
    host, _, err := net.SplitHostPort(addr)
    if err != nil { host = addr }
    if host == "" || host == "localhost" || host == "::1" { return true }
    ip := net.ParseIP(host)
    return ip != nil && ip.IsLoopback()
}
```

`"0.0.0.0"` / 非空域名 / 公网 IP → 非 loopback。

## 5. 安全基线复用

| 机制 | 入口 | 复用方式 |
|------|------|----------|
| MCP 凭据过滤 | `configs.BuildMCPCredentialScanner` | 直接传给 `WithCredentialScanner`；回调用 `slog.Warn` 打 `EventMCPCredentialDetected` 同款日志，不引入新事件类型 |
| Tool-result injection guard | 继承 `setup.Init` 给 agent 注入的 `ToolResultGuards` | 不做改动，agent 内部已挂 |
| Bash 分类 + 路径守门 + allowed_dirs | 在 `setup.Init` 内注入到各 agent 的 tool registry | MCP server 只转发到 agent.Run，路径守门天然生效 |
| CLI permission 模式 | —— | **不使用**。MCP 模式等价 HTTP："非交互"，不挂 `WrapRegistryWithPermission` |
| `ask_user` | `askuser.NonInteractiveInteractor` | 由 `setup.Options.UserInteractor` 传入（与 HTTP 同） |

### 5.1 `setup.Options` 构造（main.go）
```go
if cfg.Mode == "mcp" {
    interactor = askuser.NonInteractiveInteractor{}   // 同 HTTP
    // 不传 WrapToolRegistry
}
```
`-p` 模式与 MCP 互斥：`promptSet && cfg.Mode == "mcp"` → 报错退出。

## 6. 日志与可观测

- 启动：`slog.Info("vv: mcp server ready", transport, addr (stdio 时 "stdio"), exposed_tools, auth=true/false, session_timeout)`。
- 每次 `tools/call`：`vage` 内部已发 `ToolCallStart/End` 事件；MCP server 不再重复打一遍，仅保留凭据命中回调的 warn。
- credscrub 命中：回调里 `slog.Warn("vv: mcp credential scanner hit", direction, tool, action, hit_types)`（字段与 feature-implement.md 的既有约定一致，`Masked` 仅放摘要）。

## 7. 生命周期与信号

- `main.go` 已有 `signal.NotifyContext(ctx, SIGINT, SIGTERM)`。
- `mcps.Serve` 严格尊重 ctx：
  - stdio: `srv.Serve(ctx, &StdioTransport{})` 在 ctx 取消时返回。
  - http: 独立 goroutine 监听 `<-ctx.Done()` 后 `server.Shutdown`。
- 不创建 goroutine 泄漏：`Shutdown` 后 `Serve` 返回即函数退出。

## 8. 单元测试（`mcps/*_test.go`）

- `TestResolveTransport_Defaults` — 空配置 → stdio。
- `TestResolveTransport_HTTPLoopback` — `127.0.0.1:7801` + 无 token 合法；`Addr=""` 走默认。
- `TestResolveTransport_HTTPNonLoopbackRequiresToken` — `0.0.0.0:7801` 无 token → 错误；有 token 合法。
- `TestResolveTransport_UnknownTransport` — `transport=foo` → 错误。
- `TestBearerAuth_Missing` / `_Wrong` / `_OK`（用 `httptest.NewRecorder`）。
- `TestSelectAgents_All` / `_Whitelist` / `_UnknownID` — 用 fake registry 构造。
- `TestIsLoopbackAddr` — `127.0.0.1 / ::1 / localhost / 0.0.0.0 / example.com` 五种输入。

## 9. 集成测试（`vv/integrations/mcp_tests/`）

使用 `mcp.NewInMemoryTransports()`，避免真 LLM 调用：注入一个 stub `aimodel.ChatCompleter`（`vage/integrations/mcp_tests/integration_test.go` 有参考样例），验收：

- `TestMCPServer_ListsExposedAgents` — `cli.ListTools` 返回预期工具名集合。
- `TestMCPServer_CallCoder_EchoStub` — 调用 `coder` 工具，stub 返回固定文本；验证入参/出参贯通。
- `TestMCPServer_Whitelist` — 配置 `agents: ["coder"]`，只看到 coder。
- `TestMCPServer_ExposeDispatcher` — 打开开关后能看到 `dispatcher`。
- `TestMCPServer_AskUserNotInteractive` — 触发 `ask_user` 工具在服务端返回显式错误而不挂起。
- `TestMCPServer_CredScrub_Redact` — 触发凭据规则，验证出参被 redact。
- （HTTP 模式）`TestMCPServer_HTTPBearerAuth` — 无 header → 401，正确 header → 200 正文是合法 MCP 响应。

手动烟囱（写进 applications 文档而非测试）：
```bash
npx @modelcontextprotocol/inspector vv --mode mcp
```

## 10. 实现清单（给 developer）

有序任务，按此顺序写代码：

1. `configs/config.go` — 新增 `MCPConfig`、`MCPServerConfig`，`Config` 加字段，Load 里三个 env 覆盖 + 校验（`validateMCPServer`）。
2. `configs/config_test.go` — 补默认值 / env 覆盖 / 校验的 unit test。
3. 新增 `vv/mcps/transport.go` — `Transport` 结构体、`ResolveTransport`、`isLoopbackAddr`。
4. 新增 `vv/mcps/transport_test.go`。
5. 新增 `vv/mcps/auth.go` — `bearerAuth`。
6. 新增 `vv/mcps/auth_test.go`。
7. 新增 `vv/mcps/register.go` — `selectAgents`、`toolNames`、`buildScanCallback`。
8. 新增 `vv/mcps/register_test.go`。
9. 新增 `vv/mcps/serve.go` — `Serve` 入口 + `serveHTTP`。
10. 新增 `vv/integrations/mcp_tests/` 下 7 个用例。
11. `main.go` — 加 `case "mcp"`；`promptSet && mode==mcp` 报错；构造 Options 时按模式选 interactor，不挂 CLI permission 包装；`cfg.Mode=="mcp"` 时不构造 `permissionState`。
12. `vv/Makefile`：无需改动（`make test`自动包含新包）。
13. `doc/prd/applications/` 下新增或更新一份 MCP server 用法 + 安全注意事项（documenter 阶段）。
14. `doc/prd/overview.md` — "Does Not Cover" 中删除 "MCP server mode (planned for future)"，"Covers" 新增条目。
15. `doc/prd/feature-todo.md` — 删 P1-1 行，调整里程碑 M2 勾选状态。
16. `doc/prd/feature-implement.md` — 新增 "MCP 服务端（vv 集成）" 条目，链路关键文件写 `vv/mcps/*`。

## 11. 非目标再次确认

- 不暴露 bash/read/write/edit/glob/grep 的**原子工具**（design 刻意不给配置键，避免被解读为推荐用法）。
- 不做多用户认证、OAuth、audience、JWT —— 保留给 P4-1。
- 不做 MCP passthrough（vv 自己连接的 MCP 工具不再次暴露）。
- 不动 REST API（`httpapis/`）的端口 / 路由 —— MCP HTTP 走独立 listener。
- 不启用 Prompts / Resources / Elicitation / 动态 tool list 更新。
