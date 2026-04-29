# 技术设计 —— Session 实体化(MVP)

## 1. 总体设计原则

| 原则 | 体现 |
|---|---|
| 简单优先 | 单一新包 `vage/session/`,不新增跨包依赖,不破坏 memory/schema 现状 |
| 复用现有 | events 直接复用 `schema.Event`,hook 直接复用 `hook.AsyncHook` |
| 物理布局清晰 | 文件后端目录结构与 `vv/traces/tracelog` 一致,二者并存 |
| 接口分层但单包 | 公开一个胖接口 `SessionStore`,实现按文件拆分 |
| 可逆 | 接口本次签名稳定,checkpoint/state-projection 等是**新增**而非改写 |
| 文件 ≤800 行 | 拆为 7 个源文件,每个独立可读 |

## 2. 包布局

```
vage/session/
├── session.go         # Session 结构、SessionState 枚举、SessionFilter
├── errors.go          # ErrSessionNotFound、ErrSessionExists 等
├── store.go           # SessionStore 接口、EventQuery 结构
├── mapstore.go        # MapSessionStore (in-memory)
├── filestore.go       # FileSessionStore (filesystem)
├── filestore_io.go    # FileSessionStore 的内部 I/O 助手 (atomic write 等)
├── hook.go            # SessionHook (implements hook.AsyncHook)
├── session_test.go
├── mapstore_test.go
├── filestore_test.go
├── hook_test.go
└── store_conformance_test.go  # Map/File 共用 conformance 黑盒测试
```

每个文件预估 ≤300 行。

## 3. 公开类型

### 3.1 `Session` (元数据)

```go
type SessionState string

const (
    StateActive    SessionState = "active"
    StatePaused    SessionState = "paused"
    StateCompleted SessionState = "completed"
    StateFailed    SessionState = "failed"
)

// Session is the metadata view of a persistent agent conversation.
// Events and state KV are addressable separately via SessionStore.
type Session struct {
    ID        string         `json:"id"`
    AgentID   string         `json:"agent_id,omitempty"`
    UserID    string         `json:"user_id,omitempty"`
    Title     string         `json:"title,omitempty"`
    State     SessionState   `json:"state"`
    Metadata  map[string]any `json:"metadata,omitempty"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
}

// New creates a Session with the given ID, sets State=Active and timestamps to now.
// Generated id (when "") falls back to a sortable UUID-like string built from time + crypto/rand.
func New(id string) *Session
```

**设计要点**:
- Session 仅保留**元数据**,不内嵌 `Events` / `StateKV` —— 二者是大体量数据,通过 store 单独寻址,避免 `Get` 把全量加载进内存。
- `New("")` 自动生成 ID(时间戳前缀 + 8 字节 crypto/rand 后缀,Hex 编码,易排序),**ID 必填** 是设计 invariant。
- `State` 默认 `Active`;迁移由调用方更新。

### 3.2 `SessionFilter`

```go
type SessionFilter struct {
    UserID  string         // 精确匹配,空忽略
    AgentID string         // 精确匹配,空忽略
    State   SessionState   // 精确匹配,空忽略
    Limit   int            // 0 = 不限
    Offset  int            // 偏移
}
```

仅最少必要字段;`CreatedAt` 范围/全文搜索等留作后续。

### 3.3 `EventQuery`

```go
type EventQuery struct {
    Types  []string  // 类型过滤;空 = 全部
    Limit  int       // 0 = 不限
    Offset int       // 跳过前 N 条
    Since  time.Time // 时间下限,Zero = 不限
}
```

### 3.4 `SessionStore` 接口

```go
type SessionStore interface {
    // ---- Session metadata (5) ----
    Create(ctx context.Context, s *Session) error
    Get(ctx context.Context, id string) (*Session, error)
    Update(ctx context.Context, s *Session) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, f SessionFilter) ([]*Session, error)

    // ---- Events: append-only (2) ----
    AppendEvent(ctx context.Context, id string, e schema.Event) error
    ListEvents(ctx context.Context, id string, q EventQuery) ([]schema.Event, error)

    // ---- Structured state: overwrite semantics (4) ----
    SetState(ctx context.Context, id, key string, value any) error
    GetState(ctx context.Context, id, key string) (any, bool, error)
    DeleteState(ctx context.Context, id, key string) error
    ListState(ctx context.Context, id string) (map[string]any, error)
}
```

**12 个方法,三组语义**:
- 元数据:常规 CRUD + List。
- 事件流:append-only,只能追加和读取,语义与 §1 用户故事 2 对齐。
- 状态 KV:覆盖语义,与 events 互补;故意**不**做 transactional 多 key 操作(用户场景简单,加事务是过度设计)。

返回 `*Session` 都是 caller-owned 的副本,store 内部不共享指针,避免外部修改污染状态。

### 3.5 错误

```go
var (
    ErrSessionNotFound = errors.New("session: not found")
    ErrSessionExists   = errors.New("session: already exists")
    ErrInvalidArgument = errors.New("session: invalid argument")
)
```

state key 不存在时:`GetState` 返回 `(nil, false, nil)`(惯用 Go 形态);`DeleteState` 不存在的 key 是 no-op,不报错(幂等)。

## 4. `MapSessionStore` 设计

```go
type MapSessionStore struct {
    mu       sync.RWMutex
    sessions map[string]*sessionRecord
}

