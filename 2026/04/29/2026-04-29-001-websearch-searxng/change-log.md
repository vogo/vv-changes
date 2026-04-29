# Change Log — 2026-04-29-001-websearch-searxng

## 改动列表

### vage 模块

- `vage/tool/websearch/searxng.go`(新增)
  - `SearXNGProvider`、`SearXNGOption`、`NewSearXNG(endpoint, opts...)`
    构造函数(endpoint 空 → nil,语义对齐 tavily/brave/firecrawl)。
  - GET `?q=...&format=json&safesearch=1&pageno=1` + 可选 language /
    categories / results_count / Authorization Bearer。
  - 端点归一化:`http://host` 与 `http://host/search` 都接受。
  - Topic 通过 `TopicFromContext` 取出后映射为 `categories`,只放行
    SearXNG 内置类别白名单(general/news/images 等),未知 token 静默
    丢弃以避免上游 400。
  - 错误映射:401/403 → `ErrInvalidAPIKey`、其他非 2xx → `*HTTPError`、
    JSON decode → `asParseError`、200 + 顶层 `message` 且无 results →
    包装为 `HTTPError`(对齐 firecrawl)。
- `vage/tool/websearch/searxng_test.go`(新增,14 个 case 全绿)
  - 端点归一化、success、maxResults trimming、401/403、429/5xx、
    parse 失败、空结果、200+message 异常、categories+topic 解析、
    optional bearer + custom UA、language + result_count、Name()。
- `vage/tool/websearch/websearch_tool.go`
  - LLM-facing `toolDescription` 在 provider 列表里加上 SearXNG。

### vv 模块

- `vv/configs/config.go`
  - 新增 `WebSearchProviderSearXNG = "searxng"`。
  - `WebSearchConfig` 增加字段 `BaseURL / Language / Categories /
    UserAgent`(只在 searxng 时使用)。
  - `IsEnabled` 改为按 provider 分支判断:tavily/brave 要 APIKey,
    searxng 要 BaseURL,APIKey 可选(forwarded as bearer)。
  - `NormalizedWebSearchProvider` 新增 searxng case。
  - 新增 env 覆盖:`VV_WEB_SEARCH_BASE_URL`、
    `VV_WEB_SEARCH_LANGUAGE`、`VV_WEB_SEARCH_CATEGORIES`、
    `VV_WEB_SEARCH_USER_AGENT`。
  - 未知 provider 警告里把 searxng 加到 valid 列表。
- `vv/configs/web_search_test.go`
  - `TestNormalizedWebSearchProvider` 增加 searxng 大小写 case。
  - `TestWebSearchConfig_IsEnabled` 增加 4 个 searxng 子 case。
  - 新增 `TestLoad_WebSearchSearXNGEnvOverride`(全部 4 个 env 字段
    round-trip)。
  - 新增 `TestLoad_WebSearchSearXNGRequiresBaseURL`(无 base_url 不
    启用)。
- `vv/tools/websearch_factory.go`
  - `buildWebSearchProvider` 新增 searxng 分支,把所有 SearXNG-only
    config 字段透传给 provider option。
- `vv/tools/tools_test.go`
  - 新增 `TestRegister_WithWebSearchSearXNGEnabled`(三个 profile
    都注册到 web_search)。
  - 新增 `TestBuildWebSearchProvider_SearXNG_NilSafe`(空 base_url
    返回 honest nil interface)。
- `vv/integrations/setup_tests/setup_websearch_tests/setup_websearch_test.go`
  - 新增 `TestIntegration_SetupNew_WebSearch_ProviderSearXNG`。
- `vv/integrations/tools_tests/web_search_tests/searxng_live_test.go`
  (新增,opt-in via env)
  - `TestSearXNG_Live_RealInstance`,默认 Skip;通过
    `VV_WEBSEARCH_LIVE_SEARXNG_URL` 触发,可选 `_UA / _API_KEY /
    _LANGUAGE`。
  - 失败时打印操作员排查 hint(formats / limiter / UA)。

## 验证

