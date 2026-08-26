# 迁移 aimodel/composes 至 vage/largemodel

## 背景

多端点路由、调用内重试与端点健康判定属于 agent framework 控制面,不应留在 thin SDK (`aimodel`) 中。实际消费方仅有 `vage/largemodel`。

## 变更

### vage

- 新增 `largemodel/router/` — 原 `aimodel/composes` 路由核
- 新增 `largemodel/composes/openais/`、`composes/anthropics/` — 协议绑定层
- 新增 `OpenAIConfig` / `AnthropicConfig` 与 `NewOpenAIChatCallerFromConfig`
- 扁平 Option:`WithRetryPolicy`、`WithRecoverTime`、`WithAttemptObserver`、`WithConcurrency`
- 公开类型 re-export:`Strategy`、`EndpointStat`、`ErrNoActiveEndpoints` 等
- 旧 API 保留并标记 Deprecated:`NewOpenAIChatComposeCaller`、`WithComposeRouterOptions` 等

### aimodel

- 删除 `composes/`、`integrations/compose_tests/`、`doc/design/compose.md`
- 删除 ADR 0003–0005(路由专属,实现已迁至 vage)
- README / architecture / ADR 索引 / AGENTS 去除一切 composes 说明

### vage

- 新增 `vage/doc/domains/capability/model/router-design.md`
- `model.md` / `model-design.md` 链到 router-design

## 测试

- `vage/largemodel/...` 全绿(含自 aimodel 迁入的 router/composes 测试)
- `aimodel/...` 全绿(dependency 测试更新)
- `vv/configs` 的 `NewLLMClient` 改用 `NewOpenAIChatCallerFromConfig` / `NewAnthropicMessagesCallerFromConfig`

## 后续

- 可选:router 可注入 failure classifier,改善 400 被当作 transient 重试的行为
- 移除 Deprecated API(下一 major)