type sessionRecord struct {
    meta   Session
    events []schema.Event
    state  map[string]any
}
```

**关键点**:
- 单一 `sync.RWMutex` 保护整张表,简单粗暴但足够。
- 读路径(`Get` / `List` / `ListEvents` / `GetState` / `ListState`)拷贝出去,避免外部并发修改撕坏内部数据。
- `AppendEvent` 拷贝传入的 event,append 到内部 slice,顺带更新 `meta.UpdatedAt`。
- 不实现 TTL / 容量上限 —— 进程内,信任调用方。

**ID 校验**:`Create` 拒绝 ID 为空、含 `/` 或 `..` 的输入(`ErrInvalidArgument`),与文件后端规则一致,保证两个实现可互换。

## 5. `FileSessionStore` 设计

### 5.1 目录结构

```
<root>/                 # 默认 ~/.vv/sessions/  (可配置)
└── <session_id>/
    ├── meta.json       # Session 元数据(覆盖写)
    ├── events.jsonl    # 一行一个 schema.Event,append-only
    └── state.json      # state KV 全量(覆盖写)
```

`<session_id>` 必须满足 `^[A-Za-z0-9._-]{1,128}$`,与 tracelog 一致。

### 5.2 写入语义

| 操作 | 实现 |
|---|---|
| `Create` | `mkdir <root>/<id>` (排他,已存在则 ErrSessionExists);写 meta.json;创建空 events.jsonl 与 state.json |
| `Update` | atomic write (temp + rename) meta.json |
| `Delete` | `os.RemoveAll(<root>/<id>)` |
| `AppendEvent` | open `events.jsonl` 以 `O_APPEND\|O_CREATE\|O_WRONLY` 打开;写一行 JSON + `\n`;同时刷新 meta.UpdatedAt(覆盖 meta.json) |
| `SetState` | 读 state.json → mutate → atomic write |
| `DeleteState` | 同上 |
| `ListState` | 读 state.json |

**meta.UpdatedAt 同步**:每次 `AppendEvent` / `SetState` / `DeleteState` 后,异步刷新 meta.json 容易引入竞态;直接同步刷新虽多一次小 I/O,但语义清晰、避免脏 stat。MVP 选同步刷新。

### 5.3 并发控制

- 进程内: 持有 `sync.Map[id]*sync.Mutex`(lazy 创建),按 session 串行化所有写操作。
- 跨进程: **不保证**(本次明示 out-of-scope,文档说明)。后续若需要,按文件锁(`syscall.Flock`)再加一层,目前不引入。
- 读路径: 不锁,直接读文件;偶发读到正在被原子重命名的旧版本是接受的。`AppendEvent` 与 `ListEvents` 间存在 ABA-like 时序模糊性,这与 OpenAI Threads 的最终一致语义同档。

### 5.4 atomic write

```go
// internal helper in filestore_io.go
func writeJSONAtomic(path string, v any) error {
    tmp := path + ".tmp"
    f, err := os.OpenFile(tmp, os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0o600)
    ...
    enc.Encode(v)
    f.Sync()
    f.Close()
    os.Rename(tmp, path)
}
```

权限沿用 tracelog 约定:dir `0o700`、file `0o600`。

## 6. `SessionHook` 设计

```go
type SessionHook struct {
    store    SessionStore
    ch       chan schema.Event
    filter   []string
    autoCreate bool        // optional: 若 session 不存在则隐式创建
    wg       sync.WaitGroup
    stopOnce sync.Once
}

type Option func(*SessionHook)

func WithBufferSize(n int) Option
func WithFilter(types ...string) Option
func WithAutoCreate(b bool) Option

