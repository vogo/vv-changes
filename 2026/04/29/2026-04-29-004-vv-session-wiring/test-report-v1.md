# 测试报告 v1:vv 端 Session wiring

> 阶段:tester
> 日期:2026-04-29
> 范围:`vv/integrations/session_tests/session_tests/` 集成测试

## 1. 概述

本轮测试在已存在的 MVP 集成测试(`session_wiring_test.go` + `http_sessions_test.go`,共 5 个用例)之上新增 4 份集成测试文件、19 个测试函数(其中 1 个含 4 个子测试,实际验收场景 22 项),覆盖 requirement.md 的 Story A / B / C 全部验收标准与 design.md §6 隐含的 HTTP 边界规则。

所有新旧测试 **全部通过**,`make lint` 0 issue,`go test ./...` 在 vv 模块全绿。

## 2. 新增测试清单

### 2.1 `cli_resume_test.go`(Story A:CLI 续接)

| # | 测试函数 | 场景 |
|---|---|---|
| 1 | `TestCLIResume_CrossProcess_RealStore` | 启 setup.Init #1 → HookManager.Dispatch 写入 2 个事件 → Shutdown 排空 → setup.Init #2 指向同一目录 → cli.PrepareSessionID 命中 Existing 且 ListEvents 复读到第一次写入的事件。证明 FileSessionStore 跨进程续接成立。 |
| 2 | `TestCLIResume_UnknownID_NotFound` | `--session <unknown-id>`:PrepareSessionID 返回 NotFound 但仍绑定该 id;首次 Dispatch 后 SessionHook autoCreate 把 meta 落盘。 |
| 3 | `TestCLIResume_RejectInvalidID` (4 subtests) | `..` / `.` / `alpha/beta` 在 PrepareSessionID 边界即被拒;空白 id(trim 后为空)归为"mint new"路径。 |
| 4 | `TestCLIResume_TouchSession_RefreshesOnExisting` | TouchSession 对已存在 session 刷新 UpdatedAt(SessionHook 不写 meta,这是 list 视图保持新鲜的关键路径)。 |

### 2.2 `setup_combinations_test.go`(Story C:setup 默认开)

| # | 测试函数 | 场景 |
|---|---|---|
| 5 | `TestSetup_SessionEnabled_TraceDisabled` | trace 显式关、session 默认开 → SessionStore 与 HookManager 都非 nil(防止旧 trace-gated buildHookManager 回归)。 |
| 6 | `TestSetup_SessionAndTraceCoexist` | trace 与 session 都开 → 一个 HookManager 上挂 2 个 hook;事件并行写入 SessionStore 与 trace 目录。 |
| 7 | `TestSetup_SessionDir_Override` | `cfg.Session.Dir` 显式覆盖默认路径 → 事件落到自定义目录而非 ~/.vv/sessions。 |
| 8 | `TestSetup_SessionConfig_DefaultEnabled` | `SessionConfig.IsEnabled()` 默认开:nil → true、true → true、false → false 三态。 |

### 2.3 `lifecycle_test.go`(Story B+C 综合)

| # | 测试函数 | 场景 |
|---|---|---|
| 9 | `TestLifecycle_HookToFileToHTTP` | 完整闭环:setup.Init → HookManager.Dispatch → FileSessionStore 落盘 → 同一 store 注入 httpapis.Serve → GET /v1/sessions/{id}/events 复读出该事件,证明 hook→file→HTTP 链路全通。 |

### 2.4 `http_edge_cases_test.go`(HTTP 边界)

