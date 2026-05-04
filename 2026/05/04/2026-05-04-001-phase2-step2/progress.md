# Phase 2 第 2 步 —— 进度看板

> 每完成一个步骤更新本文件。失败时保留失败现场说明,便于后续 resume。

---

## 状态总览

| 阶段 | 状态 | 说明 |
|---|---|---|
| Spec & Plan | ✅ 已完成 | 等待用户批准 |
| A. P0-3 Resume | 🟡 in progress | A1 ✅ / A2 ✅ / A3 ✅ / A4 ⏸ deferred to C2 |
| B. P0-4 Editor 增强 | ⏸ pending | B1/B2/B3/B4 |
| C. P0-5 Observability | ✅ 全部完成 | C1 ✅ / C2 ✅ / C3 ✅ / C5 ✅ (C4 已并入 C2) |
| B. P0-4 Editor 增强 | ⏸ 推迟到独立 dev session | 见下方决策说明 |
| Docs Sync | ✅ 完成 | checkpoint.md §10 / session.md §9 / context.md §11 / phase2 doc P0-3 P0-5 ✅ / phase1 doc §4.5 §4.7 |
| Result writeup | ✅ 完成 | result.md 9 节 |

---

## 步骤详情

### A1. setup IterationStore 装配 — ✅ 2026-05-04

**新增**:
- `vv/setup/checkpoint.go` —— `sessionRootDir(cfg)` + `buildIterationStore(cfg)` helper(57 行)
- `vv/setup/checkpoint_test.go` —— 6 个单测覆盖 disabled/nil/enabled/threaded/no-store/layout(207 行)

**修改**:
- `vv/registries/registry.go` —— `FactoryOptions.IterationStore` 字段 + import
- `vv/agents/coder.go` / `researcher.go` / `reviewer.go` / `primary.go` —— 各自透传 `taskagent.WithIterationStore(opts.IterationStore)`(每个 +4 行)
- `vv/setup/setup.go`:
  - import `vage/checkpoint`
  - `Options.IterationStore` 字段 + `getIterationStore` helper
  - `InitResult.IterationStore` 字段
  - `Init` 中在 hook/session 装配后调用 `buildIterationStore`,塞 opts + 写 InitResult
  - `factoryOpts.IterationStore` 透传(子 agent 循环 + Primary)
  - `buildHookManagerAndSession` / `buildTreeStore` 用统一的 `sessionRootDir(cfg)`(DRY)

**验证**:
- ✅ `go build ./...` (vv) 通过
- ✅ `go test ./setup/` 全绿 (含原有测试 + 新增 6 个 checkpoint 测试)
- ✅ `go test ./agents/ ./registries/` 全绿
- LSP 上有几个看起来是错误但实际是 stale 的 diagnostic(`Resume undefined`、`WithExtraSources undefined`、`dispatches.WithTreeStore undefined`),`go test`/`go build` 都通过,可忽略

**偏差 vs spec**:
- 原计划 `vv/setup/checkpoint.go` ~40 行,实际 57 行(把 `sessionRootDir` 一起放进来更清晰,顺手把 `buildHookManagerAndSession` / `buildTreeStore` 内重复的 `filepath.Join(...)` 也替换为该 helper —— 一次性消除 3 处重复)。
- 原计划测试 ~150 行,实际 207 行。多了 1 个 `TestBuildIterationStore_NilCfg` 防御 + 1 个 `TestSessionRootDir_DeterministicLayout` 钉死路径约定。

**等待用户批准进 A2: CLI `--resume` 入口**

### A2. CLI `--resume` 入口 — ✅ 2026-05-04

**新增**:
- `vv/cli/resume.go` —— `RunResume(ctx, initResult, sid, stdout, stderr) error` 主入口 + `resolveResumeAgent` helper + `lastAssistantText`/`renderResumeResponse` 渲染辅助 + 4 个 sentinel 错误(`ErrSessionDisabled` / `ErrNoIterationStore` / `ErrAgentNotFound` / `ErrAgentNotResumable`)(184 行)
- `vv/cli/resume_test.go` —— 9 个测试用例覆盖 nil InitResult / session disabled / no IterationStore / empty sid / not found / final cp / agent not found / Primary happy path / resolve helper 防御(257 行)

