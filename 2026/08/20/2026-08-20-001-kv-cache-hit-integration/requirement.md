# Prompt / KV 缓存命中集成验证

- **日期**:2026-08-20
- **状态**:implemented
- **范围**:`vv/integrations/agents_tests/prompt_cache_tests/`;`specs/testing/testing-strategy.md`;`specs/non-functional/performance.md`

## Background & objectives

既有测试只证明 vv 会发出 prompt-cache 断点(`PromptCaching` / Anthropic `cache_control`),不能证明厂商 **KV / prefix cache 实际被读到**。需要一条真实 LLM 集成,在两条生产路径上检查 `Usage.CacheReadTokens > 0`。

## User scenarios

1. **普通对话**:同一 researcher 前缀(系统提示 + 工具表 + 加长 Project Instructions)上连续两次无工具 user turn;第二次 LLM 调用应命中缓存。
2. **工具调用**:限制 `glob`,强制 ReAct 至少两轮 LLM;第二轮及之后应命中同一稳定前缀的缓存。

## Scope boundaries

- **In**:vv 装配路径(`setup.New` → researcher TaskAgent → 真实 Caller)上的缓存命中观测。
- **Out**:不改运行时缓存策略;不新增配置项;无 API key / 前缀不足 1024 token 时 skip,不在默认 `make test` 上制造红灯。
