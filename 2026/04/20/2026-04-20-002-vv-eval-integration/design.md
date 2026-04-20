# 技术设计：P1-2 vv 集成 Eval

## 1. 总体架构

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │                               vv                                    │
 │                                                                     │
 │  main.go ─┬── -eval <dataset.jsonl>  ──► RunEval() [new, blocking]  │
 │           ├── --mode http             ──► httpapis.Serve (+eval EP) │
 │           └── --mode cli|mcp          ──► 原路径                    │
 │                                                                     │
 │  vv/eval/ (new, thin wrapper)                                       │
 │    │                                                                │
 │    ├── dataset.go   JSONL 加载/编解码                               │
 │    ├── evaluator.go 按 config 组装 evaluator 组合                   │
 │    ├── runner.go    调用 vage/eval.RunAndEvaluate                   │
 │    └── report.go    报告 stdout 渲染 + JSON 落盘                    │
 │                                                                     │
 │  vv/httpapis/eval.go (new) - POST /v1/eval/run handler              │
 └─────────────────────────────────────────────────────────────────────┘
                   │
                   ▼
           vage/eval (不修改)
```

**设计原则**：vv 只做"数据搬运 + 命令行/HTTP 胶水"；任何真正的评估逻辑都委托给 `vage/eval`。

## 2. 配置新增

### 2.1 `configs.EvalConfig`

在 `vv/configs/config.go` 里加新字段：

```go
type EvalConfig struct {
    Enabled            bool     `yaml:"enabled,omitempty"`             // default false
    Concurrency        int      `yaml:"concurrency,omitempty"`         // default 1
    TimeoutMs          int      `yaml:"timeout_ms,omitempty"`          // default 60000
    Evaluators         []string `yaml:"evaluators,omitempty"`          // default [latency, cost]
    LatencyThresholdMs int64    `yaml:"latency_threshold_ms,omitempty"`// default 60000
    CostBudgetTokens   int      `yaml:"cost_budget_tokens,omitempty"`  // default 10000
    ContainsKeywords   []string `yaml:"contains_keywords,omitempty"`
    LLMJudgeModel      string   `yaml:"llm_judge_model,omitempty"`     // empty = cfg.LLM.Model
}
```

挂到 `Config` 顶层 `Eval EvalConfig` 字段。

`applyDefaults`：当切片/数值为零值时填默认值。环境变量只支持 `VV_EVAL_ENABLED`（HTTP 端点开关），不覆盖其他参数（避免污染 `configs.Load` 的环境矩阵）。

### 2.2 校验

`Load` 尾部新增 `ValidateEval(&cfg.Eval)`：
- `Concurrency <= 0` → 置 1
- `TimeoutMs < 0` → 返回错误
- `Evaluators` 中出现未知名称 → 返回错误（不静默忽略，让用户及时知道）

## 3. 数据集格式（JSONL）

每行一个 JSON 对象；字段与 `vage/eval.EvalCase` 对齐，但为了命令行易写，输入收紧为"最后一条 user 消息"的字符串快捷写法：

```jsonl
{"id":"case-1","input":"你好，用中文简单介绍 Go 的并发模型","tags":["smoke"]}
{"id":"case-2","input":{"messages":[{"role":"user","content":"summarize main.go"}]},"criteria":["简洁","不虚构内容"]}
```

**解析规则**（`vv/eval/dataset.go`）：

- 如果 `input` 是字符串：构造 `schema.RunRequest{Messages:[schema.NewUserMessage(input)]}`。
- 如果 `input` 是对象且含 `messages`：反序列化为 `schema.RunRequest`。
- 两者都没有 → 该行 Parse 失败，记一条错误（不中断批次）。
- `expected` 可选，结构对称；缺省即 nil。
- `criteria` `[]string` 可选。
- `tags` `[]string` 可选。
- 行解析失败时 stderr 打印 `line N: <err>`，失败行记入最终 report 的 `ErrorCases`。

数据集文件 Schema 顶层就是一行一个对象；**不支持**把整个数组包进 `{"cases":[...]}` ——文件是 JSONL，HTTP 是 JSON；不混用。

## 4. CLI 接入（`main.go`）

新增 flag：

```go
evalFlag := flag.String("eval", "", "run evaluation on the given JSONL dataset file and exit")
evalOutFlag := flag.String("eval-out", "", "write full JSON report to this path")
evalConcurrencyFlag := flag.Int("eval-concurrency", 0, "override eval.concurrency (default 1)")
evalTimeoutMsFlag := flag.Int("eval-timeout-ms", 0, "override eval.timeout_ms per case (default 60000)")
```

**路径**（放在现有 `-p` 单发模式之后、普通 mode 分派之前）：

```go
if *evalFlag != "" {
    if promptSet {
        fatal("-eval is incompatible with -p")
    }
    if cfg.Mode == "http" || cfg.Mode == "mcp" {
        fatal("-eval is incompatible with --mode http/mcp")
    }
    // override from flags
    if *evalConcurrencyFlag > 0 { cfg.Eval.Concurrency = *evalConcurrencyFlag }
    if *evalTimeoutMsFlag > 0   { cfg.Eval.TimeoutMs = *evalTimeoutMsFlag }

    initResult, err := setup.Init(cfg, &setup.Options{
        UserInteractor: askuser.NonInteractiveInteractor{},
        AskUserTimeout: time.Duration(cfg.Agents.AskUserTimeout) * time.Second,
        DebugSink:      debugSink,
    })
    if err != nil { fatal(err) }

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    exitCode, err := vveval.RunCLI(ctx, initResult, cfg, *evalFlag, *evalOutFlag, os.Stdout, os.Stderr)
    if err != nil {
        fmt.Fprintf(os.Stderr, "vv: %s\n", err)
        os.Exit(1)
    }
    os.Exit(exitCode)
}
```

## 5. HTTP 接入（`httpapis`）

### 5.1 新 handler：`POST /v1/eval/run`

只有 `cfg.Eval.Enabled == true` 时才挂载。handler 位于 `vv/httpapis/eval.go`：

```go
func handleEvalRun(cfg *configs.Config, dispatcher agent.Agent, llm aimodel.ChatCompleter) http.HandlerFunc
```

请求 body：

```json
{
  "cases": [
    { "id": "c1", "input": "你好", "criteria": ["简洁"] }
  ]
}
```

响应：`eval.EvalReport`（vage 原类型）的 JSON 序列化结果。

错误路径：
- body 解析失败 → 400 `{"code":"bad_request"}`
- `cases` 为空 → 400
- 单个 case 非法（缺 id/input）→ 记入 `ErrorCases`、`Results[i].Error`，整批不中断
- 评估器构造失败（比如配置错误）→ 500

### 5.2 `Serve` 注册

在 `httpapis/http.go::Serve` 里，当 `cfg.Eval.Enabled` 为 true 时，注册：

```go
mux.HandleFunc("POST /v1/eval/run", handleEvalRun(cfg, dispatcher, llmClient))
```

为了拿到 `llmClient`（LLM Judge 需要），修改 `Serve` 签名增加 `llm aimodel.ChatCompleter` 参数；`main.go` 从 `initResult.LLMClient`（或 wrappedLLM）把它传下去。这里沿用未被 debug 包装的 `initResult.LLMClient`——评估不需要 debug 拦截。

## 6. 新包 `vv/eval/`

### 6.1 文件布局

```
vv/eval/
├── dataset.go       // LoadJSONL(path) ([]*eval.EvalCase, []LoadError, error)
├── dataset_test.go
├── evaluator.go     // Build(cfg EvalConfig, llm, model) (eval.Evaluator, error)
├── evaluator_test.go
├── runner.go        // RunCLI + RunHTTP 核心
├── runner_test.go
├── report.go        // stdout 渲染 + JSON 文件落盘
└── report_test.go
```

包路径 `github.com/vogo/vv/eval`（**注意**：与 `vage/eval` 不同命名；本地导入时写 `vveval "github.com/vogo/vv/eval"` 防止歧义）。

### 6.2 关键函数

```go
// dataset.go
type LoadError struct { Line int; Err error }
func LoadJSONL(path string) (cases []*eval.EvalCase, loadErrs []LoadError, err error)
func DecodeCaseLine(line []byte) (*eval.EvalCase, error)
```

```go
// evaluator.go
func Build(cfg configs.EvalConfig, llm aimodel.ChatCompleter, defaultModel string) (eval.Evaluator, error)
```
内部按 `cfg.Evaluators` 列表实例化：
- `"latency"` → `eval.NewLatencyEval(cfg.LatencyThresholdMs)`
- `"cost"`    → `eval.NewCostEval(&eval.CostConfig{Budget: cfg.CostBudgetTokens})`
- `"contains"`→ `eval.NewContainsEval(&eval.ContainsConfig{Keywords: cfg.ContainsKeywords})`
  - 若 `Keywords` 为空则返回错误
- `"llm_judge"`→ `eval.NewLLMJudgeEval(llm, effectiveModel)`

单评估器直接返回；两个以上则用 `eval.NewCompositeEvaluator` 等权重组合（`Weight: 1.0`）。

```go
// runner.go
func RunCLI(ctx context.Context, init *setup.InitResult, cfg *configs.Config,
    datasetPath, outPath string, stdout, stderr io.Writer) (exitCode int, err error)

