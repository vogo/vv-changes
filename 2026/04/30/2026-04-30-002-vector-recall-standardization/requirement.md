# 需求:向量召回标准化(Vector Recall Standardization)

## 1. 背景与目标

`doc/design/session-context-solution.md` §3 将"检索式上下文(Retrieval-Augmented Context, P3)"列为高效 context 的关键模式之一,§4.2 把 `VectorRecallSource` 列在 ContextBuilder 的"留作后续"清单中,§8 现状汇总将"向量召回"标记为 ⚠️ 缺标准实现。

vage 当前的 `memory` 包提供了三级 KV(Working / Session / Persistent),但 KV 不是相似性检索——长对话、知识密集场景需要从历史中按当前 intent 语义召回若干片段塞进 prompt。本次目标是**为 vage 标准化向量召回链路**,使其作为 `vctx.Builder` 的一类 Source 自然接入,同时不强求引入特定向量库后端。

**对齐文档原文**:
- §3 P3:历史/知识向量化按需召回(代表:MemGPT archival、LangChain VectorStoreRetrieverMemory)
- §5 工程决策:**召回时机 = 同步**,Build 时按当前 intent 语义召回
- §4.8.3:树驱动 context 算法依赖 `vector_recall(intent, scope=..., top_k=5)`——为未来 SessionTree(P10)预备依赖

## 2. 用户故事与验收标准

### 故事 A:框架使用者把向量召回挂进 Context Builder

**作为** vage 的下游使用者(vv 或第三方应用),
**我希望** 通过类似 `vctx.WithSource(&vctx.VectorRecallSource{Store: myStore, Embedder: myEmbedder})` 的方式启用向量召回,
**以便** 让 LLM 在每轮 ReAct 之前看到与当前请求语义最相关的历史片段或知识。

**验收标准**:
- [ ] `vage/vector/` 包提供 `VectorStore` 与 `Embedder` 两个接口
- [ ] `vage/context/sources_vector.go` 提供 `VectorRecallSource`,实现 `vctx.Source`
- [ ] `VectorRecallSource` 不带 `MustInclude` 标记(召回是优化,不是必需)
- [ ] 与 Builder 集成后:正常路径返回 top-k 片段拼成的 system 消息;空 store / 无 embedder / 空 query 三种情况都 fail-open(`Status=skipped`,不报错)
- [ ] Embedder / Store 调用错误 fail-open(`Status=error`,Builder 仍继续)

### 故事 B:开发与测试可以脱离真实向量库后端工作

**作为** vage 的核心开发者,
**我希望** 向量召回链路有一个内置的进程内实现,
**以便** 单元测试与本地实验不依赖外部服务。

**验收标准**:
- [ ] 提供 `MapVectorStore`(内存版,余弦相似度)作为默认实现
- [ ] 提供 `EmbedderFunc` 函数适配器,便于测试(如返回 hash-based 伪嵌入)
- [ ] 内存实现支持 Add / Search / Delete / List 四个核心操作
- [ ] 全部用例通过 `go test ./vector/... ./context/...` 验证

### 故事 C:可观测性与已有体系一致

**作为** 平台运维者,
**我希望** 向量召回的命中数、相似度分布、耗时可以被审计,
**以便** 评估其对响应质量与成本的影响。

**验收标准**:
- [ ] `VectorRecallSource.Fetch` 产出的 `schema.ContextSourceReport` 在 `Note` 里附带 top-k 命中摘要(at least scores 范围 + 命中数)
- [ ] `OutputN` 反映最终注入 prompt 的消息数(通常为 1 条聚合的 system 消息)
- [ ] `InputN` 反映 store 中扫描或查询返回的候选数量
- [ ] `EventContextBuilt` payload 中已有的 `Sources` 字段自动覆盖该 source

## 3. 范围

### 3.1 In-Scope

1. **接口层**(`vage/vector/`):
   - `VectorStore` 接口:Add / Search / Delete / List
   - `Embedder` 接口 + `EmbedderFunc` 适配器
   - `Document` / `SearchHit` / `SearchOptions` 数据结构
