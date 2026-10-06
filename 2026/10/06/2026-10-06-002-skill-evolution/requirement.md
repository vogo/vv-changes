# 需求：从会话日志提取 Skill 提案并经人工确认入库

> 状态：实施中
> 日期：2026-10-06
> 归属领域：`agents`（主）、`session` / `configuration` / `cli` / `http-api`（次）
> 仓库：`vv`（`github.com/vogo/vv`）
> 不涉及：`vage`、`largemodel`

原文见工作区 `.work/todo.md`。计划评审见同目录 `design-review.md`（实现者不得覆盖）。

## 目标

给出一条 **opt-in、零成本默认、人工确认** 的闭环：操作者对一份已经落盘的会话 transcript 显式触发提取 → 从消息与工具轨迹生成可复用的 Skill 提案 → 语义/词面去重 → 人确认后写入 `agents.skill_dir/<name>/SKILL.md` → 运行期注册进现有 `SkillRegistry` + vage `skill.Registry`，并刷新 Primary 的 `use_skill` / `spawn_worker` schema enum。下一轮 LLM 调用即可看见新 ID。不重启进程。

## 范围

本迭代交付 **显式提取 + 提案队列 + 人工确认 + 写盘 + 热注册**。

| 项 | 本轮 |
|----|------|
| 触发 | 仅显式：CLI `/skill-extract [session_id]`；HTTP `POST /v1/sessions/{id}/skill-extract`。无 hook、无定时器 |
| 资格门 | latest checkpoint `Final && StopReason==complete`；主链 checkpoint 数 ≥ `min_turns`；至少一次工具调用；工具 `IsError` 率满足 `min_tool_success_rate`。工具计数来自 latest checkpoint 的完整 Messages。无 eval 综合分 |
| 提取 | 生产默认 `LLMExtractor`。测试用 stub / `HeuristicExtractor`（不接到 Init） |
| 去重 | 内置 ID 永不覆盖。已注册文件 skill：向量可用且非 HashEmbedder 时余弦；否则 name 精确 + description Jaccard |
| 确认 | 磁盘队列 `<skill_dir>/.proposals/`。CLI `/skill-proposals` / `/skill-approve` / `/skill-reject`。HTTP 四端点。未装配不挂路由 → 404 |
| 持久化 | 批准后写 `<skill_dir>/<name>/SKILL.md`。`allowed_tools` 不写进 frontmatter |
| 生效 | 批准路径调用与启动期相同的 merge（剥离 AllowedTools）写入同一份 vv / vage Registry，再 Refresh Primary schema |
| 配置 | 顶层 `skill_evolution.enabled` 默认 **false**。开启时强制 `session.enabled` 且 `agents.skill_dir` 非空 |
| 包名 | `vv/skillevolve` |

## 非目标

- 会话结束自动触发（无 `SessionEnded`；不挂 `EventAgentEnd`）
- `auto_approve`（配置为 true 则 Validate 失败）
- 定期批量扫描、跨会话聚类
- 用 `vv/eval` 给现场会话打分；不做进化前后质量对比
- Skill 的 `allowed_tools` 进入 vage Registry 或 SKILL.md frontmatter
- `RequiredContextSources` 生效、提取 `Resources`
- 用 `vv/memories` 存提案
- 仅为进化打开向量子系统
- 给 agent 增加 `skill_extract` 工具
- 改 MCP 工具面、改 vage、改 largemodel
- 把 sibling `replace` 写进已发布的 `go.mod`

## 验收要点

- 默认关：不创建 Engine、不建 `.proposals`、HTTP 四路径 404、CLI 提示未配置且拦截斜杠
- 开启却缺 session / skill_dir：Validate 失败
- `auto_approve: true`：Validate 失败
- YAML 数值 0 → 默认（min_turns 5、min_tool_success_rate 0.9、similarity 0.85）；env `VV_SKILL_EVOLUTION_ENABLED` 覆盖 YAML false
- 合格会话 Extract 入队 pending；不合格不入队，HTTP 409 `not_eligible`
- Approve 写 SKILL.md，vv 与 vage 都有该 ID，Activation 里 AllowedTools 为空，frontmatter 无 `allowed_tools`
- 热注册后 `use_skill` / `spawn_worker` enum 含新 ID（含 Truncating 包装）；worker validate 接受新 ID
- 第二次同类提取 → duplicate，不能 approve
- Description、Instructions 与渲染后的 SKILL.md 不含用户消息里 ≥12 字符连续 token
- 内置 `review` / `research` 不被覆盖
- HeuristicExtractor 存在且单测覆盖，Init 不使用它
