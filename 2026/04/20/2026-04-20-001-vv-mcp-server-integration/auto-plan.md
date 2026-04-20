# Complexity Assessment

**Effective pipeline:** analyst → designer → developer → reviewer → tester → documenter

## Decision

| Phase | Decision | Reason |
|-------|----------|--------|
| improver | **Skip** | 设计复用已有的 `vage/mcp/server`（无新抽象）；面板仅在 vv 内新增一个薄胶水包，实现计划有明确线性步骤（~3 个主文件），没有引入新架构模式或跨模块协议变动。 |
| reviewer | **Include** | 涉及安全敏感（Bearer token、loopback 校验、credscrub 接入）+ 新的公共配置结构 `MCPServerConfig`；触及 `main.go` 启动分支，回归面不小。独立第二双眼能快速把关。 |
| tester | **Include** | 新增跨进程能力 + 多条验收标准（工具列举、白名单、dispatcher、auth、凭据过滤），都可被集成测试覆盖；in-memory transport 允许脱离真实 LLM 做端到端验证。 |

**Tie-breaker**：reviewer 与 tester 两端都偏向"有边际收益且成本低"，按规则应纳入。
