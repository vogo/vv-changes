# 原始需求

参考 doc/design/session-context-solution.md, 评估方案完整性,调研行业优秀实践, 设计方案,并实现.

本次要实现的功能是: **Context Builder 抽象**

实现后在 doc/design/session-context-solution.md 标记完成。

## 背景

vage 当前 TaskAgent 中"从 session 到 prompt"的拼装过程是隐式硬编码的：
- 直接读取 memory（working/session/store）
- 直接拼接 messages
- 没有 token 预算管理
- 没有 BuildReport 用于审计/可观察性
- 无法插拔不同的 context source

差距详见 `doc/design/session-context-solution.md` §8 表格中 "Context Builder" 一行：
> Context Builder | ⚠️ 隐式于 TaskAgent 内部 | **缺抽象**：无法插拔 source、无 BuildReport
