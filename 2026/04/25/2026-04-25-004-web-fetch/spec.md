# Web Fetch Spec

## Restate

参考 Anthropic Web Fetch 的产品边界，在 `vage/tool` 中新增一个 `web_fetch` 本地工具，并在 `vv` 的 tool registry / setup wiring 中接入，使 Primary / researcher / reviewer / coder 等能按各自 profile 使用它。V1 只承诺公开 URL 的静态抓取与内容提取，不承诺浏览器渲染、登录、点击、滚动或反爬绕过。

## Core Goal

给 vv agent 增加一个安全、可控、对 LLM 友好的 `URL -> 内容` 工具，作为外部信息入口，且复用现有 tool-result guard 处理第三方内容注入风险。

## Done Contract

完成条件：
- `vage/tool/webfetch` 存在可注册工具实现，支持 HTTP(S) 抓取、HTML/PDF/文本处理、内容截断、域名 allow/block、基础 robots 检查。
- `vv` 默认 wiring 能把该工具加入合适的 profile / agent 工具集。
- 有单元测试和至少一条 vv/registry 或 setup 级测试证明 wiring 生效。

不算完成：
- 只完成底层包但 vv agent 看不到该工具。
- 把动态页面当成功返回空壳内容，没有显式降级信号。

## Scope

### In

- `vage/tool/webfetch/*` 新工具包
- 工具参数 schema / handler / register
- HTML -> Markdown 主内容提取
- PDF 文本提取
- `max_content_bytes` / `max_content_chars` 类截断
- `allowed_domains` / `blocked_domains`
- robots.txt best-effort 检查
- 结果 metadata（原始 URL、最终 URL、content type、状态、是否截断）
- vv registry / setup / tool profile 接线
- 相关测试

### Out

- JS 渲染
- 浏览器自动化
- 登录态 / cookie 会话编排
- CAPTCHA / 反爬绕过
- 多 URL crawl / search

## Current Design Direction

- 工具名：`web_fetch`
- 结果保持 `schema.ToolResult` 文本输出，内容本身带结构化 JSON envelope，便于现有模型与 guard 消费。
- 对 HTML：
  - 先 fetch
  - 再 main-content 提取
  - 再转 Markdown
- 对 PDF：
  - 抽文本并按统一 envelope 返回
- 对明显动态页：
  - 返回结构化错误或 warning，提示需要后续 browser-fetch / computer-use 能力

## Affected Areas

- `vage/tool/webfetch/`
- `vv/registries/tool_access.go`
- `vv/tools/tools.go`
- `vv/setup/setup.go`
- `vv/dispatches/*` 或相关测试（如需验证 primary tools）
- 可能新增 `vv/configs/config.go` 的 `tools.web_fetch.*` 配置；若现有配置不够承载则扩展

## Risks

- 新增 HTML/PDF 依赖后的模块兼容性与体积
- robots 语义实现过重，需保持 best-effort 而非复杂 crawler
- markdown 提取质量不稳定，需要选轻量稳妥库
- 网络测试易脆弱，优先使用 `httptest` + 本地 fixture

## Validation Plan

- `go test` 覆盖 `vage/tool/webfetch`
- `go test` 覆盖 vv wiring / registry
- 如依赖允许，再跑受影响 package 的 targeted tests

## Resume Anchor

优先确认三件事：
1. `ToolProfile` 哪些 capability 需要包含 `web_fetch`
2. `vv` 是否需要新增配置项还是先走代码默认值
3. 结果 envelope 字段与错误码最小集合

## Change Log

- 新增 `vage/tool/webfetch` 本地工具，支持：
  - HTTP(S) 抓取
  - HTML 主内容提取 + 简化 Markdown 输出
  - 纯文本 / JSON 文本回传
  - 轻量 PDF 文本抽取
  - 域名 allow/block
  - best-effort robots.txt 检查
  - 动态页显式降级：`dynamic_content_requires_browser`
- `vv` 接线：
  - `CapRead` 现在同时注册 `read` + `web_fetch`
  - `vv/tools.Register*` 系列默认包含 `web_fetch`
  - coder / researcher / reviewer / primary prompt 更新为提及 `web_fetch`
  - debug registry 将 `web_fetch` 归类为 builtin + read-only
- 测试更新：
  - tool/profile/setup/http tools count 全部按新增工具调整
  - 新增 `vage/tool/webfetch/webfetch_tool_test.go`
  - `vv/go.mod` 增加本地 monorepo `replace`，确保 `vv` 看到本地 `vage` 新包

## Validation

- 通过：
  - `cd vage && env GOCACHE=/tmp/gocache go test ./tool/webfetch`
  - `cd vv && env GOCACHE=/tmp/gocache go test ./registries ./tools ./agents ./debugs ./setup ./integrations/tools_tests/tools_tests`
- 受当前沙箱限制未完成：
  - `cd vv && env GOCACHE=/tmp/gocache go test ./integrations/setup_tests/setup_tests ./integrations/httpapis_tests/http_tests`
  - 失败原因：`httptest.NewServer` 需要本地端口监听，当前环境 `bind: operation not permitted`

## Current Core Goal Status

已由代码和可运行测试证明的部分：
- `web_fetch` 工具已实现并可注册
- `vv` 已能把该工具暴露给 read-capable agents

未在当前环境完成外部证据验证的部分：
- 依赖本地监听端口的 HTTP 集成测试，仅因沙箱限制未执行完
