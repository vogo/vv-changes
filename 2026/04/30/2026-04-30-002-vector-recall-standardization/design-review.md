# 设计评审:向量召回标准化(Vector Recall Standardization)

**评审目标**:对照 `design-raw.md`,在"对齐既有抽象 / 简化无痛 / 不为假设未来设计"三条原则下提出材料性改进。已采纳的改进会写入新的 `design.md`,被驳回的改动在末尾说明理由。

---

## 总评

整体设计与 `vage/memory.Store` / `vctx.Source` 既有约定基本对齐,fail-open / source-name 常量 / `ContextSourceReport` 复用 / 不破坏 Builder 的姿态都正确。主要改动集中在**接口最小化**与**修掉一处真实的预算交互 footgun**上。

---

## 改进项(按重要性排序)

### I1. `Embedder` 去掉 `Dimension()` 方法,改用 store 的"首次 Add 锁定" + `WithLockedDimension` 显式构造

**问题**:`Dimension() int` 在 `Embedder` 接口里要求每个实现都自报维度。这强迫所有真实后端(OpenAI / Anthropic / voyage)在构造时即知维度——但 OpenAI 的 text-embedding-3 系列允许在调用时通过 `dimensions` 参数动态截断,此时 SDK 实例的"维度"并不是固定常量。同时,`HashEmbedder` 这类测试 Embedder 已经把维度写在结构体字段里,接口方法就是冗余壳。

**改动**:
- `Embedder` 接口收缩为单方法:`Embed(ctx, text) ([]float32, error)`。
- 维度校验完全交给 `VectorStore`:首次 `Add` 锁定;后续 `Add` 与 `Search` 的 query 长度按锁定值校验,不一致返回 `ErrDimensionMismatch`。
- `MapVectorStore` 仍提供 `WithLockedDimension(d int)`,允许调用者在构造时显式锁定(适用于"先 Search 后 Add"的极端场景)。

**理由**:接口越窄越好实现。Embedder 实现者已经知道自己在产多长的向量;Store 是唯一真正需要"对齐"的角色,它已经能从首条数据自学。砍掉 `Dimension()` 不丢任何能力。

---

### I2. 删除 `SearchOptions.Namespaces` 字段

**问题**:本期所有 in-scope 实现都不会读这个字段。doc/原则明示"不为假设未来设计"。同时,真实后端的命名空间语义差异极大(qdrant 是 collection,pgvector 是 schema/table,chroma 是 collection),用一个抽象 `[]string` 强行统一只会带来错配。

**改动**:从 `SearchOptions` 移除 `Namespaces`。需要按 namespace 区分的调用者,用多 `VectorStore` 实例(每个 namespace 一个)即可——这是 Go 生态更习惯的姿态(对比 `*sql.DB` 一个实例对应一个 schema)。

**理由**:删一个字段比未来重命名/重定义它便宜得多。

---

### I3. `SearchOptions.Filter` 改为 `Metadata` 等值过滤 + 保留可选 `Predicate`

**问题**:`Filter func(d Document) bool` 看起来灵活但有两个隐患:
1. 真实后端(qdrant `must.match`、pgvector `WHERE metadata @>`)用的是**声明式过滤**,Go 闭包无法被翻译过去——任何真实后端要么忽略此字段、要么 client 侧二次过滤(等于先全部 fetch 再扔掉,违背向量库设计目标)。
2. 闭包关闭了优化窗口:无法做"过滤再排序"的下推。

**改动**:
- 主推 `MetadataEquals map[string]any`:声明式键值等值过滤,真实后端可下推。
- 保留 `Predicate func(d Document) bool` 作为客户端二次过滤逃生口,但在 godoc 显式标注"may be slow on large stores; backends may apply this AFTER vector search"。
- `MapVectorStore` 同时支持两者:先 `MetadataEquals` 再 `Predicate`。

**理由**:这是**唯一**直接影响真实后端实现者人体工学的接口决策。改成声明式后,实现 qdrant/pgvector 适配器时不用尴尬地"忽略 Filter"或"全表扫"。Predicate 留作兜底,不丢失测试灵活性。

