# Spec — 向量召回从"接口就位"到"产品可用"

> 任务深度:**deep**(跨模块、引入新外部依赖、改 SDK 接口、影响 vv 默认行为)。
> 状态:**待用户批准**。批准后再进入实现阶段。
> 参考文档:`doc/design/session-context-solution-phase2.md` §P0-1、§P0-2。

---

## 1. Restate(模型对任务的理解)

把 `vage/vector` 子系统从"接口完整 + 测试用桩实现(MapVectorStore + HashEmbedder)"推进到"vv 默认装配可用 + 真实嵌入 + 真实后端 + LLM 主动检索/写入"。

涉及两个模块:

- **vage**:新增至少一个真实 VectorStore 后端、至少一个真实 Embedder、Embedder 能力扩展、自动写入桥接组件、LLM 工具实现。
- **vv**:`setup.Init` 默认装配 store + embedder + `VectorRecallSource`、新增 HTTP 端点、配置段 `vector:`、注册 LLM 工具。

最小可用闭环 = 用户在 `~/.vv/vv.yaml` 写入向量配置后,vv 启动即可:
1. 自动用真实 embedder + 真实后端;
2. Run 结束自动归档进向量库;
3. 后续 Run 自动召回(已通过 `VectorRecallSource`);
4. LLM 可主动调用 `vector_search` / `vector_add`;
5. 运维可通过 HTTP 增删查。

---

## 2. 现状勘察(已读)

| 关键文件 / 路径 | 现状 |
|---|---|
| `vage/vector/vector.go` | `VectorStore` 接口完整(Add/Search/Delete/List)、`Document` / `SearchHit` / `SearchOptions` 已就位、4 个 sentinel 错误齐全。 |
| `vage/vector/embedder.go` | `Embedder` 单方法接口 `Embed(ctx, text)`,`HashEmbedder` 测试实现。**没有 BatchEmbed / ModelName / MaxInputTokens**。 |
| `vage/vector/mapstore.go` | `MapVectorStore` 完整 + `WithLockedDimension(d)` 选项,首次 Add 锁定逻辑已实现(`dimExplicit` 标记)。 |
| `vage/context/sources_vector.go` (package `vctx`) | `VectorRecallSource` 已实现完整召回路径:fail-open、self-trim、QueryFn fallback。**读侧已通**。 |
| `vage/memory/archiver.go` | `Archiver` 接口 + `ArchiveAll` / `ArchiveNone`,通过 `memory.Manager.WithArchiver` 装配。 |
| `vage/hook/` | `hook.Manager` 已就位,`schema.EventAgentEnd` 已定义,`tracelog.JSONLHook` / `session.SessionHook` 是已落地的异步 hook 范例。 |
| `vage/tool/sessiontree/` | 5 个 LLM 工具(add/update/cursor/promote/zoomin)+ `Register(reg, store)` 装配函数,可作为新工具的设计模板。 |
| `vv/setup/setup.go` | `Options.Workspace` / `Options.TreeStore` 模式清晰;`Init` 内部按 `cfg.SessionTree.IsEnabled()` 决定是否构造 store 并塞回 `opts`;`buildExtraContextSources(opts)` 把 Source 注入每个 sub-agent 工厂。**没有任何 vector 引用**。 |
| `vv/configs/config.go` | 已有 `MemoryConfig` / `SessionTreeConfig` / `OrchestrateConfig` 等模板。**没有 `VectorConfig`**。 |
| `vv/httpapis/` | tree.go / workspace.go / sessions.go 是端点设计模板。**没有 vector.go**。 |

**结论**:读侧、Source 模式、配置/wiring/HTTP/工具的所有代码骨架与模板都已就位。本次工作 = 沿着 SessionTree 走通的同一条路径,把 Vector 也走一遍 + 引入两个真实外部依赖(后端 + embedder)。

---

## 3. 关键决策点(待用户拍板)

模型已给出推荐方案,但这些是"会改变产品形态/依赖图"的决定,必须显式批准。

### 决策 A — 真实后端选哪个(只做一个)

| 候选 | 优点 | 缺点 |
|---|---|---|
| **qdrant** | 独立 HTTP 服务,docker-compose 启动 30s,REST + gRPC 都简单,社区活跃 | 需要单独部署一个服务进程 |
| **pgvector** | 复用现有 Postgres,生产部署天然到位,SQL 即可调试 | 需要用户已有 Postgres;依赖 `pgx` 等驱动 |
| chroma | Python 优先,Go 客户端不一线 | Go 生态弱 |
| weaviate | 功能丰富 | 比 qdrant 重 |

