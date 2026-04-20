# 需求规格：P1-2 vv 集成评估/质量指标（Eval）

## 1. 背景与目标

### 1.1 背景

vage 框架层（`vage/eval/`）已经完整实现了 6 类评估器（ExactMatch、Contains、LLMJudge、ToolCall、Latency、Cost）、批量评估 `BatchEval`、一体化执行 `RunAndEvaluate` 与组合加权 `CompositeEvaluator`。但 vv 应用层目前尚未接入这些能力——用户没有办法对 vv 的 Dispatcher/子 Agent 做离线质量回归，也无法在 HTTP 服务上获取任一条请求的质量指标。

`doc/prd/overview.md` 第 75 行明确把"Evaluation and quality metrics"列为 `Does Not Cover (planned for future)`。P1-2 的目标即是把框架层已经实现的评估能力**接通**到 vv 应用层，让 vv 用户可以像主流 LLM 应用平台那样对自己的 Agent 做离线回归与运行时指标采集。

### 1.2 调研结论摘要

业界主流做法（OpenAI Evals / LangSmith / Langfuse / MLflow / Promptfoo / maragu.dev/gai/eval）的共识：

| 层级 | 用途 | 典型做法 |
|------|------|---------|
| 离线/回归（Offline） | 开发期回归测试，CI 门禁 | JSONL/YAML 数据集 + 命令行/测试框架执行 + 报告输出 |
| 主观质量（LLM-as-Judge） | 打分开放式回答 | 小模型评判器 + 人工校准反馈 |
| 在线采样（Online） | 生产流量抽样打分 | 异步 Hook + 采样率 + 廉价评估器（latency/cost/heuristic） |

常见问题：LLM Judge 不稳定；全量在线评估成本高（需采样 5-10%）；评估链路不能加生产请求延迟（必须异步）；数据格式应当使用 JSONL 以便与 LLM 平台互通。

### 1.3 目标

在 vv 中引入一层薄封装（不重写 vage/eval），让用户能通过：

- **CLI 一次性评估**：`vv -eval <dataset.jsonl>`（执行后即退出，类似现有 `-p` 模式）
- **HTTP 评估端点**：`POST /v1/eval/run`（用于外部 CI/CD 系统调用）

…对 **Dispatcher** 执行一组测试用例并得到评分报告。在线采样 Hook 属于 P3 范畴，本次不覆盖。

### 1.4 非目标（Out-of-Scope）

明确**不做**的事：

- 生产流量在线采样评估（属于 P3-5/P3-6 范畴，依赖轨迹落盘）
- 评估结果 UI/Web 仪表盘（Web UI 本身是 P4）
- 人工评分反馈回路（依赖 P3-6）
- 持久化评估运行历史（依赖 P1-6 SQLite）
- 评估器自定义注册（仅暴露 6 个内建评估器的组合配置）
- 对 vv 单个子 Agent（coder/researcher/…）分别评估——本次只评估 Dispatcher 入口（与 HTTP `POST /v1/agents/...` 语义一致）
- MCP 模式下的评估（MCP 暴露的是 stdio/HTTP transport，不涉及 Dispatcher 直连）
- 基于 `Expected` 精确匹配的评估（ExactMatch）：保留评估器本身，但数据集 MVP 不要求强约束 Expected 字段
- CI 集成脚本/Action 模板（本次只提供 exit code 语义，具体 CI 模板留给用户）

## 2. 用户故事与验收标准

### US-1：开发者离线回归

**作为**一名基于 vv 定制 Agent 行为的开发者，
**我希望**把过去的 Bug 现场和典型对话整理成 JSONL 数据集，通过命令行跑完所有用例并拿到一个评分报告，
**以便**每次调整 prompt / 模型 / 工具后能快速看出有没有回退。

**验收标准**：

- AC-1.1：用户执行 `vv -eval ./testcases.jsonl` 后，vv 以非交互模式启动，依次读取 JSONL 的每一行作为一个 `EvalCase`，调用 Dispatcher 执行，最后将汇总报告打印到 stdout 并在所有用例都 `Passed` 时以 exit code 0 退出，否则以 exit code 1 退出。
- AC-1.2：用户可以通过 `-eval-out <path>` 额外把完整 JSON 报告写入文件（包含每个用例的 Score/Details/Duration/Usage）。
- AC-1.3：JSONL 每行至少包含 `id`、`input.messages[...]`；`expected`、`criteria`、`tags` 为可选字段；解析失败时打印对应行号与原因，跳过该行继续处理剩余用例，最后把跳过数计入错误数（error_cases）。
- AC-1.4：`-eval` 与 `-p` 互斥；`-eval` 与 `--mode http|mcp` 互斥（出现冲突时 stderr 报错并 exit 1）。
- AC-1.5：支持 `-eval-concurrency <N>`（默认 1）控制并发运行用例数；`-eval-timeout-ms <N>`（默认 60000）为每个用例设置超时。
- AC-1.6：评估器组合通过配置决定（见 §3 配置约定）；无需命令行标志切换。

### US-2：HTTP 评估端点

**作为**一套 CI/CD 流水线，
**我希望**对已经启动的 vv HTTP 服务发 `POST /v1/eval/run` 请求，附带一组测试用例，
**以便**拿到整批评估结果用作发布门禁。

**验收标准**：

