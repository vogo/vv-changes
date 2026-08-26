# provider 包目录收敛

## 背景

模型路由 Backend 位于 `largemodel/composes/{openais,anthropics}`,message codec 位于 `largemodel/provider` 根包。同一 provider 的 wire 类型、capability、endpoint routing 与 canonical message 转换被拆分到两个目录,职责边界不直观。

## 调整

- `largemodel/composes/openais` 迁入 `largemodel/provider/openais`。
- `largemodel/composes/anthropics` 迁入 `largemodel/provider/anthropics`。
- OpenAI message codec 并入 `provider/openais`。
- Anthropic message codec 与 tool-result 合并逻辑并入 `provider/anthropics`。
- `largemodel/provider` 仅作为 provider namespace,不再承载混合厂商的 Go package。
- 调整 `largemodel` caller、endpoint 配置、测试与设计文档中的 import 和目录引用。

## 结果

每个 provider 包完整拥有自身的 native wire adapter:Backend routing、capability、endpoint construction 与 message codec。`largemodel/router` 继续只提供 provider-neutral 的路由机制,`schema` 继续只维护 canonical message。

## Caller facade 边界与文件布局

- `largemodel/provider/{openais,anthropics}` 定位为 native wire codec 与 routed backend,不反向依赖 `largemodel` 根包。
- `AnthropicMessagesBackend`、`OpenAIChatBackend` 保留在消费它们的根包 Caller adapter 附近;它们是根包到 native backend 的最小 port,不与 provider endpoint 接口合并。
- `OpenAIChatComposeCaller`、`AnthropicMessagesComposeCaller` 使用根包的 `Request`、`Response`、`Stream` 并实现公开 `Caller`,因此保留在根包;下移到 provider 子包会造成 `largemodel → provider → largemodel` import cycle。
- 原 `largemodel/compose.go` 按职责拆为 `compose_options.go`、`compose_pool.go`、`openai_compose.go`、`anthropic_compose.go`,只调整文件布局,不改变公开 API 与运行行为。

## 验证

- `gofmt` 覆盖新增及调整的 Go 文件。
- `go test ./largemodel/...` 验证 Caller、池并发、provider codec、router 与依赖边界,通过。
- `go test ./...` 在 `vage` 模块执行全量回归,通过。
