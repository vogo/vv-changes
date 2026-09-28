# largemodel 仓库迁移与版本联动

## 背景

原 `aimodel` 仓库已经在 GitHub 更名为 `largemodel`。同时，原先由
`vage/largemodel` 和 `vage/schema` 承载的通用模型能力不应继续归属于 Agent
框架，而应与 OpenAI、Anthropic 等协议实现统一归入 `largemodel`。

## 目标

- 将 Go module 从 `github.com/vogo/aimodel` 更名为
  `github.com/vogo/largemodel`。
- 保留协议原生客户端，并新增通用模型能力层：
  `github.com/vogo/largemodel/model`。
- 将跨框架的消息、事件、工具和用量契约收敛到
  `github.com/vogo/largemodel/schema`。
- 让 `vage` 和 `vv` 只依赖已发布的 `largemodel` 版本，不依赖本地
  `replace`。

## 架构结果

依赖方向调整为：

```text
vv v0.1.0
  ├── vage v0.12.0
  └── largemodel v0.9.0

vage v0.12.0
  └── largemodel v0.9.0
        ├── model   通用 Caller、路由、middleware、provider codec
        ├── schema 统一消息、事件、工具与用量契约
        ├── openai  OpenAI 原生协议客户端
        └── anthropic Anthropic 原生协议客户端
```

`vage/schema` 暂时保留为类型别名兼容门面，保证已有调用方的类型身份、错误
身份和常量值不变。原 `vage/largemodel` 的生产实现已删除；目录中仅保留使用
外部 `largemodel` 包验证 `vage` 装配方式的可编译示例。

## 发布顺序

1. 发布 `largemodel v0.9.0`，提交 `177ca78`。
2. `vage` 改为依赖 `largemodel v0.9.0`，发布 `vage v0.12.0`，提交
   `9b726b7`。
3. `vv` 改为依赖 `largemodel v0.9.0` 和 `vage v0.12.0`，发布首个标签
   `vv v0.1.0`，提交 `73a456e`。

每一步发布后均通过 `GOPROXY=direct go mod download` 从 GitHub 标签重新下载，
确认下游验证使用的是公开发布物而不是本地工作区代码。

## 验证记录

- `largemodel`: `make build` 通过，包括 license、格式化、lint 和全量测试。
- `vage`: `make build` 通过，包括 license、格式化、lint 和全量测试。
- `vv`: 在删除所有本地 `replace` 后执行 `make build` 通过，包括格式化、
  lint、单元测试和集成测试。
- 旧 import `github.com/vogo/aimodel` 与
  `github.com/vogo/vage/largemodel` 已在三个项目中清零。
- `vv` 的真实 LLM golden/debug 用例在无 API key 环境按设计跳过；其余门禁
  全部通过。

## 兼容性说明

- `vage/largemodel` import 路径被移除，调用方需要迁移到
  `github.com/vogo/largemodel/model` 及其子包。
- `vage/schema` 当前仍可用，但新代码应直接引用
  `github.com/vogo/largemodel/schema`。
- `largemodel` 的根包继续聚焦协议原生客户端；通用抽象明确位于 `model`
  子包，避免原生协议类型与统一模型类型混杂。
