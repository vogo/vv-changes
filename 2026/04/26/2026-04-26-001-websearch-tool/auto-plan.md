# Auto Plan — Complexity Assessment

| Phase | Decision | Rationale |
|-------|----------|-----------|
| analyst | include | 默认必含 |
| designer | include | 默认必含 |
| improver | **skip** | 设计严格沿用 webfetch 既有套路；无新架构、无新算法、无新抽象；6 处文件改动均为机械接线，无 senior second-opinion 收益空间。 |
| developer | include | 默认必含 |
| reviewer | **include** | 涉及 `ToolsConfig` 共享配置、新外部 API 集成、API Key 处理、错误码契约、ToolDef schema —— 属于 contract-shaping 改动，需要独立审视。 |
| tester | **include** | 新外部集成 + 多组件交互（vage tool + vv configs + vv registries + vv setup）+ 明确可测的 AC（"缺 key 不注册 / 配 key 出现"）。 |
| documenter | include | 默认必含；需要在 PRD 与 feature-implement 同步 web_search 状态 |

**Effective pipeline:** `analyst → designer → developer → reviewer → tester → documenter`
