# 需求规格 —— Session 实体化(MVP)

## 1. 背景与目标

### 1.1 背景

`doc/design/session-context-solution.md` 已经系统调研了业界 11 类框架(Anthropic、OpenAI、LangChain/LangGraph、LlamaIndex、MemGPT、Semantic Kernel/AutoGen、Google ADK、Devin/SWE-Agent、Cursor/Aider、Temporal/LangGraph durable execution、RAPTOR/A-MEM 等),提炼出 10 种 Session→Context 构造范式,并按优先级给出 vage / vv 的演进路径:

> Session 实体化 → Context Builder 抽象 → Plan 工作区 → 迭代级 Checkpoint → Context Editing → 向量召回标准化 → Session Tree → LLM 主动换页

**该方案的完整性评估**:

| 维度 | 评估 |
|---|---|
| 问题界定 | ✅ Session 与 Context 的概念辨析清晰、有可操作性 |
| 业界覆盖 | ✅ 11 类框架,涵盖 SDK 厂商 / 开源框架 / 学术原型 / 产品应用 |
| 范式归纳 | ✅ 10 种范式,P1~P10 可单独评估,亦可组合 |
| 落地路径 | ✅ 8 步演进,有先后依赖、有最小完备子集 |
| 工程决策 | ✅ §5 给出存储后端、装配粒度、token 预算分配等关键决策 |
| 现状差距 | ✅ §8 列出 12 项能力的现状/差距对照,Session 实体化是首要差距 |

**结论**: 设计文档已具备启动 Session 实体化所需的全部上下文,本次需求直接以 §4.1 与 §8 中的"Session 实体化"差距条目为目标。

### 1.2 本次目标

把 Session 从"`memory.Entry` 上的字符串标签"提升为 vage 中的**一等公民实体**。
- 提供独立的 `vage/session` 包,封装身份、事件流、结构化状态、生命周期与持久化。
- 提供两个开箱即用的存储实现:in-memory 与 filesystem。
- 提供与现有 `hook.Manager` 的可选桥接,把已有事件流以最小代价灌入 Session。
- 不破坏现有 `memory.Memory` 三层抽象,不改变 `schema.RunRequest.SessionID` 字段语义。

### 1.3 非目标(本次明确**不做**)

| 范畴 | 本次不做 | 留作后续 |
|---|---|---|
| Checkpoint / Snapshot / Restore | ❌ | P8 follow-up |
| Context Builder 抽象 | ❌ | §4.2 follow-up |
| Plan/Scratchpad 工作目录 | ❌ | §4.4 follow-up |
| Session Tree(P10) | ❌ | §4.8 follow-up |
| vv 端到端 wiring(CLI 续接、HTTP CRUD) | ❌ | 下一迭代 |
| SQLite / Postgres 后端 | ❌ | 落地后置 |
| 跨 Session / 父子 Session 链 | ❌ | P5 配套 |

## 2. 用户故事与验收标准

### 用户故事 1: Session 是可寻址实体

> 作为 vage 的下游使用者(vv 或第三方),我希望可以通过 `SessionID` 拿到一个 Session 对象,读取它的元数据、全量事件流、结构化状态,而不必自己维护这些数据。

**验收**:
- [ ] 存在 `session.Session` 结构体,字段至少含 `ID, AgentID, UserID, Title, State, CreatedAt, UpdatedAt, Metadata`。
- [ ] 存在 `session.Event` 切片字段或等价的 events 接入 API,事件类型即 `schema.Event`(无重复定义)。
- [ ] 存在 `session.SessionStore` 接口暴露 `Create / Get / Append / List / Delete`(checkpoint 系列接口本次**不必出现**)。
- [ ] `SessionState` 至少枚举 `active / paused / completed / failed`。

### 用户故事 2: 事件流 append-only

> 作为 agent 框架使用者,我希望事件一旦 append 就不能再修改/删除,以保留完整审计链路。

**验收**:
- [ ] `SessionStore.Append(ctx, id, event)` 仅追加,不暴露事件级别的 update/delete。
- [ ] 删除整条 Session(`SessionStore.Delete`)允许,但同时清掉所有事件——是会话级别的删除,不是事件级别的修剪。
- [ ] 文件后端的事件落盘形态为 append-only 文件(JSON Lines)。

### 用户故事 3: 结构化状态可读写

> 作为 agent 实现者,我希望在 Session 上保存"非事件"形态的结构化状态(如计划进度、变量、最新摘要)并随时读写。

**验收**:
- [ ] Session 持有 `StateKV map[string]any`(或同义结构)。
- [ ] `SessionStore` 暴露 `SetState(ctx, id, key, value)` / `GetState(ctx, id, key)`(或等价批量 API),允许在不改动 events 的前提下读写。
- [ ] StateKV 的写入是覆盖语义(后写覆盖先写),与 events 的 append-only 语义形成互补。

### 用户故事 4: 内存与文件系统两种后端互换

> 作为框架集成者,我希望测试用 in-memory store,生产用文件系统 store,且二者通过同一接口可换用。

**验收**:
- [ ] 实现 `MapSessionStore`(in-memory,默认,基于 sync.RWMutex)。
- [ ] 实现 `FileSessionStore`(文件系统;路径根可配置,默认 `~/.vv/sessions/`)。
- [ ] 文件后端按 `<root>/<session_id>/{meta.json, events.jsonl, state.json}` 落盘。
- [ ] 两个实现通过同一组接口 conformance 测试。

