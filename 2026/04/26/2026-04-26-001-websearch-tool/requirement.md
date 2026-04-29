# P2-10 WebSearch 工具 — 需求

## 1. 背景与目标

### 背景
- 来源：`doc/prd/feature-todo.md` P2-10 「WebSearch 工具」，依赖 P1-10 WebFetch（已落地）。
- 当前 vv 已经具备 `web_fetch`（URL → 内容），但缺少 **关键词 → URL** 的入口。Agent 在面对开放性问题（"近期 Go 1.24 有什么新特性"、"某框架的最新文档地址"）时只能依赖训练截止日期前的内部知识。
- 主流对齐：Claude Code、Perplexity、ChatGPT 已将 WebSearch 作为标配工具；2026 Q1 起 Anthropic / OpenAI 客户端开始内置 first-party search。

### 目标
为 vv agent 增加一个 **安全、可配置、对 LLM 友好** 的 `web_search` 工具：
- 关键词 → 候选 URL/title/snippet 列表（不抓正文）
- 通过可插拔 Provider 抽象解耦具体搜索后端
- V1 支持 **Tavily** + **Brave Search** 两个 Provider
- 与已有 `web_fetch` 组合即可让 LLM 自主完成「搜→读」链路，无需在工具侧绑定

### 非目标（V1）
- 不在 `web_search` 内部自动调用 `web_fetch` 抓正文（保持 token 可控、错误隔离）
- 不实现深度研究 dispatcher（属 P3-12，依赖本工具）
- 不做向量重排 / 多 Provider 融合排序（V1 单 Provider 单调用）
- 不实现搜索结果缓存（短期可由 Cache Middleware 在 LLM 层兜底）
- 不实现自定义爬虫 / SERP 解析（一律走 Provider 官方 API）

## 2. 用户故事 & 验收标准

### US-1：作为 Agent，我能用关键词搜索互联网
**前置**：vv 配置中至少存在一个 Provider 的 API Key。
**操作**：LLM 在工具调用中选择 `web_search`，传入 `query` 和可选 `max_results`。
**期望**：在 ≤ 10s 内拿到 ≤ N 条结构化结果（每条含 `url` / `title` / `snippet`），且响应是确定性的 JSON envelope。
**验收**：
- AC-1.1 默认 `max_results=5`，上限 `max_results=20`，越界自动夹取并在 `warnings` 中记录。
- AC-1.2 结果按 Provider 自身排序保留；不进行二次重排。
- AC-1.3 Provider 返回 0 结果时，envelope 含空 `results: []` 且 `IsError=false`，附 `warnings: ["no_results"]`。

### US-2：作为运维，我能通过配置选择 Provider
**前置**：本地 `~/.vv/vv.yaml`。
**操作**：在 `tools.web_search.provider` 指定 `tavily` 或 `brave`；在 `tools.web_search.api_key` 或对应 env (`VV_WEB_SEARCH_API_KEY`) 注入凭据。
**期望**：vv 启动时根据配置加载对应 Provider，并把 `web_search` 暴露给所有 read-capable agents。
**验收**：
- AC-2.1 `tools.web_search.provider` 为空 / `api_key` 为空 → `web_search` **不注册**（不出现在任何 agent 的 ToolDef 列表）。
- AC-2.2 `provider` 为未知值 → vv 启动时 `slog.Warn` 并跳过注册（不阻塞启动；与 trace/eval 默认关失败处理一致）。
- AC-2.3 `provider=tavily` 但仅设置了 brave 的 key（或反之）→ 视同缺 key，不注册。
- AC-2.4 环境变量 `VV_WEB_SEARCH_PROVIDER` / `VV_WEB_SEARCH_API_KEY` 优先于 YAML。

### US-3：作为 Agent，我能用搜索结果作为 web_fetch 输入
**前置**：`web_search` 与 `web_fetch` 均已注册。
**操作**：LLM 用 `web_search` 拿到 `results[0].url` 后，下一轮 tool call 用 `web_fetch` 抓取该 URL。
**期望**：URL 字段是合法 HTTPS 字符串，可直接被 `web_fetch` 消费；不需要二次清洗。
**验收**：
- AC-3.1 `url` 字段保留 Provider 返回的原值（去除前后空白），不重写、不做短链解析。
- AC-3.2 `web_fetch` 现有 SSRF 防护（`blockPrivateAddrControl`）独立生效，与 `web_search` 互不影响。

### US-4：作为安全侧，恶意 query / Provider 返回的恶意结果不应放大攻击面
**前置**：默认安全配置生效。
**操作**：LLM 提交超长 query；或 Provider 返回含 prompt-injection 标记的 snippet。
**期望**：
**验收**：
- AC-4.1 `query` 长度上限 1024 字符，超过即报 `invalid_arguments`，不发往 Provider。
- AC-4.2 结果包装为 JSON envelope 进入 `schema.ToolResult`，自动经过现有 `ToolResultInjectionGuard`（已生效，无需新增代码）。
- AC-4.3 Provider 返回 4xx/5xx → envelope `error_code: provider_error`，原始 status_code 透传，`IsError=true`。
- AC-4.4 网络超时（默认 10s，可通过 `WithTimeout` 调整）→ `error_code: timeout`。

