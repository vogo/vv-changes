# Session Tree —— 结构化要求

> 来源:`doc/design/session-context-solution.md` §4.8 / §8 差距汇总。
> 作为该文档差距汇总中**最后一项未交付能力**(P10),本期负责落地它的 MVP。

---

## 1. 背景与目标

### 1.1 业务背景

vage 已具备:

- 三层 memory(Working / Session / Store)
- ContextBuilder + 多个 Source(SystemPrompt / SessionMemory / SessionState / Workspace / VectorRecall / RequestMessages)
- Session 一等公民(meta + events + state KV)
- Plan Workspace(plan.md + notes/)
- 迭代级 Checkpoint
- Context Editing 中间件(折叠老 tool_result)

但**当任务跨越多日、多 session、多子目标时**,现有能力仍有短板:

1. **plan.md 是平铺文本**,LLM 只能"记账",无法清晰区分主目标 / 子目标 / 已完成事实 / 当前焦点;
2. **events 是事实流水**,LLM 无法快速"回到根",也无法折叠已完成子树;
3. **VectorRecall 提供相似性兜底**,但缺乏"结构感"——LLM 不知道每条命中相对于任务结构在哪里;
4. **Subagent 隔离**还未带 SessionView 快照,父子任务之间没有显式承载。

P10(层级树记忆)在文档中的核心命题:**当 session 走向"长 + 强目标"时,结构本身就是一种压缩**——树的形状即任务进度,路径即 context spine,兄弟与已完成子树折叠为父节点摘要。

### 1.2 本期目标

按 §4.8.6 的渐进路线,本期交付 **第 1 步:MVP 手工节点**(无 promotion、无双索引、无 LLM 工具暴露):

> "MVP:手工节点(无 promotion),仅作为 plan.md 的结构化版本"

具体:

1. **节点模型**:实现 §4.8.1 的 `TreeNode` / `SessionTree` 数据模型,持久化到磁盘。
2. **TreeStore**:CRUD + 树结构操作的最小后端接口 + 内存实现 + 文件实现。
3. **SessionTreeSource**:作为 Builder 的可插拔 Source,把 root → cursor 路径 + 兄弟标题 + 焦点子节点摘要按 §4.8.3 算法注入 prompt。
4. **事件**:节点创建 / 更新 / cursor 移动事件入 hook 体系,可被 SessionHook / tracelog 落盘审计。
5. **行为兼容**:不开启时(未注册 source),所有现有 ContextBuilder 用例字节级一致;现有单测全绿。

### 1.3 不在本期范围(明示 out-of-scope)

| 范围 | 决定 | 理由 |
|---|---|---|
| 自动 promotion / 反思 | ❌ 不实现 | §4.8.6 第 2 步;依赖低优 LLM 异步管线,接口先窄;手工节点先证明价值 |
| 双索引(summary 入向量库) | ❌ 不实现 | §4.8.6 第 3 步;依赖 promotion 的 stable summary;本期向量召回与树相互独立 |
| LLM 主动调用 zoom/promote/pin 工具 | ❌ 不实现 | §4.8.6 第 4 步;LLM 自治需要稳定的工具协议,先让人类/上层代码维护树 |
| 跨 session 树森林 | ❌ 不实现 | §4.8.6 第 5 步;依赖 SessionView 快照 + 父子 session 链 |
| vv 端 wiring(CLI / HTTP API) | ❌ 不实现 | 与 §4.4 / §4.5 的处理方式一致;vage 框架先齐全,vv 应用层下迭代再接 |
| TaskAgent 默认开启 SessionTreeSource | ❌ 不实现 | §4.8.5 启用条件:**默认不开启**——只在长任务场景手动 `WithExtraSources(&vctx.SessionTreeSource{...})` |
| Context 装箱算法的整体替换 | ❌ 不实现 | 复用现有 ordered_greedy 装箱;SessionTreeSource 像 WorkspaceSource 一样作为 optional source 注入 |

---

## 2. 用户故事 + 验收标准

### 故事 1:作为框架开发者,我可以用 SessionTreeStore 创建一棵会话任务树

**验收标准**:

- 可以创建 root goal 节点(NodeGoal),返回节点 ID;
- 可以在任意节点下追加子节点(NodeSubtask / NodeFact / NodeObservation / NodeArtifactRef);
- 可以读取整棵树(返回 `SessionTree` 含 RootID / Cursor / Nodes);
- 可以更新节点(Title / Summary / Status / Metadata);
- 可以移动 cursor 到某节点;
- 可以删除整棵树(随 SessionStore.Delete 联动)。

**不变量**:

- root 节点不可被父节点覆盖、不可被删除;
- 节点 ID 全局唯一,使用 `tn-<unix-nanos>-<8 hex>` 格式;
- 父子双向引用一致(`Children` 列表与每个 child 的 `Parent` 字段相互对应);
- 删除某节点时联动从父节点 `Children` 中移除;
- **不允许删除非叶节点**(防止意外丢失子树);需要时调用方先递归删子。

### 故事 2:作为框架开发者,我可以把 SessionTree 注入到 ContextBuilder

**验收标准**:

- `vctx.SessionTreeSource` 实现 `Source` 接口,**不**实现 `MustIncludeSource`(树是增强,不是前提);
- 配置:`Store`(必需)+ `MaxPathDepth`(默认 6)+ `MaxSiblingTitles`(默认 8)+ `MaxBytes`(默认 8 KiB);
- 渲染输出**单一 system 消息**,内容包含:
  - 顶部声明(像 WorkspaceSource);
  - root → cursor 路径上每个节点的 title + 状态 + summary(超 budget 时降级为 title-only);
  - cursor 节点的子节点 title 列表(导航视野);
  - 最近完成兄弟节点的 summary(LLM 看"刚才做了什么");
