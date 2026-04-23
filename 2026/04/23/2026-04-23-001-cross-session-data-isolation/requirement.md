# Requirement: 跨会话数据强隔离（Namespace 绑定 SessionID）— P1-4

## 1. 背景与目标

### 背景

vv 的持久化内存（`vv/memories/filestore.go`）当前以命名空间（namespace）为唯一组织维度：

1. **Key 格式**：`"<namespace>:<key>"`，例如 `"project:conventions"`。
2. **文件布局**：`<memory_dir>/<namespace>/<key>.json`。
3. **命名空间取值**：运行时由调用方（Agent、CLI 用户、HTTP 客户端）任意自由指定，无任何校验。

这会带来三类潜在问题：

1. **软约束**：调用方可自选 namespace 字符串。一个运行中的 Agent 能够写入、读取、甚至删除任何 namespace（含他人）的条目。
2. **自学习风险**：P3-1 ~ P3-4（自学习闭环）计划让后台子 Agent 在会话结束后依据对话轨迹自动生成 Skill/记忆。若 `session_A` 生成的草稿误落入 `session_B` 的读取范围，会导致跨会话泄漏、污染 Skill，或让模型根据他人上下文做决策。
3. **跨会话知识与会话私有数据界限模糊**：`project` / `user` / `conventions` / `notes` 四个内置 namespace 的语义是"跨会话共享"，但代码层没有任何机制把它们与"仅本会话可见"的草稿/中间态数据区分开来。

原 `feature-todo.md` 明确标注该需求为：

> 当前 namespace 为软约束；改为 sessionID 强绑定 + 读写校验；自学习生成 Skill 前必须就位。

### 目标

在保持 vv "单用户本地工具"定位不变、对现有 CLI/HTTP/4 个共享 namespace 零破坏的前提下，为持久化内存引入**基于 SessionID 的访问控制**，使：

- 由 Agent（通过工具调用）写入的"私有数据"自动绑定到当前 SessionID，跨 Session 默认不可访问。
- 由 User（通过 `/memory` CLI 指令或 `PUT /v1/memory`）直接管理的"共享知识"仍然跨会话可用。
- 一切违规的跨会话读写都返回明确错误，而非静默成功或静默空结果。
- 该机制是 P3 自学习闭环的前置条件：后台 Agent 在 `session_A` 生成的草稿 Skill 不会被 `session_B` 无意消费。

## 2. 行业与类似产品调研（2026-04-23）

### 2.1 AI Agent 生态：记忆/会话隔离现状

| 产品 | 记忆单元 | 隔离维度 | 访问控制 | 可借鉴点 |
|---|---|---|---|---|
| **Mem0 (2025)** | `memory` 对象 | `user_id` + `agent_id` + `run_id` 三级 tag | API 入参必填 `user_id`；服务端按 tag 过滤检索 | **三级 tag 模型** — 比单 namespace 更细；vv 可对应 `project / session_id / run_id` |
| **Zep Graphiti** | 图节点 | 按 `group_id`（≈ tenant）分区 | 检索 API 强制 `group_id` 参数 | **强制分区参数** — API 层面不给绕过机会 |
| **LangGraph Checkpointer** | state snapshot | `thread_id` 作为主键 | 按 `thread_id` 查询；不同 thread 互不可见 | **thread_id 作为存储键前缀** — 天然物理隔离 |
| **Claude Code `/memory`** | `CLAUDE.md` 文件 | 按文件路径（项目/用户两级） | 启动时按 cwd 自动加载项目级 + 全局用户级 | **作用域分级** — 项目 = 共享、用户 = 账户全局；vv 可复用此分级语义 |
| **ChatGPT Memory (user memory)** | `(user_id, memory_id)` 条目 | `user_id` 主键强制 | 任何查询必须带 `user_id`；模型看不到他人条目 | **session_id/user_id 作为一等公民存进 record** |
| **Cursor "Rules"** | `.cursorrules` 文件 | 项目目录隔离（仓库边界） | 文件系统级天然隔离 | 证明"用目录边界当隔离边界"是最朴素也最稳健的路径 |
| **Cline `.cline/` 目录** | 项目内目录 | 项目目录隔离 | 同上 | 同上 |
| **OpenAI Assistants v2** | `thread_id` + `assistant_id` | API 入参必填 `thread_id` | 服务端按 `thread_id` 完全隔离 | **API 不给 "无 thread_id" 的接口形态** |

