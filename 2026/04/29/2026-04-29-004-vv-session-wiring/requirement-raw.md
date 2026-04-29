# 原始需求

参考 doc/design/session-context-solution.md, 评估方案完整性,调研行业优秀实践, 设计方案,并实现.

本次要实现的功能是: **vv 端 Session wiring**(CLI 续接、HTTP CRUD、setup 默认挂 SessionHook)

实现后在 doc/design/session-context-solution.md 标记完成.

## 上下文(在 design 文档中已经声明)

- vage/session/ 包已经落地(2026-04-29-002 迭代),提供 Session、SessionStore、SessionHook
- vage/context/ 包已经落地(2026-04-29-003 迭代),提供 ContextBuilder
- 本次实现属于 P1(三层 memory + Session 实体)在 vv 应用层的最后一公里
