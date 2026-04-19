# Auto Plan — Phase decisions

| Phase | Decision | Rationale |
|-------|----------|-----------|
| analyst | Run (always, inline) | Always on; ran with industry research per user request |
| designer | Run (always, inline) | Always on |
| improver | **Include** | New cross-cutting security primitive (`PathGuard`) introduced across 6 tools + config + setup + permission layer. Design spans `vage/tool/toolkit`, `vage/tool/{read,write,edit,glob,grep,bash}`, `vv/configs`, `vv/setup`, `vv/registries`, `vv/cli`. ≫ 3 modules, 10+ ordered tasks, security-sensitive, non-trivial TOCTOU + symlink decisions. Senior second-opinion warranted. |
| developer | Run (always, inline) | Always on |
| reviewer | **Include** | Touches security-sensitive code, new data types, multi-file diff well over 50 LOC of non-boilerplate, concurrency adjacent (fd lifecycle), tool-interface reshaping. |
| tester | **Include** | Every AC is concrete integration-testable (six tools × escape vectors). Design explicitly spec's 24 rejection + parity legitimate-access scenarios. |
| documenter | Run (always, inline) | Always on; also moves the requirement from `feature-todo.md` → `feature-implement.md` per user request. |

**Effective pipeline:** analyst → designer → **improver** → developer → **reviewer** → **tester** → documenter (with up to 3 developer→tester fix iterations if needed).
