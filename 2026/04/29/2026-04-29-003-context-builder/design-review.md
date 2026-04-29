# 设计评审：Context Builder 抽象

针对 `design-raw.md` 的逐项评审，按"接受 / 拒绝 / 微调"标注。原则是**简单优先**——只接受材料性收益的改动。

---

## 总体评价

草稿质量较高：
- Builder/Source 双层结构清晰；
- `MustIncludeSource` 类型断言扩展点比一刀切的 required flag 更灵活；
- 显式声明"Builder 不重排消息、顺序由声明决定"，把语义复杂度推给调用者（正确选择）；
- TaskAgent 集成路径保持外部签名不变；
- 复用 `memory.DefaultTokenEstimator` 避免引入新估算器。

下面是**少量值得调整**的点，以及**明确拒绝**几条会带来麻烦的复杂度。

---

## A. 接受的改动

### A1. 简化 `Budget` —— 去掉 `Reserve`，只留 `Total`

**问题：** v1 的 `Budget{Total, Reserve}` 中 `Reserve` 是"必须留给 thinking + tool_result 的下限"。但本次需求范围里：
- AC-3.4 验收只需要 prompt cache breakpoint 仍工作；
- 没有任何 source 真正消费 Reserve；
- TaskAgent 集成路径明确传 `Budget{}`（无限），Reserve 完全不参与。

引入两个字段意味着两个字段都要测、要文档化、要序列化（`ContextBudget` schema 也跟着多一个字段），但本次 zero usage。

**建议：** v1 改为单字段 `Budget int`（语义即"输出 token 上限，0 = 无限"）。后续真要 Reserve，新增 `BudgetSpec` 结构体或往 Builder option 里加 `WithReserve(n)`，向后兼容性不会被一个 int 卡住。

**收益：** 少一个字段、少一组 helper（`Available()`、`ContextBudget` 转换）、少一组测试组合、文档更短。

---

### A2. 去掉 `BuildInput.Session *session.Session` 字段 —— 只保留 `SessionID`

**问题：** `BuildInput` 同时携带 `Session *session.Session` 和 `SessionID string`，design 自己也承认"Session 有就直接用，没有也能跑"。

但实际上：
- `requirement.md` 末尾"与现有文档不一致点"指出 TaskAgent 当前**只有 `SessionID string`**；
- v1 内置 source（System/SessionMemory/SessionState/RequestMessages）**没有一个真的需要 `*session.Session` 对象**——SessionMemorySource 用 `memory.Manager`、SessionStateSource 用 `SessionStateStore`，都是按 ID 查；
- 双入口 = 两套校验逻辑（"Session 非空但 ID 不一致怎么办？"）+ 测试矩阵翻倍 + 给 source 作者潜在陷阱。

**建议：** v1 只接 `SessionID string`。若未来某个 source 真的需要预解析的 `*session.Session`（比如要拿 metadata 不想再查一次 store），那是开放 v2 的合适时机；那时再加 `WithSession(sess)` option 到 `BuildInput`，仍然向后兼容。

**收益：** 接口窄一档；删除 `Session *session.Session` 字段、`FetchInput.Session`、相关分支与测试。

---

### A3. `SessionMemorySource` 把"session message count"放进 `FetchReport.Note`，不要塞进 `InputN`

**问题：** design §6.2 让 TaskAgent 从 `FetchReport.InputN` 拿 `sessionMsgCount`，作为 storeAndPromoteMessages 的 idx offset。但 `InputN` 的语义在 §2.2 注释里写的是**"候选消息数"**——所有 source 通用。SessionMemorySource 用 InputN 装"压缩前数量"是可以，但其他 source 的"候选数"含义就会和它对不齐（比如 SessionStateSource 的"候选 key 数量"、RequestMessagesSource 的"输入消息数"），跨 source 比较时会困惑。

更糟的是：TaskAgent 集成代码必须**字符串匹配 `Source` 名字**（`"session_memory"`）才能从 `Sources []FetchReport` 里找到那条 entry——脆，且把 ordering 责任泄漏到调用方。

**建议：** 两选一，二者收益相当，design 阶段定其一：

- **方案 A（推荐）：** `FetchReport` 增加 `Extra map[string]any` 字段（或 `OriginalCount int`，更类型安全），SessionMemorySource 把"压缩前 entry 数"写进去；TaskAgent 通过类型断言从对应 source（用 `Sources[i].Source == ...` 匹配）拿。仍然有字符串匹配，但语义更明显。
- **方案 B（更简洁）：** `SessionMemorySource` 暴露一个**带状态的实例方法** `LastOriginalCount() int`，TaskAgent 持有 source 实例（DefaultBuilder 也持有），Build 完直接读。无字符串匹配，但 source 不再是无状态——有少量并发隐患（Builder 并发 Build 时该字段竞争），需要文档说明"SessionMemorySource 实例不可跨 Build 并发使用"。

