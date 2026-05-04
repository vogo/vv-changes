# Phase 2 第 2 步 —— 实施计划

> 配套 [`spec.md`](./spec.md) 使用。先 spec 后 plan,先 vage 后 vv,先单测后集成。

---

## A. 顺序与依赖

```
P0-3 Resume                    P0-4 ContextEditor             P0-5 Observability
─────────────                   ──────────────────              ───────────────────
A1. setup IterationStore        B1. ResourceTracker 协议       C1. SessionMetrics 模型
A2. CLI --resume                B2. fs 工具 ResourceID 标注    C2. SessionMetricsHook
A3. HTTP /resume                B3. stale 检测 + 折叠         C3. BuildReport 落盘
A4. Resume 后指标 +1 (→C2)      B4. 单条转引用 + artifacts/   C4. checkpoint 失败计数
                                                              C5. HTTP /metrics + /build-reports
```

P0-3 与 P0-4 完全独立。P0-5 的 C2 依赖 P0-3 的 Resume 路径(用于 `+resume_count`),其余子项与 P0-3/P0-4 解耦。

建议执行顺序: **P0-3 → P0-5(C1-C4) → P0-4 → P0-5(C5)**。原因: P0-4 改动最大且是 vage/tool 协议层的扩展,放在最稳的两块之后做最安全;P0-5 的 HTTP 路由放最后一并加,避免后续 P0-4 增加的 elided 计数还要回头改 HTTP layer。

---

## B. 步骤明细

### A. P0-3 Resume vv wiring

#### A1. `vv/setup/setup.go` —— 装配 IterationStore

- 在 `buildHookManagerAndSession` 之后、`New` 之前: 当 `cfg.Session.IsEnabled()` 时构造 `checkpoint.NewFileIterationStore(<sessionRoot>)`,塞进 `Options` 新字段 `IterationStore`。
- `registries.FactoryOptions` 新增字段 `IterationStore checkpoint.IterationStore`;`agents/coder.go|researcher.go|reviewer.go|primary.go` 在构造 TaskAgent 时透传 `taskagent.WithIterationStore(opts.IterationStore)`。
- `InitResult` 新增 `IterationStore checkpoint.IterationStore` 暴露给 main.go。

**文件**:
- `vv/setup/setup.go` (修改; +30 行)
- `vv/setup/checkpoint.go` (新建; ~40 行,含 helper `buildIterationStore(cfg)` + 单测)
- `vv/registries/registry.go` (修改; +3 行)
- `vv/agents/coder.go` / `researcher.go` / `reviewer.go` / `primary.go` (修改; 每个 +3 行)

#### A2. `vv/cli/` —— `--resume` 入口

- `vv/main.go`: 新增 `resumeFlag := flag.String("resume", "", "session id whose latest checkpoint to resume")`;与 `--session` / `--p` / `--eval` / `http`/`mcp` 互斥校验。
- `vv/cli/resume.go` (新建): `func RunResume(ctx, initResult, sessionID, out, errW) error`,负责:
  1. `iterStore.List(ctx, sid)` → 取最新 checkpoint
  2. 校验 `cp.Final == false`(否则返回 `ErrAlreadyFinal` 友好提示)
  3. 用 `cp.AgentID` 在 `subAgents` 中找对应 agent;同时支持 Primary(`dispatcher.PrimaryAgentID()` 等价 ID)
  4. 调用 `agent.(*taskagent.Agent).Resume(ctx, sid)`,把 RunResponse 渲染输出
  5. 累加 `metrics.resume_count` (调用 SessionMetrics.RecordResume,见 C2)

**文件**:
- `vv/main.go` (修改; +30 行)
- `vv/cli/resume.go` (新建; ~120 行)
- `vv/cli/resume_test.go` (新建; ~150 行;用 MapIterationStore + 假 agent)

#### A3. HTTP `POST /v1/sessions/{id}/resume`

- `vv/httpapis/http.go` `Serve` 签名增加 `iterationStore checkpoint.IterationStore`(可 nil),路由 `mux.HandleFunc("/v1/sessions/", ...)` 中分支识别 `/resume` 后缀,委派给 `vv/httpapis/resume.go` 的 `handleResumeSession`。
- `vv/httpapis/resume.go` (新建):
  - `POST /v1/sessions/{id}/resume` 同步 → 返回 `RunResponse` JSON;
  - `POST /v1/sessions/{id}/resume?stream=1` → SSE,事件流复用 dispatcher 现有的 stream 写法;
  - 错误映射: `ErrAlreadyFinal` → 409 + `{"error":"...","code":"already_final"}`,`ErrCheckpointNotFound` → 404,其余 500。
