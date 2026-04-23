# 结构化对话轨迹落盘（JSONL）— P1-5 Design (refined)

> This is the refined design after improver review. The original is preserved
> as `design-raw.md`. See `design-review.md` for the decision trail.

## 总体思路

复用 vage 已有的事件基础设施（`hook.Manager` + `schema.Event` +
`taskagent.WithHookManager`），在 vv 侧装配：

1. 在 `setup.Init` 构造**进程级** `*hook.Manager`。
2. 若 `trace.enabled` 开启，构造 `tracelog.JSONLHook`（`hook.AsyncHook`），
   `RegisterAsync` 到 Manager。
3. Manager 通过 `registries.FactoryOptions.HookManager` 注入到每一个 Agent
   工厂（coder / researcher / reviewer / chat / explorer / planner），所有
   `taskagent.New(..., WithHookManager(m))`。
4. `setup.Init` 自己 **Start** Hook Manager（装配的最后一步），并返回一个
   `Shutdown func(ctx)` 闭包；`main.go` 在三种模式下统一 `defer initResult.Shutdown(ctx)`。
5. Hook 自己维护按 `session_id` 懒打开的文件句柄，消费由 Manager 拥有的
   channel，每条事件一行 JSON。

**不改 vage，不改 aimodel。** 全部改动落在 `vv/`。

## 模块清单

| 新增 / 修改 | 位置 | 职责 |
|------------|------|------|
| 新增包 | `vv/traces/tracelog/` | JSONL 落盘实现 |
| 新增配置 | `vv/configs/config.go`：`TraceConfig` | `enabled` / `dir` / `max_file_bytes` / `buffer_size` |
| 新增字段 | `vv/registries/registry.go`：`FactoryOptions.HookManager` | 工厂注入 |
| 修改 | `vv/agents/*.go`（6 个工厂） | 把 HookManager 透传到 `taskagent.New` |
| 修改 | `vv/setup/setup.go` | 装配 Manager + Trace Hook；`Start` Hook；`InitResult.HookManager` + `InitResult.Shutdown`；更新 L516 旧注释 |
| 修改 | `vv/main.go` | `defer initResult.Shutdown(ctx)` —— 三模式统一 |
| 修改 | `vv/configs/config.go` 环境变量段 | `VV_TRACE_ENABLED` / `VV_TRACE_DIR` |

## 关键数据结构

### `TraceConfig`（vv/configs/config.go）

```go
// TraceConfig controls structured conversation trace logging (JSONL).
// When Enabled is nil or false (default), no hook manager is installed on
// the LLM call path and there is zero measurable runtime cost.
type TraceConfig struct {
    Enabled      *bool  `yaml:"enabled,omitempty"`        // default false
    Dir          string `yaml:"dir,omitempty"`             // default ~/.vv/traces
    MaxFileBytes int64  `yaml:"max_file_bytes,omitempty"`  // default 64 MiB; 0 = disable rotation (single file per session)
    BufferSize   int    `yaml:"buffer_size,omitempty"`     // default 1024; AsyncHook channel capacity
}

func (t TraceConfig) IsEnabled() bool {
    return t.Enabled != nil && *t.Enabled
}
```

- 环境变量：`VV_TRACE_ENABLED`（`true`/`false`）、`VV_TRACE_DIR`（绝对路径）。
- 未设 `Dir` 时默认 `filepath.Join(os.UserHomeDir(), ".vv", "traces")`；home
  不可得时 `./.vv/traces`。
- 负值 / 异常值被回退为默认（`MaxFileBytes < 0` → 64 MiB，`BufferSize <= 0`
  → 1024）。**只有** `MaxFileBytes == 0` 被解释为"不切分"。

### `JSONLHook`（vv/traces/tracelog/tracelog.go）

