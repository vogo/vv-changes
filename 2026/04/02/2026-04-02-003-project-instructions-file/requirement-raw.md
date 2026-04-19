# 项目级指令文件支持 (Project Instructions File)

## 需求描述

vv 当前仅支持全局配置 (`~/.vv/vv.yaml` + 环境变量), 缺少项目级别的自定义指令能力. 用户无法针对不同项目定制 agent 的行为、规范和上下文.

需要支持在项目工作目录中放置指令文件 (如 `VV.md`), vv 启动时自动读取并注入到 system prompt 中, 使 agent 能够理解项目特有的约定、架构、编码规范等.

## 对比分析

### Claude Code 的实现

Claude Code 在 Prompt Assembly 流程中, 将 `CLAUDE.md` 作为 Dynamic Section 之一注入 system prompt:

- **多级加载**: 支持全局 (`~/.claude/CLAUDE.md`)、项目根目录、当前子目录等多层级, 合并注入
- **动态组装**: 每次 API 调用时, 将 CLAUDE.md 内容拼接到 system prompt 的 dynamic sections 中
- **缓存边界**: 通过 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记, 静态部分可被 API 缓存, CLAUDE.md 等动态内容在边界之后
- **高优先级**: PRD 明确标注 "IMPORTANT: These instructions OVERRIDE any default behavior"

### vv 的现状

- 仅有全局 YAML 配置, 无项目级指令
- system prompt 是硬编码的模板, 无动态注入机制
- 用户无法在不同项目间差异化 agent 行为

### 差距

| 维度 | Claude Code | vv |
|------|-------------|-----|
| 项目级指令 | CLAUDE.md, 多级加载 | 无 |
| 指令优先级 | 覆盖默认行为 | 不适用 |
| 动态注入 | 每次 API 调用组装 | 无 |

## 实现建议

### 核心流程

1. **启动时读取**: 在 Application Startup 的 Step 2 (Capture Working Directory) 之后, 新增步骤扫描工作目录下的 `VV.md` 文件
2. **注入 system prompt**: 将文件内容作为额外上下文追加到各 agent 的 system prompt 中
3. **标记为项目指令**: 用明确的标记包裹 (如 `# Project Instructions` 前缀), 使 agent 知道这是用户提供的项目级指令

### 文件约定

- 文件名: `VV.md` (大写, 与项目名一致, 易于识别)
- 位置: 工作目录根 (`working_dir/VV.md`)
- 格式: Markdown 纯文本
- 可选: 未来支持 `~/.vv/VV.md` 全局指令 (本次不要求)

### 影响范围

- **Configuration 模型**: 新增 `project_instructions` 属性 (string, 可选)
- **Application Startup 流程**: 新增读取 VV.md 步骤
- **Agent 创建**: 各 agent 的 system prompt 追加项目指令内容
- **无破坏性变更**: 文件不存在时静默跳过, 完全向后兼容

### 实现复杂度: 低

- 核心改动: 读取文件 + 拼接字符串到 prompt
- 无新依赖, 无新模型, 无新 API
- 预计改动文件: 3-5 个
