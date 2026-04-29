# SearXNG Web Search 接入

会话目录: `changes/2026/04/29/2026-04-29-001-websearch-searxng/`
任务深度: `standard`(新增 provider + vv 配置接入 + 集成测试)
日期: 2026-04-29

## 1. 核心目标(Loop Anchor)

在 `vage/tool/websearch` 增加 SearXNG provider 实现,并在 `vv` 中接入,使
operator 仅通过配置即可让所有支持 `web_search` 的 agent 走 SearXNG 后端;
完成集成测试以证明 envelope 输出与现有 tavily/brave/firecrawl 一致。

## 2. 复述理解

- **Provider 层**: 在 `vage/tool/websearch` 加入 `searxng.go`,实现
  `Provider` 接口,沿用与 brave/tavily/firecrawl 相同的构造范式
  (`NewSearXNG(...)`, `WithSearXNG*` Option,nil 表示未配置)。
- **配置层**: `vv/configs.WebSearchConfig` 增加 `searxng` 作为合法 provider,
  补充必要字段(至少 `base_url`),同步 env 覆盖与 normalize 逻辑。
- **Wiring 层**: `vv/tools/websearch_factory.go` 增加 searxng 分支,沿用
  「concrete-typed local 防 nil interface」的 pattern。
- **测试层**:
  - `vage` 单测: httptest 模拟 SearXNG JSON 响应,覆盖成功 / 401 /
    5xx / 解析失败 / 空结果。
  - `vv` 集成测试: 沿 `setup_websearch_tests` 既有结构,新增
    `provider=searxng + base_url` 场景,验证工具被正确注册到所有期望的
    agent。
  - 真实集成验证: 由我或用户对 `http://10.225.32.180/search` 跑一次
    端到端调用(见 §6 风险),将日志/输出回写本 spec。

## 3. 范围与边界

- 仅新增 provider,不修改 `Tool` / `Provider` interface / envelope 结构。
- SearXNG 不需要 api key(社区自托管常见模式),但允许通过配置传入
  `Authorization` header(部分实例启用 secret_key/limiter token)。
- 维持 vage `Provider` 单方法 interface,topic 仍走 context(SearXNG
  支持 `categories=news` 时按 topic 映射)。
- 不动 vage 默认 provider 默认 alias、不动 LLM-facing tool description
  里的「Tavily / Brave」枚举(只在描述字符串里追加 SearXNG)。

## 4. 关键事实(已确认)

- 现有 provider: `tavily.go`, `brave.go`, `firecrawl.go`,均使用
  `httpClient + endpoint + userAgent + apiKey`;`NewXxx(apiKey, opts...)`
  在 key 为空时返回 nil。
- LLM-facing tool 在 `websearch_tool.go` 的常量 `toolDescription` 当前
  写死为「Provider is configured by the operator (Tavily / Brave)」,需
  把 SearXNG 加进去,但行为本身不变。
- vv 接入: `vv/tools/websearch_factory.go::buildWebSearchProvider` 用
  switch 分发,依赖 `configs.NormalizedWebSearchProvider`。
- vv config 当前仅有 `tavily | brave` 两个常量,`firecrawl` 尚未在 vv 端
  接入(`grep firecrawl vv/` 仅命中 vage 引用),所以本次只加 searxng,
  不顺带补 firecrawl。
- SearXNG 实例 `http://10.225.32.180/search`:
  - 浏览器 UA + GET HTML 200 OK(已 curl 验证)。
  - `format=json`(GET 或 POST)目前 **返回 429**(原因待确认: 实例
    `settings.yml` 里 `search.formats` 是否包含 `json`,或者 limiter
    针对非 HTML 请求做了过滤)。
  - 默认 UA(`curl/8.x` 或 `vv-agent-web-search/1.0`)直接 429 — 需要
    像浏览器一样的 UA。

## 5. 计划

### 5.1 vage 侧

1. 新建 `vage/tool/websearch/searxng.go`,800 行内单文件:
   - 常量: `SearXNGName = "searxng"`、默认 endpoint 留空
     (operator 必须显式配置 base_url)、`searxngMaxBodyBytes = 1<<20`。
   - `SearXNGProvider` 字段: `endpoint, httpClient, userAgent, apiKey,
     defaultLanguage, defaultCategories`。
   - `SearXNGOption`: `WithSearXNGHTTPClient/Endpoint/UserAgent/APIKey/
     Language/Categories`。
   - 构造: `NewSearXNG(endpoint string, opts...) *SearXNGProvider` —
     **endpoint 为空返回 nil**(对齐 tavily/brave 的「未配置返回 nil」
     语义);apiKey 为可选,空则不发 Authorization。
   - `Search`: 用 GET `?q=...&format=json&safesearch=1&pageno=1`,
     必要时附 `language` / `categories`。`TopicFromContext` 在
     `news` 时映射为 `categories=news`。
   - 响应解析: `{results:[{url,title,content,publishedDate}, ...]}`。
   - 错误映射: 401/403 → `ErrInvalidAPIKey`、其他非 2xx → `HTTPError`、
     decode 失败 → `asParseError`。
2. 新建 `vage/tool/websearch/searxng_test.go`:
   - 成功(3 条结果)、空结果、401、5xx、parse_failed、topic→categories
     映射。
