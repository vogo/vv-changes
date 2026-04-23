# Design: 跨会话数据强隔离（Namespace 绑定 SessionID）

## 1. 设计目标与约束

- **最小侵入**：改动集中在 `vv/memories/`，保留 `memory.Store` 接口不变；vage 层 `PersistentMemory` 不需要改动。
- **默认拒绝，显式共享**：session-private namespace 写入必带 session_id；读时按 session_id 过滤；共享 namespace 走 allowlist 绕过校验。
- **向后兼容**：遗留 `fileRecord` 无 `session_id` 字段时视为"legacy shared"，可读不可被覆盖（由非共享 namespace 触发时）。
- **两类入口清晰区分**：
  - **User 入口**（CLI `/memory`、HTTP `/v1/memory/*`）→ 只能操作共享 namespace；对 session-private namespace 写入返回 403/错误提示。
  - **Agent 入口**（未来的 `write_memory` 工具等）→ 通过 context 注入 session_id；强制绑定校验。
- **行业经验落地**（取自调研）：
  - Mem0 / LangGraph 模式：session_id 作为 record 一等公民。
  - RLS 中间件模式：在存储层"拦截所有读写入口"一致注入过滤条件。
  - Key-prefix 风险规避：session_id 不放进对外 key，避免调用方手工拼接绕过；只在 record 元数据与校验逻辑里使用。

## 2. 行业方案对照选型

| 候选方案 | 描述 | 是否采用 | 理由 |
|---|---|---|---|
| A. Key 前缀方案 | Key = `"session:<id>:<key>"`，调用方负责拼接 | ❌ | 客户端可伪造前缀；绕过天然存在 |
| B. 分目录方案 | 每 session 一个子目录 `<dir>/session/<id>/...` | 部分采用 | 文件布局上采用（session-private 条目落盘到独立子目录），**但校验仍走 record 元数据**而非信赖路径 |
| C. Record 元数据 + 框架层校验（Mem0/Zep 风格） | `fileRecord.SessionID` 作为校验真相源；读写统一 Store 层做 RLS | ✅ 主方案 | 客户端无法绕过；单点真相；符合 Mem0/Zep/LangGraph 共识 |
| D. 共享 namespace allowlist（Claude Code 项目级/用户级分层） | 把 "shared vs private" 维度显式纳入 dictionary；allowlist 决定是否绕过校验 | ✅ 辅助 | 匹配已有的 4 个内置 namespace 语义；语义清晰 |

最终方案 = **C + D + B 的文件布局优化**。

## 3. 代码级架构

### 3.1 新增类型 / 常量（`vv/memories/session.go`）

```go
package memories

import (
    "context"
    "errors"
)

// ErrSessionForbidden is returned when a session-scoped entry is accessed
// from a different session (via Agent path) or when the user path tries to
// write a session-private namespace.
var ErrSessionForbidden = errors.New("memories: session access forbidden")

// sessionCtxKey is the private context key for the current session_id.
type sessionCtxKey struct{}

// userPathCtxKey is the private context key for "user path" marker (CLI / HTTP).
type userPathCtxKey struct{}

// WithSessionID returns a context with the session_id attached. Agent-path
// callers (tool handlers invoked by TaskAgent) use this to inform the store
// about the owning session.
func WithSessionID(ctx context.Context, sessionID string) context.Context {
    if sessionID == "" {
        return ctx
    }
    return context.WithValue(ctx, sessionCtxKey{}, sessionID)
}

// SessionIDFrom extracts the session_id from the context. Returns "" when
// none set. Callers MUST treat empty as "no agent-path identity".
func SessionIDFrom(ctx context.Context) string {
    v, _ := ctx.Value(sessionCtxKey{}).(string)
    return v
}

// WithUserPath marks the context as a "user-path" operation (CLI / HTTP memory
// CRUD). User-path operations:
//   - bypass session_id matching on reads
//   - are restricted to shared namespaces on writes/deletes
func WithUserPath(ctx context.Context) context.Context {
    return context.WithValue(ctx, userPathCtxKey{}, true)
}

// IsUserPath reports whether the context is marked as user-path.
func IsUserPath(ctx context.Context) bool {
    v, _ := ctx.Value(userPathCtxKey{}).(bool)
    return v
}
```

### 3.2 共享 Namespace Allowlist（`vv/memories/namespaces.go`）

```go
package memories

// defaultSharedNamespaces mirrors the four values defined in
// doc/prd/dictionaries/core/dictionary-memory-namespace.md.
// Any namespace NOT in this set is treated as session-private.
var defaultSharedNamespaces = map[string]struct{}{
    "project":     {},
    "user":        {},
    "conventions": {},
    "notes":       {},
    "default":     {}, // retain backward-compat with the implicit default
}

// isShared reports whether a namespace is globally shared across sessions.
func isShared(ns string, extra map[string]struct{}) bool {
    if _, ok := defaultSharedNamespaces[ns]; ok {
        return true
    }
    if extra != nil {
        _, ok := extra[ns]
        return ok
    }
    return false
}
```

