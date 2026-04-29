# 设计:vv 端 Session wiring

## 0. 业界实践参考(简明版)

| 平台 | 续接形态 | 启示 |
|---|---|---|
| OpenAI Threads | `thread_id` 持久化,`Run` 自动复用,服务端管 truncation | id 是入参、消息存储不靠 client 重塑 → vv 也走"id-only 续接" |
| Anthropic Claude Code | `claude --resume <id>` / `claude --continue` | CLI flag 命名直接采用 `--session <id>` / `--session list` / `--session new` |
| LangGraph Checkpointer | `thread_id` 路由到 SQLite/Postgres 后端,从 checkpoint 恢复 state | vv 暂只做 events+meta,不做 checkpoint — out-of-scope 已声明 |
| Cursor / Aider | 持久会话目录,展示最近 N 条 | `vv --session list` 借鉴这一形态 |

**结论**:vv 端 wiring 走 OpenAI/Anthropic 同款"thread/session id 持久化 + CLI flag 续接"的最朴素路线;不做 LangGraph 那种从 checkpoint 重放 state 的能力,本期只做"id 复用 + meta 展示 + events 落盘 + HTTP 暴露",符合 design 文档 §4.1 的 P1 加固定位。

---

## 1. 改动总览(三条主线)

```
1) configs.SessionConfig (新增)
       ↓ 默认开启 → 影响 setup.Init 的依赖注入
2) setup.Init / buildHookManager (重构)
       ├── 始终在"trace 或 session 任一启用"时构造 hook.Manager
       ├── 构造 FileSessionStore
       ├── 注册 SessionHook 到 Manager
       └── 通过 InitResult.SessionStore 暴露给上层
3) 消费方接入
       ├── main.go 解析 --session flag
       ├── cli/ (cli.go + 新增 cli/session.go) Resume/List 逻辑
       └── httpapis/ (新增 sessions.go) 5 个端点
```

## 2. configs.SessionConfig

新文件不需要 — 加在 `vv/configs/config.go` 里贴着 `TraceConfig` 写。

```go
// SessionConfig controls the persistent session subsystem (vage/session
// integration). Default-on: a nil Enabled pointer means "enabled" so that a
// fresh install gets durable conversation history without configuration. Set
// `enabled: false` (YAML) or VV_SESSION_ENABLED=false to opt out.
type SessionConfig struct {
    Enabled *bool  `yaml:"enabled,omitempty"`           // default true
    Dir     string `yaml:"dir,omitempty"`               // default ~/.vv/sessions
    HistoryReplayMaxEvents int `yaml:"history_replay_max_events,omitempty"` // default 5000
}

func (s SessionConfig) IsEnabled() bool { return s.Enabled == nil || *s.Enabled }
func (s SessionConfig) EffectiveDir() string {
    if s.Dir != "" { return s.Dir }
    return filepath.Join(DefaultDir(), "sessions")
}
```

`Config` 结构里加 `Session SessionConfig \`yaml:"session,omitempty"\``。

`applyDefaults` 加:
```go
if cfg.Session.HistoryReplayMaxEvents == 0 {
    cfg.Session.HistoryReplayMaxEvents = 5000
}
```

env 解析:
```go
if v := os.Getenv("VV_SESSION_ENABLED"); v != "" {
    if b, err := strconv.ParseBool(v); err == nil {
        cfg.Session.Enabled = &b
    }
}
if v := os.Getenv("VV_SESSION_DIR"); v != "" { cfg.Session.Dir = v }
```

## 3. setup.Init 改造

### 3.1 重构 `buildHookManager` → `buildHookManagerWithSession`

**当前问题**:`buildHookManager` 写死 `cfg.Trace.IsEnabled()` 开关。新需求是"trace 或 session 启用 → 都需要 Manager"。

**目标签名**:
```go
// 返回 Manager(可能为 nil)、SessionStore(可能为 nil)、shutdown、error
func buildHookManagerAndSession(cfg *configs.Config) (
    *hook.Manager, session.SessionStore, func(context.Context), error,
)
```