**接受方案 A**：保持 source 无状态；用结构化 `OriginalCount int` 字段表达 source 自身需要回传的"压缩前条数"。

```go
type FetchReport struct {
    ...
    OriginalCount int `json:"original_count,omitempty"` // session 类 source 用：压缩前 message 条数
}
```

TaskAgent 从 `BuildReport.Sources` 中按 `Source == "session_memory"` 取 `OriginalCount`。是有耦合，但比 InputN 一字多义清晰。

**收益：** 字段语义清晰；其他 source 的 InputN 仍然表达"候选数"；schema event 也跟着加一个 `original_count` 字段。

---

### A4. `EventContextBuilt` 的 schema event data 用扁平结构、复用 `FetchReport`

**问题：** design §5 在 schema 包定义了 `ContextBuiltData` / `ContextBudget` / `ContextSourceReport` 三个新结构体，与 `vctx.BuildReport` / `Budget` / `FetchReport` 字段几乎一模一样，再加一个 `BuildReport.ToEventData()` 转换。

**理由是"避免 schema 反向依赖 vctx"**。这是对的——但**事件数据本身可以扁平展开 + 直接用 schema 自己的类型**，不需要"镜像"vctx。

具体：
- `ContextBudget` 跟着 A1 简化变成单字段 `Budget int`，直接 inline 进 `ContextBuiltData.BudgetTotal int`，不需要嵌套 struct。
- `ContextSourceReport` 与 `vctx.FetchReport` 字段相同；让 `vctx.FetchReport` **直接定义在 `vage/schema/`** 包下（取名 `schema.ContextSourceReport`），vctx 包反向使用 schema 类型。这样 `vctx.BuildReport.Sources` 就直接是 `[]schema.ContextSourceReport`，`ToEventData` 几乎是身份转换。

**建议：** 把 `FetchReport` 类型搬到 `schema/`（命名 `ContextSourceReport`，与事件数据共用）；`ContextBuiltData` 直接持有 `[]ContextSourceReport`；`vctx.BuildReport` 也用同一类型。`ToEventData` 退化为字段拷贝。

**收益：** 删除 `ContextBudget` 包装结构、`ContextSourceReport` 镜像类型；schema/vctx 共享同一报告类型；少 ~30 行代码。代价：`FetchReport`（重命名 `ContextSourceReport`）住在 schema 包——但这不是循环依赖（vctx → schema 是已有方向），且 schema 已经有 GuardCheckData / TodoUpdateData 这类业务结构，再加一个不破坏其纯度。

---

### A5. 明确 source 命名常量，避免字符串散落

**问题：** SessionMemorySource 的 `Name() string` 返回的字符串既是 hook 报告字段、又是 TaskAgent 识别 entry 的 key（A3 之后仍然要按名匹配）。design 没明示标准化命名。

**建议：** 在 `vage/context/source.go` 顶端定义：

```go
const (
    SourceNameSystemPrompt   = "system_prompt"
    SourceNameSessionMemory  = "session_memory"
    SourceNameSessionState   = "session_state"
    SourceNameRequestMessages = "request_messages"
)
```

**收益：** 单点真理；TaskAgent / 测试 / 文档都引用常量；防止 typo。

---

### A6. Skill 注入仍保留为 post-step，**显式说明且不引入 SkillSource**

**问题：** design §3 列出了 4 个内置 source，没有 SkillSource；§6.2 集成代码中 `injectSkillInstructions` 仍是 post-step。但 design 没解释为什么不做 SkillSource——读者会以为是遗漏。

**建议：** 在 design §3 末尾加一小节"**为什么不做 SkillSource（v1）**":

> Skill 指令注入需要**改写已有的 system message**（追加内容），而不是产出新 message——这是装箱模型不擅长的语义。Source 接口只能产出消息，不能编辑前面 source 的输出。强行做 SkillSource 会要求引入 "post-mutation hook" 或 "system message merger"，复杂度大。
>
> v1 保留 `injectSkillInstructions` 作为 Builder 之后的 post-step，与 prompt cache breakpoint 标记同处一个阶段。未来若 Source 接口扩展为"返回 patch 而非 messages"，再迁移。

**收益：** 把"看似缺失"显式化为"已知决策"；防止后续被误改。

---

### A7. 装箱算法明确：MustInclude **不参与 budget 余额扣减**？

