# 设计：Skill 文件发现 + 会话激活

> 日期：2026-10-05
> 对应需求：同目录 `requirement.md`

## 1. 范围

只在 vv 装配层接线。vage `skill.FileLoader` / `DefaultValidator` / `Manager` / `taskagent.WithSkillManager` 已完备，**不改 vage**。

## 2. 配置

`AgentsConfig.SkillDir string \`yaml:"skill_dir,omitempty"\``；env `VV_AGENTS_SKILL_DIR`。留空 = 不扫盘，两个内置 skill 照常注册。

## 3. 单一构造点

`registries.LoadSkillStack(ctx, dir, dispatcher)` 一次产出：

| 产物 | 用途 |
|------|------|
| vv `SkillRegistry` | `spawn_worker` enum、DAG `skills` 校验、worker `Instructions()` |
| vage `skill.Manager` | Primary `use_skill` → `Activate`；TaskAgent 下一轮快照注入 `<skill name>` |

转换：`Def.Name→ID`，`Description` / `Instructions` 原样。**写入 vage Registry 前清空 `AllowedTools`**（AGENTS-R11）。用户写了 `allowed_tools` 打 Warn，日志含该字段名。

发现 / 校验 / 与内置 ID 冲突：`slog.Warn` 跳过该条，**不阻断启动**；内置优先。`DefaultSkills()` 签名与行为不变；`DefaultSkillsWithFileSkills` 是只返回 vv 注册表的薄封装。

## 4. Primary `use_skill`

- 仅 `buildPrimaryAssistant` 注册；worker 不授予。
- 参数 `{skill}`，enum = `SkillRegistry.IDs()`。
- 已激活：先查 `ActiveSkills`，成功返回（不调用会报错的 `Activate`）。
- 未注册：`ErrorResult`，错误文本含可用 ID。
- 描述与返回文本写明「下一轮生效，本轮系统提示与工具面不变」。
- session ID 取 `schema.SessionIDFromContext`。

## 5. 装配

- `FactoryOptions.SkillManager`；仅 Primary Factory 调 `taskagent.WithSkillManager`。
- `setup.Options` 持同一 Manager + SkillRegistry（单一实例）。
- `WithEventDispatcher` 接到 hook 总线；适配器把 payload 的 `session_id` 填进 `Event.SessionID`（vage Manager 信封字段为空，session/metrics hook 否则会丢弃）。

## 6. 提示与文档

- Primary 系统提示补一句：用 `use_skill` 按需加载，下一轮进入系统提示，不授予工具。
- AGENTS-R11 不变。Non-goals：skill 集合 = 启动期内置 ∪ `skill_dir`，无热插拔（原「内置常量」条款写在 Non-goals，不是 AGENTS-R7）。
- `architecture.md` 能力维度表、`glossary.md` 补 `SKILL.md`。
