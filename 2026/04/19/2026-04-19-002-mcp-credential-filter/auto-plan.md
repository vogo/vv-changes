# Complexity Assessment

## 评估

### improver: **skip**
- 现有设计几乎是 P0-3 的对称 + 行业共识（gitleaks-style 规则 + 字段名敏感词 + allowlist），无新架构模式、无跨模块深度耦合。
- 代码量预估 < 600 行，分布在一个新包 + 两处 MCP 文件 + 一处 vv config 扩展。
- 决策明确：不做熵值、不做回验证、不做 binary 扫描——没有"多方案待定"的设计悬点。
- 设计已显式对齐 11 条 AC + 列出 7 条 known limitations；二次审查收益低。

### reviewer: **include**
- 安全敏感代码（P0 类别），即便单元测试全绿也需独立双人查。
- 涉及新增公开 API（`vage/security/credscrub/` 的 `Scanner`/`Rule`/`FieldRule`/`Config`/`Hit`/`ScanResult` 等），接口一旦发布改动成本高。
- 触及 MCP Client/Server 边界（第三方攻击面）+ 新事件类型（schema 稳定性），风险面非平凡。
- 预计 diff 超过 50 行非模板代码。

### tester: **include**
- 业务规则分支明确（三档 action × 四个挂载点 = 12 条行为路径），典型整合测试适用场景。
- `vage/integrations/mcp_tests/` 目录已存在，新增本地 echo MCP Server + Client 用例成本低。
- 安全类需求必须有端到端回归兜底：默认规则集不能悄无声息漏检知名凭据格式。

## 有效流水线

`analyst → designer → developer → reviewer → tester → documenter`

(tester 失败则回到 developer，上限 3 轮)
