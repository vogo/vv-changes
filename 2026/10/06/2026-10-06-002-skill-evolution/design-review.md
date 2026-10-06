# 计划评审：从会话日志提取 Skill 提案并经人工确认入库

- 日期：2026-10-06
- 对象：`.work/todo.md`（开发计划，不是实现）
- 结论：计划已按本次评审修正，可以交给实现。实现者写 `requirement.md` 与 `design.md`，不要覆盖本文件。

评审对照的是当前工作区源码，不是计划里的转述。`vage` / `largemodel` 不改。范围仍是显式提取 + 队列 + 人工确认 + 写盘 + 热注册。没有把 Phase 3 eval、`SessionEnded` / `EventAgentEnd` 自动提取、`auto_approve` 塞回来。

## 1. 相对源码的正确性

**问题（blocker），已改计划。** 下列与现码不一致或会让实现者走错的地方已经写回 `.work/todo.md`。

- **`tool.Registry.Get` 只返回 `(schema.ToolDef, bool)`**（`vage/tool/registry.go` 136–144 行），没有 handler。计划原先写「Get 现有 def + 包内保存的 handler」，但又给出 `RefreshUseSkillSchema(reg, skills, mgr)` 这种拿不到旧 handler 的签名。改为具名结构体 `dispatches.UseSkillTool` / `SpawnWorkerTool`：`Register*` 返回该结构体并保存 handler；`Refresh` 是方法，禁止第二次 `new*Handler`。调用点只有 `setup.go` 与对应测试 / 一条 spawn_worker integration test。
- **Refresh 必须打在包装后的 `finalToolReg`。** `RegisterUseSkillTool` 今天写在未包装的 `toolReg` 上，`finalToolReg := toolRegistryWrapper(...)(toolReg)` 在 546 行才发生。TaskAgent 持有包装后的表。`TruncatingToolRegistry` / `DebuggingToolRegistry` 转发 `Register`；`permissionExecutor` 内嵌 `tool.ToolRegistry`，`Register` 提升转发。
- **`setup.New` 看不见 `assembly.wrappedLLM`。** `Init` 经 `installAgents` 调 `New` 时把 `a.wrappedLLM` 当作 `llm` 参数传入。`installSkillEvolution` 放在 `New` 末尾、`buildPrimaryAssistant` 之后，用 `New` 的 `llm`。不要塞进 `runInstallers`（那时 Primary 工具表还不存在）。
- **事件不要用 `schema.EmitCustomData`。** 它要从 ctx 取 emitter，SessionID 从 ctx 继承；提取/批准不在 agent Run 内。改为 `hook.Manager.Dispatch` + `schema.NewEvent(EventCustom, "", sessionID, CustomEventData{...})`。`Dispatch` 对 nil receiver 直接 return（`vage/hook/manager.go` 64–67 行）。
- **`RegisterFileSkill` 不能只是 `mergeOneFileSkill` 改名。** 现顺序是先 vage `Register` 再 vv `Register`；vv 失败时 vage 侧会残留，Activate 成功但 `ValidateRef` 失败。vage `Registry` 已有 `Unregister`。新函数返回 `error`，vv 失败则 Unregister；启动路径继续 Warn + skip。
- **CLI `handleCommand` 在 `/compact` `/permission` `/budget` 之后，非 `/memory` 直接 `return nil` 交给代理**（`vv/cli/memory.go` 33–35 行）。`/skill-*` 必须在这个 early-return 之前分流，否则违反 CLI-R8。
- **`session.IDPattern` 属于 `vage/session`，不是 vv 类型。** 非法 id → `checkpoint.ErrInvalidArgument`；空会话 / 无 ckpt → `checkpoint.ErrCheckpointNotFound`。不要混成一种 error。
- **工具 part 类型常量是 `schema.MessagePartToolResult`**，不是字符串字面量 `"tool_result"`。
- **`vv/configs/env_test.go` 的 `TestEnvBindings_KeysUniqueAndComplete` 写死全量 `VV_*` 列表。** 加 `VV_SKILL_EVOLUTION_ENABLED` 必须改这个 want，否则必红。
- 若干行号已按 2026-10-06 工作区校正：`ValidateEval` 在 `config.go` 360 行起（不是 341）；`vector.Document` 67 行、`VectorStore` 107 行（不是 18–23）；`SearchOptions.MetadataEquals` 86–96 行。

**通过（修正后仍成立）。**

