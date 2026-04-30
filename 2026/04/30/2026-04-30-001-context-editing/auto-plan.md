# Complexity Assessment

- **improver**: skip — 设计沿用现有 Middleware/DispatchFunc/EventData 模式，无新跨切关注；策略默认 keep-last-K，方案单一无需二次改进。
- **reviewer**: include — 触及 LLM 中间件链、协议无关 `aimodel.Message` 编辑、引入新公开 API 与 schema EventData，影响所有 TaskAgent 用户的潜在路径；需要独立的正确性 / 并发 / 错误处理审视。
- **tester**: include — 行为可被多场景集成测试覆盖（折叠、阈值、事件、流式、option 装配），acceptance criteria 都可验证；按设计文档 §9 已列出的整合验证。

Effective pipeline: analyst → designer → developer → reviewer† → tester† → documenter