**修改**:
- `vv/dispatches/dispatch.go` —— 新增 `Dispatcher.Primary() agent.Agent` 公开 getter(必需:Primary 不在 dispatchable subAgents 中,resume 必须直接拿到它而不是调 Run)
- `vv/main.go`:
  - 新增 `--resume <sid>` flag + Examples 一行
  - 新增 resume 短路块(在 eval 之后、mode dispatch 之前):互斥校验 `--p / -eval / --session ≠ "" / --tree ≠ "" / mode ∈ {http,mcp}`,任一冲突 → exit 1 友好提示
  - 互斥校验通过后调 `setup.Init` + `cli.RunResume`,与 `-p` 路径对称

**验证**:
- ✅ `go build ./...` (vv 全包) 通过
- ✅ `go test ./cli/ -run 'TestRunResume|TestResolveResumeAgent' -v` 全 10 个测试绿
- ✅ `go test ./...` (vv 全包,含集成测试 dispatches_tests / setup_tests / session_tests / etc.) 全绿
- ✅ `cd vage && go test ./...` 全绿(确认 dispatcher 加 Primary getter 没有破坏 vage)

**关键设计决策**:
1. **Resume 走非流式**:`Resume(ctx, sid)` 是 sync 调用,不强行套 stream;CLI 输出是"完成态" — 用户看到 `[resume]` banner、最终消息、`[done]` 摘要。如果 future 加 SSE resume,可以单独再做。
2. **Primary 通过 dispatcher.Primary() 拿**:Primary 是 `Dispatchable: false`,不在 `Result.subAgents` 里;原来设计就是把它挂在 dispatcher 上,所以最自然的 lookup 是 `dispatcher.Primary()`。其他 sub-agent 走 `Result.Agent(id)`。
3. **错误链保留 errors.Is**:把 `ErrCheckpointNotFound` / `ErrAlreadyFinal` 包到 user-friendly 文本里时用 `%w` 保持 errors.Is 语义,callers (e.g., 未来 HTTP layer 映射 409) 不需要 string-matching。

**偏差 vs spec**:
- 原计划测试 ~150 行,实际 257 行(多了 nil InitResult 防御 + resolve helper 直接覆盖 + Primary happy-path 用真 *taskagent.Agent 跑通 Resume 全流程)
- 增加了一次 `vv/dispatches/dispatch.go` 改动(+12 行加 `Primary()` getter),原 spec 没列这一项 —— 实施时发现 Primary 没有公开 accessor,补上是必要的

**等待用户批准进 A3: HTTP `POST /v1/sessions/{id}/resume` 端点**

### A3. HTTP `POST /v1/sessions/{id}/resume` — ✅ 2026-05-04

**新增**:
- `vv/setup/resume.go` —— 重构: 把 sentinel errors `ErrSessionDisabled` / `ErrNoIterationStore` / `ErrAgentNotFound` + agent 解析逻辑从 `cli/resume.go` 迁移到 setup 包,以 `(ir *InitResult).ResumeAgent(agentID) (agent.Agent, error)` 方法形式暴露 —— cli 与 httpapis 都通过它做 agent lookup,避免重复(80 行)
- `vv/httpapis/resume.go` —— `handleResumeSession(initResult *setup.InitResult) http.HandlerFunc` 主入口 + `resumeResponse`/`resumeMessage` JSON 包络 + `toResumeResponse` 投影(190 行)
- `vv/httpapis/resume_test.go` —— 8 个测试用例覆盖 empty id / streaming 501 / session disabled (含 nil/empty InitResult) / no iter store / cp not found / final cp 409 / agent not found / Primary happy path 200 + body 验证(290 行)

**修改**:
- `vv/cli/resume.go` —— 删除 `resolveResumeAgent` 与重复的 sentinel errors;改用 `initResult.ResumeAgent(...)` + `setup.ErrSessionDisabled` / `setup.ErrNoIterationStore` / `setup.ErrAgentNotFound`;只保留 cli 独有的 `ErrAgentNotResumable`
- `vv/cli/resume_test.go` —— 测试用 setup.* sentinel 改写;`TestResolveResumeAgent_PrimaryWithoutDispatcher` 改为调 `c.ir.ResumeAgent(...)` 方法
- `vv/httpapis/http.go` —— `Serve` 签名追加 `initResult *setup.InitResult` 参数;`POST /v1/sessions/{id}/resume` 路由在 sessionStore 块内挂(因为 405-vs-503 区分需要 sessionStore 在场)
- `vv/main.go` —— `Serve` 调用追加 `initResult` 实参
- 6 个集成测试文件 —— 同步加 `nil` 占位 (eval / session_tests x3 / vector_tests x2 处)

