# 设计：Context Builder 抽象

> 本设计在 `design-raw.md` 基础上经评审（见 `design-review.md`）后做了 7 处针对性收紧：单字段 Budget、去除 Session 双入口、`OriginalCount` 字段、报告类型搬到 schema 包、source 名称常量、明确"为何无 SkillSource"、`§2.3` 伪代码与 `§4` 一致化。

## 1. 总览

新建 `vage/context/` 包（包名 `vctx`，避免与标准库 `context` 冲突），提供一组接口与 DefaultBuilder 的实现：

```
vage/context/
├── builder.go          # Builder 接口 + DefaultBuilder
├── builder_test.go
├── source.go           # Source 接口 + Budget + 内置 source 通用工具 + Source 名常量
├── source_test.go
├── sources_system.go   # SystemPromptSource
├── sources_session.go  # SessionMemorySource (复刻当前行为)
├── sources_state.go    # SessionStateSource (示例的非平凡 source)
├── sources_request.go  # RequestMessagesSource (本轮 RunRequest 消息)
├── report.go           # BuildReport (内嵌 schema.ContextSourceReport)
└── report_test.go
```

加上：
- `vage/schema/event.go`：新增 `EventContextBuilt` 常量、`ContextSourceReport`（共享类型）与 `ContextBuiltData`。
- `vage/agent/taskagent/task.go`：`buildInitialMessages` 改写为构造并调用 `vctx.Builder`，保持外部行为兼容。

## 2. 核心接口

### 2.1 Builder

```go
// Builder 接收 BuildInput,装配出送 LLM 的 message 序列与 BuildReport。
type Builder interface {
    Build(ctx context.Context, in BuildInput) (BuildResult, error)
    Name() string
}

// BuildInput 是 Builder 的输入。调用者只传"这一轮跟 LLM 对话的素材";
// Sources 由 Builder 内部固定。
type BuildInput struct {
    SessionID string             // 必传
    AgentID   string
    Intent    string             // 可选标签,如 "react-iter" / "explore" / "plan"
    Request   *schema.RunRequest // 本轮请求(包含本轮 user message)
    Budget    int                // 0 = 无限;> 0 表示输出 token 上限
    Vars      map[string]any     // 透传给 system prompt 模板
}

// BuildResult 是 Builder 的输出。
type BuildResult struct {
    Messages []aimodel.Message
    Report   BuildReport
}
```

设计要点（与原稿差异已评审）：
- **只接 `SessionID`**：v1 内置 source 都按 ID 查询，不需要预解析的 `*session.Session` 对象。若未来某个 source 需要 metadata，再加 `WithSession(sess)` option，向后兼容。
- **`Budget` 退化为单 `int`**：v1 没有 source 真正消费 `Reserve`，单字段语义更直接。后续若需要预留区，扩成 `BudgetSpec` 结构体即可。
- **不暴露 source 列表**：Builder 在构造时固定 sources；调用者只调 `Build`。
- **Vars** 让 PromptTemplate 渲染时拿到外部变量，与现有 `prompt.PromptTemplate.Render(ctx, vars)` 接口对齐。

### 2.2 Source

```go
// Source 是 Builder 的可插拔插件。每个 Source 负责拉取 / 生成一段 messages。
type Source interface {
    Name() string
    Fetch(ctx context.Context, in FetchInput) (FetchResult, error)
}

// FetchInput 是 Source 的输入。
type FetchInput struct {
    SessionID string
    AgentID   string
    Intent    string
    Request   *schema.RunRequest
    Budget    int            // 当前 source 可用的 token budget;0 = 无限
    Vars      map[string]any
}

// FetchResult 是 Source 的输出。
type FetchResult struct {
    Messages []aimodel.Message
    Report   schema.ContextSourceReport
}
```

`schema.ContextSourceReport` 在 schema 包中定义（见 §5），vctx 直接复用——避免镜像类型与 ToEventData 体力活。

