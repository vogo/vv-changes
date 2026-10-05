# 设计综合评审：agent 记忆工具

> 日期：2026-10-05
> 视角：与现有 TOOLS / MEM / AGENTS 不变量对照

## 评审发现与修正

| # | 风险 | 裁决 |
|---|------|------|
| 1 | 新增 EventType 会动 largemodel 第三仓库 | 复用 `EventCustom` + `EmitCustomData("memory.set")` |
| 2 | 在 `BuildRegistry` 注册会要求 store 进入 RegistryOption | CapRemember 在 BuildRegistry 为 no-op；装配期按 store 注入 |
| 3 | vage 直接依赖 `memory.Memory` 违反 tool→memory 红线 | 工具包定义本地 `Store` 接口；vv `AgentToolStore` 适配 |
| 4 | 记忆写挂 `CapWrite` 让 Review/Edit 改长期事实 | 独立 `CapRemember`，只进 Full |
| 5 | `memory_set` ReadOnly=false 会让 CLI default 模式每次确认 | 与 todo_write 一样 ReadOnly=true（非工作区文件写入）；符合「不额外授权」 |
| 6 | 空 namespace recall 若返回会话私有会泄漏草稿 | 空 = 仅共享枚举 |
| 7 | 需求只写 Primary / Full worker，Coder 也是 ProfileFull | 正交模型：Has(CapRemember) 即注册；AGENTS-R10 仍只约束 prompt 全量渲染 |
| 8 | 新 `memory.enabled` 旋钮 | 拒绝；`persistentMem == nil` 即为关闭 |

## 残留风险（接受）

- `EventToolCallStart.Arguments` 仍可能带 value（与 write 工具同样的既有审计面）。本次事件契约保证自定义 `memory.set` payload 不含 value。
- plan 模式可写记忆（ReadOnly=true）。视为可接受：不改工作区文件。