- `cd vage && go test ./...` 全绿。
- `cd vv && go test ./...` 全绿(36 个包,含所有 integration tests)。
- `cd vage && make lint` → 0 issues。
- `cd vv && make lint` → 0 issues。
- `cd vage && go test ./tool/websearch/ -run SearXNG` → 14/14 PASS。
- `cd vv && go test ./configs/ ./tools/ -run WebSearch` → 全绿。
- `cd vv && go test ./integrations/setup_tests/setup_websearch_tests/`
  → 8/8 PASS,含新增的 SearXNG 场景。

## 真机验证(端到端)

执行命令:

```bash
cd vv
VV_WEBSEARCH_LIVE_SEARXNG_URL="http://10.225.32.180/search" \
VV_WEBSEARCH_LIVE_SEARXNG_UA="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
no_proxy="10.225.32.180" NO_PROXY="10.225.32.180" \
go test ./integrations/tools_tests/web_search_tests/ -run SearXNG_Live -v -count=1
```

结果:**FAIL — 但是属于环境侧已知约束,不阻塞代码合入。**

```
upstream returned error envelope:
  code="provider_error"
  message="websearch: upstream HTTP 429"
  status=429
hint: ensure SearXNG settings.yml has `search.formats: [html, json]`
      and either disable `server.limiter` or whitelist the supplied User-Agent
```

代码侧观察到的事实(全部符合契约):
1. 请求成功送达 `http://10.225.32.180/search?q=...&format=json` 等等。
2. 上游返回 HTTP 429。
3. `SearXNGProvider.Search` 把 429 映射为 `*HTTPError{Status:429}`。
4. `Tool.translateProviderError` 把 `*HTTPError` 翻译为
   `error_code=provider_error, status_code=429`。
5. 集成测试以可读的错误信息把锅明确甩给 operator,而不是把
   429 误报成「网络挂了」。

复制 curl 行为重现(见 spec §4):无论 GET/POST、`format=json`、
浏览器 UA,该实例对 JSON 输出一律 429,而 HTML GET (`Accept:
text/html`)正常。结论:该 SearXNG 实例的 `settings.yml` 没有把
`json` 加入 `search.formats`,或 limiter 对非 HTML 请求做了过滤。
两者都属于实例侧配置,不在本任务代码范围。

**待 operator 操作**:
1. 在该实例的 `settings.yml` 加上:
   ```yaml
   search:
     formats:
       - html
       - json
   server:
     limiter: false   # 或保留 true 并把上述浏览器 UA 加入白名单
   ```
2. 重启 SearXNG。
3. 重跑上面的 `go test ./integrations/tools_tests/web_search_tests/
   -run SearXNG_Live` 即可看到 PASS。

## Done Contract 复核

- `make build` (lint+test) 在 vage / vv 双双绿:**✅ 达成**。
- vage 单测覆盖 success / 401 / 5xx / parse_failed:**✅ 14 个 case
  全绿,远超 4 条最低线**。
- vv config 单测覆盖 searxng 启用 / 缺 base_url / env 覆盖:
  **✅ 三类场景齐备**。
- vv setup integration 覆盖 searxng + base_url 镜像 brave 的 agent
  装载:**✅ TestIntegration_SetupNew_WebSearch_ProviderSearXNG 通过**。
- 端到端真实调用:**⚠️ 环境侧未达成**(代码契约已被 429 路径完整
  验证,等 operator 调实例设置即可转 PASS)。

## Resume / Handoff

- 所有代码已合入工作树,提交时建议拆为 2 个 commit:
  1. `vage: add SearXNG provider for websearch tool` —
     `vage/tool/websearch/searxng*.go` + `websearch_tool.go` 描述更新。
  2. `vv: wire SearXNG into web_search tool` —
     `vv/configs/*` + `vv/tools/*` + `vv/integrations/**`。
- 未再分文件拆分 `searxng.go`(目前 ~245 行,远低于 800 行硬上限)。
- 真机测试一旦 operator 放开 JSON 格式,无须再改代码。
