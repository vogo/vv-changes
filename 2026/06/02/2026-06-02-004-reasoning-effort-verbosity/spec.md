# 2026-06-02-004 OpenAI:同步 reasoning_effort 取值并新增 verbosity 参数

## 背景

官方 OpenAI Chat Completions API 变更:

- `reasoning_effort` 取值已扩展为 `none / minimal / low / medium / high / xhigh`(GPT-5.1 默认 `none`)。当前 `ChatRequest.ReasoningEffort` 为裸 `string`、无常量、文档未同步。
- 新增 `verbosity` 参数(`low / medium / high`)控制输出详略,当前缺失。

## 目标

1. 为 `reasoning_effort` 定义取值常量(`none/minimal/low/medium/high/xhigh`),字段仍保持 `string` 透传以兼容不同后端。
2. `ChatRequest` 新增 `Verbosity string`(`json:"verbosity,omitempty"`);`clone()` 标量无需特殊处理。

## 边界

- 仅 OpenAI 协议(canonical 类型),不改动 Anthropic 译码逻辑。
- 不引入校验/枚举类型,常量仅作为便捷取值供调用方引用。

## 方案

- `schema.go`:新增 `ReasoningEffort*` 一组 `string` 常量;`ChatRequest` 增 `Verbosity string` 字段(紧随 `ReasoningEffort`)。
- `openai_chat_test.go`:新增测试覆盖 `verbosity` 序列化(含 omitempty 缺省不出现)与各 `reasoning_effort` 取值常量的请求体序列化。
- 文档:`CLAUDE.md`(Key Types 说明)、`README.md`(请求参数说明)、`CHANGES.md`(新增同步条目)。

## Done Contract

- 完成:常量齐全且与官方一致;`Verbosity` 序列化为 `verbosity` 且 `omitempty` 生效;测试覆盖到位;三处文档同步。
- 证据:`make test` 通过(或 `go test ./...`),新增测试通过。
- 未完成:任一文档/代码/测试缺失,或常量取值与官方不一致。

## Change Log / Validation

### 变更

- `schema.go`:新增 `ReasoningEffort*` 常量(`none/minimal/low/medium/high/xhigh`)与 `Verbosity*` 常量(`low/medium/high`);`ChatRequest` 新增 `Verbosity string`(`json:"verbosity,omitempty"`),并为 `ReasoningEffort`/`Verbosity` 补字段注释。`clone()` 标量自动随值拷贝,未改动。
- `openai_chat_test.go`:新增 `captureRequestBody` 辅助 + `TestOpenAIChatRequestVerbosity`(verbosity 序列化 + omitempty 缺省不出现)+ `TestOpenAIChatRequestReasoningEffortValues`(六个取值常量序列化 + 与官方字面值一致校验)。
- 文档:`CHANGES.md` 新增同步条目;`CLAUDE.md` Key Types 补说明;`README.md` 新增「Reasoning effort & verbosity」小节含示例。

### 验证

- `go test ./...` → 212 passed in 7 packages(含新增 9 个 reasoning/verbosity 用例)。无 `.test` 二进制残留。

### 结论

核心目标已由测试通过证据证明完成:常量与官方一致、`Verbosity` 序列化为 `verbosity` 且 `omitempty` 生效、三处文档与代码/测试同一改动内同步。
