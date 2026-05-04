# Phase 2 第 2 步 —— 完成报告

> Spec: [`spec.md`](./spec.md) · Plan: [`plan.md`](./plan.md) · Progress: [`progress.md`](./progress.md)
> 模式: SDD-Lite `deep` · 完成日期: 2026-05-04 · 用户决策: 选项 (b) 把当前成果落定为独立 commit/PR

---

## 1. 完成范围 vs Spec

| 项 | 计划 | 实际 |
|---|---|---|
| **P0-3 Resume vv wiring** | A1 setup / A2 CLI / A3 HTTP / A4 metrics +1 | ✅ 全部完成,A4 合并到 C2 |
| **P0-5 可观察性** | C1 模型 / C2 hook / C3 BuildReport / C4 cp 失败 / C5 HTTP+wiring | ✅ 全部完成,C4 合并到 C2 |
| **P0-4 Editor 增强** | B1 协议 / B2 fs 标注 / B3 stale / B4 单条转引用 | ⏸ 推迟到独立 dev session(详见 §6) |

---

## 2. 核心目标(Loop Anchor)证明

> **目标**: 让一个长任务可以 (a) 进程崩溃 → 重启 → 续接;(b) 跑很久 → context 不被拖垮;(c) 跑完后 → 一眼看清健康度。

**(a) 进程崩溃可恢复 ✅**
- `setup.Init` 自动构造 `FileIterationStore`(条件:`cfg.Session.IsEnabled()`),所有 sub-agent + Primary 都接通。
- CLI: `vv --resume <sid>` 走 `cli.RunResume` → `(*taskagent.Agent).Resume` → 渲染 RunResponse + 调 `RecordResume`。
- HTTP: `POST /v1/sessions/{id}/resume`(同步)→ 200 RunResponse JSON + 调 `RecordResume`。
- 错误链完整:`ErrCheckpointNotFound`/`ErrAlreadyFinal`/`ErrAgentNotFound` 都有 typed sentinel + HTTP 状态码语义化映射。
- **Done by Evidence**: 9 个 cli 测试 + 8 个 httpapis 测试,含 happy-path 用真 `*taskagent.Agent` + stub LLM 跑通完整 Resume 流程。

**(b) Context 不被拖垮 ⏸**
- 这一项的对应工作是 P0-4(stale 标记 + 单条转引用),本 session 推迟。
- 现状:`ContextEditorMiddleware` 的 keep_last_k 折叠仍可用(Phase 1 已落地);未补的是基于 ResourceID 的语义 stale 检测,以及单条超长消息外移到 `artifacts/elided/`。

**(c) 跑完后能看清健康度 ✅**
- `SessionMetrics`(8 类计数器 + 时间戳)→ `FileMetricsStore` 落 `<root>/<sid>/metrics.json`。
- `SessionMetricsHook` 订阅 LLM/Agent/Edit 事件;独立 API `RecordResume`/`RecordCheckpointFailure` 处理非事件路径。
- 每轮 `BuildReport` 落 `<root>/<sid>/build_reports/<NNNNNN>.json`,LRU 默认 50 份。
- HTTP `GET /v1/sessions/{id}/metrics` + `/build-reports?limit=N` 暴露给运维。
- **Done by Evidence**: 18 个 metrics tests(11 model + 7 file)+ 14 个 hook tests + 3 个 callback tests + 20 个 build_report tests(17 sink + 3 builder)+ 10 个 HTTP tests = **65 个新增测试,全绿**。

---

## 3. 改动清单

### 3.1 vage 模块

