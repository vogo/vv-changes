# Phase 2 第 2 步 —— Spec

> 范围: P0-3 (Resume vv wiring) + P0-4 (ContextEditor 增强) + P0-5 (会话级可观察性)
> 引用: [`doc/design/session-context-solution-phase2.md`](../../../../../doc/design/session-context-solution-phase2.md) §P0-3/P0-4/P0-5
> 模式: SDD-Lite `deep`

---

## 1. 模型自述（Restate）

把 Phase 1 已经"骨架就位"但还差最后一公里的三件事一次性补完,使**长任务在 vv 中真正可生产用**:

1. **P0-3 Resume**: vage 已实现 `checkpoint.IterationStore` + `TaskAgent.Resume(ctx, sessionID)`,但 vv 没有装配 IterationStore,也没暴露 CLI/HTTP 入口。结果是:即使设了 `session.enabled: true`,长任务进程崩溃后**没有任何方式续接迭代**。当前 `vv --session <id>` 仅做 UI 历史回放(`PrepareSessionID` → meta lookup),不重启 ReAct 循环。本期要把 IterationStore 接到 setup,新增独立的 `--resume <sid>` CLI 入口与 `POST /v1/sessions/{id}/resume` HTTP 端点。
2. **P0-4 ContextEditor 增强**: 现有 `ContextEditorMiddleware` 只折叠"超过 N 轮的老 tool_result"。在工具密集场景下还有两个让上下文撑爆的源:(a) 同一文件被后续 write/edit 改动后,旧的 read 结果**仍然占着**完整字节;(b) 单条 tool_result(如全文件读、长 grep)单条就能把 context 打满。本期要补 **stale 标记**(基于资源 ID 的"被覆写就折叠")与 **单条转引用**(超长内容外移到 `artifacts/elided/<id>.txt`,prompt 只保留指针)。
3. **P0-5 可观察性**: hook 体系完整、`BuildReport` 已生成,但 session 级的"累计 tokens/cost/时长/重启次数"没有标准化指标聚合,checkpoint 失败只有 `slog.Warn` 没有结构化计数,BuildReport 也没 per-turn 落盘。本期把这三块补成可被 HTTP `GET /v1/sessions/{id}/metrics` 一口聚合查询的形态。

---

## 2. 核心目标(Loop Anchor)

**让一个长任务可以**:

- (a) 进程崩溃 → 重启 → `vv --resume <sid>` 接着跑(P0-3);
- (b) 跑很久 → context 不被"老旧、过期、超长"的 tool_result 拖垮(P0-4);
- (c) 跑完后 → `GET /v1/sessions/{id}/metrics` 一眼看清 token/cost/时长/被 resume 几次/checkpoint 链是否健康(P0-5)。

每件事都独立可验证、可回滚,一件失败不阻塞另两件。

---

## 3. 非目标(Non-Goals)

- ❌ Fork(`POST /v1/sessions/{id}/fork`) → P2-1
- ❌ Interrupt / HITL → P2-2
- ❌ scratch/ 工作区(只做 artifacts/elided/ 的最小子集) → 留给 P1-3
- ❌ Notes ↔ memory.Store 双向桥 → P1-4
- ❌ SessionView 子 agent 隔离 → P1-1
- ❌ Resume 接受 RunOptions 覆盖 → 与 §4.5 偏差表保持一致,本期沿用 agent 默认值
- ❌ Prometheus / OpenTelemetry exporter → 只暴露 JSON 指标,后续按需加适配
- ❌ Resume 时已有的 `SessionResumeMode`(UI 历史回放)的语义改造 → 共存,不互相替代

---

## 4. Done Contract

每一项都必须**由证据证明**:

### P0-3 Resume

- ✅ 单测: `setup.Init` 在 `cfg.Session.Enabled=true` 时构造 `FileIterationStore`,根目录为 `<sessionRoot>/<id>/checkpoints/`;sub-agents(coder/researcher/reviewer)的 TaskAgent 都注入了同一 IterationStore。
- ✅ 单测: `--resume <sid>` 能找到该 session 最新 checkpoint 的 `AgentID`,把请求路由到该 agent 并调用 `Resume()`。
- ✅ 单测: HTTP `POST /v1/sessions/{id}/resume` 同步返回 RunResponse,流式返回 SSE 事件流。
- ✅ 单测: `ErrAlreadyFinal` → HTTP 409 + body `{"error":"session already finalized","code":"already_final"}`,CLI 输出 "session already finalized; start a new session or fork (not yet supported)"。
- ✅ 单测: `--resume` 时 `SessionStore` 未启用 → CLI exit 1 + 友好提示。
- ✅ 集成: 一个手工触发的"假设进程崩溃"用例(模拟:跑 1 个 iteration → 不调 finalize → 退出 → resume),`assistant.message` 数量在 resume 后增加。

### P0-4 ContextEditor 增强

- ✅ 单测: 同一 `resource_id`(如文件路径)被多次 read,且后续出现 write tool_call 影响该资源时,**旧的 tool_result 被折叠**为 `[context_edited: tool_result <id> stale (file <path> modified by <tool_call_id>), <N> bytes]`,事件 `Strategy = "stale_resource"`。
- ✅ 单测: 单条 tool_result 的 `Content.Text()` 长度超过 `WithMaxBytesPerMessage(N)` 时,正文外移到 `<sessionRoot>/<sid>/workspace/artifacts/elided/<msg-hash>.txt`,prompt 改为 `[elided: see artifacts/elided/abc123.txt, 12.3 KiB]`,事件 `Strategy = "elide_to_artifact"`。
- ✅ 单测: 没有 workspace 时 `WithMaxBytesPerMessage` 优雅降级为内联占位符(同 stale 风格,只截断不外移)。
- ✅ 集成: 三种策略可叠加(同次 `edit()` 可同时产生 stale + elided + keep_last_k 编辑)。