```go
type JSONLHook struct {
    baseDir      string // absolute path; guaranteed to exist at Start
    projectHash  string // derived from cwd; stable across restarts
    maxFileBytes int64
    ch           chan schema.Event // Manager 写入端；hook 消费端

    files map[string]*sessionFile // consumer goroutine 独占访问，无锁
    wg    sync.WaitGroup
}

type sessionFile struct {
    f       *os.File
    written int64
    part    int // 0 = <sid>.jsonl, 1 = <sid>.1.jsonl, …
}
```

`hook.AsyncHook` 接口实现：
- `EventChan() chan<- schema.Event` → 返回 `h.ch`（单个共享 channel）。
- `Filter() []string` → 返回 `nil`（全量订阅）。
- `Start(ctx)` → 创建目录、启动 consumer goroutine；**不** 创建 channel
  （channel 由 `New` 构造时创建，由 Manager 的生命周期通过 `range` 终止来驱动）。
- `Stop(ctx)` → 等待 `wg` 让 consumer goroutine 退出后，逐文件 `Sync+Close`。
  **不** `close(h.ch)` —— 见下文"Channel 所有权"。

### Channel 所有权与关停顺序 (refined A3)

明确的所有权契约：
- **`hook.Manager` 拥有 dispatch 生命周期**。`Manager.Stop()` 保证停止向
  hook 派发事件。
- **`JSONLHook` 拥有 channel** —— 由 `New` 创建，由 `Stop` 在确认
  Manager 不再派发之后 `close`。

关停顺序：
1. `setup.Shutdown(ctx)` 调用 `hook.Manager.Stop(ctx)`。Manager 内部逻辑
   停止派发并调用各 hook 的 `Stop`。
2. `JSONLHook.Stop(ctx)` 被 Manager 调用：
   a. `close(h.ch)`（Manager 已不再 send，安全）；
   b. `wg.Wait()` 等待 consumer goroutine 退出（consumer `for ev := range
      h.ch` 正常结束）；
   c. 对 `files` map 里每个 `sessionFile`：`Sync()` + `Close()`。

若实现期间 vage 侧 `hook.Manager` 生命周期模型与上述不一致，developer 阶段需
在实现前回到 `vage/hook` 核对，并把差异反馈到此文档；不得让两侧同时 `close`
同一个 channel。

### 消费 goroutine 逻辑

```go
for ev := range h.ch {
    sid := sanitizeSessionID(ev.SessionID) // "" → "default"
    line, err := json.Marshal(ev)
    if err != nil { slog.Warn("trace: marshal", "err", err); continue }
    line = append(line, '\n')

    sf, err := h.ensureFile(sid)
    if err != nil { slog.Warn("trace: open", "sid", sid, "err", err); continue }

    // Size-based rotation: single atomic step.
    if h.maxFileBytes > 0 && sf.written+int64(len(line)) > h.maxFileBytes {
        if err := h.rollSession(sid); err != nil {
            slog.Warn("trace: rotate", "sid", sid, "err", err); continue
        }
        sf, _ = h.ensureFile(sid) // must succeed; rollSession opened the new file
    }

    n, werr := sf.f.Write(line)
    if werr != nil { slog.Warn("trace: write", "sid", sid, "err", werr); continue }
    sf.written += int64(n)
}
```

- `ensureFile(sid)`：首次访问时打开 `<baseDir>/<projectHash>/<sid>.jsonl`，
  `O_CREATE|O_WRONLY|O_APPEND`，权限 0o600；目录 0o700。
- `rollSession(sid)`（single step, A1）：`Sync + Close` 旧文件，从 `files`
  map 删除，`part++`，打开 `<sid>.<part>.jsonl`，回写 `files[sid]`，
  `written = 0`。下一次 `ensureFile` 直接命中新文件。
- 所有错误只写 `slog.Warn`，**绝不 panic**，**绝不 return err**（AsyncHook
  consumer 返回即丢事件）。

### Project Hash（vv/traces/tracelog/projecthash.go）