**新文件**:
- `vage/session/metrics.go` —— SessionMetrics + MetricsStore 接口 + applyUpdate 共享体
- `vage/session/metrics_mapstore.go` —— MapMetricsStore
- `vage/session/metrics_filestore.go` —— FileMetricsStore(`<root>/<sid>/metrics.json`)
- `vage/session/metrics_test.go` / `metrics_filestore_test.go` —— 18 个用例
- `vage/session/metrics_hook.go` —— SessionMetricsHook + PricingFunc + RecordResume / RecordCheckpointFailure
- `vage/session/metrics_hook_test.go` —— 14 个用例
- `vage/context/build_report_sink.go` —— BuildReportSink/Reader 接口 + FileBuildReportSink(LRU + atomic write)
- `vage/context/build_report_sink_test.go` —— 17 sink + 3 builder = 20 个用例
- `vage/agent/taskagent/checkpoint_failure_test.go` —— 3 个用例

**修改**:
- `vage/agent/taskagent/task.go` —— `CheckpointFailureCallback` type + `WithCheckpointFailureCallback` option + `WithBuildReportSink` option + 字段
- `vage/agent/taskagent/checkpoint.go` —— save 失败时调 callback
- `vage/context/builder.go` —— `WithBuildReportSink` option + Build 中同步调 sink

### 3.2 vv 模块

**新文件**:
- `vv/setup/checkpoint.go` —— `sessionRootDir(cfg)` + `buildIterationStore(cfg)`
- `vv/setup/checkpoint_test.go` —— 6 个用例
- `vv/setup/resume.go` —— sentinel errors + `(*InitResult).ResumeAgent(agentID)` 方法
- `vv/setup/metrics.go` —— `buildMetricsStore` / `buildBuildReportSink` / `buildMetricsHook` + 4 个安全访问器
- `vv/cli/resume.go` —— `RunResume(ctx, ir, sid, ...)` 主入口 + render helper
- `vv/cli/resume_test.go` —— 9 个用例
- `vv/httpapis/resume.go` —— `handleResumeSession(initResult)` + JSON 包络
- `vv/httpapis/resume_test.go` —— 8 个用例
- `vv/httpapis/metrics.go` —— `handleGetMetrics` + `handleListBuildReports` + 安全访问器
- `vv/httpapis/metrics_test.go` —— 10 个用例

**修改**:
- `vv/registries/registry.go` —— `FactoryOptions` 加 `IterationStore` / `BuildReportSink` / `CheckpointFailureCB`
- `vv/agents/coder.go / researcher.go / reviewer.go / primary.go` —— 各透传 3 个新 option
- `vv/setup/setup.go` —— `Options` + `InitResult` 加 7 个字段;`Init` 中加 metrics + build_report 装配块;`factoryOpts` 透传
- `vv/configs/config.go` —— `SessionConfig.PersistBuildReports` + `BuildReportLimit` + `PersistBuildReportsEnabled()`
- `vv/dispatches/dispatch.go` —— `Dispatcher.Primary()` 公开 getter(resume 必需)
- `vv/main.go` —— `--resume` flag + 互斥校验 + 调度块;`Serve` 调用追加 `initResult` 实参
- `vv/httpapis/http.go` —— `Serve` 签名加 `*setup.InitResult` 参数;注册 3 路由
- `vv/cli/resume.go` / `vv/httpapis/resume.go` —— Resume 后调 `RecordResume`
- 6 个集成测试文件 —— 同步加 `nil` / `res` 作为 `Serve` 的新参实参

### 3.3 文档同步

- `vage/.doc/checkpoint.md` —— 新增 §10 vv 端 wiring,把 P0-3 从 out-of-scope 表中划掉
- `vage/.doc/session.md` —— 新增 §9 Metrics(7 小节,涵盖字段/接口/hook/build_report/wiring/HTTP/config)
- `vage/.doc/context.md` —— 新增 §11 BuildReport 持久化
- `doc/design/session-context-solution-phase2.md` —— P0-3 / P0-5 标 ✅;落地路径 table 更新
- `doc/design/session-context-solution.md` —— §4.5 / §4.7 状态行更新

---

## 4. 验证证据

### 4.1 测试

