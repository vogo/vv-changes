# Code Review — Context Editing (P6)

## 范围

复审 dev session `2026-04-30-001-context-editing` 提交：

- **新增**：`vage/largemodel/context_editor.go`、`vage/largemodel/context_editor_test.go`、`vage/agent/taskagent/task_context_editor_test.go`
- **修改**：`vage/schema/event.go`、`vage/agent/taskagent/task.go`

总体评价：实现与 design.md 高度一致，关键不变量（不修改 caller 请求、`tool_use → tool_result` 配对完整、流式与非流式同路径、stateless）均落地。无安全/并发缺陷。下面按严重度记录可改进点，已直接施加的修改在文末列出。

---

## 严重度等级

S = severity（critical / high / medium / low / nit）。

---

## 问题清单

### M1（medium · 已修复）— 报告 `Kept` 时多遍历一次 messages

**File**: `vage/largemodel/context_editor.go:165-172`（修复前）

`edit()` 在 dispatch 事件时调用 `countToolMessages(req.Messages) - len(older)` 重新走一遍切片以拿到 `kept`。但同样的信息在 `scanElidable` 里已经算过：

```
older = toolIdx[:cut]
kept  = m.keepLast            // 因为 len(toolIdx) > keepLast 是进入此分支的前置条件
```

设计文档 §3 也明确 `kept = len(keep)`。第二次遍历多余。

**修复**：让 `scanElidable` 同时返回 `kept`；删除 `countToolMessages` 辅助函数。事件 payload 与之前完全一致（`kept = m.keepLast`），但每次编辑省一次 O(n) 扫描。

### M2（medium · 已修复）— `applyElision` 用 `map[int]struct{}` 做命中查找浪费分配

**File**: `vage/largemodel/context_editor.go:208-212`（修复前）

`older` 从 `scanElidable` 出来本就是升序的（按 `range msgs` 顺序追加）。在 `applyElision` 里再为它构造一个 hash set 用于 O(1) 命中判断，纯粹是浪费一次 map 分配。

**修复**：改为单指针线性合并 —— 维护 `cursor` 在 `older` 上前进，比较 `older[cursor] == i`。代码更短、零额外分配，行为完全等价。已在 `scanElidable` 注释里明确 "older 升序" 不变量，避免后续维护误用。

### L1（low · 已修复）— 测试中的死代码

**File**: `vage/largemodel/context_editor_test.go:166-171`（修复前）

`TestContextEditor_BelowThresholdNoOp` 里有：

```go
mw  := NewContextEditorMiddleware(WithKeepLastTools(5))   // 构造但不用
mw2 := NewContextEditorMiddleware(WithKeepLastTools(5), ...)
_ = mw                                                     // 显式丢弃
```

明显是开发过程残留。

**修复**：删除冗余的 `mw`，统一使用单个 `mw`，名称回归直观。

### L2（low · 不修复，记录在案）— `TestContextEditor_DoesNotMutateCaller` 的"shallow copy"语义

**File**: `vage/largemodel/context_editor_test.go:139-161`

测试用 `make + copy` 浅拷贝原始切片，再用 `reflect.DeepEqual` 对比。这套断言对**元素级原地修改**（mutation of `aimodel.Message` fields）能查出来；但若有人将来把 `aimodel.Message.Content.parts` 改成可变 slice 并原地修改，则 deep copy 都会被同时观测，测试失效。当前实现里 `applyElision` 一律 `make` 新切片并构造全新 `Message{...}` 值，不会触发该死角。**保留现状**，但建议未来若 `Content` 引入可变内部状态，配合改造此测试。

### L3（low · 不修复，记录在案）— Anthropic 协议下空占位符的边界

**File**: `vage/largemodel/context_editor.go:104-110` (`WithPlaceholder`)

