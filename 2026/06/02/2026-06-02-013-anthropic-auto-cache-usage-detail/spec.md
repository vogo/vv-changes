# aimodel:Anthropic 顶层 automatic caching + usage 缓存写入明细

- 日期:2026-06-02
- 模块:`aimodel`(多协议 LLM 薄封装)
- 深度:standard

## 背景

官方 Anthropic Messages API(2026-02-19 起)支持 **automatic caching**:在请求体**根级**加单个 `cache_control`(`{"type":"ephemeral"}`,可选 `"ttl":"1h"`),系统自动把缓存断点打在最后一个可缓存块上,并随会话增长自动前移,无需逐块手打断点。同时响应 `usage` 在使用 1h 缓存或混合 TTL 时返回 `cache_creation` 对象,按 TTL 细分缓存写入:`ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens`(二者之和 = `cache_creation_input_tokens`)。

当前封装现状:
- 仅支持**块级** `cache_control`(`Message.CacheBreakpoint` / `Tool.CacheBreakpoint`,见 `anthropic.go` 的 `ephemeralCache()`),无顶层开关。
- `anthropicUsage` 仅有 `cache_creation_input_tokens` / `cache_read_input_tokens` 合计;canonical `Usage` 只暴露 `CacheReadTokens`,**完全没有**缓存写入(creation)相关字段,也无 5m/1h 细分。

线上格式(已查证官方文档):

```jsonc
// 请求根级
{ "model": "...", "max_tokens": 1024, "cache_control": { "type": "ephemeral", "ttl": "1h" }, "system": "...", "messages": [...] }
// 响应 usage
{ "usage": { "input_tokens": 2048, "cache_read_input_tokens": 1800,
  "cache_creation_input_tokens": 248, "output_tokens": 503,
  "cache_creation": { "ephemeral_5m_input_tokens": 148, "ephemeral_1h_input_tokens": 100 } } }
```

## 目标

1. `ChatRequest` 级开关驱动顶层 `cache_control` 输出(Anthropic 侧),与现有块级缓存共存。
2. 解析 `usage.cache_creation` 5m/1h 细分,在 canonical `Usage` 暴露(非流式 + 流式)。

## 关键设计决策(★ 待用户确认开关形态)

### A. 顶层 cache_control 开关 —— 两个候选形态

- **形态 1(bool 开关,推荐起点)**:`ChatRequest.AutoCache bool`(`json:"-"`,结构体局部,绝不进 OpenAI canonical 请求体)。`true` → `anthropicRequest` 顶层输出 `{"type":"ephemeral"}`(默认 5m)。最贴合"一个开关"语义,与既有 `CacheBreakpoint bool` 同构。**不支持 1h**——则自生成流量的 `ephemeral_1h_input_tokens` 恒为 0(明细解析仍对手工/外部场景有效)。
- **形态 2(bool + TTL,功能完整)**:在形态 1 基础上加 `ChatRequest.AutoCacheTTL string`(`json:"-"`,空=默认 5m,可设 `"1h"`)。`anthropicCacheControl` 加 `TTL string json:"ttl,omitempty"`。使 1h 缓存可达,令 1h 明细端到端可被本封装自身触发。

> 两形态对"不开关零变化""共存""明细解析"均满足;差异仅在能否主动请求 1h 缓存。**默认推荐形态 2**(完整且代价低:多一个可选字段 + `anthropicCacheControl` 加一个 omitempty 字段)。

### B. canonical Usage 暴露(已查证当前缺失缓存写入字段)

新增三个扁平字段(与既有 `CacheReadTokens` 扁平风格一致,均 `omitempty`):
- `CacheWriteTokens int json:"cache_write_tokens,omitempty"` ← `cache_creation_input_tokens`(总写入,当前完全未暴露,补齐与 `CacheReadTokens` 对称)。
- `CacheWrite5mTokens int json:"cache_write_5m_tokens,omitempty"` ← `cache_creation.ephemeral_5m_input_tokens`。
- `CacheWrite1hTokens int json:"cache_write_1h_tokens,omitempty"` ← `cache_creation.ephemeral_1h_input_tokens`。