- 失败模式:nil store / empty session / 空树 → `Status="skipped"`,无错误返回(fail-open);
- 树读失败 → `Status="error"`,空消息 + 不阻断 Builder。

**不变量**:

- Source 永不返回 error(全部 fail-open);
- 输出消息字节数 ≤ MaxBytes(自截断);
- 路径节点过多时优先丢"远端"(深度小)的 summary,保留 cursor 周围。

### 故事 3:作为可观测性消费者,我可以从 hook 体系订阅树节点变化

**验收标准**:

- 创建节点 / 更新节点 / 移动 cursor 时通过 ctx.Emitter 发出 `EventSessionTreeUpdated` 事件;
- payload 含 `SessionID / Operation(create/update/cursor/delete)/ NodeID / NodeType / Status`;
- SessionHook 与 tracelog.JSONLHook 共享同一事件,无需特别处理;
- 写存储失败时不发事件(invariant: hook 计数 = 成功写次数)。

### 故事 4:作为运维人员,我可以从 FileTreeStore 在磁盘上看到树状态

**验收标准**:

- 目录布局 `<root>/<sessionID>/tree/tree.json`,与 `vage/session.FileSessionStore` / `workspace.FileWorkspace` 共享 root,SessionStore.Delete 联动清理;
- 单文件原子重写(atomic rename),与 workspace.writeFileAtomic 一致语义;
- 权限 0o700 / 0o600;
- 进程内并发安全(per-session mutex);跨进程不保证(与 workspace / session 一致)。

### 故事 5:作为开发者,我可以确认现有功能未受影响

**验收标准**:

- 现有 ContextBuilder / SessionMemorySource / WorkspaceSource / VectorRecallSource / RequestMessagesSource 单测全绿;
- TaskAgent 单测全绿;
- TaskAgent.Run 的输出 messages 顺序在不注册 SessionTreeSource 时**字节级**与现状一致。

---

## 3. 受影响的角色与模块

| 维度 | 范围 |
|---|---|
| 受影响模块 | 新增 `vage/session/tree/` + `vage/context/sources_tree.go`;`vage/schema/event.go` 新增事件常量 + payload 类型 |
| 不修改 | TaskAgent 主循环、memory 三层、largemodel 链、orchestrate、guard、tool 注册 |
| 测试 | unit-only(本期不引入 integration tests);新建 `tree_test.go` / `mapstore_test.go` / `filestore_test.go` / `sources_tree_test.go` |
| 文档 | 新增 `vage/.doc/session-tree.md`;更新 `doc/design/session-context-solution.md` §4.8 标记完成 + §8 差距汇总最后一行打钩 |

## 4. 已识别的待澄清点 + 默认决策

> 没有同步用户即时确认,以下默认决策记入设计文档,执行中遇到反例再回调。

| 待澄清点 | 默认决策 | 理由 |
|---|---|---|
| 节点 ID 生成策略 | 时间戳前缀 + 8 hex 后缀,与 `session.GenerateID` 同构 | 排序友好、跨进程足够唯一 |
| 渲染时是否包含 cursor 自身的 title? | **包含**:cursor 是路径终点,既出现在路径段也作为"焦点节"被展开 summary | 双重出现避免 LLM 在路径列表中找不到当前节点 |
| 兄弟节点列出时是否按状态分组? | **按 Status 分组**:active 在前,pending 居中,done/superseded 折叠为计数 | 给 LLM 清晰的"还能做什么"视野 |
| 节点 metadata 是否进 prompt? | **不进**:metadata 留给运维 / 反思逻辑,不污染 LLM context | 与 WorkspaceSource notes 索引"只露名字"一致 |
| Title / Summary 长度上限 | Title ≤ 200 字节(留给中文标题约 65-80 字),Summary ≤ 2 KiB | 与 §4.8.1 设想的"<= 80 字 / <= 400 字"接近,字节而非字符更严格 |
| 单棵树节点上限 | 1024 个 | 防止 prompt 爆炸的硬约束;超出时 Add 返回 `ErrTreeFull`,迫使调用方 promote 或 archive |
| ContentRef / EmbeddingID / Evidence / Supersedes / Pinned 字段 | **保留字段位**(struct 字段)但本期 Source 不渲染、Store 不解释 | 接口前置以避免后续 promotion / 双索引 / pin 落地时改 schema |

---

## 5. 与现有文档的内部一致性检查

读完相关 PRD 与设计文档,识别以下注意点(不阻断本期实施,仅记录):

- §4.8.1 节点字段 `Children []string` —— 与 §4.8.5 启用条件一致(只在长任务才用),保留单层数组;但在 root 节点 Children 数量较多时,渲染应限制 `MaxSiblingTitles`(已在故事 2 中明示)。
- §4.8.3 渲染算法假设有 `vector_recall(intent, scope=non_path_nodes, top_k=5)` 步骤 —— 本期 SessionTreeSource **不**做向量召回(向量召回由独立的 VectorRecallSource 承担);若用户希望"非路径节点的向量召回",可在 `VectorRecallSource.Predicate` 中按 `Metadata["node_id"] not in path` 过滤(详见 §4.9 vector.md)。这与 vector.md §7 "scope 由 SessionTree 调用方通过 Predicate 实现,不侵入接口"的承诺一致。
- §8 差距汇总 "Session Tree(P10) ❌ 无" —— 实施完成后改为 ✅ 并列出本期落地子集 + 后续迭代清单。
- `vage/.doc/index.md` 应新增 `session-tree.md` 引用(本期由 documenter 阶段处理)。
