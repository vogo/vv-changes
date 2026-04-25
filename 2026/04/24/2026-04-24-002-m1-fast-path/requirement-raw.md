# Raw Requirement

Source: `doc/design/enhance-prompt-understand-and-solve.md` § 3.1 "Layer 0: 启发式快速路径 (Quick Win)"

User instruction: 实现其 M1 阶段功能 (heuristic fast-path in Dispatcher).

## Excerpt from the design doc (§3.1)

在 `Dispatcher.Run` 最前面增加 `fastPath(req)`:

```go
if hit, agent := d.fastPathClassify(req); hit {
    return d.runDirectShortCircuit(ctx, req, agent)
}
```

规则 (可配置):
1. Greeting / Small-talk 正则: `^(hi|hello|hey|你好|在吗|thanks|bye)\b` + 消息 < 60 字符 → 直接 `chat`.
2. 纯工具触发词: `^(calc|echo|date|pwd|ls)\s` → 直接 `coder`.
3. 历史上下文检测: 对话历史中已经确定 agent → 后续同 session 短消息继续沿用.

Config: `configs.Dispatcher.FastPath = {Enabled: true, GreetingPatterns: [...], MaxChars: 60}`.

## Open question from the design doc (§6.2)

当用户在复杂任务上下文中突然发 "thanks" 时,不应错误地走 fast-path。建议 fast-path 仅在 `len(req.Messages) <= 2` 且无工具调用历史时生效。