逻辑:
1. 如果 trace 与 session 都关 → 返回 `(nil, nil, noop, nil)`(零开销路径,与今天一致)。
2. 否则 `mgr := hook.NewManager()`。
3. 若 trace 开:走原有 tracelog 构造 + `mgr.RegisterAsync(tracer)` 流程。
4. 若 session 开:`store, err := session.NewFileSessionStore(cfg.Session.EffectiveDir())`,失败回滚(已注册的 trace shutdown)→ `sessionHook := session.NewSessionHook(store)` → `mgr.RegisterAsync(sessionHook)`。
5. `mgr.Start(ctx)` 失败 → 同样回滚(关 store 句柄、关 trace 文件)。
6. shutdown 闭包:`mgr.Stop(ctx)` —— Manager.Stop 会逐个 Stop 已注册的 AsyncHook,SessionHook 自带 `sync.Once`,FileSessionStore 没有 close 句柄(纯文件),trace tracer 的 Stop 已被 Manager 接管。

`InitResult` 加字段:
```go
type InitResult struct {
    // ...原有字段
    SessionStore session.SessionStore // nil when cfg.Session.Enabled = false
}
```

### 3.2 setup.Init 装配顺序调整

`Init` 把原来的 `buildHookManager` 调用替换:
```go
hookManager, sessionStore, hookShutdown, err := buildHookManagerAndSession(cfg)
if err != nil { closeStore(); return nil, fmt.Errorf("setup hooks: %w", err) }

if hookManager != nil {
    if opts == nil { opts = &Options{} }
    opts.HookManager = hookManager
}
// ... result 构造
return &InitResult{
    // ...
    SessionStore: sessionStore,
    Shutdown:     chainShutdown(hookShutdown, closeStore),
}, nil
```

不引入新 Options 字段 —— `SessionStore` 不需要传给 setup.New(子 agents 不消费),只通过 `InitResult` 暴露给 main.go。

## 4. main.go:`--session` flag

新增 flag(放 `--debug` 后面,保持视觉聚类):

```go
sessionFlag := flag.String("session", "", "session id to resume (or 'list' to show recent sessions, 'new' to force a new session)")
```

紧跟 `setup.Init(...)` 之后、CLI 启动之前,新增分支:

```go
// Session subsystem decisions:
//   --session list   → print recent sessions and exit (works in CLI mode only)
//   --session <id>   → forwarded to cli.New() to drive resume; HTTP/MCP modes reject
//   --session new    → equivalent to omitting --session (force-new session)
//   --session ""     → omitted; default behavior unchanged
sessionArg := strings.TrimSpace(*sessionFlag)
switch {
case sessionArg == "":
    // default
case sessionArg == "list":
    if cfg.Mode != "cli" && cfg.Mode != "" {
        fmt.Fprintln(os.Stderr, "vv: --session list is only available in CLI mode")
        os.Exit(1)
    }
    if initResult.SessionStore == nil {
        fmt.Fprintln(os.Stderr, "vv: session subsystem is disabled; cannot list")
        os.Exit(1)
    }
    if err := cli.PrintSessionList(ctx, initResult.SessionStore, os.Stdout); err != nil {
        fmt.Fprintf(os.Stderr, "vv: %s\n", err)
        os.Exit(1)
    }
    shutdownInit(initResult)
    os.Exit(0)
case sessionArg == "new":
    sessionArg = "" // fall through; CLI will mint a new id
default:
    // explicit id; will be passed to cli.New
    if cfg.Mode != "cli" && cfg.Mode != "" {
        fmt.Fprintln(os.Stderr, "vv: --session <id> is only available in CLI mode (use HTTP /v1/sessions for inspection)")
        os.Exit(1)
    }
    if initResult.SessionStore == nil {
        fmt.Fprintln(os.Stderr, "vv: session subsystem is disabled; cannot resume")
        os.Exit(1)
    }
}
```

把 `sessionArg` 传到 `cli.New(...)` 通过新 functional option:`cli.WithSessionResume(initResult.SessionStore, sessionArg)`。

**与 -p 模式的关系**:`-p` 一发即走、不构造 TUI,原则上不需要持久 session id。本次保持一致 —— `-p` 不接收 `--session`(组合时 stderr 警告但不 fail),让 `-p` 仍然每次新建一个临时 id(由 SessionHook autoCreate 落盘);如果用户真的需要 -p 续接,后续迭代再讨论。