### 用户故事 5: hook 自动捕获(可选)

> 作为 vv 的接入者,我希望可以选择"一行注册" 把现有 `schema.Event` 流自动写入 Session,而不必修改每一处 emit 点。

**验收**:
- [ ] 提供 `session.NewSessionHook(store)`,实现 `vage/hook.AsyncHook`(或同等 sync hook)接口。
- [ ] 注册到 `hook.Manager` 后,任何带 `SessionID` 的 event 自动 append 到 store 中对应 session;`SessionID==""` 的事件忽略。
- [ ] hook 失败(store 异常、session 不存在等)非致命,记 `slog.Warn` 并 drop,不阻塞 agent 主路径。
- [ ] 事件 marshal/store 异常不可影响事件分发到其他 hook(与现有 tracelog 保持同等的弹性)。
- [ ] 默认**不**在 vv setup 中挂上,本次仅在 vage 层面提供构造函数。

### 用户故事 6: 并发安全

> 作为框架使用者,我希望可以从多个 goroutine 同时 append events、读 metadata,而不需要外层锁。

**验收**:
- [ ] `MapSessionStore` 在并发读写下不会 race(go test -race 通过)。
- [ ] `FileSessionStore` 对同一 session 的写入在进程内串行化;跨进程并发的协调本次明确不保证(留作 follow-up,文档说明)。

### 用户故事 7: 与现有 memory 系统互不破坏

> 作为已经在用 `memory.Manager` 的下游,我希望引入 Session 包不会迫使我修改任何现有调用。

**验收**:
- [ ] 不改 `vage/memory/` 任何已有公开 API。
- [ ] 不改 `schema.RunRequest.SessionID` 字段名/类型。
- [ ] `memory.Entry.SessionID` 保留;Session 实体的 ID 与之复用同一 string。
- [ ] vage 现有所有单元测试 / 集成测试维持通过。

## 3. 范围

### 3.1 In-scope

- 新增 `vage/session/` 包(单包,内部按文件拆分 ≤800 行/文件)。
- `session.Session` 结构、`SessionState` 枚举、`SessionStore` 接口。
- `MapSessionStore`(in-memory)。
- `FileSessionStore`(filesystem,JSON + JSONL)。
- `SessionHook`(实现 `hook.AsyncHook`,可选注册)。
- 单元测试:每个组件独立 + 一组 store conformance 测试。
- 模块文档 `vage/.doc/session.md`(简版,定位 + API + 与 memory 的关系)。

### 3.2 Out-of-scope

- Checkpoint / Snapshot / Restore。
- Context Builder、Plan/Scratchpad、Session Tree、向量召回、Context Editing。
- vv 侧 wiring(CLI 续接、HTTP CRUD、setup 默认挂载 SessionHook)。
- SQLite / Postgres / Redis 后端。
- 跨进程文件锁、ETag/版本号并发控制。
- 大文件 artifacts 存储(本次只放 metadata、events、state)。
- 事件压缩、归档、TTL 过期。
- 安全/权限:Session 内不再做单独鉴权(假设由调用方上层负责)。
- UI / observability dashboard。

## 4. 影响面

| 维度 | 影响 |
|---|---|
| 受影响 vage 包 | 新增 `vage/session/`;**不**改 `memory/`、`schema/`、`hook/`、`agent/` 公开 API |
| 受影响 vv 包 | 无改动(本次不 wire) |
| Roles | 不变 |
| Dictionaries | 新增 `Session State`、`Session Store Backend` 词条 |
| Models | 新增 `Session`(独立于 `Session Memory`,二者并存) |
| Procedures | 新增"Session 创建/读写/事件追加流程"(简版,无 vv 端) |
| Applications | 不变(vv 暂未消费) |

## 5. 假设与已识别问题

### 5.1 假设

- `schema.Event` 是稳定的事件载体;后续所有事件演化通过 `EventData` 接口扩展。
- `memory.Entry.SessionID` 与 `session.Session.ID` 共用同一字符串值,无歧义。
- Session 在 SDK 层是**进程内单例存储 + 可选持久化**的语义;跨进程并发场景由调用方负责。

### 5.2 设计 doc 中已存在的不一致(非本次解决)

- §4.1 草案中 `Checkpoints []Checkpoint` 与 §8 差距表中"未下沉到 TaskAgent 迭代级"暗示 checkpoint 的位置:本次按"差距表里的优先级"为准,checkpoint 不入 Session 实体。
- §4.1 提到"`memory.Session` = 由 `Session.StateKV` 投影而来的快取热数据"——这是一个**未来**目标,本次不实现两者间的投影同步,仅约定 ID 共用。

## 6. 验证手段

- 单元测试: 覆盖 Session 结构、MapStore 与 FileStore 的所有 CRUD/Append、SessionHook 三类。
- Conformance test: 共享一组黑盒测例,两个 store 实现都跑一遍以保证接口语义一致。
- 并发测试: `t.Parallel()` 多 goroutine 同时 append,go test -race 通过。
- Lint: `make lint` 与 `make format` 通过。

## 7. 非功能要求

- **零外部依赖**: 仅使用标准库与项目内已存在的依赖(`vage/schema`, `vage/hook`)。
- **文件单文件 ≤800 行**(项目约定)。
- **公开 API 注释**: 全部对外 API 必须有 GoDoc 注释。
- **License header**: 所有新增 .go 文件携带 Apache 2.0 license header(符合项目 license-check)。
