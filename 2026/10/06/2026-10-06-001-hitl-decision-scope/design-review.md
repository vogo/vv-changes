# 计划评审：HITL 授权边界、策略指纹与记忆删除

- 日期：2026-10-06
- 对象：`.work/todo.md`（开发计划，不是实现）
- 结论：计划已按本次评审修正，可以交给实现。实现者写 `requirement.md` 与 `design.md`，不要覆盖本文件。

评审对照的是当前工作区源码，不是计划里的转述。`largemodel` 不改。`schema.InterruptDecision` 没有 `Execute`；HTTP 批准走 `interrupt.Decision`，经 `Store.SubmitDecisions` 写入。`toStoreDecisions` 不拷贝 `Execute`，这是既有行为，批准测试必须先写 store 再 `ResumeInterrupt`。

## 1. 相对源码的正确性

**通过。** 下列行为与计划引用一致，行号仍对：

- `reconcileInterruptBatch`（`vage/agent/taskagent/interrupt.go` 649 行）对整个 `toExecute` 调用 `WithApprovedExecute`。640–641 行把从未 pending 的兄弟直接 append。
- `executeToolBatch` 在 115 与 130 行把同一个 ctx 传给每次 `executeToolCall`。默认 `maxParallelToolCalls` 是 4。
- `vv/cli/permission.go` 190–193 行先拒绝 `TierBlocked`，196 行才看 `IsApprovedExecute`。
- `vv/main.go` 179、218、266、296 行是 `NonInteractiveInteractor`，311 行对 `http` / `mcp` 设非交互。计划原先漏了 296，已补上，结论不变。
- `CurrentVersion = 2`；`FileStore.readRecord` 与 `MapStore.Get` 要求版本相等。`FileStore.List` 448–451 行对 `readRecord` 错误 `continue`。
- `applyFactoryInterrupt` 比较 `ProfileFull.Name`。Primary 描述符是 `ProfileReadOnly`，`primaryToolProfile` 返回 `ProfileFull`，interrupt 三字段在 `buildPrimaryAssistant` 559–561 行手写。`buildFallbackPrimary` 的 `FactoryOptions` 没有 interrupt 字段。
- `CapRemember` 在 `registerCapabilityTools` 里 `return nil`。`maybeRegisterMemoryTools` 在 store 为 nil 时直接 return。worker 264 行是另一处内联判断。
- `WithApprovedExecute` / `IsApprovedExecute` 的调用点只有 `taskagent/interrupt.go`、`vv/cli/permission.go` 及对应测试。`approved.go` 没有 Apache-2.0 头。
- `memory.Store`（工具小接口）只有 `Get` / `Set` / `List`。`agentPathMemory.Delete` 已存在，`agentToolStore` 还没有 `Delete`。
- `NewDangerousBashPolicy` 今天返回 `InterruptPolicyFunc`。调用点只有 `vv/setup/interrupt.go` 与 `vv/dispatches/interrupt_policy_test.go`。
- `PathGuardian` 的目录字段未导出；`NewPathGuardian` 在空目录时返回 nil。`Classifier.rules` 未导出。`buildPathEnforcement` 在 `installInterrupt` 之前写回 `cfg.Tools.AllowedDirs`。
- `resumeFromInterrupt` 先发 `AgentStart` / `interrupt_resumed` 再执行。`Create` 只覆盖 Version / Revision / 时间 / Status，不清 `Decisions` 与租约字段。`initialStatus` 只看 `Pending` 是否为空。

## 2. 安全

**问题（blocker），已改计划。** 继任记录若用 `newRec := *old` 再改几个字段，旧的 `Execute=true` 会留下来。`Create` 不会清 `Decisions`。新 Pending 若比旧决策多一个 id，人补上那一条之后 `applyDecisions` 会把状态推进到 Ready，旧批准就会执行。D4 改为字段白名单，`Decisions` 与租约必须是零值。

**问题（blocker），已改计划。** 查找 `Supersedes` 必须在任何执行路径之前，包括「当前指纹又与旧记录一致」的情况。否则规则先漂移、继任已写下、规则再改回去时，会按旧决策跑 handler。D4 把查找放在指纹分支之前。

**通过（修正后仍成立）。**