**关键观察**：所有主流方案都把会话/线程 ID 作为记忆写入时的**一等公民存储字段**，而不是让调用方"约定"命名空间前缀。访问时按 ID 过滤是"默认拒绝"、共享是显式例外。

### 2.2 SaaS / 多租户数据库隔离方案对照

| 方案 | 代表案例 | 核心机制 | 成本 | 泄漏风险 |
|---|---|---|---|---|
| **分表/分库** | Shopify Plus、Slack Enterprise | 每租户独立表/库 | 高运维成本 | 最低 |
| **Row-Level Security (RLS)** | PostgreSQL RLS、Supabase | 每行带 `tenant_id`，SQL 策略自动过滤 | 低，查询无感 | 策略漏写即泄漏 |
| **Tenant-prefixed Key** | Redis、DynamoDB、S3 prefix | Key = `<tenant_id>:<key>` | 极低 | 中 — 易被遗漏前缀 |
| **中间件拦截** | Django tenant schemas、Rails multitenant | Web framework middleware 自动注入 `tenant_id` 条件 | 低，应用层透明 | 低 — 与业务代码解耦 |

**vv 的对应**：FileStore 本质是 key-value 存储，行业通用套路是**"Key 前缀 + 入口层强制注入 tenant_id"**。vv 的 Session 对应租户，但因为是单用户场景，"tenant_id" 不用来分账只用来做一致性隔离。

### 2.3 常见陷阱（从上述方案总结）

| 陷阱 | 典型案例 | 在 vv 场景的表现 |
|---|---|---|
| **Key 前缀被客户端伪造** | Redis 多租户中恶意客户端手工拼 `other_tenant:key` | Agent 通过工具调用时自行指定 `namespace`，绕过 session_id 绑定 |
| **List/Scan 漏过滤** | RLS 策略只覆盖 SELECT 忘了 JOIN | `List(prefix="")` 返回全部条目 |
| **共享/私有边界模糊** | 共享字典和私有数据混用一个 key 空间 | 已有 `project/user/...` 共享 + 新引入的会话私有数据 |
| **遗留数据迁移缺失** | 租户化改造上线后老数据"无主" | 现存 FileStore 文件没有 session_id 字段 → 默认归谁？ |
| **CLI/HTTP 用户身份 ≠ Agent 身份** | 管理员工具看得到全量，LLM 应用看自己 | CLI `/memory set` 是用户操作（应可访问共享）；Agent 调用未来的 `write_memory` tool（应限制到当前 session） |
| **TOCTOU / 删除泄漏** | 删除时用元数据校验，读时只比前缀 | Delete 校验 session_id 对即可，但 List 可能漏 |

### 2.4 注意事项汇总

1. **强制注入而非乐观约定**：session_id 必须由框架层注入，不能信任 Agent 传入的字符串。
2. **默认拒绝，显式共享**：未匹配 session_id 的私有条目应报错；共享 namespace 需显式 allowlist。
3. **一致性写入**：Set 写入时必须带 session_id；文件名/路径/元数据三者不能冲突。
4. **覆盖全部读写入口**：Get / Set / Delete / List / Clear 五个方法都要过校验。
5. **向后兼容**：现存条目没有 session_id → 需要"legacy treated as shared" 或迁移回填；一次性升级不能让用户丢数据。
6. **显式区分"用户入口"与"Agent 入口"**：同一份 Memory 接口被两类调用者使用，需要在上层（CLI、HTTP、Tool Handler）决定是否传 session_id。

## 3. 用户故事与验收标准

### US-1：Agent 写入的数据默认会话私有

**作为** 会话 A 内的 Coder Agent，**当我** 通过未来的 `write_memory` 工具写入一条记忆，**那么** 这条记忆应该只能由会话 A 的 Agent 读到，会话 B 的任何 Agent 都无法看到/覆盖/删除它。

