# Anthropic:补全新 stop_reason 常量并解析 stop_details 拒绝分类

- 日期:2026-06-02
- 模块:`aimodel`(Anthropic Messages API 薄封装)
- 深度:standard

## 背景

Anthropic Messages API 官方变更:

1. **2025-09-29**:新增 `stop_reason: "model_context_window_exceeded"`(输入+输出超出模型上下文窗口,区别于命中请求的 `max_tokens`)。
2. 服务端工具:`stop_reason: "pause_turn"`(长耗时回合被暂停,客户端可回传以继续)。
3. **2026-05-28(Opus 4.8)**:新增 `stop_reason: "refusal"`(流式分类器拦截潜在违规),并在响应/流式 `message_delta` 返回 `stop_details` 携带拒绝分类。

`stop_details` 结构(仅 `refusal` 时出现):

```json
"stop_details": { "type": "refusal", "category": "cyber"|"bio"|null, "explanation": "string"|null }
```

当前 `aimodel`:
- `mapAnthropicStopReason`(`anthropic.go`)仅显式映射 `end_turn`/`stop_sequence`→`stop`、`max_tokens`→`length`、`tool_use`→`tool_calls`,其余走 `FinishReason(reason)` 原样透传。
- `anthropicResponse` / `anthropicMessageDeltaData` 均未解析 `stop_details`,canonical 响应无字段暴露拒绝分类。

## 目标

- 为新 stop_reason 增加**可读命名常量**并在 `mapAnthropicStopReason` 显式映射:`model_context_window_exceeded`、`refusal`、`pause_turn`。
- 解析响应与流式 `message_delta` 的 `stop_details`,在 canonical `Choice` / `StreamChunkChoice` 上以 `StopDetails` 字段暴露拒绝分类(`omitempty`,向后兼容)。

## 关键设计决策

- **保持透传 + 补常量**,而非折叠成 `content_filter`/`length`。理由:① 当前 `refusal`/`model_context_window_exceeded` 已原样透传为 `FinishReason(raw)`,折叠会**改变现有行为**且丢失 Anthropic 语义;② 纯增量、向后兼容;③ `model_context_window_exceeded` 与 `max_tokens`(`length`)语义不同,不应混淆。新常量取值即官方原始字符串。
- `StopDetails` 定义为 canonical 类型(`schema.go`),Anthropic 私有类型直接以它作为 `stop_details` 的反序列化目标(字段完全一致,无需额外翻译结构)。

## 边界 / 非目标

- 不改 OpenAI 侧;`FinishReason` 仍以 OpenAI 取值为 canonical,新增的三个常量是 Anthropic 透传扩展(沿用既有默认透传约定,只是显式命名)。
- 不实现 `pause_turn` 的自动续跑逻辑(SDK 是薄封装,仅暴露原因供上层决策)。
- 不对 `category`/`explanation` 做枚举校验(透传字符串,官方声明 explanation 不保证跨版本稳定)。

## 改动点

1. `schema.go`
   - 新增 `FinishReason` 常量:`FinishReasonModelContextWindowExceeded`、`FinishReasonRefusal`、`FinishReasonPauseTurn`(取值=官方原始字符串),附注释说明是 Anthropic 扩展、非 OpenAI canonical。
   - 新增 `StopDetails` 结构:`Type` / `Category` / `Explanation`(均 `string`,`omitempty`)。
   - `Choice` 增 `StopDetails *StopDetails \`json:"stop_details,omitempty"\``。
   - `StreamChunkChoice` 增 `StopDetails *StopDetails \`json:"stop_details,omitempty"\``。
2. `anthropic.go`
   - `anthropicResponse` 增 `StopDetails *StopDetails \`json:"stop_details"\``。
   - `anthropicMessageDeltaData` 增 `StopDetails *StopDetails \`json:"stop_details,omitempty"\``。
   - `mapAnthropicStopReason` 增三个显式 case 返回新常量。
   - `fromAnthropicResponse` 把 `ar.StopDetails` 写入 `Choice.StopDetails`。
