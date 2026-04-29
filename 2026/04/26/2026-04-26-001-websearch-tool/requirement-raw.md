# 原始需求

来源：feature-todo / feature-implement 优先级表（P2-10）

| 编号 | 名称 | 类别 | 依赖 | 复杂度 | 路径 | 描述 |
|------|------|------|------|--------|------|------|
| 🆕 P2-10 | **WebSearch 工具** | 产品功能（主流对齐） | P1-10 | 低 | `vage/tool/` | 支持 Serper / Brave / Exa / Tavily 等可插拔 Provider；结果可直接交给 WebFetch 深读。配置注入 API Key；与 WebFetch 组合即可低成本覆盖 Perplexity 核心体验 |

## 澄清问答（2026-04-26）

| 问题 | 选项 | 用户选择 |
|------|------|---------|
| V1 应包含哪些 Provider？ | Tavily / Brave / Serper / Exa | **Tavily + Brave Search** |
| 默认 Provider 与 API Key 缺失时的行为？ | 无默认+缺key不注册 / 注册但报错 / 内置优先级自选 | **无默认 + 缺 key 即不注册**（零成本路径，与 trace/eval/budget 一致） |
| WebSearch 注册到哪个 Tool Capability？ | 并入 CapRead / 新增 CapWeb / 仅 researcher | **并入 CapRead**（与 web_fetch 同档） |
| 搜索结果是否要内嵌正文？ | 只返 URL+title+snippet / 支持 fetch_top_n / 全部内嵌 | **只返回 URL+title+snippet**（与 WebFetch 解耦，token 可控） |

