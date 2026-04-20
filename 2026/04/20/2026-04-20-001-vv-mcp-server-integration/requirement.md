# Requirement: vv 集成 MCP Server 模式

> P1-1 · 能力接通 · `vage/mcp/server` 已就绪，vv 未接入。

## 1. 背景与目标

### 1.1 现状
- `vage/mcp/server.Server` 已完整实现：支持 `RegisterAgent` / `RegisterTool`、inbound/outbound 凭据扫描、基于 `mcp.Transport` 的任意传输层。
- `vage/mcp/client` 已集成进 vv（间接通过 vage 工具系统）。
- vv 只能充当 MCP **消费者**（调用外部 MCP server 的工具），不能把自身的 Agent/工具能力作为 MCP **生产者**暴露给 Claude Desktop、Cursor、Cline、Goose 等 MCP 客户端。
- `doc/prd/overview.md` 明确把 "MCP server mode (planned for future)" 列在 Does Not Cover。

### 1.2 目标
让 vv 可运行为 **MCP Server**，把内置 agents（coder / researcher / reviewer / chat）与受治理的工具（read / glob / grep 等只读工具，以及受控的 bash / write / edit）暴露给外部 MCP 客户端使用。交付完成后此能力从 feature-todo 迁到 feature-implement。

### 1.3 价值
| 对象 | 获益 |
|------|------|
| vv 用户 | 用同一个 vv 配置同时服务 CLI、HTTP 和 MCP 客户端（Claude Desktop / Cursor / Cline / Goose 等） |
| 多 agent 编排场景 | 父 agent 可通过 MCP 把子任务委派给 vv 的 coder/researcher |
| 生态对齐 | 与 Claude Code `claude mcp serve`、Goose MCP 扩展、Cursor/Cline agent-as-server 等保持接口一致 |

---

## 2. 同类产品调研

### 2.1 Claude Code — `claude mcp serve`
- **传输**：仅支持 stdio（见 anthropics/claude-code issue #17949，HTTP/SSE 为 feature request，未上线）。
- **暴露内容**：Bash、Read、Write、Edit、LS、GrepTool、GlobTool、Replace — 既有 CLI 工具直接转发。
- **安全模型**：stdio 进程隔离，"only processes that launch the server can connect"；无 auth、无网络暴露。
- **限制**：无 MCP passthrough — 即使 Claude Code 自身连接了其它 MCP server，其客户端也看不到被连接 server 的工具。
- **上下文优化**：使用 Tool Search 按需发现工具，相比一次性注入全量 tool schema 减少 85–95% 上下文。

### 2.2 Goose
- MCP 是 Goose 的扩展系统底座；Goose 既是 MCP client（加载 extensions）也能作为 MCP server（通过 "subagents" 能力）。
- Extensions 可与 Claude Desktop、Cursor、Cline 通用。

### 2.3 Cursor / Cline
- 主要作为 MCP **client**，但可通过社区 wrapper（如 steipete/claude-code-mcp、mkXultra/ai-cli-mcp）把 Cursor / Cline / Codex CLI 作为 MCP server 暴露给父 agent。
- 启示：即使没有一等公民的 server 能力，CLI agent 被外部 agent 调用的需求是真实存在的，vv 直接原生支持会省掉 wrapper 层。

### 2.4 MCP Inspector / Home Assistant / Graphlit 等
- 都选择同时支持 stdio 与 Streamable HTTP：本地开发 stdio，远程/共享部署 Streamable HTTP。
- 远程模式的共识：必须 Bearer token + DNS rebinding 防护 + Origin/Host 校验。

---

## 3. 行业技术方案调研

### 3.1 传输协议（MCP 规范 2025-11-25 / 2025-03-26）
| 传输 | 适用场景 | 优点 | 风险/成本 |
|------|----------|------|-----------|
| **stdio** | 单客户端、同一台机器（被父进程拉起） | 零配置、进程隔离天然 auth、最低延迟 | 无法服务远程/多客户端；每个客户端独占一个 vv 进程 |
| **Streamable HTTP** | 跨机器 / 多客户端共享 / 长运行服务 | 多会话、可选 SSE 流、可无状态、带 session 恢复 | 必须处理 DNS 鉴权、Origin/Host 校验、Bearer token；否则局域网可直连 |
| ~~HTTP+SSE~~ | 旧客户端兼容 | — | 2025-03-26 规范已 **deprecated**，新实现不必做 |