**问题：** design §3.5 说"MustInclude 总是先入 prompt，**占用先记账，余下 budget 才分给可选 source**"。但 §2.3 的伪代码没有体现 mustInclude 优先；§4 BuildResult 里说"3. 依次执行 must-include source，累加 messages，记账 tokens(不限 budget) 4. 计算 optional sources 的总预算 = Available - mustIncludeTokens"。

两段描述基本一致，但 §2.3 的伪代码没改。**建议：** 把 §2.3 的伪代码改成两遍循环（先 must-include 不限 budget，再 optional 顺序贪心），与 §4 对齐。否则读者按 §2.3 实现就错了。

**收益：** 文档自洽，避免实现歧义。

---

## B. 拒绝的改动（讨论但不采纳）

### B1. ❌ 把 `MustIncludeSource` 替换为"Source 注册时的 Required flag"

考虑过，**拒绝**。理由：
- 类型断言扩展接口是 Go idiomatic（io.WriterTo / io.Closer 等），不增加 Source 接口的强制方法；
- 用 flag 意味着 `WithSource(s, WithRequired(true))`，调用者要记得标注；用 MustIncludeSource 接口让 source 自己声明"我天然不可丢"，与本身耦合，更不易出错；
- 系统提示和本轮请求 must-include 是**它们的本征属性**，不是"调用者的 policy 选择"。让 source 自己声明语义更对。
- "两层结构难理解"在实践中不成立：default Builder 用户根本不知道 MustIncludeSource 存在；只有写自定义 source 的人才接触它，那时一行接口很容易消化。

### B2. ❌ 引入 `Intent` 类型 / 枚举

requirement.md 与 design 都把 Intent 定为 `string`。已经够简单。引入类型 / 枚举 = 跨包导入 + iota 定义 + 校验路径。string 标签的 cardinality 完全可控（v1 就 1-2 个值）。**保持 string。**

### B3. ❌ Builder 暴露成 TaskAgent option（`WithContextBuilder(b)`）

design 的当前选择是"内部固定使用 DefaultBuilder + 现有字段构造"。这是对的。requirement §假设 §权衡也明确建议 A 但 design 选 B（不暴露），并理由"避免本次范围扩张"。**保留**——v1 不开放扩展点，等真有用例（例如 vv 的 dispatcher 想用不同 source 组合）再开放。

### B4. ❌ Token 估算引入 context 自己的实现

design 复用 `memory.DefaultTokenEstimator`。正确。**理由：** 所有 compressor 已经基于这套估算器，prompt 内 token 与 compressor 内 token 必须用同一套，否则"压缩后还超 budget"或"裁剪后明明还有空"等不一致 bug 会出现。**保留复用。**

### B5. ❌ 把 BuildReport 挂回 RunResponse

requirement §权衡已经讨论过。Hook 路径是对的。RunResponse 已经有 Usage/Duration，再挂 BuildReport 让接口臃肿且"每次 run 都要序列化一份 BuildReport"，hook 路径是 opt-in 观察者模式，更合适。**保持 hook。**

### B6. ❌ Source 失败语义改为统一 fail-closed

design 的差别化（SystemPromptSource fail-closed，其他 fail-open）是对的：
- 系统提示渲染失败 = agent 配置 bug，应该立刻让调用者知道；
- 历史压缩失败、状态投影失败 = 运行时偶发，不应阻断对话。

这种 fail-closed/open 区分非常符合"基础设施 vs 增量信息"的语义直觉。**保留。**

---

## C. 行文 / 文档微调（非结构性）

- §2.1 注释里写"Session/SessionID 二选一" → 改为"调用者只传 SessionID"（A2 之后）。
- §6.2 改造后伪代码里的 `buildResult{ messages, sessionMsgCount }` → 注明 `sessionMsgCount` 的取值来自 `BuildReport.Sources` 中 `SessionMemorySource` 的 `OriginalCount`（A3）。
- §4 的 `WithHookManager(m *hook.Manager)` option 文档明确"hook 失败 fail-open，不影响 Build 返回"（已在 §7 提到，但 option doc 也加一句）。

---

## D. 总结

接受的改动汇总（按影响排序）：

| # | 改动 | 文件影响 |
|---|---|---|
| A1 | Budget 简化为单字段 int | builder.go / source.go / report.go / event.go |
| A2 | 去掉 BuildInput.Session 字段 | builder.go / source.go / 集成 |
| A3 | 增加 FetchReport.OriginalCount | source.go / sources_session.go / 集成 |
| A4 | FetchReport 移到 schema 包，event data 复用 | schema/event.go / report.go |
| A5 | SourceName 常量 | source.go |
| A6 | 文档说明为何无 SkillSource | design §3 末尾 |
| A7 | §2.3 伪代码与 §4 对齐 | design §2.3 |

代码量净减少；语义更明显；外部接口更窄。

无重大重写——结构保持原样。
