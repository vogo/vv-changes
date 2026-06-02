# 2026-06-02-006 OpenAI:多模态输入/输出扩展(input_audio/file 输入,modalities/audio 输出)

## 背景

官方 OpenAI Chat Completions 已支持音频/文件多模态。当前 `aimodel/schema.go` 的 `ContentPart` 仅支持 `text` 与 `image_url`,`ChatRequest` 无 `modalities`/`audio` 输出配置,`Message` 无 `audio` 响应字段,无法覆盖音频输入与音频输出场景。

## 目标

### 1. ContentPart 输入扩展(均 omitempty)

| 字段 | 类型 | JSON 键 | 内嵌结构 |
|------|------|---------|---------|
| InputAudio | `*InputAudio` | `input_audio` | `InputAudio{ Data, Format string }` |
| File | `*FilePart` | `file` | `FilePart{ FileID, Filename, FileData string }`(均 omitempty) |

- `InputAudio`:`data`(base64)+ `format`(`wav`/`mp3`)。
- `FilePart`:`file_id`(已上传文件引用)或 `filename`+`file_data`(base64 内联),三者按需出现。
- Content 的 `MarshalJSON`/`UnmarshalJSON` 多态(string ↔ parts)行为不变:ContentPart 仅是普通结构体字段扩展,多态在 Content 层。

### 2. ChatRequest 输出配置(均 omitempty)

| 字段 | 类型 | JSON 键 |
|------|------|---------|
| Modalities | `[]string` | `modalities` |
| Audio | `*AudioConfig` | `audio` |

- `AudioConfig{ Voice, Format string }` → `audio.voice` / `audio.format`(format 如 `wav`/`mp3`/`flac`/`opus`/`pcm16`)。
- `clone()` 深拷贝 `Modalities` 切片(对齐既有 `Stop`/`Tools` 处理)。

### 3. 响应 message.audio(已确认纳入)

`Message` 新增 `Audio *MessageAudio`(`json:"audio,omitempty"`):
- `MessageAudio{ ID, Data, Transcript string; ExpiresAt int64 }`(均 omitempty),解析 assistant 消息生成的音频。

## 边界

- 仅 OpenAI canonical 类型;不改动 Anthropic 译码逻辑(这些字段 Anthropic 无对应,随 canonical 序列化但 Anthropic 路径不读取;`anthropic.go` 仅对 `text`/`image_url` 分支处理,新类型落入 default 被跳过,行为安全)。
- 不引入校验/枚举类型;voice/format 保持裸 `string` 透传。
- 不改 `AppendDelta`(音频流式增量本期不处理,超出范围)。

## 方案

- `schema.go`:`ContentPart` 增 `InputAudio`/`File` 两字段 + `InputAudio`/`FilePart` 类型;`ChatRequest` 增 `Modalities`/`Audio` + `AudioConfig` 类型;`Message` 增 `Audio` + `MessageAudio` 类型;`clone()` 深拷贝 `Modalities`。
- `schema_test.go`:input_audio/file 多模态消息序列化/反序列化往返;现有 text/image_url 用例保持通过;message.audio 响应解析。
- `openai_chat_test.go`:`modalities`/`audio` 请求体序列化键名/值校验 + omitempty 缺省不出现。
- 文档:`CLAUDE.md`(Key Types)、`README.md`(多模态小节)、`CHANGES.md`(同步条目)。

## Done Contract

- 完成:input_audio/file 多模态消息正确序列化/反序列化且不破坏纯文本与 image_url;`modalities`/`audio` 请求字段序列化正确、缺省 omit;message.audio 响应解析正确;`Modalities` 在 `clone()` 深拷贝隔离;三处文档与代码/测试同一改动内同步。
- 证据:`go test ./...` 通过,新增测试通过,无 `.test` 二进制残留。
- 未完成:任一代码/测试/文档缺失,或破坏现有 image_url/纯文本用例,或 `Modalities` 仍浅拷贝。

## Change Log / Validation

### 变更

- `schema.go`:`ContentPart` 新增 `InputAudio *InputAudio`(`input_audio`)、`File *FilePart`(`file`),并补 `InputAudio{Data,Format}`、`FilePart{FileID,Filename,FileData}` 类型(payload 均 omitempty);`ChatRequest` 新增 `Modalities []string`(`modalities`)、`Audio *AudioConfig`(`audio`)+ `AudioConfig{Voice,Format}` 类型;`Message` 新增 `Audio *MessageAudio`(`audio`)+ `MessageAudio{ID,Data,Transcript,ExpiresAt}` 类型;`clone()` 深拷贝 `Modalities` 切片。Content 多态(MarshalJSON/UnmarshalJSON)未改动。
- `schema_test.go`:`TestContentMarshalInputAudioAndFile` / `TestContentUnmarshalInputAudioAndFile`(input_audio/file 序列化往返)、`TestContentPartOmitsUnsetPayloads`(纯文本 part 不带其他 payload)、`TestMessageAudioUnmarshal` / `TestMessageAudioOmittedWhenNil`(响应 message.audio 解析与 nil omit)、`TestChatRequestCloneModalities`(改副本 Modalities 不影响原值)。
- `openai_chat_test.go`:`TestOpenAIChatRequestModalitiesAudio`(modalities/audio 请求体键名/值)、`TestOpenAIChatRequestInputAudioContent`(input_audio 多模态消息体序列化);并在既有 omitempty 用例中追加 `modalities`/`audio` 缺省断言。
- 文档:`CLAUDE.md`(Key Types 补 ContentPart payload + 音频输入/输出)、`README.md`(新增「Multimodal input & audio output」小节含示例)、`CHANGES.md`(新增同步条目)。

### 验证

- `go test ./...` → 224 passed in 7 packages(新增测试全部通过)。
- `gofmt -l` 无输出、`go vet ./...` 无问题、无 `.test` 二进制残留。

### 结论

核心目标已由测试证据证明完成:input_audio/file 多模态消息正确序列化/反序列化且不破坏现有纯文本与 image_url 用例;`modalities`/`audio` 请求字段序列化正确、缺省 omit;响应 `message.audio` 解析到位;`Modalities` 在 `clone()` 深拷贝隔离;三处文档与代码/测试同一改动内同步。无遗留下一循环。