（`extra` 预留 hook，后续用户可在 config 里追加——首版不暴露到 YAML。）

### 3.3 `FileStore` 扩展

保留 `FileStore` 现有 API，仅：

1. 在 `fileRecord` 增加 `SessionID string \`json:"session_id,omitempty"\``。
2. 在 `NewFileStore` 上增加 functional option: `WithSharedNamespaces(extra ...string)`（为未来 config 扩展留接口；首版 `setup.New` 不传）。
3. 五大方法内部先查 `IsUserPath(ctx)` + `SessionIDFrom(ctx)` + `parseKey(key)` 得到 `(isUser, sid, ns, _)`。
4. 路径判定矩阵：

| 入口 | Namespace | 动作 | 处理 |
|---|---|---|---|
| User（IsUserPath=true） | shared | 任意 | 直接放行；写入时 `record.SessionID = ""` |
| User | session-private | Set/Delete | 返回 `ErrSessionForbidden` |
| User | session-private | Get/List | 视为 not-found（不泄露存在性）；List 自动过滤掉 |
| Agent（有 sid） | shared | 任意 | 直接放行；写入时 `record.SessionID = ""`（避免"把共享条目锁定到会话") |
| Agent | session-private | Set | `record.SessionID = sid`；落盘到 `<dir>/session/<sid>/<key>.json` |
| Agent | session-private | Get | 读盘后若 `record.SessionID != sid` 视为 not-found；否则返回 |
| Agent | session-private | Delete | 读盘校验 `SessionID == sid`；不匹配返回 `ErrSessionForbidden`；`not found` 照旧返回 nil |
| Agent | session-private | List(prefix) | 遍历全部文件，`record.SessionID == sid` 才返回；加 prefix 过滤 |
| 无上下文（sid="" 且非 User） | shared | 任意 | 放行（兼容旧调用方） |
| 无上下文 | session-private | 任意 | 默认按 User-path 严格模式 → 写返回 `ErrSessionForbidden`；读走 legacy 兼容 |

5. **Legacy 条目兼容（US-4）**：
   - Set 覆盖时，若目标文件存在且 `record.SessionID != sid`（新 sid 与旧记录不匹配）→ 返回 `ErrSessionForbidden`。
   - Get/List 读到无 `session_id` 的旧记录时：
     - 在 shared namespace 中：照常返回（与现状一致）。
     - 在 session-private namespace 中：视为 legacy shared——只读可见给任意调用方；写入覆盖触发上一条规则保护。

6. **文件布局演进**（向后兼容）：
   - 共享 namespace 条目仍然落在 `<dir>/<ns>/<key>.json`。
   - session-private 条目落在 `<dir>/session/<sid>/<key>.json`（键会经 sanitize）。
   - `filePath(key, sid)` 按照 `isShared(ns)` 分流：
     - shared → `<dir>/<ns>/<name>.json`
     - private with sid → `<dir>/session/<sanitize(sid)>/<sanitize(ns)>_<sanitize(name)>.json`
   - 遗留文件（位于 `<dir>/<ns>/<name>.json` 但 ns 不在 shared 集合）继续在原路径可读——`Get/List` 增加一个"fallback 到旧布局"的回退路径，首次读到即可触发 lazy-migration（仅当有 write 上下文时做迁移；否则保持只读）。
   - **首版策略**：不做自动迁移；回退路径只读；文档提示用户手动迁移。

### 3.4 `Get` / `Set` / `Delete` / `List` / `Clear` 关键伪代码