**HTTP 状态码映射**:
| 错误 | HTTP | code |
|---|---|---|
| empty session id | 400 | bad_request |
| `?stream=1` | 501 | not_implemented(SSE 暂未支持) |
| nil InitResult / SessionStore == nil | 503 | session_disabled |
| IterationStore == nil | 503 | iteration_store_missing |
| `ErrCheckpointNotFound` | 404 | checkpoint_not_found |
| `ErrAlreadyFinal`(pre-check 与 race-after-Resume 都映射) | 409 | already_final |
| `ErrAgentNotFound` | 404 | agent_not_found |
| 非 *taskagent.Agent | 422 | agent_not_resumable |
| 其他 Load/Resume 错误 | 500 | load_failed / resume_failed |

**A4 决策**:原计划 A4 是"Resume 路径 +1 resume_count"。这跟 P0-5 SessionMetrics 强耦合,**合并到 C2 SessionMetricsHook 一起做**——拆开做意味着 C2 之前要先建半截指标系统。等 C1/C2 落地后,在 cli `RunResume` + httpapis `handleResumeSession` 两处统一调用 `metrics.RecordResume(ctx, sid)`。

**验证**:
- ✅ `go build ./...` 全包通过
- ✅ `go test ./httpapis/ -run TestHandleResume -v` 8 个新测试全绿(含 nil/empty InitResult sub-table)
- ✅ `go test ./...` (vv 全包,含集成 dispatches/eval/golden/httpapis/mcps/session/setup/tools/traces/vector) 0 个 FAIL
- ✅ `cd vage && go test ./...` 全绿

**关键设计决策**:
1. **错误抽到 setup 层**:`setup` 已经是"装配 + InitResult 拥有者",sentinel + 解析方法放它最自然,cli 与 httpapis 都从这里取,避免在两处重复维护"agent_id → agent" 的逻辑(并增加未来 mcps 加 resume 端点的复用面)。
2. **流式 resume 标 501 而非"假流式"**:Resume 现在只有 sync 形态;返回 fake SSE 不会增加价值反而让客户端误以为 resume 是 incremental 的。501 + clear hint 让 future 加 `ResumeStream` 时只需扩,不需 break。
3. **HTTP 408? 没有**:Load/Resume 超时由 `r.Context()` 串入,客户端取消触发的 ctx.Err() 会通过 500 path 出来。要 408 得加 `errors.Is(err, context.DeadlineExceeded)` 分支,本期不需要。
4. **Primary 不在 subAgents 但 cp.AgentID 可能写"primary"**:通过 `Dispatcher.Primary()`(A2 已加)解决;`InitResult.ResumeAgent("primary")` 走专门分支。

**偏差 vs spec**:
- 原计划"流式 SSE 同步两种姿态" → 改为 sync + 501 stream(理由见决策 2);未来需要时单独 follow-up
- 错误抽到 setup 是 spec 没列的重构,实施时发现 cli 与 httpapis 各做一份会漂移;hoist 是必要的
- 集成测试 6 个文件改 `nil` 也是 spec 没列的连带改动

**P0-3 整体完成情况**:A1+A2+A3 已完成,A4 延迟到 P0-5 一起做。**长任务崩溃可恢复**这个核心目标在 CLI 与 HTTP 两侧都已通路:
- CLI: `vv --resume <sid>` 续接最新 cp
- HTTP: `POST /v1/sessions/{id}/resume` 续接最新 cp
- vv 装配自动构造 `FileIterationStore`(session enabled 时)
- 4 个 sub-agent (coder/researcher/reviewer/primary) 都配 IterationStore

**等待用户批准进 P0-5 阶段(C1: SessionMetrics 模型 + FileMetricsStore)**

### C1. SessionMetrics 模型 + Map/File MetricsStore — ✅ 2026-05-04

