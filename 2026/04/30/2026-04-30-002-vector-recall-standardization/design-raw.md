# 设计:向量召回标准化

## 1. 设计目标

把"向量召回"提升为 vage 的一等公民:
- **接口标准** — 让 `VectorStore` 与 `Embedder` 像 `memory.Store` 一样可插拔。
- **Builder 集成** — 提供 `VectorRecallSource`,无缝挂入现有 `vctx.Builder` 管线。
- **下游零依赖** — 内置进程内实现 + 测试用 hash embedder,使用者不强制引入向量库。
- **可观测** — 复用 `schema.ContextSourceReport` 与 `EventContextBuilt`。

## 2. 业界实践对照

| 框架 | 接口形态 | 核心抽象 | 我们采纳 |
|---|---|---|---|
| LangChain `VectorStoreRetrieverMemory` | retriever + vectorstore 两层 | `Retriever.GetRelevantDocuments(query)` | ✅ Source 内部组合 Embedder + Store,等价于 retriever |
| MemGPT archival | 工具暴露 `archival_search` | `Document{text, embedding, metadata}` | ✅ Document 模型 + 后续可暴露为 LLM 工具 |
| OpenAI File Search | thread-bound vector store | 服务端隐式管理 | ❌ 非框架职责 |
| LlamaIndex VectorMemory | composable memory | top-k + token budget | ✅ 复用 vctx Source 的 budget 协议 |
| pgvector / qdrant / chroma | 后端类 | namespace + filter + hybrid | ⚠️ 用 `SearchOptions` 预留过滤位 |

**抽取出的最小核心**:
```
                  ┌────────────┐                   ┌────────────────┐
  query string ─▶ │  Embedder  │ ─ vector ─▶ Store │  VectorStore   │ ─▶ []SearchHit
                  └────────────┘                   └────────────────┘
                                                          ▲
                                            Add(Document) │
                                                          │
                                                  外部插入路径(本期不绑定)
```

## 3. 包与文件布局

```
vage/vector/                    # 新增包
├── vector.go                   # VectorStore, Embedder, Document, SearchHit, SearchOptions
├── mapstore.go                 # MapVectorStore - 进程内 map 实现 + 余弦相似度
├── mapstore_test.go
├── embedder.go                 # EmbedderFunc 适配器, HashEmbedder(测试用)
└── embedder_test.go

vage/context/
├── sources_vector.go           # 新增:VectorRecallSource
└── sources_vector_test.go      # 新增:对应单测
```

`vage/context/source.go` 增加常量 `SourceNameVectorRecall = "vector_recall"`。

行长上限遵守仓库规则(单文件 ≤ 800 行)。当前规划每文件远低于该上限。

## 4. 核心 API

### 4.1 `vector.Document` / `vector.SearchHit`

```go
// Package vector defines a small, pluggable vector-recall surface used by
// vage/context/VectorRecallSource and other future retrieval-driven sources.
package vector

import (
    "context"
    "time"
)

// Document is a single indexed item.
type Document struct {
    ID        string         // caller-supplied stable identifier
    Text      string         // the human-readable payload returned to the LLM
    Embedding []float32      // length must match the store's locked dimension
    Metadata  map[string]any // optional filtering / display hints
    CreatedAt time.Time      // populated by the store on Add if zero
}

// SearchHit pairs a Document with the similarity score to the query vector.
type SearchHit struct {
    Document Document
    Score    float32 // higher is more similar; cosine in MapVectorStore
}

// SearchOptions controls a Search call.
type SearchOptions struct {
    TopK       int                                  // 0 -> store default (e.g. 5)
    MinScore   float32                              // 0 -> no threshold
    Filter     func(d Document) bool                // optional post-filter
    Namespaces []string                             // optional: restrict to a set
}
```

### 4.2 `vector.VectorStore`

```go
// VectorStore is the pluggable backend. Implementations must be safe for
// concurrent use unless documented otherwise.
type VectorStore interface {
    // Add inserts or replaces a document. The store locks the embedding
    // dimension on the first Add and rejects mismatched future Adds with
    // ErrDimensionMismatch.
    Add(ctx context.Context, doc Document) error

    // Search returns up to TopK hits whose embedding is closest to query
    // under the store's similarity metric. Ordering: highest score first.
    Search(ctx context.Context, query []float32, opts SearchOptions) ([]SearchHit, error)

    // Delete removes the document with the given ID. Missing IDs are not an
    // error.
    Delete(ctx context.Context, id string) error

    // List returns every stored document (without scores). Useful for
    // introspection and tests; production stores may return ErrNotSupported.
    List(ctx context.Context) ([]Document, error)
}
```

错误标准化:
```go
var (
    ErrEmptyQuery         = errors.New("vector: empty query")
    ErrDimensionMismatch  = errors.New("vector: embedding dimension mismatch")
    ErrNotSupported       = errors.New("vector: operation not supported by backend")
)
```

### 4.3 `vector.Embedder`

