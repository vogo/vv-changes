# Context Editing — 设计

## 1. 范围与思路

实现 `doc/design/session-context-solution.md` §4.3 第一条建议（"折叠超过 N 轮的 tool_result 为占位摘要，保留 tool_use_id 链路"），其余两条（stale 标记、单条转引用）显式留作后续。

实现位置：`vage/largemodel/`，新增 `context_editor.go`。形态：标准 `Middleware`（与 `BudgetMiddleware`、`MetricsMiddleware`、`CacheMiddleware` 同形态）。

策略：**最近 K 条 tool_result 完整保留，更早的 tool_result 内容替换为占位符**（保留 `Role` / `ToolCallID`）。这是 Anthropic *context editing* 在工程上的最简形态，对长任务收益最大、行为可解释、易测试。复杂策略（按 token 阈值 / 工具白名单 / 错误标记）通过 functional option 留扩展位，本期不实现。

## 2. 包级 API

```go
// vage/largemodel/context_editor.go

const (
    defaultKeepLastTools = 5
)

// ContextEditorMiddleware folds older tool_result messages into short
// placeholders before the request reaches the underlying ChatCompleter,
// so multi-iteration ReAct loops do not pay for the full tool_result
// payload on every turn.
//
// Editing is applied to a SHALLOW COPY of *aimodel.ChatRequest. The
// caller's request and its Messages slice are never mutated; modified
// messages are constructed as new aimodel.Message values placed in a
// fresh slice.
//
// The middleware is stateless: each Chat / Stream call is judged
// independently from req.Messages alone.
type ContextEditorMiddleware struct {
    keepLast       int
    minElidedBytes int
    dispatch       DispatchFunc                                 // optional; nil ⇒ silent
    placeholderFn  func(toolCallID string, originalBytes int) string
}

// ContextEditorOption configures ContextEditorMiddleware.
type ContextEditorOption func(*ContextEditorMiddleware)

// WithKeepLastTools sets how many of the most recent tool_result
// messages to keep verbatim. Older tool_result messages have their
// content replaced with a placeholder. n <= 0 falls back to default 5.
func WithKeepLastTools(n int) ContextEditorOption

// WithMinElidedBytes sets the minimum freed-byte budget for a single
// editing pass. If freeing all eligible older tool_results would save
// fewer than n bytes, no editing happens (and no event fires). n <= 0
// disables the threshold (always edit). Default: 0.
func WithMinElidedBytes(n int) ContextEditorOption

// WithContextEditDispatch wires an event sink. When at least one
// tool_result is elided in a request, the middleware dispatches a
// schema.EventContextEdited event. nil dispatch ⇒ silent (no panic).
func WithContextEditDispatch(d DispatchFunc) ContextEditorOption

// WithPlaceholder customises the placeholder text. Default:
//   "[context_edited: tool_result <id> elided, <N> bytes]"
// The function receives the original tool_call_id and the byte length
// of the elided text content.
func WithPlaceholder(fn func(toolCallID string, originalBytes int) string) ContextEditorOption

// NewContextEditorMiddleware constructs a middleware. Both editing and
// event dispatch are enabled by default; pass options to customise.
func NewContextEditorMiddleware(opts ...ContextEditorOption) *ContextEditorMiddleware

// Wrap implements Middleware.
func (m *ContextEditorMiddleware) Wrap(next aimodel.ChatCompleter) aimodel.ChatCompleter
```

### 2.1 schema 事件

```go
// vage/schema/event.go

const EventContextEdited = "context_edited"

// ContextEditedData reports a single Context Editing pass on the
// outgoing ChatRequest. Emitted only when at least one tool_result
// was elided.
type ContextEditedData struct {
    Edited        int    `json:"edited"`         // count of elided tool_results
    Kept          int    `json:"kept"`           // count of intact tool_results
    Total         int    `json:"total"`          // total messages in request
    OriginalBytes int    `json:"original_bytes"` // sum of elided original Content.Text() bytes
    Placeholder   int    `json:"placeholder_bytes,omitempty"` // bytes occupied by placeholders
    Strategy      string `json:"strategy"`       // "keep_last_k"
}

func (ContextEditedData) eventData() {}
```

### 2.2 TaskAgent 集成姿态

新增 option：

```go
// vage/agent/taskagent/task.go

// WithContextEditor wraps the agent's ChatCompleter with a Context
// Editing middleware so multi-iteration ReAct loops automatically
// fold older tool_result messages into short placeholders. Pass nil
// to disable.
//
// Wrapping happens at agent.New time, AFTER WithChatCompleter is
// resolved. If the agent has no chatCompleter set, this option is a
// no-op (the agent itself will fail at first Run as before).
func WithContextEditor(mw *largemodel.ContextEditorMiddleware) Option
```

