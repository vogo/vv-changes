# 原始需求

参考 doc/design/session-context-solution.md, 评估方案完整性, 调研行业优秀实践, 设计方案, 并实现。
实现后在 doc/design/session-context-solution.md 标记完成。

本次要实现的功能是: vage 框架层（promotion + 折叠）

## 设计文档中对应章节（A1–A5）

A1. 数据模型（vage/session/tree/tree.go）
- TreeNode 新增 Promoted bool + PromotedAt time.Time
- Metadata["summary_source"] = "user"|"promotion" 区分摘要来源（不破坏字段）
- 新增触发阈值常量

A2. Store 接口扩展（store.go）
- PromoteNode(ctx, sessionID, nodeID) (*TreeNode, error) —— 同步聚合，store 内部串行
- GetTreeView(ctx, sessionID, opts ViewOptions) (*SessionTree, error) —— 含 IncludePromoted bool

A3. Promoter（新文件 promoter.go）
- 接口：Summarize(ctx, parent, children) (string, error)
- 内置：LLMPromoter（注入 aimodel.ChatCompleter + 可选 model name）/ CompressorPromoter（复用 memory.ContextCompressor）/ NoopPromoter

A4. 触发器与异步执行（triggers.go）
- PromotionDecider 接口 + 阈值组合（AnyOf / AllOf）
- store 在 AddNode/UpdateNode 写成功后同步判断、异步执行
- per-(session,node) singleflight 防重入
- 三个事件：EventSessionTreePromotionStarted/Completed/Failed

A5. 渲染（vage/context/sources_tree.go）
- buildView 跳过 Promoted=true（非 path 节点）
- 父节点渲染时显示 (folded: N children, M done) —— "折叠"的可视化
- 新增 option WithIncludePromoted(true) 给 zoom-in 工具用

## 范围说明（明示）

本期仅交付 A1–A5（vage 框架层），不包括：
- A6（vage LLM 工具包 vage/tool/sessiontree/）
- B1–B7（vv 应用层 wiring）
- 双索引、跨 session 森林等

## 实现完成后

在 doc/design/session-context-solution.md 中：
- §4.8.6 "渐进路线" 第 2 步打 ✅
- §8 差距汇总 "Session Tree" 行的注释更新