### 2.3 Budget & 装箱算法

```go
// Budget 是 BuildInput.Budget 的同义类型;0 表示无限。
//
// v1 仅一个 Total 概念。如未来需要"必须留给 thinking + tool_result 的下限",
// 通过新增 BudgetSpec 结构体扩展,不破坏现有 API。
```

**装箱算法（顺序贪心，先 must-include 后 optional）：**

```
// 第一轮: must-include sources 不限 budget,先记账。
mustIncludeTokens := 0
for src in mustIncludeSources:
    res := src.Fetch(in.WithBudget(0))    // 0 = 不限
    append(out, res.Messages)
    mustIncludeTokens += res.Tokens

// 第二轮: optional sources 顺序贪心。
remaining := 0
if budget > 0:
    remaining = budget - mustIncludeTokens
    if remaining < 0: remaining = 0     // 已超,后续 source 无 budget

for src in optionalSources:
    sub := remaining   // 0 = 无限(若 budget 本身 = 0)
    res := src.Fetch(in.WithBudget(sub))
    if budget > 0 && res.Tokens > sub:
        // Source 没自己裁,Builder 在末端兜底:从老到新丢
        res = trimByTokens(res, sub)
    append(out, res.Messages)
    if budget > 0:
        remaining -= res.Tokens
        if remaining < 0: remaining = 0

// 最终输出顺序: 按 sources 声明顺序拼接,Builder 不重排。
```

设计要点：
- **MustInclude 优先**：系统提示与本轮请求**永远进 prompt**，先记账，余下的才给 optional source 贪心。
- **顺序优先级 = 配置顺序**；Builder 不重排。第一个 optional source 拿满预算的能力最强。
- **Source 自己截断优先**（知道哪些消息可丢）；否则 Builder 兜底。
- 兜底裁剪策略简单：从前往后丢（优先丢老的），保留 message 边界（不在 message 中间切）。
- **不做比例分配**（out-of-scope）；如果未来要，新增 `BudgetAllocator` 接口，DefaultBuilder 默认沿用顺序贪心。

### 2.4 Token 估算

复用 `memory.EstimateTextTokens` / `memory.DefaultTokenEstimator`，不引入新估算器。封装一个内部 helper：

```go
// estimateMessageTokens 返回 aimodel.Message 的近似 token 数。
// 复用 memory 包的估算器(text/4 启发式)。
func estimateMessageTokens(m aimodel.Message) int { ... }
```

### 2.5 Source 名称常量

为避免字符串散落（hook 数据匹配、TaskAgent 取 OriginalCount、测试断言都需要稳定 key），在 `source.go` 顶端定义：

```go
const (
    SourceNameSystemPrompt     = "system_prompt"
    SourceNameSessionMemory    = "session_memory"
    SourceNameSessionState     = "session_state"
    SourceNameRequestMessages  = "request_messages"
)
```

各内置 source 的 `Name()` 返回对应常量；外部 source 自己起名。

## 3. 内置 Source 实现

### 3.1 SystemPromptSource

```go
type SystemPromptSource struct {
    Template prompt.PromptTemplate
}
```

行为：
- Template 为 nil → status="skipped"。
- Render 失败 → 返回 error（系统提示是基础设施层，**不能** fail-open；详见 §7）。
- Render 出空字符串 → status="skipped"，不输出 message。
- 输出一条 `Role: aimodel.RoleSystem` 消息。
- 实现 `MustInclude() bool { return true }` —— 不参与 budget 抢占。

### 3.2 SessionMemorySource

复刻当前 `loadAndCompressSessionHistory` 的行为：
- 从 `memory.Manager.Session()` 读取 `msg:` 前缀的所有 entry。
- 按 key 字典序排序。
- 应用 `memory.Manager.Compressor()`（若有）。
- 输出 `[]aimodel.Message`（转换 `schema.Message` → `aimodel.Message`）。
- **`FetchReport.OriginalCount` 填"压缩前的 entry 数"**——TaskAgent 用作 idx offset（详见 §6）。