实现细节：option 不立即包裹 chatCompleter（顺序未定），而是把 mw 存在 agent 上；`New(...)` 在 opts 全部应用后，做一次"如果 mw != nil 且 chatCompleter != nil → `a.chatCompleter = largemodel.Chain(a.chatCompleter, mw)`"的尾部装配。这样 caller 调用顺序无关。

DispatchFunc 由调用方在构造 mw 时直接传入（与 vv 的 `MetricsMiddleware` 接法一致）。TaskAgent 不替 caller 决定 dispatch 来源，避免出现"既看 hookManager 又看 mw 自带 dispatch"的双源歧义。

## 3. 编辑算法

伪代码：

```
Chat(ctx, req):
    eligibleIdx, kept, totalElidedBytes = scan(req.Messages)
        # eligibleIdx[i] = absolute index in Messages where Role==RoleTool
        # split into [older = eligibleIdx[:len-keepLast], keep = eligibleIdx[len-keepLast:]]
        # kept = len(keep)
        # totalElidedBytes = sum of older[i].Content.Text() byte length
    if len(older) == 0:
        return next(req)                     # nothing to do, no copy
    if minElidedBytes > 0 and totalElidedBytes < minElidedBytes:
        return next(req)                     # under threshold, no edit, no event
    edited = build new []aimodel.Message by:
        copy req.Messages
        for each idx in older:
            m = req.Messages[idx]
            placeholder = placeholderFn(m.ToolCallID, len(m.Content.Text()))
            edited[idx] = aimodel.Message{
                Role:            aimodel.RoleTool,
                Content:         aimodel.NewTextContent(placeholder),
                ToolCallID:      m.ToolCallID,
                CacheBreakpoint: m.CacheBreakpoint,
                # Thinking / ToolCalls left zero (tool messages never carry these)
            }
    edReq = *req                                       # shallow copy
    edReq.Messages = edited
    if dispatch != nil:
        dispatch(ctx, EventContextEdited{...})
    return next(&edReq)
```

不变量：

1. `req` 与 `req.Messages` 切片**绝不被原地修改**（只读副本即可——caller 的 ReAct 累积态保持完整）。
2. `assistant` 消息的 `ToolCalls[].ID` 不动；`tool_result` 消息的 `ToolCallID` 不动 —— `tool_use → tool_result` 配对结构永远完整。
3. 编辑顺序与原 messages 顺序一致；只替内容、不删消息。
4. `system` / `user` / `assistant` 消息**完全不动**。
5. 事件**仅在实际编辑发生时**派发；threshold 拦下来不编辑也不派发（与 `GuardCheck` 的 silent pass 一致）。

## 4. 流式行为

`ChatCompletionStream` 与 `ChatCompletion` 共用同一 edit 路径：编辑发生在请求出站前，下游 `ChatCompleter.ChatCompletionStream` 收到的是已编辑副本。流响应不参与编辑（响应是 LLM 输出，不在 Context Editing 关心的范围）。

## 5. 与现有中间件的位置关系

ContextEditor 的"输入"是 caller 已经组装好的 `ChatRequest.Messages`，"输出"是给下游的精简副本。它**应当位于**：

- **`MetricsMiddleware` 之内**（或与之同一层）：让 `EventLLMCallStart.Messages` 反映**编辑后**的实际发送 token 计数；如果放在外面，metrics 里的 messages 数就是编辑前的 = 假账。
- **`CacheMiddleware` 之内**（或与之同一层）：缓存 key 必须基于真实送给 LLM 的 messages，否则同一原始 request 因不同编辑结果命中错误缓存。
- **`BudgetMiddleware` 之内**：budget 关心实际消耗，编辑后再计算更准确（虽然 BudgetMiddleware 用的是 `resp.Usage` 不是 `req.Messages` 的估算，所以放哪都可——为简化"在内层"的统一原则，仍放最内层）。
- **`RetryMiddleware` 内或外都行**：编辑结果是确定性的（无随机），retry 重新走一次 wrap 也得到同样副本——位置不影响正确性，但放内层更省（一次编辑、多次重试）。

**推荐位置**：作为 caller 显式装配的"最内层中间件"，紧贴底层 `aimodel.ChatCompleter`。这与本设计提供的 TaskAgent option 行为一致——`WithContextEditor` 在 `New` 尾部 `largemodel.Chain(a.chatCompleter, mw)`，得到 `mw → a.chatCompleter`，是相对于其他 vv 注入中间件最内的位置。

> **注意**：TaskAgent 把它装在最内层，不影响 vv `setup.wrapLLMClient` 在外面再叠加 Debug / Budget——后者看到的就是"装好 ContextEditor 的 chatCompleter"。这正是希望的语义：所有外层中间件都看到的是编辑后请求。

## 6. Cache breakpoint 的影响