## 5. cli 改造

### 5.1 新文件 `vv/cli/session.go`

承载 3 个能力 —— 切出独立文件保住 `cli.go` 800 行红线。

```go
package cli

import (
    "context"
    "errors"
    "fmt"
    "io"
    "sort"
    "strings"
    "text/tabwriter"
    "time"

    "github.com/vogo/vage/session"
)

// SessionResumeMode is the result of cli.PrepareSessionID. Run() consults it
// to decide how to introduce the session at TUI startup.
type SessionResumeMode int

const (
    SessionResumeNew      SessionResumeMode = iota // new session, fresh id
    SessionResumeExisting                          // existing id, history found
    SessionResumeNotFound                          // explicit id, but absent — still bind, treat as fresh
)

// PrepareSessionID resolves the user-supplied --session value against the
// store. Returns the id to bind to the App, the resolved mode, and any
// loaded history events for display.
//
// Behavior matrix:
//   want == ""           → mint new id (mode=New).
//   want != "", exists   → return id (mode=Existing). Caller may render banner.
//   want != "", missing  → return id (mode=NotFound). The first SessionHook
//                          AppendEvent will Create the session via autoCreate.
//   want != "", invalid  → return error.
func PrepareSessionID(ctx context.Context, store session.SessionMetaStore, want string) (string, SessionResumeMode, *session.Session, error) {
    if want == "" {
        return session.GenerateID(), SessionResumeNew, nil, nil
    }
    if err := session.IDPattern.MatchString(want); !err { /* validation via store */ }
    s, err := store.Get(ctx, want)
    if errors.Is(err, session.ErrSessionNotFound) {
        return want, SessionResumeNotFound, nil, nil
    }
    if err != nil {
        return "", 0, nil, fmt.Errorf("resolve session %q: %w", want, err)
    }
    return s.ID, SessionResumeExisting, s, nil
}

// PrintSessionList writes a formatted table of the most recent sessions to w.
const sessionListLimit = 20

func PrintSessionList(ctx context.Context, store session.SessionStore, w io.Writer) error {
    sessions, err := store.List(ctx, session.SessionFilter{Limit: sessionListLimit})
    if err != nil { return fmt.Errorf("list sessions: %w", err) }
    sort.Slice(sessions, func(i, j int) bool {
        return sessions[i].UpdatedAt.After(sessions[j].UpdatedAt)
    })
    tw := tabwriter.NewWriter(w, 0, 0, 2, ' ', 0)
    fmt.Fprintln(tw, "ID\tAGENT\tTITLE\tSTATE\tEVENTS\tUPDATED")
    for _, s := range sessions {
        events, _ := store.ListEvents(ctx, s.ID) // best-effort count
        fmt.Fprintf(tw, "%s\t%s\t%s\t%s\t%d\t%s\n",
            s.ID, dashIfEmpty(s.AgentID), dashIfEmpty(s.Title),
            string(s.State), len(events),
            s.UpdatedAt.Local().Format("2006-01-02 15:04"))
    }
    return tw.Flush()
}

// TouchSession refreshes the UpdatedAt timestamp on the session metadata so
// resume-listings reflect activity even though SessionHook only writes
// events, not meta. Called by App.Run before the first user message.
func TouchSession(ctx context.Context, store session.SessionMetaStore, sessionID, agentID string) error {
    s, err := store.Get(ctx, sessionID)
    if errors.Is(err, session.ErrSessionNotFound) {
        return store.Create(ctx, &session.Session{
            ID: sessionID, AgentID: agentID, State: session.StateActive,
            CreatedAt: time.Now(),
        })
    }
    if err != nil { return err }
    if agentID != "" && s.AgentID == "" { s.AgentID = agentID }
    return store.Update(ctx, s)
}
```

(草案中的伪 `IDPattern.MatchString` 写法在实现阶段会替换为复用 `validateID` 等价路径 —— 实际 `session.New(want)` 的 panic 风险需要避免,因此实现里直接用 `store.Get`,把校验交给 store —— 任何不合法的 id 都会被 store 拒为 `ErrInvalidArgument`。)

