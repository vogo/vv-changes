# 设计：durable HITL（vage interrupt 接入 vv）

> 日期：2026-10-05
> 对应需求：同目录 `requirement.md`

## 1. vage 缺口（通用，不注入 vv 概念）

### 1.1 `Decision.Execute`：批准后跑 handler

现有 pending call 把决策 Content 注入为工具结果、不跑 handler，匹配 `ask_user`。危险 bash 需要「人批准 → 执行原调用」。

在 `interrupt.Decision` 增加 `Execute bool`（omitempty）。Resume 时：

| Execute | IsError | 行为 |
|---------|---------|------|
| false | false | 注入 `Content` 为成功结果（既有 ask_user 路径） |
| false | true | 注入错误结果（拒绝） |
| true | false | **跑原 tool handler**（批准） |
| true | true | 视为拒绝（IsError 优先），不执行 |

幂等比较纳入 `Execute`。不改 `largemodel/schema.InterruptDecision`：HTTP 决策走 `Store.SubmitDecisions`，resume 带空 Decisions。JSON 向后兼容，不升 `CurrentVersion`。

### 1.2 流式发出 `interrupt_created`

`maybeInterrupt` 持久化后由 `reactMode.emitInterruptCreated` 投递：stream 走 `send`（`buildSend` 同时进 hook），sync 只 `dispatch`。SSE 因此能拿到 `interrupt_id` + pending ids，不新增 EventType、不占用 `pending_interaction`。

## 2. vv 策略与装配

- `dispatches.NewDangerousBashPolicy(classifier, guardian)`：仅 `bash` 且合并分类为 `TierDangerous` 时返回 call ID。Blocked / Caution / 非 bash 不拦。
- 分类与 permission 共用 `configs.ClassifyBashArgs`（JSON `command` + classifier/guardian 取最高档）。
- `agents.interrupt_enabled` 默认 false。开启时：
  - 要求 `session.enabled` 且 `bash_rules` 未关闭（否则启动失败）。
  - `interrupt.NewFileStore(<session-root>/interrupts)`，与 ADR-0004 共根。
  - Store + Policy 同时注入 Primary 与 **长期存活的 ProfileFull**（coder）。**不**注入 ephemeral worker（见评审）。
  - 租约 TTL 默认 `ask_user_timeout`（秒）。
- 非交互 + interrupt 开启：`TierDangerous` 不再硬拒绝（闸门已在执行前；批准后 handler 必须能跑）。`TierBlocked` 仍硬拒绝。
- CLI 默认路径不变。

## 3. HTTP 三端点

| 端点 | 语义 |
|------|------|
| `GET /v1/sessions/{id}/interrupts` | `Store.List`，只返回 Meta |
| `POST /v1/interrupts/{id}/decisions` | `SubmitDecisions`；批准 `{tool_call_id, execute:true}`，拒绝 `{tool_call_id, is_error:true, content}` |
| `POST /v1/interrupts/{id}/resume` | `ResumeInterrupt`（空 Decisions）+ 租约；按记录 `AgentID` 经 `ResumeAgent` 解析 |

决策载荷不进日志/trace 明文。删除 session 时 List+Delete 该 session 的 interrupt。路由仅在 store 非 nil 时挂载。

## 4. 提示与可观测

- Primary 系统提示：命中闸门时告知用户已挂起等待批准，禁止改用其它工具绕过。
- 事件：复用 `interrupt_created` / `interrupt_decision_stored` / `interrupt_resumed`；加入 session control-plane 白名单。
