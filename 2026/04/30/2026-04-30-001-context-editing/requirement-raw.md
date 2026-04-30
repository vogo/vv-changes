# 原始需求

参考 `doc/design/session-context-solution.md`，评估方案完整性，调研行业优秀实践，设计方案并实现。
实现后在 `doc/design/session-context-solution.md` 标记完成。

本次要实现的功能是：**Context Editing**（即 P6：主动擦除/折叠老的、过期的、超大体量的 tool_result，把已经在历史里"消费完"的中间产物从送给 LLM 的消息序列中移除/压缩）。