**验收**：
- 会话 A 以 session_id=`A` 写入 `session:draft-skill-1` → 存储层记录 session_id=`A`。
- 会话 B（session_id=`B`）Get `session:draft-skill-1` → 返回 `not found`，不抛出敏感元数据。
- 会话 B List(prefix=`"session:"`) → 不包含会话 A 的条目。
- 会话 B Delete/Set 同一 key → 返回 `forbidden` 错误；磁盘上会话 A 的文件不变。

### US-2：共享 Namespace 跨会话可访问

**作为** 任意会话内的 Agent 或 CLI 用户，**当我** 读取 `project`、`user`、`conventions`、`notes` 四个共享 namespace 中的条目，**那么** 无论它们由哪个会话写入，都应该能正常读取。

**验收**：
- 会话 A 写入 `project:arch` → 会话 B 读取 `project:arch` 成功返回。
- 共享 namespace 的写入同样跨会话可见；可由 CLI 用户或 Agent 写入。
- 共享 namespace 存储层不强制记录 session_id（可记录但不参与访问校验）。

### US-3：User 级操作跳过 Session 绑定

**作为** 使用 `/memory set project:arch ...` 或 `PUT /v1/memory/project/arch` 的 User（而非 Agent），**当我** 操作共享 namespace，**那么** 不应被 session_id 访问控制拦截。

**验收**：
- CLI `/memory` 命令调用内存接口时走"用户路径"，跳过 session 校验；仅限 4 个共享 namespace。
- CLI `/memory set session:foo bar` → 被拒绝（会话私有 namespace 不允许用户通过 CLI 跨会话写入，避免用户误操作污染会话私有空间）。
- HTTP `PUT /v1/memory/session/foo` → 返回 403（同上）。

### US-4：遗留数据兼容

**作为** 升级到新版本前已经在 `~/.vv/memory/` 里存了条目的用户，**当我** 启动新版本，**那么** 已有条目不能消失或变成无法访问。

**验收**：
- 遗留 FileStore 文件（无 `session_id` 字段）在 4 个共享 namespace 中 → 正常读取。
- 遗留文件若出现在非共享 namespace 中（不期望，但可能存在）→ 视为"legacy shared"、可读但不可被任何 Agent 写入覆盖，可由 CLI 手动迁移或删除。
- 不做强制迁移；首次升级即用即能。

### US-5：违规操作返回清晰错误

**作为** 排查问题的开发者，**当** 某次 Set/Get/Delete 因 session 校验失败，**那么** 错误信息应明确指出 `expected_session`、`actual_session`、`namespace` 等字段，方便定位。

**验收**：
- 错误类型可以被 `errors.Is(err, memories.ErrSessionForbidden)` 判断。
- slog 日志记录违规尝试，包括 namespace、key、当前 session_id；**不记录值内容**。

### US-6：List 与 Clear 正确过滤

**验收**：
- `List(ctx, prefix)` 在有 session 上下文时，仅返回：(a) 共享 namespace 条目 + (b) 当前 session_id 的条目。
- `Clear(ctx)` 在有 session 上下文时，仅清除当前 session 的私有条目 + 共享 namespace（或保持现有"清全部"语义并通过文档警告——待 US-6 设计阶段确定）。

### US-7：自学习路径就位

**作为** 未来 P3 自学习子系统，**当我** 以 `session_id=A` 生成草稿 Skill 写入 `session:draft-skill-xyz`，**那么** 该条目不能在后续的 `session_B` 中被检索到，直到被"提升"为共享 namespace 下的正式 Skill。

**验收**：US-1 全部验收条件成立即可。

## 4. 范围

### 4.1 In-Scope

1. **存储层扩展**：`vv/memories/FileStore` 的 `fileRecord` 增加 `session_id` 字段。
2. **访问控制层**：在 FileStore 五大接口（Get/Set/Delete/List/Clear）上加 session 上下文校验。
3. **入口层上下文注入**：通过 Go `context.Context` 向下传递 session_id；定义 `SessionContextKey` / `WithSessionID` / `SessionIDFrom` helpers。
4. **Session 绑定策略**：
   - 共享 namespace allowlist（`project / user / conventions / notes`，可配）。
   - 私有 namespace 写入自动带 session_id；读时强制匹配。