### 5.2 cli/cli.go 接入

`App` 加字段:
```go
type App struct {
    // ...
    sessionStore session.SessionStore // optional
    requestedSessionID string         // empty for "new"
}
```

新 functional option:
```go
func WithSessionResume(store session.SessionStore, requestedID string) func(*App) {
    return func(a *App) {
        a.sessionStore = store
        a.requestedSessionID = requestedID
    }
}
```

`App.Run` 顶部替换:
```go
- // Generate session ID.
- b := make([]byte, 8); _, _ = rand.Read(b); a.sessionID = hex.EncodeToString(b)
+ if a.sessionStore != nil {
+     id, mode, prev, err := PrepareSessionID(ctx, a.sessionStore, a.requestedSessionID)
+     if err != nil { return err }
+     a.sessionID = id
+     // Touch meta so list shows fresh UpdatedAt; safe even when prev exists.
+     _ = TouchSession(ctx, a.sessionStore, id, "")
+     a.printResumeBanner(mode, prev) // no-op for SessionResumeNew
+ } else {
+     // Session subsystem disabled — keep the legacy random id so trace logs
+     // and any other consumers still see a stable per-process id.
+     b := make([]byte, 8); _, _ = rand.Read(b); a.sessionID = hex.EncodeToString(b)
+ }
```

`printResumeBanner` 通过 `m.printSystem` 在 TUI 上显示一行"resuming session <id> · <events> events · last update <updated_at>"。

**为什么不重放 transcript 进 history**:`schema.Event.Data` 是带未导出方法的接口,`session.FileStore.ListEvents` 把 `Data` 反序列化为 nil。要还原 user/assistant 文本必须要么改 schema(动 vage 包接口),要么单独维护一份"transcript jsonl"。前者超出本次 scope,后者属于"加文件等于加复杂度"——和 design 文档 §4.1 的"P1 加固"定位违背。**结论**:本次 resume = 复用 id + meta banner + events 文件留作 HTTP 审计与未来重放,**不重塑 history**。下一迭代(P8 checkpoint)再做"恢复对话状态"。

### 5.3 cli/cli.go 删除 `crypto/rand` import 的条件

如果 session 子系统始终在(默认开),`crypto/rand` 仍被 fallback 路径(disabled)使用,保留即可。

## 6. httpapis 改造

### 6.1 新文件 `vv/httpapis/sessions.go`

```go
package httpapis

import (
    "encoding/json"
    "errors"
    "net/http"
    "strconv"
    "strings"

    "github.com/vogo/vage/schema"
    "github.com/vogo/vage/session"
)

const (
    httpSessionListDefaultLimit  = 50
    httpSessionListMaxLimit      = 200
    httpEventListDefaultLimit    = 1000
    httpEventListMaxLimit        = 5000
)

// 5 endpoints; struct types listed for self-documentation.

type sessionMetaResponse struct {
    ID        string         `json:"id"`
    AgentID   string         `json:"agent_id,omitempty"`
    UserID    string         `json:"user_id,omitempty"`
    Title     string         `json:"title,omitempty"`
    State     string         `json:"state"`
    Metadata  map[string]any `json:"metadata,omitempty"`
    CreatedAt string         `json:"created_at"`
    UpdatedAt string         `json:"updated_at"`
}

type sessionListResponse struct {
    Sessions []sessionMetaResponse `json:"sessions"`
}

type sessionDetailResponse struct {
    sessionMetaResponse
    State_ map[string]any `json:"state,omitempty"` // session state KV
}

type eventListResponse struct {
    Events []schema.Event `json:"events"`
}

type sessionPatchRequest struct {
    Title    *string         `json:"title,omitempty"`
    State    *string         `json:"state,omitempty"`
    Metadata *map[string]any `json:"metadata,omitempty"`
}

func handleListSessions(store session.SessionStore) http.HandlerFunc { /* ... */ }
func handleGetSession(store session.SessionStore) http.HandlerFunc { /* ... */ }
func handleListEvents(store session.SessionStore) http.HandlerFunc { /* ... */ }
func handleDeleteSession(store session.SessionStore) http.HandlerFunc { /* ... */ }
func handlePatchSession(store session.SessionStore) http.HandlerFunc { /* ... */ }
```