- `LoadSkillStack` / `mergeOneFileSkill` 在写入 vage Registry 前剥离 `AllowedTools`（`skill_load.go` 70–82 行）；`TestLoadSkillStack_MergesValidSkipsInvalid` 67–68 行断言 Manager 里为空。AGENTS-R11 保持。
- `FileLoader.Discover` 只认下一层子目录里的 `SKILL.md`；无该文件的 `.proposals/` 会被跳过。`StructureValidator` 扫的是 skill 子目录（`BasePath`），不是 `skill_dir` 根。
- `schema.StopReasonComplete` 常量名正确（`message.go` 32 行）。
- HTTP-R9 已占用；新规则用 **HTTP-R10**，不碰撞。HTTP-R6 的子系统列表要追加 skill-evolution，R6 语义（未启用不挂路由）不动。
- `SkillStack` 今天只有 `Registry` + `Manager`；vage Registry 在 `InMemoryManager.registry` 未导出。热注册必须把同一指针暴露为 `VageRegistry`。
- 包装链转发 `Register` 的现码足够支撑覆盖 ToolDef。
- YAML 无 `KnownFields`：旧二进制遇到未知顶层键 `skill_evolution` 会丢弃，不会启动失败。
- vv 当前 tag `v0.1.4`，默认关着发 **v0.1.5** patch 正确。`vv/go.mod` 保持 `vage v0.13.0` / `largemodel v0.9.0`，无 sibling `replace`。

## 2. 会话 transcript 与资格门（高风险）

**问题（blocker），已改计划。** 计划写「从 latest checkpoint 的 Messages 计 ToolCalls」，但没证明 latest 是全文还是最后一轮。

现码：`saveIterationCheckpoint` 把当时 ReAct 的**整份** `messages` 切片写入 checkpoint（`vage/agent/taskagent/checkpoint.go` 53–70 行）。CLI 下一轮用 `Store.Messages`（= `Load(sid,"")` 的 Messages）恢复后再追加用户消息。因此主链最后一条 ckpt 还原后是**累计历史**。资格门的 `ToolCalls` / `ToolErrors` **只扫 latest `Load` 即可**；把所有 checkpoint 的 Messages 加总会重复计数。

`Turns` 另走 `Store.List` 的 ckpt 条数，含 `Final==false` 的 tool-batch 快照，**不是** `RoleUser` 条数。一次带多次 tool-batch 的 Run 更容易过 `min_turns=5`。MVP 接受，计划与风险表已写清。补一条「两轮用户对话、latest Messages 含两轮合计 ToolCalls」的测试，锁住这个语义。

Spill 已被 `decodeMessage` 透明还原，不必自己读 `tool-results/`。

## 3. 内部矛盾

**问题（blocker），已改计划。** D1 写 HTTP 501（子系统未装配），P0-6 / HTTP-R6 写不挂路由所以 404。vv 惯例是 HTTP-R6：interrupt / vector / eval 未启用都不挂。统一为：

- 未装配：**不挂路由 → 404**。不要 501。
- 已挂载：非法 id 400 `bad_request`；transcript / proposal 不存在 404 `not_found`；不合格 409 `not_eligible`；非 pending 409 `not_pending`；LLM 失败 502 `extract_failed`。
- 四端点路径钉死：`POST /v1/sessions/{id}/skill-extract`、`GET /v1/skill-proposals`、`POST /v1/skill-proposals/{id}/approve`、`POST /v1/skill-proposals/{id}/reject`。
- CLI nil 时打印 `Skill evolution is not configured.`，**仍拦截斜杠**。

D1、P0-6、验收、HTTP 文档清单已全部改成同一套。抽出 `mountSkillEvolveRoutes` 以便 httptest 不断端口。

## 4. 配置零值与 env

**问题（blocker），已改计划。** 「`MinToolSuccessRate` 若显式设置且不在 (0,1]；0 表示默认」在 `float64` + `applyDefaults` 之后无法实现：YAML 写 0 和省略都会变成 0.9。vv 现有约定是 0 = 默认（`MaxIterations`、`EffectiveResumeMaxMessages`），不用 `*float64`。三个阈值都按这个做；无法用 0 关闭门。Validate 在填充之后检查 `(0,1]` / `MinTurns>=1`。

`Enabled bool`（不要 `*bool`）足够：默认关。`VV_SKILL_EVOLUTION_ENABLED` 用 `applyBoolValWarn`，**会覆盖 YAML 显式 `enabled: false`**（CONFIG-R1；与 `VV_DEBUG` 同类）。