**新增**:
- `vage/session/metrics.go` —— `SessionMetrics` 结构体(11 个字段:identity / 5 类计数器 / cost / active time / 2 个时间戳)+ `MetricsStore` 接口(Get/Update/Delete)+ 共享 `applyUpdate` body + `cloneMetrics` helper + `ErrMetricsNotFound` sentinel(180 行)
- `vage/session/metrics_mapstore.go` —— `MapMetricsStore` 内存实现 + `WithClock` 测试钩子(110 行)
- `vage/session/metrics_filestore.go` —— `FileMetricsStore` 文件实现,布局 `<root>/<sid>/metrics.json` 与 FileSessionStore 对齐;原子写复用 `writeJSONAtomic`;per-session mutex(195 行)
- `vage/session/metrics_test.go` —— 11 个 MapStore 测试覆盖 bootstrap / accumulate / total auto-derive / no-op / nil fn / missing / delete idempotent / invalid id / ctx cancel / get returns copy / 200 协程并发(310 行)
- `vage/session/metrics_filestore_test.go` —— 7 个 FileStore 测试覆盖 reject empty / round-trip across reopen / on-disk layout pinned / delete unlinks file / get missing / pretty-printed wire-format / 50 sessions × 4 concurrent(220 行)

**修改**: 无(全新增)

**关键设计决策**:
1. **`applyUpdate` 共享 body**:Map 和 File 两个实现调用同一函数,确保 bootstrap / timestamp / TotalTokens auto-derive 三个不变量在两处行为一致。
2. **`TotalTokens` 自动派生**:不管 fn 是否触碰 Total 字段,store 都用 `Prompt + Completion` 覆盖 —— 防止只更新两个分量的 hook 留下 stale Total。
3. **`Get` 返回 deep copy**:防止调用方改回内存里的记录;`TestSessionMetrics_GetReturnsCopy` 钉死。
4. **`ErrMetricsNotFound` 与 zero 区分**:首次访问时返回 ErrMetricsNotFound,而非 zero-valued struct —— 让 HTTP 层可以选择 404 vs 200+zeros。
5. **fn 为 nil 合法**:允许"materialise 一个空记录"(给后续 Get 用),不强制 fn 非 nil。
6. **WithClock 注入**:Map 和 File 都暴露,测试可用 deterministic 时间戳验证 FirstSeen/LastUpdated 行为。
7. **Per-session 锁(File)+ 全局 RWMutex(Map)**:Map 的全局锁简单足够(几次/Run);File 用 per-session 锁让不同 session 的写入并行。

**验证**:
- ✅ `go build ./...` (vage) 通过
- ✅ `go test ./session/ -run TestSessionMetrics -v` 11 个测试全绿
- ✅ `go test ./session/ -run TestFileMetricsStore -v` 7 个测试全绿
- ✅ `go vet ./...` (vage) 通过
- ✅ `go test ./...` (vage 全包) 0 个 FAIL
- ✅ `go test ./...` (vv 全包) 0 个 FAIL(确认新文件没破坏 vv 端的 transitive deps)

**偏差 vs spec**:
- 原计划 metrics.go ~150 行实际 180 行(多了 applyUpdate 共享 body 的注释 + 字段分组规则注释,这部分注释能让后续人加字段时不破坏 wire-compat)
- 原计划 metrics_filestore.go ~120 行实际 195 行(分了 Update / loadMetricsFile 两个函数,Update 路径更清晰;另外 Update 自动 mkdir 父目录,免得 hook 必须在 SessionStore.Create 之后才能调)
- 多写了一个 `metrics_mapstore.go` 单独文件(原计划与 metrics.go 同文件)—— 让两个 store 实现并列,且单文件 ≤ 200 行,符合项目 "单文件 ≤ 800 行" 规则
- 测试 18 个,远超原计划 ~6 个 —— 关键不变量(TotalTokens 派生、Get 返回 copy、ctx cancel、并发序列化)各自独立钉死

**等待用户批准进 C2: SessionMetricsHook(订阅 EventAgentEnd / EventCheckpointWritten / EventContextEdited)+ Resume 路径累计 resume_count(A4 合并)+ checkpoint 失败计数注入**

### C2. SessionMetricsHook + 失败回调注入(含 A4 / C4 合并) — ✅ 2026-05-04

