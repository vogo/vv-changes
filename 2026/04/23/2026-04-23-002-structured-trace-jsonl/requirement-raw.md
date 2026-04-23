# Requirement Raw — P1-5 结构化对话轨迹落盘（JSONL）

## 来源

`doc/prd/feature-todo.md` 第一个未实现的需求。

## 优先级表原文

| 优先级 | 需求名称 | 所属类别 | 依赖项 | 难度 | 可复用的现有能力 | 排序理由 |
|--------|---------|---------|--------|------|-----------------|---------|
| P1-5 | 结构化对话轨迹落盘（JSONL） | 记忆 + 飞轮底座 | 无 | 中 | `vage/hook/` 事件流、`schema.Event` | 自学习 & 训练飞轮共同依赖的数据出口；通过 AsyncHook 订阅写入 JSONL；是后续 P2/P3 的原料 |

## 用户需求说明（原文）

1. 读取 `doc/prd/feature-todo.md`，获取第一个未实现的需求（即 P1-5）。
2. 基于这个需求做详细的需求调研：
   - 了解类似产品，做这个需求都有哪些实现？
   - 注意些什么问题？
3. 调研相关行业，这些功能的实现都有什么具体的方案？
4. 在设计方案的时候做相关的参考。
5. 实现完了以后，将这个功能从 `feature-todo.md` 移除，放到 `feature-implement.md` 中。

## 里程碑

M4 · 记忆搜索引擎（M4 交付物之一）。

## 依赖关系（图摘录）

```
数据基础设施
  P1-5 轨迹落盘 ──► P1-6 SQLite+WAL ──► P2-3 FTS5+session_search
                                    └──► P2-2 历史成本日志 ──► P2-5 成本预警
                                    └──► P2-7 双画像快照
                                    └──► P2-8 摘要召回链（依赖 P2-3）
                                    └──► P2-14 会话检查点 ──► P4-13 对话分支
                                    └──► P3-10 向量嵌入召回
                  └──► P3-5 训练集导出 ──► P3-6 奖励信号 ──► P3-7 灰度发布
                  └──► P4-12 会话录制回放（依赖 P3-9 OTEL）
```

P1-5 是下游 P1-6、P3-5、P4-12 等多个需求共同依赖的上游数据出口。

---

## 行业调研（2026-04-23）

### 1. Claude Code — 最直接对标产品

- **布局**：`~/.claude/projects/<encoded-project-path>/<session-id>.jsonl`，一会话一文件。
- **字段**：每行一条事件，包括 `parentUuid`、`sessionId`、`version`、`gitBranch`、`cwd`、`message{role, content[]}`、`usage{input_tokens, output_tokens, cache_*}`、`uuid`、`timestamp`（ISO-8601）、`toolUseResult`。
- **使用场景**：`claude --resume`、第三方浏览器（`claude-JSONL-browser`、`clog`、`claude-code-log`）读回渲染 HTML。
- **已知痛点**（`anthropics/claude-code#50014`）：明文密钥、无自动轮转、无自动清理；社区提案建议默认 30 天轮转 + 密钥清洗。

**借鉴结论**：一会话一 JSONL 文件、project-scoped 目录 + append-only + 明确的字段 schema，是可靠、易消费的工程模式。轮转/密钥处理需要在 MVP 就设计退路。

### 2. OpenTelemetry GenAI Semantic Conventions（2026-Q1 实验阶段）

- **状态**：2026-03 OpenTelemetry 已标准化 GenAI span/event 命名（如 `invoke_agent <agent_name>`、`chat <model>`、`execute_tool <tool_name>`），由 OpenLLMetry 提案发起，Datadog 在 OTel v1.37 开始原生支持。
- **双发**：`OTEL_SEMCONV_STABILITY_OPT_IN` 允许旧属性+新属性同时发射，过渡期常用。
- **典型属性**：`gen_ai.system`、`gen_ai.request.model`、`gen_ai.response.model`、`gen_ai.usage.input_tokens`、`gen_ai.usage.output_tokens`、`gen_ai.operation.name`、`gen_ai.agent.name`、`gen_ai.tool.name`、`gen_ai.tool.call.id`。

**借鉴结论**：P1-5 本次是本地 JSONL 落盘不是 OTEL exporter（OTEL 是 P3-9 单独立项）。但在 JSONL schema 命名时应尽量"向 GenAI semconv 对齐"（event type + 关键属性），后续 P3-9 写 OTEL 适配器时可以直接映射、不必返工。

### 3. Langfuse / OpenAI Agents SDK / OpenInference

- Langfuse 通过 OpenInference OpenAI Agents 自动仪器化，导出 OTel spans → Langfuse 后端。
- JSON Schema 对接：Structured Output 产生的 schema 会被 trace 完整记录（request/response pair）。
- **训练集导出**：Langfuse 有 Datasets 功能（把 trace 子集导出成 SFT/eval），字段是 `{input, output, expected_output, metadata, tags}`。

**借鉴结论**：P1-5 落盘的字段必须足够丰富，才能在 P3-5"训练集导出 Pipeline"阶段过滤出 SFT/DPO 可用数据。最低需要保留：user-input → final-output 链、tool-call 链、token usage、模型名、耗时、session/agent ID。

### 4. Go + slog / logfusc / masq / CockroachDB redact

- Go 生态里用 `slog`+`masq` 或 `logfusc` 可以声明式把 PII 字段打标、输出时替换占位符。
- 高吞吐场景建议异步 + buffer。
- 注意点：**redaction 必须在写盘前完成**；**部分实体跨 chunk** 时要 buffer 2-3 chunk 避免半截泄露。