每个 handler 都把 `session.ErrSessionNotFound` → 404、`session.ErrInvalidArgument` → 400、其它错误 → 500,沿用现有 `writeJSON` + `{"code", "message"}` 错误格式(见 `httpapis/http.go:146`)。

### 6.2 注入到 mux

在 `httpapis/http.go` 的 `Serve` 加可选参数。最小改动:把 `sessionStore` 加到 `Serve(...)` 的签名末尾,配合 main.go 调用点。

```go
func Serve(ctx context.Context, cfg *configs.Config, llm aimodel.ChatCompleter,
    dispatcher agent.Agent, agents []agent.Agent, persistentMem memory.Memory,
    interactionStore *InteractionStore, compactor *memory.ConversationCompactor,
    sessionBudget, dailyBudget *budgets.Tracker,
    sessionStore session.SessionStore, // <-- new last arg
) error {
    // ... existing setup ...

    if sessionStore != nil {
        mux.HandleFunc("GET /v1/sessions",         handleListSessions(sessionStore))
        mux.HandleFunc("GET /v1/sessions/{id}",    handleGetSession(sessionStore))
        mux.HandleFunc("GET /v1/sessions/{id}/events", handleListEvents(sessionStore))
        mux.HandleFunc("DELETE /v1/sessions/{id}", handleDeleteSession(sessionStore))
        mux.HandleFunc("PATCH /v1/sessions/{id}",  handlePatchSession(sessionStore))
    }
    // ...
}
```

`main.go` 调用点同步把 `initResult.SessionStore` 传进去。

### 6.3 PATCH 字段语义

- `title`:覆盖。
- `state`:必须是 `active`/`paused`/`completed`/`failed` 之一,否则 400。
- `metadata`:整体覆盖 map(语义 = 完全替换),与 `Update` 调用一致。这样语义最简单 —— 如果未来需要 patch-merge 再加 `?merge=true`。

## 7. 错误处理 / 关闭顺序

### 7.1 启动失败回滚

新版本 `buildHookManagerAndSession`:
- trace 构造成功后再起 session;session 失败 → 调 trace 的 stop。
- Manager.Start 失败 → 同样 stop 已注册的 hook。

### 7.2 进程退出

`InitResult.Shutdown` = `chainShutdown(hookShutdown, closeStore)`:
- `hookShutdown` 通过 `mgr.Stop(ctx)` 顺序关闭所有 AsyncHook(trace/session)。
- `closeStore` 关闭持久化记忆 store(SQLite 句柄)。

`hook.Manager.Stop` 是否串行 stop 多个 hook?不确定 —— 假设是。如果实现是并行,顺序无所谓,因为各 hook 内部独立。**实现阶段会通过 grep `hook.Manager` Stop 实现确认**;若不串行也没问题(每个 hook 独立资源)。

## 8. 测试方案

### 8.1 单元测试

| 文件 | 用例 |
|---|---|
| `vv/configs/config_test.go` | `TestSessionConfig_DefaultEnabled`、`TestSessionConfig_EnvOverride`、`TestSessionConfig_EffectiveDir` |
| `vv/cli/session_test.go` | `TestPrepareSessionID_New / Existing / NotFound / Invalid`、`TestTouchSession_CreateOnMissing / RefreshOnExisting`、`TestPrintSessionList_Sorted / Empty` |
| `vv/httpapis/sessions_test.go` | 每个 endpoint 的 `200 / 404 / 400 / 405`(未支持 method)+ List 的 filter 组合 |
| `vv/setup/setup_test.go` | `TestInit_SessionStoreCreated_ByDefault`、`TestInit_SessionDisabled_NoStore`、`TestInit_SessionAndTraceCoexist`(都开 → Manager 上有 2 个 hook) |

### 8.2 集成测试

`vv/integrations/session_tests/`(新目录):

| 用例 | 场景 |
|---|---|
| `cli_resume_tests/main_test.go` | 启动 vv -p,记录 session id;再用 `--session <id>` 启动,确认能 PrepareSessionID 拿到 Existing |
| `http_sessions_tests/main_test.go` | spin up HTTP server,POST 一个 RunRequest 让 SessionHook 落盘,然后 GET /v1/sessions 看到、GET /v1/sessions/{id}/events 拿到事件、DELETE 后再 GET 404 |

