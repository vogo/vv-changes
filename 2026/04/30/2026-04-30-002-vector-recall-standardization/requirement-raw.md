# 原始需求

参考 `doc/design/session-context-solution.md`,评估方案完整性,调研行业优秀实践,设计方案,并实现。

实现后在 `doc/design/session-context-solution.md` 标记完成。

本次要实现的功能是: **向量召回标准化(Vector Recall Standardization)**。

## 上下文

`doc/design/session-context-solution.md` §4 列出了 vage / vv 的演进路径,本次聚焦于**向量召回标准化**。从该文档 §8 "与 vage 现状的差距汇总" 可见:

> | 向量召回 | ⚠️ 依赖 store 实现 | 缺标准 VectorRecallSource |

§4.2 表格中也明确把 `VectorRecallSource` 列在"留作后续迭代"的清单里,这次落地。

§3 把检索式上下文构造(P3 — Retrieval-Augmented Context)定位为:
- 历史/知识向量化按需召回(MemGPT archival、LangChain VectorStoreRetrieverMemory)
- 对长对话与知识密集场景必备
- 与现有 P1(三层 memory)、P2(压缩器)正交,作为 ContextSource 接入 Builder

§4.8.3 中,树驱动 context 构造也依赖向量检索补丁(`vector_recall(intent, scope=non_path_nodes, top_k=5)`)。所以本次落地不仅服务长对话,也是后续 SessionTree(P10)的依赖。

§5 工程决策:
- 检索召回时机 = **同步**,Build 时按当前 intent 召回
- Token 预算分配建议 retrieval ≈ 20%