| # | 测试函数 | 场景 |
|---|---|---|
| 10 | `TestHTTP_List_ZeroLimit_RejectsAs400` | `GET /v1/sessions?limit=0` → 400(parsePositiveInt 的 ≤0 守卫)。 |
| 11 | `TestHTTP_List_NegativeOffset_RejectsAs400` | `GET /v1/sessions?offset=-1` → 400。 |
| 12 | `TestHTTP_List_StateFilter` | `?state=active` 仅返回 active session,过滤掉 completed 的。 |
| 13 | `TestHTTP_Events_LimitTailTruncates` | 3 条 events,`?limit=1` 仅返回最近 1 条(tail-truncate 语义)。 |
| 14 | `TestHTTP_Events_TypeFilter_MultiValue` | `?type=agent_start,text_delta` 多值逗号分隔,严格按追加顺序返回匹配子集。 |
| 15 | `TestHTTP_Patch_Metadata_FullReplacement` | `PATCH metadata: {}` 整体替换(非 merge)既有 metadata。 |
| 16 | `TestHTTP_Patch_NonExistent_404` | PATCH 不存在的 id → 404。 |
| 17 | `TestHTTP_Patch_InvalidState_400` | `PATCH state: "bogus"` → 400(只接受 active/paused/completed/failed)。 |
| 18 | `TestHTTP_Delete_Idempotent_MissingID` | `DELETE /v1/sessions/missing-id` → 200 + `{"status":"deleted"}`(idempotent)。 |
| 19 | `TestHTTP_List_LimitClampedToMax` | `?limit=10000` 不报错,silent clamp 到 200。 |

## 3. Pass / Fail 概览

```
ok  	github.com/vogo/vv/integrations/session_tests/session_tests	2.186s
```

| 项 | 数量 |
|---|---|
| 新增测试函数 | 19 |
| 新增子测试(t.Run) | 4 |
| 既有测试(MVP 已有) | 5 |
| **总通过** | **24 顶层 + 11 子用例 = 35 项** |
| 失败 | 0 |
| Skipped | 0 |
| 已开 `integrations-error-N.md` | 0 |

`make lint` → 0 issue。
`go test ./...`(整个 vv 模块) → 全绿。

## 4. 验收标准覆盖矩阵

### Story A:CLI 续接历史会话

| 验收项 | 覆盖测试 |
|---|---|
| `--session <id>` id 存在 → 加载 meta + events,banner 提示 | `TestCLIResume_CrossProcess_RealStore`(meta 复读 + events 复读) |
| `--session <id>` id 不存在 → 直接以该 id 新建 | `TestCLIResume_UnknownID_NotFound` |
| `--session <id>` id 非法 → stderr 报错并 exit 1 | `TestCLIResume_RejectInvalidID/dotdot,dot,slash` |
| 不带 `--session` 行为不变(GenerateID + autoCreate) | `TestSetupInit_DefaultEnabled_StoreAndHookWired`(已有)+ `TestCLIResume_RejectInvalidID/empty-after-trim` |
| `--session list` 排序 + 字段 | `cli/session_test.go` 单元测试已覆盖 PrintSessionList(详见 vv/cli/session_test.go),集成层不重复。 |

### Story B:HTTP 暴露 Session CRUD