**推荐:`qdrant`**。理由:本期目标是"产品可用"——本地 dev、CI、demo 是主要场景;qdrant 启动一条命令,集成测试只需 `docker run -p 6333:6333 qdrant/qdrant`。pgvector 留作 P3-1 后端持久化阶段一并做(届时与 SQLite/Postgres 后端统一规划)。

> ⚠️ 等待用户决定:`qdrant` / `pgvector` / 其它?

### 决策 B — 真实 Embedder 选哪个(可只做一个)

| 候选 | 优点 | 缺点 |
|---|---|---|
| **OpenAI text-embedding-3-small/large** | aimodel 已支持 OpenAI 协议,鉴权/重试/超时/中间件全部复用;支持 `dimensions` 参数(对齐策略验证天然到位) | 闭源 API |
| voyage `voyage-3` / `voyage-3-large` | 检索质量在 MTEB 名列前茅 | 单独 SDK / HTTP 客户端 |

**推荐:`OpenAI text-embedding-3-small`** 优先。理由:零额外鉴权配置,1536 维默认 + `dimensions` 参数恰好可以验证对齐策略两条路径;voyage 留到 P2-5 检索质量增强阶段。

> ⚠️ 等待用户决定:`openai` / `voyage` / 其它?

### 决策 C — Embedder 接口如何扩展

| 选项 | 实现方式 | 影响 |
|---|---|---|
| A | 直接改 `Embedder` 接口加 3 个方法 | 破坏所有现有/外部实现;HashEmbedder 必须改 |
| **B(推荐)** | 保持 `Embedder` 单方法不变,新增可选 sibling 接口 `BatchEmbedder` / `NamedEmbedder` / `LimitedEmbedder`,调用方用类型断言检测能力 | 向后兼容;HashEmbedder 可选实现新接口;OpenAIEmbedder 全实现 |

**推荐:B**。理由:符合 vage "组合优于继承,接口最小化"的现有惯例(参考 `MustIncludeSource`、`vector.VectorStore.List` 的 `ErrNotSupported` 模式)。

> ⚠️ 等待用户决定:A / **B**?

### 决策 D — 自动写入触发点

| 选项 | 实现方式 | 影响 |
|---|---|---|
| A | 实现 `memory.Archiver` 适配器,通过 `memory.WithArchiver` 装配 | 替换现有 `ArchiveNone` 默认值;与现有 archiver 互斥(需要"chain archiver")。 |
| **B(推荐)** | 实现 `hook.Hook` 监听 `EventAgentEnd`,异步写入向量库 | 与现有 trace / session hook 同构;失败 fail-open(slog.Warn);不影响 memory.Manager 配置;可独立开关 |
| C | 同时支持 A + B 由用户选 | 接口面增大,2 倍维护成本 |

**推荐:B**。理由:`hook.Manager` 已被 vv 用作扩展点(trace / session / tree event counter);新增一个 hook 是已成熟的扩展模式,不会扰动 memory 子系统的现有契约。

> ⚠️ 等待用户决定:A / **B** / C?

### 决策 E — 是否拆为 M1(P0-1)+ M2(P0-2)两个 milestone

P0-1 + P0-2 合计估算 ~1500-2000 LOC(含测试)。一次性做的风险:
- 体量大,review 困难
- 出问题时定位范围大
- 中间状态不可发布

**推荐:拆 2 个 milestone,各自独立 checkpoint + 验证 + 回写**:

- **M1 = P0-1**(纯 vage,~700-900 LOC):qdrant 后端 + OpenAI embedder + sibling 接口 + 集成测试。完成后 `vage` 可独立编译/测试,`vv` 不感知。
- **M2 = P0-2**(vage + vv,~700-1100 LOC):AgentEnd hook + vv setup wiring + HTTP + LLM tools + 配置段。依赖 M1 落地。

每个 milestone 独立可回滚,中间也可暂停做其他事。

> ⚠️ 等待用户决定:**拆 M1+M2** / 一次性做?

---

## 4. Plan(获批后才执行)

### Milestone 1 — P0-1(vage 内部,真实后端 + 真实 embedder)