约定:与现有 `vv/integrations/cli_tests/permission_tests/` 同款骨架,要求 `VV_LLM_API_KEY` 环境变量;mocks 时用 `aimodel/mock`。

## 9. 文档更新计划(documenter 阶段)

1. `doc/design/session-context-solution.md` §4.1 把"vv 端 wiring 留作下一迭代"改为"已落地(2026-04-29-004)";§8 表格"Session 实体"行补完成度,新增"vv Session wiring"行打 ✅。
2. `vv/CLAUDE.md` 增加"## Session subsystem"段:开关、目录布局、CLI flag、HTTP 端点速查、与 Trace 的关系。
3. `vv/.doc/`(若存在;不存在则跳过)。

## 10. 文件改动清单(预估)

| 文件 | 类型 | 说明 |
|---|---|---|
| `vv/configs/config.go` | M | +SessionConfig +applyDefaults +env override |
| `vv/configs/config_test.go` | M | +TestSessionConfig_* |
| `vv/setup/setup.go` | M | buildHookManagerAndSession 重构 + InitResult.SessionStore |
| `vv/setup/setup_test.go` | M | +TestInit_Session_* |
| `vv/main.go` | M | --session flag + main 分支 |
| `vv/cli/cli.go` | M | App.sessionStore + WithSessionResume + Run resume logic |
| `vv/cli/session.go` | A | PrepareSessionID / PrintSessionList / TouchSession |
| `vv/cli/session_test.go` | A | 单元测试 |
| `vv/httpapis/http.go` | M | Serve 签名加 sessionStore + mux 注入 |
| `vv/httpapis/sessions.go` | A | 5 个 handler |
| `vv/httpapis/sessions_test.go` | A | endpoint 单元测试 |
| `vv/integrations/session_tests/cli_resume_tests/main_test.go` | A | 续接 e2e |
| `vv/integrations/session_tests/http_sessions_tests/main_test.go` | A | HTTP CRUD e2e |
| `doc/design/session-context-solution.md` | M | 标记完成(documenter 阶段) |
| `vv/CLAUDE.md` | M | 新增 Session 段(documenter 阶段) |

总文件数:13 个(developer 阶段) + 2 个文档(documenter 阶段)。

## 11. 风险与权衡

1. **History 不重放** — 用户期望可能是"接着昨天的对话讲",当前只能"接着昨天的 id 写"。在 banner 里明示这一点("resume = id only;message history not restored")可以管住预期。后续 P8 checkpoint 会真正解决。
2. **List 端点 + ListEvents 计数** — `len(events)` 计数需要遍历 events.jsonl,N 个 session 时 List 是 O(总事件数)。MVP 可以先计入返回体,如果实测慢,改为只 stat 文件大小或加 `?include_event_count=false`。**本次按朴素版做**;真要上生产再加索引。
3. **HTTP 端点鉴权缺失** — 与现有 `/v1/memory/*` 一致,假设由本机/可信网络消费。文档需提示;真正多租户上线时统一加。
4. **Resume 时序** — 如果用户在 session 还没 SessionHook 写第一个事件之前就退出再续接,store 里只有 `Touch` 留下的空 meta。这在产品语义上是"复用 id 但没历史",可接受。
5. **Graceful Stop 时序** — Manager.Stop 把上下文超时透传给每个 hook 的 Stop;FileSessionStore 没有内部 goroutine,只有 SessionHook 的 consumer goroutine(一个),Stop 等待 wg。和 trace 的语义完全对齐。

## 12. 不动的事项(仅供 reviewer 校对)

- `vage/session` 包不动。
- `vage/context` 包不动。
- `setup.New`(子 agent 装配)签名不动 —— SessionStore 不下发到 TaskAgent。
- `httpapis/http.go` 已有的 `/v1/memory` / `/v1/interactions` / `/v1/eval` / `/v1/budget` 路由不动。
- 现有 trace 行为不动(端到端可以同时启用 trace + session)。