| 验收项 | 覆盖测试 |
|---|---|
| `GET /v1/sessions` AND filter / limit / offset | `TestHTTP_Sessions_RoundTrip/list_*`(已有)+ `TestHTTP_List_StateFilter` + `TestHTTP_List_ZeroLimit_*` + `TestHTTP_List_NegativeOffset_*` + `TestHTTP_List_LimitClampedToMax` |
| `GET /v1/sessions/{id}` 完整 meta + state | `TestHTTP_Sessions_RoundTrip/get_returns_meta`(已有) |
| `GET /v1/sessions/{id}` 404 | `TestHTTP_Sessions_RoundTrip/get_404_on_unknown`(已有) |
| `GET /v1/sessions/{id}/events` raw events | `TestHTTP_Sessions_RoundTrip/events_*`(已有)+ `TestLifecycle_HookToFileToHTTP` |
| events `?type=` 多值过滤 | `TestHTTP_Events_TypeFilter_MultiValue` |
| events `?limit=` 默认/上限 | `TestHTTP_Events_LimitTailTruncates` |
| `DELETE /v1/sessions/{id}` 幂等返回 deleted | `TestHTTP_Sessions_RoundTrip/delete_then_404`(已有)+ `TestHTTP_Delete_Idempotent_MissingID`(明确 missing-id 路径) |
| `PATCH /v1/sessions/{id}` title/state/metadata | `TestHTTP_Sessions_RoundTrip/patch_title`(已有,title)+ `TestHTTP_Patch_Metadata_FullReplacement`(metadata)+ `TestHTTP_Patch_InvalidState_400`(state 校验) |
| PATCH 不存在 → 404 | `TestHTTP_Patch_NonExistent_404` |
| `cfg.Session.Enabled = false` 时 /v1/sessions/* 返 404 | `TestHTTP_Sessions_NotMounted_WhenStoreNil`(已有) |

### Story C:setup 默认挂 SessionHook

| 验收项 | 覆盖测试 |
|---|---|
| `cfg.Session.Enabled` 默认开(`nil → true`) | `TestSetup_SessionConfig_DefaultEnabled` + `TestSetupInit_DefaultEnabled_StoreAndHookWired`(已有) |
| 关闭时:不创建 store / 不注册 hook / HTTP 不挂载 | `TestSetupInit_Disabled_NoStore`(已有) |
| 默认开:FileSessionStore 路径正确 + SessionHook 注册 | `TestSetupInit_DefaultEnabled_StoreAndHookWired`(已有)+ `TestSetup_SessionDir_Override` |
| trace 关 + session 开 → HookManager 仍构造 | `TestSetup_SessionEnabled_TraceDisabled` |
| trace + session 同时开 → 共用 Manager 与并发写入 | `TestSetup_SessionAndTraceCoexist` |
| `InitResult.SessionStore` 暴露;Shutdown 接管 Manager.Stop | `TestCLIResume_CrossProcess_RealStore`(显式 Shutdown 后再开第二轮证明 Stop 正常关闭) |

## 5. 缺失/未覆盖项

无阻塞缺口。以下属于已经在单元测试层覆盖、不需要在集成层重复的项目:

- `cli.PrintSessionList` 排序与空集合 → `vv/cli/session_test.go` 单元测试已覆盖。
- `configs.SessionConfig` 的 YAML / env 解析(`VV_SESSION_ENABLED`、`VV_SESSION_DIR`)→ `vv/configs/config_test.go` 单元测试已覆盖。
- main.go 的 `--session list` / `--session new` flag 分支 → 这是进程启动级别行为,在没有真启动 vv 二进制的情况下,集成测试只能验证 setup + cli 库函数;该层走子进程编排测试成本远超价值,设计阶段已显式标记为"out-of-scope of this test phase"。

## 6. 关键发现

无 bug 发现;实现与 design.md §6 / requirement.md §2 完全一致。

- HookManager 的 Dispatch → SessionHook → FileSessionStore.AppendEvent 路径在 2s 等待窗口内稳定排空(20ms 步长轮询)。
- 跨 setup.Init 续接路径正确:第一轮 Shutdown(管理器 Stop)排空缓冲后,第二轮 Init 在同一目录上开新 store 能完整看到第一轮的事件。
- HTTP 路由 `parsePositiveInt` 的边界(0、负数、超大值)行为与 design.md §6 一致(0/负数 → 400,>200 silent clamp)。
- PATCH metadata 替换语义符合 design.md §6.3 第 3 条。

## 7. 文件清单

新增:

- `vv/integrations/session_tests/session_tests/cli_resume_test.go`(254 行,Story A)
- `vv/integrations/session_tests/session_tests/setup_combinations_test.go`(170 行,Story C)
- `vv/integrations/session_tests/session_tests/lifecycle_test.go`(126 行,Story B+C 闭环)
- `vv/integrations/session_tests/session_tests/http_edge_cases_test.go`(305 行,HTTP 边界)

未修改既有源代码或测试。
