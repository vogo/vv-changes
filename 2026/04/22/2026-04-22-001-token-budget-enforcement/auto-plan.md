# Auto-Plan: Complexity Assessment

## Decision summary

- **improver**: **INCLUDE** — introduces a new cross-cutting concern (budget layer spanning middleware + hook + persistence), new event types, and multi-module (aimodel/vage/vv) touch points. A senior second-opinion can catch over-engineering before implementation starts.
- **reviewer**: **INCLUDE** — touches core/shared `vage/largemodel/` middleware chain, new public event types, new public types in `vv/traces/budgets`, concurrent accumulation with atomic ops, error path design. Substantive diff (>500 LOC).
- **tester**: **INCLUDE** — backend logic (accumulate/check/reset), persistence layer (JSONL replay), concurrency (concurrent Add), daily window reset, pre/post-call interaction via middleware. Design lists 8 integration scenarios; acceptance criteria are integration-testable.

## Effective pipeline

```
analyst ✅ → designer ✅ → improver → developer → reviewer → tester → documenter
```