3. 修改 `vage/tool/websearch/websearch_tool.go::toolDescription`,
   把「(Tavily / Brave)」改为「(Tavily / Brave / SearXNG)」。

### 5.2 vv 侧

1. `vv/configs/config.go`:
   - 增加常量 `WebSearchProviderSearXNG = "searxng"`。
   - `NormalizedWebSearchProvider` 增加 case。
   - `WebSearchConfig` 增加字段: `BaseURL string yaml:"base_url"`、
     可选 `Language string yaml:"language"`、`Categories string
     yaml:"categories"`。`APIKey` 复用现有字段(searxng 时可空,
     此时 `IsEnabled` 需特判)。
   - `IsEnabled`: searxng provider 时不要求 APIKey,但 BaseURL 必填;
     tavily/brave 仍要求 APIKey。
2. env 覆盖:
   - `VV_WEB_SEARCH_BASE_URL` → `BaseURL`。
   - `VV_WEB_SEARCH_LANGUAGE` → `Language`。
   - `VV_WEB_SEARCH_CATEGORIES` → `Categories`。
3. `vv/tools/websearch_factory.go::buildWebSearchProvider`:
   - 增加 searxng case。

### 5.3 测试

1. `vage/tool/websearch/searxng_test.go` 单测覆盖。
2. `vv/configs/web_search_test.go` 增加场景: searxng + base_url 启用、
   searxng 缺 base_url 不启用、env 覆盖 base_url。
3. `vv/integrations/setup_tests/setup_websearch_tests/setup_websearch_test.go`
   增加 searxng 镜像测试。
4. 端到端真机验证: 写一个独立可手动跑的 integration test
   (`vv/integrations/tools_tests/web_search_tests/searxng_live_test.go`,
   `t.Skip` if `VV_WEBSEARCH_LIVE_SEARXNG_URL` env 不存在),在
   配置好的 SearXNG 上跑一次真实查询。

## 6. 风险

- **JSON 输出 429**: 该 SearXNG 实例当前不允许我们以默认 UA 拿 JSON。
  需要 operator 在 settings.yml 中放开 `search.formats: [html, json]`,
  并允许我们设置浏览器 UA。这点不在代码改动范围,但要在 spec 里
  flagged,并由真机验证暴露。
- **User-Agent 限流**: 我们已经允许 `WithUserAgent` 透传到 provider,
  searxng provider 必须读取 `Tool.UserAgent()` 而不是写死。
- **Topic 映射**: SearXNG 的 `categories` 是显式列表(general/news/
  science 等),如果 LLM 传了无意义 topic 字符串,我们要忽略而不是
  原样塞进 query — 已通过 maxTopicRunes 边界限制,再 SearXNG 内部白
  名单过滤。
- **endpoint 配置**: SearXNG 没有「公共默认地址」,operator 必须自带,
  因此 endpoint 空必须返回 nil(和 tavily 空 apiKey 行为一致)。
- **POST vs GET**: 一些公网实例只允许 POST(防爬虫),少数只允许 GET。
  本次默认 GET,后续若有需要再补 POST option。

## 7. Done Contract

完成,当且仅当:
- `make build` 在 `vage/` 和 `vv/` 双双绿(format/lint/test 全过)。
- 新增 vage 单测覆盖成功 / 鉴权失败 / 5xx / parse_failed 至少 4 条
  分支。
- vv config 单测覆盖 searxng 启用、未配 base_url、env 覆盖至少 3 条。
- vv setup integration test 覆盖 searxng + base_url 镜像 brave 的
  agent 装载。
- 端到端: 至少 1 次真实 `http://10.225.32.180/search` 调用返回
  非空 results(若该实例 JSON 格式仍 429,记录失败原因并请 operator
  调整配置,本条作为「环境侧未达成」记录,但不阻塞代码合入)。

未达成,当:
- 上述任一条 fail。
- 改动破坏现有 tavily/brave/firecrawl 测试。
- LLM-facing tool name / envelope schema 改了。

## 8. Resume / Handoff

**实现已完成** — 详见 `change-log.md`。

- 代码侧 Done Contract 全部达成(vage / vv 双 lint + test 绿,
  含 14 个新增 SearXNG 单测 + 4 类 vv config 单测 + 1 个 setup
  integration 测试 + 1 个 opt-in live 测试)。
- 端到端真机调用验证为「环境侧未达成」: SearXNG 实例
  `http://10.225.32.180/search` 对 `format=json` 一律 429,
  代码契约已经把 429 → `provider_error` + `status_code=429`
  正确翻译,集成测试用清晰的 hint 指出需要 operator 在该实例
  `settings.yml` 启用 `search.formats: [html, json]` 并放开
  limiter / UA 白名单。详见 `change-log.md` 的「真机验证」节。
- 当 operator 调整完该实例后重跑:
  ```
  VV_WEBSEARCH_LIVE_SEARXNG_URL="http://10.225.32.180/search" \
  VV_WEBSEARCH_LIVE_SEARXNG_UA="<browser UA>" \
  go test ./vv/integrations/tools_tests/web_search_tests/ \
    -run SearXNG_Live -v -count=1
  ```
  无须再改代码。
