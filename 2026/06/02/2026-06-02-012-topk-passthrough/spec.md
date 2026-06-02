# aimodel:透传 top_k 采样参数(canonical + Anthropic 映射)

- 日期:2026-06-02
- 模块:`aimodel`(多协议 LLM 薄封装)
- 深度:standard

## 背景

Anthropic Messages API 原生支持 `top_k` 采样参数(top-k 截断采样),但 canonical `ChatRequest`(`schema.go`)当前无对应字段,Anthropic 封装(`anthropicRequest` / `toAnthropicRequest`)也未输出,导致调用方无法触达该参数。

## 目标

在 canonical 增加 `TopK *int`(`json:"top_k,omitempty"`),并在 `toAnthropicRequest` 映射到 Anthropic `top_k`;OpenAI 侧按其序列化路径自然透传(`openai_chat.go` 直接 `json.Marshal(req)`,`omitempty` 保证未设置时不输出,不影响不支持该字段的后端)。

## 关键设计决策

- canonical 字段紧邻 `TopP`(`schema.go:153`),指针 + omitempty 与 `TopP`/`Seed`/`N` 等可选数值字段保持一致风格。
- `anthropicRequest` 字段紧邻其 `TopP`(`anthropic.go:41`),`toAnthropicRequest`(`anthropic.go:192`)在结构体字面量内直接 `TopK: req.TopK`。
- OpenAI 侧**零改动**:canonical 即 OpenAI 形态,新增带 `omitempty` 的字段自动透传;未设置则不出现在请求体。

## 边界 / 非目标

- 不做取值范围校验(透传指针,薄封装不含 validation,见 aimodel/CLAUDE.md)。
- 不动 `composes`、不动流式逻辑、不动响应解析。
- 不为 OpenAI 侧单独写映射代码(无需要)。

## 改动点

1. `schema.go`:`ChatRequest` 在 `TopP` 后新增 `TopK *int json:"top_k,omitempty"`,附注释。
2. `anthropic.go`:`anthropicRequest` 在 `TopP` 后新增 `TopK *int json:"top_k,omitempty"`;`toAnthropicRequest` 字面量加 `TopK: req.TopK`。
3. 测试(`anthropic_test.go`):覆盖 设置 `TopK` → `ar.TopK` 等值且序列化含 `top_k`;未设置 → `ar.TopK==nil` 且序列化不含 `top_k`。
4. 文档同步:`CHANGES.md`(新增条目)、`CLAUDE.md`(Key Types 段)、`README.md`(Key Types 段)。

## Done Contract

- **完成标准**:canonical 与 anthropicRequest 均有 `TopK *int`(omitempty);`toAnthropicRequest` 正确映射;未设置时两侧序列化均不输出 `top_k`;单测覆盖设置/未设置两情形;三处文档同步。
- **完成证据**:`go test ./...`(aimodel)全绿;`make build`(license/format/lint/test)EXIT=0 且无遗留二进制。
- **未完成判定**:任一测试场景缺失/失败;未设置时输出了 `top_k`;文档未同步。

## Spec Self-Check

- 完整性:覆盖 canonical/Anthropic/测试/文档四面;OpenAI 侧已论证无需改动。
- 一致性:与 `TopP` 等既有可选字段风格、命名、json tag 约定一致。
- 可测试性:设置/未设置两情形可由结构体断言 + JSON 序列化断言验证。
- 无歧义:字段类型(`*int`)、tag(`top_k,omitempty`)、映射点明确。
- 可行性:纯本地小改动,无外部依赖。

## Change Log(2026-06-02 已执行)

- `schema.go`:`ChatRequest` 在 `TopP` 后新增 `TopK *int`(`json:"top_k,omitempty"`)并附注释(gofmt 重排了字段对齐分组)。
- `anthropic.go`:`anthropicRequest` 在 `TopP` 后新增 `TopK *int`(`json:"top_k,omitempty"`);`toAnthropicRequest` 字面量加 `TopK: req.TopK`。
- `anthropic_test.go`:新增 `TestToAnthropicRequestTopK`,含 `set`(映射等值 + 序列化含 `"top_k":40`)、`unset`(`nil` + 序列化不含 `top_k`)两子测试。
- 文档:`CHANGES.md` 新增条目;`CLAUDE.md` Key Types 段 common request fields 补 `TopK *int`;`README.md` Common request fields 段补 `TopK *int → top_k` 说明。

## Validation

- `go test -run TestToAnthropicRequestTopK ./...`:3 passed in 7 packages(含两子测试)。
- `make build`(license-check → format → lint → test):EXIT=0;无遗留二进制(已确认 `aimodel` 二进制不存在,`coverage.out` 正常生成)。
- 默认行为零变化:未设置 `TopK` 时两侧序列化均不输出 `top_k`(由 `unset` 子测试断言)。

## 结论

核心目标已由证据证明完成:canonical `ChatRequest` 与 `anthropicRequest` 均新增 `TopK *int`(omitempty),`toAnthropicRequest` 正确映射,OpenAI 侧经 `json.Marshal` 自然透传;设置/未设置两情形单测全过;三处文档同步。无后续遗留循环。