- `vv/main.go` 调用 `httpapis.Serve(...)` 处增加 `initResult.IterationStore` 实参。

**文件**:
- `vv/httpapis/http.go` (修改; +5 行)
- `vv/httpapis/resume.go` (新建; ~180 行)
- `vv/httpapis/resume_test.go` (新建; ~150 行)

#### A4. Resume 路径调用 SessionMetrics(放在 C2 之后接入)

- 见 C2 结尾。

---

### B. P0-4 ContextEditor 增强

#### B1. `vage/tool/resource.go` (新建) —— ResourceTracker 协议

```go
package tool

// ResourceTracker is implemented by tools that read or write identifiable
// resources (files, db rows, etc.). The Editor uses ResourceID + Mode to
// detect stale tool_results.
type ResourceTracker interface {
    // ResourceIDs returns the canonical IDs touched by this invocation.
    // Returns nil for invocations that don't touch any tracked resource.
    ResourceIDs(args map[string]any) []ResourceRef
}

type ResourceRef struct {
    ID   string         // canonical id, e.g., absolute file path
    Mode ResourceMode   // "read" or "write"
}

type ResourceMode string
const (
    ResourceRead  ResourceMode = "read"
    ResourceWrite ResourceMode = "write"
)
```

- `tool.Tool` 接口扩展(可选): 通过类型断言而非强制实现。
- 单测: 接口形态、Mode 常量、ResourceRef 序列化。

**文件**:
- `vage/tool/resource.go` (新建; ~80 行)
- `vage/tool/resource_test.go` (新建; ~60 行)

#### B2. `vage/tool/fs/` 工具挂 ResourceTracker

- `fs.ReadFile` / `fs.WriteFile` / `fs.EditFile` 的 Handler struct 实现 `ResourceTracker.ResourceIDs(args)` 返回 `{ID: filepath.Clean(absPath), Mode: read/write}`。
- bash 工具不挂(命令任意,无法静态识别资源)。
- 单测: 各工具调用一次,`ResourceIDs` 返回正确 ref。

**文件**:
- `vage/tool/fs/read.go` 等 (修改; 每个 +15 行)
- `vage/tool/fs/resource_test.go` (新建; ~80 行)

#### B3. `vage/largemodel/context_editor.go` —— stale 检测

- 在 `ContextEditorMiddleware` 加字段:
  - `resourceTrackerFn func(toolName string) ResourceTracker`(由调用方注入,默认 nil 关闭 stale 模式)
  - `staleStrategy string = "stale_resource"`
- 新方法 `scanStaleByResource(msgs)`: 单遍扫描 messages,维护 `resourceID → latestWriteToolCallID` 表,任何在 latestWrite 之前的 read 类 tool_result 标 stale。
- `applyElision` 路径合并: 一条消息可能同时被 keep_last_k 与 stale 命中,**stale 优先**(占位符更明确)。
- 占位符函数升级: `PlaceholderFunc(toolCallID, originalBytes, reason string) string`,`reason` 取值 `"keep_last_k" | "stale_resource" | "elide_to_artifact"`。**这是一个公开 API 的破坏性变更**,需要在 `vage/.doc/largemodel.md` 标注;旧的 `WithPlaceholder` 签名保留为废弃 alias 外加 `WithPlaceholderV2` 新函数。
- 配置: `WithStaleResourceTracker(fn)` 新 option。

**文件**:
- `vage/largemodel/context_editor.go` (修改; +120 行,接近 360 行,**仍 ≤ 800 行 OK**)
- `vage/largemodel/context_editor_stale_test.go` (新建; ~200 行)

#### B4. 单条转引用 + workspace.Artifact

- `vage/workspace/workspace.go` 接口扩展:
  ```go
  WriteArtifact(ctx, sid, name string, content []byte) (path string, err error)
  ReadArtifact(ctx, sid, name string) ([]byte, error)
  ```