**借鉴结论**：vv 侧已经有 `vage/mcp credscrub` 与 `guard.ToolResultInjectionGuard` 在请求/工具 I/O 边界先做一次清洗。P1-5 的 JSONL 侧做"**不引入新的 redaction 算法**，但为 P3-5 预留一个可插拔的 `RedactFunc`"是合适的折中。

### 5. 通用注意事项（来自行业经验）

- **大字段膨胀**：LLM 响应/工具结果容易上万字节，文件增长快 → 需要 per-line cap + rotation。
- **写阻塞不能拖 Agent 回路**：必须 AsyncHook，channel 满就 drop + warn（已是 vage Manager 的既有行为）。
- **崩溃安全**：append-only + 每行独立 JSON；一行损坏不影响后续行；Stop 时 flush。
- **文件句柄**：长会话可能占住句柄数小时，OS 上不是问题；但要在 session 结束/关闭 hook 时显式 Close。
- **权限**：目录 0o700、文件 0o600（沿用 vv/debugs 惯例）。
- **磁盘压力**：默认关闭或提供明显 on/off 开关；提供 max_file_bytes / max_age_days 旋钮。

---

## 需求关键决策（Q&A — 基于现有 vv 约定与行业实践的默认取值）

> 这些是需求里用户没说、但实现前必须先定下来的点。下面给出多个可选方案，各选一个默认（标注理由），后续设计阶段如用户有不同意见可在 Complexity Assessment 前回滚。

### Q1. 落盘目录布局？

- (a) `~/.vv/traces/<session-id>.jsonl`（扁平）
- (b) `~/.vv/traces/<project-hash>/<session-id>.jsonl`（项目分桶，类 Claude Code）
- (c) `<cwd>/.vv/traces/<session-id>.jsonl`（随项目走）
- **默认选 (b)**：和 Claude Code 的用户心智一致，便于同一项目内跨 session 汇总；hash 用 `cfg.Tools.BashWorkingDir` 的 SHA-256 前 12 字节的 base32 形式，避免路径里奇怪字符。目录可通过 `trace.dir` 完全覆盖。

### Q2. 文件切分粒度？

- (a) 一会话一文件
- (b) 一天一文件
- (c) 一进程一文件
- **默认选 (a)**：语义清晰，最接近"对话轨迹"的自然边界；后续 P2-14 会话检查点/恢复正好可直接复用文件名索引。单 session 超长时提供 `max_file_bytes`（默认 64 MiB）触发 `.1/.2` 分片。

### Q3. 事件范围？

- (a) 全量（所有 `schema.Event*` 常量）
- (b) 仅核心（AgentStart/End + ToolCall/Result + LLMCallEnd + Error + 用户输入/最终输出）
- (c) 可配置白名单
- **默认选 (a)**：P1-5 定位"飞轮底座"，下游 P3-5/P4-12 的消费者宁要多不要少。AsyncHook 的 Filter 参数保留"empty = all"语义，后续可按 `trace.event_types` 收紧。

### Q4. 是否默认开启？

- (a) 默认 on（disk 积累）
- (b) 默认 off（opt-in）
- **默认选 (b)**：和 `debug`、`eval`、`mcp.server` 一致的"需要就开"姿态；隐私+磁盘中性；`trace.enabled: true` 打开。

### Q5. Redaction 策略？

- (a) 复用 credscrub 再跑一次
- (b) 依赖既有 guard（MCP credscrub + tool-result 注入扫描）已在上游清洗
- (c) 预留 `RedactFunc` 钩子但 MVP 不实现
- **默认选 (c)**：上游已有护栏，再扫一次 CPU 浪费；但为 P3-5"脱敏训练集导出"预留可插拔点不返工。MVP 文档里明确写"落盘内容未做额外脱敏，使用者需承担"。

### Q6. 轮转/保留？

- (a) 不做，纯 append
- (b) 按 size 分片（非 cleanup）
- (c) 按天数 cleanup
- **默认选 (a)+(b) 的最小组合**：MVP 支持 `max_file_bytes` 分片（避免单文件无上限），不做自动 cleanup（留给 P1-6 SQLite 做结构化存储/查询时再统一处理）。文档明确"旧文件需用户手动清理"。

### Q7. 模式覆盖？

- (a) 仅 CLI
- (b) CLI + HTTP
- (c) CLI + HTTP + MCP
- **默认选 (c)**：AsyncHook 装在 `hook.Manager` 上，三种模式都经过同一个 TaskAgent ReAct loop 发射事件，一次接入三处受益。成本几乎为 0。

### Q8. 何时 Start/Stop？

- (a) 在 `setup.Init` 启动、在进程退出时 Stop
- (b) 每个 session 单独启停
- **默认选 (a)**：与 vage `hook.Manager.Start/Stop` 语义一致（进程级），文件 handle 按 session_id 懒打开、首次写入时创建、进程退出时统一 flush+close。

---

## 附：关键成功指标（用于 designer / tester 阶段判定）

1. 启用 `trace.enabled: true` 后，CLI 跑一次 prompt，能在 `~/.vv/traces/<project-hash>/<session-id>.jsonl` 看到 ≥ 1 行 `agent_start` + N 行中间事件 + 1 行 `agent_end`。
2. JSONL 每行都是合法 JSON，能用 `jq -c .` 通读；字段包含 `type`、`timestamp`、`session_id`、`agent_id`、`data`。
3. channel 满（合成高吞吐测试）时会 drop 并 slog.Warn，不会阻塞 Agent。
4. 进程 SIGINT 退出时 Stop 调用确实 flush 到盘（写完未丢 buffer）。
5. `trace.enabled: false`（默认）时，不开文件、不启 goroutine、对现有性能零影响。
6. LLM/工具字段足以在后续 P3-5 导出 SFT 数据时构建 "{user_prompt, tool_trace, final_response, usage}" 记录。
