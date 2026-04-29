# 复杂度评估

| 阶段 | 决定 | 理由 |
|---|---|---|
| improver | **跳过** | 设计完全沿用 vage/session 已有 API,新增的 vv 层 SessionConfig / FileStore 注入 / 5 个 HTTP handler / CLI flag 都是常规 wiring,没有引入新架构模式或公开接口的不可逆决策;复杂度低,senior 二审收益小。|
| reviewer | **包含** | 改动横跨 13 个文件,触及 `setup.Init` 启动顺序(回滚关闭 + Manager 二 hook)、HTTP API 新表面、CLI 启动路径,正是"多文件 + 50+ 行非样板 + 涉及生命周期/并发"的 reviewer 触发条件。 |
| tester | **包含** | 业务接口、HTTP API、CLI flag 都新增,需求文档列出了每条端点的 200/400/404 行为以及 CLI resume 三种分支(New/Existing/NotFound),全部可用集成测试覆盖且必须覆盖,避免 regress。 |

## 有效流水线

`analyst → designer → developer → reviewer (sub-agent) → tester (sub-agent) → documenter`

(improver 跳过,因复杂度评估为"中等",reviewer 已经能管住质量。)