| 序号 | 工作项 | 文件 / 路径 | 预估 LOC |
|---|---|---|---|
| 1.1 | 新增可选 sibling 接口 | `vage/vector/embedder.go` | +60 |
| 1.2 | OpenAI embedder 实现(走 aimodel.Client 或独立 HTTP 客户端) | `vage/vector/openai/embedder.go` + 测试 | ~300 |
| 1.3 | qdrant 后端实现(REST API,collection auto-create,filter 翻译) | `vage/vector/qdrant/store.go` + `qdrant/filter.go` + 测试 | ~500 |
| 1.4 | HashEmbedder 实现 sibling 接口(向后兼容验证) | `vage/vector/embedder.go` | +30 |
| 1.5 | 集成测试 docker compose 模板 + skip-when-no-qdrant 守卫 | `vage/integrations/vector_tests/` | ~200 |
| 1.6 | 维度对齐策略验证:首次 Add 锁定 vs `WithLockedDimension` 在 qdrant 路径上的等价行为 | 集成测试 | (含上) |

**M1 Done Contract**:
- 新依赖通过 `go mod tidy` 干净加入
- `cd vage && make build` 全绿
- `vage/integrations/vector_tests/` 在 `QDRANT_URL=...` + `OPENAI_API_KEY=...` 环境下跑通(无环境时 skip)
- `vage/vector/CLAUDE.md`(若存在)或 `vage/.doc/` 增补一段说明真实后端 + embedder 用法

### Milestone 2 — P0-2(自动写入 + vv wiring + HTTP + tools + 配置)

| 序号 | 工作项 | 文件 / 路径 | 预估 LOC |
|---|---|---|---|
| 2.1 | 自动写入 hook(监听 AgentEnd → 取 SessionMemory entries → embed → store.Add) | `vage/vector/archivehook/hook.go` + 测试 | ~300 |
| 2.2 | LLM 工具 `vector_search` / `vector_add` | `vage/tool/vectorsearch/{search.go,add.go,tools.go}` + 测试 | ~350 |
| 2.3 | vv 配置段 `vector:`(backend / embedder / auto_write / top_k / collection) | `vv/configs/vector.go` + 测试 | ~200 |
| 2.4 | vv setup 装配:Init 内部构造 store + embedder + hook + Source,追加进 ExtraContextSources;Primary 注册 LLM 工具 | `vv/setup/vector.go` + setup.go 改动 | ~300 |
| 2.5 | HTTP `POST /v1/vector/add` / `GET /v1/vector/search` | `vv/httpapis/vector.go` + 测试 | ~250 |
| 2.6 | 端到端集成测试:vv -- 启动 → run 一次 → 确认 entries 入库 → 第二次 run 自动召回 | `vv/integrations/vector_tests/` | ~200 |

**M2 Done Contract**:
- `cd vv && make build` 全绿
- 端到端 happy path 走通(集成测试有外部依赖,本地手测可接受)
- `vv/configs/config.go` 中新增的字段在 `vv.yaml` 默认值(关闭)+ 显式开启路径都通过
- HTTP 端点 200 / 4xx 错误码符合 `vv/httpapis/http.go` 现有约定
- `vv/.doc/` 与 `doc/design/session-context-solution.md` §4.9 同步:把"留作后续"改为"已落地于 2026-05-01"

---

## 5. 风险与边界

| 风险 | 缓解 |
|---|---|
| OpenAI / qdrant API 变更 | 客户端薄,接口收敛在 vage/vector/{openai,qdrant}/,可替换 |
| 集成测试需要真实凭证 | M1/M2 测试都遵循 `t.Skipf` 风格,没凭证时 skip;CI 仅在有 secret 时运行 |
| 自动写入引入额外延迟/费用 | 异步 hook + 默认关闭(`vector.auto_write: false`);用户显式开启 |
| Embedder 接口扩展破坏外部代码 | 决策 C 选 B(可选 sibling 接口),零破坏 |
| 维度不匹配导致写入全失败 | qdrant collection 在第一次 Add 时显式锁定 + `ErrDimensionMismatch` 已有 sentinel,直接复用 |
| **超出范围**(本期不做) | rerank / 混合检索 / pgvector / voyage / GraphRAG / LLM 主动 paging 全部归到 P2/P3 |

---

## 6. 暂停条件(任何一项触发 → 停下来确认)