`Usage.Add` 累加这三项;`Usage.UnmarshalJSON` / `usageJSON` 同步加三字段并拷贝(保证 canonical Usage 的 JSON 往返不丢字段)。

> 写入 token 仍计入 `PromptTokens`(经 `totalInputTokens()` 折算,行为不变);新增字段为**附加可观测性**,语义与 `CacheReadTokens` 一致(也是 `PromptTokens` 的子集且单列)。

## 边界 / 非目标

- 不改块级 `CacheBreakpoint` 既有行为;顶层与块级为独立字段,可并存。
- OpenAI 侧零改动:新开关 `json:"-"` 不进 canonical 请求体;OpenAI 响应无 `cache_creation`,新 Usage 字段 `omitempty` 自然为 0/省略。
- 不动 `composes`;不做取值校验(薄封装,无 validation)。
- `Usage.UnmarshalJSON` 不解析 Anthropic 的嵌套 `cache_creation`(Anthropic 响应走 `anthropicUsage` → `fromAnthropicResponse`,不经 canonical Usage 的 Unmarshal);仅为 canonical Usage 自身 JSON 往返补字段。

## 改动点

1. `schema.go`:`ChatRequest` 加顶层缓存开关字段(形态 1 或 2,`json:"-"`);`Usage` + `usageJSON` 加三个缓存写入字段并在 `UnmarshalJSON` 拷贝;`Usage.Add` 累加三项。
2. `anthropic.go`:
   - `anthropicRequest` 加顶层 `CacheControl *anthropicCacheControl json:"cache_control,omitempty"`;(形态 2)`anthropicCacheControl` 加 `TTL string json:"ttl,omitempty"`。
   - `toAnthropicRequest`:开关开启时设 `ar.CacheControl`(形态 2 含 ttl)。
   - `anthropicUsage` 加 `CacheCreation *anthropicCacheCreation json:"cache_creation,omitempty"`(`{Ephemeral5mInputTokens, Ephemeral1hInputTokens int}`)。
   - `fromAnthropicResponse` 的 `Usage{}` 填 `CacheWriteTokens`(= `CacheCreationInputTokens`)与 5m/1h(来自 `CacheCreation`,nil 安全)。
3. `anthropic_stream.go`:`message_start` 捕获缓存写入 token,终态 `message_delta` 的 usage chunk 填入三字段。
4. 测试:
   - `anthropic_cache_test.go`:开关开启 → 顶层 `cache_control` 序列化(形态 2 再断言 ttl);开关关闭 → 请求体无顶层 `cache_control`(且与块级共存场景计数正确)。
   - usage 明细:`anthropicUsage` JSON 反序列化 + `fromAnthropicResponse` 映射 5m/1h/总写入;流式终态 chunk 携带明细。
   - `Usage.Add` / `Usage` JSON 往返覆盖新字段。
5. 文档同步:`CHANGES.md`(新增条目)、`CLAUDE.md`(Key Types 的 `Usage` 段 + Anthropic Translation 段)、`README.md`(对应段)。

## Done Contract

- **完成标准**:开关开启时 Anthropic 请求体含顶层 `cache_control`(形态 2 含 ttl),关闭时不含且 OpenAI 侧无任何泄漏;`usage.cache_creation` 5m/1h + 总写入在非流式与流式 canonical `Usage` 均正确暴露;`Usage.Add`/JSON 往返覆盖新字段;单测覆盖"开关序列化(开/关)"与"usage 明细解析";三处文档同步。
- **完成证据**:`go test ./...`(aimodel)全绿;`make build`(license/format/lint/test)EXIT=0 且无遗留二进制。
- **未完成判定**:开关关闭时出现顶层 `cache_control`,或字段泄漏进 OpenAI 请求体;明细解析缺失/错位;往返丢字段;文档未同步。

## Spec Self-Check