---

### I4. 修复"单条聚合消息 + Builder trim"footgun

**问题**:design-raw.md §6 声称"返回的 messages 始终 ≤ 1 条"且"不依赖 Builder 的 trim 兜底"。但 Builder 的 `trimByTokens`(builder.go:325-356)对 optional source 在 `in.Budget > 0 && rep.Tokens > fin.Budget` 时无条件触发——它从头部丢消息直到 fit。当 source 只产 1 条聚合消息且超预算时,trim 会**整条丢掉**,导致 `OutputN=0` / 整组召回信息消失,且 `Status=truncated` 但 message 为空——这是 silent data loss。

**改动**(两个互补机制,都加进 source 实现):
1. **Source 自截**:在 `VectorRecallSource.Fetch` 渲染前,先用 `EstimatedTokens(text)` 估算,若 `in.Budget > 0` 且超预算,**逐条丢分数最低的 hit**直到 fit;若仍超(只剩 1 条且超预算),则按字符截断该条 text(末尾保留 `... [truncated]` marker)。这样 source 自己保证 `Tokens <= Budget`,Builder 的 trim 永远不会触发。
2. **报告字段**:被自截丢掉的 hit 计入 `DroppedN`,被字符截断时 `Status=truncated`、Note 标注 `"truncated to fit budget"`。

新增 option:`MaxBytesPerHit int`(默认 0 = 不限),供调用者把单条 hit 强行截断到上限——避免一条超长 Document 把整个预算吃掉。

**理由**:这是真正的 bug 而非 cosmetic;不修就会出现"调高预算反而看不到召回"的反常行为。源端自截算法简单(7 行代码),不引入新的预算抽象。

---

### I5. `HitsRenderer` 签名扩展为 `func(in FetchInput, hits []vector.SearchHit) string`

**问题**:渲染函数拿不到 `FetchInput` 时,无法做"按 Intent 调整 header 文案"或"附带 SessionID 给运营调试"等定制。这是日常会用到的能力,且没有真实成本(`FetchInput` 是值类型且已经在调用栈上)。

**改动**:`HitsRenderer = func(in FetchInput, hits []vector.SearchHit) string`。`defaultHitsRender` 忽略 `in` 即可,向后兼容 0 成本。

**理由**:`FetchInput` 是免费信息,留着比丢了好。`SessionStateSource` 的 `StateRenderer` 没扩这个是因为它没有 hits 概念,但这里的 hits 与 input 配对天然有用(例如 SessionTree 后续用 `Intent` 走 `non_path_nodes` scope)。

---

### I6. `defaultQuery` 选取规则更鲁棒:Intent → 倒序找最后一条带文本的 user message

**问题**:design-raw.md §5 假设 5 写"fallback 用 `BuildInput.Request.Messages` 的最后一条 user message 文本"。但:
- `Request.Messages` 可能为空(纯 SessionID 续跑)。
- 最后一条 user message 的 Content 可能是 tool_result 或 image-only,文本为空。
- 在 `BuildInput.Request == nil` 时(checkpoint resume 等场景)需 graceful fallback。

**改动**:`defaultQuery(in FetchInput) string` 流程:
1. 若 `in.Intent != ""` 返回 `in.Intent`。
2. 若 `in.Request == nil || len(Request.Messages) == 0` 返回 `""`(skipped)。
3. 倒序遍历 `Request.Messages`,返回第一条 `Role=user` 且文本部分非空的 content;否则返回 `""`。

**理由**:这是"Source 永不抛错"原则的延伸——查询提取本身要 graceful。逻辑短(10 行),覆盖三种真实失败模式。

---

### I7. 新增 `ErrNotFound` 错误哨兵(可选增强)

**问题**:`Delete` 在 design-raw.md §4.2 注释中"missing IDs are not an error",但其他真实后端可能希望区分。同时未来若加 `Get(id) (Document, error)` 也需要标准的 not-found。