```go
func (s *FileStore) Get(ctx context.Context, key string) (any, bool, error) {
    sid, isUser := SessionIDFrom(ctx), IsUserPath(ctx)
    ns, name := parseKey(key)
    shared := isShared(ns, s.extraShared)

    fp := s.resolveReadPath(ns, name, sid, shared)   // walks private path first, then legacy
    data, err := os.ReadFile(fp)
    if os.IsNotExist(err) { return nil, false, nil }
    if err != nil { return nil, false, fmt.Errorf(...) }

    var rec fileRecord
    if err := json.Unmarshal(data, &rec); err != nil { return nil, false, fmt.Errorf(...) }

    if !shared {
        // session-private namespace → enforce match
        if rec.SessionID != "" {
            // normal scoped entry: require sid match (any user path -> not-found)
            if isUser || rec.SessionID != sid {
                slog.Warn("memories: session mismatch on get",
                    "namespace", ns, "expected", rec.SessionID, "got", sid, "user_path", isUser)
                return nil, false, nil  // not-found posture
            }
        }
        // rec.SessionID == "" => legacy entry in private ns; treat as shared-read (no match check)
    }
    // TTL handling unchanged
    return rec.Value, true, nil
}

func (s *FileStore) Set(ctx context.Context, key string, value any, ttl int64) error {
    sid, isUser := SessionIDFrom(ctx), IsUserPath(ctx)
    ns, _ := parseKey(key)
    shared := isShared(ns, s.extraShared)

    if !shared {
        // private write rules
        if isUser {
            return fmt.Errorf("%w: user path cannot write private namespace %q",
                ErrSessionForbidden, ns)
        }
        if sid == "" {
            return fmt.Errorf("%w: no session_id in context for private namespace %q",
                ErrSessionForbidden, ns)
        }
    }

    // collision check on overwrite
    fp := s.resolveWritePath(ns, key, sid, shared)
    if existing, err := readRecord(fp); err == nil {
        if !shared && existing.SessionID != "" && existing.SessionID != sid {
            return fmt.Errorf("%w: overwrite blocked (owner=%s, caller=%s)",
                ErrSessionForbidden, existing.SessionID, sid)
        }
    }

    rec := fileRecord{..., SessionID: sidForRecord(shared, sid)}
    // write
}
```

- `sidForRecord(shared, sid)`: 共享 namespace 或 user 路径下为空，私有路径为 sid。

### 3.5 `List` 过滤语义

- 遍历 shared 目录、legacy 目录 + `<dir>/session/<sid>/` 目录。
- 在 Agent 路径（有 sid）：返回 shared + legacy + 自身 session 目录中条目。
- 在 User 路径：仅返回 shared + legacy。
- 遍历时 `rec.SessionID` 非空且 ≠ sid 的条目跳过（兜底防御，即使误读错目录也拦住）。

### 3.6 `Clear` 语义

- **保守策略**：保持既有 "删除根目录全部内容" 语义——但仅限 User 路径；Agent 路径调用 `Clear` 返回 `ErrSessionForbidden`。
- 这避免了"Agent 一次调用把所有共享条目清空"的风险，同时不破坏现有 `TestFileStore_Clear` 测试。

### 3.7 入口层改造

1. **CLI（`vv/cli/memory.go`）**：
   - 所有 `ctx := context.Background()` 改为 `ctx := memories.WithUserPath(context.Background())`。
   - 用户试图写/删 `session:*` 或任意非共享 namespace 时，直接在 CLI 层拦截并打印"禁止用户写入会话私有 namespace"，不让错误冒到 Store 层。

2. **HTTP（`vv/httpapis/http.go`）**：
   - `handleListMemory/handleGetMemory/handleSetMemory/handleDeleteMemory` 在操作 `mem` 前用 `ctx = memories.WithUserPath(ctx)` 包裹请求上下文。
   - Set/Delete 在操作前做与 CLI 相同的 namespace 校验；私有 namespace 返回 403 JSON: `{"code":"forbidden","message":"namespace X is session-private"}`。

3. **Setup / Agent 工具侧**：
   - 本次不挂 Agent 级 `write_memory` 工具（out-of-scope）。
   - 在 `setup.New` 中不做修改；`PersistentMemory` 直接透传 context。
   - 为将来 P3 自学习留接口：Tool handler 里可自行用 `memories.WithSessionID(ctx, runSessionID)` 包裹 ctx 再调用 `memory.Memory.Set`。

## 4. 数据结构变更

### 4.1 `fileRecord`

```go
type fileRecord struct {
    Key       string    `json:"key"`
    Value     string    `json:"value"`
    Namespace string    `json:"namespace"`
    SessionID string    `json:"session_id,omitempty"` // NEW: owner session; empty = shared/legacy
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    TTL       int64     `json:"ttl"`
}
```

`omitempty` 保证共享 namespace 的 JSON 文件体积与语义不变；仅在 session-private 条目里出现新字段。

### 4.2 磁盘路径

- 共享 namespace: `<dir>/<ns>/<key>.json`（不变）。
- session-private: `<dir>/session/<sid>/<ns>__<key>.json`（新增）。使用双下划线分隔 ns 与 key 以避免冲突；`sanitize` 复用既有规则。

（首版仅按 sid 建子目录即可实现物理隔离；文件名里再带 ns 只是为了 list-by-prefix 时能按 ns 过滤而不必解析 JSON。）

## 5. 测试设计（Developer 阶段落实）

**单元测试（`vv/memories/filestore_test.go` 扩展）**：

