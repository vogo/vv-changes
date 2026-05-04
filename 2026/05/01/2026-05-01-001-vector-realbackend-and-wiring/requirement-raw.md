# 原始需求

> 来自用户 `/sdd-lite` 输入,日期 2026-05-01。

## 输入

```
ref doc/design/session-context-solution-phase2.md, impl: | **第 1 步** | P0-1 + P0-2(向量真实后端 + 自动写入 + vv wiring + LLM 工具) | 召回能力从"接口就位"变"产品可用",影响范围最广 |
```

## 引用文档

`doc/design/session-context-solution-phase2.md` 中 §"落地建议路径" 第 1 步,展开为:

### P0-1 Vector 真实后端 + 真实 Embedder

- 真实后端适配:qdrant / pgvector / chroma / weaviate **至少一个**
- 真实 Embedder:OpenAI `text-embedding-3-*` 与 voyage **至少一个**
- `Embedder` 接口扩展:`BatchEmbed` / `ModelName()` / `MaxInputTokens()`
- 维度对齐策略验证:首次 Add 锁定 vs `WithLockedDimension(d)` 显式声明

### P0-2 Vector 自动写入路径 + vv 端 wiring + LLM 工具

- 写入侧:Run 结束时把 SessionMemory 转向量索引(归档器联动 / 异步触发)
- vv `setup.Init` 默认挂 store + embedder + `VectorRecallSource`(可配置启停)
- HTTP `POST /v1/vector/add` / `GET /v1/vector/search`
- LLM 工具 `vector_search` / `vector_add`
- 配置段 `vector:`(后端、embedder、自动写入开关、top_k 默认值)