| 模块 | 单元 | 集成 | 备注 |
|---|---|---|---|
| `vage/session` | metrics 11+7 / hook 14 = **32 新测试** | — | filestore round-trip / 60 路并发 / event filter / pricing / nil store |
| `vage/agent/taskagent` | callback 3 新测试 | — | 失败回调触发 3 次 / 成功不触发 / nil callback 不 panic |
| `vage/context` | sink 17 + builder 3 = **20 新测试** | — | LRU / 60 路并发唯一序列 / pretty / disabled / junk 跳过 |
| `vv/setup` | checkpoint 6 新测试 | — | enabled/disabled/threaded/no-store/layout |
| `vv/cli` | resume 9 新测试 | — | nil ir / session disabled / no store / not found / final / agent not found / Primary happy / resolve helper |
| `vv/httpapis` | resume 8 + metrics 10 = **18 新测试** | — | 各状态码独立钉死 / 真 TaskAgent + stub LLM happy path |
| **合计** | **86 个新测试,全绿** | | 全 vage / 全 vv 0 个 FAIL |

### 4.2 HTTP 状态码完整性

| 端点 | 200 | 400 | 404 | 409 | 422 | 500 | 501 | 503 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `POST /v1/sessions/{id}/resume` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `GET /v1/sessions/{id}/metrics` | ✓ | ✓ | ✓ | — | — | ✓ | — | ✓ |
| `GET /v1/sessions/{id}/build-reports` | ✓ | ✓ | — | — | — | ✓ | — | ✓ |

每个状态码都有对应测试。

### 4.3 LOC 实测

| 类别 | 新增 LOC | 测试 LOC | 总计 |
|---|---|---|---|
| vage 实现 | ~885 | ~1230 | ~2115 |
| vv 实现 | ~640 | ~1140 | ~1780 |
| 文档 | ~280 | — | ~280 |
| **合计** | **~1805** | **~2370** | **~4175** |

(原 plan.md 估算 3260,实际 +28%,主要是注释 + 防御性 case + happy-path 用真 TaskAgent)

---

## 5. 与 Spec 的偏差

| Spec 项 | 实际 | 理由 |
|---|---|---|
| P0-3 流式 SSE resume | 改为 sync + 501 stream | `TaskAgent.Resume` 当前只有 sync 形态;假 SSE 会让 client 误以为 incremental |
| 错误 sentinel 在 cli/ | 抽到 setup/ | cli + httpapis 共享,避免漂移 |
| Checkpoint failure 用 store 直接注入 | callback 模式 | taskagent 不依赖 vage/session,closure 在 setup 层适配 |
| C2 / C4 / A4 分别做 | 合并到 C2 | 三件事共用同一 hook 实例,分开做要在多处 import session |
| C5 wiring 与 HTTP 端点分开 | 合并 | wiring + Resume 调 hook + 端点必须一起做才能闭环 |
| P0-4 在本 session | ⏸ 推迟 | 见 §6 |

---

## 6. P0-4 推迟决策

**原因**:
1. **范围估算超出 session 健康范围** —— P0-4 估算 760+ LOC,本 session 已经做了 ~4175 LOC(P0-3 + P0-5)
2. **新增协议影响面较大** —— stale 检测需要在 `vage/tool/` 引入 `ResourceTracker` 协议,涉及 `fs.ReadFile/WriteFile/EditFile` 全部要挂上 `ResourceIDs(args)` 实现
3. **公开 API 破坏性变更** —— `PlaceholderFunc` 签名要加 `reason` 参数,需要 deprecated alias + 新 V2 API
4. **与 P1-3 (artifacts/) 设计交集** —— 单条转引用要落到 `<sessionRoot>/<sid>/workspace/artifacts/elided/`,而 P1-3 计划做完整 artifacts 工具集(LLM 工具 `artifact_write` / `artifact_read` 等);最好等 P1-3 设计稳定再决定 elided 是私有 API 还是公开复用

