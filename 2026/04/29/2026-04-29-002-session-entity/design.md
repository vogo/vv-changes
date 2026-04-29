# 技术设计 —— Session 实体化(MVP) v2

> 本设计为评审后的第二版,原始设计保留在 `design-raw.md`,变更说明见 `design-review.md`。

## 1. 总体设计原则

| 原则 | 体现 |
|---|---|
| 简单优先 | 单一新包 `vage/session/`,不新增跨包依赖,不破坏 memory/schema 现状 |
| 复用现有 | events 直接复用 `schema.Event`;hook 直接复用 `hook.AsyncHook`;ID 字符集与 tracelog 一致 |
| 物理布局清晰 | 文件后端目录结构与 `vv/traces/tracelog` 同档,JSONL 行格式逐字节一致 |
| 接口小而组合 | 拆为 `SessionMetaStore` / `SessionEventStore` / `SessionStateStore` 三个子接口,`SessionStore` 是组合 |
| 默认友好 | SessionHook `autoCreate=true`、ID 由调用方显式提供或显式调用 `GenerateID()` |
| 可逆 | MVP 接口签名稳定,checkpoint / EventQuery / 事务等是**新增**而非改写 |
| 文件 ≤800 行 | 拆为 6 个源文件,每个独立可读 |

## 2. 包布局

```
vage/session/
├── session.go         # Session 结构、SessionState 枚举、SessionFilter、GenerateID
├── errors.go          # ErrSessionNotFound、ErrSessionExists、ErrInvalidArgument
├── store.go           # 三个子接口 + SessionStore 组合接口
├── mapstore.go        # MapSessionStore (in-memory)
├── filestore.go       # FileSessionStore (filesystem) + atomic write helpers
├── hook.go            # SessionHook (implements hook.AsyncHook)
├── session_test.go
├── mapstore_test.go
├── filestore_test.go
├── hook_test.go
└── store_conformance_test.go  # Map/File 共用 conformance 黑盒测试
```

每个源文件预估 ≤350 行,远低于 800 行上限。

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
// Events and state KV are addressable separately via SessionStore — Session
// itself only carries identity and lifecycle metadata.
type Session struct {
    ID        string         `json:"id"`
    AgentID   string         `json:"agent_id,omitempty"`
    UserID    string         `json:"user_id,omitempty"`
    Title     string         `json:"title,omitempty"`
    State     SessionState   `json:"state"`
    Metadata  map[string]any `json:"metadata,omitempty"` // free-form; agreed convention: metadata["tags"] = []string
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
}

// New constructs a Session with the given non-empty ID, sets State=Active,
// and CreatedAt/UpdatedAt to time.Now(). Panics if id == "" — callers must
// either supply an externally generated id or call GenerateID() explicitly.
func New(id string) *Session

// GenerateID returns a sortable, filesystem-safe session id of the form
// "<unix-nanos>-<8-byte-hex>". Character set is constrained to
// [A-Za-z0-9._-] (length ≤ 128), matching tracelog's session id rules so
// the same id can be reused across the two systems without sanitisation.
func GenerateID() string
```

**设计要点**:
- Session 仅保留**元数据**;events 与 state 通过 store 单独寻址,避免 `Get` 把全量加载进内存。
- `New("")` 显式 **panic**:让两套 ID 来源(调用方与 Session 包)互相干扰的失败模式提前暴露。需要"如果空就生成"语义的调用方在外层做 `if id == "" { id = session.GenerateID() }`。
- `State` 默认 `Active`;状态迁移由调用方更新。
- ID 字符集与长度限制(`^[A-Za-z0-9._-]{1,128}$`)对外作为常量 `IDPattern` / `IDMaxLen` 暴露,方便集成方在更早阶段校验。

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

仅最少必要字段;`CreatedAt` 范围/全文搜索/标签过滤等留作后续。

### 3.3 错误

```go
var (
    ErrSessionNotFound = errors.New("session: not found")
    ErrSessionExists   = errors.New("session: already exists")
    ErrInvalidArgument = errors.New("session: invalid argument")
)
```

不存在的 state key:
- `GetState` 返回 `(nil, false, nil)`(惯用 Go 形态);
- `DeleteState` 是 no-op,不报错(幂等)。

## 4. `SessionStore` 接口分解

将原来的"12 方法胖接口"拆成三组小接口,组合成 `SessionStore`。下游可按最小依赖取(SessionHook 只依赖事件子接口),mock 也更轻。

```go
// SessionMetaStore: 元数据 CRUD + List。
type SessionMetaStore interface {
    Create(ctx context.Context, s *Session) error
    Get(ctx context.Context, id string) (*Session, error)
    Update(ctx context.Context, s *Session) error
    Delete(ctx context.Context, id string) error
    List(ctx context.Context, f SessionFilter) ([]*Session, error)
}