**新增**:
- `vage/session/metrics_hook.go` —— `SessionMetricsHook` 实现 `hook.Hook`,订阅 `EventLLMCallEnd / EventAgentEnd / EventContextEdited` 三个事件;`PricingFunc` 类型 + `WithMetricsPricing` / `WithMetricsLogger` 选项;独立 API `RecordResume` / `RecordCheckpointFailure`;`computeCost` 私有 helper;常量 `ContextEditStrategyElideArtifact`(195 行)
- `vage/session/metrics_hook_test.go` —— 14 个测试覆盖 filter scope / LLMCallEnd 计数 / pricing 计算 / unknown model / AgentEnd 累计活跃秒 / 0 duration 跳过 / ContextEdited strategy 区分 / RecordResume / RecordCheckpointFailure / nil store no-op / empty sessionID skip / 错误 payload type 跳过 / logger option / 0-token LLM 事件还落盘(310 行)
- `vage/agent/taskagent/checkpoint_failure_test.go` —— 3 个测试覆盖 fail-callback 触发 3 次 + Run 仍正常完成 / nil callback 不 panic / 成功路径不调 callback(170 行)

**修改**:
- `vage/agent/taskagent/task.go` —— `Agent.checkpointFailureCB` 字段 + `CheckpointFailureCallback` 类型 + `WithCheckpointFailureCallback` option(+25 行)
- `vage/agent/taskagent/checkpoint.go` —— `saveIterationCheckpoint` 失败路径在 `slog.Warn` 之后调 callback(+3 行)

**关键事件 → 指标映射**:
| 事件 | 指标更新 |
|---|---|
| `EventLLMCallEnd` | PromptTokens += / CompletionTokens += / CostUSD += |
| `EventAgentEnd` | ActiveSeconds += Duration/1000 |
| `EventContextEdited` | ContextEdits++; if Strategy=="elide_to_artifact" → ElidedArtifacts++ |
| `Resume()` 成功调用 | ResumeCount++(`RecordResume` 显式调) |
| `IterationStore.Save` 失败 | CheckpointSaveFailures++(taskagent callback → `RecordCheckpointFailure`) |

**关键设计决策**:
1. **callback 模式而非 store 注入**:taskagent 不依赖 vage/session,通过 `CheckpointFailureCallback` 类型解耦。vv setup 在装配阶段把回调闭包指向 `metricsHook.RecordCheckpointFailure` 实现连接。
2. **失败计数与成功事件不对称**:成功保存 → 派发 `EventCheckpointWritten`;失败保存 → 不派发事件、走 callback。这样 `EventCheckpointWritten` 计数仍是"成功保存数",`CheckpointSaveFailures` 是"失败数",两个指标独立无混淆。
3. **PricingFunc 抽象**:hook 不依赖 vv/costtraces,接 `func(model)(prompt, completion, ok)` 由 vv 装配时注入。pricing miss(ok=false) → cost 不增,但 token 计数仍更新;pricing 表稀疏不影响其他指标。
4. **error log + drop**:hook 内部任何 store 写入失败都 `slog.Warn` 后丢弃 —— 指标是 observability,不应影响 hot path。
5. **A4 / C4 合并到 C2**:原计划 A4(resume_count++)和 C4(失败结构化计数)都是"在某个动作发生时调一次 store.Update",和 C2 共用 hook 实例;放一起做避免在两个地方 import session 包。

**验证**:
- ✅ `go build ./...` (vage) 通过
- ✅ `go test ./session/ -run TestMetricsHook -v` 14 个测试全绿
- ✅ `go test ./agent/taskagent/ -run TestSaveCheckpointFailure -v` 3 个测试全绿(且观察到 slog.Warn 输出确认失败路径走通)
- ✅ `go test ./...` (vage 全包) 0 个 FAIL
- ✅ `go test ./...` (vv 全包,确认 taskagent 加 callback option 没破坏 vv) 0 个 FAIL

**偏差 vs spec**:
- 原计划 metrics_hook.go ~120 行实际 195 行(多了 PricingFunc / WithMetricsLogger / 常量 / 详细错误处理 +注释)
- 原计划测试 ~150 行实际 hook 310 + callback 170 = 480 行;关键不变量(filter 范围、pricing 缺失不爆、payload type 错不 panic、empty sid 跳过)各自独立钉死
- 没有触碰 `vage/agent/taskagent/checkpoint_test.go`(原文件保持不动,新建独立的 `checkpoint_failure_test.go` 让两类测试不混杂)

