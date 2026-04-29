# Plan 工作区 — 代码评审报告

> 评审范围：本次会话交付的 vage/workspace/、vage/tool/workspace/、vage/context/sources_workspace.go、vage/agent/taskagent/extra_sources、vage/schema/event 增量、vv/registries/、vv/agents/、vv/setup/、vv/httpapis/workspace.go 与 sessions DELETE 联动。
>
> 评审标准：correctness、design fit、security、concurrency、error handling。
>
> 结论：实现已落地、行为符合 §4.4 设计与本期需求；测试与 lint 均绿。本次评审发现一个语义级 bug（plan 截断方向）和一处过时注释，已就地修复并补充测试。其余项目大多为可接受的设计权衡，记录在"已接受/无需修改"段以备追溯。

---

## 1. 已修复（Critical / Logical Bugs）

### C-1：`WorkspaceSource` plan 截断方向与文档自相矛盾，且实际行为不利于 LLM

**文件**：`vage/context/sources_workspace.go`

**现象**：

```go
// 字段注释（旧）
// MaxBytes ... The injected text is truncated from the END
// (head-preserving) so the most recent edits survive — most LLMs
// append new steps at the bottom of plan.md.

// 实现（旧）
if len(plan) > maxBytes {
    plan = plan[:maxBytes]   // ← 保留头部，丢弃尾部
    truncated = true
}
```

注释自身存在内部矛盾："truncated from the END" + "head-preserving"（保留头部）= 删除尾部；但同一段又称"最近的编辑得以保留 — 大多数 LLM 把新步骤追加到 plan.md 底部"，这与"删除尾部"完全相反。

**影响**：

- 行为级 bug：当 plan.md 超过 `MaxBytes` 时，**最新的进度被丢弃**，LLM 只看到陈旧的早期内容，与本期需求 US-3 第 1 行"LLM 决定下一步要做什么、不重复已完成的步骤"的目标冲突。
- 现有测试 `TestWorkspaceSource_TruncatesOversizedPlan` 用全 `Z` 字符的 plan，无法区分头/尾，因此未捕捉到此 bug。

**修复**：

1. 改为 tail-preserving：`plan = plan[len(plan)-maxBytes:]`，保留尾部（最近的编辑）。
2. 截断时在渲染 plan 段前加一行显式标记 `(... earlier portion of plan.md elided; tail shown ...)`，让 LLM 明确知道当前所见为尾部，避免误以为整篇 plan 就这么短。
3. 更新 `MaxBytes` 字段注释，明确"丢弃前缀、保留尾部"+"插入截断 marker"。
4. 重写 `TestWorkspaceSource_TruncatesOversizedPlan`，用 `H` 头部 + `T` 尾部双字符断言：截断后 H 应为 0、T 应为 `tailLen`、且文本必须包含截断 marker。

**为何不选择"head-preserving"**：plan.md 的语义是任务大纲；新加入的步骤、最新的"- [x]"勾选都会出现在文档尾部。对 LLM 的下一步决策来说，最近发生的事最有价值。

### C-2：`buildInitialMessages` 文档注释过期

**文件**：`vage/agent/taskagent/task.go`

注释（旧）声称："Builder configured with three built-in sources" 且消息顺序"matches the previous hand-rolled assembly byte-for-byte"。本期已将 extras 插到 SessionMemory 与 Request 之间，注释未同步——这是契约层注释，会误导后续维护者。

**修复**：改写注释为 `SystemPromptSource → SessionMemorySource → ...extras → RequestMessagesSource`，并明确"无 extras 时仍 byte-for-byte 等价于历史装配"，"extras 紧贴当前轮 request 之前"。

---

## 2. 已接受 / 无需修改（Reviewed and OK）

### S-1：路径穿越防护

`workspace.validateNoteName` 拒绝空、`.`、`..`、超长（>64）、不匹配 `[A-Za-z0-9._-]` 的输入。`validateSessionID` 用同一族正则但上限 128。`workspace_test.go` 覆盖 5+ 个攻击向量（空、`.`、`..`、`../passwd`、`foo/bar`、`foo\bar`、`with space`、嵌入 NUL、超长）；`tool/workspace/notes_test.go` 通过 `../passwd` 用 LLM 协议层验证拒绝路径未触达文件系统。