- D1 让未批准的兄弟不进集合；`executeToolCall` 各自派生 child ctx。同批 Dangerous bash 不会再被批级 bool 连带放行。
- `TierBlocked` 在 permission 里先于批准判断返回。`BashTool.execute` 对 guardian 的 `TierBlocked` 再拒一次。批准标记绕不过 Blocked。
- 无 Witness、或新 flagged 集合为空：不取租约、不执行、不建空 Pending。`ReleaseLease` 会把 Resuming 退回 Ready 并保留决策，所以这两条失败路径禁止先 `AcquireLease`。
- handler 把同一 ctx 再送进 `Registry.Execute` 时，`ExecutingCallID` 仍是外层 id。当前 bash handler 不这样做。本轮不改工具内部。这是残余风险，不是把 P0 放宽。

## 3. 兼容与版本

**通过，补了一处测试写法。** v3 写入、v2 可读且不改写文件、v1 与其它版本 `ErrUnknownVersion`，与 FileStore / MapStore / `cloneRecord` / 现有 v1 测试一致。`Create` 会覆盖 `Version`，v2 用例必须绕过 `Create` 改版本后再 `Get`。两处版本判断收成一个 `versionReadable`，避免只改一个 store。

`interrupt.Store` 不加方法。`tool/memory.Store` 增加 `Delete` 是计划内的破坏性变更，随 vage v0.13.0。公开 API 删除两个批准函数、新增四个，vv 是唯一调用方。

版本顺序与仓库规则一致：实现阶段靠 `go.work`，不往已发布的 `go.mod` 写 sibling `replace`，也不在 tag 之前改 `vv/go.mod` 的 require。之后由发版步骤先 tag vage `v0.13.0`，再改 vv 的 require，再 tag vv。未要求发版时不打 tag。

## 4. D4 继任流程

**问题（important），已改计划。**

- 核对点改到 `SubmitDecisions` 之后、`AcquireLease` 与 `resumeFromInterrupt` 之前。同一次调用若刚好把记录推进到 Ready，也要先核对再执行。仍是 Pending 则保持探针，不建继任者。
- 已有继任者时，`ErrLeaseHeld` 或 `Complete` 失败仍然返回继任者，不执行旧 handler，不再 Create。`Complete` 失败不要 `ReleaseLease`。
- 继任者自己已经 Completed 时返回 `ErrAlreadyCompleted`，不把它伪装成新的挂起。
- 多条 `Supersedes` 取 `CreatedAt` 最早的一条并打错误日志。
- resume 侧对冻住的 `rec.ToolCalls` 再做一次 Witness 与 Intercept 的集合核对；flagged id 不是该批子集则 `ErrInterruptPolicyDrift`，不 Create。
- `POST .../decisions` 不跟随 `Supersedes`。`writeInterruptErr` 的 default 是 500，`policy_drift` 必须单独分支。继任成功是 200，body 的 `interrupt_id` 来自 `resp.Interrupt`（`handleResumeInterrupt` 已经这样赋值）。

死锁：FileStore 的文件锁只包住单次读改写，MapStore 的互斥也只包住单次方法。`AcquireLease(旧)` 返回之后再 `Create(新 id)`，不会嵌套占住同一把锁。

## 5. 能力接线（P1-2）

**通过。** Primary 必须走 `primaryToolProfile`（今天就是 `ProfileFull`），不能用描述符上的 `ProfileReadOnly`。`CapInterrupt` 只进 `ProfileFull`。`registerCapabilityTools` 对它返回 nil，否则 `BuildRegistry(ProfileFull)` 会掉进 `unknown capability`。名字等于 `full` 但没有该能力的 profile 不装配。worker 不调用 `AppendInterrupt`；`tool_access: full` 也不接。Fallback 的字面量不调用 `ApplyInterrupt`。不导出内部字段来为这两处写反射测试。

## 6. 记忆契约（P2-1 / P2-2）

**问题（important），已改计划。** 计划原先写「他会话私有条目一律 `ErrSessionForbidden`」。`FileStore.Delete` 在 owner 不匹配时确实返回该错误。`SQLiteStore.Delete` 只删除调用者 `session_id`（以及 legacy 空 session）的行，他会话的行是静默 no-op，返回 nil。缺失 key 两边都是 nil。本轮不改 SQLite。测试走 `FileStore`。

其余保持：不新增 `memory_delete` 工具；不把 `Clear` 开放给 agent-path（MEM-R5 原文不动）；修订 MEM-R9，单 key delete 走 `memory_set` 的 `op=delete`。`value` 移出 JSON Schema 的 `required`，由 handler 校验。`ReadOnly` 保持 true。事件用已有的 `EmitCustomData`，不新增 largemodel EventType。`ttl` 单位是秒，与 `vage/memory` 的 `TTL` 注释一致。nil store 只 `slog.Warn`，不中止启动；没有 `CapRemember` 时不 Warn。