```go
type SessionMemorySource struct {
    Manager *memory.Manager
    Prefix  string // 默认 "msg:"
}
```

错误处理：
- Manager / Session() 为 nil → status="skipped"。
- List 失败 → fail-open（slog.Warn + 返回空，与现有行为一致）；report 标 status="error"，Error 填错误信息。
- Compressor 失败 → fail-open（slog.Warn + 用未压缩版本）。

### 3.3 SessionStateSource（示例非平凡 source）

把 `SessionStateStore` 中的若干指定 key 投影成一段 system 消息附加到 prompt：

```go
type SessionStateSource struct {
    Store  session.SessionStateStore
    Keys   []string                    // 想读取的 key 列表;空则不读
    Render func(map[string]any) string // 渲染函数;nil 用默认 KV 列表
}
```

行为：
- Store/Keys 为空 → status="skipped"。
- 对每个 key 调 GetState；不存在的 key 跳过（不报错）。
- 用 Render 把 KV 渲染成一段文本，包成 `Role: aimodel.RoleSystem` 消息。
- 任何 GetState 失败 → fail-open（slog.Warn + 跳过该 key）；全部 key 都失败 → status="error"。

这是为了证明 Source 接口能接 `session.SessionStateStore`——为 §4.4 Plan 工作区、§4.8 Session Tree 等未来 source 探路。

### 3.4 RequestMessagesSource

把 `BuildInput.Request.Messages` 直接作为 messages 附加。这是必须有的最后一步——本轮用户输入。

```go
type RequestMessagesSource struct{}
```

行为：
- Request 为 nil 或 Messages 为空 → status="skipped"。
- 输出 `schema.ToAIModelMessages(Request.Messages)`。
- 实现 `MustInclude() bool { return true }` —— 本轮请求一旦丢，LLM 就完全不知道用户问了什么。

### 3.5 "必带" Source 的处理

System prompt 与 RequestMessages 是不能丢的（否则 LLM 完全无法工作）。

约定：`Source` 接口外，定义可选扩展接口 `MustIncludeSource`，由内置 source 通过类型断言被 Builder 识别：

```go
// MustIncludeSource 是 Source 的可选扩展接口。返回 true 表示该 source 的输出
// 永远进 prompt,Builder 装箱时跳过 budget 抢占,先入账再给 optional source。
type MustIncludeSource interface {
    MustInclude() bool
}
```

`SystemPromptSource` 与 `RequestMessagesSource` 实现 `MustInclude() bool { return true }`；其它默认 false。

裁剪策略：budget 兜底裁剪只针对**非 MustInclude source 的累计输出**；MustInclude 总是先入 prompt，占用先记账，余下 budget 才分给可选 source（与 §2.3 算法一致）。

### 3.6 为什么 v1 不做 SkillSource

Skill 指令注入需要**改写已有的 system message**（追加内容），而不是产出新 message——这是装箱模型不擅长的语义。Source 接口只能产出消息，不能编辑前面 source 的输出。强行做 SkillSource 会要求引入"post-mutation hook"或"system message merger"，复杂度大。

v1 保留 `injectSkillInstructions` 作为 Builder 之后的 post-step（与 prompt cache breakpoint 标记同处一个阶段）。未来若 Source 接口扩展为"返回 patch 而非 messages"，再迁移。

## 4. DefaultBuilder

```go
type DefaultBuilder struct {
    name       string
    sources    []Source
    estimator  memory.TokenEstimator // 默认 DefaultTokenEstimator
    hooks      *hook.Manager         // 用于发 EventContextBuilt;可 nil
}

type Option func(*DefaultBuilder)

func WithSource(s Source) Option { ... }
func WithSources(s ...Source) Option { ... }
func WithName(n string) Option { ... }
func WithTokenEstimator(e memory.TokenEstimator) Option { ... }
func WithHookManager(m *hook.Manager) Option { ... } // hook 失败 fail-open,不影响 Build 返回

func NewDefaultBuilder(opts ...Option) *DefaultBuilder { ... }
```

