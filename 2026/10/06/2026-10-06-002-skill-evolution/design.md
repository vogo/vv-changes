# 设计：从会话日志提取 Skill 提案并经人工确认入库

> 日期：2026-10-06
> 对应需求：同目录 `requirement.md`
> 计划来源：`.work/todo.md`。本文件记录 D1–D10 与明确不做；不另做一轮独立设计。

不改 vage、不改 largemodel。包名 `vv/skillevolve`。

## 明确不做

SessionEnded / AgentEnd 自动提取、auto_approve、批量扫描、eval 质量对比、向量强制索引、Resources、prompt 字节级 diff、RequiredContextSources 生效、MCP、给 agent 的提取工具、新增 largemodel EventType、把自定义事件加入 control-plane 白名单、用 `schema.EmitCustomData`。

## D1. 仅显式 user-path 提取

触发面：

- CLI：`/skill-extract` 使用当前 `App.sessionID`；`/skill-extract <id>` 读任意已落盘会话（须 `vage/session.IDPattern` 合法）。四条 `/skill-*` 在 `/memory` early-return **之前**分流。`SkillEvolve==nil` 时打印 `Skill evolution is not configured.`，仍拦截斜杠。
- HTTP 四端点仅 `InitResult.SkillEvolve != nil` 时挂载（HTTP-R6 / HTTP-R10）。未挂 = **404**，不要 501：
  - `POST /v1/sessions/{id}/skill-extract`
  - `GET /v1/skill-proposals`
  - `POST /v1/skill-proposals/{id}/approve`
  - `POST /v1/skill-proposals/{id}/reject`

挂载后错误：非法 id 400 `bad_request`；transcript / 未知提案 404 `not_found`；资格门 409 `not_eligible`；非 pending 409 `not_pending`；LLM 失败 502 `extract_failed`。提取在请求 goroutine 同步跑。

## D2. 资格门只用 transcript

`Analyze` 读 `sessionlogs.Store.Load(sid,"")` 与 `Store.List`。合格当且仅当：

1. latest `Final==true` 且 `StopReason==schema.StopReasonComplete`
2. `Turns >= min_turns`（Turns = `len(Store.List)`，含非 final）
3. `ToolCalls >= 1` 且 `SuccessRate >= min_tool_success_rate`

`Eligible` 在 `Final==false` 时 Reason=`not_final`（即使 `StopReason==complete`）；`StopReason` 非 complete 时 Reason=`stop_reason`。工具统计只扫 **latest** checkpoint 的完整 Messages（累计历史，不要加总全部 checkpoint）。YAML/结构体 0 与省略相同：`applyDefaults` 填 min_turns=5、min_tool_success_rate=0.9。子代理 run 不参与资格门，但 ToolSequence 可含子代理步骤。不引入 `min_session_score`。

## D3. Extractor 接口，测试不打模型

`Extractor` / `ExtractorFunc`。生产 `NewLLMExtractor(caller, model)`：prompt 含 `feat.UserTask` 并写明 do-not-copy；只返回 JSON；解析失败 repair 一次。`normalizeExtracted`：kebab-case（`_` → `-`）、`ValidateName`、撞内置 → `-evolved`、Instructions ≤500 行、SKILL.md ≤50KiB、Metadata 强制 `source_session` / `extracted_at` / `origin=evolution`、从 Description、Instructions 以及渲染后的 markdown 删除 `feat.UserTask` 里 ≥12 字符连续子串。`HeuristicExtractor` 必须存在，**不**接到 `installSkillEvolution` / `Init`。

## D4. AllowedTools 永不进入 Registry 或 frontmatter

`WriteSkillMD` frontmatter 只有 `name` / `description` / `license` / `metadata`；工具名进 `metadata.observed_tools`。写盘用 `os.Mkdir` 独占创建 `<skill_dir>/<name>/`（父目录可 MkdirAll）；目录已存在则返回 `skill exists`，**不** `RemoveAll` 他人目录。仅当本调用 `created==true` 时，Load/Register 失败才删除该目录。`Engine` 对 Extract / Approve / Reject 加互斥锁，避免并发 Approve 同名删掉胜者文件。`registries.RegisterFileSkill` 剥离 AllowedTools；先 vage `Register` 再 vv `Register`，vv 失败则 vage `Unregister`。启动路径 `mergeOneFileSkill` 包装它并 Warn+skip。AGENTS-R11 正文不改。

## D5. 去重：内置硬挡；向量可选；HashEmbedder 当没有

1. 规范化 name 已在已注册表 → duplicate similarity 1
2. 否则 `VectorStore` 与非 Hash embedder 都在 → 余弦 `MinScore=similarity_threshold`（0 → 0.85）
3. 否则 description Jaccard
4. 否则 pending

`isHashEmbedder`：`*vector.HashEmbedder` 或 `NamedEmbedder.ModelName()==HashEmbedderModelName`。批准成功后可选 `Add` 一篇 `kind=skill` 文档；失败 slog.Warn，不回滚 SKILL.md。

## D6. 提案队列是 `skill_dir/.proposals/`

`FileQueue`：目录 0o700、文件 0o600。`duplicate` 也落盘，不能 approve。同 session 允许新提案；去重对已注册 skill，不对 pending。Discover 忽略无 SKILL.md 的 `.proposals`。

## D7. 热生效 = 同一 Registry 指针 + 覆盖 ToolDef

`SkillStack.VageRegistry` 必须是 Manager 持有的同一指针。`tool.Registry.Get` 只返回 ToolDef，故 `UseSkillTool` / `SpawnWorkerTool` 存 handler；`Refresh` 是方法，打在包装后的 `finalToolReg`。`Register*` 先写未包装 inner，再 Refresh 包装表。修订 AGENTS-R13：允许进化批准追加文件 skill；永不覆盖 `review`/`research`；永不卸载。

## D8. 配置与装配：interrupt 同构的零成本门

`configs.SkillEvolutionConfig`：`enabled` 默认 false（不要 `*bool`）。`VV_SKILL_EVOLUTION_ENABLED` 经 `applyBoolValWarn` 覆盖 YAML false。数值 0 → 默认。`auto_approve==true` Validate 失败。`installSkillEvolution` 在 `setup.New` **末尾**、`buildPrimaryAssistant` 之后，用 New 的 `llm` 参数，不进 `runInstallers`，不用 `assembly.wrappedLLM`。`enabled=false` 返回 `(nil, nil)`，不 new 队列、不 Mkdir `.proposals`。生产 Extractor 是 `LLMExtractor`。

## D9. 事件：`EventCustom`

`hook.Manager.Dispatch` + `schema.NewEvent(EventCustom, "", sessionID, CustomEventData{Name, Payload})`。名字：`skill.proposed` / `skill.registered` / `skill.rejected`。Payload 不含 Instructions 全文与用户原文。不要 `EmitCustomData`。不要加入 `controlPlaneEvents`。

## D10. 不改 vage、不改 spawn_worker 成功路径

热注册后 `WorkerSpec.validate` 走活的 `ValidateRef`。新增测试锁住：运行期 `SkillRegistry.Register` 后 validate 成功、`buildWorker` 提示含 Instructions。`use_skill` / `spawn_worker` handler 成功路径逻辑不变。
