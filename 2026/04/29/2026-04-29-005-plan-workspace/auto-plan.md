# Auto Plan — Complexity Assessment

## Phase decisions

| Phase | Include? | Rationale |
|---|---|---|
| analyst | ✅ | Always runs |
| designer | ✅ | Always runs |
| **improver** | ❌ skip | Localised additive: 3 small new packages + a single new option on TaskAgent + 4 routes in vv. No new architectural pattern, no schema reshape (events are additive, not breaking). The design reuses existing patterns (Source / ToolDef+Handler / setup-based wiring) verbatim. No senior second-opinion required. |
| developer | ✅ | Always runs |
| **reviewer** | ✅ include | Touches multiple modules (vage/workspace, vage/tool/workspace, vage/context, vage/agent/taskagent, vage/schema, vv/setup, vv/registries, vv/apis). Path traversal validation is security-sensitive. Hook event additions are observable surface. Worth a code-review pass to catch oversight in the cross-cutting integration. |
| **tester** | ❌ skip | Acceptance criteria are exercised by colocated unit tests (path traversal vectors, source ok/skipped/error tri-state, taskagent extra-sources order assertion). No new external integration to drive — workspace is process-local file IO. The vv→vage wiring is exercised by existing setup/agent tests once we update them. Adding integration tests for filesystem manipulation would duplicate the unit tests without raising coverage. |

## Effective pipeline

```
analyst → designer → developer → reviewer → documenter
```