// SessionEventStore: append-only 事件流。
type SessionEventStore interface {
    AppendEvent(ctx context.Context, id string, e schema.Event) error
    ListEvents(ctx context.Context, id string) ([]schema.Event, error)
}

// SessionStateStore: 覆盖语义的结构化 state KV。
type SessionStateStore interface {
    GetState(ctx context.Context, id, key string) (any, bool, error)
    SetState(ctx context.Context, id, key string, value any) error
    DeleteState(ctx context.Context, id, key string) error
    ListState(ctx context.Context, id string) (map[string]any, error)
}

// SessionStore: 所有内置实现都满足这个组合接口。
type SessionStore interface {
    SessionMetaStore
    SessionEventStore
    SessionStateStore
}
```

**接口说明**:
- 元数据:常规 CRUD + List,语义对齐 `memory.Memory` 风格。
- 事件流:append-only,只能追加和顺序读取。MVP **不**暴露 `EventQuery`(类型/时间/分页过滤)—— 调用方按需在内存中过滤;若未来真出现需求,新增 `ListEventsQuery(ctx, id, EventQuery)` 方法,而非修改既有签名,保持向后兼容。
- 状态 KV:覆盖语义,与 events 互补;**不**做事务/批量 set/CAS,过度设计的可能性远高于真实需求。

返回的 `*Session` 都是 caller-owned 的副本,store 内部不共享指针;返回的 `[]schema.Event` 也是浅拷贝的 slice(`schema.Event` 是值类型,`Data` 字段为 `EventData` 接口,所有内置实现都是值类型且不暴露指针字段,所以浅拷贝足够)。

## 5. `MapSessionStore` 设计

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
- 单一 `sync.RWMutex` 保护整张表 —— 简单粗暴但完全够用。
- 读路径(`Get` / `List` / `ListEvents` / `GetState` / `ListState`)拷贝出去,避免外部并发修改撕坏内部数据。
- `AppendEvent` 拷贝传入的 event,append 到内部 slice,顺带更新 `meta.UpdatedAt`(内存里更新无 I/O 代价,与 FileStore 行为有意分歧 —— 见 §6)。
- 不实现 TTL / 容量上限 —— 进程内,信任调用方。

**ID 校验**: `Create` 拒绝 ID 为空、不匹配 `IDPattern`、长度 > `IDMaxLen` 的输入(`ErrInvalidArgument`),与 FileStore 完全一致 —— 这是两个实现可互换的前提。

## 6. `FileSessionStore` 设计

### 6.1 目录结构

```
<root>/                 # 默认 ~/.vv/sessions/  (可配置)
└── <session_id>/
    ├── meta.json       # Session 元数据(覆盖写)
    ├── events.jsonl    # 一行一个 schema.Event,append-only
    └── state.json      # state KV 全量(覆盖写)