- AC-2.1：在 HTTP 模式下，`POST /v1/eval/run` 接受 JSON body `{ "cases": [...] }`，其中每个 case 的 Schema 与 JSONL 单行一致。
- AC-2.2：响应为 `EvalReport` JSON，包含 `total_cases`、`passed_cases`、`failed_cases`、`error_cases`、`avg_score`、`total_duration`、`results[...]`。
- AC-2.3：请求 body 为空或 `cases` 为空数组时返回 400；单个 case 缺少 `id`/`input.messages` 时该 case 计入 `error_cases` 但整个请求不中断。
- AC-2.4：该端点和现有的 `/v1/memory/*`、`/v1/interactions/*` 共存，不互相干扰；受同一个 `requestIDMiddleware` 包裹。
- AC-2.5：未启用时（`eval.enabled: false`）端点返回 404 Not Found——**关闭的功能不应暴露 API**。

### US-3：最小配置与可预测的默认评估器

**作为**一位只想快速看看自己 Agent 表现的用户，
**我希望**不做任何配置也能跑 `vv -eval` 并看到至少 Latency 和 Cost 两项基础指标，
**以便**首次使用的门槛足够低。

**验收标准**：

- AC-3.1：配置 `eval.enabled` 默认为 `false`；当用户通过 `-eval` 命令行标志启动时隐式启用（不强制要求 YAML 改动）。
- AC-3.2：默认评估器组合（`eval.evaluators: ["latency","cost"]`）只用廉价本地评估器——不发起 LLM 判分调用，保证首次运行零额外 LLM 成本。
- AC-3.3：用户可以在 YAML 指定 `eval.evaluators: ["llm_judge","latency","cost"]` 启用 LLM 判分；判分使用 `llm.model`（同主模型）。
- AC-3.4：`eval.latency_threshold_ms`（默认 60000）与 `eval.cost_budget_tokens`（默认 10000）控制 Latency/Cost 评估器阈值。
- AC-3.5：用例内如果提供了 `criteria` 字段且 `llm_judge` 在启用列表中，由 LLM 判分器消费；否则评估器忽略 criteria（不报错）。

## 3. 配置约定（informative，供设计参考）

```yaml
eval:
  enabled: false             # HTTP 端点可见性
  concurrency: 1             # 并发度
  timeout_ms: 60000          # 单用例上限（透传到 context）
  evaluators: [latency, cost] # latency | cost | contains | llm_judge
  latency_threshold_ms: 60000
  cost_budget_tokens: 10000
  contains_keywords: []      # 仅当 contains 在 evaluators 中时生效
  llm_judge_model: ""        # 为空时退回 llm.model
```

对应环境变量：`VV_EVAL_ENABLED`。

## 4. 影响面

| 受影响对象 | 变化点 |
|-----------|--------|
| 角色 | 无新增角色（复用 Developer / DevOps / External Systems） |
| 模型/字典 | PRD 需新增 `EvalCase`、`EvalReport` 术语 |
| 流程/规则 | 新增"离线评估流程"（`vv -eval` → 加载数据集 → 跑 Dispatcher → 评估 → 输出报告）|
| 应用 | CLI 应用：新增 `-eval`、`-eval-out`、`-eval-concurrency`、`-eval-timeout-ms` 标志；新增一个"离线评估"交互模式（只输出报告，无 TUI）。HTTP 应用：新增 `POST /v1/eval/run` 端点 |
| 配置 | 新增 `eval:` 段与 `VV_EVAL_ENABLED` 环境变量 |
| PRD 文档 | `overview.md` 第 75 行 `Evaluation and quality metrics (planned for future)` 删除并迁移到 Covers 段；`feature-todo.md` P1-2 搬入 `feature-implement.md`；新增一份 `procedures/eval.md` 说明使用方法 |

## 5. 假设与开放问题

本次主代理采用"自主交付"模式；以下假设若与用户真实意图不符，请在事后反馈：

1. **Dispatcher 是唯一评估对象**：用户没有对单子 Agent 做评估的紧迫需求（否则得新增 `-eval-agent <id>` 标志）。
2. **Expected 字段不强制**：业界大部分 Agent 数据集不带绝对 Expected，用 LLM Judge + Criteria 是主流；ExactMatch 留给未来需要时再暴露。
3. **评估阻塞式**：MVP 下 CLI `-eval` 和 HTTP `/v1/eval/run` 都是同步阻塞；用户可以通过 `-eval-concurrency` 提高吞吐。异步任务模式 (`/v1/eval/tasks`) 不在本次范围。
4. **Dataset 只读 JSONL**：不支持 CSV/YAML；数据集不大时 JSONL 足够，大数据集属于 P3 训练飞轮范畴。
5. **无在线采样 Hook**：在线评估链路需要先有 P1-5 轨迹落盘；现在加反而提前消耗接口设计预算。

## 6. 强可验证成功标准（Definition of Done）

- ✅ `vv -eval testdata/eval_smoke.jsonl` 在 apikey 已配置的情况下能跑完并打印报告；手动 break 其中一个 case（如把 input.messages 删掉）能复现 AC-1.3 的错误计数；所有 case 通过时 exit 0、有失败时 exit 1。
- ✅ HTTP 模式启动 `vv --mode http`（YAML `eval.enabled: true`）后，`curl -X POST localhost:8080/v1/eval/run -d @cases.json` 返回 200 + `EvalReport`；`eval.enabled: false` 返回 404。
- ✅ 单元测试覆盖：JSONL 加载器（正常/格式错误/缺必填）、报告写入器、Case→RunRequest 转换、配置默认值、HTTP handler 基本路径；不落入集成测试依赖 LLM 的边界。
- ✅ 文档更新：`doc/prd/feature-implement.md` 新增 P1-2 条目；`doc/prd/feature-todo.md` 的 P1-2 行划线 `~~...~~`（与 P1-1 一致的样式）；`doc/prd/procedures/` 下新增 `eval.md`；`doc/prd/overview.md` Does-Not-Cover 删除"Evaluation and quality metrics"；里程碑表 M2 的 P1-2 打 ✅。
- ✅ `make lint` 与 `make test`（vv 模块）都在当前环境通过。
