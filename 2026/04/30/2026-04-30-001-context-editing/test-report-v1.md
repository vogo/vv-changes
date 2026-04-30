# Context Editing — 集成测试报告 v1

## 1. 总体结论

**集成测试阶段：通过 (PASS)**

- 新增集成测试文件：`vage/integrations/largemodel_tests/context_editor_tests/context_editor_test.go`
- 测试用例数：**7** 个集成场景，全部通过。
- 上游单测（`vage/largemodel/`、`vage/agent/taskagent/`）保持通过，无回归。
- `vage/integrations/...` 全包跑通，无回归。
- `make lint` 通过（0 issues）。

## 2. 覆盖矩阵

针对 P6 *Context Editing* 的设计契约（design.md）与三条用户故事（requirement.md US-1 / US-2 / US-3），集成层面新增的 7 个场景如下：

| # | 集成场景 | 覆盖的 AC / 设计点 | 结果 |
|---|---|---|---|
| 1 | `TestIntegration_TaskAgent_ContextEditor_LongReActLoop` | US-1 全部 AC（折叠老结果、保留近 K、ToolCallID 配对、最终响应不变）；4 轮以上 ReAct 端到端 | PASS |
| 2 | `TestIntegration_ContextEditor_EventEmission` | US-2 AC-2.1 / AC-2.5（hook.Manager 实例化 + manager.Dispatch 注入 + EventContextEdited payload 校验：Edited、Kept、Total、OriginalBytes、Placeholder、Strategy） | PASS |
| 3 | `TestIntegration_ContextEditor_SilentPassUnderK` | US-2 AC-2.2（≤ K 不派发事件）；下游与 caller 共享同一 slice header（零拷贝快路径） | PASS |
| 4 | `TestIntegration_ContextEditor_MinElidedBytesThreshold` | 设计 §3 阈值分支（minElidedBytes 拦截）；US-2 AC-2.2 同等行为 | PASS |
| 5 | `TestIntegration_TaskAgent_ContextEditor_StreamPath` | US-1 AC-1.5 + 设计 §4（`RunStream` 走 SSE，迭代 ≥ K+2 时至少一笔出站请求被擦除） | PASS |
| 6 | `TestIntegration_TaskAgent_NoContextEditor_NoElision` | US-2 AC-2.4 + US-3 AC-3.3（不挂中间件零回归） | PASS |
| 7 | `TestIntegration_TaskAgent_ContextEditor_CallerMutationInvariant` | US-1 AC-1.3（caller 持有的 RunRequest 不被原地修改）+ 设计 §3 不变量 1（agent 内部累积态完整、最新一条 tool_result 仍为原文） | PASS |

## 3. 测试形态说明

- 包风格：`package context_editor_tests`（`_test` 包），全部走公开 API（`largemodel.NewContextEditorMiddleware` / `largemodel.WithKeepLastTools` / `largemodel.WithMinElidedBytes` / `largemodel.WithContextEditDispatch` / `taskagent.WithContextEditor` / `taskagent.New` / `hook.NewManager` / `hook.NewHookFunc`）。无任何对内部字段的反射或包内访问。
- 关键 fixture：
  - `recordingCompleter` — 实现 `aimodel.ChatCompleter`，深拷贝快照每次 `ChatCompletion` 收到的 `req.Messages`，避免 ReAct 后续 append 反向污染断言。
  - `streamCapturer` — 实现 `aimodel.ChatCompleter`，把 `ChatCompletionStream` 委托给真实 `aimodel.NewClient` 的 SSE 路径，同时快照 outbound messages。
  - SSE 工具函数（`sseStreamServer` / `toolCallChunks` / `textDeltaChunk` / `stopChunk` / `mustMarshal`）从 `taskagent` 包内对应的私有 helper 复制过来，测试包不引入新基础设施，仅复用已有协议层文本。
- Hook 集成：场景 #2 用 `hook.NewManager()` + `hook.NewHookFunc(... , schema.EventContextEdited)` 注册一个同步 listener，把 `manager.Dispatch` 当 `DispatchFunc` 注入中间件，验证事件路径端到端真打通。

## 4. 关键不变量已逐条验证

来自 design.md §3：

