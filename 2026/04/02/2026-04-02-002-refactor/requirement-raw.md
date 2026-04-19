# Dispatcher 重构需求：从刚性管道到自适应决策循环

## 一、问题描述

当前 `Dispatcher` 采用固定三阶段管道：`explore → classify → dispatch`。

**核心问题：**

1. **刚性流程浪费资源** — 每次请求都执行完整管道，即使"你好"这样的简单问候也要先 explore 项目上下文、再 classify、再 dispatch。
2. **explorer/planner 是固定阶段而非按需工具** — 简单任务不需要 explore，复杂任务可能需要多次 explore，当前架构无法适应。
3. **Plan 一旦生成不可变** — DAG 构建后按固定拓扑执行，无法根据中间步骤结果动态调整后续计划。
4. **主 agent 与 sub-agent 行为模式不统一** — Dispatcher 走管道逻辑，sub-agent 走 ReAct 循环，两套执行模型增加认知和维护成本。

## 二、目标架构

将 Dispatcher 从刚性管道重构为**统一的自适应决策循环**，主 agent 和 sub-agent 共享相同的主线流程：

```
意图识别 → 规划 Tasks → 执行 Tasks（含动态重规划）→ 执行总结
```

其中 explorer、planner 从"固定阶段"降级为"按需调用的能力"。

## 三、详细需求

### 需求 1：统一 Agent 主循环

**现状：** Dispatcher 走 explore→classify→dispatch 管道，TaskAgent 走 ReAct 循环，两套模型。

**目标：** 所有 agent（包括 Dispatcher 自身）共享统一的四阶段主循环：

```
┌─────────────────────────────────────────────┐
│  1. 意图识别 (Intent Recognition)            │
│     - LLM 判断用户意图                       │
│     - 简单意图直接进入执行，跳过规划          │
│     - 不明确时可调用 explorer 补充上下文      │
│                                              │
│  2. 规划 Tasks (Planning)                    │
│     - 简单任务：LLM 一次输出 intent + plan   │
│     - 复杂任务：调用 planner 生成执行计划     │
│     - 输出：有序的 Task 列表（含依赖关系）    │
│                                              │
│  3. 执行 Tasks (Execution)                   │
│     - 按计划顺序/并行执行各 Task              │
│     - 每个 Task 可由 sub-agent 承担           │
│     - sub-agent 内部递归使用同一主循环        │
│     - 支持执行中动态重规划（见需求 3）        │
│                                              │
│  4. 执行总结 (Summarization)                 │
│     - 汇总所有 Task 结果                     │
│     - 可选：由上层 agent 决定是否需要总结     │
└─────────────────────────────────────────────┘
```

**关键设计点：**
- 意图识别和规划可合并为一次 LLM 调用，避免简单任务多余的往返。
- 当 LLM 判断意图明确且任务简单时，直接跳到执行（单 Task，无需 DAG）。

**涉及文件：**
- `vv/dispatches/dispatch.go` — 重构 `Run()`/`RunStream()` 主流程
- `vv/dispatches/classify.go` — 重构为意图识别 + 规划的统一调用
- `vv/dispatches/explore.go` — 从固定阶段改为按需调用

---

### 需求 2：Explorer/Planner 降级为按需能力

**现状：** Explorer 是 `Dispatcher.Run()` 的第一个固定阶段，每次请求必经。Planner 是第二个固定阶段。

**目标：** Explorer 和 Planner 作为 agent 可调用的"内部工具"或"内部能力"，由 LLM 在意图识别/规划阶段按需触发。

**改造方案：**

```go
// Explorer 不再是固定阶段，而是 Dispatcher 的内部能力
// LLM 在意图识别阶段判断是否需要补充上下文
type DispatcherCapability interface {
    // Explore 按需获取项目上下文
    Explore(ctx context.Context, query string) (string, error)
    // Plan 按需生成执行计划
    Plan(ctx context.Context, intent string, context string) (*Plan, error)
}
```

**触发条件：**
- Explorer：LLM 判断需要了解项目结构/代码上下文时调用
- Planner：LLM 判断任务需要多步骤协作时调用

**涉及文件：**
- `vv/dispatches/explore.go` — 改为按需调用接口
- `vv/dispatches/classify.go` — 改为按需调用接口
- `vv/agents/explorer.go` — 保持 agent 实现不变，调用方式变化
- `vv/agents/planner.go` — 保持 agent 实现不变，调用方式变化

---

### 需求 3：执行中动态重规划

**现状：** Plan 生成后构建 DAG，DAG 拓扑固定，无法根据中间结果调整。

**目标：** 在 Task 执行过程中，支持根据上下文变化动态调整后续 Tasks。

**触发条件（克制原则）：**
- Task 执行**失败**（error 或明确的失败输出）
- Task 输出与预期**严重不符**（由 LLM 评估）
- **不要**每步都重新评估整个 Plan（成本过高）

**重规划机制：**

```go
type ReplanPolicy struct {
    // 仅在失败或输出异常时触发重规划
    TriggerOnFailure     bool  // Task 失败时触发
    TriggerOnDeviation   bool  // 输出偏离预期时触发（需 LLM 评估）
    MaxReplans           int   // 重规划次数上限，防止死循环，建议默认 2
}
```