- `vage/workspace/filestore.go`: artifacts 存到 `<root>/<sid>/workspace/artifacts/<name>`;不进 `notes/` index;不计入 MaxNoteCount。
- `vage/largemodel/context_editor.go` 加 option:
  ```go
  WithMaxBytesPerMessage(n int) ContextEditorOption          // 单条阈值
  WithArtifactWriter(w ArtifactWriter) ContextEditorOption   // 注入 workspace
  ```
- `ArtifactWriter` 是接口 `Write(sid, name, content) (path string, err error)`,适配器在 vv 装配时把 workspace 包装进来。
- 单测三种 case: 不超阈值不动 / 超阈值且 writer != nil 落盘 / 超阈值但 writer == nil 退化为内联截断。

**文件**:
- `vage/workspace/workspace.go` (修改; +20 行接口)
- `vage/workspace/filestore.go` (修改; +60 行)
- `vage/workspace/artifact_test.go` (新建; ~120 行)
- `vage/largemodel/context_editor.go` (修改; +80 行,total ≈ 440)
- `vage/largemodel/context_editor_elide_test.go` (新建; ~150 行)
- `vv/setup/setup.go` (修改; +10 行装配)

---

### C. P0-5 可观察性

#### C1. `vage/session/metrics.go` (新建) —— SessionMetrics 模型

```go
type SessionMetrics struct {
    SessionID                string    `json:"session_id"`
    PromptTokens             int       `json:"prompt_tokens"`
    CompletionTokens         int       `json:"completion_tokens"`
    TotalTokens              int       `json:"total_tokens"`
    CostUSD                  float64   `json:"cost_usd"`
    ActiveSeconds            int64     `json:"active_seconds"`
    ResumeCount              int       `json:"resume_count"`
    CheckpointSaveFailures   int       `json:"checkpoint_save_failures"`
    ContextEdits             int       `json:"context_edits"`
    ElidedArtifacts          int       `json:"elided_artifacts"`
    FirstSeen                time.Time `json:"first_seen"`
    LastUpdated              time.Time `json:"last_updated"`
}

type MetricsStore interface {
    Get(ctx, sid string) (*SessionMetrics, error)
    Update(ctx, sid string, fn func(*SessionMetrics)) error  // CAS
    Delete(ctx, sid string) error
}

type FileMetricsStore struct{ root string }  // <root>/<sid>/metrics.json
type MapMetricsStore struct{ ... }
```

**文件**:
- `vage/session/metrics.go` (新建; ~150 行)
- `vage/session/metrics_filestore.go` (新建; ~120 行)
- `vage/session/metrics_test.go` (新建; ~150 行)

#### C2. `vage/session/metrics_hook.go` —— SessionMetricsHook

- 实现 `hook.Hook` 接口,订阅:
  - `EventAgentEnd` → 累加 `AgentEndData.PromptTokens / CompletionTokens / TotalTokens`,加 cost(用现有 `costtraces` 的 pricing 表查 `Model`)
  - `EventCheckpointWritten` → no-op(只为后续扩展)
  - `EventContextEdited` → `ContextEdits += 1`,如果 strategy 是 elide_to_artifact 则 `ElidedArtifacts += 1`
- 新增独立小 API: `RecordResume(ctx, sid)` 在 vv resume 路径显式调用(不走 hook,因为没有对应事件)。
- 新增独立小 API: `RecordCheckpointFailure(ctx, sid)` 在 `vage/agent/taskagent/checkpoint.go` 的 save 失败路径**显式调用**(注入 metricsStore,可 nil)。

**文件**:
- `vage/session/metrics_hook.go` (新建; ~120 行)
- `vage/session/metrics_hook_test.go` (新建; ~150 行)
- `vage/agent/taskagent/checkpoint.go` (修改; +15 行,接收可选的 metricsStore)
- `vage/agent/taskagent/task.go` (修改; +5 行 option `WithMetricsStore`)

#### C3. BuildReport per-turn 落盘

- `vage/context/builder.go` `Build(...)` 之后,如果配置了 `WithBuildReportSink(sink)` 则调用 `sink.Save(ctx, sid, report)`。
- 新接口 `BuildReportSink interface { Save(ctx, sid string, r BuildReport) error }`,内置 `FileBuildReportSink`(append-only `<root>/<sid>/build_reports/<NNNNNN>.json`,LRU 淘汰超过上限的旧文件)。
- `vv/configs/config.go`: SessionConfig 新增 `PersistBuildReports *bool`(默认 true)+ `BuildReportLimit int`(默认 50)。
- `vv/setup/setup.go` 装配。

