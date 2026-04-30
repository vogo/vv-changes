# 需求：Session Tree promotion + 折叠 与 vv 端 wiring 合并落地（剩余功能）

## 背景

`doc/design/session-context-solution.md` 已经记录了 Session Tree 演进路线。截止本期前，已完成：

- A1–A5 (vage 框架层 promotion + 折叠核心) — 2026-04-30-004 已落地
- 数据模型、Store 接口扩展、Promoter、PromotionDecider、`SessionTreeSource.IncludePromoted` 与 `(folded: N children, M done)` 渲染均已实现并 ✅

**本期目标**：实现设计文档第 754 行起 "promotion + 折叠 与 vv 端 wiring 合并后的核心方案" 中所有**剩余**功能：

- A6：`vage/tool/sessiontree/` LLM 工具包（tree_add / tree_update / tree_cursor / tree_promote / tree_zoom_in）
- B1：vv 配置（`SessionTreeConfig`）+ 环境变量覆盖
- B2：vv `setup.Init` 构造 store / promoter / decider，把 tree 传到 Result
- B3：vv CLI 新增 `--tree show <id>` 子命令
- B4：vv HTTP `/v1/sessions/{id}/tree*` 路由
- B5：vv agent 注册 — Primary 注册 5 个 tree 工具 + 所有 dispatchable agent 注入 SessionTreeSource
- B7：Primary system prompt 增加 SessionTree 使用约定
- B8：文档与 PRD（vv/CLAUDE.md, doc/prd, 标记设计文档完成）

**纳入本期** (用户追加)：
- B6：Dispatcher 自动写树 — `dispatcher.write_tree` feature flag，启用时 planner 输出 plan 路径会把 plan 步骤写为 SessionTree 子节点

**out of scope**（明示跳过）：
- B9 中的 e2e LLM 集成测试（受限于 LLM API key；写 HTTP unit/integration 测试）
- 双索引（向量召回与 tree 联动）
- 跨 session 树森林
- SQLite 后端、跨进程文件锁

## 用户故事

1. 作为 vv 用户，我可以在 `vv.yaml` 中通过 `session_tree.enabled` 开关 SessionTree 子系统。开启后：
   - HTTP `GET /v1/sessions/{id}/tree` 返回当前会话的 SessionTree（默认隐藏 promoted；`?include_promoted=1` 显示全部）。
   - HTTP `POST /v1/sessions/{id}/tree` 创建 root 节点。
   - 节点级 CRUD 通过 `/tree/nodes/{nid}` 路径，兼有 `POST /tree/cursor` 与 `POST /tree/promote/{nid}`。
   - DELETE 整棵树通过 `DELETE /v1/sessions/{id}/tree`，DELETE 整 session 已经联动清理（共享根目录）。

2. 作为 LLM Primary 调用方，我可以通过工具操作树：
   - `tree_add` / `tree_update` / `tree_cursor` / `tree_promote` / `tree_zoom_in`
   - 工具风格与 plan_update / notes_write 一致：自 ctx 取 sessionID，参数无 path，事件结构化。

3. 作为 dispatchable sub-agent 使用方，read-only 注入 SessionTree 到 prompt 顶部（已通过 ExtraSources 路径），sub-agent 不写树。

4. 作为运维，关闭 `session_tree.enabled` 时：tools 不注册、source 不注入、HTTP 路由不挂载，保持零开销。

## 验收标准（可验证）

| # | 验收点 | 验证方式 |
|---|---|---|
| AC-1 | `session_tree.enabled=false` 时整套配置零开销，HTTP 路由不挂载 | Init 单测 + HTTP 路由列表断言 |
| AC-2 | `session_tree.enabled=true` 时 setup 构造 `tree.FileTreeStore`（共用 sessions 根） | Init 单测 |
| AC-3 | promoter=llm/compressor/noop 三种模式均可构造 | unit test |
| AC-4 | A6 五个工具均按 plan_update 风格事件化，参数 schema 有效 | unit test |
| AC-5 | Primary 启用工具，coder/researcher/reviewer 不启用工具但有 source 注入 | unit test on tool registry / agent ExtraSources |
| AC-6 | HTTP 路由 happy path：GET tree (含 include_promoted)、POST root、POST nodes、PATCH/DELETE node、POST cursor、POST promote/{nid}、DELETE tree | httpapis_test |
| AC-7 | 非法 sessionID / nodeID / 越界值返回 400 | httpapis_test |
| AC-8 | DELETE 整 session 时 tree 自然消失（共享目录清理） | 已存在 SessionStore 测试覆盖；新增 tree-aware delete 流验证 |
| AC-9 | CLI `vv --tree show <id>` 输出 SessionTreeSource 同款渲染 | cli_test |
| AC-10 | 设计文档 §4.8.6 第 2、4 步打 ✅；§8 差距汇总更新；新增 PRD 文档 | 文档级别变更 |

## 影响面

- **vage 框架层**：新增 `vage/tool/sessiontree/` 包；schema 已有事件无需改动。
- **vv 应用层**：新增 `vv/configs/SessionTreeConfig`、`vv/setup/` 中 tree 构造、`vv/httpapis/tree.go`、`vv/cli/tree.go`、Primary 与 sub-agent 注册更新、prompt 微调。
- **PRD/设计文档**：`doc/prd/applications/api/pages/core/sessions/tree.md`、`doc/prd/models/core/session/model-session-tree.md`、`model-tree-node.md`，标记 §4.8.6 与 §8。

## Out of scope（明示）

- B6 dispatcher 自动写树 — 留待后续灰度
- vector recall 与 tree 双索引联动 — 留待后续
- 跨 session 父子树森林 — 留待后续
- SQLite/Postgres 后端、跨进程锁 — 留待后续
- 真实 LLM 驱动的 e2e tree 测试 — 留待 integrations 层补