```

`<session_id>` 必须满足 `^[A-Za-z0-9._-]{1,128}$`,与 `vv/traces/tracelog` 的 `sanitizeSessionID` 字符集与上限完全一致。两个系统可共享同一组 session id,不需要二次转义。

### 6.2 events.jsonl 行格式

`events.jsonl` 的每一行是 `json.Marshal(schema.Event)` + `'\n'`,与 `vv/traces/tracelog` 写入的 `<sid>.jsonl` 行格式**逐字节一致**。这是有意为之 —— 未来若需要"导入旧 tracelog → Session"或反向,工具可以共用。

### 6.3 写入语义

| 操作 | 实现 | meta.UpdatedAt |
|---|---|---|
| `Create` | mkdir(排他,已存在则 ErrSessionExists);写 meta.json;创建空 events.jsonl 与空 state.json `{}` | 写入 |
| `Update` | atomic write(temp + rename + fsync)meta.json | 写入 |
| `Delete` | `os.RemoveAll(<root>/<id>)` | — |
| `AppendEvent` | open `events.jsonl` (`O_APPEND\|O_CREATE\|O_WRONLY`);写一行 JSON + `\n`;**不**触碰 meta.json | **不刷新**(见下) |
| `SetState` | 读 state.json → mutate → atomic write;同步刷新 meta.json | 写入 |
| `DeleteState` | 同上 | 写入 |
| `ListState` | 读 state.json | — |

**关于 meta.UpdatedAt 的取舍**(本版关键改动):
原设计每次 `AppendEvent` 都同步重写 meta.json。一个典型 ReAct 循环里 events 是几十到几百条 / 任务,每条都做 4~5 syscall 的 atomic-rewrite,既慢又对 SSD 不友好;而 `meta.UpdatedAt` 不是审计字段,只用于排序/列表展示,**最终一致即可**。

权衡后:
- `AppendEvent` 只做事件追加,不刷新 meta.json。`meta.UpdatedAt` 反映的是"最后一次 metadata 或 state 变更",而非"最后一条事件"。语义在 godoc 中清楚说明。
- 调用方若需要更精确的"最后活跃时间",可自行 `os.Stat(<root>/<id>/events.jsonl).ModTime()`。MVP 不内建。

吞吐收益:`AppendEvent` 从 `1 atomic-rewrite + 1 append` 降到 `1 append`,在批量事件场景下数量级改善。

### 6.4 并发控制

- **进程内**: per-session 串行化,使用 `sync.Map` + `LoadOrStore` 模式拿 mutex:
  ```go
  func (s *FileSessionStore) lockFor(id string) *sync.Mutex {
      v, _ := s.locks.LoadOrStore(id, &sync.Mutex{})
      return v.(*sync.Mutex)
  }
  ```
  所有写操作(Create/Update/Delete/AppendEvent/Set/DeleteState)都先 `lockFor(id).Lock()`。读路径不锁。

- **跨进程**: **不保证**。本次明示 out-of-scope,godoc 说明。后续若需要,按文件锁(`syscall.Flock`)再加一层,目前不引入 —— 引入了会拉高 Windows 兼容成本。

- **读路径的弱一致**: 不锁,直接读文件;偶发读到正在被原子重命名的旧版本是接受的。`AppendEvent` 与 `ListEvents` 间存在 ABA-like 时序模糊性,这与 OpenAI Threads 的最终一致语义同档,真实场景下无害。

### 6.5 atomic write helper

```go
// 内部 helper(私有,定义在 filestore.go 内)
func writeJSONAtomic(path string, v any) error {
    tmp := path + ".tmp"
    f, err := os.OpenFile(tmp, os.O_CREATE|os.O_TRUNC|os.O_WRONLY, 0o600)
    if err != nil { return err }
    enc := json.NewEncoder(f)
    enc.SetIndent("", "  ")
    if err := enc.Encode(v); err != nil { _ = f.Close(); _ = os.Remove(tmp); return err }
    if err := f.Sync(); err != nil { _ = f.Close(); _ = os.Remove(tmp); return err }
    if err := f.Close(); err != nil { _ = os.Remove(tmp); return err }
    return os.Rename(tmp, path)
}
```

权限沿用 tracelog 约定:dir `0o700`、file `0o600`。

## 7. `SessionHook` 设计

```go
type SessionHook struct {
    store      SessionStore
    ch         chan schema.Event
    filter     []string
    autoCreate bool                 // 见下,默认 true
    bufferSize int

    wg          sync.WaitGroup
    stopOnce    sync.Once

    // last-warned-id: 简单去重,避免同一 session 连续失败时刷屏
    lastWarnSID string
}