`aimodel.Message.CacheBreakpoint` 是 Anthropic prompt cache 的关键。两点保护：

1. 占位副本**继承**原 message 的 `CacheBreakpoint` 字段——位置不变。
2. 由于 vage 当前的 cache breakpoint 主要在 system message 与最后一个 tool definition 上（见 `WithPromptCaching`），不在 tool_result 上——所以擦除 tool_result 内容**不会击穿现有缓存断点**。
3. 真要避免缓存抖动，最稳妥的策略是 caller 把 ContextEditor 的"keepLast"设大于一次 ReAct 循环内可能新增的 tool_result 数 —— 默认 K=5 已足够覆盖大部分场景。

## 7. 错误处理

中间件**不返回新错误**：

- placeholderFn 由 caller 提供时，用 `recover` 兜住 panic（防止单个错误格式化把整个 LLM 调用打死）；本期不实现，因为默认实现是常量字符串 + `fmt.Sprintf`，可控；caller 自定义出 panic 视为编程错误，让其自然冒出。
- `dispatch != nil` 时直接同步调用，与 `MetricsMiddleware` 行为一致；下游 hook 失败不阻塞主路径已由 hook.Manager 自身保证。

## 8. 可观察性

派发的 `EventContextEdited` 走 `hook.Manager`（如果 caller 把 manager.Dispatch 注入为 DispatchFunc），与 `EventLLMCallStart` / `EventGuardCheck` 同级。tracelog / SessionHook 自动落盘——本设计不在 vv 端默认接线，由用户显式挂入；保持本期范围聚焦。

文档：`vage/.doc/largemodel.md` 在 §3 中间件列表 + §4 流式行为表追加 ContextEditor 段落。

## 9. 测试计划

### 9.1 单元测试 `context_editor_test.go`

- TC-1（折叠老结果）：构造一个 `ChatRequest`，含 7 个 `RoleTool` 消息，K=3 → 前 4 条内容被替换为占位符，后 3 条保留；`ToolCallID` 全部不变。
- TC-2（不动 caller）：构造请求 → 中间件调用后断言 `req.Messages` 完全等于原始切片（同一 slice、同一 message Content）；副本走给下游。
- TC-3（不到阈值不编辑）：tool_result 数 ≤ K → 不编辑、不派发事件。
- TC-4（minElidedBytes 拦截）：可编辑 tool_result 内容总长 < threshold → 不编辑、不派发。
- TC-5（事件 payload）：edited / kept / total / originalBytes / placeholderBytes 字段全部匹配预期。
- TC-6（nil dispatch 不 panic）：DispatchFunc 为 nil + 编辑发生 → 不 panic、下游被调用。
- TC-7（流式路径）：`ChatCompletionStream` 也按同样规则编辑；下游收到的 req.Messages 与非流式一致。
- TC-8（不破坏 system / user / assistant）：mixed messages，断言 `RoleSystem` / `RoleUser` / `RoleAssistant`（含 ToolCalls）原样。
- TC-9（assistant.ToolCalls.ID 保留）：assistant 消息 `ToolCalls[].ID` 与对应 `RoleTool.ToolCallID` 配对完整无破坏。
- TC-10（自定义占位符）：`WithPlaceholder(fn)` 生效；`OriginalBytes` 计算用的是被替换 message 的 `Content.Text()` byte 长度。
- TC-11（CacheBreakpoint 继承）：原消息有 CacheBreakpoint=true → 占位副本同样为 true。

### 9.2 TaskAgent 集成测试 `task_context_editor_test.go`

- TC-A（option 装配）：`WithContextEditor(mw)` + 现有 fake completer，跑两轮 ReAct，断言第二轮 LLM 收到的请求里早期 tool_result 已被占位（即 mw 真的被装到了内层）。
- TC-B（无 option = 现状）：不传 option，行为与现网完全一致（zero-overhead，快速回归保护）。

测试形态延用 `task_test.go` / `task_cache_test.go` 的 fake completer 风格——避免引入新测试基础设施。

### 9.3 lint & 全量测试

- `cd vage && make test` 通过
- `cd vage && make lint` 通过
- `cd vv && make test` 通过（保护 vv 端无回归——vv 不默认启用，本身不应受影响）

## 10. 文件清单

新增：

- `vage/largemodel/context_editor.go`（实现）
- `vage/largemodel/context_editor_test.go`（单测）
- `vage/agent/taskagent/task_context_editor_test.go`（集成测）

修改：

- `vage/schema/event.go` —— 增 `EventContextEdited` + `ContextEditedData`
- `vage/agent/taskagent/task.go` —— 增 `contextEditor` 字段、`WithContextEditor` option、`New` 尾部装配逻辑
- `vage/.doc/largemodel.md` —— 文档段落
- `doc/design/session-context-solution.md` —— §4.3 / §8 / 末尾路线段标注完成
