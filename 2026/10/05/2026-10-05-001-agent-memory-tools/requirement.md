# 需求：闭合长期记忆回路 —— agent 自主的记忆写入 / 召回工具

> 日期：2026-10-05
> 来源：`.work/todo.md`
> 仓库：`vage`（工具包）、`vv`（CapRemember + 接线 + 提示 + 文档）

完整原文见归档后的 `.work/done/20261005-memory-agent-tools-requirement.md`。摘要：

vv 已有三层记忆底座与 namespace ACL，但 agent 侧没有写入/按需召回入口；持久事实只能由 `/memory set` 与 HTTP 填入，Coder 每轮吞全量 persistent。本次补一对最小工具闭合回路。

## 范围

- vage：`tool/memory` 提供 `memory_set` / `memory_recall`
- vv：`CapRemember` 仅 `ProfileFull`；Primary 与 Full 档 worker（及同档 Coder）在 store 启用时注册
- 无 delete；不改 Coder 全量渲染；不接 Session 层；不加新配置旋钮

## 验收

见 requirement 原文 §5：工具层、能力接线、提示与可观测性、文档同步。
