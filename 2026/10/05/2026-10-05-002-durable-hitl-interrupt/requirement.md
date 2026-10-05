# 需求 —— 接入 durable HITL：把 vage 的 interrupt 状态机接到 vv 的 Primary 与 HTTP 面

> 状态：已实施
> 日期：2026-10-05
> 归属领域：`orchestration`（主）、`tools` / `http-api` / `configuration`（次）
> 前序：`.work/done/20261005-memory-agent-tools-*.md` 已闭合长期记忆写侧

原文见工作区 `.work/todo.md`（归档后为 `.work/done/20261005-durable-hitl-interrupt-requirement.md`）。

给 Primary（及长期存活的 Full 档代理）接入 interrupt 挂载点，并在 HTTP 面暴露「挂起 → 提交决策 → 原地恢复」三个动作。范围：装配接线 + 一个策略 + 三个端点；vage 仅补「批准后执行」与流式事件投递两个通用缺口。
