# Auto-plan: phase inclusion

Effective pipeline: `analyst → designer → developer → documenter` (all inline; no sub-agents dispatched).

- **improver**: **skip**. Design reuses existing patterns (regex, `permissionExecutor` wrap, three-state dialog). No new architecture, no cross-cutting concern, no multi-module reshape. Scope is 3 files of new code + 3 files of small edits.
- **reviewer**: **skip**. Change is additive and localized. No reshape of public interfaces (new functional setter on `PermissionState`, new sub-struct in `ToolsConfig`). Security-sensitive code — but the security logic is a pure function with unit tests, and the wiring is surgical.
- **tester**: **skip**. Unit tests in `classifier_test.go`, `permission_test.go`, and `config_test.go` cover all acceptance criteria directly (regex matching, tier routing, config loading). Integration tests would exercise the same seams with more scaffolding and no new signal. If acceptance drifts, revisit.

Tie-breaker note: security work normally leans toward including reviewer, but this change's security value comes from the rule *content* and the `permissionExecutor` ordering — both of which are small, test-driven, and reviewable in the diff. Adding a reviewer sub-agent would be ceremony, not scrutiny.