caller 自定义的 `PlaceholderFunc` 返回空字符串时，下游 `aimodel.Message.Content` 会是空文本。Anthropic translator 在 `Role==RoleTool` 时会包成 `tool_result` block；空 text 是否合法由 Anthropic 后端定。本期默认占位符是非空字符串，且需求/设计文档没有强约束，**保持现状**。如未来出现报错，可在 `applyElision` 里对空占位符兜底为 `"[elided]"`。

### L4（nit · 不修复）— `WithPlaceholder(nil)` 静默忽略

**File**: `vage/largemodel/context_editor.go:104-110`

传 nil 不报错也不 panic，沿用默认占位符。这是合理的 functional option 行为，但 `WithPlaceholder` 的 doc comment 没明确说明 "nil → 沿用默认"。**保持现状**，与同包其他 `With*` 选项约定一致。

### L5（nit · 不修复）— `WithMinElidedBytes` 文档与代码一致性

`WithMinElidedBytes(n)` 文档说 "n <= 0 disables the threshold (always edit)"。实现是 `if n < 0 { n = 0 }`，然后在 `edit()` 用 `if m.minElidedBytes > 0`。两者等价（n=0 也走非 > 0 分支），但 `n <= 0` 与 `n < 0` 描述上略有出入。**保持现状**，行为正确。

---

## 不变量复核（Invariant Check）

| 不变量 | 来源 | 验证方式 | 结论 |
|---|---|---|---|
| caller 的 `req` / `req.Messages` 不被原地修改 | requirement AC-1.3 / design §3 | `applyElision` 用 `make([]aimodel.Message, len(msgs))` 构造新切片，元素是新 `aimodel.Message{...}` 字面量；TC-2 显式断言 deep equality | ✅ |
| `assistant.ToolCalls[].ID` / `tool_result.ToolCallID` 完整保留 | design §3 不变量 #2 | TC-1 / TC-8 / 集成测 `TestAgent_WithContextEditor_FoldsOldToolResults` 末段验证 | ✅ |
| 编辑顺序与原 messages 顺序一致；只替内容、不删消息 | design §3 不变量 #3 | `applyElision` 输出长度 == 输入长度，cursor 单向前进 | ✅ |
| system / user / assistant 完全不动 | design §3 不变量 #4 | TC-8 + 单指针线性合并跳过非 `older` 索引时直接 `out[i] = msgs[i]`（值拷贝） | ✅ |
| 事件仅在实际编辑发生时派发 | design §3 不变量 #5 | TC-3 / TC-4 显式断言 `dispatched == 0` | ✅ |
| 流式与非流式同 edit 路径 | design §4 / requirement AC-1.5 | `Wrap()` 的 `chat` / `stream` 都调用 `m.edit(ctx, req)` | ✅ |
| middleware 无状态 | requirement AC-2.5 | 字段全部只在 `New*` 时写入；`Wrap` / `edit` 只读 | ✅ |
| nil dispatch 静默不 panic | requirement AC-2.3 | TC-6 显式断言 | ✅ |
| `kept = total tools − elided` 报告不变量 | session 任务说明 | 修复后 `kept = m.keepLast`，`edited = len(older)`，`total tools = kept + edited` —— 由 `scanElidable` 在 `len(toolIdx) > keepLast` 分支直接给出 | ✅ |

---

## 修复后的验证

```
cd /Users/hk/workspaces/github/vogo/vagents/vage
go test ./largemodel/ ./agent/taskagent/ ./schema/ -count=1
# ok largemodel / agent/taskagent / schema —— 全绿

go test ./... -count=1
# 全部 47 个包通过

make lint
# 0 issues
```

---

## 已应用的改动文件

- `vage/largemodel/context_editor.go` —— `scanElidable` 多返回 `kept`；`applyElision` 用单指针线性合并替代 map；删除 `countToolMessages`。
- `vage/largemodel/context_editor_test.go` —— `TestContextEditor_BelowThresholdNoOp` 清理死代码。

无外部行为变化、无 API 变化、无事件 payload 变化。
