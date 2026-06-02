# aimodel/OpenAI:支持 max_completion_tokens 并标注 max_tokens 弃用

- 日期:2026-06-02
- 任务深度:standard
- 状态:已完成

## 核心目标(Loop Anchor)

在 `ChatRequest` 新增 `MaxCompletionTokens` 字段以支持 OpenAI o 系列 / GPT-5.x 等推理模型(它们拒绝 `max_tokens`),同时保留并标注 `MaxTokens` 已弃用;保证 Anthropic 协议转换在仅提供新字段时仍能正确折算出 `max_tokens`。代码、测试、文档一并完成。

## 背景事实

- OpenAI 官方已将 Chat Completions 的 `max_tokens` 标记为 deprecated,推理模型必须改用 `max_completion_tokens`(上限同时涵盖可见输出 token 与内部 reasoning token)。
- `aimodel/schema.go:58` 的 `ChatRequest` 当前仅有 `MaxTokens *int`(`json:"max_tokens,omitempty"`),无 `max_completion_tokens`,无法调用上述模型。
- OpenAI 路径:`ChatRequest` 即规范请求体,直接序列化发送(见 `openai_chat.go`),新增字段加 omitempty 即可生效。
- Anthropic 路径:`anthropic.go:179 toAnthropicRequest` 当前 `if req.MaxTokens != nil { ar.MaxTokens = *req.MaxTokens } else { ar.MaxTokens = anthropicDefaultMaxTokens }`。Anthropic 侧字段仍为 `max_tokens`(必填,默认 4096)。
- `clone()` 在 `schema.go:264`,值拷贝即可覆盖指针字段(指针本身拷贝,语义与现有 `MaxTokens` 一致)。

## 范围 / 边界

- 改动文件:`aimodel/schema.go`、`aimodel/anthropic.go`、`aimodel/openai_chat_test.go`、`aimodel/anthropic_test.go`、`aimodel/CLAUDE.md`、`aimodel/README.md`、`aimodel/CHANGES.md`。
- 不改协议分发逻辑、不改其它字段语义。

## 设计决策

1. `ChatRequest` 新增 `MaxCompletionTokens *int` `json:"max_completion_tokens,omitempty"`;`MaxTokens` 注释标注「已弃用,推理模型(o 系列 / GPT-5.x)会拒绝,改用 MaxCompletionTokens;仅保留兼容旧/非推理模型」。
2. `clone()`:值拷贝即覆盖两个指针字段,无需额外处理(与现有指针字段一致);在 spec 中说明,代码无需新增分支。
3. Anthropic 折算优先级:`MaxCompletionTokens` 优先于 `MaxTokens`(新字段为首选),两者皆空时用 `anthropicDefaultMaxTokens`。逻辑:
   `if req.MaxCompletionTokens != nil { use it } else if req.MaxTokens != nil { use it } else { default }`。

## 计划

1. `schema.go`:新增 `MaxCompletionTokens` 字段并补注释;`MaxTokens` 加弃用注释。
2. `anthropic.go`:`toAnthropicRequest` 改为按上述优先级折算 `max_tokens`。
3. `openai_chat_test.go`:新增序列化测试,覆盖 仅设 MaxCompletionTokens / 仅设 MaxTokens / 两者皆空 三种请求体的 JSON 序列化结果。
4. `anthropic_test.go`:新增「仅提供 MaxCompletionTokens」→ `ar.MaxTokens` 正确;补「两者皆设时 MaxCompletionTokens 优先」用例。
5. 文档:CLAUDE.md / README.md 补字段说明;CHANGES.md 增加 2026-06-02 条目(官方协议变更 + 受影响文件)。
6. `go test ./...` 验证;按 aimodel 规则删除构建产物。

## Done Contract

- 完成:(1) `ChatRequest` 含 `MaxCompletionTokens`,序列化为 `max_completion_tokens` 且 omitempty 生效,`MaxTokens` 有弃用注释;(2) openai 测试覆盖三种序列化场景且通过;(3) Anthropic 仅给 `MaxCompletionTokens` 时正确生成 `max_tokens`,测试通过;(4) CLAUDE.md / README.md / CHANGES.md 同步更新。
- 证据:`go test ./...` 全绿;序列化测试断言字段名与 omitempty;文档三处一致。
- 未完成:任一测试失败、字段名/omitempty 不符、或文档缺更新。

## Change Log / Validation(执行后回填)

### Change Log
- `aimodel/schema.go`:`ChatRequest` 新增 `MaxCompletionTokens *int`(`json:"max_completion_tokens,omitempty"`);`MaxTokens` 补 `Deprecated:` 注释说明推理模型拒绝、仅兼容旧/非推理模型。`clone()` 值拷贝即覆盖两指针,无需新增分支。
- `aimodel/anthropic.go`:`toAnthropicRequest` 折算改为 `switch`,优先级 `MaxCompletionTokens > MaxTokens > anthropicDefaultMaxTokens(4096)`,Anthropic 侧仍输出 `max_tokens`。
- `aimodel/openai_chat_test.go`:新增 `TestChatRequestMaxTokensSerialization`,表驱动覆盖 仅 MaxCompletionTokens / 仅 MaxTokens / 两者皆空 三种序列化(断言字段名 + omitempty)。
- `aimodel/anthropic_test.go`:新增 `TestToAnthropicRequestMaxCompletionTokens`(仅新字段→max_tokens 正确)与 `TestToAnthropicRequestMaxCompletionTokensPreferred`(两者皆设时新字段优先)。
- `aimodel/CLAUDE.md`、`aimodel/README.md`:补 token 上限字段说明(含推理模型示例 `ModelOpenaiO3` + `*int` 指针写法)。
- `aimodel/CHANGES.md`:新增 2026-06-02 条目(官方协议变更 + 受影响文件)。

### Validation
- `go vet ./...`:无问题;`go test ./...`:200 passed in 7 packages;`gofmt -l` 四文件无输出(已格式化);构建二进制已删除。
- README 示例校验:`ModelOpenaiO3` 常量存在;代码库无 `Int` 辅助函数,改用 `&maxCompletionTokens` 指针惯用法。

### 结论
核心目标已由证据(测试全绿 + 序列化断言)达成。四项验收标准全部满足。无后续 loop。