### 3.2 安全治理关键点（MCP 安全 advisory + go-sdk 1.4.0 + auth0/solo.io 博客）
- **DNS rebinding 防护**：go-sdk `StreamableHTTPOptions.DisableLocalhostProtection` 默认开启，监听 localhost 时校验 Host header 必须也是 loopback，非法直接 403。绑定 0.0.0.0 时此默认不启用，需要额外的身份校验。
- **Origin / Host 校验**：非浏览器 client 场景仍建议做 allowlist。
- **Bearer token**：Streamable HTTP 下应允许在 `Authorization: Bearer <token>` 上挂一个简单共享密钥；对需要多用户的场景再引入 audience / JWT。
- **Session ID**：如启用会话恢复必须 cryptographically secure（UUID v4 / JWT / 哈希）。go-sdk 已内置，我们仅需不关闭它。
- **凭据过滤复用**：vv 已有 `credscrub` 与 `WithCredentialScanner` — server 入/出参的凭据扫描应沿用同一份 `MCPCredentialFilterConfig`，不要再起一份。
- **工具权限模型**：server 模式下 CLI 的 Allow / Allow-Always 交互消失；必须强制等同于 **非交互** 模式，应用 `Dangerous` 分类硬阻断、bash 路径守门、工具结果注入扫描。
- **Ask-user 工具**：server 模式无交互终端，`ask_user` 必须用 `NonInteractiveInteractor` 或直接禁用。

### 3.3 "暴露什么" 的设计分歧
| 方案 | 代表产品 | 优缺点 |
|------|----------|--------|
| A. 把原子工具（bash/read/…）直接 1:1 暴露 | Claude Code `mcp serve` | 功能面最大、父 agent 自由编排；但把 vv 的系统提示 / 调度管道旁路掉，安全面最大 |
| B. 把 Agent（coder / researcher / reviewer / chat）作为工具暴露 | Goose subagents | 保留 vv 的系统 prompt、guardrails、dispatcher 经验；父 agent 只需给一段任务文本 |
| C. 同时暴露 Dispatcher 作为单一顶层工具 | 类似 mkXultra/ai-cli-mcp 一键封装 | 最轻量；但父 agent 无法点名 agent |
| D. 混合（A + B + C） | — | 配置驱动，默认只给安全子集，用户可按需放开 |

vv 适配性评估：
- 默认推荐 **B（暴露 agents 作为工具）**：对齐现有 HTTP API 模型（`svc.RegisterAgent(...)`），`vage/mcp/server` 原生有 `RegisterAgent`；父 agent 把 vv 当成一个"可选的 coder/researcher"，而非一把散装工具。
- 可选 **C（Dispatcher）**：对只想"交一段自然语言任务、让 vv 自己分派"的客户端友好，实现成本极低（把 `dispatcher` 当 Agent 注册即可）。
- 暂不实现 **A**：把 bash / write / edit 这样高危工具裸露出去，即便有 credscrub + 注入扫描 + 路径守门，也会越过 CLI 的 permission 模式，默认关闭更稳。

### 3.4 启动模式设计（行业共识）
- Claude Code 路径：`claude` / `claude -p` / `claude mcp serve` —— server 是独立**子命令**，与交互模式互斥。
- Home Assistant、FastMCP、MCP Inspector：基本都是独立子命令/独立 flag。
- 对齐 vv 现状：vv 现有 `--mode cli|http`。**新增 `--mode mcp`** 或新增 `--mcp-stdio` / `--mcp-http=:port` 都可行；本需求在 design 阶段二选一。

---

## 4. 用户故事与验收

