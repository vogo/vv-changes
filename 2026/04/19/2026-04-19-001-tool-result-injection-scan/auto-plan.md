# Complexity Assessment — P0-3 Tool-Result Injection Scanning

## Phase-by-phase decision

- **improver = INCLUDE** — 设计跨 4 个包（guard/schema/taskagent/vv），引入新 Direction、新 Guard、新事件，且需求文档明确列了 3 个 Open Questions（rewrite 产物形式、vv 默认动作、新旧事件复用）。正是 second-opinion 该介入的场景。
- **reviewer = INCLUDE** — 属于 P0 安全类变更，触及 core/shared（`vage/guard/`、`vage/agent/taskagent/`），多文件改动，正则/截断/错误路径都需要独立把关。
- **tester = INCLUDE** — 需求 AC-1.4（无 guard 时零破坏）、AC-5.1（50KB 内 p99 < 2ms）、AC-2.3（输入侧行为不回退）等都是可集成测试的断言。

## Effective pipeline

analyst ✅ → designer ✅ → **improver† → developer → reviewer† → tester†** → documenter
