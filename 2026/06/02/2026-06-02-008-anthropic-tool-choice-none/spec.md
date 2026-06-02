# 2026-06-02-008 Anthropic:tool_choice "none" 映射与 disable_parallel_tool_use 支持

## 背景

`aimodel/anthropic.go` 的 `convertToolChoice`(465-485 行)与 tool_choice 装配(305-308 行)存在两处语义缺口:

1. **"none" 被省略**:收到 `"none"` 时返回 `nil`,导致请求体省略 `tool_choice` 字段。官方自 2025-02-27 起支持 `tool_choice:{type:"none"}`(显式禁止调用任何工具),二者语义不同:**省略 = 模型自选(等价 auto);none = 禁止调用**。
2. **disable_parallel_tool_use 未映射**:canonical 的 `ChatRequest.ParallelToolCalls *bool`(schema.go:143)显式为 `false` 时,未映射为 Anthropic 的 `tool_choice.disable_parallel_tool_use:true`(官方 2024-10-03 起支持)。

## 目标

- `"none"` 映射为 `{type:"none"}`(不再省略字段)。
- `ParallelToolCalls` 显式为 `false` 时,在 `tool_choice` 上输出 `disable_parallel_tool_use:true`,需要时与 `auto`/`any`/`tool` 类型组合。

## 关键事实

- `tool_choice` 仅在 `req.ToolChoice != nil` 时被装配(anthropic.go:305-308);而 `ParallelToolCalls` 是独立字段,可在未设 `ToolChoice` 时单独给出。
- `disable_parallel_tool_use` 寄存于 `tool_choice` 内,且只对 `auto`/`any`/`tool` 有意义;对 `none`(不调用工具)无意义,不应附加。
- 官方:`tool_choice` 仅在请求带 `tools` 时有效;无工具的请求注入 `tool_choice` 会被拒。

## 边界

- 仅改 Anthropic 译码路径(`anthropic.go`);不动 OpenAI 路径与 canonical `schema.go`。
- 不改既有 `auto`/`required(any)`/具体工具 三种映射结果。
- `none` 不附加 `disable_parallel_tool_use`(即便 `ParallelToolCalls=false`)。
- 未设 `ToolChoice` 且 `ParallelToolCalls=false` 时,仅在**有工具**(`len(req.Tools) > 0`)时合成 `{type:"auto"}` 承载该标志,避免产出无工具的非法 `tool_choice`。

## 方案

- **结构体** `anthropicToolChoice`(anthropic.go:99-102)新增:
  `DisableParallelToolUse *bool `json:"disable_parallel_tool_use,omitempty"``
- **convertToolChoice**:`"none"` 分支由 `return nil` 改为 `return &anthropicToolChoice{Type: "none"}`;其余分支不变;非 string/非具名工具仍返回 nil。
- **装配逻辑**(anthropic.go:305-308)改为始终经 `convertToolChoice(req.ToolChoice)` 求值(nil 输入安全返回 nil),再折叠 `ParallelToolCalls`:
  ```go
  tc := convertToolChoice(req.ToolChoice)
  if req.ParallelToolCalls != nil && !*req.ParallelToolCalls {
      if tc == nil && len(req.Tools) > 0 {
          tc = &anthropicToolChoice{Type: "auto"} // 合成承载标志
      }
      if tc != nil && tc.Type != "none" {
          disable := true
          tc.DisableParallelToolUse = &disable
      }
  }
  ar.ToolChoice = tc
  ```
- **测试**(`anthropic_test.go`):
  - `TestToAnthropicRequestToolChoice`:`"none"` 用例由 `wantNil` 改为断言 `{type:"none"}`;`auto`/`required(any)`/具体工具 三例保留。
  - 新增 `TestToAnthropicRequestParallelToolCalls`:`false`(带工具,未设 ToolChoice)→ `{type:"auto",disable_parallel_tool_use:true}`;`false` + 显式 `auto`/具体工具 → 对应 type + 标志;`false` + `none` → `{type:"none"}` 无标志;`true` → 无标志;未设 → 无标志(nil 或不含标志)。
- **文档同步**:`CHANGES.md` 新增条目(官方协议/版本/变更摘要);`CLAUDE.md`、`README.md` 增补 Anthropic tool_choice 映射说明("none"→`{type:"none"}`、`ParallelToolCalls=false`→`disable_parallel_tool_use`)。

## Done Contract

- 完成:`anthropicToolChoice` 含 `DisableParallelToolUse *bool`(omitempty);`"none"`→`{type:"none"}`;`ParallelToolCalls=false` 在 auto/any/tool 上输出 `disable_parallel_tool_use:true`,对 none 不输出;四种 tool_choice 映射 + 三种 ParallelToolCalls 情形单测通过;CHANGES.md/CLAUDE.md/README.md 同步。
- 证据:`go test ./...`(含新增/改写测试)通过;既有 tool_choice / tools 用例不回归;无 `.test` 二进制残留。
- 未完成:任一断言失败、既有用例回归,或文档未同步。

## Change Log / Validation

(执行后回填)
