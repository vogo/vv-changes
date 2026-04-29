# 需求:vv 端 Session wiring

## 1. 背景与目标

`vage/session` 包(2026-04-29-002 落地)与 `vage/context` 包(2026-04-29-003 落地)已经把 Session 提升为框架一等公民,并提供 `ContextBuilder` 替换 TaskAgent 内部的消息构造路径。但应用层(vv)目前还没有把这两个能力串起来:

- vv CLI 启动时随机生成一个 8 字节 session id,过程退出 = 数据丢失,无法续接对话。
- vv 的 HTTP 模式只暴露 `/v1/memory/*`,没有 Session 相关的 CRUD/检索/重放接口。
- `setup.Init` 没有构造 `SessionStore`,也没有把 `SessionHook` 挂到 `hook.Manager` —— 即便用户启用了 Trace,Session 实体仍然是空的,只有 `vv/traces/tracelog` 这条被动审计旁路在落盘。

本次目标:**让 vv 默认拥有一份"持久且可寻址"的会话实体,使 CLI 能挑选历史会话续接,HTTP 能列/查/删 session,并默认把 SessionHook 挂在 hook.Manager 上**。

产品级成功标准:

1. CLI 启动后默认仍然新建一个 session,但用户可通过 `--session <id>` 显式续接;启动时如果该 id 已经在 `SessionStore` 里则装载历史 events 重放出 `[]schema.Message`,作为新一轮 Run 的 `history`。
2. CLI 退出后再次启动,用户可通过 `--session list` 看到最近会话列表(id、title、updated_at、消息数);若不传 `--session`,默认行为不变(新建)。
3. HTTP 模式下 `/v1/sessions/*` 至少提供:`GET /v1/sessions`(列表+过滤)、`GET /v1/sessions/{id}`(meta)、`GET /v1/sessions/{id}/events`(原始事件)、`DELETE /v1/sessions/{id}`、`PATCH /v1/sessions/{id}`(改 title / metadata / state)。MVP 不含 `POST /v1/sessions`,因为 SessionHook autoCreate 已经覆盖隐式创建路径。
4. 用户没有显式禁用的前提下,`setup.Init` 默认创建 FileSessionStore(根目录 `~/.vv/sessions/`,可由 `cfg.Session.Dir` / `VV_SESSION_DIR` 覆盖),把 `SessionHook` 注册到 `hook.Manager`。原有 trace logger 与新 SessionHook **并存**,共用同一 `hook.Manager`。
5. 所有路径在 `cfg.Session.Enabled = false` 时完全不创建 store / 不挂 hook,**零开销路径保持不变**(等同于今天的行为)。

## 2. 用户故事 & 验收标准

### Story A:CLI 续接历史会话

> 作为一名长任务用户,我希望昨天没写完的会话今天还能从上次结束的地方继续。

**验收**:
- `vv --session <id>` 时:
  - id 存在 → 加载 meta + 全部 events,把含 `MessageRole=user/assistant` 的消息按时序拼成 `history`,屏幕打印"resuming session <id> (<n> messages)";
  - id 不存在 → 直接以该 id 新建 session,屏幕打印"starting session <id>";
  - id 非法 → 退到 stderr 报错并 exit 1。
- 不带 `--session` 时,行为与今天一致(用 `session.GenerateID()` 新生成),但 id 会写入 store(由 SessionHook autoCreate 完成)。
- `vv --session list` 列出最近 N 条 session(默认 N=20),按 `UpdatedAt` 倒序,展示:id、title、agent、events 数、updated_at(本地时区)。

### Story B:HTTP 暴露 Session CRUD

> 作为一名平台对接方,我希望通过 HTTP 拉取某个 session 的元数据 + 事件,做审计/回放/UI 渲染。

**验收**:
- `GET /v1/sessions?user_id=&agent_id=&state=&limit=&offset=` 返回 `{sessions: [{id, agent_id, user_id, title, state, created_at, updated_at}]}`,过滤参数组合时用 AND 语义;`limit` 默认 50、上限 200。
- `GET /v1/sessions/{id}` 返回完整 meta + state KV;404 当不存在。
- `GET /v1/sessions/{id}/events?type=&limit=` 返回原始 `[]schema.Event`(JSON 数组),`type` 过滤多值用逗号分隔;不指定 `limit` 默认 1000、上限 5000。
- `DELETE /v1/sessions/{id}` 删除 meta+events+state(`SessionStore.Delete` 幂等),返回 `{status: deleted}`。
- `PATCH /v1/sessions/{id}` 接受 `{title?, state?, metadata?}` 三个可选字段,服务端对修改字段做合并、保留未传字段、自动刷新 `UpdatedAt`。
- `Session` HTTP API 在 `cfg.Session.Enabled = false` 时不挂载,`/v1/sessions/*` 返回 404。

### Story C:setup 默认挂 SessionHook

> 作为一名首次体验 vv 的用户,我希望开箱即用就能看到自己的会话历史落地,而不需要去翻文档调配置。