**P0-5 进度小结**: C1 + C2 已落地,vage 端"指标怎么算"完整就位。还差:
- C3:BuildReport per-turn 落盘
- C5:HTTP `/metrics` + `/build-reports` + vv 装配 hook + Resume 调 RecordResume

**等待用户批准进 C3: BuildReport per-turn 落盘(`vage/context/build_report_sink.go` + LRU + 配置项)**

### C3. BuildReport per-turn 落盘 — ✅ 2026-05-04

**新增**:
- `vage/context/build_report_sink.go` —— `BuildReportSink` 写接口 + `BuildReportReader` 读接口 + `FileBuildReportSink` 同时实现两者;布局 `<root>/<sid>/build_reports/<NNNNNN>.json`,LRU 默认 50 上限;atomic write(tmp + rename + fsync);per-session mutex 防序列号冲突;选项 `WithBuildReportLimit / WithBuildReportPretty / DisableBuildReportPersistence`;常量 `DefaultBuildReportLimit=50` / `MaxListLimit=500`(345 行)
- `vage/context/build_report_sink_test.go` —— 17 个 sink 测试 + 3 个 Builder 集成测试,共 20 个用例覆盖 reject empty root / save-and-list newest-first / LRU 保留最新 N / 默认 limit / 负 limit fallback / disabled-mode no-op / list missing dir / list zero limit / list MaxLimit clamp / reject empty sid / pretty default / non-pretty option / 60 协程并发唯一序列 / 跨 session 隔离 / 布局钉死 / 跳过非 report 文件 / 序列命名格式 / Builder 调 sink 一次 / Builder 空 sid 跳过 / Builder 错误 best-effort(420 行)

**修改**:
- `vage/context/builder.go` —— `DefaultBuilder.reportSink` 字段 + `WithBuildReportSink` option;`Build()` 在 dispatch 事件后调 `b.reportSink.Save`,失败 `slog.Warn` 不阻断(+18 行)

**关键设计决策**:
1. **写读两接口分离**:`BuildReportSink`(只写)给 Builder,`BuildReportReader`(只读)给 HTTP / CLI。`FileBuildReportSink` 同时实现 → 一份后端,两套窄契约,callers 不需要对方法面广的依赖。
2. **同步 sink 调用,best-effort 错误**:Sink 在 Build 内部同步调,但失败 `slog.Warn` 即丢。理由:写操作通常 < 1ms;若 async 则 Build 退出后任何 panic 没人收;同步 + 容错是观测性最佳实践。
3. **per-session mutex,无全局锁**:序列号分配 = `lockFor(sid).Lock()` + `scanReportSequences(dir)` + `next = max+1`。不同 session 完全并行,同 session 内 60 路并发实测 0 collision。
4. **LRU = trim by sequence ascending**:每次 Save 后 `if len > limit: rm 0..len-limit`。无后台 goroutine,无 atime 跟踪;按序号 LRU 与按时间 LRU 等价(写入序就是时间序)。
5. **MaxListLimit=500 硬上限**:HTTP 端点不让 caller 一次拉光整库;真要拉就 paginate。
6. **non-report 文件跳过**:`buildReportFilePattern` 正则严格匹配 `^\d{6}\.json$`,`.DS_Store` / `.tmp` 等不影响序列号分配。
7. **JSON 默认 pretty**:ops `cat <NNNNNN>.json` 直接读;高写场景可 `WithBuildReportPretty(false)` 关掉。
8. **`DisableBuildReportPersistence` option**:让 vv 配置层可以"装好 sink 但运行时关掉",不需要重建 builder。

**验证**:
- ✅ `go build ./...` (vage) 通过
- ✅ `go test ./context/ -run TestFileBuildReportSink|TestSequenceNamingFormat|TestDefaultBuilder_BuildReportSink -v` 全 20 个测试绿
- ✅ `go test ./context/` 全包绿(确认没破坏现有 builder 测试)
- ✅ `go test ./...` (vage 全包) 0 个 FAIL
- ✅ `go test ./...` (vv 全包) 0 个 FAIL

