# 设计评审 —— Session 实体化(MVP)

本评审针对 `design-raw.md`,围绕"最小可行 + 与项目现有模式一致 + 不留下显然的破坏性升级路径"三条原则给出建议。每条建议带有【接受/拒绝】决议,被接受的项已并入新版 `design.md`。

## 1. 总体评价

原设计已经基本合理:
- 选择"元数据 + 事件流 + 状态 KV"三段式存储,语义清楚,与业界(OpenAI Threads、Anthropic Memory tool、ADK 等)一致。
- 文件后端目录结构与 `vv/traces/tracelog` 同档(同样使用 `<sid>/` 子目录与 0o700/0o600 权限),迁移/合并的成本低。
- 接口面适度,刻意排除了 checkpoint。

主要可优化的方向集中在两点:
- 接口粒度可以再小一档,把"单 key state 操作"合并、把不必要的 `EventQuery` 字段砍掉。
- 文件后端"每次 AppendEvent 同步重写 meta.json"的代价被低估了 —— 高频事件流下这是热点。

## 2. 评审条目

### A. SessionStore 是否一个胖接口? 【建议拆分:接受】

**观察**: 原设计把 metadata / events / state 三组共 12 个方法塞进同一个 `SessionStore`。这违反了项目里常见的"小接口组合"风格(参见 `memory.Store` 5 个方法 + `memory.BatchStore` 可选叠加)。下游(SessionHook)只需要 `AppendEvent` 一个方法 + `Create` 用于 autoCreate;一个胖接口让 mock 测试和未来加 SQLite 后端都更累。

**建议**: 拆为三个内聚接口,再用一个组合接口让两个内置实现都满足:

```go
type SessionMetaStore interface {
    Create(ctx, *Session) error
    Get(ctx, id) (*Session, error)
    Update(ctx, *Session) error
    Delete(ctx, id) error
    List(ctx, SessionFilter) ([]*Session, error)
}

type SessionEventStore interface {
    AppendEvent(ctx, id, schema.Event) error
    ListEvents(ctx, id) ([]schema.Event, error) // 或带 EventQuery,见 C
}

type SessionStateStore interface {
    GetState(ctx, id, key) (any, bool, error)
    SetState(ctx, id, key, value) error
    DeleteState(ctx, id, key) error
    ListState(ctx, id) (map[string]any, error)
}

// SessionStore 是三者的并集,默认所有内置实现都满足。
type SessionStore interface {
    SessionMetaStore
    SessionEventStore
    SessionStateStore
}
```

下游接收方按最小接口取。SessionHook 仅依赖 `SessionEventStore` (+ optional `SessionMetaStore` for autoCreate)。这是 MVP 即可享受的小重构,几乎零成本。

**决议**: 接受。

---

### B. `Session.New("")` 自动生成 ID 【建议改:接受】

**观察**: 原设计 `New(id string)` 在传 `""` 时回落到时间戳 + crypto/rand,看似贴心,但有两个问题:
1. 与 `RunRequest.SessionID` 的现状不协调 —— 调用方目前已经在外层(vv setup / HTTP handler)生成 session_id,Session 实体侧再做一次"如果空就生成"会出现两套 ID 来源,排错时不清楚谁是权威。
2. 会让 SessionHook autoCreate 路径上无 ID 也能产生 session,污染 store。

**建议**:
- `New(id string)` 当 `id==""` 直接 panic 或返回 `(nil, ErrInvalidArgument)`(更 Go-idiom 的做法是后者,但 `New` 当前签名不返回 error)。
- 把 ID 生成提到一个独立的 `func GenerateID() string`,谁需要谁显式调用 —— 保留便利性,但语义清晰。
- ID 字符集与 `tracelog.sanitizeSessionID` 完全一致(`[A-Za-z0-9._-]`,长度 ≤128),并复用一个常量,便于双向迁移。

**决议**: 接受。新版 `New(id string) *Session` 要求 `id != ""`(空时 panic + 文档说明),独立函数 `GenerateID()` 输出 `时间戳-rand` 形态。

