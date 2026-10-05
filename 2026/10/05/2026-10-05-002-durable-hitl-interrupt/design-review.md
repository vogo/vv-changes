# 设计综合评审 —— durable HITL

> 日期：2026-10-05

| # | 初稿风险 | 裁决 |
|---|---------|------|
| 1 | vage pending 不执行 handler，批准 bash 无意义 | 补通用 `Decision.Execute`；不把 bash 分类器注入 vage |
| 2 | 新增 `EventInterruptPending` / 占用 `pending_interaction` | 复用已有 `EventInterruptCreated`；经 stream emitter 投递 |
| 3 | `interrupt_rejected` 新 EventType | 拒绝走 `is_error` 决策 + 既有 `interrupt_decision_stored`（载荷不含正文） |
| 4 | 给 Full 档 ephemeral worker 接线 | **不接**。worker AgentID 即用即弃，ResumeInterrupt 会 AgentID mismatch；nested HITL 仍 TOOL-6b。只接 Primary + 启动期 ProfileFull（coder） |
| 5 | 批准执行后 permission 仍硬拒绝 Dangerous | interrupt 开启时跳过非交互 Dangerous 硬拒绝；Blocked 不变 |
| 6 | FileStore 与 session 目录混放 | 根目录 `<session-root>/interrupts/`；`interrupt_enabled` 强依赖 `session.enabled` |
| 7 | 补 `NewToolNamePolicy` | 不需要，`WithInterruptToolNames` 已有；vv 用自定义 Dangerous bash 策略 |
| 8 | 动 largemodel | 不改。HTTP 决策走 vage `interrupt.Decision` |
| 9 | classifier 关闭 + interrupt 开启 | 启动失败：闸门无判据却放开硬拒绝会放行危险命令 |
| 10 | 决策正文进 trace | 事件只带 interrupt_id / tool_call_id / ready，不含 Content |