**改动**:加入 `ErrNotFound = errors.New("vector: document not found")`。`MapVectorStore.Delete` 仍保持"missing 静默成功"的语义(与 `memory.MapStore.Delete` 一致),但**接口注释**更新为"实现者**可以**返回 `ErrNotFound`,调用者应该 `errors.Is` 检查";为后续 `Get` 操作预备好沿用同一哨兵。

**理由**:零成本,贴合 Go 标准库哨兵风格(`os.ErrNotExist`)。不强制行为,只规范命名。

---

## 不采纳的建议(明示)

### N1. 给 `Embedder` 加 `BatchEmbed`

**理由**:本期 `VectorRecallSource` 单次 Build 只需要嵌入一个 query 文本(intent 或 last user message),没有批量需求。索引侧的批量入库是"自动写入路径"的事,已被 §3.2 明示 out-of-scope。等真有调用方需要批量再加;现在加只会污染接口。

### N2. 给 `Embedder` 暴露 `ModelName()` / `ModelInfo()`

**理由**:用于审计/事件 payload,但 `ContextSourceReport.Note` 已能容纳这些信息(由调用者按需在 `Render` 或 source 包装层注入)。接口里强加 model name 等于把可观测性私货塞进核心契约。

### N3. 把 `MapVectorStore` 升级为带 KD-Tree / HNSW 的近似搜索

**理由**:本实现的目标客户群是"单测 + 本地实验",doc 注释明示"production stores can return ErrNotSupported on List"。暴力扫 N≤10k 的语义是足够的;真要加近似搜索就该新写一个 `HNSWMapStore`。给 `MapVectorStore` 加复杂度等于改变它的契约。会在 godoc 加 footgun warning(see I-doc 下面)。

### N4. 修改 `vctx.Builder` 装箱算法以支持"按 source 类型分配预算"

**理由**:`BudgetAllocator` 已被 design-raw §11 明示 out-of-scope。Source 自截(I4)足以解决本期所有已知问题。

### N5. 把 `VectorRecallSource` 也实现 `MustIncludeSource`

**理由**:design-raw.md §4.6 已正确说明"recall is enhancement, not a precondition"。MustInclude 用于系统提示和当前请求,这两类失败 = agent 不能工作;召回失败只是看不到历史,理应继续运行。

### N6. 把 `Embedder.Embed` 改为返回 `[]float64`

**理由**:行业惯例(OpenAI/HuggingFace/Anthropic 全部用 float32 序列化)+ 内存占用 2x。`float32` 的 7 位有效数字对余弦相似度足够。

---

## 文档/godoc 待补的 footgun(交给 documenter 阶段落实,不需进 design.md)

- `MapVectorStore` godoc 显式标 "暴力扫,N>10k 时会有可见延迟,不要在生产路径直接挂上 Builder";
- `HashEmbedder` godoc 显式标 "测试用,语义信号弱,不要用于真实评估";
- `vector.md` 在"使用建议"小节告知"VectorRecallSource 应放在 SessionMemorySource **之后**、RequestMessagesSource **之前**",并解释原因(吃完 session_memory 剩下的预算,在 request 之前出现以保持时间序);
- `Score` 字段 godoc 明确 "MapVectorStore 用余弦,范围 [-1, 1];真实后端可能用 inner-product 或 L2 距离 → MinScore 的语义随后端而变,调用者按后端调"。

---

## 验收 checklist(改后)

- [ ] `Embedder` 接口仅有 `Embed(ctx, text)`(I1)
- [ ] `SearchOptions` 字段:`TopK`、`MinScore`、`MetadataEquals`、`Predicate`(I2 + I3)
- [ ] `VectorRecallSource.Fetch` 在 `Budget > 0 && tokens > Budget` 时自截到 fit(I4)
- [ ] `HitsRenderer` 签名 `func(FetchInput, []SearchHit) string`(I5)
- [ ] `defaultQuery` 处理 nil Request / 空 Messages / 非 user 末位三种 fallback(I6)
- [ ] `ErrNotFound` 哨兵存在(I7)
- [ ] 新增单测覆盖 I4 自截路径与 I6 nil-Request 路径