5. **CLI / HTTP 入口改造**：
   - CLI `/memory` 走"用户路径"：不注入 session_id。
   - HTTP `GET/PUT/DELETE /v1/memory/{namespace}/{key}` 走"用户路径"。
   - 未来 Agent 工具（`write_memory` 之类）走"Agent 路径"：强制注入当前 session_id。
6. **错误类型**：`memories.ErrSessionForbidden`；入口层对其进行合理映射（CLI 提示 / HTTP 403）。
7. **测试**：单元覆盖所有跨会话读写路径；集成测试覆盖 CLI/HTTP 不受影响。
8. **文档**：PRD 的 `dictionary-memory-namespace.md`、`procedure-persistent-memory-management.md`、`model-memory-entry.md` 同步更新。

### 4.2 Out-of-Scope（明确不做）

1. **"Agent 写入工具"本体**：本需求只做隔离骨架，不新增 Agent 级 `write_memory` 工具——该工具是 P3-3 的工作。
2. **加密/ACL**：内存不加密，不引入 ACL 模型（P4-10）。
3. **多用户/多租户**：仍然是单用户本地工具；session_id 仅用于一致性隔离，不做计费或权限划分。
4. **SQLite 迁移**：FileStore 仍然是 JSON 文件；SQLite 迁移是 P1-6。
5. **已有 namespace 语义变更**：不改动 `project/user/conventions/notes` 四个 namespace 的现有行为，仅把它们加进共享 allowlist。
6. **清空语义变更**：`Clear` 的行为若与现有测试冲突，优先保持现有语义并在设计阶段给出兼容策略；不做激进改变。

## 5. 受影响角色、模型、流程、应用

| 类型 | 条目 | 影响 |
|---|---|---|
| 角色 | Agent（Coder/Researcher/Reviewer/Chat） | 通过 context 注入的 session_id 被框架层识别 |
| 角色 | User（CLI、HTTP） | 现有 `/memory` 命令与 `/v1/memory/*` API 行为保持不变 |
| 模型 | Memory Entry | 新增 `session_id` 属性 |
| 字典 | Memory Namespace | 新增 "shared / session-private" 分类标注 |
| 流程 | Persistent Memory Management | Set/Get/Delete/List 步骤加 session 校验 |
| 应用 | CLI `/memory` 指令 | 进入"用户路径"；禁写 session-private namespace |
| 应用 | HTTP `/v1/memory/*` | 进入"用户路径"；禁写 session-private namespace |

## 6. 成功标准（可验证）

1. 全部 US-1 ~ US-7 验收条件通过。
2. 新增单元测试覆盖：session match、session mismatch、shared namespace、legacy entry、List 过滤、Clear 范围。
3. 现有 `vv/memories/filestore_test.go` 全部保持通过；现有 `vv/integrations/http_tests` 全部通过。
4. 无磁盘上的条目在升级后丢失。
5. `lint + test` 在 `vv/` 模块内全绿。

## 7. 已识别的开放问题（交给 Designer 决策）

1. **session_id 注入点选择**：在 `PersistentMemory` wrapper 层注入还是在 `FileStore` 层校验？—— 倾向 Store 层（单点真相），wrapper 层透传 context。
2. **Clear 语义**：会话级 Clear 还是全局 Clear？—— 倾向保留全局 Clear 但仅在"用户路径"允许；Agent 路径 Clear 返回 forbidden。
3. **Legacy 条目判定**：无 `session_id` 字段是否等价于 `"*"` 通配？—— 倾向将其视为 shared-only（即仅限共享 namespace 的遗留条目才可读），非共享 namespace 的 legacy 条目按安全默认"禁止写覆盖"处理。
4. **session-private namespace 的名称空间**：使用保留前缀 `session:` 还是按配置允许任意非共享 namespace？—— 倾向"任意非共享 namespace = 私有"，以减少对 Agent 命名的约束，但在 dictionary 里添加说明。
5. **错误暴露粒度**：Get 命中但 session 不匹配时，返回 `not_found` 还是 `forbidden`？—— 倾向 `not_found`，避免枚举泄漏；在 slog 日志中记录真实原因。