### US-1 作为 Claude Desktop / Cursor 用户，我希望把 vv 注册为 MCP server，从而在它们的界面里直接让 vv 完成 coding / research / review 任务
- AC-1.1 我在客户端 MCP 配置中写 `"command": "vv", "args": ["--mode", "mcp"]` 即可成功建立 MCP 会话。
- AC-1.2 客户端 `tools/list` 能列出 `coder`、`researcher`、`reviewer`、`chat`（以及可选的 `dispatcher`），每个工具都带有中文或英文描述。
- AC-1.3 客户端 `tools/call coder {"input":"..."}` 返回 coder agent 的最终文本答复，错误时返回 MCP error 结构且不崩溃服务。
- AC-1.4 每个 vv 实例可连续处理多次 `tools/call` 不泄漏状态（同一 MCP session 的连续调用复用同一 memory manager）。

### US-2 作为 vv 用户，我希望 MCP server 模式复用 CLI/HTTP 模式的安全基线
- AC-2.1 工作目录白名单、bash 危险命令分类、工具返回注入扫描、MCP 凭据过滤**全部默认启用**，与 HTTP 模式一致。
- AC-2.2 `ask_user` 工具在 MCP 模式下使用非交互 interactor，agent 触发时返回显式的"当前会话不支持交互式提问"错误而非挂起。
- AC-2.3 MCP server 入站/出站均经过 credscrub；命中高等级规则时按配置 log/redact/block。
- AC-2.4 MCP 模式下 permission 模式强制为"非交互"等价；高危 bash 分类（Dangerous / Blocked）直接阻断。

### US-3 作为 vv 用户，我希望本地（stdio）和受控 HTTP 两种暴露方式都可用
- AC-3.1 `vv --mode mcp` 默认走 stdio，适合被父 agent 拉起。
- AC-3.2 `vv --mode mcp --mcp-transport http --mcp-addr :7801` 走 Streamable HTTP，启动时强制 DNS rebinding 防护（当监听地址是 loopback 时自动启用）。
- AC-3.3 HTTP 模式下若配置了 `mcp.auth_token`，server 拒绝缺失或不匹配 `Authorization: Bearer` 的请求并返回 401。
- AC-3.4 无论哪种 transport，Ctrl+C / SIGTERM 下都能在 2 秒内优雅退出。

### US-4 作为 vv 用户，我希望通过配置选择暴露哪些 agent
- AC-4.1 `vv.yaml` 的 `mcp.server.agents` 列表支持白名单（默认 = registry 全部 dispatchable）。
- AC-4.2 支持 `mcp.server.expose_dispatcher: true|false`（默认 false），打开后以 `dispatcher` 为名额外注册一个工具。
- AC-4.3 关键配置支持 env 覆盖：`VV_MCP_TRANSPORT`、`VV_MCP_ADDR`、`VV_MCP_AUTH_TOKEN`。

### US-5 作为可观测运维，我希望 MCP server 能被统一观测
- AC-5.1 启动日志包含 transport、监听地址/stdio、暴露工具名清单、auth 开关状态。
- AC-5.2 每次 `tools/call` 以 `slog.Info` 记录 tool、latency、是否出错（不记录参数原文）。
- AC-5.3 credscrub 命中事件发到既有 `EventMCPCredentialDetected`（复用 vv 的 hook 基础设施）。

---

## 5. 范围

### In-Scope
- 新增 `--mode mcp` 启动形态（stdio 默认，Streamable HTTP 可选）。
- 暴露注册表中的 dispatchable agents 作为 MCP tools。
- 可选暴露 Dispatcher 作为顶层工具。
- 复用既有 credscrub、tool-result injection guard、bash path guardian、工作目录白名单。
- 基础 Bearer token 鉴权（共享密钥级别）。
- 配置与 env 覆盖。
- 单元测试 + 集成测试（in-memory transport 已有 vage 侧集成测试可借鉴）。
- 更新 PRD、overview.md、feature-todo.md、feature-implement.md、applications 文档。

### Out-of-Scope（留给后续迭代）
- 把 bash/read/write/edit/glob/grep 作为**原子工具**暴露（参见 §3.3-A，默认关闭；未来可加 `mcp.server.expose_tools`）。
- 多用户 auth（JWT / audience / OAuth / mTLS）—— P4-1 用户认证覆盖。
- MCP passthrough（把 vv 自己连接的 MCP server 的工具再暴露出去）。
- Session resume（go-sdk 默认能力，但我们不显式配置 EventStore）。
- Prompts / Resources / Elicitation / 动态 tool list 变更通知。
- 暴露 Skill 作为 MCP 命名空间（可在 P3 Skill 阶段再考虑）。

