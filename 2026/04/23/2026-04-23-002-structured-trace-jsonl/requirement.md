# 结构化对话轨迹落盘（JSONL）— P1-5 Requirement

## 背景与目标

vv 目前只在终端/文件里打出 `slog` 级别的松散日志（`vv/debugs` 仅 `--debug` 开启时生效，且面向人工调试而非后续机器消费），没有把"一次会话里完整的 Agent 行为轨迹"以结构化、可机读、可回放的形式持久化。

下游多个需求（P1-6 SQLite 存储、P3-5 训练集导出、P4-12 会话录制回放、P2-14 会话检查点）都依赖一份**稳定的、按会话隔离的、append-only 的事件流**作为"数据出口"。P1-5 的目标是建立这个出口：**把 `vage/hook.Manager` 已经在广播的 `schema.Event` 事件，通过一个 AsyncHook 写到本地 JSONL 文件。**

不是建一套通用观测框架（那是 P3-9 OTEL），不是把 trace 推给第三方后端（Langfuse/Phoenix 走 P3-9），不是做可视化（另说）。**只做"把已有事件流落一份到磁盘"。**

## 用户故事与验收标准

### US-1（作为运维/数据侧用户）
> 我打开 `trace.enabled: true` 后，每个会话结束能在本地找到一个独立的 JSONL 文件，里面是本会话发生过的每一个 Agent 生命周期事件。

**验收**：
- CLI 跑一次 `-p "help me"` 后，在默认目录 `~/.vv/traces/<project-hash>/<session-id>.jsonl` 下产生 ≥ 1 个 JSONL 文件。
- 文件每一行是合法 JSON；`jq -c . <file>` 能成功通读。
- 文件首行必含 `"type":"agent_start"`；末行必含 `"type":"agent_end"` 或 `"type":"error"`。
- 关键字段存在：`type`、`timestamp`（RFC 3339 毫秒精度）、`session_id`、`agent_id`、`data`。

### US-2（作为开发者/框架调试方）
> 我关掉 `trace.enabled` 时，不希望多一个 goroutine、多开一个文件句柄、或者对请求延迟造成任何可测量的影响。

**验收**：
- `trace.enabled: false`（默认）时：不打开文件、不启动 consumer goroutine、`hook.Manager` 上不挂额外 AsyncHook。
- 启停行为可以通过 `Init` 的快路径代码走查验证。

### US-3（作为 Agent 使用者）
> 即使我发的请求跑出海量工具调用和 LLM 流式事件，写 trace 也不能卡住 Agent 回复。

**验收**：
- 轨迹写入通过 `hook.AsyncHook` 异步化，`EventChan()` 背后 channel 满时事件被 drop，不阻塞 TaskAgent 的 Dispatch 路径。
- drop 事件有 `slog.Warn`（复用 `hook.Manager.Dispatch` 的既有行为）。

### US-4（作为崩溃/退出场景）
> 进程正常退出时，我希望未刷盘的事件都写进去；进程被 kill 时，即便丢最后几行，已写入的行仍是合法 JSONL。

**验收**：
- `Stop(ctx)` 调用后，consumer goroutine 把 channel 里剩余事件逐条 encode+write，然后 `f.Sync()` + `f.Close()`。
- 文件 I/O 使用行级 `append` + `\n`，一行写入失败不会产生半截 JSON 污染后续行（单行 `encoding/json.Encoder` 调用，失败则写 slog.Warn 并跳过）。

### US-5（作为下游 P3-5 消费者）
> 我能从 JSONL 里重建 "{user_prompt → tool_use 链 → final_response → usage}" 四元组。

**验收**：
- 事件覆盖：`agent_start`、`iteration_start`、`tool_call_start`、`tool_call_end`、`tool_result`、`text_delta`（或 `agent_end.message`）、`llm_call_end`（含 `prompt_tokens/completion_tokens/cache_read_tokens`）、`agent_end`。
- 事件顺序保留（单 session 单 goroutine 消费即可保证串行）。

### US-6（作为三种运行模式的用户）
> CLI、HTTP、MCP 三种模式都应该有相同的 trace 能力。

**验收**：
- trace hook 挂在 `setup.Init` 的公共路径上，三种模式都会触发。
- 单元测试覆盖：在 `setup.Init` 层 mock 一个跑几步的 agent，确认 trace 文件被写入。