**验收**:
- `cfg.Session.Enabled` 默认开启(`nil → true`);可通过 YAML / env(`VV_SESSION_ENABLED=false`)显式关闭。
- 关闭时:不创建 store、不注册 hook、HTTP `/v1/sessions/*` 不挂载,`vv --session` 报"session subsystem disabled"并 exit 1。
- 默认开启时:`setup.Init` 构造 `FileSessionStore` 于 `cfg.Session.Dir`(默认 `~/.vv/sessions`),把 `session.NewSessionHook(store)` 注册到 `hook.Manager` —— 与已有 tracelog hook 并列。
- 当 `cfg.Trace.Enabled = false` 但 Session 默认开启时,`hook.Manager` 也要被构造(今天只在 trace 启用时才构造;需要泛化)。
- `InitResult` 暴露 `SessionStore` 字段供 CLI/HTTP 消费;Shutdown 不再只关 trace,而是统一调度 Manager.Stop。

## 3. 范围

### In-scope

- `configs.SessionConfig`:`Enabled`(*bool)、`Dir`(string,默认 `~/.vv/sessions`)、`HistoryReplayMaxEvents`(int,默认 5000,防止历史会话超大)。
- `setup.Init` 构造 store + hook + 重构 `buildHookManager` 让它能装多个 hook。
- `cli.New` 接收 `SessionStore` + 一个可选的 `--session` 入参,`Run` 启动前完成续接逻辑。
- `httpapis.Serve` 接收 `SessionStore`,挂载 `/v1/sessions/*`(5 个端点)。
- `--session` flag(短句):`<id>` 续接、`list` 列出最近会话、`new`(显式开新)。
- 单元测试 + 集成测试(沿用 `vv/integrations/`)覆盖 CLI flag 解析、HTTP 端点、ResumeFromSession。
- 文档:`vv/CLAUDE.md` 增加 session 段、`doc/design/session-context-solution.md` §4.1 和 §8 把 vv 端 wiring 标记完成。

### Out-of-scope(显式)

- Checkpoint/Snapshot/Restore(P8 范畴,留给下一迭代)。
- 用 `Session.StateKV` 投影到 `memory.Session`(原 design 文档 §4.1 已声明 future)。
- Context Builder 暴露为 TaskAgent option(P2 已经默认接管 buildInitialMessages,本次不动)。
- SQLite/Postgres 后端、跨进程文件锁。
- Plan/Scratchpad 工作目录(P7 范畴)。
- HTTP `POST /v1/sessions`(显式创建)—— SessionHook autoCreate 已经覆盖,加了反而要再设计 id 冲突语义;若未来 HTTP 调用方需要"先 Create 后 Run",再加。
- 跨设备/多租户鉴权:HTTP 端点假设由本机/可信网络消费,沿用现有 `httpapis` 的"无鉴权"姿态;真正落地多租户时再加。

## 4. 涉及的角色 / 模型 / 应用

- **应用**:`vv` CLI 与 HTTP 服务。
- **模型**:`vage.Session`(已存在)、`vage.SessionStore`(已存在)、`schema.Event`(已存在)、`vv.configs.SessionConfig`(本次新增)。
- **角色**:终端用户(CLI)、平台对接方(HTTP)。

## 5. 假设与未决项(已在文档显式标注)

1. **Resume 时 events → messages 的还原**:`schema.Event.Data` 是接口,`session.FileStore.ListEvents` 已经 caveat 说反序列化的 Data 是 nil。我们在 vv 这层只关心 `EventTextDelta`(assistant) / 用户消息(实际上是 `EventAgentStart` 触发前外部塞进去的 `RunRequest.Messages`)。结论:**resume 不重放 LLM 内容**,而是从 `events.jsonl` 的 `MessageDelta`/`AgentEnd` 里拼出粗略 transcript,或者直接读 vv 已有的 `vv/traces/tracelog` 同 session 文件(行格式一致)。本次设计阶段会做最终决策,目前倾向于:**resume 不依赖 events.jsonl 的内容字段,只用 meta 显示、history 走 user 自己输入的延续**(即:resume = "复用 id + 显示历史标题+轮数",不重放 prompt,避免 schema.Event.Data 反序列化的复杂度)。
2. **--session 与 HTTP 同步语义**:CLI resume 拿到的 session 在写新事件时,SessionHook autoCreate 不会重复 Create(Hook 已对 ErrSessionExists 处理),但 UpdatedAt 不会自动刷新 —— 因为 hook 不写 meta。我们需要在 CLI / HTTP 写新一轮事件前显式 `Update(session)` 触摸一次时间戳。本次设计会决定该触摸放在哪一层(候选:CLI Run 启动时、SessionHook 内首事件触发时)。
3. **Session 默认开关**:design 文档暗示"默认开启",但和今天"trace 默认关"不一致。设计阶段需明确:Session 默认开 vs 默认关。倾向**默认开**(零外部网络成本、纯本地文件、和 trace 不冲突),允许 `VV_SESSION_ENABLED=false` 关闭。

## 6. 跨文档不一致点(留给 documenter 阶段处理)

- 现 design 文档 §8 对照表里 Session Tree 标"无",但 §4.8 也明确 P10 仅长任务场景启用,不冲突;本次只标完成的是 §4.1 的"vv 端 wiring"那行。
- vv/CLAUDE.md 当前没有 session 段落;documenter 阶段需要新增。
