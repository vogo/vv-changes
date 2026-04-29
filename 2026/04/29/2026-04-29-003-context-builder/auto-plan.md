# 复杂度评估 / Complexity Assessment

## 决策

| 阶段 | 是否包含 | 一句话理由 |
|---|---|---|
| analyst | ✅ 必须 | 始终运行 |
| designer | ✅ 必须 | 始终运行 |
| improver | ✅ 包含 | 新公共接口（Builder/Source）+ 跨多 module（context/schema/taskagent）+ 多种可选实现路径，需要二次评审收敛 |
| developer | ✅ 必须 | 始终运行 |
| reviewer | ✅ 包含 | 改动 TaskAgent 核心路径（buildInitialMessages）+ 新公共接口 + 行为兼容性敏感，需独立 review |
| tester | ✅ 包含 | 多组件交互（Source × Builder × TaskAgent × Hook）、行为兼容是验收关键、新事件触发可被集成测试 |
| documenter | ✅ 必须 | 始终运行 |

## 有效流水线

`analyst → designer → improver → developer → reviewer → tester → documenter`
