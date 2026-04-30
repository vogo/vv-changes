# Code Review：迭代级 Checkpoint

> 评审范围：本次新增 `vage/checkpoint/` 包 + `vage/agent/taskagent/checkpoint.go` + 修改的 `vage/agent/taskagent/task.go` 与 `vage/schema/event.go`。  
> 评审角度：正确性、设计契合度、并发、错误处理。

## 已修复（applied in this review）

### 1. [HIGH] `runResumeLoop` 错误处理屏蔽真实失败

**问题**：`runResumeLoop` 在 v1 实现里把 `chatCompleter.ChatCompletion` 的错误用 `slog.Warn` 吞掉，然后返回 `finalizeRun(ctx, rc, schema.StopReasonComplete)` —— 调用方看到的是"成功完成"的 RunResponse，与原始 `Run` 路径"propagate fmt.Errorf 包装错误"的契约严重不一致。同样的，`len(resp.Choices) == 0` 也错误地返回 Complete 而不是上报错误。

**根因**：v1 设计里 `runResumeLoop` 仅返回 `*schema.RunResponse`（无 error 通道），导致只能"硬编码一个 stop reason 退场"。

**修复**：将 `runResumeLoop` 的签名改为 `(*schema.RunResponse, error)`，错误用 `fmt.Errorf("vage: chat completion: %w", err)` 包装上报，与原始 `Run` 路径行为对齐；新增 `ErrEmptyLLMResponse` 哨兵错误便于调用方区分。

**影响面**：仅本次新增代码；现有测试全过。`Resume` 的对外契约从`(*RunResponse, error)`变 → 不变（外层接口已经是这个签名），内部传播链补上即可。

---

## 已识别但未修改（recommendation only）

### 2. [MEDIUM] `Resume` 不重新注入 skill instructions

**观察**：`Run` 在初始化路径调用 `a.injectSkillInstructions(&rc.br, rc.sessionID)`。`Resume` 则不调用 —— 直接复用 cp.Messages 中已包含的 system 消息。

**判断**：保留现状。理由：
- cp.Messages 本身是首次 Run 写入时 system 消息已注入完成的版本；恢复时不重复注入是正确的（重复会让 system 消息被二次拼接）。
- 真正的语义模糊在于：如果在 Run 失败到 Resume 之间，`a.skillManager` 的 active skills 集合已经变化（用户在 CLI 里 deactivate 了某 skill），cp 里的 system 消息仍然包含旧 skill 的 instructions —— 但这属于"恢复点存的就是当时的事实，不试图同步事后变更"的合理语义。

**建议**：在 `vage/.doc/checkpoint.md` 里写明这一不变量，避免日后误解。documenter 阶段补上。

### 3. [LOW] cloneMessagesForCheckpoint 的浅拷贝安全前提

**观察**：`cloneMessagesForCheckpoint` 仅复制顶层切片头，`aimodel.Message.Content` 与 `ToolCalls` 的内部指针/切片不深拷贝。设计文档已经在 §5.7 论述安全前提。

**判断**：保留现状。理由：
- ReAct 循环中 messages 元素一旦 append 不再 mutation。
- `markPromptCacheBreakpoints` 会在 `messages[i].CacheBreakpoint = true` 处 in-place 改 system 消息 —— 但 system 消息是 cp.Messages[0]，每次 Resume 重建 messages 切片时 backing array 是新分配的，不会污染已存的 cp。

**建议**：在 `vage/.doc/checkpoint.md` 里把这条不变量明示，避免后续重构时无意打破。

### 4. [LOW] FileIterationStore.List 读每个文件的 O(N) 代价

**观察**：`List` 为了构造 `CheckpointMeta` 需要解 JSON，目前是逐文件 `os.ReadFile + json.Unmarshal`。N=数百以下完全 OK，文档也已说明。

**判断**：保留现状。如果未来真触发性能问题，应当迁移到 SQLite 后端（设计文档已规划），而不是在 file 后端为了 List 单独加 metadata 索引文件——后者会让 atomic write 复杂化。