type Option func(*SessionHook)

func WithBufferSize(n int) Option           // 默认 1024,对齐 tracelog
func WithFilter(types ...string) Option     // 默认空 = 全订阅
func WithAutoCreate(b bool) Option          // 默认 true

func NewSessionHook(store SessionStore, opts ...Option) *SessionHook
```

**实现约束**:
- 实现 `hook.AsyncHook`(`EventChan`、`Filter`、`Start`、`Stop`)。
- 单消费者 goroutine,从 chan 读 event;`event.SessionID == ""` 直接跳过(对齐 tracelog 的"default" 分桶不同 —— Session 里不应该有匿名会话,无 SessionID 的事件没有归属语义)。
- `store.AppendEvent` 失败处理:
  - `ErrSessionNotFound` 且 `autoCreate=true`:用 `&Session{ID: e.SessionID, AgentID: e.AgentID, State: StateActive, CreatedAt: e.Timestamp}` 调 `Create`,然后重试 AppendEvent 一次。再次失败按下条处理。
  - 其它(或 `autoCreate=false`):若 `e.SessionID != lastWarnSID`,`slog.Warn` 一次并把 `lastWarnSID` 设为该 id;事件丢弃,不阻塞主路径。
- `Stop`: 关闭 chan,等待 wg。
- 默认 `BufferSize=1024`,与 tracelog 默认一致。

**`autoCreate` 默认值取舍**:
- 默认 `true`。理由:vage 现状下 SessionID 由调用方自由生成(`schema.RunRequest.SessionID`),没有"先 Create 后用"的强制契约。如果 hook 默认不 autoCreate,集成方接好了 hook 但不知道还要主动 Create,失败模式是"什么都没写"且无报错。默认 true 更贴合"挂上就工作"的预期。
- 集成方需要严格控制(例如希望未注册的 SessionID 报错)时可显式 `WithAutoCreate(false)`。

**与 tracelog 的并存**:
- `tracelog.JSONLHook` 直接落 JSONL,无业务结构;
- `SessionHook` 走 SessionStore,带 metadata 维护;
- 二者在 `hook.Manager` 上独立注册,互不感知,互不替代。两者订阅同一个事件源不会双重消费 —— Manager 是 fan-out。
- 两者的 JSONL 行格式逐字节一致(见 §6.2),未来工具可共用。

## 8. 与现有系统的关系

| 系统 | 改动 |
|---|---|
| `vage/memory/` | 不改,保留 `Entry.SessionID` 标签语义。Session 实体的 ID 与之复用 string |
| `schema.Event` / `schema.RunRequest.SessionID` | 不改 |
| `vage/hook/` | 不改;SessionHook 是 `AsyncHook` 的新实现者 |
| `vv/traces/tracelog` | 不改;与 SessionHook 并存,JSONL 行格式一致 |
| `vv/setup` | 本次**不**自动注册 SessionHook;留下 godoc 用例展示如何 wire |

## 9. 关键测试设计

### 9.1 单元测试(每文件一个 *_test.go)
- `session_test.go`: `New(id)` 在 `id==""` 时 panic、`GenerateID()` 唯一/可排序、`SessionState` 枚举值。
- `mapstore_test.go`: 全部方法各有 happy path + 1~2 个 error case;ID 校验拒绝 `""` / `"a/b"` / `".."` / 超长。
- `filestore_test.go`: 落盘格式断言(meta.json 字段、events.jsonl 行格式与 tracelog 比对)、atomic write 不留 .tmp 残留、AppendEvent 不触碰 meta.json 的回归测试。
- `hook_test.go`: 事件转发、`SessionID==""` 跳过、`autoCreate=true` 生效、`autoCreate=false` 时 `ErrSessionNotFound` 触发 warn 且去重(同 SessionID 只 warn 一次)、`Stop` 后 EventChan 不再使用。

### 9.2 接口 conformance 测试
- `store_conformance_test.go`: 一组黑盒用例(共 ~20 子测试)同时跑两个实现。
- 用 `t.Run("MapSessionStore", ...)`、`t.Run("FileSessionStore", ...)`,后者 `t.TempDir()` 隔离。
- 测试只依赖 `SessionStore` 接口 —— 任何后续后端(SQLite 等)都可直接复用这套测试。

### 9.3 并发测试
- `TestMapSessionStore_ConcurrentAppend`: 8 goroutine × 100 events,go test -race 通过,事件总数无丢失。
- `TestFileSessionStore_ConcurrentAppend`: 同上(同一 session)。

## 10. Token 与文件预算

预估各文件行数(含 license header 18 行):

| 文件 | 行数 |
|---|---|
| session.go | ~110 |
| errors.go | ~30 |
| store.go | ~80 |
| mapstore.go | ~250 |
| filestore.go | ~340(合并了 io helper) |
| hook.go | ~160 |
| 各 *_test.go | 100~250 |

无单文件超过 800 行。

## 11. 与 doc/design/session-context-solution.md 的对应

| 章节 | 对应实现 |
|---|---|
| §4.1 Session 实体显式化 | 本设计完整覆盖(扣除 Checkpoint) |
| §5 关键工程决策 - Session 存储后端 | MapStore + 文件系统(无 SQLite,本次足够) |
| §5 关键工程决策 - Events append-only | `AppendEvent`-only,无 update/delete |
| §8 差距汇总 - Session 实体化 | 本次落地 |
| §8 差距汇总 - Context Builder 抽象 | 下一迭代 |
| §8 差距汇总 - Checkpoint | P8 后续迭代 |

## 12. 后续工作明确清单

按本次实现完成后建议的下一步顺序:

1. **Context Builder 抽象**(§4.2): 引入 `ContextBuilder` 与多个 `ContextSource`,与 Session 结合产生真正的 prompt。
2. **vv 端 wiring**: setup 默认挂 SessionHook、CLI 启动恢复上次 session、HTTP `/v1/sessions/*` 端点。
3. **EventQuery / ListEventsQuery**: 在出现真实分页/类型过滤需求时新增方法(不改既有签名)。
4. **迭代级 Checkpoint**(§4.5): 在 Session 上加 `SnapshotState/RestoreState`(不是事件级别 snapshot,而是 state KV + 最近事件指针)。
5. **持久化后端**: 提供 SQLite 实现(与 MapStore/FileStore 平行,直接复用 conformance test)。
6. **跨进程文件锁**: 若 vv 多实例并发场景出现。

## 13. 设计变更摘要(相对 design-raw.md)

- **接口拆分**: `SessionStore` 拆为三个子接口(Meta / Event / State)再组合,SessionHook 仅依赖最小子集。
- **ID 生成**: `New("")` 改为 panic;独立 `GenerateID()` 函数;ID 字符集与 tracelog 显式对齐。
- **EventQuery 砍掉**: MVP `ListEvents` 无参数版;后续若需要新增 `ListEventsQuery`。
- **AppendEvent 不再刷新 meta.UpdatedAt**: 大幅降低高频事件流下的 I/O 代价;语义在 godoc 中说明为最终一致。
- **JSONL 行格式与 tracelog 显式对齐**: 字符集 + 单行 JSON + `\n`,逐字节一致。
- **SessionHook autoCreate 默认 true**: 避免"挂了 hook 却没写入"的隐性失败模式。
- **warn 去重简化**: 用单变量 `lastWarnSID` 替代 LRU。
- **per-session mutex**: 显式给出 `sync.Map.LoadOrStore` 模式,避免 lazy 创建竞态。
- **filestore_io.go 合并到 filestore.go**: 减少跨文件跳转,文件预算仍远低于 800 行上限。
