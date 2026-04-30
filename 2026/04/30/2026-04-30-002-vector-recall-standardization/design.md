# 设计:向量召回标准化(Vector Recall Standardization)

> 本版基于 `design-raw.md` 经 improver 评审后产生。改动追溯见 `design-review.md`。原始草案保留为 `design-raw.md` 供审计。

## 1. 设计目标

把"向量召回"提升为 vage 的一等公民:
- **接口标准** — 让 `VectorStore` 与 `Embedder` 像 `memory.Store` 一样可插拔,接口最小化(易于 qdrant/pgvector/chroma 等真实后端落地)。
- **Builder 集成** — 提供 `VectorRecallSource`,无缝挂入现有 `vctx.Builder` 管线;source 自截预算,不依赖 Builder 的 trim 兜底。
- **下游零依赖** — 内置进程内实现 + 测试用 hash embedder,使用者不强制引入向量库。
- **可观测** — 复用 `schema.ContextSourceReport` 与 `EventContextBuilt`。

## 2. 业界实践对照

| 框架 | 接口形态 | 核心抽象 | 我们采纳 |
|---|---|---|---|
| LangChain `VectorStoreRetrieverMemory` | retriever + vectorstore 两层 | `Retriever.GetRelevantDocuments(query)` | ✅ Source 内部组合 Embedder + Store,等价于 retriever |
| MemGPT archival | 工具暴露 `archival_search` | `Document{text, embedding, metadata}` | ✅ Document 模型 + 后续可暴露为 LLM 工具 |
| OpenAI File Search | thread-bound vector store | 服务端隐式管理 | ❌ 非框架职责 |
| LlamaIndex VectorMemory | composable memory | top-k + token budget | ✅ 复用 vctx Source 的 budget 协议 |
| pgvector / qdrant / chroma | 后端类 | metadata filter + collection | ✅ `MetadataEquals` 声明式过滤可下推;collection/namespace 用多 store 实例隔离 |

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
├── vector.go                   # VectorStore, Embedder, Document, SearchHit, SearchOptions, errors
├── mapstore.go                 # MapVectorStore - 进程内 map 实现 + 余弦相似度
├── mapstore_test.go
├── embedder.go                 # EmbedderFunc 适配器, HashEmbedder(测试用)
└── embedder_test.go

vage/context/
├── sources_vector.go           # 新增:VectorRecallSource + 自截预算逻辑 + defaultQuery
└── sources_vector_test.go      # 新增:对应单测
```

`vage/context/source.go` 增加常量 `SourceNameVectorRecall = "vector_recall"`(命名风格与现有 `SourceNameSessionMemory` / `SourceNameWorkspace` 一致)。

行长上限遵守仓库规则(单文件 ≤ 800 行)。当前规划每文件远低于该上限。

## 4. 核心 API

### 4.1 `vector.Document` / `vector.SearchHit`

```go
// Package vector defines a small, pluggable vector-recall surface used by
// vage/context/VectorRecallSource and other future retrieval-driven sources.
package vector

import (
    "context"
    "errors"
    "time"
)

// Document is a single indexed item.
type Document struct {
    ID        string         // caller-supplied stable identifier
    Text      string         // the human-readable payload returned to the LLM
    Embedding []float32      // length must match the store's locked dimension
    Metadata  map[string]any // optional; used for declarative filtering
    CreatedAt time.Time      // populated by the store on Add if zero
}

// SearchHit pairs a Document with the similarity score to the query vector.
type SearchHit struct {
    Document Document
    Score    float32 // higher is more similar; cosine in MapVectorStore (range [-1,1])
}