`Build` 执行流程：

```go
func (b *DefaultBuilder) Build(ctx, in) (BuildResult, error) {
    1. 检查 ctx.Err()
    2. 把 sources 分两组:must-include / optional
    3. 依次执行 must-include source,累加 messages,记账 tokens(不限 budget)
    4. 计算 optional sources 的剩余预算 = max(0, in.Budget - mustIncludeTokens)
       如果 in.Budget == 0,剩余视为无限。
    5. 顺序贪心分配 optional sources,逐个 Fetch + 兜底裁剪
    6. 拼接最终 messages: 按 sources 声明顺序拼接,Builder 不重排。
       (SystemPromptSource 输出 system role,自然在前;SessionMemorySource 输出历史;
        RequestMessagesSource 输出本轮)
    7. 构造 BuildReport,通过 hook 发 EventContextBuilt。
    8. 返回 BuildResult。
}
```

**关键决策：Builder 不重排消息**

调用者声明 source 时按"系统 → 历史 → 本轮"顺序声明，输出顺序就对了。Builder 的职责是装箱与审计，不假定语义顺序——这样未来加 `PlanSource`（注入到 system 之后、历史之前）等都靠声明顺序解决。

## 5. BuildReport 与事件

`schema.ContextSourceReport` 既是 Source.Fetch 的报告项，也是事件数据中的 source 列表元素——同一个类型，避免镜像。

```go
// vage/schema/event.go 增加:

const EventContextBuilt = "context_built"

// ContextSourceReport 描述单个 Source.Fetch 的执行情况。
// vage/context 包直接复用此类型作为 FetchReport。
type ContextSourceReport struct {
    Source        string `json:"source"`
    Status        string `json:"status"`                   // "ok" | "skipped" | "error" | "truncated"
    InputN        int    `json:"input_n,omitempty"`         // 候选消息/数据条数
    OutputN       int    `json:"output_n"`                  // 实际输出消息数
    DroppedN      int    `json:"dropped_n,omitempty"`       // 被丢弃数
    Tokens        int    `json:"tokens"`                    // 输出消息估算 token 数
    OriginalCount int    `json:"original_count,omitempty"`  // session 类 source 用:压缩前 message 条数
    Note          string `json:"note,omitempty"`            // 简短说明
    Error         string `json:"error,omitempty"`
}

// ContextBuiltData 是 EventContextBuilt 的事件 payload。
type ContextBuiltData struct {
    Builder      string                `json:"builder"`
    Strategy     string                `json:"strategy"`     // 当前固定 "ordered_greedy"
    BudgetTotal  int                   `json:"budget_total"` // 0 = unlimited
    OutputCount  int                   `json:"output_count"`
    OutputTokens int                   `json:"output_tokens"`
    DroppedCount int                   `json:"dropped_count"`
    Sources      []ContextSourceReport `json:"sources"`
    Duration     int64                 `json:"duration_ms"`
}

func (ContextBuiltData) eventData() {}
```

```go
// vage/context/report.go:

type BuildReport struct {
    BuilderName  string                       `json:"builder"`
    Strategy     string                       `json:"strategy"`     // 固定 "ordered_greedy"
    InputBudget  int                          `json:"input_budget"` // 0 = unlimited
    OutputCount  int                          `json:"output_count"`
    OutputTokens int                          `json:"output_tokens"`
    DroppedCount int                          `json:"dropped_count"`
    Sources      []schema.ContextSourceReport `json:"sources"`
    Duration     int64                        `json:"duration_ms"`
}

// ToEventData 转成事件 payload。字段几乎全是直接拷贝。
func (r BuildReport) ToEventData() schema.ContextBuiltData {
    return schema.ContextBuiltData{
        Builder:      r.BuilderName,
        Strategy:     r.Strategy,
        BudgetTotal:  r.InputBudget,
        OutputCount:  r.OutputCount,
        OutputTokens: r.OutputTokens,
        DroppedCount: r.DroppedCount,
        Sources:      r.Sources, // 同一类型,直接共用
        Duration:     r.Duration,
    }
}
```

