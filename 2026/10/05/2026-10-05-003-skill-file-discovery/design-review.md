# 设计综合评审 —— Skill 文件发现 + 会话激活

> 日期：2026-10-05

| # | 初稿风险 | 裁决 |
|---|---------|------|
| 1 | 双注册表（vv Skill vs vage Def）漂移 | **接受双写**。Manager 只认 vage Registry；worker 只认 vv SkillRegistry。单一 `LoadSkillStack` 同源填充。 |
| 2 | 把 `allowed_tools` 原样写入 vage Registry 会让 TaskAgent 收窄工具面，违反 AGENTS-R11 / 验收 4 | **剥离后再 Register**。Warn 日志必须含 `allowed_tools` 字样。 |
| 3 | vage `Activate` 不幂等 | 工具层先查 `ActiveSkills`，已激活直接成功；不改 vage。 |
| 4 | vage 事件信封 `SessionID` 为空，session/metrics 丢事件，验收 3 失败 | vv 适配器从 payload 回填信封。不改 vage（框架缺口记一笔，本期不发 vage 版）。 |
| 5 | `skill_dir` 留空时 Primary 工具面因 `use_skill` 变化，与「逐字节不变」字面冲突 | 「不变」约束在 **worker 工具面 + 两个内置 skill 指令**。`use_skill` 是本期产品能力，空目录时 enum 仅为 `research`/`review`。现有「是否包含某工具」测试保持；新增断言 Primary 持有 `use_skill`。 |
| 6 | 需求把「skill 集合是启动期常量」写成 AGENTS-R7 | AGENTS-R7 实际是「描述符声明一次、多处消费」。修订 **Non-goals**，并新增 AGENTS-R13 写清「内置 ∪ skill_dir、无热插拔」。 |
| 7 | Discover 两次（注册表一次、Manager 一次）重复 Warn | `LoadSkillStack` 只 Discover 一次。 |
| 8 | 给 worker 接 SkillManager | **不接**。worker 即用即弃，激活无处持久；worker 仍用 spec.Skills → `Instructions()`。 |
| 9 | 目录不存在 / 非法 SKILL.md 阻断启动 | 一律 Warn 跳过，返回含内置的可用 stack。 |
| 10 | 改 vage 让 Activate 幂等或补信封 SessionID | **不改**。本期是装配遗漏，不是框架缺口到必须发版的程度。 |
