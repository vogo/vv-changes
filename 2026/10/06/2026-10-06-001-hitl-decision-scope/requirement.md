# 需求 —— HITL 决策范围：逐 call 批准、策略指纹与单 key 记忆删除

> 状态：已实施
> 日期：2026-10-06
> 归属领域：`orchestration`（主）、`tools` / `memory` / `http-api` / `agents`（次）
> 前序：`changes/2026/10/05/2026-10-05-002-durable-hitl-interrupt/`

原文见工作区 `.work/todo.md`。评审记录在同目录 `design-review.md`，不在本文复述过程。

Durable HITL 已能把 Dangerous bash 挂起并跨进程恢复。本轮收紧决策范围：批准只覆盖被冻结且显式 Execute 的那一次调用；规则或路径护栏变化后，旧决策不得继续执行。同时补上记忆工具的单 key 删除，以及声明了记忆能力但 store 缺失时的启动告警。