```go
// Embedder maps a textual query/document to a fixed-length vector.
type Embedder interface {
    // Embed returns the embedding. Implementations may be backed by an LLM
    // API; failures must be returned for caller fail-open handling.
    Embed(ctx context.Context, text string) ([]float32, error)

    // Dimension reports the embedding length. Returned for sanity checks
    // and for the store's first-Add dimension lock.
    Dimension() int
}

// EmbedderFunc adapts a function into an Embedder. Dimension is a fixed
// value supplied at construction.
type EmbedderFunc struct {
    Fn  func(ctx context.Context, text string) ([]float32, error)
    Dim int
}

func (f EmbedderFunc) Embed(ctx context.Context, text string) ([]float32, error) { ... }
func (f EmbedderFunc) Dimension() int                                            { ... }
```

### 4.4 `MapVectorStore` (内置)

- 后端 `map[string]Document`,`sync.RWMutex` 保护。
- Search 全表扫 + 余弦相似度 + 排序 + 截 TopK + 应用 MinScore / Filter。
- 默认 TopK = 5。
- `lockedDim`:首次 Add 设置;后续不一致返回 `ErrDimensionMismatch`。
- 余弦实现:`dot(a,b) / (norm(a) * norm(b))`;输入零向量返回 0(避免 NaN)。

```go
func NewMapVectorStore(opts ...MapStoreOption) *MapVectorStore
type MapStoreOption func(*MapVectorStore)
func WithDefaultTopK(k int) MapStoreOption       // default 5
func WithLockedDimension(d int) MapStoreOption   // explicit dim, skips first-Add lock
```

### 4.5 `HashEmbedder`(测试用)

- 维度可配(默认 128)。
- 实现:对文本做小写 token split,token 哈希到桶,L2 归一化。
- 同文本一致输出,不同文本得到弱相关向量,够用作单测语义信号。
- 文件 doc-comment 明确说"NOT for production"。

### 4.6 `vctx.VectorRecallSource`

```go
package vctx

import (
    "github.com/vogo/vage/vector"
)

// HitsRenderer formats search hits into a single message body.
// Returning "" makes the source emit Status="skipped".
type HitsRenderer func(hits []vector.SearchHit) string

// VectorRecallSource is an optional Source. It does NOT implement
// MustIncludeSource — recall is an enhancement, not a precondition.
type VectorRecallSource struct {
    Store    vector.VectorStore   // required; nil -> skipped
    Embedder vector.Embedder      // required; nil -> skipped
    TopK     int                  // 0 -> use store default
    MinScore float32              // 0 -> no threshold
    Filter   func(vector.Document) bool   // optional pre-render filter
    Render   HitsRenderer         // nil -> defaultHitsRender
    // QueryFn returns the query text. nil -> defaultQuery: prefer Intent,
    // fall back to last user message in Request.
    QueryFn func(in FetchInput) string
}

func (s *VectorRecallSource) Name() string { return SourceNameVectorRecall }
func (s *VectorRecallSource) Fetch(ctx context.Context, in FetchInput) (FetchResult, error)
```

**Fetch 行为流**:
1. 若 `Store == nil` 或 `Embedder == nil` → `Status=skipped`,Note 说明。
2. 计算 query 文本(`QueryFn` 或 `defaultQuery`)→ 空则 skipped。
3. `Embedder.Embed(ctx, query)` 失败 → fail-open,`Status=error`,记录 error,`InputN=0`。
4. `Store.Search(ctx, vec, opts)` 失败 → fail-open,同上。
5. 应用 `MinScore` / `Filter` 二次过滤(若 store 已支持则不再次重叠应用)。
6. 渲染为单条 system 消息。当 hits 为空 / 渲染为空字符串 → `Status=skipped`。
7. 报告字段:
   - `InputN` = store 返回的候选数(过滤前)
   - `OutputN` = 1(成功时)/ 0(skipped/error)
   - `DroppedN` = `InputN - len(filtered_hits)`(过滤后丢弃的数量)
   - `Note` = e.g. `"hits=3 score=[0.62..0.81]"`

**默认渲染格式**(可被 `Render` 覆盖):
```
## Relevant Memories
1. (score=0.81) <text>
2. (score=0.74) <text>
3. (score=0.62) <text>
```

**预算交互**:source 自行控制 TopK 与每条命中长度。**不**依赖 Builder 的 trim 兜底——因为 Builder 的 trim 是按消息(头丢)切,而本 source 只产出 1 条聚合消息,头丢会直接把它整条拿掉。预算分配仍由声明顺序决定:把 VectorRecallSource 放在 SessionMemorySource 之后即可享受其消费完后的剩余预算,但每条 hit 长度由调用者控制 TopK 来管。

## 5. 集成点

