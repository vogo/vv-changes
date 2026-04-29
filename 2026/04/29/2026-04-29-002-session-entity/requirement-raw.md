# 需求原稿(Raw Requirement)

## 用户原始诉求

参考 `doc/design/session-context-solution.md`,评估方案完整性,调研行业优秀实践,设计方案并实现。

本次要实现的功能是: **Session 实体化(Session Entity-ization)**

## 上下文

`doc/design/session-context-solution.md` 已经做了系统性的业界调研,并在第 4.1 节给出了 Session 实体化的初步草案:

```go
type Session struct {
    ID         string
    AgentID    string
    UserID     string
    Title      string
    State      SessionState
    Events     []schema.Event
    StateKV    map[string]any
    Checkpoints []Checkpoint
    CreatedAt, UpdatedAt time.Time
    Metadata   map[string]any
}

type SessionStore interface {
    Create(ctx, *Session) error
    Get(ctx, id string) (*Session, error)
    Append(ctx, id string, e schema.Event) error
    Snapshot(ctx, id string, ckpt Checkpoint) error
    Restore(ctx, id string, ckptID string) (*Session, error)
    List(ctx, filter SessionFilter) ([]*Session, error)
}
```

并定义了与 memory 的关系:
- `Session.Events` = 事实全集(append-only,永不删)
- `memory.Working` = 当次 Run 的临时态
- `memory.Session` = 由 Session.StateKV 投影而来的"快取热数据"
- `memory.Store` = 跨 session 的长期事实

## 当前 vage / vv 现状(已调研)

- vage 中 `session_id` 仅作为 `memory.Entry` 的字符串标签,无独立实体;
- `vv/memories/session.go` 仅提供 `WithSessionID` / `SessionIDFrom` context helper;
- `schema.Event` 已经存在,所有 agent 生命周期事件类型齐全;
- `schema.RunRequest` 携带 `SessionID` 但无对应实体可寻址;
- 没有任何持久化的 Session 抽象; CLI 每次启动都是孤岛, HTTP 也无法基于 session_id 续接对话。

---

## Q&A(analyst 阶段澄清)

### Q1. 本次迭代范围
**选**: Vage 内基础实体。

在 `vage/session/` 新增包: Session 实体 + 事件流 + StateKV + SessionStore 接口 + 内存/文件系统两个实现 + 与 hook.Manager 的可选自动捕获桥。
**不含** checkpoint/snapshot/restore (留 P8 后续迭代)。
**不强制 wire 到 vv**(留为 follow-up)。

### Q2. 持久化后端
**选**: 内存 + 文件系统。

- `MapSessionStore` —— in-memory,测试与默认。
- `FileSessionStore` —— 默认根 `~/.vv/sessions/<id>/`,内含 `meta.json`、`events.jsonl`、`state.json`。

不引入 SQLite。

### Q3. 与 hook.Manager / tracelog 关系
**选**: 提供桥但不默认开启。

新增 `SessionHook` 实现 `hook.AsyncHook`,把 `schema.Event` 自动 append 到对应 Session;调用方显式注册才生效。tracelog JSONL 路径不变,二者并存(职责不同:tracelog = 被动落盘审计,Session = 有结构的可寻址实体)。