**偏差 vs spec**:
- 原计划 build_report_sink.go ~150 行,实际 345 行(完整两接口分离 + LRU 实现 + atomic write + scan helpers + 详细注释 + 选项)
- 原计划测试 ~120 行,实际 420 行(20 个独立用例,关键不变量都有钉死:newest-first 顺序、LRU 保留集合、并发唯一序列、跨 session 隔离、junk 文件跳过)
- 原计划要把 vv config 项 `session.persist_build_reports` + `session.build_report_limit` 一起做,**这部分推迟到 C5** 一起做(C5 同时做 vv 装配 + HTTP 端点 + 配置项,一次走完整闭环)

**P0-5 进度小结**: C1(metrics 模型/store)+ C2(hook + callback)+ C3(BuildReport sink)三件 vage 端基建已落地。还差 C5 把它们与 vv 装配 + HTTP 端点连起来做完整闭环。

**等待用户批准进 C5: vv 装配 SessionMetricsHook + FileMetricsStore + FileBuildReportSink + HTTP `GET /v1/sessions/{id}/metrics` 与 `GET /v1/sessions/{id}/build-reports` + Resume 路径调 `RecordResume`**

### C5. vv 装配 + HTTP 端点 + Resume 路径累计 — ✅ 2026-05-04

**新增**:
- `vv/setup/metrics.go` —— `buildMetricsStore` / `buildBuildReportSink` / `buildMetricsHook` 三个构造 helper + 三个 `getXxx(opts)` 安全访问器 + `getCheckpointFailureCB`;`buildMetricsHook` 把 vv 的 `costtraces.LookupPricing`(per-million tokens)适配为 vage 的 `session.PricingFunc`(per-1k tokens)(135 行)
- `vv/httpapis/metrics.go` —— `handleGetMetrics` + `handleListBuildReports` 两个 handler;`buildReportsResponse` 包络;`metricsStore(ir)` / `buildReportReader(ir)` 安全访问器(155 行)
- `vv/httpapis/metrics_test.go` —— 10 个 HTTP 测试覆盖 nil store→503 / empty id→400 / not found→404 / happy path 200+JSON / nil reader→503 / bad limit→400 / empty session→200+空列表 / happy path newest-first / 上限 clamp(220 行)
- `vage/agent/taskagent/task.go` —— `WithBuildReportSink` option(向 internal builder 转发 sink)(+12 行)

**修改**:
- `vv/configs/config.go` —— `SessionConfig.PersistBuildReports *bool`(默认 true) + `BuildReportLimit int`(0 → 50);新方法 `PersistBuildReportsEnabled()`(+22 行)
- `vv/registries/registry.go` —— `FactoryOptions.BuildReportSink` + `CheckpointFailureCB` 字段(+15 行)
- `vv/agents/coder.go / researcher.go / reviewer.go / primary.go` —— 各透传两个新 option(每个 +8 行)
- `vv/setup/setup.go`:
  - `Options` 加 `MetricsStore` / `MetricsHook` / `BuildReportSink` / `CheckpointFailureCB` 四个字段
  - `InitResult` 加 `MetricsStore` / `MetricsHook` / `BuildReportSink` 三个字段
  - `Init` 中在 IterationStore 之后新增装配块:构造 metrics store + hook + 注册到 hookManager(若不存在则启动一个);构造 build_report sink;注入 `CheckpointFailureCB` 闭包
  - `factoryOpts` 透传 `BuildReportSink` + `CheckpointFailureCB`(子 agent 循环 + Primary build)(+70 行总)
- `vv/cli/resume.go` —— Resume 成功后调 `initResult.MetricsHook.RecordResume(ctx, sid)`(+5 行)
- `vv/httpapis/resume.go` —— 同上(+7 行)
- `vv/httpapis/http.go` —— 注册 `/metrics` 与 `/build-reports` 路由(+4 行)

