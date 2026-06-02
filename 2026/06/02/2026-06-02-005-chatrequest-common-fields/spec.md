# 2026-06-02-005 OpenAI:扩展 ChatRequest 常用请求字段

## 背景

官方 OpenAI Chat Completions 提供多项常用请求参数,当前 `aimodel/schema.go` 的 `ChatRequest` 均未覆盖,影响可观测性、缓存路由与工具并发控制。

## 目标(请求字段,均 omitempty)

| 字段 | 类型 | JSON 键 |
|------|------|---------|
| Logprobs | `*bool` | `logprobs` |
| TopLogprobs | `*int` | `top_logprobs` |
| LogitBias | `map[string]int` | `logit_bias` |
| ParallelToolCalls | `*bool` | `parallel_tool_calls` |
| ServiceTier | `string` | `service_tier` |
| Store | `*bool` | `store` |
| Metadata | `map[string]string` | `metadata` |
| PromptCacheKey | `string` | `prompt_cache_key` |

`clone()` 对两个 map 字段(`LogitBias` / `Metadata`)做深拷贝,避免副本修改影响原请求。

## 决策:响应端 logprobs 解析(已确认纳入)

`Choice` 新增 `LogProbs *LogProbs`(`json:"logprobs,omitempty"`)解析响应 `choices[].logprobs`:
- `LogProbs{ Content []TokenLogprob; Refusal []TokenLogprob }`
- `TokenLogprob{ Token string; Logprob float64; Bytes []int; TopLogprobs []TopLogprob }`
- `TopLogprob{ Token string; Logprob float64; Bytes []int }`
并补响应解析测试。

## 边界

- 仅 OpenAI canonical 类型;不改动 Anthropic 译码逻辑(这些字段 Anthropic 无对应,随 canonical 序列化但 Anthropic 路径不读取)。
- 不引入校验/枚举类型;`ServiceTier` 保持裸 `string` 透传。

## 方案

- `schema.go`:`ChatRequest` 末尾新增 8 个字段;`clone()` 增加 `LogitBias`/`Metadata` 深拷贝。
- `openai_chat_test.go`:覆盖各字段请求体序列化(含 omitempty 缺省不出现)。
- `schema_test.go`:扩展/新增 `TestChatRequestClone`,验证修改副本 map 不影响原值。
- 文档:`CLAUDE.md`、`README.md`、`CHANGES.md` 同步。

## Done Contract

- 完成:8 字段序列化键名与 omitempty 正确;map 深拷贝在 `clone()` 生效(测试证明改副本不影响原值);测试覆盖到位;三处文档同步。
- 证据:`go test ./...` 通过,新增测试通过,无 `.test` 二进制残留。
- 未完成:任一代码/测试/文档缺失,或 map 仍为浅拷贝。

## Change Log / Validation

### 变更

- `schema.go`:`ChatRequest` 新增 8 个 `omitempty` 字段(Logprobs/TopLogprobs/LogitBias/ParallelToolCalls/ServiceTier/Store/Metadata/PromptCacheKey);`clone()` 用 `maps.Clone` 深拷贝 `LogitBias`/`Metadata`(新增 `maps` import);`Choice` 新增 `LogProbs *LogProbs`,并补 `LogProbs`/`TokenLogprob`/`TopLogprob` 三个响应类型。
- `openai_chat_test.go`:`TestOpenAIChatRequestCommonFields`(8 字段请求体序列化键名/值校验)+ `TestOpenAIChatRequestCommonFieldsOmitEmpty`(缺省时全部 omit)。
- `schema_test.go`:扩展 `TestChatRequestClone`(改副本 map 不影响原值)+ `TestChoiceLogProbsUnmarshal`(响应 logprobs 含 top_logprobs 嵌套解析)+ `TestChoiceLogProbsOmittedWhenNil`(nil 时不序列化)。
- 文档:`CLAUDE.md` Key Types 补 8 字段 + LogProbs 说明;`README.md` 新增「Common request fields」小节含示例;`CHANGES.md` 新增同步条目。

### 验证

- `go test ./...` → 216 passed in 7 packages。`gofmt -l` 无输出、`go vet` 无问题、无 `.test` 二进制残留。

### 结论

核心目标已由测试证据证明完成:8 字段序列化键名与 omitempty 正确、map 深拷贝在 `clone()` 隔离生效、响应 logprobs 解析到位、三处文档与代码/测试同一改动内同步。