```go
// ProjectHash returns a stable, filesystem-safe token derived from the
// absolute working directory. Empty path yields "default".
func ProjectHash(workingDir string) string {
    if workingDir == "" { return "default" }
    abs, err := filepath.Abs(workingDir)
    if err != nil { abs = workingDir }
    sum := sha256.Sum256([]byte(abs))
    // first 12 bytes → base32 (lowercase, no padding) ≈ 20 chars
    enc := strings.ToLower(base32.StdEncoding.WithPadding(base32.NoPadding).EncodeToString(sum[:12]))
    return enc
}
```

- 输入稳定 → 输出稳定（幂等，可用于重启后聚合同一项目的 trace）。
- base32 无大写/padding，文件系统友好；长度可预测。
- 12 字节（96 bit）抗碰撞对于单用户历史上可能打开的 ~10^3 项目数量级绰绰有
  余；简单粗暴，不值得为更高强度付出字符数代价。

### Session ID 消毒

```go
var sidPattern = regexp.MustCompile(`[^a-zA-Z0-9._-]`)
func sanitizeSessionID(sid string) string {
    if sid == "" { return "default" }
    s := sidPattern.ReplaceAllString(sid, "_")
    if len(s) > 128 { s = s[:128] } // conservative cap; most FS allow 255
    return s
}
```

## 与 `hook.Manager` 装配

```go
// setup.go (new)
func buildHookManager(cfg *configs.Config) (*hook.Manager, error) {
    if !cfg.Trace.IsEnabled() { return nil, nil }
    mgr := hook.NewManager()
    h, err := tracelog.New(cfg.Trace, cfg.Tools.BashWorkingDir)
    if err != nil { return nil, err }
    mgr.RegisterAsync(h)
    return mgr, nil
}
```

- `nil` Manager 是合法且安全的 —— `Dispatch` 的第一行就是 `if m == nil { return }`
  （见 `vage/hook/manager.go:65`）。
- 下游 `taskagent.WithHookManager(nil)` 合法（`a.hookManager.Dispatch` 同样
  nil-safe）。

## FactoryOptions 扩展

```go
// vv/registries/registry.go
type FactoryOptions struct {
    // ... 既有字段不动
    HookManager *hook.Manager // nil 表示不安装 hook；默认不打开
}
```

每个 `vv/agents/*.go` 工厂在 `taskOpts` 组装处新增：

```go
if opts.HookManager != nil {
    taskOpts = append(taskOpts, taskagent.WithHookManager(opts.HookManager))
}
```

同样加到 `setup.go` 中 `planGen`、`explorer`、`planner` 三处直接
`taskagent.New(...)` 的调用（它们目前不走 factory）。

## Lifecycle（setup.Init 封装，main.go 极简）(refined A5/A6)

### `setup.Init` 内部

```go
mgr, err := buildHookManager(cfg)
if err != nil { return nil, err } // atomic: no partially wired state
if mgr != nil {
    if err := mgr.Start(ctx); err != nil {
        return nil, fmt.Errorf("start trace hook: %w", err)
    }
}
// ... build agents, injecting mgr via FactoryOptions.HookManager ...

result := &InitResult{
    // ... existing fields
    HookManager: mgr, // may be nil
    Shutdown: func(sctx context.Context) {
        if mgr == nil { return }
        if err := mgr.Stop(sctx); err != nil {
            slog.Warn("vv: stop trace hook", "error", err)
        }
    },
}
```

### `main.go` —— 三模式统一

```go
initResult, err := setup.Init(ctx, cfg, setupOpts)
if err != nil { /* existing error path */ }
defer func() {
    sctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    initResult.Shutdown(sctx)
}()
```

- 三种模式（CLI / HTTP / MCP）复用同一段 —— 它们都从 `setup.Init` 出发。
- `Shutdown` 使用独立 context；主 ctx 此时可能已取消。
- `main.go` 不再感知 `HookManager` 的存在，仅调用 `Shutdown` 闭包。

（若 `SetupResult` 已有 `Shutdown`/`Close` 约定，把 trace shutdown 合并进去；
否则新增。）

## JSONL Line 格式