**执行流程调整：**

```
执行 Task[i]
    ↓
检查结果 → 正常 → 继续执行 Task[i+1]
    ↓ 异常
触发重规划 → LLM 基于已完成 Tasks 和当前异常重新生成后续 Tasks
    ↓
替换剩余 Tasks → 继续执行
    ↓ 超过 MaxReplans
中止并返回部分结果 + 错误说明
```

**涉及文件：**
- `vv/dispatches/dag.go` — DAG 执行增加重规划钩子
- `vv/dispatches/types.go` — 新增 `ReplanPolicy` 类型
- `vv/dispatches/dispatch.go` — 主循环集成重规划逻辑

---

### 需求 4：Sub-Agent 递归深度控制

**现状：** Sub-agent（如 coder、researcher）作为 TaskAgent 执行时无递归深度概念。

**目标：** 当 sub-agent 也能触发 sub-sub-agent 时（统一主循环后的自然结果），需限制递归深度。

**设计：**

```go
// 通过 context 传递当前递归深度
type depthKey struct{}

func WithDepth(ctx context.Context, depth int) context.Context {
    return context.WithValue(ctx, depthKey{}, depth)
}

func GetDepth(ctx context.Context) int {
    if v, ok := ctx.Value(depthKey{}).(int); ok {
        return v
    }
    return 0
}
```

**行为规则：**
- `depth == 0`：主 Dispatcher，完整主循环（意图→规划→执行→总结）
- `depth == 1`：一级 sub-agent，可规划但不再拆分子任务
- `depth >= maxDepth`：叶子节点，直接执行（跳过规划，走简化 ReAct 循环）
- 建议 `maxDepth` 默认值为 **2**

**涉及文件：**
- `vv/dispatches/dispatch.go` — 主循环根据 depth 选择完整/简化流程
- `vv/dispatches/dag.go` — 创建 sub-agent 时递增 depth

---

### 需求 5：总结阶段可选化

**现状：** Plan 模式下固定有 `planGen` agent 做最终总结（`dag.go` 中的 summary node）。

**目标：** 总结行为由上层 agent 按需决定，而非默认执行。

**规则：**
- **CLI 流式输出**：用户已看到过程，默认**不**生成总结
- **HTTP API 调用**：默认生成总结（调用方可能只看最终结果）
- **Sub-agent 向上汇报**：由父 agent 决定是否要求子 agent 总结
- 支持通过配置项控制：`WithSummaryPolicy(policy)`

```go
type SummaryPolicy int

const (
    SummaryAuto   SummaryPolicy = iota // 默认：HTTP 生成，CLI 不生成
    SummaryAlways                       // 强制生成
    SummaryNever                        // 不生成
)
```

**涉及文件：**
- `vv/dispatches/dag.go` — summary node 生成逻辑条件化
- `vv/dispatches/dispatch.go` — 新增 `SummaryPolicy` 配置

---

## 四、改造优先级与依赖关系

```
需求 1（统一主循环）     ← 基础，其他需求依赖此项
    ↓
需求 2（按需能力）       ← 依赖需求 1 的主循环框架
    ↓
需求 3（动态重规划）     ← 依赖需求 1 的执行阶段
需求 4（递归深度控制）   ← 依赖需求 1 的统一模型
需求 5（总结可选化）     ← 独立，可并行实施
```

**建议实施顺序：** 1 → 2 → (3 + 4 + 5 并行)

## 五、改造范围总结

| 模块 | 变更程度 | 说明 |
|------|---------|------|
| `vv/dispatches/dispatch.go` | **重构** | 主流程从管道改为决策循环 |
| `vv/dispatches/classify.go` | **重构** | 合并意图识别与规划 |
| `vv/dispatches/explore.go` | **重构** | 从固定阶段改为按需调用 |
| `vv/dispatches/dag.go` | **修改** | 增加重规划钩子、depth 传递、总结条件化 |
| `vv/dispatches/types.go` | **修改** | 新增 ReplanPolicy、SummaryPolicy |
| `vv/dispatches/stream.go` | **修改** | 适配新的阶段事件模型 |
| `vv/dispatches/input.go` | **小改** | 适配新的输入构造方式 |
| `vv/dispatches/helpers.go` | **小改** | 适配新的辅助逻辑 |
| `vv/agents/*.go` | **不变** | Agent 实现保持不变，调用方式变化 |
| `vage/agent/` | **不变** | 框架层无需改动 |

## 六、风险与注意事项

1. **意图识别准确性** — 合并 intent+plan 为一次 LLM 调用时，prompt 设计要确保 LLM 能可靠区分"直接回答"和"需要规划"。建议提供明确的输出 schema。
2. **重规划成本** — 每次重规划至少一次 LLM 调用。MaxReplans=2 意味着最坏情况多 2 次 LLM 往返，需在 token budget 中预留。
3. **流式体验连续性** — 从固定三阶段事件改为动态阶段事件后，CLI 的渲染逻辑需同步适配。
4. **向后兼容** — HTTP API 的响应格式（phase events）会变化，需考虑客户端适配。
5. **测试覆盖** — 重构涉及核心调度逻辑，需补充：简单意图直达、复杂任务规划、重规划触发、递归深度限制、总结策略等场景的集成测试。
