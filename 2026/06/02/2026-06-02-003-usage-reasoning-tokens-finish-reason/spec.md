# 2026-06-02-003 aimodel/OpenAI 响应类型对齐:reasoning_tokens 与 finish_reason 常量

## 核心目标(Loop Anchor)

让 `aimodel` 的响应类型与官方 OpenAI Chat Completions 对齐两处:
1. `Usage` 能解析并累加推理模型的 `completion_tokens_details.reasoning_tokens`;
2. `FinishReason` 补全官方取值(至少 `content_filter`,评估补 `function_call`)。

## 边界

- 仅改 `aimodel/`,不动 `vage`/`vv`。
- 不引入新依赖(SDK 保持零依赖)。
- 不改动 Anthropic 翻译逻辑(reasoning_tokens 是 OpenAI 字段;Anthropic 无此嵌套结构)。
- 字段命名遵循现有风格:导出字段 `ReasoningTokens int` + `json:"reasoning_tokens,omitempty"`。

## 关键事实

- `schema.go:318` `Usage` 结构;`:327` `usageJSON` 已用嵌套 details 模式解析 `prompt_tokens_details.cached_tokens`(`promptTokensDetails`,`:336`)。照此模式新增 `completionTokensDetails`。
- `Usage.UnmarshalJSON`(`:343`)、`Usage.Add`(`:363`)需同步处理新字段。
- `FinishReason` 常量在 `schema.go:40-44`,仅 `stop/length/tool_calls`。
- 官方 finish_reason 取值:`stop`/`length`/`tool_calls`/`content_filter`/`function_call`(后者为已弃用 functions API 的历史兼容值)。
- 测试参照 `schema_test.go:569` `TestUsageOpenAIPromptTokensDetails` 模式新增。
- 文档:README.md `## Usage`(无响应 Usage 字段说明)、CLAUDE.md Key Types、CHANGES.md(顶部追加 dated 条目)。

## 方案

1. `Usage` 新增 `ReasoningTokens int json:"reasoning_tokens,omitempty"`。
2. `usageJSON` 新增 `ReasoningTokens int json:"reasoning_tokens,omitempty"` 与 `CompletionTokensDetails *completionTokensDetails json:"completion_tokens_details,omitempty"`;新增 `completionTokensDetails{ ReasoningTokens int json:"reasoning_tokens" }`。
3. `UnmarshalJSON`:`u.ReasoningTokens = raw.ReasoningTokens`;若为 0 且 `raw.CompletionTokensDetails != nil` 则取嵌套值(与 cached_tokens 的回退一致,显式平铺值优先)。
4. `Add`:`u.ReasoningTokens += other.ReasoningTokens`。
5. 常量:新增 `FinishReasonContentFilter = "content_filter"`、`FinishReasonFunctionCall = "function_call"`。
6. 测试:嵌套解析、显式优先、Add 累加各一例。
7. 文档:README 新增响应 Usage 字段说明;CLAUDE.md 补 reasoning 说明;CHANGES.md 顶部追加 2026-06-02 条目。

## Done Contract

- 含 `completion_tokens_details.reasoning_tokens` 的响应体解析后 `Usage.ReasoningTokens` 正确;`Add` 正确累加;显式 `reasoning_tokens` 平铺值优先于嵌套。
- `FinishReasonContentFilter`/`FinishReasonFunctionCall` 存在且取值与官方一致。
- `go test ./...`(aimodel)全绿;新增测试覆盖上述路径。
- 代码、测试、文档(CLAUDE.md/README.md/CHANGES.md)同一改动内完成。
- 未完成判据:任一测试失败、文档未同步、或引入新依赖。

## Change Log / Validation

### 改动(2026-06-02)

- `schema.go`:
  - `Usage` 新增 `ReasoningTokens int json:"reasoning_tokens,omitempty"`。
  - `usageJSON` 新增 `ReasoningTokens` 与 `CompletionTokensDetails *completionTokensDetails`;新增 `completionTokensDetails{ReasoningTokens}`。
  - `UnmarshalJSON`:平铺 `reasoning_tokens` 优先,回退嵌套 `completion_tokens_details.reasoning_tokens`(与 cached_tokens 一致)。
  - `Add`:累加 `ReasoningTokens`。
  - `FinishReason` 常量新增 `FinishReasonContentFilter`("content_filter")、`FinishReasonFunctionCall`("function_call",legacy)。
- `schema_test.go`:新增 4 个测试(嵌套解析 / 显式优先 / Add 累加 / 零值 omitempty)。
- 文档:`README.md`(新增 Response usage + finish_reason 说明)、`CLAUDE.md`(Key Types 补 Usage/FinishReason)、`CHANGES.md`(顶部追加 dated 条目)。

### 验证

- `go test ./`(aimodel):163 passed。
- `gofmt -l schema.go schema_test.go`:无输出(格式合规)。
- 零新增依赖,未触及 Anthropic 翻译。

### 结论

核心目标已由证据证明完成(Done Contract 全部满足)。无遗留下一循环。