直接 `json.Marshal(schema.Event)`（Event 已经是 JSON-tagged struct with
`EventData` sealed interface）。时间戳格式以 `schema.Event` 自带的
marshaling 为准，不在 hook 侧重新格式化。示例（示意，真实字段来自 schema）：

```json
{"type":"agent_start","agent_id":"coder","session_id":"s-abc","timestamp":"2026-04-23T10:15:00.123Z","data":{}}
{"type":"iteration_start","agent_id":"coder","session_id":"s-abc","timestamp":"2026-04-23T10:15:00.125Z","data":{"iteration":1}}
{"type":"tool_call_start","agent_id":"coder","session_id":"s-abc","timestamp":"2026-04-23T10:15:00.200Z","data":{"tool_call_id":"call_1","tool_name":"read","arguments":"{\"path\":\"main.go\"}"}}
{"type":"tool_result","agent_id":"coder","session_id":"s-abc","timestamp":"2026-04-23T10:15:00.210Z","data":{"tool_call_id":"call_1","tool_name":"read","result":{"content":"...","is_error":false}}}
{"type":"llm_call_end","agent_id":"coder","session_id":"s-abc","timestamp":"2026-04-23T10:15:00.900Z","data":{"model":"gpt-5","duration_ms":700,"prompt_tokens":1200,"completion_tokens":80,"total_tokens":1280,"stream":false}}
{"type":"agent_end","agent_id":"coder","session_id":"s-abc","timestamp":"2026-04-23T10:15:00.950Z","data":{"duration_ms":827,"message":"...","stop_reason":"end_turn"}}
```

无需 wrapper：保留原生 `schema.Event` 的 JSON 序列化，下游消费者（P2-14
resume / P3-5 SFT 导出 / P4-12 replay）直接 `json.Unmarshal` 即可还原。

## 并发 / 顺序性

- AsyncHook 由 `hook.Manager` 单线程发送到 channel（非阻塞 `select default`）。
- 单 consumer goroutine 读 channel → 写盘；**单 session 内事件顺序严格保留**。
- 跨 session 的事件**不保证交错顺序**（channel 层已乱序），但每个 session
  自己的文件行序稳定。
- 文件句柄 map（`files`）由 consumer goroutine 独占访问 → 无需锁。

## 失败模式清单

| 场景 | 行为 |
|------|------|
| 目录创建失败（`Start`） | `Start` 返回 error；`setup.Init` 回滚并返回 error；main 进程退出 |
| 写文件失败（consumer） | `slog.Warn` + 跳过该事件；下一条继续 |
| Marshal 失败（consumer） | 同上 |
| channel 满 | `hook.Manager.Dispatch` 已实现 drop + slog.Warn |
| Stop 时 channel 有积压 | Manager 先停止派发，再调用 hook.Stop → `close(ch)`；consumer drain 完毕后 flush + close 文件 |
| SIGKILL | 已 flush 的行完整；最后一行可能半条，但 append-only 不影响前面行 |

## 测试计划

单元测试（`tracelog_test.go`）：

1. **Happy path**：`New` → `Start` → `EventChan() <- event` × 5 → `Stop` → 读取文件断言：
   - 5 行合法 JSON
   - 字段 `type`/`session_id`/`timestamp`/`data` 存在
   - 每行 `json.Decode` 成功
2. **Session 路由**：两个不同 `SessionID` 的事件 → 生成 2 个独立文件。
3. **Session 消毒**：含 `../` 或特殊字符的 SessionID 被替换为 `_`；空
   SessionID 落到 `default.jsonl`。
4. **文件切分**：`MaxFileBytes=200` + 发送若干事件 → 产生 `<sid>.jsonl` 和
   `<sid>.1.jsonl`；每个文件首尾行仍是合法 JSON。