### P0-5 可观察性

- ✅ 单测: `session.SessionMetrics` 结构持久化到 `<sessionRoot>/<id>/metrics.json`,字段含 `prompt_tokens / completion_tokens / total_tokens / cost_usd / active_seconds / resume_count / checkpoint_save_failures / context_edits / elided_artifacts`。
- ✅ 单测: `SessionMetricsHook` 监听 `EventAgentEnd / EventCheckpointWritten / EventContextEdited` 自动累加;Resume 路径 `+1 resume_count`。
- ✅ 单测: 每 turn 的 `BuildReport` 落到 `<sessionRoot>/<id>/build_reports/<NNNNNN>.json`(append-only 序列号),配置项 `session.persist_build_reports` 控开关,默认 true 但有 LRU 上限(默认保留最近 50 份)。
- ✅ 单测: Checkpoint save 失败 → `metrics.checkpoint_save_failures += 1` + `slog.Warn`(保留)。
- ✅ HTTP: `GET /v1/sessions/{id}/metrics` 返回聚合 JSON;`GET /v1/sessions/{id}/build-reports?limit=20` 返回最近 N 份 BuildReport。

### 通用

- ✅ `make build` 在 vage/ 与 vv/ 都通过(format + lint + test + license-check)。
- ✅ 不破坏现有测试(回归测试全绿)。
- ✅ 单文件 ≤ 800 行(项目规则)。

---

## 5. 实施计划

详见同目录 [`plan.md`](./plan.md)。

---

## 6. 主要风险

| 风险 | 缓解 |
|---|---|
| **P0-3 Resume 路由歧义**: vv 有多个 sub-agent(coder/researcher/...),CLI `--resume <sid>` 怎么知道路由到哪个? | 从 latest checkpoint 的 `AgentID` 反查 `subAgents[id]`,找不到则报错 + 提示;不允许跨 agent resume(契合 `Resume` 已有的 AgentID 校验) |
| **P0-3 与 `--session` 语义冲突**: 用户可能误以为 `--session <id>` 就是 resume | `--session` 仍只做 UI 回放;`--resume` 必须显式;两者**互斥**(同时给 → exit 1 提示);文档明示区别 |
| **P0-4 stale 检测语义**: vage/tool 现在没有"读写资源 ID"统一概念。要做 stale 检测必须先建一个最小语义层 | 引入轻量约定: `tool.ResourceTracker` 接口由具体工具(file_read / file_write / bash 写文件)选择实现;Editor 在扫描时按 `tool_call_id → resource_ids` 表判断;不实现的工具继续走 keep_last_k 老路。**先窄后扩**——本期只在内置 `fs.ReadFile` / `fs.WriteFile` / `fs.EditFile` 上挂 |
| **P0-4 artifacts/elided/ 与 P1-3 冲突**: P1-3 计划做完整的 artifacts/ 工具集 | 本期只暴露**内部** API `workspace.WriteArtifact(sid, name, content)` 给 Editor 用,**不**注册 LLM 工具,不阻塞 P1-3 的设计 |
| **P0-5 BuildReport 写满磁盘**: 每 turn 一份 JSON,长任务上千 turns 会爆 | LRU + 配置上限;默认 50 份;`persist_build_reports: false` 完全关闭 |
| **跨模块改动并发**: vage/checkpoint + vage/largemodel + vage/session/metrics + vage/workspace 都要动,vv 全装配 | 严格按"先 vage 单测 → 再 vv 装配 → 再 HTTP/CLI"顺序,每层独立提交;失败可只回滚 vv 装配层而保留 vage 增量 |

---

## 7. Validation 清单(执行后回写到 result.md)

- [ ] vage 单测通过 (`cd vage && make test`)
- [ ] vv 单测通过 (`cd vv && make test`)
- [ ] vage lint 通过
- [ ] vv lint 通过
- [ ] 集成手工: 模拟崩溃 + resume 流程跑通
- [ ] 集成手工: 跑一个长 tool 调用,检查 `artifacts/elided/` 落盘 + prompt 占位符
- [ ] 集成手工: `curl GET /v1/sessions/{id}/metrics` 返回非空指标
- [ ] 任何文件 ≤ 800 行
- [ ] `vv/CLAUDE.md` 与 `vage/.doc/checkpoint.md` / `largemodel.md` / `session.md` 文档同步更新
- [ ] phase2 doc 中 P0-3/P0-4/P0-5 标注 ✅
- [ ] phase1 doc §4.3 / §4.5 / §4.7 状态行更新

---

## 8. Resume / Handoff 锚点

如果本 session 中途暂停,接续者需要的最小信息:

- 本 spec 路径: `changes/2026/05/04/2026-05-04-001-phase2-step2/`
- 当前阶段: 见 `progress.md`(每完成一个 P0 step 更新)
- 关键事实:
  - vage/checkpoint 包已实现,只缺 vv 装配 + 入口
  - vage/largemodel/context_editor.go 已经是 keep_last_k 实现,新增策略**复用**已有结构
  - `session.SessionStore` 没有持久化"指标"概念,需要新文件 `vage/session/metrics.go`
  - HTTP 路由集中在 `vv/httpapis/sessions.go`,新增端点遵循同样 `func handleX(store ...) http.HandlerFunc` 模式
- 还没动:Phase 2 第 3 步及以后(P1 系列)。