1. ✅ caller 的 `req.Messages` slice 不被原地修改（场景 #3 用 `reflect.DeepEqual` 比对原始 vs 下游捕获副本；场景 #7 在 TaskAgent Run 后比对原始 RunRequest）。
2. ✅ assistant.ToolCalls[].ID 不动；tool_result.ToolCallID 不动（场景 #1 + #5 全量遍历）。
3. ✅ 编辑顺序与原 messages 一致；只替内容、不删消息（场景 #1 通过 `wantTotal[]` / `wantElided[]` 数组逐位对照）。
4. ✅ system / user / assistant 完全不动（场景 #1 走 TaskAgent 真实路径，request 中 system/user/assistant 显式存在并未被擦）。
5. ✅ 事件仅在实际编辑发生时派发；threshold 拦下来不编辑也不派发（场景 #2 / #3 / #4）。

## 5. 主动选择不再叠加的覆盖（理由）

- **`WithPlaceholder` 自定义占位符** —— 已在 `largemodel/context_editor_test.go::TC-10` 的单测覆盖（middleware 内部行为，不需要再走 TaskAgent 一遍）。
- **`CacheBreakpoint` 字段透传** —— 已在 TC-11 单测覆盖；TaskAgent 集成路径目前由 `markPromptCacheBreakpoints` 标在 system message + 最后一个 tool definition，不在 tool_result 上，所以再加一层集成验证收益低于成本。
- **nil DispatchFunc 不 panic** —— 已在 TC-6 单测覆盖；集成层等价场景就是"不传 `WithContextEditDispatch`"，已经隐含在场景 #1 和 #5 中（这两个 case 都没注册 dispatcher，且执行不 panic）。
- **流式路径的事件派发** —— 当前 ContextEditorMiddleware 在流式与非流式上共用 `edit()`，事件分发逻辑相同。集成层把流式路径的"擦除发生"作为代理验证，事件 payload 由非流式场景 #2 充分验证。

## 6. 命令记录

```
$ cd vage && go test ./integrations/largemodel_tests/context_editor_tests/ -count=1 -v
… 7/7 PASS …
ok  github.com/vogo/vage/integrations/largemodel_tests/context_editor_tests  0.520s

$ cd vage && go test ./integrations/... -count=1
ok  github.com/vogo/vage/integrations/agent_tests/routeragent_tests       0.467s
ok  github.com/vogo/vage/integrations/agent_tests/taskagent_tests         1.398s
ok  github.com/vogo/vage/integrations/agent_tests/workflowagent_tests     1.827s
ok  github.com/vogo/vage/integrations/context_tests                       2.187s
ok  github.com/vogo/vage/integrations/eval_tests                          2.632s
ok  github.com/vogo/vage/integrations/guard_tests                         3.259s
ok  github.com/vogo/vage/integrations/largemodel_tests/context_editor_tests 3.603s
ok  github.com/vogo/vage/integrations/mcp_tests                           4.053s
ok  github.com/vogo/vage/integrations/memory_tests/compressor_tests       4.407s
ok  github.com/vogo/vage/integrations/metrics_tests                       3.811s
ok  github.com/vogo/vage/integrations/orchestrate_tests                   4.088s
ok  github.com/vogo/vage/integrations/service_tests                       3.918s
ok  github.com/vogo/vage/integrations/skill_tests                         3.905s
ok  github.com/vogo/vage/integrations/tool_tests/agenttool_tests          3.556s
ok  github.com/vogo/vage/integrations/tool_tests/askuser_tests            4.230s
ok  github.com/vogo/vage/integrations/tool_tests/bash_tests               5.176s
ok  github.com/vogo/vage/integrations/tool_tests/edit_tests               4.119s
ok  github.com/vogo/vage/integrations/tool_tests/glob_tests               4.173s
ok  github.com/vogo/vage/integrations/tool_tests/grep_tests               3.895s
ok  github.com/vogo/vage/integrations/tool_tests/pathguard_tests          4.146s
ok  github.com/vogo/vage/integrations/tool_tests/read_tests               4.068s
ok  github.com/vogo/vage/integrations/tool_tests/write_tests              4.062s

$ cd vage && go test ./largemodel/ ./agent/taskagent/ -count=1
ok  github.com/vogo/vage/largemodel       0.563s
ok  github.com/vogo/vage/agent/taskagent  2.188s

$ cd vage && make lint
golangci-lint run
0 issues.
```

## 7. 文件清单

新增：

- `vage/integrations/largemodel_tests/context_editor_tests/context_editor_test.go`
- `changes/2026/04/30/2026-04-30-001-context-editing/test-report-v1.md`（本文档）

未修改任何源代码。