// SearchOptions controls a Search call. The shape is intentionally minimal
// and forward-compatible: declarative MetadataEquals can be pushed down to
// real backends (qdrant must.match, pgvector @>); Predicate is a client-
// side escape hatch and may be slow on large stores.
type SearchOptions struct {
    TopK           int                          // 0 -> store default (e.g. 5)
    MinScore       float32                      // 0 -> no threshold
    MetadataEquals map[string]any               // optional declarative filter
    Predicate      func(d Document) bool        // optional client-side filter; backends MAY apply this AFTER the vector search
}
```

> **Namespace/collection 选择**:不暴露 `Namespaces []string` 字段。需要按命名空间隔离的调用者,使用多个 `VectorStore` 实例(每个 namespace 一个),与 `*sql.DB / *sql.DB-per-schema` 的 Go 习惯一致。

### 4.2 `vector.VectorStore`

```go
// VectorStore is the pluggable backend. Implementations must be safe for
// concurrent use unless documented otherwise.
type VectorStore interface {
    // Add inserts or replaces a document. The store locks the embedding
    // dimension on the first Add (or at construction via WithLockedDimension)
    // and rejects mismatched future Adds with ErrDimensionMismatch.
    Add(ctx context.Context, doc Document) error

    // Search returns up to TopK hits whose embedding is closest to query
    // under the store's similarity metric. Ordering: highest score first.
    // The query length is validated against the locked dimension.
    Search(ctx context.Context, query []float32, opts SearchOptions) ([]SearchHit, error)

    // Delete removes the document with the given ID. Implementations MAY
    // return ErrNotFound when the ID does not exist; MapVectorStore (like
    // memory.MapStore) treats missing IDs as a silent success. Callers
    // wanting strict semantics should errors.Is(err, ErrNotFound).
    Delete(ctx context.Context, id string) error

    // List returns every stored document (without scores). Useful for
    // introspection and tests; production stores may return ErrNotSupported.
    List(ctx context.Context) ([]Document, error)
}
```

错误标准化:
```go
var (
    ErrEmptyQuery        = errors.New("vector: empty query")
    ErrDimensionMismatch = errors.New("vector: embedding dimension mismatch")
    ErrNotFound          = errors.New("vector: document not found")
    ErrNotSupported      = errors.New("vector: operation not supported by backend")
)
```

### 4.3 `vector.Embedder`

```go
// Embedder maps a textual query/document to a fixed-length vector.
//
// The interface is deliberately single-method so real backends (OpenAI,
// Anthropic, voyage) can implement it with a single API call. Embedding
// dimension is validated by the VectorStore (first-Add lock or explicit
// WithLockedDimension), not exposed on Embedder — embedders may legitimately
// produce different lengths per call (e.g. OpenAI text-embedding-3 with the
// `dimensions` parameter).
type Embedder interface {
    Embed(ctx context.Context, text string) ([]float32, error)
}

// EmbedderFunc adapts a function into an Embedder.
type EmbedderFunc func(ctx context.Context, text string) ([]float32, error)

func (f EmbedderFunc) Embed(ctx context.Context, text string) ([]float32, error) {
    return f(ctx, text)
}
```

> 与 `design-raw.md` 的差异:取消 `Dimension()` 方法。维度对齐由 Store 负责;`EmbedderFunc` 简化为 `type Foo func(...)` 而非 struct,与 `http.HandlerFunc` 同形。

### 4.4 `MapVectorStore` (内置)

- 后端 `map[string]Document`,`sync.RWMutex` 保护(并发安全)。
- Search 全表扫 + 余弦相似度 + 排序 + 截 TopK + 应用 MinScore / MetadataEquals / Predicate。
- 默认 TopK = 5。
- `lockedDim`:首次 Add 设置;后续不一致返回 `ErrDimensionMismatch`。也可通过 `WithLockedDimension(d)` 在构造时显式锁定。
- 余弦实现:`dot(a,b) / (norm(a) * norm(b))`;输入零向量返回 0(避免 NaN)。

```go
func NewMapVectorStore(opts ...MapStoreOption) *MapVectorStore
type MapStoreOption func(*MapVectorStore)
func WithDefaultTopK(k int) MapStoreOption       // default 5
func WithLockedDimension(d int) MapStoreOption   // explicit dim, skips first-Add lock
```

> **性能 footgun**(交给 documenter 写进 godoc):暴力扫;N > 10k 时延迟可见。生产路径请挂真实后端。

### 4.5 `HashEmbedder`(测试用)

- 维度可配(默认 128)。
- 实现:对文本做小写 token split,token 哈希到桶,L2 归一化。
- 同文本一致输出,不同文本得到弱相关向量,够用作单测语义信号。
- 文件 doc-comment 与 `vage/.doc/vector.md` 都明确"测试用,**不**用于真实评估"。

### 4.6 `vctx.VectorRecallSource`

```go
package vctx