2. **内置实现**(`vage/vector/`):
   - `MapVectorStore` — 进程内 map 后端,余弦相似度暴力扫
   - `HashEmbedder`(测试用) — 基于 token-bag 的伪嵌入,不依赖真实 LLM
3. **接入 Builder**(`vage/context/sources_vector.go`):
   - `VectorRecallSource`(可选,非 MustInclude)
   - 渲染 top-k 命中为单条 system 消息
   - 完整 fail-open 路径
4. **测试**:
   - 单测覆盖:happy path / 空 store / Embedder 报错 / Store 报错 / top-k 限制 / 相似度阈值过滤
5. **文档**:
   - `vage/.doc/vector.md` 新增
   - `vage/.doc/context.md` 更新 §3 内置 Source 表格 + §10 移除 VectorRecallSource 条目
6. **回写设计文档**:
   - 更新 `doc/design/session-context-solution.md`:§4.2、§8 行、§8 末尾 roadmap 中"向量召回标准化"标记为 ✅ 已落地

### 3.2 Out-of-Scope(明确不做)

- **真实向量库后端集成**(qdrant / pgvector / chroma / pinecone)— `VectorStore` 是接口,使用者按需提供实现
- **生产级 Embedder 实现**(OpenAI / Anthropic embeddings 调用)— `Embedder` 是接口,使用者按需提供;本次仅给测试用 HashEmbedder
- **自动写入(insertion)路径**(从 `memory.Session` 自动批量索引到 `VectorStore`、Run 结束时回写、归档器联动)— 索引时机与策略涉及业务决策,本次不绑定。下游可以在 hook 或 Run 结束钩子里手动索引
- **Builder 多预算策略**(`BudgetAllocator` 接口、按比例分配)— 沿用 `ordered_greedy`
- **混合检索**(BM25 + dense)— 留待后续 GraphRAG / HippoRAG 等方向
- **vv 端 wiring** — 本次只完成 vage 层标准化;vv 何时启用、用什么后端,后续按需对接

## 4. 涉及的角色与模块

| 维度 | 影响 |
|---|---|
| 角色 | 框架使用者(vv 与第三方)、core developer、运维 |
| 模块(vage) | 新增 `vector/` 包;新增 `context/sources_vector.go`;无破坏性改动 |
| 模块(vv) | 不动 |
| 文档 | 新增 `vage/.doc/vector.md`;更新 `vage/.doc/context.md`;更新 PRD `doc/prd/models/core/memory/`(新增 vector 模型卡);回写 `doc/design/session-context-solution.md` |

## 5. 关键澄清/假设

1. **embedding 维度由 Embedder 自行决定**,VectorStore 在首次 Add 时记住维度并对后续 Add/Search 校验。
2. **相似度度量统一为余弦相似度**(归一化点积)。其他度量(欧氏、点积无归一化)留作未来扩展。
3. **召回结果的 Document 内容由调用者注入**,VectorRecallSource 只负责"如何把 hits 渲染成 system message",渲染函数可被注入(`HitsRenderer`)。
4. **Intent**:本次将 `BuildInput.Intent`(已有 string 字段)作为可选 query。当 Intent 为空时,fallback 用 `BuildInput.Request.Messages` 的最后一条 user message 文本。两者都为空 → skipped。
5. **不修改 `vctx.Builder` 装箱算法**,VectorRecallSource 自行控制 top-k 与每条 hit 长度,不依赖 Builder 的 trim 兜底(避免把单条聚合消息拦腰截断)。

## 6. 与已有模块的兼容性

- 不修改任何已有公开接口(`Memory`、`Builder`、`Source` 等)。
- 复用 `schema.ContextSourceReport`(已有字段足够)。
- 复用 `vctx` 的 fail-open 约定与 source-name 常量风格(新增 `SourceNameVectorRecall = "vector_recall"`)。

## 7. 与已有 PRD/文档的不一致

调研后未发现冲突。`doc/prd/models/core/memory/` 当前不涉及向量,新增向量模型卡是增量。
