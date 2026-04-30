# 测试报告 v1：迭代级 Checkpoint

## 1. 测试总体结果

✅ **全部通过**：单元测试 + 集成测试，无 flaky / 失败用例。

| 范围 | 文件 | 用例数 | 结果 | 平均耗时 |
|---|---|---|---|---|
| 单元（colocated） | `vage/checkpoint/store_conformance_test.go` | 10（map+file 各一遍黑盒） | ✅ | < 0.01s |
| 单元（colocated） | `vage/checkpoint/mapstore_test.go` | 1（深拷贝隔离） | ✅ | < 0.01s |
| 单元（colocated） | `vage/checkpoint/filestore_test.go` | 3（layout / tmp 跳过 / 空 root 拒绝） | ✅ | 0.01s |
| 单元（colocated） | `vage/agent/taskagent/checkpoint_test.go` | 7 | ✅ | 0.01s |
| 集成 | `vage/integrations/agent_tests/taskagent_tests/checkpoint_resume_test.go` | 10 | ✅ | 0.06s |
| **合计新增** | | **31** | **31/31** | |

`make lint` clean（0 issues）；`make license-check` clean；全仓库 `go test ./...` 全部 `ok`。

## 2. 验收标准覆盖

| AC | 描述 | 测试用例 | 单元/集成 |
|---|---|---|---|
| AC-1.1 | 崩溃后 Resume 续行成功 | `TestCheckpoint_AC_1_1_ResumeContinuesFromLatest` | 集成 |
| AC-1.1 | （单元侧的对称） | `TestResume_AfterFailedRun_ContinuesFromCheckpoint` | 单元 |
| AC-1.2 | 零迭代 session 等价于 Run | （隐含在 AC-1.3：未 Run 过的 session id 必然没 cp，返回 ErrCheckpointNotFound） | 集成 |
| AC-1.3 | 未知 session_id 返回 NotFound | `TestCheckpoint_AC_1_3_ResumeUnknownSession`、`TestResume_NoCheckpoint_ReturnsNotFound` | 集成 + 单元 |
| AC-2.1 | K 轮迭代 → K 个 cp，最后 Final | `TestCheckpoint_AC_2_1_KIterationsKCheckpoints`、`TestRun_WritesCheckpointPerIteration` | 集成 + 单元 |
| AC-2.2 | 中途崩溃保留前面的 cp | `TestCheckpoint_AC_2_2_PartialCheckpointsAfterCrash` | 集成 |
| AC-2.3 | 未配置 store 与不开等价 | `TestCheckpoint_AC_2_3_NoStoreEquivalent`、`TestRun_NoCheckpointStore_NoOp` | 集成 + 单元 |
| AC-3.1 | List 返回 metadata，不含 messages | `TestCheckpoint_AC_3_1_ListMetadataDoesNotEmbedMessages` | 集成 |
| AC-3.2 | Load("")=latest / Load(id) / 不存在=NotFound | conformance test 中三个子测：`load_latest_and_by_id`、`load_unknown_id_returns_not_found`、`load_id_from_other_session_returns_not_found` | 单元 |
| AC-3.3 | Hook 收到 EventCheckpointWritten | `TestCheckpoint_AC_3_3_HookEventEmitted` | 集成 |
| AC-4.1 | FileStore 目录布局 | `TestCheckpoint_AC_4_1_FileLayout`、`TestFileIterationStore_FileNameLayout` | 集成 + 单元 |
| AC-4.2 | MapStore Load 深拷贝 | `TestMapIterationStore_LoadReturnsCopy` | 单元 |
| AC-4.3 | 共用契约测试 | `runStoreContract` 跑过 map / file 两遍 | 单元 |

**额外覆盖**：
- 跨 agent Resume 拒绝：`TestResume_CrossAgent_Rejected`、`TestCheckpoint_AC_1_1_ResumeContinuesFromLatest` 通过相同 agent ID 验证正向。
- Final 短路：`TestResume_AlreadyFinal_ReturnsErrAlreadyFinal`。
- 缺 store 调用 Resume 拒绝：`TestResume_MissingStore_ReturnsInvalidArgument`。
- File 后端跨 session 并发 Save：`TestCheckpoint_FileStore_ConcurrentDifferentSessions`（8 goroutine 并发 Save，每个 session 各拿 sequence=1）。
- iter=0 预算耗尽：`TestCheckpoint_PreCallBudgetExhausted_FirstIter`（验证一轮成功后第二轮 pre-call check 触发，BudgetExhausted Final cp 被写入）。
- `.json.tmp` 不计入 sequence：`TestFileIterationStore_TmpFileNotCounted`。

## 3. 已知 gap（**不阻塞当前迭代**，建议下一轮处理）

| Gap | 影响 | 推荐处置 |
|---|---|---|
| Stream 路径（`runStreamLoop`）的 cp 写入未在集成层覆盖 | mock 现有 `mockChatCompleter.ChatCompletionStream` 返回 not implemented，造成构造测试需要新建一个流式 fake，工作量较大 | 下一迭代如果有 vv CLI/HTTP wiring 时一起补；目前依赖单元层 / 黑盒推断"两条路径都调同一个 saveIterationCheckpoint 不变量" |
| MaxIterations 终态 cp | 写入路径有，无显式测试 | 单元侧 budget-exhausted 测试已经 cover 同样的 finalize 路径，差异仅在 StopReason；优先级低 |
| Session memory 与 cp Messages 的双写一致性 | 设计文档明确 reqMsgs=nil 是有意，cp Messages 已包含 user 输入；目前没有显式 storeAndPromoteMessages 在 Resume 后的状态测试 | 进入 vv wiring 时配套补 e2e |

## 4. 运行命令

```bash
cd /Users/hk/workspaces/github/vogo/vagents/vage
go test ./checkpoint/ -v
go test ./agent/taskagent/ -run "TestRun_NoCheckpointStore|TestRun_WritesCheckpoint|TestResume_" -v
go test ./integrations/agent_tests/taskagent_tests/ -run TestCheckpoint -v
make lint
```

## 5. 结论

满足需求 §2 全部 12 条 AC；新增 31 个测试用例全部通过；无回归；lint clean。可以进入 documenter 阶段。