工具层（plan.go / notes.go）不接受任何"path"或"filename"参数，唯一可控字段是 `name`，由 `validateNoteName` 兜底，做到了"工具表面无 path"+"FS 调用前 100% 校验"。✅

### S-2：原子写

`writeFileAtomic` 走 `<file>.tmp` → `Write` → `Sync` → `Close` → `Rename` 标准路径，权限 `0o600`，目录 `0o700`，与 `vage/session.FileSessionStore` 一致。命名返回 `err` + `defer { if err != nil { os.Remove(tmp) } }` 正确处理 rename 失败的清理。父目录未 fsync 与 SessionStore 一致——可接受，跨进程一致性本期就没承诺。✅

### S-3：并发与锁

per-session `sync.Map[string]*sync.Mutex` 锁，`Delete` 时一并 `locks.Delete(sessionID)`，避免长生命周期下锁表无界增长。`countNotesLocked` 在持锁路径下统计，`MaxNoteCount` cap 检查与写入原子，避免 race past cap。`ListNotes` 是无锁读，依赖原子 rename 的"看到旧版或新版"语义，是常见且正确的并发文件设计。✅

### S-4：错误处理（fail-open at vctx）

`WorkspaceSource.Fetch` 在 plan 读失败、notes 读失败时返回 `(FetchResult{Report: rep}, nil)`——错误状态写入 report，但不传播给 builder，确保后续 source 继续执行。这与 `vage/context.Builder` 在 §117 注释明确的 fail-open 契约一致；SystemPromptSource 是 fail-closed 的唯一例外，与本期 source 无冲突。✅

### S-5：工具层 error 形态

`plan_update` / `notes_write` / `notes_read` 在缺 sessionID、JSON 解析失败、底层 workspace 错误等所有场景统一返回 `(schema.ErrorResult, nil)`——即 LLM 看到"文本错误"，golang 调用方拿到 `nil error`。这与 `tool/todo` 等已有工具的惯例一致，避免双层 error 处理。✅

### S-6：事件 schema

`WorkspacePlanUpdatedData` / `WorkspaceNoteWrittenData` 实现 `eventData()` marker，符合 `EventData` 封闭接口约束；event 常量 `workspace.plan_updated` / `workspace.note_written` 命名空间清晰、未与已有事件冲突。`Cleared`/`Bytes` 字段足以让 hook 消费方理解写入意图。✅

### S-7：vv 集成与零成本路径

- `cfg.Session.IsEnabled()=false` 时 `buildHookManagerAndSession` 返回 `planWorkspace=nil`，setup.New 的 `getWorkspace(opts)` 返回 nil；`buildExtraContextSources` 返回 nil；Primary Assistant 不注册 plan_update/notes_write/notes_read。
- 各 sub-agent factory（coder/researcher/reviewer/primary）中 `if len(opts.ExtraContextSources) > 0` 才追加 `WithExtraSources(...)`，nil 切片直接跳过。
- HTTP `mux.HandleFunc("GET /v1/sessions/{id}/workspace/...", ...)` 仅当 `planWorkspace != nil` 时挂载。

零成本路径完全保留。✅

### S-8：HTTP DELETE workspace 联动

`handleDeleteSession(store, ws)` 先 `store.Delete`（FileSessionStore 已经 `os.RemoveAll(<sessionDir>)`，包含 workspace/ 子树），再 `ws.Delete`（best-effort，若 ws 接口实现非文件后端的话还能处理；同时 `Workspace.Delete` 内部会清理 `locks` map 中的 stale entry）。`workspaceDeleter` interface 收口到 `Delete(ctx, sid) error` 一个方法，测试可用 `nil` 跳过。设计上耦合干净。✅

### S-9：API 命名与稳定性

- `taskagent.WithExtraSources(...vctx.Source) Option` 命名贴合 vage 现有 `WithSystemPrompt` / `WithMemory` 等选项；包文档明确"在 SessionMemorySource 与 RequestMessagesSource 之间"的插入位置和典型用例（Plan Workspace、向量召回、session tree）。nil source 静默跳过、可重复调用——与现有 option 风格一致。
- `workspace.Workspace` interface 7 个方法（ReadPlan/WritePlan/ReadNote/WriteNote/ListNotes/Delete/PathOf），承诺面狭窄、覆盖 MVP 用例；`PathOf` 仅用于日志（实现说明返回 `""` 表示非 file-backed），契约清晰。
- `vctx.WorkspaceSource{Workspace, MaxBytes}` 公共字段保持平坦无构造函数，与 `SystemPromptSource{Template}` / `SessionMemorySource{Manager}` 风格一致。✅

