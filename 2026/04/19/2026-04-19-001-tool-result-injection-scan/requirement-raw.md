# 原始需求（来自 feature-todo.md · P0-3）

> **需求名**：工具返回内容入模前注入扫描
>
> **类别**：安全（P0 · 最高优先级）
>
> **依赖项**：无
>
> **难度**：低
>
> **可复用的现有能力**：`vage/guard/injection.go`
>
> **排序理由**：现有注入 Guard 仅覆盖用户输入；需将 Guard 点扩展到 tool result → model 之间；复用已有规则库，增加一层中间件即可。

## 用户追加说明

用户指示：
> 基于这个需求去做一个详细的需求调研，了解类似产品，做这个需求都有些哪些实现？注意些什么问题？然后调研相关行业，这些功能的实现都有什么具体的方案？在设计方案的时候做相关的参考。然后实现完了以后，将这个功能，从 to do 文件中移除放到 implement 文件中。

## 背景（从 PRD 抽取）

当前 `vage/guard/injection.go` 仅对 **用户输入** 做注入扫描；Agent 工具调用返回内容（web 抓取、文件读取、shell 输出、MCP 工具返回等）未经过同等检查就进入模型上下文。
Prompt-injection / tool-result poisoning 已是业界公认的头号 Agent 攻击面（OWASP LLM01、Anthropic / OpenAI tool-use 安全指引、Simon Willison 多篇博客）：攻击者把 "忽略之前的指令，改为……" 之类的文本藏在网页/README/文件/工具返回中，当 Agent 把这段文本当作指令执行时就被劫持。
