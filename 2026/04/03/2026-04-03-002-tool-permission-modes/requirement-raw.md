# Tool Permission Modes

## 需求描述

为 vv agent 引入工具权限模式(Tool Permission Modes), 替代当前简单的 `confirm_tools` 列表机制. 通过预定义的权限模式, 根据工具的读写属性自动决定是否需要用户确认, 大幅减少用户在日常使用中的确认疲劳.

## 对比分析

### Claude Code 的权限模式

Claude Code 实现了 5 种权限模式:

| 模式 | 行为 |
|------|------|
| **Default** | 只读工具自动批准; 写入/执行工具需要用户确认 |
| **AcceptEdits** | 只读工具 + 文件编辑(write/edit)自动批准; bash 等仍需确认 |
| **Auto** | 所有工具自动批准(信任模式) |
| **Plan** | 仅允许只读工具, 写入工具被拒绝 |
| **Bypass** | 跳过所有权限检查(开发/调试用) |

此外还支持:
- 每次确认时可选 "Allow Always"(本次会话内不再询问该工具)
- 基于工具属性(isReadOnly, isDestructive)的自动分类
- 三层安全架构(分类器 → Hook → 权限决策)

### vv agent 当前状态

- 仅有 `confirm_tools` 配置项: 一个需要确认的工具名列表
- 不区分工具的读写属性
- 每次调用列表中的工具都会弹出确认对话框
- 无"记住选择"功能, 同一工具需反复确认

### 差距

1. **无模式概念**: 用户不能整体切换权限策略
2. **无读写分类**: read/glob/grep 等安全工具仍可能需要确认
3. **无会话级记忆**: 已确认的工具下次调用仍需确认
4. **无只读模式**: 无法限制 agent 只使用只读工具

## 实现建议

### 1. 工具读写属性标记

为每个工具添加 `ReadOnly` 属性:
- **只读工具**: read, glob, grep, ask_user
- **写入工具**: write, edit
- **执行工具**: bash

### 2. 权限模式定义

引入 4 种权限模式:

```go
type PermissionMode string

const (
    PermissionDefault     PermissionMode = "default"      // 只读自动批准, 其余确认
    PermissionAcceptEdits PermissionMode = "accept-edits"  // 只读+编辑自动批准, bash确认
    PermissionAuto        PermissionMode = "auto"          // 全部自动批准
    PermissionPlan        PermissionMode = "plan"          // 仅允许只读工具
)
```

### 3. 权限决策逻辑

```
收到工具调用请求:
  1. 如果模式是 auto → 直接批准
  2. 如果模式是 plan 且工具非只读 → 直接拒绝
  3. 如果模式是 default 且工具只读 → 直接批准
  4. 如果模式是 accept-edits 且工具只读或是编辑工具 → 直接批准
  5. 如果工具在 session_allowed 集合中 → 直接批准
  6. 否则 → 弹出确认对话框
```

### 4. 会话级记忆

在确认对话框中增加选项:
- **Allow**: 本次批准
- **Allow Always**: 本次会话内该工具不再询问(加入 session_allowed 集合)
- **Deny**: 本次拒绝

### 5. 配置方式

```yaml
# vv.yaml
permission_mode: default  # default | accept-edits | auto | plan
```

也可通过环境变量: `VV_PERMISSION_MODE=accept-edits`

CLI 启动参数: `vv --permission-mode accept-edits`

运行时切换: `/permission <mode>` 命令

### 6. 实现范围

**需要修改的模块**:
- `vv/config`: 添加 permission_mode 配置项
- `vv/agents` 或 `vv/dispatches`: 工具调用前的权限检查逻辑
- `vv/cli`: 确认对话框增加 "Allow Always" 选项
- `vage/tool`: 为 Tool 接口添加 ReadOnly 属性

**预计改动量**: 中等偏低, 主要是在现有工具确认流程中插入模式判断逻辑.

## 预期收益

1. **减少确认疲劳**: default 模式下, read/glob/grep 不再弹确认框
2. **灵活安全策略**: 熟练用户可选 accept-edits 或 auto 模式提升效率
3. **只读审查模式**: plan 模式下 agent 无法修改任何文件, 适合代码审查场景
4. **会话级免确认**: "Allow Always" 避免同一工具反复确认