## 关键决策（已定默认）

| 决策项 | 选择 | 理由 |
|--------|------|------|
| 默认开关 | 默认关（`trace.enabled: false`） | 与 `debug`/`eval`/`mcp.server` 一致的 opt-in 姿态；隐私 + 磁盘中性 |
| 目录布局 | `~/.vv/traces/<project-hash>/<session-id>.jsonl` | 对齐 Claude Code 心智；project 分桶便于跨 session 聚合；目录可 `trace.dir` 覆盖 |
| 项目哈希算法 | `SHA-256(cfg.Tools.BashWorkingDir)` 前 12 字节 → base32（小写、无 padding） | 稳定、文件系统安全、无奇怪字符 |
| 文件切分粒度 | 一会话一文件（触达 `max_file_bytes` 时切 `.1/.2`） | 语义清晰；下游 P2-14 会话 resume 直接按 session_id 找 |
| 事件范围 | 全量（`schema.Event*` 所有类型） | 飞轮底座宁要多不要少；过滤放到 `trace.event_types` 配置项（MVP 可选） |
| Redaction | 不在本需求做；预留 `RedactFunc` 钩子 | 上游 guard 已清洗；正式脱敏在 P3-5 做 |
| 轮转/清理 | MVP 只做 `max_file_bytes` 分片；不做按天 cleanup | 清理留给 P1-6 SQLite 阶段 |
| Start/Stop 粒度 | 进程级（`setup.Init` / 进程退出） | 与 `hook.Manager.Start/Stop` 语义一致；每个 session_id 懒开句柄 |
| 模式覆盖 | CLI + HTTP + MCP 全覆盖 | AsyncHook 装在公共路径，一次接入三处受益 |

## 范围内

- 新增 `vv/traces/tracelog/` 包，提供 `JSONLHook` 实现 `hook.AsyncHook`。
- 在 `setup.Init` 中构造并注册一个进程级 `hook.Manager`；把 `JSONLHook` 挂到 Manager；把 Manager 通过 `taskagent.WithHookManager` 注入到每一个子 Agent 工厂。
- 新增 `configs.TraceConfig`（`enabled`、`dir`、`max_file_bytes`）+ 环境变量 `VV_TRACE_ENABLED`、`VV_TRACE_DIR`。
- Start/Stop 接入到 vv 的启动/关闭链（与现有 `SetupResult` 类似的生命周期）。
- project-hash 的工具函数 + `<cwd>` fallback 到 `default`。
- 单元测试：AsyncHook 行为（buffer、drop、序列化、文件切分）、路径解析、enabled 开关路径。

## 范围外

- OTEL / Langfuse 导出 —— P3-9。
- 结构化训练集（脱敏、采样、schema 定型）—— P3-5。
- 会话 resume —— P2-14（**会消费 P1-5 的输出**，但恢复逻辑在 P2-14）。
- 可视化前端 —— 社区/本地 `claude-JSONL-browser` 风格工具可复用 JSONL，无需 vv 内置。
- SQLite 存储 —— P1-6，P1-5 只做 JSONL。
- 跨进程合并同一 session —— 暂不支持（P1-6 SQLite 引入后由数据库侧天然合并）。

## 受影响角色 / 模块 / 进程

- **角色**：Developer（新功能的首批使用者）、DevOps（运维时可能需要关注磁盘占用）、External System（未来通过 JSONL 导入训练 pipeline）。
- **模型**：新增"Trace Event"（参考 `schema.Event`）、"Trace File"。
- **进程**：应用启动（Init 时构造 Manager + hook）、CLI/HTTP/MCP 消息处理（事件发射）、优雅关停（Stop + flush）。
- **应用**：核心运行时（CLI / HTTP / MCP 三模式）。

## 与既有 PRD 的冲突 / 待后续统一项

- `overview.md` 既有文本明确"Does Not Cover: Historical cost tracking across sessions" —— 本需求的 JSONL 里会自然带 LLMCallEnd 的 token 字段，P2-2"跨会话成本日志"未来可以直接消费 JSONL 得到，二者不矛盾但 PRD 应在 P2-2 里加引用。
- `architecture.md` 目前未列 Hook/Trace 模块；documenter 阶段需要补上。
- `setup.go:516` 注释称"vv does not currently install a hook.Manager" —— 本需求会改变这一事实，相关注释应同步更新。
