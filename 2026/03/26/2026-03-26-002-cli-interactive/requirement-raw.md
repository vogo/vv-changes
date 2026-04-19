# 让 vv 支持命令行交互

## 架构建议

```
vv/
├── main.go          — 根据参数选择 HTTP 模式或 CLI 模式
├── cli/
│   ├── app.go       — bubbletea 主程序（Model/View/Update）
│   ├── input.go     — 用户输入处理、命令解析
│   ├── output.go    — agent 输出渲染（流式、Markdown）
│   └── confirm.go   — 用户确认/选择交互（嵌入 huh）
├── agents/          — 现有 agent 逻辑（不变）
├── config/          — 现有配置（不变）
└── tools/           — 现有工具（不变）
```

## 核心流程

1. 用户在 CLI 输入指令
2. 通过 bubbletea 消息机制将请求发给 agent（直接调用，不走 HTTP）
3. Agent 执行过程中，通过 channel/callback 流式推送执行信息（计划、工具调用、中间结果）
4. 需要确认时，暂停 agent 执行，通过 huh 弹出交互选择
5. 用户确认后继续执行

## Clarified Assumptions (Analyst Decisions)

The following decisions were made during requirement analysis:

1. **Q: Should CLI mode be the default?** A: Yes. CLI interactive mode is the default (`--mode cli`). HTTP mode requires `--mode http`. Rationale: vv is a developer tool, and direct terminal interaction is the most natural workflow.

2. **Q: What happens with Ctrl+C?** A: First Ctrl+C during agent processing cancels the current run and returns to idle. Second Ctrl+C (or Ctrl+C while idle) exits the application. This follows standard CLI tool conventions.

3. **Q: Can CLI and HTTP modes run simultaneously?** A: No. They are mutually exclusive in a single process. This keeps the initial implementation simple.

4. **Q: Which tools require confirmation by default?** A: None. Tool confirmation is opt-in via `confirm_tools` configuration. The operator decides which tools need confirmation.

5. **Q: Is conversation history persisted across sessions?** A: No. Conversation exists only in memory for the current session. Persistent history is a future enhancement.

6. **Q: How is markdown rendered in the terminal?** A: Basic terminal formatting (bold, italic, code blocks with syntax highlighting where possible). Full fidelity markdown rendering is not expected in a terminal.