func RunBatch(ctx context.Context, dispatcher agent.Agent, evaluator eval.Evaluator,
    cases []*eval.EvalCase, cfg configs.EvalConfig) (*eval.EvalReport, error)
```

`RunBatch` 的关键点：
- 给每个 case 加一个 **per-case timeout**：`caseCtx, cancel := context.WithTimeout(ctx, cfg.TimeoutMs * ms)`。
- 用 `eval.RunAndEvaluate(caseCtx, dispatcher.Run, evaluator, []*eval.EvalCase{c})` 单 case 调用，再合并到总 report——不直接用全局 concurrency 参数，为了可靠地实现 per-case timeout。
- 并发由 `cfg.Concurrency` 控制的 goroutine pool（标准 semaphore channel，复用 `vage/eval/batch.go` 的模式；但因为单 case-timeout 的约束，我们自己实现 10-15 行的 loop，**不重新实现** BatchEval 的全部逻辑）。

```go
// report.go
func WriteReportJSON(path string, report *eval.EvalReport) error
func PrintSummary(w io.Writer, report *eval.EvalReport)
```

`PrintSummary` 打印类似：

```
Eval Report
  Total      : 10
  Passed     : 8
  Failed     : 2
  Errors     : 0
  Avg Score  : 0.82
  Duration   : 4321 ms
