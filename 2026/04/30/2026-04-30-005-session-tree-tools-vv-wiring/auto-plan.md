# Complexity Assessment

## Phase decisions

| Phase | Decision | Rationale |
|---|---|---|
| analyst  | run inline (always) | scoped requirement clarification |
| designer | run inline (always) | technical layout & API shape |
| improver | **skip** | Design reuses existing patterns (workspace tools as template, session HTTP as template). No novel architectural concept; tradeoffs are mostly mechanical. |
| developer | run inline (always) | implementation across 5 packages |
| reviewer | **skip** | Surface is wide but mechanically simple — wrapping existing store APIs as tools and HTTP. Each file is small, parallel to existing precedents. The risk surface (concurrency / data races / error mapping) is already handled inside `vage/session/tree`; the new code is plumbing. |
| tester | **skip** integration; rely on inline unit tests | Integration tests need real LLM keys (out of scope per requirement); HTTP unit tests are written alongside developer work. |
| documenter | run inline (always) | update CLAUDE.md, PRD, design doc completion marks |

## Effective pipeline

`analyst → designer → developer → documenter`

## Notes

- Heavy parallelism in development (A6 tools + B1 config + B4 HTTP can be authored independently) — the main agent will inline these.
- Risk concentrations are at the boundaries (env override parsing, HTTP routing path conflicts, store init wiring); each gets a focused unit test in the same commit.