3. `anthropic_stream.go` — `message_delta` 分支把 `md.Delta.StopDetails` 写入 `StreamChunkChoice.StopDetails`。
4. 测试
   - `anthropic_test.go` — `TestMapAnthropicStopReason` 增三个 case;新增 `refusal` + `stop_details` 的 `fromAnthropicResponse` 断言。
   - `anthropic_stream_test.go` — 新增含 `stop_details` 的 `message_delta` chunk 断言(FinishReason=`refusal`、StopDetails 已填充)。
5. 文档同步:`CHANGES.md`(新增条目)、`CLAUDE.md`(Anthropic Translation + FinishReason 说明)、`README.md`(finish_reason 段)。

## Done Contract

- **算完成**:三个新 `FinishReason` 常量存在且 `mapAnthropicStopReason` 显式映射;`StopDetails` 在 `Choice` / `StreamChunkChoice` 暴露并由响应/流式正确填充;新增单测覆盖各新 stop_reason 与含 stop_details 的响应/流式 chunk;`make build` 全绿;三份文档同步。
- **证据**:`go test ./...` 通过(含新增子测试);`make build` EXIT=0 且无遗留二进制。
- **未完成**:任一文档未同步、`stop_details` 未解析、或新 stop_reason 仍只走默认透传未命名。

## Spec Self-Check

- 完整性:代码 + 测试(响应+流式)+ 三份文档全覆盖。✔
- 一致性:沿用 Anthropic 私有类型/默认透传约定;新常量取值即原始字符串,不破坏 OpenAI canonical 契约。✔
- 可测性:各 stop_reason 映射、refusal+stop_details 响应、流式 chunk 均可断言。✔
- 无歧义:字段名、json tag、omitempty、映射目标明确。✔
- 可行性:结构体字段 + map case + 赋值,低风险纯增量。✔

## Change Log

- `schema.go`:新增 `FinishReasonModelContextWindowExceeded` / `FinishReasonRefusal` / `FinishReasonPauseTurn`(取值=官方原始字符串,附 Anthropic 扩展说明);新增 `StopDetails{Type,Category,Explanation string}`(均 omitempty);`Choice` 与 `StreamChunkChoice` 各增 `StopDetails *StopDetails`(`stop_details,omitempty`)。
- `anthropic.go`:`anthropicResponse` 增 `StopDetails *StopDetails`(`stop_details`);`anthropicMessageDeltaData` 增 `StopDetails *StopDetails`(`stop_details,omitempty`);`mapAnthropicStopReason` 增三个显式 case;`fromAnthropicResponse` 写入 `Choice.StopDetails = ar.StopDetails`。
- `anthropic_stream.go`:`message_delta` 分支 `StopDetails: md.Delta.StopDetails`。
- `anthropic_test.go`:`TestMapAnthropicStopReason` 增三 case;新增 `TestFromAnthropicResponseRefusalStopDetails`、`TestFromAnthropicResponseNoStopDetails`。
- `anthropic_stream_test.go`:新增 `TestAnthropicStreamRefusalStopDetails`。
- 文档:`CHANGES.md` 新增条目;`CLAUDE.md` Key Types(FinishReason + StopDetails)与 Anthropic Translation 段补充;`README.md` finish_reason 段补充新 stop_reason 与 StopDetails。

## Validation

- `go test ./...` → 244 passed in 7 packages(含 4 个新增/扩展子测试)。
- `make build` → EXIT=0(license-check → format → lint → test 全过),无遗留二进制。

## 结论

核心目标已由证据证明完成:三个新 stop_reason 已命名常量化并显式映射;`stop_details` 在响应与流式 `message_delta` 均解析并暴露到 canonical `StopDetails`;新增单测覆盖各新 stop_reason 与含 stop_details 的响应/流式 chunk;三份文档已同步。纯增量、向后兼容,无后续 loop。