**为什么 schema 持有 ContextSourceReport？** schema 包已经容纳 GuardCheckData / TodoUpdateData 等业务结构（不是纯通信原语），再加一个 ContextSourceReport 不破坏其纯度。换来的是 vctx → schema 单向依赖 + 类型唯一不镜像。

## 6. TaskAgent 集成

### 6.1 当前调用栈

```
Run / RunStream
  → buildInitialMessages
    → loadAndCompressSessionHistory
      → loadSessionMessages (memory.Session().List)
      → memory.Compressor().Compress
    → systemPrompt.Render
    → 拼接 [system, sessionMsgs, reqMsgs]
  → injectSkillInstructions
  → markPromptCacheBreakpoints
  ...
```

### 6.2 改造后

```
Run / RunStream
  → buildInitialMessages
    → vctx.NewDefaultBuilder(
         WithSource(&SystemPromptSource{Template: a.systemPrompt}),
         WithSource(&SessionMemorySource{Manager: a.memoryManager, Prefix: "msg:"}),
         WithSource(&RequestMessagesSource{}),
         WithHookManager(a.hookManager))
       .Build(ctx, BuildInput{
         SessionID: req.SessionID,
         AgentID:   a.ID(),
         Intent:    "react-iter",
         Request:   req,
         Budget:    0, // 当前 TaskAgent 不传 budget(无限),保持兼容
       })
    → buildResult{ messages, sessionMsgCount }
  → injectSkillInstructions    // 不变
  → markPromptCacheBreakpoints // 不变
```

`sessionMsgCount` 从 `BuildReport.Sources` 中取——按 `Source == vctx.SourceNameSessionMemory` 匹配，读 `OriginalCount` 字段。helper：

```go
func sessionMsgCountFromReport(r vctx.BuildReport) int {
    for _, s := range r.Sources {
        if s.Source == vctx.SourceNameSessionMemory {
            return s.OriginalCount
        }
    }
    return 0
}
```

字段名 `OriginalCount` 比 `InputN` 语义清晰（InputN 在 SessionStateSource 表达"候选 key 数"、在 RequestMessagesSource 表达"输入消息数"，跨 source 不能比较）。

### 6.3 兼容性保证

- `buildResult.messages` 输出顺序与原实现完全一致（verified via 现有测试）。
- `sessionMsgCount` 等价于"原 List 返回的 entry 数量"。
- 错误处理与原代码一致：source 内部 fail-open + slog.Warn，Builder 不返 error 给 caller（除 SystemPromptSource Render 失败这一原本就 error 的路径）。

### 6.4 不暴露为 TaskAgent option（本次）

Builder 在 TaskAgent 内部固定使用 DefaultBuilder + 现有字段构造的 sources。不对外暴露 `WithContextBuilder` option——避免本次需求范围扩张。后续迭代再开放。

## 7. 错误处理策略汇总

| 失败点 | 策略 |
|---|---|
| Builder.Build 内部 ctx 取消 | 立即 return error |
| Source.Fetch 返回 error | fail-open:slog.Warn,该 source status="error",不阻断后续 source |
| SystemPromptSource Render 失败 | fail-closed:return error。系统提示是基础设施 |
| Token 估算 panic（理论不会） | recover + fallback 0 |
| Hook dispatch 失败 | 已经是 fail-open 设计（hook 体系本身不抛错） |

差别化（SystemPromptSource fail-closed，其他 fail-open）符合"基础设施 vs 增量信息"语义直觉：系统提示渲染失败 = agent 配置 bug，立刻让调用者知道；历史压缩失败、状态投影失败 = 运行时偶发，不阻断对话。