---

### C. EventQuery 是否 MVP 必要? 【建议简化:接受】

**观察**: `EventQuery{Types, Limit, Offset, Since}` 四个字段在 MVP 阶段都没有强需求:
- `Types` 过滤可以让调用方在内存里做(events 体量在 MVP 不会很大,且没有 streaming 读取场景);
- `Limit/Offset` 页码语义在 append-only 数据集上有边界条件(insert-while-paging),先不做更稳;
- `Since` 在 events 已有 `Timestamp` 字段时调用方也能自己过滤。

**建议**: MVP 只保留 `ListEvents(ctx, id) ([]schema.Event, error)`。`EventQuery` 留作后续。如果后续需要,引入新的方法 `ListEventsQuery` 而不是修改既有签名,保持向后兼容。

**决议**: 接受。砍掉 `EventQuery`。

---

### D. 同步刷新 meta.UpdatedAt 在 FileStore 上的代价 【建议放宽:接受】

**观察**: 原设计每次 `AppendEvent` 都要做一次 meta.json 的 atomic-write(temp + rename + fsync)。一个典型 ReAct 循环里 events 是几十到几百条 / 任务,每条都做一次 4~5 syscall 的 atomic-rewrite,既慢也对 SSD 不友好。

更重要的是:**meta.UpdatedAt 不需要强一致** —— 它是用来排序/列表展示的,不是审计字段;偶尔落后几条事件不会有正确性问题。

**建议(MVP 可达)**:
- `AppendEvent` 时**只**追加 `events.jsonl`,**不**触碰 meta.json。
- meta.UpdatedAt 只在 `Update`、`SetState/DeleteState` 时同步刷新。
- 列表时若需要更精确的 UpdatedAt,可以 `Stat(events.jsonl).ModTime()` 作为 fallback(但 MVP 不必,直接返回 meta.json 中的 UpdatedAt 即可)。

这一处改动把 FileStore 的事件吞吐从 "每条事件 = 1 次 atomic-rewrite" 降到 "每条事件 = 1 次 append-write",收益巨大,代价仅为"meta.UpdatedAt 比真实最后写入时间略旧"。

**决议**: 接受。

---

### E. state.json 的全量重写 【保留:接受现状】

**观察**: 原设计 `SetState/DeleteState` 都是 read-modify-atomic-write 整个 state.json。

**评估**: 状态 KV 的写入频率应远低于事件流(典型场景:计划进度、最新摘要、变量),全量重写完全可以接受;引入增量 log 等于重新发明事件流。**保留原设计**。

**决议**: 不变。

---

### F. JSONL 事件格式与 tracelog 对齐 【加一句注释:接受】

**观察**: 原设计与 tracelog 的事件落盘格式都是"一行一个 `schema.Event` 的 JSON"。但原设计文档没有明文要求"必须与 tracelog 格式可互换"。未来若有"导入旧 tracelog → Session"或反向,统一格式是低成本前提。

**建议**: 在 §5 显式声明 "events.jsonl 的行格式与 `vv/traces/tracelog` 的 `<sid>.jsonl` 格式逐字节一致 —— 单行 `json.Marshal(schema.Event)` + `\n`",并引用 tracelog 的 `sanitizeSessionID`/字符集规则。便于未来共用工具脚本。

**决议**: 接受,文档加一句即可,不改代码语义。

---

### G. SessionHook 的 autoCreate 默认值 【建议默认 true:接受】

**观察**: 原设计 `autoCreate` 默认值未明确。如果默认 `false`,任何先没在 store Create 过 session 的 event 都会丢失,这与 vage 现状(`schema.RunRequest.SessionID` 直接传字符串、没有显式注册步骤)不匹配,导致 hook 实际"看着接好了但什么都不写"的失败模式。

**建议**: `autoCreate` 默认 `true`,文档说明"hook 是一个被动观察者,看到陌生 SessionID 即创建一条骨架 metadata"。需要严格控制时由集成方关闭。

**决议**: 接受。

