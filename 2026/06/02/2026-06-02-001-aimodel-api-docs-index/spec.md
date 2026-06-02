# aimodel 官方 API 文档索引与版本同步记录

- 日期:2026-06-02
- 任务深度:standard
- 状态:已完成

## 核心目标(Loop Anchor)

为 aimodel 建立集中、可持续维护的「官方 API 文档索引 + 版本同步记录」机制,让维护者与 AI 能追踪「封装对应哪个官方 API、同步到什么版本」。

## 背景事实

- aimodel 是对两种官方协议的薄封装:
  - OpenAI-compatible:`openai_chat.go` / `openai_stream.go`
  - Anthropic Messages API:`anthropic.go` / `anthropic_chat.go` / `anthropic_stream.go`
- 协议分发在 `chat.go`(`ProtocolOpenAI` 默认 / `ProtocolAnthropic`)。
- 现状:`CLAUDE.md` 第 68 行有一条散落的 Anthropic 文档引用;README/CLAUDE 均无统一「官方 API 文档」段落;仓库无任何 CHANGES.md。

## 范围 / 边界

- 仅改动 `aimodel/README.md`、`aimodel/CLAUDE.md`,新建 `aimodel/CHANGES.md`。
- 不改任何 `.go` 代码、不改协议行为。

## 计划

1. README.md 新增「官方 API 文档」段落(OpenAI + Anthropic 入口 + 指向 CHANGES.md + 维护约定)。
2. CLAUDE.md 新增同一段落,并收敛第 68 行 Anthropic 引用避免重复。
3. 新建 CHANGES.md:说明用途 + 维护约定 + 基线条目(日期/协议及版本/变更摘要/受影响文件)。
4. 校验三者交叉引用一致、中文 + 英文术语。

## 官方文档入口(权威值)

- OpenAI: https://platform.openai.com/docs/api-reference/chat
- Anthropic: https://platform.claude.com/docs/en/api/messages

## Done Contract

- 完成:README + CLAUDE 均含统一「官方 API 文档」段落(含两条入口、指向 CHANGES.md、维护约定);CHANGES.md 存在且含基线条目;三者交叉引用一致;CLAUDE.md 无重复 Anthropic 引用。
- 证据:三文件内容自检 + 链接/路径一致性核对。
- 未完成:任一文件缺段落、缺基线条目、或引用路径不一致。

## Change Log / Validation(执行后回填)

### Change Log
- `aimodel/README.md`:在 Design Scope 后新增「官方 API 文档」段落(表格:协议 / 官方文档入口 / 封装代码 + 指向 CHANGES.md + 维护约定)。
- `aimodel/CLAUDE.md`:在 thin wrapper 说明后新增同一「官方 API 文档」段落;将原 Anthropic Translation 段第 68 行散落引用收敛为「见上文『官方 API 文档』段落」。
- `aimodel/CHANGES.md`:新建,含用途说明 + 维护约定 + 2026-06-02 基线条目(OpenAI / Anthropic 各含官方协议及版本、官方文档、变更摘要、受影响文件)。

### Validation
- README ↔ CLAUDE「官方 API 文档」段落内容一致,均含 OpenAI + Anthropic 两条入口、指向 CHANGES.md、维护约定。
- CHANGES.md 反向链接 README/CLAUDE;CLAUDE.md Anthropic 引用去重无残留;三者交叉引用一致。
- 未改动任何 .go 代码,协议行为不变。

### 结论
核心目标已由证据达成(三文件交叉引用一致、基线条目齐全)。无后续 loop。
