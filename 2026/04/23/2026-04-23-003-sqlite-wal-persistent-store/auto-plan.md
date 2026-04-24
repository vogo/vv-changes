# Auto plan — P1-6 · SQLite + WAL

## Phase decisions

- **improver: SKIP** — design already surfaces the major trade-offs (pure-Go vs CGO driver, schema versioning, backend-selector default, no-auto-migration). No obvious alternative approach would materially change the plan; risk of churn > benefit.
- **reviewer: INCLUDE** — touches core shared code (`vv/memories`, `vv/configs`, `vv/setup`), multi-file diff, SQL with session-isolation semantics where subtle bugs can leak cross-session data. Tie-breaker says include.
- **tester: INCLUDE** — acceptance criteria require integration-testable behaviour (concurrent writes under WAL, config-backed backend selection, fail-fast on corrupt DB). Classic integration surface.

## Effective pipeline

`analyst → designer → developer → reviewer → tester → documenter`
