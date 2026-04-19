# Phase & Sub-Agent Stats Display

## Requirement

vv agent, 每个phase 和 每个 subagent , 在结束的时候显示 耗时, 请求/回复 token 使用情况.

### Examples

- Phase completion: `● phase Dispatch complete.  (3 tool uses · 26s  · ↑ 5.3k · ↓ 10.5k  )`
- Sub-agent completion: `● sub-agent research complete.  (5 tool uses · 26s · ↑ 5.3k · ↓ 10.5k)`
- Task completion: `● task  complete.  (5m26s · ↑ 11.3k · ↓ 30.5k)`

### Details

- ↑ represents request/input tokens
- ↓ represents response/output tokens
- Duration should be formatted appropriately (seconds for short, minutes+seconds for longer)
- Token counts should use human-readable format (e.g., 5.3k, 10.5k, 30.5k)