1. `TestFileStore_UserPath_SharedWrite_OK` — user 路径写 `project:x` 成功。
2. `TestFileStore_UserPath_PrivateWrite_Forbidden` — user 路径写 `ephemeral:x` 返回 `ErrSessionForbidden`。
3. `TestFileStore_AgentPath_PrivateWriteRead_RoundTrip` — Agent A 写 `scratch:x` → Agent A 读 OK。
4. `TestFileStore_AgentPath_CrossSessionRead_NotFound` — Agent A 写 `scratch:x` → Agent B 读 not-found。
5. `TestFileStore_AgentPath_CrossSessionOverwrite_Forbidden` — Agent A 写 `scratch:x` → Agent B 写 `scratch:x` 返回 `ErrSessionForbidden`。
6. `TestFileStore_AgentPath_CrossSessionDelete_Forbidden` — 同上，Delete 返回 forbidden。
7. `TestFileStore_List_FiltersBySession` — A 写 `scratch:x`、B 写 `scratch:y`、共写 `project:z`；A 调 List 返回 `scratch:x + project:z`。
8. `TestFileStore_Legacy_NoSessionID_Readable` — 手工构造无 `session_id` 字段的旧文件 → Get 仍成功。
9. `TestFileStore_Legacy_NoSessionID_PrivateNamespace_WriteBlocked` — 同上，Agent B Set 覆盖返回 `ErrSessionForbidden`。
10. `TestFileStore_Clear_UserPath_OK / Clear_AgentPath_Forbidden`。
11. `TestFileStore_NoSessionID_PrivateNamespace_Write_Forbidden` — 无 session 上下文 Set `scratch:x` 返回 forbidden。
12. 保留既有所有单元测试不变。

**集成测试复用**：`vv/integrations/http_tests/http_test.go` 现有 memory 用例全走 user path，应该无需改动；加一个 "HTTP Set `ephemeral:x` → 403" 验证人机路径限制。

## 6. 风险与缓解

| 风险 | 触发场景 | 缓解 |
|---|---|---|
| 现有调用方未包 `WithUserPath` | 某处内部代码直接用 `context.Background()` 对 `PersistentMemory` 调 Set/Delete/Clear 一个私有 ns | 私有 ns 依赖 session_id 或 user_path；未包的直接得到 `ErrSessionForbidden`——失败即显性，便于发现 |
| `PersistentMemoryPrompt` 列出所有 entries 时漏过滤 | `List("")` 在无上下文时会返回什么？ | 无 session + 非 user 路径：返回 shared + legacy。保持现在对 prompt 拼接没有破坏性；有 session 上下文时只返回 shared + legacy + 自己的 |
| Windows/路径 sanitize | session_id 可能包含奇怪字符 | 复用现有 `sanitize`；session_id 由 `hex.EncodeToString` 生成（见 `vv/cli/cli.go:102`），天然是 hex 字符 |
| 并发读写 | FileStore 注释已说明自身非并发安全；`syncMemory` 已提供 mutex | 保持原注释，不做改动 |
| List 性能退化 | 多出 `<dir>/session/` 子目录遍历 | session 目录按 sid 分子目录，只遍历当前 sid + shared 一次 |

## 7. 影响文档（Documenter 阶段更新）

- `doc/prd/models/core/memory/model-memory-entry.md`：新增 `session_id` 属性说明（optional; null for shared）。
- `doc/prd/dictionaries/core/dictionary-memory-namespace.md`：新增 "Scope" 列（shared vs session-private），标注 default shared 集合。
- `doc/prd/procedures/core/memory/procedure-persistent-memory-management.md`：在 Rules 章节新增 MEM-11（session binding rule）、MEM-12（user path restriction）；在 Exception Handling 加入 "ErrSessionForbidden" 的场景。
- `doc/prd/overview.md`：在 "Covers" 列表中 persistent memory 条补充 "session-scoped isolation for Agent writes"。
- `doc/prd/architecture/architecture.md`：在 Memory Management 章节补一行 "Session-scoped access control on persistent store"。

## 8. 上线回滚策略

- 失败回滚代价低：只改 `vv/memories/*` 与两处入口（CLI / HTTP）；revert 一个 commit 即可。
- 数据层向前兼容：新字段 `session_id` 经 `omitempty` 不影响旧版反序列化；session 子目录对旧代码不可见但也不会干扰（旧代码 List 时走原始 ns 目录遍历）。

## 9. 待 Developer 阶段确定的小项

- 是否把 `ErrSessionForbidden` 映射到 HTTP 404 而非 403 以避免存在性泄漏？—— 默认 403；Get 路径在遇到 session mismatch 时返回 not-found（内部），HTTP 层看到 `val==nil` 就给 404，与现有 handler 已有行为一致。
- `Clear` 在 user 路径下是否需要额外确认提示？—— 本次不加；维持 CLI 不暴露 `/memory clear`，HTTP 也无对应端点。