```

不试图按用例逐条渲染——那是 `-eval-out` 的职责。

## 7. 默认评估器选择的依据

调研显示 LLM-as-Judge 在 MVP 场景下有两大问题：

1. **稳定性差**：同一个 prompt 连续跑几次评分差异可达 0.2+。
2. **双成本**：评估本身消耗 token，把验收链路做成 `LLM → Agent → LLM`，影响开发者快速迭代。

因此 `latency + cost` 作为零依赖、零额外成本的默认组合——它们只需要 `Actual.Duration` 与 `Actual.Usage`，这两个字段本来就会被 vv Dispatcher 填好。用户确实需要主观评分时再显式 opt-in `llm_judge`。

## 8. 错误与边界

| 场景 | 行为 |
|------|------|
| 数据集文件不存在 | CLI 直接 `fatal`；HTTP 不经过文件路径，不涉及 |
| JSONL 某一行解析失败 | stderr 打印行号；报告 `ErrorCases += 1`；继续下一行 |
| 用例 `id` 重复 | MVP 不去重，由用户自己保证；report 里会出现多条同 id |
| 用例超时 | context.DeadlineExceeded 被 `RunAndEvaluate` 原样返回；runner 捕获后写 `Result.Error = "timeout"`，计入 `ErrorCases` |
| LLM Judge 开启但 `llm.api_key` 缺失 | `setup.Init` 已校验；进到 `RunCLI` 前已经 fatal |
| HTTP 禁用时请求 `/v1/eval/run` | 404（mux 里根本没有注册该路径）—— 保证"关闭的功能不暴露"|
| HTTP 调用超过 `eval.timeout_ms × cases` | 由 HTTP server 自己 handle；我们不额外设 request-level timeout（用户可用反向代理做）|

## 9. 测试计划

### 9.1 单元测试

- `dataset_test.go`：
  - 有效 JSONL 的逐行解析
  - string input vs object input 两种格式
  - 缺 id / 缺 input / 坏 JSON 分别记一条 LoadError
  - 空文件返回空切片无错误
- `evaluator_test.go`：
  - 单评估器（latency/cost）直接返回
  - 多评估器返回 `*CompositeEvaluator`
  - 未知 evaluator 名称返回错误
  - `contains` 不给关键词返回错误
  - `llm_judge` 且 llm 为 nil 返回错误
- `report_test.go`：
  - `PrintSummary` 输出固定格式（snapshot-ish）
  - `WriteReportJSON` 写出后能被 `json.Decode` 还原
- `runner_test.go`：
  - 用 `agent.CustomAgent` 构造一个"echo" dispatcher，跑 3 个 case，验证 exit code、report 内容
  - case timeout 触发时记为 ErrorCases

### 9.2 HTTP handler 测试

在 `vv/httpapis/eval_test.go` 里：
- body 非法 → 400
- cases 为空 → 400
- 正常 case 走 echo dispatcher → 200 + 正确 report
- eval.enabled=false 时路径未注册（Serve 启动后打 `/v1/eval/run` 应返回 404）—— 这一条放到整合 Serve 的测试里，单 handler 测试不覆盖

### 9.3 集成测试（跳过 if no LLM key）

`vv/integrations/eval_tests/` 放一个 `testdata/smoke.jsonl`，用 build tag + env 守卫：
- `go test -tags=integration ./integrations/eval_tests/...` 实际打 LLM；默认 CI 不跑。

## 10. 向后兼容与弃用

- 零向后不兼容：纯新增；现有 `-p`、`--mode` 不改语义。
- `httpapis.Serve` 签名扩展一个 `llm aimodel.ChatCompleter` 参数——内部包，不是稳定 API，直接改 call-site。
- 配置 YAML 新增 `eval:` 段，旧配置文件无此段时使用默认值。

## 11. 工程清单（给 developer 参考）

- [ ] `vv/configs/config.go`: 加 `EvalConfig`、`applyDefaults`、`VV_EVAL_ENABLED` 环境变量、`ValidateEval`。
- [ ] `vv/configs/config_test.go`: 默认值与校验单测。
- [ ] `vv/eval/`: 新包，含 dataset/evaluator/runner/report 四个文件 + 对应 _test。
- [ ] `vv/httpapis/http.go`: `Serve` 签名加 `llm`；在 `cfg.Eval.Enabled` 时挂载路由。
- [ ] `vv/httpapis/eval.go` + `eval_test.go`: handler 实现与测试。
- [ ] `vv/main.go`: 新 flag + 评估入口分派；把 `initResult.LLMClient` 传进 `httpapis.Serve`。
- [ ] `vv/integrations/eval_tests/`: smoke 数据集 + build-tag 守卫的集成测试（可空，MVP 不强求）。
- [ ] `make lint` + `make test` 通过。

## 12. Risks & Mitigations

| 风险 | 缓解 |
|------|------|
| 用户误开 `llm_judge` + 大 dataset → 成本爆炸 | 默认关闭；首次启用时 stderr 打 `warn: llm_judge enabled, each case will make an extra LLM call`。|
| HTTP `/v1/eval/run` 被外部滥用 | 依赖现有 HTTP 服务边界（loopback / `server.addr`）；`eval.enabled=false` 时直接不注册路由；将来加 auth 属于 P4-1。|
| JSONL 文件过大 OOM | MVP 用 `bufio.Scanner` 逐行解析，不一次性 ReadFile；但 case 本身会被全部装进内存——1000 条内无压力，万条级别不在 MVP 范围。|
| `RunAndEvaluate` 串行语义 | 我们自己做并发 loop，不依赖 `WithConcurrency`；语义清晰，per-case timeout 可控。|