---

### H. SessionHook 的 warn 抑制 【建议简化:接受】

**观察**: 原设计提到"用 LRU 或 last-warned-id 避免日志风暴"。MVP 阶段引入 LRU 是过度工程。

**建议**: 简单的 last-warned-id (单变量)足够 —— 同一 session 连续失败时只 warn 一次,看到不同 session id 重置。代码 5 行。

**决议**: 接受。

---

### I. Session.Metadata 的形态 【保留:接受现状】

**观察**: 原设计 `Metadata map[string]any`。是否需要单独的 `Tags []string` ?

**评估**: 现阶段没有明确的标签筛选需求,`SessionFilter` 也没暴露 tag 维度。`map[string]any` 已经能承载任何上层约定的标签结构,加 `Tags` 反而引入"两个地方放标签"的不一致。**保留原设计**,文档建议"标签使用 `metadata["tags"]` 约定"。

**决议**: 不变。

---

### J. 并发控制:lazy 创建的 per-session mutex 【建议改 sync.Map.LoadOrStore:接受】

**观察**: 原设计 "`sync.Map[id]*sync.Mutex`(lazy 创建)"。用 `sync.Map` 的话需要 `LoadOrStore` 模式,否则 lazy 创建本身就有竞态。

**建议**: 显式用 `sync.Map.LoadOrStore(id, &sync.Mutex{})` 拿到 mutex,文档明示这是"per-session serialization, not cross-process"。

**决议**: 接受,在新设计里给出代码片段。

---

### K. 测试粒度 【保持原样:接受】

原设计的 "conformance test + 并发 race test" 已经覆盖了主要风险。无需改动。

---

### L. 文件清单 【微调:接受】

原设计列出 7 个源文件 + 5 个测试文件。砍掉 `EventQuery` 后,`store.go` 内容变薄,可以把 errors.go 内联到 store.go 或 session.go(项目里其他包也常见这种合并)。但保持 `errors.go` 单独文件不算成本,**保留**。

把 `filestore_io.go` 与 `filestore.go` 合并的可能性: 评估文件预算后,filestore.go ~280 行 + io helper ~80 行 = 360 行,远低于 800 行上限,可以**合并为单一 filestore.go**,减少跨文件跳转。

**决议**: 接受合并 io helper 进 filestore.go。最终 6 个源文件。

---

## 3. 不接受的扩展(避免 churn)

下面这些点考虑过、明确**不**进 MVP:

- **跨进程文件锁(flock)**: 需求明示 out-of-scope。
- **events 二进制格式 / 压缩**: 与 §F 的 JSONL 互通性目标冲突,且 MVP 量级不需要。
- **Session 父子链 / Tree**: 需求明示 out-of-scope。
- **SessionStore.Stream(ctx, id) <-chan Event**: 看似贴近事件源,但 MVP 没有"实时订阅"消费方;hook.Manager 已经是订阅入口,Session 端再做一遍是重复。
- **state KV 的批量 SetMany / 事务**: 原设计已经拒绝,继续保持。
- **`EventQuery` 在 MVP 里**: 见 §C。

## 4. 给开发者阶段的开放问题

1. `New(id string)` 在 `id==""` 时是 panic 还是返回 nil?新版设计选 panic(与 `errors.New` 包级 var 同档,简化调用方),开发者实现时遵循即可,如有强烈反对可改成 fail-fast 的 ` Must`+`Try` 双路径,但不建议。
2. `MapSessionStore` 的读路径返回内部副本时,events slice 的元素 `schema.Event` 自身是值类型且 `Data` 字段是接口,理论上 caller 修改 `Data` 内部字段会污染共享 —— 实际 `EventData` 实现都是值类型且不暴露指针字段,所以浅拷贝 events slice 即可。开发者注意保留这一不变量。
3. FileStore 的 `Stat(events.jsonl).ModTime()` 在某些文件系统(如 NFS)会有 1 秒精度,但 MVP 直接用 meta.UpdatedAt 不依赖 ModTime,问题不存在。
