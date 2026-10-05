# 设计：agent 记忆工具（vage/tool/memory + vv CapRemember）

> 日期：2026-10-05
> 对应需求：同目录 `requirement.md`

## 1. vage/tool/memory

两个工具，`Register(reg, Store)` 与 `vectorsearch.Register` 同构（nil fail-fast、all-or-nothing）。`Store` 是本包接口（Get/Set/List），**不 import `vage/memory`**，遵守 L2 tool 不得依赖 L1 memory 的架构红线。

| 工具 | ReadOnly | 行为 |
|------|----------|------|
| `memory_set` | true | 写 `namespace:key`；共享枚举外视为会话私有 |
| `memory_recall` | true | namespace 空 = 全部共享枚举；支持 key 前缀与 limit（默认 20，上限 50） |

成功 `memory_set` 经 `schema.EmitCustomData("memory.set", {namespace, key, shared})` 发事件，**value 不入 payload**。不新增 largemodel EventType。

框架不实现 ACL：Store 实现负责。共享 namespace 名与 MEM-R2 对齐，导出 `SharedNamespaceNames` 供 vv 回归。

## 2. vv 接线

- `CapRemember`（值 `memory`）只进 `ProfileFull`。`BuildRegistry` 对 Remember 为 no-op（store 此时可能尚未注入）。
- `setup` / Full worker 在 `persistentMem != nil` 时 `memtool.Register(reg, memories.AgentToolStore(mem))`。
- `AgentToolStore` 适配 `memory.Memory` 并把 `schema.SessionIDFromContext` 转为 `memories.WithSessionID`，复用既有 ErrSessionForbidden / not-found。
- store 为 nil：不注册（零成本，无新旋钮）。
- Primary 系统提示增加 when-to-write 门槛，与工具描述同源校验。

## 3. 明确不做

LLM 抽取器、TTL/合并/打分、向量自动入库、改 Coder 全量渲染、接 Session 层、`memory_delete`。