5. **`MaxFileBytes=0`**：发送足够多事件，不切分，单文件累积。
6. **停止幂等**：两次 `Stop` 不 panic。
7. **[MUST PASS] 禁用路径零成本 (US-2, refined A7)**：
   - `cfg.Trace.Enabled=false` 经 `setup.Init` 后：
     - `runtime.NumGoroutine()` 在 Init 前后的 delta 不包含新增 trace consumer。
     - `InitResult.HookManager == nil`。
     - `traces/` 目录不存在（或至少无新文件）。
   这个测试是 US-2 的核心守门，必须具体断言而非"感觉上没开"。
8. **`ProjectHash` 稳定性**：同一 cwd 重复调用结果相同；不同 cwd 不相同；空
   cwd 返回 "default"。

集成测试（`integrations/traces_tests/tracelog_tests/`，与既有
`integrations/traces_tests/budget_tests/` 布局并列 —— refined A8）：

9. **端到端**：构造一个 `MockChatCompleter` → 通过 setup 创建 coder agent →
   跑一个短 prompt → 验证 JSONL 文件里有完整链（agent_start … tool_call …
   llm_call_end … agent_end）。

## 验收标准映射

| Requirement US | 设计覆盖 |
|----------------|----------|
| US-1（启用后有 JSONL 文件） | `tracelog.JSONLHook.Start` + `ensureFile` + 消费循环 |
| US-2（禁用零成本） | `cfg.Trace.IsEnabled()` 快路径；Manager/Hook 不构造；测试 #7 硬断言 |
| US-3（不阻塞 Agent） | AsyncHook channel + `hook.Manager` 既有 non-blocking send |
| US-4（崩溃安全） | append-only + 行级 write + `Stop` flush + Sync/Close |
| US-5（SFT 四元组重建） | 全量事件订阅；保留 schema.Event 原生字段 |
| US-6（三模式覆盖） | setup.Init 公共路径装配 + setup.Init 内 Start + `InitResult.Shutdown` 统一 |

## 规模与风险

- 代码量估算：`tracelog.go` ~180 LOC、`projecthash.go` ~25 LOC、
  `tracelog_test.go` ~220 LOC（新增零成本测试）、`configs.go` 增量 ~30 LOC、
  工厂/setup 改动共 ~60 LOC（多了 Shutdown 闭包）、`main.go` 增量 ~8 LOC
  （改为仅 defer）；总 ~500 LOC。
- 主要风险：`taskagent` 里 hookManager 字段是否真的 nil-safe → 已确认
  （`a.hookManager.Dispatch` 里先判 `m == nil`）。
- 次要风险：`hook.Manager.Stop` 与 `JSONLHook.Stop` 的 channel 关闭顺序 →
  由本设计显式声明所有权：hook 拥有 channel，hook 在 Stop 中 close；
  developer 阶段在动工前核对 vage 侧是否一致，如不一致，调整 hook 实现（永
  远不双方 close 同一 channel）。
- 次要风险：`hook.Manager.Start` 失败时其他初始化资源泄漏 → `setup.Init`
  内 atomic 构造（失败直接 return，已装配部分由上层 main error 处理，与现
  有失败路径一致）。

## 不做的事（明确 out）

- 不做异步批量 buffer（单条 write 足够；消费者已是异步）。
- 不做 gzip / 压缩（交给下游 logrotate 或 P1-6 SQLite）。
- 不做 redaction / PII scrub；**不** 预留 `RedactFunc` 字段（refined A4）
  —— 上游 guard 已处理，P3-5 时再按需设计 redaction 管道。
- 不做 schema version 字段（`schema.Event.Type` + 封闭 `EventData` 已足够
  演进）。
- 不做 `trace.event_types` 过滤配置（消费侧 `jq` 足以；P3-5 若需要再设计）。
- 不做 OTEL / Langfuse exporter（P3-9）。
- 不做 resume 逻辑（P2-14）。
- 不做跨进程 session 合并（P1-6 SQLite 解决）。
- 不动 `vv/debugs/`（那是 `--debug` 人工调试专用，内容/时机/格式都不同）。
- 不导出文件路径给 `InitResult`（只有 `HookManager` 和 `Shutdown` 两个新字
  段；文件路径是 hook 内部实现）。