**文件**:
- `vage/context/build_report_sink.go` (新建; ~150 行)
- `vage/context/build_report_sink_test.go` (新建; ~120 行)
- `vage/context/builder.go` (修改; +20 行)
- `vv/configs/config.go` (修改; +15 行)
- `vv/configs/config_test.go` (修改; +30 行)
- `vv/setup/setup.go` (修改; +20 行)

#### C4. Checkpoint failure 结构化计数

- 见 C2 中的 `RecordCheckpointFailure`。
- `vage/agent/taskagent/checkpoint.go` save 失败的 `slog.Warn` 之后调用 `metricsStore.Update(...)`。

#### C5. HTTP `/metrics` 与 `/build-reports`

- `vv/httpapis/sessions.go` 新增:
  - `GET /v1/sessions/{id}/metrics` → 200 SessionMetrics JSON / 404
  - `GET /v1/sessions/{id}/build-reports?limit=20` → 200 [BuildReport] JSON / 404
- `Serve` 签名再加 `metricsStore session.MetricsStore` + `buildReportSink BuildReportReader`(读侧接口)。

**文件**:
- `vv/httpapis/metrics.go` (新建; ~150 行)
- `vv/httpapis/metrics_test.go` (新建; ~150 行)
- `vv/httpapis/http.go` (修改; +10 行)
- `vv/main.go` (修改; +5 行实参)

---

## C. 文档同步

每个 P0 完成后**立即**:

- `vage/.doc/checkpoint.md` (P0-3 末尾追加 "vv wiring" 段落)
- `vage/.doc/largemodel.md` §3 ContextEditorMiddleware (P0-4 三种策略)
- `vage/.doc/session.md` 新增 "Metrics" 段(P0-5)
- `vv/CLAUDE.md` "工程惯例" 之前新增简表
- `doc/design/session-context-solution-phase2.md` —— P0-3/P0-4/P0-5 标 ✅,引用本 dev session 路径
- `doc/design/session-context-solution.md` §4.3 / §4.5 / §4.7 状态行更新
- 本 dev session 内的 `result.md` 写最终结论(spec ↔ 实际差异、验证证据、文档链接)

---

## D. 估算

| 步骤 | LOC(含测试) | 累计 |
|---|---|---|
| A1 | 100 | 100 |
| A2 | 270 | 370 |
| A3 | 330 | 700 |
| B1 | 140 | 840 |
| B2 | 80 | 920 |
| B3 | 320 | 1240 |
| B4 | 440 | 1680 |
| C1 | 420 | 2100 |
| C2 | 290 | 2390 |
| C3 | 355 | 2745 |
| C5 | 315 | 3060 |
| 文档 | 200 | 3260 |

**总计 ≈ 3260 LOC**(其中测试约占 45%)。

> ⚠ 这个体量远超单个 dev session 的健康范围(原文档建议每个 P0 200–800 LOC)。
>
> **建议**: 如果今天就要全部完成,我可以一口气推下去;但更稳妥的方式是分**三个独立提交**(A→B→C 或 A→C→B),每完成一个 P0 就跑一次完整 test suite + 提一个独立 commit,这样某一项出问题不影响另两项的价值释放。
>
> 我会按这个节奏来,**每完成一个 P0 主步骤就停下来等批准再进下一个**。

---

## E. Checkpoint(执行前最后一次)

- **当前理解**: ✅ 三件 P0 的现状、范围、预期产出已对齐。
- **核心目标**: 让长任务在 vv 中崩溃可恢复 (P0-3) + context 不被超长/过期 tool_result 拖垮 (P0-4) + 跑完后能可观察 (P0-5)。
- **下一步 1-3 个动作**:
  1. (本次先停): 把 spec + plan 提交给用户审 → 等"开干"批准
  2. 批准后从 A1 开始(`vv/setup/setup.go` IterationStore 装配 + 单测)
  3. A1 done 后 commit → 进 A2 CLI `--resume`
- **风险**: 见 spec §6
- **验证方式**: 见 spec §7