func NewSessionHook(store SessionStore, opts ...Option) *SessionHook
```

**实现约束**:
- 实现 `hook.AsyncHook`(`EventChan`、`Filter`、`Start`、`Stop`)。
- 单消费者 goroutine,从 chan 读 event;`event.SessionID == ""` 跳过。
- `store.AppendEvent` 失败:
  - `ErrSessionNotFound`:若 `autoCreate=true`,先 `Create(&Session{ID: e.SessionID, AgentID: e.AgentID, State: StateActive, ...})` 再重试;否则 `slog.Warn` 一次后丢弃该事件,但记账(避免每条事件都 warn 形成日志风暴 —— 用 LRU 或 last-warned-id 简易去重)。
  - 其他错误:`slog.Warn` 单条事件,不阻塞主路径。
- `Stop`: 关闭 chan,等待 wg。
- 默认 `BufferSize=1024`,与 tracelog 默认一致。

**与 tracelog 的并存**:
- tracelog 每条 event 一行 JSON 落盘,无解析、无业务结构。
- SessionHook 走 SessionStore,带元数据维护。
- 二者在 hook.Manager 上独立注册,互不感知,互不替代。

## 7. 与现有系统的关系

| 系统 | 改动 |
|---|---|
| `vage/memory/` | 不改,保留 `Entry.SessionID` 标签语义。Session 实体的 ID 与之复用 string |
| `schema.Event` / `schema.RunRequest.SessionID` | 不改 |
| `vage/hook/` | 不改;SessionHook 是该接口的新实现者 |
| `vv/traces/tracelog` | 不改;与 SessionHook 并存 |
| `vv/setup` | 本次**不**自动注册 SessionHook;留下 godoc 用例展示如何 wire |

## 8. 关键测试设计

### 8.1 单元测试(每文件一个 *_test.go)
- `session_test.go`: `New("")` ID 唯一/可排序、`SessionState` 枚举值。
- `mapstore_test.go`: 12 个方法每个一个 happy + 1~2 个 error case。
- `filestore_test.go`: 等同上,落盘格式断言、atomic write 不留 .tmp 残留。
- `hook_test.go`: 事件转发、`SessionID==""` 跳过、`autoCreate=true` 生效、store 失败不阻塞、`Stop` 后再发不 panic。

### 8.2 接口 conformance 测试
- `store_conformance_test.go`: 一组黑盒用例(共 ~20 子测试)同时跑两个实现。
- 用 `t.Run("MapSessionStore", ...)`、`t.Run("FileSessionStore", ...)`,后者 `t.TempDir()` 隔离。

### 8.3 并发测试
- `TestMapSessionStore_ConcurrentAppend`: 8 goroutine x 100 events,go test -race 通过,事件总数与顺序无丢失/错乱。
- `TestFileSessionStore_ConcurrentAppend`: 同上(同 session)。

## 9. Token 与文件预算

预估各文件行数(含 license header 18 行):
- session.go ~80
- errors.go ~40
- store.go ~80
- mapstore.go ~250
- filestore.go ~280
- filestore_io.go ~80
- hook.go ~150
- 各 *_test.go 100~250

无单文件超过 800 行。

## 10. 与 doc/design/session-context-solution.md 的对应

| 章节 | 对应实现 |
|---|---|
| §4.1 Session 实体显式化 | 本设计完整覆盖(扣除 Checkpoint) |
| §5 关键工程决策 - Session 存储后端 | MapStore + 文件系统(无 SQLite,本次足够) |
| §5 关键工程决策 - Events append-only | `AppendEvent`-only,无 update/delete |
| §8 差距汇总 - Session 实体化 | ✅ 本次落地 |
| §8 差距汇总 - Context Builder 抽象 | ❌ 下一迭代 |
| §8 差距汇总 - Checkpoint | ❌ P8 后续迭代 |

## 11. 后续工作明确清单

按本次实现完成后建议的下一步顺序:
1. **Context Builder 抽象**(§4.2): 引入 `ContextBuilder` 与多个 `ContextSource`,与 Session 结合产生真正的 prompt。
2. **vv 端 wiring**: setup 默认挂 SessionHook、CLI 启动恢复上次 session、HTTP `/v1/sessions/*` 端点。
3. **迭代级 Checkpoint**(§4.5): 在 Session 上加 `SnapshotState/RestoreState`(注意:不是事件级别 snapshot,而是 state KV + 最近事件指针)。
4. **持久化后端**: 提供 SQLite 实现(与 MapStore/FileStore 平行)。
5. **跨进程文件锁**: 若 vv 多实例并发场景出现。