import (
    "github.com/vogo/vage/vector"
)

// HitsRenderer formats search hits into a single message body. The
// FetchInput is passed in so renderers can adapt by Intent / SessionID etc.
// Returning "" makes the source emit Status="skipped".
type HitsRenderer func(in FetchInput, hits []vector.SearchHit) string

// VectorRecallSource is an optional Source. It does NOT implement
// MustIncludeSource — recall is an enhancement, not a precondition.
//
// The source self-controls budget: when in.Budget > 0 it drops lowest-score
// hits and finally character-truncates the last surviving hit so the emitted
// message is guaranteed to fit. This avoids a footgun where Builder's
// trim-by-token would otherwise drop the single aggregated message entirely.
type VectorRecallSource struct {
    Store         vector.VectorStore           // required; nil -> skipped
    Embedder      vector.Embedder              // required; nil -> skipped
    TopK          int                          // 0 -> use store default
    MinScore      float32                      // 0 -> no threshold
    MetadataEquals map[string]any              // optional declarative pre-filter
    Predicate     func(vector.Document) bool   // optional pre-render filter (client side)
    Render        HitsRenderer                 // nil -> defaultHitsRender
    // QueryFn returns the query text. nil -> defaultQuery: prefer Intent,
    // else last user message with non-empty text in Request.Messages
    // (graceful fallback for nil Request / empty Messages).
    QueryFn       func(in FetchInput) string
    // MaxBytesPerHit clamps each hit's text to this many bytes before
    // rendering. 0 = unlimited. Useful guard against one outsized Document
    // consuming the whole budget.
    MaxBytesPerHit int
    // TokenEstimator overrides the default estimator used for self-trim.
    // nil -> memory.DefaultTokenEstimator.
    TokenEstimator memory.TokenEstimator
}

func (s *VectorRecallSource) Name() string { return SourceNameVectorRecall }
func (s *VectorRecallSource) Fetch(ctx context.Context, in FetchInput) (FetchResult, error)
```

**Fetch 行为流**:
1. 若 `Store == nil` 或 `Embedder == nil` → `Status=skipped`,Note 说明。
2. 计算 query 文本(`QueryFn` 或 `defaultQuery`)→ 空则 skipped。
3. `Embedder.Embed(ctx, query)` 失败 → fail-open,`Status=error`,记录 error,`InputN=0`。
4. `Store.Search(ctx, vec, opts)` 失败 → fail-open,同上。
5. 若调用者在 source 上配了 `MetadataEquals` / `Predicate` 且 store 未支持(无法预下推),source 端兜底再过滤一次。
6. 若 `MaxBytesPerHit > 0`,对每条 hit 的 text 做尾部截断(末尾追加 ` ... [truncated]`)。
7. 渲染为单条 system 消息。当 hits 为空 / 渲染为空字符串 → `Status=skipped`。
8. **预算自截**:若 `in.Budget > 0` 且 `estimateTokens(text) > in.Budget`,逐条丢分数最低的 hit 重渲后再估,直到 fit 或只剩 1 条。仍超时,对该条 text 做按字节截断到能 fit;`Status=truncated`,Note 标 `"truncated to fit budget"`。
9. 报告字段:
   - `InputN` = store 返回的候选数(过滤前)
   - `OutputN` = 1(成功时)/ 0(skipped/error)
   - `DroppedN` = `InputN - len(rendered_hits)`(过滤 + 自截丢弃合计)
   - `Tokens` = source 自报(自截后),Builder 不再重估
   - `Note` = e.g. `"hits=3 score=[0.62..0.81]"` 或 `"truncated to fit budget"`

**默认渲染格式**(可被 `Render` 覆盖):
```
## Relevant Memories
1. (score=0.81) <text>
2. (score=0.74) <text>
3. (score=0.62) <text>
```

**defaultQuery 算法**:
```go
func defaultQuery(in FetchInput) string {
    if in.Intent != "" {
        return in.Intent
    }
    if in.Request == nil || len(in.Request.Messages) == 0 {
        return ""
    }
    // Walk backwards for the last user message with non-empty text.
    for i := len(in.Request.Messages) - 1; i >= 0; i-- {
        m := in.Request.Messages[i]
        if m.Role == aimodel.RoleUser {
            if t := extractText(m); t != "" {
                return t
            }
        }
    }
    return ""
}
```

**预算交互**:Source **自截到 ≤ Budget**(见 Fetch 步骤 8),因此 Builder 的 trim 兜底不会触发。预算分配仍按声明顺序消费——把 VectorRecallSource 放在 SessionMemorySource 之后即可享受其消费完后的剩余预算。

## 5. 集成点

```go
import (
    vctx "github.com/vogo/vage/context"
    "github.com/vogo/vage/vector"
)