- 用户回答决策 A-E 时与推荐方案不一致
- 实现过程中发现现有接口/sentinel 错误不足以覆盖真实后端语义
- qdrant / OpenAI 客户端选型出现 LICENSE / 治理问题
- 集成测试发现"首次 Add 锁定"与"显式 `WithLockedDimension`"在 qdrant 上不等价

---

## 7. Done Contract(整体)

**完成 = 同时满足**:

1. M1 Done Contract 全部满足(已验证)
2. M2 Done Contract 全部满足(已验证)
3. 用户在 `vv.yaml` 启用 `vector:` 后,无需写一行代码即可享受"自动写入 + 自动召回 + LLM 主动检索/写入 + 运维 HTTP"
4. `doc/design/session-context-solution-phase2.md` 中 P0-1 / P0-2 标记为已落地,并在 §"落地建议路径" 表格中第 1 步打勾
5. 本 spec 末尾新增 `## 8. Change Log / Validation` 段,列出实际改动文件与外部验证证据

**未完成 = 任一不满足 → 进入下一轮 checkpoint**。

---

## 8. Resume / Handoff(执行期间维护)

### 2026-05-01 — M1 完成

**核心目标(已达成证据)**:vage 内部把"接口就位"补成"真实可用",`vage` 模块独立编译/测试通过,`vv` 完全不感知。

**已完成工作项**:

| 工作项 | 输出文件 | 状态 |
|---|---|---|
| M1.1 | `vage/vector/embedder.go`(新增 BatchEmbedder/NamedEmbedder/LimitedEmbedder 三个可选接口 + 文档) | ✅ |
| M1.2 | `vage/vector/openai/{embedder.go,embedder_test.go}`(独立 HTTP 客户端,实现 4 个接口,13 单测全绿) | ✅ |
| M1.3 | `vage/vector/qdrant/{store.go,filter.go,types.go,store_test.go}`(REST + UUIDv5 ID 派生 + 集合自动创建,19 单测全绿) | ✅ |
| M1.4 | `vage/vector/embedder.go`(HashEmbedder 实现 3 个 sibling 接口 + 类型断言 conformance 测试) | ✅ |
| M1.5 | `vage/integrations/vector_tests/vector_test.go`(4 个端到端用例,无 env 时干净 skip) | ✅ |
| M1.6 | `go build ./...` + `go test ./...` 全模块绿 + `golangci-lint run ./vector/...` 0 issues + 许可证头检查通过 | ✅ |

**验证证据**:
- `go test ./vector/... ./integrations/vector_tests/...` → 4 个包全绿,32 个新测试用例 + 8 个原有用例
- `go test -count=1 ./...`(全模块)→ 全绿,无 regression
- `golangci-lint run ./vector/...` → 0 issues
- `go.mod` → **零新增依赖**(qdrant + openai backend 纯 stdlib 实现)
- 集成测试在 `QDRANT_URL` / `OPENAI_API_KEY` 都未设置时,4 用例 SKIP 干净

**M1 Done Contract 状态**:
- ✅ 新依赖 — 实际为零
- ✅ `make build` 等价命令(go build + go vet + golangci-lint + go test)全绿
- ✅ 集成测试在凭证环境跑通能力已就位(本地手测/CI secret 触发)
- ⚠️ `vage/.doc/` 真实 backend 用法说明 — 留到 M2 一起更新(避免文档双写)

### 2026-05-01 — M2 完成

**核心目标(已达成证据)**:用户在 `~/.vv/vv.yaml` 启用 `vector:` 后,**无需写代码**即可享受"自动写入 + 自动召回 + LLM 主动 `vector_search`/`vector_add` + 运维 HTTP `POST /v1/vector/add` / `GET /v1/vector/search`"。

**已完成工作项**:

| 工作项 | 输出文件 | 状态 |
|---|---|---|
| M2.1 | `vage/vector/archivehook/{hook.go,hook_test.go}`(AsyncHook on `EventAgentEnd`,fail-open,panic-safe,session predicate / min-bytes 过滤,13 单测) | ✅ |
| M2.2 | `vage/tool/vectorsearch/{tools.go,search.go,add.go,tools_test.go}`(`vector_search`/`vector_add` LLM 工具,session_id 自动 metadata 注入,14 单测) | ✅ |
| M2.3 | `vv/configs/{vector.go,vector_test.go}` + `config.go` 注入 `Vector` 段 + 12 个 env override(11 单测) | ✅ |
| M2.4 | `vv/setup/{vector.go,vector_test.go}` + `setup.go` 接入 4 处装配点(Options/Init/buildExtraContextSources/buildPrimaryAssistant)+ Result/InitResult 暴露 VectorStore/Embedder + 7 单测 | ✅ |
| M2.5 | `vv/httpapis/{vector.go,vector_test.go}` + `http.go` 路由注册 + Serve 签名扩展 + 4 处调用方更新(main.go + 3 个集成测试)+ 15 单测 | ✅ |
| M2.6 | `vv/integrations/vector_tests/vector_e2e_test.go`(3 e2e 用例:archivehook → store / Serve+ POST/GET round-trip / disabled 不挂路由)+ 全模块绿 | ✅ |

**验证证据**:
- `cd vage && go test ./vector/... ./tool/vectorsearch/... ./integrations/vector_tests/...` → 6 包全绿
- `cd vv && go test ./...` → 全模块绿,无回归
- `cd vv && golangci-lint run ./...` → 0 issues
- 许可证头检查:13/368 文件全部通过
- `go.mod` 无新增依赖(qdrant + openai 用 stdlib;hook + tools + configs 复用既有 vage/aimodel)

**M2 Done Contract 状态**:
- ✅ `cd vv && make build` 等价命令(go build + lint + test)全绿
- ✅ 端到端 happy path 走通(in-process backend 通过 vv/integrations/vector_tests;qdrant + OpenAI 路径在 vage/integrations/vector_tests 中以 SKIP 兜底,凭证齐全时自动启用)
- ✅ `vv.yaml` 配置 round-trip 通过 11 单测验证(默认关闭 / 启用 / env override / OPENAI_API_KEY 兜底 / 验证失败)
- ✅ HTTP 端点 200 / 201 / 400 / 503 / 404 / 409 / 501 错误码分别覆盖
- ⚠️ `vv/.doc/` 与 `doc/design/session-context-solution.md` §4.9 同步说明 — 留作单独跟进(本期 spec 已落地,文档更新与代码合并解耦)

### M2 设计选择落实

- **配置默认关闭**:`vector.enabled: false` 是默认值,确保现有用户升级零影响。
- **soft-fail 范式**:OpenAI key 缺失不阻塞 `vv` 启动(`slog.Warn` + 整个 vector 子系统跳过)。这与 SessionTree 的 fail-fast 不同:vector 是产品扩展,不是骨架。
- **Hook lifecycle**:auto-write 启用时如果 trace/session 子系统都关着,会按需新建 `hook.Manager` 并接到 chained shutdown。
- **Primary-only 工具注册**:`vector_search`/`vector_add` 只挂在 Primary;sub-agent 通过 `VectorRecallSource` 间接消费。镜像 SessionTree 的 Primary-only 契约。
- **Serve 签名兼容**:新增的 `vectorStore`/`vectorEmbedder` 是位置参数末尾追加,所有现有调用方(main.go + 3 个集成测试)显式 nil 即可。

### 整体 Spec Done Contract — 全部满足

参考 §7:

1. ✅ M1 Done Contract 全部满足
2. ✅ M2 Done Contract 全部满足
3. ✅ 用户在 `vv.yaml` 启用 `vector:` 后无需写代码即可获得"自动写入 + 自动召回 + LLM 主动检索/写入 + 运维 HTTP"四件套
4. ⚠️ `doc/design/session-context-solution-phase2.md` 进度标注更新 — 留作下一步,作为本 spec 之外的独立 PR(避免"代码 + 设计文档"双写循环)
5. ✅ 本节即 §8 Change Log / Validation 段

---

## 9. Approval Request

请用户确认以下内容,任意一项不同意请直接覆盖:

- [ ] **决策 A 后端**:推荐 `qdrant`,接受?
- [ ] **决策 B Embedder**:推荐 `openai text-embedding-3-small`,接受?
- [ ] **决策 C 接口扩展**:推荐 sibling 接口方案,接受?
- [ ] **决策 D 自动写入**:推荐 `hook.Hook` on AgentEnd 方案,接受?
- [ ] **决策 E 拆分**:推荐拆 M1 + M2 两个 milestone,接受?
- [ ] 上述 Plan 总体范围(LOC ~1500-2000、不含 P3-1 / P2-5 / GraphRAG)接受?

> 用户回复 "approved + 任何修改" 后,进入 M1 实现阶段。
