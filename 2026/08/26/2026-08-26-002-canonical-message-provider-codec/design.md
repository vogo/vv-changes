# canonical Message 与 provider codec 收敛

## 背景

`schema.Message` 直接公开并持有 OpenAI/Anthropic native wire,导致契约层依赖 `aimodel` provider 类型。消息读取、修改和 wire 解析混在 `schema`,编码则散落在 `largemodel`,provider 映射职责与状态所有权都不清晰。

## 设计

- `schema.Message` 将 `protocol`、`role`、`parts`、`origin` 封装为私有状态。
- canonical `role + parts` 是唯一事实源;访问器返回值或深拷贝。
- `origin` 仅表示未经修改、可直接回放的 provider-native payload。
- `SetText`、`SetRole`、`ReplaceParts`、`AppendPart` mutation 统一清空 `origin`。
- JSON persistence 通过自定义 marshal/unmarshal 同时保存 canonical 状态和有效的 `origin`。
- provider-native 双向转换分别迁入对应 provider 包:
  - `vage/largemodel/provider/openais/message_codec.go`
  - `vage/largemodel/provider/anthropics/message_codec.go`
- provider encoder 在 native replay 或 canonical 编码前都执行消息与 part 校验;未知、字段混用或结构非法的 part 返回错误而非静默丢弃。
- Anthropic tool-result 合并逻辑迁入 provider adapter。

## 协议修正

`aimodel/anthropic.ContentBlock` 增加 `IsError`,vage canonical→Anthropic wire 映射完整传递 `tool_result.is_error`。

## 测试

- canonical mutation 使 `origin` 失效。
- `Parts()` 返回深拷贝。
- OpenAI 未修改消息复用 native payload,修改后从 canonical 重新编码。
- Anthropic `is_error=true` codec round-trip 与端到端 request wire。
- 非法 canonical part 被 provider encoder 拒绝。