## 7. 测试完整性

**问题（important），已改计划。** 验收项都有了具名落点，并补上这些缺口：

- P0-1 集成测试保持默认并行度，handler 读自己的 ctx。批准走 `Store.SubmitDecisions`，不走 `schema.ResumeInterruptRequest.Decisions`。
- permission 测试分开「有 executing id 但集合为空」和「完全没有 executing id」。
- 指纹改回旧值之后，resume 旧 id 仍返回继任者。
- `AuditVersions` 不把 `.lock` 算进 unknown。
- P2-2 的 Warn 用 `vv/configs` 测试里已有的 `slog.SetDefault` + buffer，不改生产签名。

Fallback / worker 不装配：由「不调用 `ApplyInterrupt` / `AppendInterrupt`」保证，不新造导出口。

## 8. 文档与 ADR

**问题（important），已补进文档清单。** 原先漏了仍会在终态里说错的句子：

- `agent-core.md` 里 “v2 snapshot” 是当前架构陈述，要改成可读集合。ADR 0003 的历史段落不动。
- `agent-core-design.md` 的 “v1 reader must refuse a v2 record” 在 v3 读取器上不成立，要删掉这句，而不是只把 “v2” 换成 “3”。
- HTTP-R9 与 http-api design 要写 409 `policy_drift` 与 200+新 `interrupt_id`。
- `agents/spec.md` 的 ToolCapability 枚举要加入 Interrupt。

vage 文档用英文终态，vv 文档用中文，终态不写「以前是 bool」。

## 9. 范围与顺序

**通过。** 七项仍都做。P1-1 不写 v1 迁移。P2-2 不把启动改成 fail-closed。不做 worker HITL、不做 checkpoint / memory accessor 重构、不改 `TierBlocked`、不改 `largemodel`。`approved.go` 重写时补 Apache-2.0 头，否则 `make build` 的 license-check 失败。实现顺序仍是 vage 契约先绿，再改 vv。

## 10. 可行性

**问题（blocker），已改计划。** 指纹函数若放在 `vv/dispatches`，编译不过：`assembleBashRules` 未导出，`Classifier.rules` 与 `PathGuardian` 的目录字段也未导出，计划又禁止反射。指纹改为 `vv/configs.BashPolicyFingerprint`，与规则组装同包。`NewDangerousBashPolicy` 只接收字符串，并返回具名结构体。若仍包成 `InterruptPolicyFunc`，类型断言永远失败，所有带指纹的 resume 都会 409。

`InterruptPolicy.Intercept` 的签名不变，`InterruptPolicyFunc` 与按工具名的策略不用改。`With*` 对 nil ctx 使用 `context.Background()`，避免 `context.WithValue` panic。新导出符号写 godoc。

## 残余风险（有意留下）

- 磁盘上的 v2 记录没有指纹。新代码仍执行其中已经 `Execute=true` 的调用；未批准的兄弟被 D1 拒绝；`TierBlocked` 仍拒绝。规则收紧后，这条已批准的调用仍会跑。窗口写在 ADR 0001。
- 分类器实现变了、但规则文本与目录没变时，指纹不变。
- bash handler 不二次进入 registry。若将来有工具把同一 ctx 再送进 `Registry.Execute`，外层 executing id 仍可能放行。本轮不改工具内部。
- `PathGuardian.Classify` 不访问磁盘。文件出现或消失不改变指纹。Blocked 在执行期仍硬拒绝。
- 决策还没凑齐时发生漂移：已经提交到旧 id 的决策在 Ready 之后被丢弃。`POST .../decisions` 不自动改写到新 id。
- `Create` 成功、`Complete` 之前崩溃：旧记录停在 Resuming，客户端仍能从 resume 拿到继任者 id；旧记录要等租约到期才能补 Completed。
- SQLite 对他会话私有 key 的 delete 是静默 no-op，不返回 `ErrSessionForbidden`。
- 回滚必须 vage 与 vv 一起回到 v0.12.2 / v0.1.2。只回滚一边会对不上 API，或让 v0.12.2 因 v3 版本号拒绝记录（这是安全方向，进行中的审批会卡住）。

## 结论

计划可以实施。上面的 blocker 与 important 都已经写回 `.work/todo.md` 的对应 D#、测试、文档清单和验收项，没有扩大范围，也没有放宽 P0。
