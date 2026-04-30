# 复杂度评估与阶段决策

## 决定

| 阶段 | 包含？ | 一句话理由 |
|---|---|---|
| improver | **跳过** | 设计已详尽：含风险表、两阶段锁/singleflight 取舍、out-of-scope 清单；再做一轮 senior review 边际收益低。 |
| reviewer | **包含** | 接口扩展（公共 SessionTreeStore 新增 2 方法）+ 并发原语（per-key singleflight、两阶段锁）+ 跨 3 模块（tree/context/schema），需要独立第二意见。 |
| tester | **跳过** | 单测计划已覆盖所有 acceptance criteria + decider 边界 + 并发触发 + Map/File 双 store conformance；本期无外部依赖，integration test 与单测会高度重叠。 |

## 有效流水线

```
analyst -> designer -> developer -> reviewer† -> documenter
```

`†` 表示 sub-agent dispatch；其余为 main agent 内联。
