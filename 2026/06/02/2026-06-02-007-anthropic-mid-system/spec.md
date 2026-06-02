# 2026-06-02-007 Anthropic:保留非首位 system 消息,修复中途 system 被上提

## 背景

官方 Anthropic Messages API 自 2026-05-28 起(Opus 4.8)支持在 `messages` 数组的**非首位**发送 `role:"system"` 消息(mid-conversation system messages),用于在长会话中途变更指令且保留 prompt 缓存命中。

当前 `aimodel/anthropic.go` 的 `toAnthropicRequest`(约 216-251 行)遍历 `req.Messages` 时,只要 `m.Role == RoleSystem` 就并入顶层 `system` 字段并 `continue`,**不分位置**。结果:会话中途的 system 消息被错误上提到开头,丢失原有位置语义,且破坏中途变更指令的缓存意图。

## 目标

- 仅提取消息序列**开头**(首条非 system 消息之前)的连续 system 消息,合并进顶层 `system` 字段(含 CacheBreakpoint 行为不变)。
- 出现在中途的 system 消息按原位置内联保留为 `role:"system"` 的 Anthropic 消息,顺序不变。

## 关键事实

- `RoleSystem = "system"`(schema.go:31),`toAnthropicMessage` 以 `am.Role = string(m.Role)` 设角色,对 system 消息天然产出 `role:"system"`,且已覆盖 parts / CacheBreakpoint / 纯文本三种内容形态。
- 因此中途 system 消息**无需新增分支**:只要主循环在"已见过非 system 消息后"不再走提取逻辑,直接交给 `toAnthropicMessage` 即可。

## 边界

- 不改 `toAnthropicMessage`;不改顶层 `system` 字段的 block/string 选择与 CacheBreakpoint 逻辑。
- 仅改"是否提取"的判定:开头连续 system → 提取;其后任何 system → 内联。
- 仅 Anthropic 译码路径;OpenAI 路径不涉及。

## 方案

- `anthropic.go`:在主循环引入 `seenNonSystem bool`。
  - `m.Role == RoleSystem && !seenNonSystem`:走现有提取逻辑(systemTexts/systemBlocks)并 `continue`。
  - 否则:非 system 时置 `seenNonSystem = true`;统一交 `toAnthropicMessage` 追加到 `ar.Messages`(中途 system 即在此内联为 `role:"system"`)。
- `anthropic_test.go`:新增/更新三场景单测,断言请求体结构:
  1. 全部 system 在开头 → 全进顶层 `system`,`ar.Messages` 不含 system。
  2. system 夹在 user/assistant 之间 → 顶层 `system` 仅含开头部分;中途 system 以 `role:"system"` 按原序留在 `ar.Messages`。
  3. 混合(开头有 + 中途有)→ 开头进顶层 `system`,中途内联,顺序与角色正确。
- 文档同步:`CHANGES.md` 新增条目(日期/官方协议与变更/改动摘要);`CLAUDE.md`、`README.md` 的 Anthropic 翻译说明更新为"仅开头 system 提取到 `system` 字段,中途 system 内联保留"。

## Done Contract

- 完成:开头 system 仍合并进顶层 `system`(含 CacheBreakpoint 不变);中途 system 按原序以 `role:"system"` 保留在 messages 不被上提;三场景单测通过;CHANGES.md/CLAUDE.md/README.md 同改动内同步。
- 证据:`go test ./...`(含新增测试)通过;既有 system / cache 用例不回归;无 `.test` 二进制残留。
- 未完成:任一场景断言失败,或现有开头 system / cache 用例回归,或文档未同步。

## Change Log / Validation

### 变更

- `anthropic.go`:`toAnthropicRequest` 主循环新增 `seenNonSystem bool`。提取条件由 `m.Role == RoleSystem` 收紧为 `m.Role == RoleSystem && !seenNonSystem`;遇到非 system 消息置 `seenNonSystem = true`。中途 system 不再 `continue`,直接经 `toAnthropicMessage` 内联为 `role:"system"`(其 `am.Role = string(m.Role)`,parts/CacheBreakpoint/纯文本三形态均复用既有路径)。顶层 `system` 的 block/string 选择与 CacheBreakpoint 逻辑未动。
- `anthropic_test.go`:新增 `TestToAnthropicRequestSystemMidConversation`(user/assistant 后的 system 不上提,`ar.System==nil`,4 条消息按 `user/assistant/system/user` 原序内联)、`TestToAnthropicRequestSystemLeadingAndMid`(开头 system 进 `system` 字段、中途 system 以 `role:"system"` 留在 `user/system/assistant` 中)。场景「全部 system 在开头」仍由既有 `TestToAnthropicRequestSystemExtraction` 覆盖。
- 文档:`CHANGES.md` 新增同步条目(2026-05-28 Opus 4.8 mid-conversation system,官方协议/变更/改动摘要);`CLAUDE.md` Anthropic Translation 段、`README.md` Anthropic Protocol 段更新为「仅开头连续 system 提取到 `system`,中途 system 内联保留」。

### 验证

- `go test ./...` → 226 passed in 7 packages(新增两测试通过,既有 system / cache 用例无回归)。
- `gofmt -l` 无输出、`go vet ./...` 无问题、无 `.test` 二进制残留。

### 结论

核心目标已由测试证据证明完成:开头连续 system 仍合并进顶层 `system`(CacheBreakpoint 不变);中途 system 按原序以 `role:"system"` 保留在 messages、不再被上提;三场景单测通过;CHANGES.md/CLAUDE.md/README.md 与代码同改动内同步。无遗留下一循环。