## 8. 单元测试计划

| 测试 | 验证 |
|---|---|
| TestDefaultBuilder_BasicCompose | 三个 source 按顺序拼接成预期 messages |
| TestDefaultBuilder_BudgetTrim | 给定预算,oldest message 被丢弃 |
| TestDefaultBuilder_MustIncludeNotTrimmed | budget 很小时 system+request 不被裁 |
| TestDefaultBuilder_MustIncludeAccountedFirst | must-include token 先扣减,optional 拿剩余 |
| TestDefaultBuilder_SourceErrorFailOpen | 中间 source 报错,后续 source 仍执行 |
| TestDefaultBuilder_EmitsEvent | hook 收到 EventContextBuilt,data 字段非空 |
| TestSystemPromptSource_RenderError | Render 失败 return error |
| TestSystemPromptSource_EmptyTemplate | Skip + 0 messages |
| TestSessionMemorySource_LoadAndCompress | 等价于现有 loadAndCompressSessionHistory |
| TestSessionMemorySource_OriginalCount | 压缩前条数正确填入 OriginalCount |
| TestSessionMemorySource_NoManager | Skip |
| TestSessionStateSource_RenderKeys | 多 key 按顺序拼成 system msg |
| TestRequestMessagesSource_Empty | Skip |
| TestBuildReport_JSON | 序列化无 unsupported field |
| TestBuildReport_ToEventData | 转换字段一一对应 |

TaskAgent 集成测试：
- 现有 task_test.go / task_parallel_test.go / 等所有测试通过（无回归）。

## 9. 整合测试计划（供 tester 阶段参考）

由于本次完全是内部抽象重构，**没有新增用户可见行为**。Acceptance 验证集中在：

1. **行为兼容**：vage/integrations/taskagent_tests/ 全套通过。
2. **新事件**：挂一个 hook，验证 `EventContextBuilt` 触发，data 类型断言为 `schema.ContextBuiltData`，字段非空。
3. **可插拔**：写一个最小集成测试，自定义 Source 输出固定 message，验证它进了最终 prompt。

无需新增大量集成 case；现有 taskagent 行为测试是核心回归网。

## 10. 风险与权衡

| 风险 | 缓解 |
|---|---|
| 改写 `buildInitialMessages` 引入回归 | 写 BuildReport 等价性测试;保留旧函数私有,Builder 失败时退化(不做)——直接用 Builder,行为兼容靠测试覆盖 |
| Token 估算不准 → 误裁 | 复用 memory 包估算器,与现有 compressor 逻辑等价;裁剪只在显式传 budget 时启用,默认无限不影响兼容性 |
| schema 反向依赖? | vctx → schema 单向;ContextSourceReport / ContextBuiltData 是普通 struct,vctx 复用而非镜像;无循环依赖 |
| Source 顺序与 prompt 顺序绑定 → 用户配错 | 文档明示约定;DefaultBuilder 提供 NewTaskAgentBuilder 工厂,内部固定正确顺序 |
| TaskAgent 按 SourceName 匹配 OriginalCount → 脆 | 用 `vctx.SourceNameSessionMemory` 常量,单点真理;source 重命名只改一处 |

## 11. 实施步骤

1. `vage/schema/event.go`：加 `EventContextBuilt`、`ContextSourceReport`、`ContextBuiltData`。
2. `vage/context/`：新建包，按 §1 文件清单实现；引入 `SourceName*` 常量。
3. `vage/agent/taskagent/task.go`：替换 `buildInitialMessages`；从 BuildReport.Sources 中取 `OriginalCount` 作为 sessionMsgCount。`loadSessionMessages` 移到 SessionMemorySource 中（保留 helper 兼容——已确认无外部调用）。
4. 跑全套测试 + lint。
5. `vage/.doc/context.md`：写包文档。
6. `doc/design/session-context-solution.md`：§4.2 与 §8 标完成。