`installSkillEvolution` 在 `enabled=true` 时再校验 session + `skill_dir` + TranscriptStore：测试直接调 `New` 会绕过 `Load`/`Validate`，不能「假定已合法」。断言 `InitResult.SkillEvolve` 不要为了它去跑完整 `Init`（需要真 LLM client）；installer / `New`+注入 `*sessionlogs.Store` 即可。

## 5. 范围

**通过。** HeuristicExtractor 从「若实现」改为 **必须实现、只给单测、不接线到 Init**。生产默认 `LLMExtractor`。推迟列表仍排除 SessionEnded、auto_approve、eval 质量对比、向量强制、Resources、RequiredContextSources 生效、MCP、给 agent 的提取工具。

## 6. 测试完整性

**问题（important），已改计划。** 验收项都有了具名落点，并补上这些缺口：

- 非法 id vs 空会话两种 error。
- 两轮累计 ToolCalls。
- YAML 0 → 默认；env 覆盖 YAML false；`env_test.go` key 全集。
- `HeuristicExtractor` 无 Caller 给出合法 Name。
- Refresh 走 Truncating 包装。
- `RegisterFileSkill` vv 失败时 vage 不残留。
- CLI 斜杠在 Engine nil 时不 `return nil` 给代理。
- HTTP nil 时四路径 404；抽出 mount helper。

## 7. 安全 / 隐私

**通过（修正后仍成立）。** `.proposals` 与 `SKILL.md` 使用 `0o700` / `0o600`，与 sessionlogs / traces / memories 一致。12 字符连续子串剥离只挡原样复制；改写、同义复述、插入空格不剥。人工确认是第二道闸。不能从 transcript / 工具结果里保证没有密钥。这些写进风险表，不把 MVP 的隐私门做成 NLP。

AGENTS-R11 正文不改。进化写入的 frontmatter 不含 `allowed_tools`。

## 8. 文档与 spec 冲突

**问题（important），已补进文档清单。**

- AGENTS-R13 与 Non-goals 第 88、90 行要改 Skill 热追加，但 **AgentDescriptor / ContextSource / AGENTS-R11 / CLI-R8 / MEM-R\* / HTTP-R1–R9 正文不动**。HTTP-R6 只在子系统列表里追加。CONFIG-R3 追加进化依赖例，保留 `session_tree` 原例。
- `agents-overview.md` 路径正确（`vv/doc/domains/core/agents/agents-overview.md`）。eval 勘误落 `eval/spec.md`，不改 EVAL-R*。
- http-api spec Interactions 暴露表（约 88 行）要补四端点。
- `SkillRegistry` / `DefaultSkills` 源码注释里的「启动后只读 / 无热插拔」必须随 AGENTS-R13 一起改，否则代码注释与 spec 再漂。

## 9. 可行性

**问题（important），已改计划。** 实现者不需要再做架构调研。不存在的符号都写成「创建 X」：`vv/skillevolve` 包、`RegisterFileSkill`、`UseSkillTool` / `SpawnWorkerTool`、`skillSchemaRefresher`、`mountSkillEvolveRoutes`、`HeuristicExtractor`、`installSkillEvolution`。`RegisterUseSkillTool` 签名变化的调用点列全。

## 残余风险（有意留下）

- 12 字符子串剥离挡不住改写后的用户原文。人批是闸。
- transcript / 工具结果里的密钥可能进入 LLM 提取上下文；批准前必须读提案。本轮不做 secret scanner。
- `Turns` 按 checkpoint 条数计，带多次 tool-batch 的短会话更容易过门。
- `MapVectorStore` 进程重启后 skill 向量文档丢失，去重退回 Jaccard。
- Refresh 失败时注册表已有 ID、enum 仍旧，直到重启。handler 的活 `IDs()` 仍可用于校验。
- 若有人在 `.proposals/` 下放 `SKILL.md`，Discover 会当 skill 加载。队列只写 `.json`。
- 回滚 vv 到 v0.1.4：已批准的 SKILL.md 仍会被 Discover 加载（这是 skill-file-discovery 已有能力）；`.proposals/` 被忽略。

## 结论

计划可以实施。上面的 blocker 与 important 都已经写回 `.work/todo.md` 的对应 D#、测试、文档清单和验收项。没有扩大范围，也没有把自动提取 / eval 闭环放回来。状态仍是「计划中」。本评审阶段不改产品代码、不提交、不开 PR。

**实现者不要覆盖本文件。**
