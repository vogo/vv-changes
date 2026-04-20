# Raw Requirement: vv 集成 MCP Server 模式 (P1-1)

## 来源
`doc/prd/feature-todo.md` 中 P1-1 条目：

> **P1-1 · vv 集成 MCP Server 模式**
> - 类别: 能力接通
> - 依赖项: 无
> - 难度: 低
> - 可复用能力: `vage/mcp/server`（框架层已实现）
> - 排序理由: vage 已有完整 MCP Server 实现；vv 只需在启动时注册并暴露；PRD 明确列为"planned for future"；接通成本极低

## 用户意图（对话中给出的指令）
> 读取 `doc/prd/feature-todo.md`，获取第一个未实现的需求，基于这个需求去做一个详细的需求调研，了解类似产品，做这个需求都有些哪些实现？注意些什么问题？然后调研相关行业，这些功能的实现都有什么具体的方案？在设计方案的时候做相关的参考，然后实现完了以后，将这个功能，从 to do 文件中移除放到 implement 文件中。

## 初步理解
- 目标：将 vv 升级为既可作为 MCP Client 使用别人工具，也可以作为 MCP Server 向外暴露自己的 Agent/工具能力。
- vage 侧已有 `vage/mcp/server` 的框架层实现，vv 侧仅需完成"启动时注册并暴露"的集成工作。
- 需要做一份详细的同类产品调研（Claude Code、Cursor、Cline、Continue、Goose 等）与行业方案调研，指导本需求的设计方案。

## 调研待办（analyst 阶段执行）
- 调研 Claude Code / Cursor / Cline / Continue / Goose / Zed AI 等 Agent CLI/IDE 工具作为 MCP Server 的做法。
- 调研主流 MCP Server 暴露方式（stdio、HTTP+SSE、Streamable HTTP）及其典型使用场景。
- 调研 MCP Server 安全治理做法（auth、工具白名单、路径约束、凭据过滤复用）。
- 结合 vv 既有架构给出启动模式、生命周期、配置入口的建议。
