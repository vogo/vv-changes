# 需求 —— Skill 从「两个内置常量」升级为「文件发现 + 会话激活」

> 状态：已实施
> 日期：2026-10-05
> 归属领域：`agents`（主）、`configuration` / `orchestration` / `tools`（次）
> 前序：`.work/done/20261005-memory-agent-tools-*.md`、`.work/done/20261005-durable-hitl-interrupt-*.md`

原文见工作区 `.work/todo.md`（归档后为 `.work/done/20261005-skill-file-discovery-requirement.md`）。

把 vv 宣称的 Skill 正交维度从硬编码的 `review` / `research` 两个常量，接到已经完工的 vage `skill` 子系统：启动期从 `agents.skill_dir` 发现 `SKILL.md`，Primary 经 `use_skill` 按 session 激活，下一轮注入系统提示。本期不导入 `allowed_tools`、不热插拔、不给 worker 激活入口。
