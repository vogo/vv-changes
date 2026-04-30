# 复杂度评估 + 阶段裁剪

## 决策

| 阶段 | 决定 | 理由 |
|---|---|---|
| improver | **跳过** | 设计已遵循 vage 既有模式(workspace / vector / vctx),无新架构、无算法决策、无多解;接口直接照 §4.8 草案 + 已落地姊妹包(session/workspace/vector)取舍。本期 MVP 范围明确。 |
| reviewer | **包含** | 新增 ~3 个 Go 包 / ~10 个文件,涉及 schema 公共 API(EventSessionTreeUpdated)、context 新 Source(可被外部组装)、并发文件 IO + atomic write;diff 体量预计 50+ 行非样板;走一遍 reviewer 兜底正确性、并发、错误处理。 |
| tester | **跳过** | 本期单元测试由 developer 内嵌完成(每个 .go 配 `_test.go`);设计中明示"不引入 integration tests"——没有跨进程真实场景需要外测。tester 角色覆盖的"acceptance criteria via integration tests"在本期不适用。 |

## 有效流水线

`analyst → designer → developer → reviewer → documenter`

## 决策分析(对照 metalclaw-dev-flash 的 Complexity Assessment)

**improver 跳过依据**:

- 不引入新架构模式(沿用 vctx Source / workspace 后端 / session.cloneSession 风格)
- 无并发/性能/安全决策——文件 store 直接复制 workspace.writeFileAtomic 模式
- 数据模型新增,但边界清晰且可逆(单文件 JSON,可推翻重做)
- 本期范围 ≤ 6 个新文件 + 1 个 schema 字段
- 没有"多解可选"——节点类型 / 接口形状已被 §4.8 草案固定
- 设计 review 命中点(MaxNodes / 不可变 Parent / 不内置 vector) 已在设计文档中明示

**reviewer 包含依据**:

- 触及 schema 包公共类型(EventSessionTreeUpdated 落入 wire format)
- 新增可被外部用户组合的 Source(SessionTreeSource)
- 涉及文件并发与原子 rename
- diff 跨多文件,有 50+ 行非样板代码

**tester 跳过依据**:

- 本期所有验收标准 = 单元测试(已在 design.md §5 列举)
- 不存在跨进程 / 真实 LLM 调用的集成场景
- 设计文档明示"不引入 integration tests"
- developer 阶段的 unit-test 覆盖足以验证全部验收标准
