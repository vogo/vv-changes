# aimodel/Client:新增 anthropic-beta 头与 anthropic-version 可配置机制

- 日期:2026-06-02
- 模块:`aimodel`(Anthropic Messages API 薄封装)
- 深度:standard

## 背景

`setAnthropicHeaders`(`anthropic_chat.go`)当前硬编码:

```go
req.Header.Set("anthropic-version", anthropicAPIVersion) // 常量 "2023-06-01"
```

且**无 `anthropic-beta` 头机制**。官方众多能力(compaction、context-editing、structured-outputs(beta 期)、fast-mode、advisor 等)需通过 `anthropic-beta` 头启用(多个值以逗号拼接),当前封装无法触达。

## 目标

提供 Client 级别可配置项:
- 可设置一个或多个 `anthropic-beta` 值(发送时逗号拼接)。
- 可覆盖 `anthropic-version`。

作为后续 beta 能力同步的**基础设施**,本次不接入任何具体 beta 能力。

## 关键设计决策

- 沿用现有 Option 模式,新增两个 Option,字段挂在 `Client` 上:
  - `WithAnthropicBeta(values ...string) Option` —— **追加**语义(多次调用累积,单次可传多个),便于按能力组合;过滤空串。
  - `WithAnthropicVersion(version string) Option` —— 覆盖默认 version;空串视为不覆盖(保持默认)。
- `setAnthropicHeaders` 行为:
  - version:`c.anthropicVersion` 非空则用之,否则用常量 `anthropicAPIVersion`。
  - beta:`c.anthropicBeta` 过滤空串后非空时,`req.Header.Set("anthropic-beta", strings.Join(vals, ","))`;否则**不设置该头**。
- **默认行为不变**:无配置时 version 仍为 `2023-06-01`,且无 `anthropic-beta` 头。

## 边界 / 非目标

- 不接入任何具体 beta 能力(不解析 compaction/context-editing 等请求/响应字段)。
- 不做 beta 值的合法性/枚举校验(透传字符串,官方 beta 名称持续演进)。
- 不动 OpenAI 侧、不动 `composes`。

## 改动点

1. `client.go`
   - `Client` 增字段:`anthropicBeta []string`、`anthropicVersion string`。
   - 新增 `WithAnthropicBeta(values ...string) Option`(append + 过滤空串)。
   - 新增 `WithAnthropicVersion(version string) Option`(空串不覆盖)。
2. `anthropic_chat.go`
   - `setAnthropicHeaders` 按上述决策输出 version 与可选 beta 头。
   - (可加私有 helper `anthropicVersionHeader()` 解析最终 version,保持函数简洁。)
3. 测试(`anthropic_chat_test.go` 或同包新增)
   - 默认:无 Option → version=`2023-06-01`、无 beta 头。
   - 单个 beta 值。
   - 多个 beta 值(单次传多 + 多次调用累积)→ 逗号拼接。
   - 自定义 version。
4. 文档同步:`CHANGES.md`(新增条目)、`CLAUDE.md`(Client Configuration 段)、`README.md`(Client Options 段)。

## Done Contract

- **完成标准**:两个 Option 实现并生效;`setAnthropicHeaders` 正确输出 version/beta 头;默认行为零变化;单测覆盖默认/单值/多值/自定义 version 四类场景;三处文档同步。
- **完成证据**:`go test ./...`(aimodel 模块)全绿,新增/相关测试通过;`make build`(license/format/lint/test)通过。
- **未完成判定**:任一测试场景缺失或失败;默认行为被改变;文档未同步。

## Spec Self-Check

- 完整性:覆盖 Option/Header/测试/文档四面。
- 一致性:与现有 Option 模式、常量命名一致。
- 可测试性:四类场景 + 默认行为均可由 httptest 断言。
- 无歧义:beta 追加语义、空串处理、version 覆盖规则已明确。
- 可行性:纯本地小改动,无外部依赖。

## Change Log(2026-06-02 已执行)

- `client.go`:`Client` 增 `anthropicBeta []string` / `anthropicVersion string`;新增 `WithAnthropicBeta(...string)`(追加 + 过滤空串)、`WithAnthropicVersion(string)`(空串不覆盖)。
- `anthropic_chat.go`:`setAnthropicHeaders` 改为 version 走 `anthropicVersionHeader()`(配置值或默认 `2023-06-01`),beta 经 `anthropicBetaHeader()` 逗号拼接且仅非空时 `Set`;新增两个私有 helper;补 `strings` import。
- `anthropic_chat_test.go`:新增表驱动 `TestSetAnthropicHeaders`,覆盖 默认 / 单 beta / 单次多 beta / 多次累积(含空串) / 自定义 version / version+beta。
- 文档:`CHANGES.md` 新增条目;`CLAUDE.md` Client Configuration 段补两 Option;`README.md` Client Options 段补 Anthropic header 示例。

## Validation

- `go test ./...`:251 passed in 7 packages(全绿)。
- `make build`(license-check → format → lint → test):EXIT=0,无遗留二进制。
- 默认行为零变化:`TestAnthropicChatCompletion`(断言 `anthropic-version == anthropicAPIVersion`)仍通过;无配置时不发 `anthropic-beta` 头(`TestSetAnthropicHeaders/default` 断言 header 缺失)。

## 结论

核心目标已由证据证明完成:Client 级 `anthropic-beta`/`anthropic-version` 可配置基础设施落地,默认行为不变,四类测试场景全覆盖,三处文档同步。无后续遗留循环。