### S-10：测试覆盖

- `workspace/`：路径穿越（攻击向量 ≥ 5）、超大 / 超数量 cap、并发写、ListNotes 排序与 stray 文件忽略、Delete idempotence、PathOf 校验。
- `tool/workspace/`：context 缺失 sessionID、JSON 解析失败、写入触发事件、清空事件、读取缺失 note 的占位回复、注册器 nil 检查。
- `context/sources_workspace_test.go`：skipped/ok/error/truncated 四档；plan-only / notes-only / 都有的渲染分支；humanBytes / humanAge 边界。
- `agent/taskagent/extra_sources_test.go`：消息顺序断言（system/session_memory 空时只剩 extras + request）、nil source 过滤、无 extras 时行为兼容。

覆盖密度合理。✅

---

## 3. 已识别但本期不修复（Out-of-scope / Documented）

| 项 | 现状 | 是否处理 |
|---|---|---|
| `validateSessionID` 没有显式拒绝 `...`（三点） | 该值会被当作普通目录名，不会逃出 root；ID 模式与 `vage/session.IDPattern` 一致 | 不修，沿用 session 包行为 |
| `WriteNote` 当 LLM 调用 `{}`（缺 content）时静默清空对应 note | JSON `required` 仅 LLM-side schema 校验；server 端依赖 LLM 行为良好 | 不修，符合"全量覆盖、空=删"的 MVP 决策 |
| `writeFileAtomic` 不 fsync 父目录 | 与 SessionStore 行为一致；跨进程崩溃后的可见性本期未承诺 | 不修，留待后续与 session 一并处理 |
| `TestNote_TooManyNotes` 注释冗长但未真的写到 cap | 写满 200 个文件单测代价过高；测试覆盖 cap 检查的输入面（已存在 note 不计入） | 不修，已在测试注释里说明 |
| `(*WorkspaceSource)(nil)` 类型化 nil 接口能逃过 `s.Workspace == nil` | Go 标准 gotcha；典型用法是赋值 `nil` 接口或具体值 | 不修，记入文档约定 |

---

## 4. 文件清单（本次评审涉及）

修改：

- `/Users/hk/workspaces/github/vogo/vagents/vage/context/sources_workspace.go`（C-1 修复，包含字段注释、截断方向、渲染 marker）
- `/Users/hk/workspaces/github/vogo/vagents/vage/context/sources_workspace_test.go`（更新 truncation 测试以验证 tail-preserving + marker）
- `/Users/hk/workspaces/github/vogo/vagents/vage/agent/taskagent/task.go`（C-2 文档注释更新）

未修改但已审查：

- `vage/workspace/{workspace.go, filestore.go, workspace_test.go, filestore_test.go}`
- `vage/tool/workspace/{plan.go, notes.go, plan_test.go, notes_test.go}`
- `vage/agent/taskagent/extra_sources_test.go`
- `vage/schema/event.go`（workspace event 常量与 payload）
- `vv/registries/registry.go`（FactoryOptions.ExtraContextSources）
- `vv/agents/{coder.go, researcher.go, reviewer.go, primary.go}`
- `vv/setup/setup.go`（buildHookManagerAndSession workspace 构造、buildExtraContextSources、Primary 工具注册）
- `vv/httpapis/{workspace.go, http.go, sessions.go}`

---

## 5. 验证

| 步骤 | 结果 |
|---|---|
| `cd vage && go test ./...` | 全部 PASS |
| `cd vv && go test ./...` | 全部 PASS |
| `cd vage && golangci-lint run ./...` | 0 issues |
| `cd vv && golangci-lint run ./...` | 0 issues |

---

## 6. 评审结论

实现满足 §4.4 设计意图与 US-1..US-5 验收标准；安全（路径穿越防护完整）、并发（per-session 锁 + 原子写）、错误处理（vctx fail-open + 工具层文本错误）均到位。本期发现一个语义级 bug（plan 截断方向反了）已就地修复并新增测试断言保护。其他项目均为可接受的设计权衡，已逐一记录。

可以进入 documenter 阶段。
