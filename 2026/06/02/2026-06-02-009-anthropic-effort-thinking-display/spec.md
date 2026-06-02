# Anthropic:映射 effort、支持 thinking.display,标注 budget_tokens 弃用

- 日期:2026-06-02
- 模块:`aimodel`(Anthropic Messages API 薄封装)
- 深度:standard

## 背景

Anthropic Messages API 两项官方变更:

1. **2026-02-05**:顶层 `effort` 参数 GA,取代 `thinking.budget_tokens` 控制思考深度。
2. **2026-03-16**:`thinking.display:"omitted"` 可省略思考内容,加速流式输出。

当前 `aimodel`:
- `toAnthropicRequest`(`anthropic.go`)仅透传 `req.Thinking`,未把 canonical 的 `ReasoningEffort` 映射到 Anthropic 顶层 `effort`。
- `Thinking` 结构(`schema.go`)只有 `Type` / `BudgetTokens`,缺 `display`。

## 目标

- `ChatRequest.ReasoningEffort` → Anthropic 顶层 `effort`(透传字符串,沿用 `ReasoningEffort*` 常量)。
- `Thinking` 增加 `Display string`(`json:"display,omitempty"`),支持 `"omitted"`;`adaptive` 类型确认可正常透传(`Type` 已是自由字符串)。
- `Thinking.BudgetTokens` 增加弃用注释(新模型推荐 effort/adaptive)。

## 边界 / 非目标

- 不改 OpenAI 侧映射;`ReasoningEffort` 仍是 canonical 字段,OpenAI 走 `reasoning_effort`、Anthropic 走 `effort`。
- 不做 effort 取值校验(沿用透传约定)。
- 不改流式解析逻辑(`display:omitted` 只是少了 thinking 内容,现有解析已可安全跳过)。

## 改动点

1. `schema.go` — `Thinking` 增加 `Display string`(omitempty);`BudgetTokens` 加弃用注释;补充 `effort` 与 anthropic `effort` 的关系说明。
2. `anthropic.go` — `anthropicRequest` 增加 `Effort string \`json:"effort,omitempty"\``;`toAnthropicRequest` 末尾把 `req.ReasoningEffort` 赋给 `ar.Effort`(为空时 omitempty 不输出)。
3. `anthropic_test.go` — 新增测试覆盖:effort 透传、`thinking.display=omitted`、`adaptive` 类型;effort 为空时不输出。
4. 文档同步:`CHANGES.md`(新增条目)、`CLAUDE.md`、`README.md`。

## Done Contract

- **算完成**:`anthropicRequest` 有 `Effort`(omitempty);`Thinking` 有 `Display`(omitempty)且 `BudgetTokens` 有弃用注释;`go test ./...` 全绿;三份文档已同步。
- **证据**:`go test -run TestToAnthropicRequest ./...` 通过 + 新增子测试通过;`make build` 通过。
- **未完成**:任一文档未同步、或 ReasoningEffort 空值仍序列化出 `effort` 字段。

## Spec Self-Check

- 完整性:涵盖代码 + 测试 + 三份文档。✔
- 一致性:与既有 `ReasoningEffort` 透传约定、Anthropic 私有类型翻译模式一致。✔
- 可测性:effort 透传 / 空值省略 / display=omitted / adaptive 均可断言。✔
- 无歧义:字段名、json tag、omitempty 行为明确。✔
- 可行性:纯结构体字段 + 一行赋值,低风险。✔

## Change Log

- `schema.go`:`Thinking` 增 `Display string`(omitempty);`BudgetTokens` 加弃用注释;`Type` 注释补充 `adaptive`;`ChatRequest.ReasoningEffort` 注释补充 Anthropic `effort` 映射关系。
- `anthropic.go`:`anthropicRequest` 增 `Effort string \`json:"effort,omitempty"\``;`toAnthropicRequest` 末尾 `ar.Effort = req.ReasoningEffort`。
- `anthropic_test.go`:新增 `TestToAnthropicRequestEffort`、`TestToAnthropicRequestEffortOmittedWhenEmpty`、`TestToAnthropicRequestThinkingDisplayOmitted`、`TestToAnthropicRequestThinkingAdaptive`。
- 文档:`CHANGES.md` 新增条目;`CLAUDE.md` Anthropic Translation 段补充 effort/display;`README.md` Reasoning 段补充 Anthropic effort 与 Thinking.Display。

## Validation

- `go test ./...` → 238 passed in 7 packages(含 4 个新增子测试)。
- `make build` → EXIT=0(license-check → format → lint → test 全过),无遗留二进制。

## 结论

核心目标已由证据证明完成:effort 透传、空值省略、thinking.display=omitted、adaptive 类型均覆盖并通过;三份文档已同步。无后续 loop。