### 5. [LOW] generateID fallback 路径的退化

**观察**：`generateID` 的 fallback 在 `crypto/rand` 失败时返回 `"fb" + hex.EncodeToString([]byte(time.Now().Format("150405.000000")))`。`fb` 是 "fallback" 缩写，hex 编码的是文本字符串字节，不是真随机源。

**判断**：保留现状。理由：crypto/rand 在 darwin/linux 失败极其罕见（kernel 故障级别）；即便 fallback，sequence 前缀（FileStore）/ map 索引（MapStore）已经保证同 session 内 cp 唯一性。fallback 仅起到"区别于本秒内其他 fallback id"的兜底作用，够用。

### 6. [LOW] Resume 的 `RunRequest.Messages` 投影丢失

**观察**：`Resume` 把 `rc.reqMsgs` 设为 nil。这意味着 `storeAndPromoteMessages` 在 finalize 时仅把 respMsgs 写入 working memory，而不写 reqMsgs。

**判断**：这是正确的：原始 Run 的 reqMsgs 已经在第一次 Run 的 finalize 写入过 —— 但等等，**第一次 Run 是崩溃的，finalize 从未被调用**。所以原始 reqMsgs 实际上没有写入 working memory。

**深度判断**：但 working memory 是 Run 范围的临时态（每次 Run 重新创建），跨进程恢复时 working memory 本就被丢弃；session memory 才是持久化的部分。`Resume` 在 finalize 时调用 `storeAndPromoteMessages` 也只是把当前 Resume 这次 Run 的 working memory 提升到 session memory —— 而 session memory 在崩溃前没有这次 Run 的内容（因为没 finalize 过），所以现在写入 respMsgs 也不会重复。**结论**：行为正确，但语义微妙。

**建议**：在 `vage/.doc/checkpoint.md` 的 Resume 段落里补一句注解，避免未来维护者把 reqMsgs 反复 promote 的逻辑加进来。

---

## 测试覆盖盘点

### 已有覆盖（unit tests）
- ✅ `TestRun_NoCheckpointStore_NoOp` — 验证未配置 store 时 Run 行为不变。
- ✅ `TestRun_WritesCheckpointPerIteration` — 3 轮 → 3 条 cp，最后 Final。
- ✅ `TestResume_MissingStore_ReturnsInvalidArgument` — 前置条件校验。
- ✅ `TestResume_NoCheckpoint_ReturnsNotFound` — 空 session 错误路径。
- ✅ `TestResume_AlreadyFinal_ReturnsErrAlreadyFinal` — Final 短路。
- ✅ `TestResume_AfterFailedRun_ContinuesFromCheckpoint` — 核心恢复路径。
- ✅ `TestResume_CrossAgent_Rejected` — 安全网。
- ✅ Conformance contract suite（map / file 共用）。
- ✅ FileStore 文件名布局 / `.json.tmp` 不计入 sequence。

### 留给 tester 阶段（integration tests）
- Stream 路径（`runStreamLoop`）的 cp 写入。
- 预算耗尽 on iter 0 的边界（messages 仍带 system + history）。
- `EventCheckpointWritten` payload 完整性（hook 订阅验证）。
- FileIterationStore 跨 session 并发 Save。
- 端到端：Run + 断点 + Resume，对照"未中断完整跑"验证最终 RunResponse 等价。

---

## 综合结论

- **Critical**：无。
- **High**：1 项（`runResumeLoop` 错误处理）已修复并跑通测试。
- **Medium / Low**：5 项均为文档建议，留给 documenter 阶段处理。
- 设计意图与代码实现对齐；invariants（Final ⇔ StopReason）在 saveIterationCheckpoint 调用点全部正确传递。
- 并发安全：MapStore / FileStore 锁粒度合理；TaskAgent.Resume 不引入新的并发面（Resume 自身串行）。
- `make test` / `make lint` / `make license-check` 全部通过（HIGH 修复后复测）。