store := vector.NewMapVectorStore()
emb := vector.NewHashEmbedder(128)

// 由调用者负责把素材索引进 store(本期不绑定写入路径)
vec, _ := emb.Embed(ctx, "User prefers dark mode")
_ = store.Add(ctx, vector.Document{
    ID: "doc-1", Text: "User prefers dark mode",
    Embedding: vec,
})

builder := vctx.NewDefaultBuilder(
    vctx.WithSource(&vctx.SystemPromptSource{Template: tmpl}),
    vctx.WithSource(&vctx.SessionMemorySource{Manager: memMgr}),
    vctx.WithSource(&vctx.VectorRecallSource{
        Store:    store,
        Embedder: emb,
        TopK:     5,
        MinScore: 0.2,
        MaxBytesPerHit: 1024,
    }),
    vctx.WithSource(&vctx.RequestMessagesSource{}),
)
```

> **建议位置**:`SessionMemorySource` **之后**(吃完会话历史剩余预算)、`RequestMessagesSource` **之前**(保持时间顺序:历史 → 召回背景 → 当前请求)。

## 6. 不变量与边界

| 不变量 | 说明 |
|---|---|
| Embedder 与 Store 维度一致 | Store 在首次 Add 锁定;Search 用 query 长度做即时校验 |
| Source 永不抛错 | 全部错误 fail-open,Builder 不被打断 |
| Source 不修改任何输入参数 | Document / FetchInput 都是值传递或只读 |
| 返回的 messages 始终 ≤ 1 条 | 简化下游装箱;hits 内部多条聚合 |
| **Source 自报 Tokens 严格 ≤ Budget(Budget>0 时)** | 自截算法保证;Builder trim 不会触发 |
| Render 返回空 = skipped | 与 SessionStateSource 一致的语义 |

## 7. 测试计划

### 7.1 单元测试

`vage/vector/mapstore_test.go`:
- `TestMapVectorStore_AddSearch` — 三个文档,query 余弦最近的命中第一
- `TestMapVectorStore_DimensionLock` — 第二条 Add 维度不一致返回 `ErrDimensionMismatch`;Search query 长度也校验
- `TestMapVectorStore_TopK_MinScore_MetadataEquals_Predicate` — 四种过滤分别生效与组合生效
- `TestMapVectorStore_Delete_MissingIsSilent` — 删除不存在 ID 不报错
- `TestMapVectorStore_Concurrent` — `go test -race` 单写多读不竞争
- `TestMapVectorStore_ZeroVectorCosine` — 零向量返回 0,不 NaN

`vage/vector/embedder_test.go`:
- `TestEmbedderFunc_Roundtrip`
- `TestHashEmbedder_Stability` — 同文本同向量
- `TestHashEmbedder_RoughSemantic` — 共享词的两个文本余弦 > 完全不同的两个文本
- `TestHashEmbedder_Normalized` — 输出 L2 范数 ≈ 1

`vage/context/sources_vector_test.go`:
- `TestVectorRecallSource_HappyPath`
- `TestVectorRecallSource_SkippedOn_NilStore / NilEmbedder / EmptyQuery / NilRequest / NoHits`
- `TestVectorRecallSource_FailOpen_EmbedError / SearchError`
- `TestVectorRecallSource_TopK / MinScore / Predicate / MetadataEquals`
- `TestVectorRecallSource_QueryFn_FallbackToLastUserMsg` — 包含倒序找带文本 user message 与跳过 tool_result 的子用例
- `TestVectorRecallSource_QueryFn_NilRequest_Graceful` — Request==nil 时 skipped
- `TestVectorRecallSource_SelfTrim_DropsLowestScore` — Budget < tokens 时丢分数最低 hit
- `TestVectorRecallSource_SelfTrim_FinalCharTruncate` — 只剩 1 条仍超 budget 时按字节截断,`Status=truncated`
- `TestVectorRecallSource_MaxBytesPerHit` — 单条 hit 被截断到上限
- `TestVectorRecallSource_RendererSeesFetchInput` — 自定义 Render 收到 FetchInput
- `TestVectorRecallSource_BuilderIntegration` — 与 DefaultBuilder 串起来,验证 `ContextBuiltData.Sources` 包含 vector_recall 报告且 Builder trim 不触发

### 7.2 集成测试

不新增;`integrations/` 中现有 taskagent 测试不被破坏(本期不修改 TaskAgent 默认 source 列表)。

## 8. 文档变更

- 新增 `vage/.doc/vector.md` 一份完整模块文档(接口、内置实现、使用示例、与 context/memory 的关系、未来扩展、footgun warning)。
- 更新 `vage/.doc/context.md`:§3 内置 Source 表格补 `VectorRecallSource` 行;§9.2 / §10 把 VectorRecallSource 从"留作后续"清单移除。
- 更新 PRD `doc/prd/models/core/memory/`:新增 `model-vector-store.md` 与 `model-vector-document.md` 两份模型卡。
- 回写 `doc/design/session-context-solution.md`:§4.2 / §8 / 末尾 roadmap 三处状态更新。

## 9. 与原始草案 §4 演进路径的关系

本次落地对应 §3 P3 与 §4.2"留作后续"清单中的 `VectorRecallSource`。完成后 §8 现状汇总:
- 「向量召回」:⚠️ 依赖 store 实现 → ✅ 标准 VectorRecallSource + MapVectorStore + Embedder 接口
- 末尾 roadmap"向量召回标准化"勾选完成

§4.8.3 树驱动 context 算法中 `vector_recall(intent, scope=non_path_nodes, top_k=5)` 的 `scope` 参数将由 SessionTree 调用方通过自定义 `Predicate` (针对 `Metadata["node_id"]` 排除 path 节点)实现,**不**侵入本期接口。

## 10. 风险与权衡

| 风险 | 处理 |
|---|---|
| 余弦相似度对未归一化向量结果不稳定 | MapVectorStore 在 Search 内部做归一化,而非要求调用者预归一 |
| HashEmbedder 太弱可能误导用户 | 文件 doc-comment + `vage/.doc/vector.md` 显式标 "test only" |
| 后续真实后端语义可能与 MapVectorStore 不同(如 score scale) | 接口契约说明:Score 单调递增即可,具体语义由后端定义;`MinScore` 由调用者按后端调 |
| `Predicate` 闭包无法下推到真实后端 | godoc 显式标 "may be slow on large stores; backends MAY apply this AFTER vector search";调用者优先用 `MetadataEquals` |
| 召回结果泄露敏感内容 | 不在本期决策;guard 包负责 output 侧拦截 |
| 维度不一致难调试 | Add 立即返回 `ErrDimensionMismatch`,Search 用 query 长度即时校验 |
| 单条聚合消息超预算导致 Builder trim 整条丢弃 | Source 自截:逐条丢低分 hit → 字节截断兜底,保证 `Tokens ≤ Budget`(详见 §4.6 Fetch 步骤 8) |
| 调用者忘了配 `MaxBytesPerHit`,一条超长 Document 吃完整个预算 | 自截算法在只剩 1 条仍超预算时会按字节截断该条;`MaxBytesPerHit` 是更早的护栏(option) |

## 11. 不在本期实现(明示)

- 真实后端(qdrant/pgvector/chroma)
- 真实 Embedder(OpenAI/Anthropic/voyage)
- `BatchEmbed` / `ModelName()` 等 Embedder 扩展
- 自动写入路径(Run 结束时把 SessionMemory 转向量索引)
- 命中重排(rerank)、混合检索(BM25+dense)
- vv 端 wiring
- LLM 主动 `vector.search` 工具暴露(留待 P9 / P10)
- `BudgetAllocator` 接口(`vctx.Builder` 装箱算法保持 ordered_greedy)