- 完整性:覆盖请求开关(非流式+流式)、usage 解析(非流式+流式)、Add/往返、测试、文档;OpenAI 侧已论证零改。
- 一致性:开关沿用 `CacheBreakpoint` 的 `json:"-"` 结构体局部约定;Usage 新字段沿用 `CacheReadTokens` 扁平 + omitempty 风格与命名(read/write 对称)。
- 可测试性:开关开/关由 JSON 序列化断言;明细由 `anthropicUsage` 反序列化 + 映射断言 + 流式 chunk 断言。
- 无歧义:字段名、json tag、映射点、ttl 取值("5m"默认/"1h")均明确;唯一待定项 = 开关形态(已列两候选并标推荐)。
- 可行性:纯本地改动,无外部依赖;线上格式已查证官方文档。

## 待确认 / 批准(已确认)

1. ★ 顶层缓存开关形态:**形态 2(bool+TTL)** —— 用户已选。
2. 执行批准:**已批准**。

## Change Log(2026-06-02 已执行)

- `schema.go`:
  - `ChatRequest` 在 `PromptCacheKey` 后新增 `AutoCache bool` + `AutoCacheTTL string`(均 `json:"-"`,结构体局部)。
  - `Usage` 新增 `CacheWriteTokens` / `CacheWrite5mTokens` / `CacheWrite1hTokens`(均 `json:",omitempty"`);`usageJSON` 同步加三字段;`UnmarshalJSON` 拷贝;`Add` 累加。
- `anthropic.go`:
  - `anthropicRequest` 新增顶层 `CacheControl *anthropicCacheControl json:"cache_control,omitempty"`;`anthropicCacheControl` 加 `TTL string json:"ttl,omitempty"`。
  - `toAnthropicRequest`:`req.AutoCache` 为真时设 `ar.CacheControl = {Type:"ephemeral", TTL:req.AutoCacheTTL}`。
  - `anthropicUsage` 加 `CacheCreation *anthropicCacheCreation json:"cache_creation,omitempty"`(`{Ephemeral5mInputTokens, Ephemeral1hInputTokens}`)。
  - 抽出共享 `anthropicCanonicalUsage(*anthropicUsage) Usage`,`fromAnthropicResponse` 改用之,填 `CacheWriteTokens`/5m/1h(nil 安全)。
- `anthropic_stream.go`:将 `inputTokens`/`cacheReadTokens` 两个标量重构为 `startUsage anthropicUsage`(message_start 捕获),终态 `message_delta` 覆盖 `OutputTokens` 后经 `anthropicCanonicalUsage` 生成 usage chunk —— 缓存写入明细随之自动携带。
- 测试(+10):`anthropic_cache_test.go`(`_AutoCacheDefault`/`_AutoCache1h`/`_AutoCacheOff` 共存/`NoAutoCacheLeak`/`_CacheCreationBreakdown`/`_NoCacheCreation`)、`anthropic_stream_test.go`(`StreamCacheCreationBreakdown`)、`schema_test.go`(`AddWithCacheWriteTokens`/`JSONRoundTrip`/`OmittedWhenZero`)。
- 文档:`CHANGES.md` 新增条目;`CLAUDE.md`(`Usage` Key Type + Anthropic Translation 段);`README.md`(common request fields 加 `AutoCache`/`AutoCacheTTL`,Response usage 加 `CacheWriteTokens`/5m/1h)。

## Validation

- `go test ./...`(aimodel):264 passed in 7 packages(含新增 10 个)。
- `make build`(license-check → format → lint → test):EXIT=0;`coverage.out` 正常生成;无遗留构建二进制(`find` 校验无 `aimodel`/`*.test`)。
- 默认行为零变化:`AutoCache` 为 false 时 Anthropic 请求体无顶层 `cache_control`(由 `_AutoCacheOff` 断言),且开关绝不进 OpenAI 体(由 `NoAutoCacheLeak` 断言);块级 `CacheBreakpoint` 行为不变并与顶层共存。

## 结论

核心目标已由证据证明完成:① 顶层 automatic caching 由 `ChatRequest.AutoCache`(+`AutoCacheTTL`)开关驱动,序列化为请求根级 `cache_control`(支持 ttl),与块级缓存共存,关闭/OpenAI 侧零泄漏;② `usage.cache_creation` 5m/1h 与总写入在非流式与流式 canonical `Usage` 正确暴露并可累加/往返。开关序列化(开/关/1h)与 usage 明细解析均有单测,三处文档同步,`make build` 全绿。无后续遗留循环。