### US-5：作为运维，我能审计每次搜索调用
**前置**：`trace.enabled=true`（P1-5 轨迹落盘已落地）。
**操作**：观测 `<trace.dir>/<projectHash>/<sid>.jsonl`。
**期望**：
**验收**：
- AC-5.1 每次 `web_search` 调用在 `tool_call_start` / `tool_call_end` 中按现有事件 schema 输出，args 含 `query` / `max_results`，result 含完整 envelope。
- AC-5.2 envelope 不包含 API Key（API Key 只出现在 HTTP 请求 header / query，不进入 result）。

## 3. In-Scope / Out-of-Scope

### In-Scope（V1）
- `vage/tool/websearch/` 新工具包：Provider 接口、Tavily 实现、Brave 实现、Tool/Handler/Register。
- 工具参数 schema：`query` (required) / `max_results` (optional) / `topic` (optional, 仅 Tavily 暴露)。
- 结果 envelope：`query` / `provider` / `results[]{url,title,snippet,published_at?}` / `retrieved_at` / `warnings[]?` / `error_code?` / `message?`。
- vv 接线：
  - `vv/configs/config.go` 新增 `tools.web_search.{provider,api_key,timeout_seconds,max_results}` 与 env override。
  - `vv/registries/tool_access.go` 在 `CapRead` 分支条件注册 `web_search`。
  - `vv/tools/tools.go` 三个工厂（Register / RegisterReadOnly / RegisterReviewTools）条件注册 `web_search`。
  - 四个 agent prompt（coder/researcher/reviewer/primary）追加一句"can use web_search for keyword → URL discovery"。
- 测试：
  - `vage/tool/websearch/` 单元测试（Provider 接口契约、参数校验、错误路径、HTTP 桩 server 端到端）。
  - `vv/configs/` 配置加载/env 覆盖单测。
  - `vv/registries/` & `vv/tools/` 注册/不注册分支单测。
  - `vv/integrations/tools_tests/` 一项 wiring smoke test。

### Out-of-Scope
- Serper / Exa / Google Custom Search Provider 实现（按主流度延后到 V2/V3）
- 内置结果缓存层
- 自动 fan-out 到 web_fetch 的"一站式"模式（feature-todo 也明确归属 P3-12 深度研究）
- 多 Provider 联合调用 / 重排
- 跨调用频率限制（暂依赖 LLM Middleware Rate Limit；若需要 per-tool 限流走 P2-9 Hooks）
- 结果差异化排版（snippet HTML 清洗仅做最小化：去标签 + 截断到 280 字）
- HTTP API / CLI 直接暴露搜索（仅 LLM tool call 入口）

## 4. 影响范围

### 角色
- **Developer**：新增 `vage/tool/websearch/` 包；维护 vv 接线。
- **Operator**：新增 1~2 项 YAML 配置，按需注入 API Key。
- **End User (Agent caller)**：行为基本透明；CLI/HTTP 模式无 UI 变化，仅 LLM 多了一个工具。

### 数据 / 模型
- 不引入新模型 / 不修改持久化 schema。
- 配置项扩展（向后兼容：缺省即不注册，零成本路径）。

### 流程
- LLM ReAct 循环增加 `web_search` 选项；与 `web_fetch` 自然组合。
- Dispatcher / Primary Assistant 无新增分支（工具发现走 ToolDef 列表）。

### 应用 / 模块
- `vage/tool/websearch/`（新增）
- `vage/tool/` README / 索引（如有）
- `vv/configs/` 配置 schema 与 setup
- `vv/registries/` Cap 注册
- `vv/tools/` 三个工厂
- `vv/agents/` prompt 文案微调
- `vv/debugs/`（如有 builtin 列表）
- 测试若干

## 5. 验证策略
- **离线**：`vage/tool/websearch/` 用 `httptest.Server` 模拟 Provider 响应，覆盖 200 / 4xx / 5xx / timeout / 空结果 / 字段缺失。
- **wiring**：vv `setup` / registry 测试断言 `web_search` 在配置齐备时存在、缺 key 时缺席。
- **回归**：现有 `tools_tests` 工具计数断言需要按"配置 key 时 +1，缺 key 时 +0"两种 case 各跑一次。

## 6. 显式假设

| 假设 | 风险 | 缓解 |
|------|------|------|
| Tavily / Brave 公开 API 在国内可访问 | 部分网络不可达 | 走 `http.ProxyFromEnvironment`（vv 已默认）；不在工具内做 proxy 配置抽象 |
| Provider HTTP 响应符合官方文档示例 | 字段微调导致解析失败 | Provider 解析层 `json.RawMessage` + 字段缺省宽容；`error_code: parse_failed` 兜底 |
| 单 query 查询足以满足 V1 | 无法支撑 multi-hop | V1 不解决；P3-12 深度研究在 dispatcher 层组合多次调用 |
| LLM 会主动降低 `max_results` 以省 token | 全开 20 时 envelope 偏大 | 上限 20 即可控（每条 ≤ 500 字符），与 todo_write 100 项类似硬上限 |

## 7. 不一致点 / 后续待解
- `feature-implement.md` 描述 P1-10 web_fetch 时未提到 web_search；本变更落地后需要 documenter 阶段在 PRD overview 与 feature-implement 同步说明 web_search 状态。
- 是否将 `web_search` 标注为 `ReadOnly=true`：本工具 **不写工作区**，应标 `ReadOnly=true`，与 `web_fetch` 一致；这意味着 CLI plan-mode 下也允许使用，需要在 documenter 阶段在 PRD 中确认这一安全立场。