---

## 6. 假设与风险

| # | 假设 / 风险 | 处理 |
|---|-------------|------|
| A1 | MCP 客户端遵循 MCP 规范的会话初始化流程 | 直接使用 go-sdk，不做协议层适配 |
| A2 | stdio 模式下客户端不会在同一 vv 进程并发触发多个 `tools/call` | go-sdk 已处理并发；若后续发现竞态再加 sync |
| R1 | 暴露 agent 可能让父 agent 把任意文本当作 prompt → prompt injection | 依赖 credscrub + 工具结果注入扫描（入模前扫描）+ agent 级系统 prompt |
| R2 | Streamable HTTP 模式监听 0.0.0.0 时失去 DNS rebinding 自动防护 | 默认只允许 loopback；非 loopback 时启动强制要求 `mcp.server.auth_token` 不为空，否则启动失败 |
| R3 | 父 agent 误用 coder 去执行高危 bash | 继承 vv 现有 Dangerous 分类硬阻 + allowed_dirs |
| R4 | MCP server 长运行状态下 memory 是否共享 | design 中明确：**每个 MCP session 一份独立 memory manager**（否则多父 agent 共享 session 会串流） |
| R5 | 同进程同时跑 CLI 与 MCP server | 当前 PRD 明确不支持 "CLI + HTTP 同时运行"，MCP 模式比照处理：与 CLI / HTTP 互斥 |
| R6 | 启动失败时用户难以排查 | 启动日志必须清晰打印 transport、addr、auth 开关、exposed tools；失败走 `slog.Error` |

---

## 7. 影响面

| 模块 | 影响 |
|------|------|
| `doc/prd/` | overview.md "Does Not Cover"、applications、feature-todo/feature-implement 均需更新 |
| `vv/main.go` | 增 `--mode mcp` 分支与 flag |
| `vv/configs/` | 新增 `MCPServerConfig` 段 + env 覆盖 |
| `vv/setup/` | 初始化 MCP server 相关组件时接入既有 LLM / memory / registry |
| 新增目录 `vv/mcpserver/`（或 `vv/mcps/`） | 组装 `vage/mcp/server`、transport 选择、auth 中间件、生命周期管理 |
| `vv/httpapis/` | **不受影响** — MCP HTTP 走独立监听器，不与 REST API 共端口 |
| `vv/registries/` | 无需结构性改动；仅读取 Dispatchable 列表 |
| `vv/integrations/` | 新增 `mcp_tests/`，in-memory transport 端到端覆盖 |

---

## 8. 验证计划（给 tester 参考）

- **Unit**
  - transport 选择逻辑：flag/env/YAML 优先级
  - config 校验：非 loopback + 无 auth_token 应报错
  - agent 暴露白名单过滤
- **Integration**（用 `mcp.NewInMemoryTransports()`）
  - AC-1.1/1.2：可列出预期工具集
  - AC-1.3：tools/call 正确路由到 agent、返回文本 + 错误路径
  - AC-2.1/2.3：注入 credscrub 触发规则，确认 block/redact 生效
  - AC-2.2：`ask_user` 调用返回显式错误
  - AC-3.3：HTTP transport 无 token 返回 401、正确 token 通过
  - AC-3.4：context cancel 下 Serve 立即返回
- **手动烟囱**（可选，doc 中注明）
  - `npx @modelcontextprotocol/inspector vv --mode mcp` 检查工具列表

---

## 9. 待确认事项（由 designer 阶段给出方案）

1. 启动入口采用 `--mode mcp`（与 cli/http 并列）还是单独的 `--mcp-serve` 开关？倾向前者以保持对称。
2. 配置键命名：`mcp.server` 与既有 `mcp_credential_filter` 是否应合并为 `mcp.{server, credential_filter}`？需 designer 决策并避免破坏现有 YAML。
3. 是否默认暴露 `dispatcher` 工具？推荐默认 false，由用户显式打开。
4. `ask_user` 默认禁用还是返回显式错误？倾向后者（保留调试信号）。
5. HTTP transport 监听默认地址（`127.0.0.1:7801` 倾向于安全默认）。