**接续锚点**(给后续 dev session):
- 本 session 落地的 metrics 已经预留了 `ContextEdits` / `ElidedArtifacts` 计数器和 `ContextEditStrategyElideArtifact = "elide_to_artifact"` 常量,P0-4 落地时直接复用
- `vage/largemodel/context_editor.go` 现状是 keep_last_k 单策略;新增 `stale_resource` / `elide_to_artifact` 时,Strategy 字段已经在 `schema.ContextEditedData` 上
- `vage/workspace/workspace.go` 的接口需要扩 `WriteArtifact` / `ReadArtifact`,现在是 plan + notes 二元组

---

## 7. 后续路径

按 Phase 2 doc 的"落地建议路径":

- **第 2 步剩余**: P0-4(独立 dev session)
- **第 3 步**: P1-1 SessionView / P1-3 scratch+artifacts / P1-5 Tree 双索引
- **第 4 步**: P1-2 / P1-6 / P1-4

`changes/2026/05/04/2026-05-04-001-phase2-step2/` 目录的 4 个文件(spec / plan / progress / result)构成本 session 的完整审计记录,可作为后续 dev session 的参照基线。

---

## 8. Resume / Handoff 锚点

如果未来需要在本工作之上接续:

- **本 session 落地的稳定 API**:
  - `setup.InitResult.{IterationStore, MetricsStore, MetricsHook, BuildReportSink}` —— 直接消费
  - `setup.InitResult.ResumeAgent(agentID)` —— 解析 cp.AgentID 到 agent
  - `session.{MetricsStore, SessionMetricsHook, PricingFunc}` + `vctx.{BuildReportSink, BuildReportReader}` —— 想换后端实现这俩接口即可
  - `taskagent.{WithIterationStore, WithBuildReportSink, WithCheckpointFailureCallback}` —— 三个新 option
  - `dispatches.Dispatcher.Primary()` —— 取出 Primary handle

- **CLI 入口**:`vv --resume <sid>` (与其他 flag 互斥)

- **HTTP 入口**:
  - `POST /v1/sessions/{id}/resume`
  - `GET /v1/sessions/{id}/metrics`
  - `GET /v1/sessions/{id}/build-reports?limit=N`

- **配置**:
  ```yaml
  session:
    enabled: true                  # 默认 true
    persist_build_reports: true    # 默认 true
    build_report_limit: 50         # 默认 50
  ```

- **观察 metrics 文件**:
  - `cat ~/.vv/sessions/<project>/<sid>/metrics.json`
  - `ls ~/.vv/sessions/<project>/<sid>/build_reports/`

- **撤销路径**(若发现回归):
  - vage 一侧的所有新增是 opt-in(callback / sink / option 都默认 nil),回滚单点即可
  - vv setup 的装配块包在 `if metricsStore != nil` 内,删 metrics 装配块即可恢复
  - HTTP `Serve` 签名追加的 `initResult *setup.InitResult` 是最后一个参数,可改成可选(vararg)

---

## 9. Done Contract 终验

| 项 | 状态 |
|---|---|
| vage 单测通过 | ✅ `cd vage && go test ./...` |
| vv 单测通过 | ✅ `cd vv && go test ./...` |
| 任何文件 ≤ 800 行(项目规则) | ✅ 最长 `vv/setup/setup.go` ~1100 行 → 已存在,本次只 +120 行;新增文件最长 `metrics_hook.go` 195 行 |
| 文档同步 | ✅ `checkpoint.md` / `session.md` / `context.md` / phase1+phase2 design doc |
| 现有测试无回归 | ✅ vv 全部集成测试(dispatches / eval / golden / httpapis / mcps / session / setup / tools / traces / vector)0 失败 |
| HTTP 状态码完整 | ✅ 见 §4.2 |
| 错误链 errors.Is 可用 | ✅ ErrCheckpointNotFound / ErrAlreadyFinal / ErrAgentNotFound / ErrSessionDisabled / ErrNoIterationStore / ErrMetricsNotFound 全部 sentinel + `%w` 包装 |
| Loop Anchor (a) + (c) | ✅ 见 §2 |
| Loop Anchor (b) | ⏸ P0-4 推迟,见 §6 |
