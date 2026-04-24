# Auto plan — P1-8 · Prompt caching hints

## Phase decisions

- **improver: SKIP** — Design surfaces the key trade-offs (struct-local vs exported flag, marker placement, `*bool` opt-out default). Approach mirrors Claude Code / Cursor. No obvious alternative approach warrants a second opinion.
- **reviewer: INCLUDE** — Multi-module diff (aimodel + vage + vv + docs), touches canonical schema types, cost-model adjacent. A second read on translation correctness and "no leakage to OpenAI" is cheap insurance.
- **tester: INCLUDE** — Concrete integration-testable contracts (exactly 2 `cache_control` markers emitted, OpenAI body contains zero, feature flag on/off).

## Effective pipeline

`analyst → designer → developer → reviewer → tester → documenter`