```go
import (
    vctx "github.com/vogo/vage/context"
    "github.com/vogo/vage/vector"
)

store := vector.NewMapVectorStore()
emb := vector.HashEmbedder{Dim: 128}

// 由调用者负责把素材索引进 store(本期不绑定写入路径)
_ = store.Add(ctx, vector.Document{
    ID: "doc-1", Text: "User prefers dark mode",
    Embedding: mustEmbed(emb, "User prefers dark mode"),
})

builder := vctx.NewDefaultBuilder(
    vctx.WithSource(&vctx.SystemPromptSource{Template: tmpl}),
    vctx.WithSource(&vctx.SessionMemorySource{Manager: memMgr}),
    vctx.WithSource(&vctx.VectorRecallSource{
        Store:    store,
        Embedder: emb,
        TopK:     5,
        MinScore: 0.2,
    }),
    vctx.WithSource(&vctx.RequestMessagesSource{}),
)
```

## 6. 不变量与边界

| 不变量 | 说明 |
|---|---|
| Embedder 与 Store 维度一致 | Store 在首次 Add 锁定;Search 用 query 长度做即时校验 |
| Source 永不抛错 | 全部错误 fail-open,Builder 不被打断 |
| Source 不修改任何输入参数 | Document / FetchInput 都是值传递或只读 |
| 返回的 messages 始终 ≤ 1 条 | 简化下游装箱;hits 内部多条聚合 |
| Render 返回空 = skipped | 与 SessionStateSource 一致的语义 |

## 7. 测试计划

### 7.1 单元测试

`vage/vector/mapstore_test.go`:
- TestMapVectorStore_AddSearch — 三个文档,query 余弦最近的命中第一
- TestMapVectorStore_DimensionLock — 第二条 Add 维度不一致返回 ErrDimensionMismatch
- TestMapVectorStore_TopK_MinScore_Filter — 三种过滤分别生效
- TestMapVectorStore_Delete — 删除后不再被 Search 命中
- TestMapVectorStore_Concurrent — go vet -race 单 goroutine 写、多 goroutine 读不竞争(go test -race)

`vage/vector/embedder_test.go`:
- TestEmbedderFunc_Roundtrip
- TestHashEmbedder_Stability — 同文本同向量
- TestHashEmbedder_RoughSemantic — 共享词的两个文本余弦 > 完全不同的两个文本

`vage/context/sources_vector_test.go`:
- TestVectorRecallSource_HappyPath
- TestVectorRecallSource_SkippedOnNilStore / NilEmbedder / EmptyQuery / NoHits
- TestVectorRecallSource_FailOpen_EmbedError / SearchError
- TestVectorRecallSource_TopK / MinScore / Filter
- TestVectorRecallSource_QueryFnFallbackToLastUserMsg
- TestVectorRecallSource_BuilderIntegration — 与 DefaultBuilder 串起来,验证 ContextBuiltData.Sources 包含 vector_recall 报告

### 7.2 集成测试

不新增;`integrations/` 中的现有 taskagent 测试不被破坏(本期不修改 TaskAgent 默认 source 列表)。

## 8. 文档变更

- 新增 `vage/.doc/vector.md` 一份完整模块文档(接口、内置实现、使用示例、与 context/memory 的关系、未来扩展)。
- 更新 `vage/.doc/context.md`:§3 内置 Source 表格补 `VectorRecallSource` 行;§9.2 / §10 把 VectorRecallSource 从"留作后续"清单移除。
- 更新 PRD `doc/prd/models/core/memory/`:新增 `model-vector-store.md` 与 `model-vector-document.md` 两份模型卡。
- 回写 `doc/design/session-context-solution.md`:§4.2 / §8 / 末尾 roadmap 三处状态更新。

## 9. 与原始草案 §4 演进路径的关系

本次落地对应 §3 P3 与 §4.2"留作后续"清单中的 `VectorRecallSource`。完成后 §8 现状汇总:
- 「向量召回」:⚠️ 依赖 store 实现 → ✅ 标准 VectorRecallSource + MapVectorStore + Embedder 接口
- 末尾 roadmap"向量召回标准化"勾选完成

## 10. 风险与权衡

| 风险 | 处理 |
|---|---|
| 余弦相似度对未归一化向量结果不稳定 | MapVectorStore 在 Search 内部做归一化,而非要求调用者预归一 |
| HashEmbedder 太弱可能误导用户 | 文件 doc-comment + README 显式标 "test only" |
| 后续真实后端语义可能与 MapVectorStore 不同(如 score scale) | 接口契约说明:Score 单调递增即可,具体语义由后端定义;`MinScore` 由调用者按后端调 |
| 召回结果泄露敏感内容 | 不在本期决策;guard 包负责 output 侧拦截 |
| 维度不一致难调试 | Add 立即返回 `ErrDimensionMismatch`,不等到 Search 才暴露 |

## 11. 不在本期实现(明示)

- 真实后端(qdrant/pgvector/chroma)
- 真实 Embedder(OpenAI/Anthropic/voyage)
- 自动写入路径(Run 结束时把 SessionMemory 转向量索引)
- 命中重排(rerank)、混合检索(BM25+dense)
- vv 端 wiring
- LLM 主动 `vector.search` 工具暴露(留待 P9 / P10)
