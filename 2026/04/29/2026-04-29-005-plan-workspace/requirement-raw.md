# Plan 工作区 — 原始需求

## 用户原始指令

参考 doc/design/session-context-solution.md, 评估方案完整性, 调研行业优秀实践, 设计方案, 并实现.
实现后在 doc/design/session-context-solution.md 标记完成.
本次要实现的功能是: **Plan 工作区**.

## 背景指引

设计文档 §4.4 Plan/Scratchpad 工作区（P7）原始描述：

> 约定每个 Session 拥有一个**工作目录** `.vv/sessions/<id>/`，含：
> - `plan.md` —— 任务大纲与进度（LLM 维护，dispatcher 注入到 prompt 顶部）
> - `scratch/` —— 子任务草稿
> - `notes/` —— 长期事实卡片（与 memory.Store 同步）
> - `artifacts/` —— 产出物（diff、报告、日志）
>
> 工作目录通过文件工具暴露给 LLM（P4），主上下文中只放 `plan.md` + 目录索引。

依赖现状（设计文档 §8 已记录）：
- ✅ vage/session/（实体 + Store + Hook）已落地
- ✅ vage/context/（Builder + Source）已落地
- ✅ vv setup wiring：FileSessionStore + SessionHook + CLI/HTTP CRUD 已落地
- ❌ Plan/Scratchpad 工作区：尚未实现

## 验收

实现完成后需在 doc/design/session-context-solution.md 标记 Plan 工作区为已落地。
