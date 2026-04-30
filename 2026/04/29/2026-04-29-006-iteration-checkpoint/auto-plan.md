# 复杂度评估

| 阶段 | 决定 | 理由 |
|---|---|---|
| improver | **include** | 新增独立包 `vage/checkpoint/`、定义新 API 契约（Store 接口 + Resume）、跨 3 个模块（checkpoint / agent/taskagent / schema），属"新架构模式 + 难以反转的接口"双命中。 |
| reviewer | **include** | 触碰核心 ReAct 循环（taskagent.Run / runStreamLoop）、引入新错误路径与 store 交互、并发可见（store 内 mutex + agent 上 nil-check），符合"核心/共享代码 + 接口重塑 + 多文件改动"标准。 |
| tester | **include** | 新增可验证业务逻辑（写入时机、Resume 语义、Final 终态识别）、需求列出 4 条 user story 都可以通过集成测试覆盖。 |

**Effective pipeline**：analyst → designer → **improver** → developer → **reviewer** → **tester** → documenter