**关键设计决策**:
1. **PricingFunc 单位换算**:vv 的 `costtraces.Pricing` 是 USD per million tokens,vage 的 `session.PricingFunc` 是 USD per 1k tokens。在 `buildMetricsHook` 一次性除 1000.0 完成换算,两边都不需要知道对方单位。
2. **`WithBuildReportSink` 在 taskagent 而非新增 builder option**:agent factory 透传一个 sink 选项就行,不必让 caller 替换整个 Builder。这与 phase2 doc P1-6 "WithContextBuilder" 不冲突 —— P1-6 是更彻底的 builder 替换。
3. **`CheckpointFailureCB` 用 closure 适配签名**:taskagent 的 callback 是 `func(ctx, sid, err)`,metrics hook 的 `RecordCheckpointFailure(ctx, sid)` 是 `func(ctx, sid) error`。在 setup 中用 closure `func(ctx, sid, _) { _ = hook.RecordCheckpointFailure(ctx, sid) }` 适配,丢弃 saveErr(slog.Warn 已经记了)和返回值(metrics 内部已 log)。
4. **HTTP `/metrics` 与 `/build-reports` 都挂在 `if sessionStore != nil` 内**:即使下游 store 是 nil(配置关闭),路由仍 register 并返回 503 —— 让用户能从 HTTP 自我诊断,而不是 404 误以为路由不存在。
5. **`/build-reports` 空 session 返 200+空列表**:`/metrics` 的"无数据"是 404(因为 metrics 是聚合,空 vs 零有意义);`/build-reports` 的"无数据"是 200+空(因为列表语义,无观测就是合法状态)。
6. **`hookManager` 自动启动**:metrics 装配时若 trace + session + vector 都关 → hookManager 还是 nil。这种情况下 metrics 自己启一个,沿用 vector 装配的同款 fallback 模式。
7. **`buildReportReader` 走类型断言**:`InitResult.BuildReportSink` 类型是 `vctx.BuildReportSink`(只写),HTTP 需要 `vctx.BuildReportReader`(只读)。`FileBuildReportSink` 同时实现两者,做一次类型断言取出读侧。

**验证**:
- ✅ `go build ./...` (vv + vage) 通过
- ✅ `go test ./httpapis/ -run TestHandleGetMetrics -v` 4 个测试绿
- ✅ `go test ./httpapis/ -run TestHandleListBuildReports -v` 6 个测试绿
- ✅ `go test ./...` (vv 全包,含集成测试) 0 个 FAIL
- ✅ `go test ./...` (vage 全包) 0 个 FAIL

**HTTP 端点完整状态**:
| 端点 | 200 | 400 | 404 | 503 | 备注 |
|---|---|---|---|---|---|
| GET /v1/sessions/{id}/metrics | metrics JSON | empty id | metrics_not_found | metrics_disabled | |
| GET /v1/sessions/{id}/build-reports | { reports: [...] } | empty id / bad limit | — | build_reports_disabled | 空 session 仍是 200+空 |

**P0-5 整体完成情况**: C1+C2+C3+C5(C4 并入 C2)。"跑完后能看清健康度"完整闭环:
- LLM 调用/Agent 完成/Context 编辑事件 → SessionMetricsHook → MetricsStore JSON 文件
- ReAct 每轮 → BuildReportSink → `<root>/<sid>/build_reports/<NNNNNN>.json`(LRU)
- Checkpoint Save 失败 → callback → 计数器 +1
- Resume 成功 → CLI / HTTP 两路都调 RecordResume
- HTTP `/metrics` / `/build-reports` 暴露给运维

**P0-4 范围决策(写在这里给后续接续)**:
原计划的 P0-4 ContextEditor 增强(stale 标记 + 单条转引用)按 plan.md 估算 760+ LOC(B1~B4),且 stale 检测需要在 `vage/tool/` 引入 `ResourceTracker` 协议,影响面较大。Phase2 doc 注明"每个 P0/P1 单项落地估算 200–800 LOC,建议拆为独立 dev session"。考虑到本 session 已落地 P0-3 + P0-5(LOC 实测约 4500 含测试,远超原"3260" 估算),**P0-4 推迟到独立 dev session**,以确保:
1. P0-3 / P0-5 的现有改动充分沉淀(完整 commit + 文档同步 + 在生产环境实测)
2. P0-4 单独评审,不被本 session 末段的疲劳影响代码质量
3. 与未来 P1-3 (artifacts/) 的设计可能有交集,等 P1 计划成型再决定如何对齐

**等待用户决策**:
- 选项 (a) 现在就开 P0-4(继续本 session,可能进入超长状态)
- 选项 (b) 把本 session 的成果(P0-3 全 + P0-5 全)落定为独立 commit/PR,P0-4 启新 session
- 选项 (c) 跳过 P0-4(短期接受现有 ContextEditor 行为),把节省下来的精力直接进 Phase 2 第 3 步(P1-1 / P1-3 / P1-5)


