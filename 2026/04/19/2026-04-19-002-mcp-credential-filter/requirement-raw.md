# 原始需求

从 `doc/prd/feature-todo.md` 选取第一个未实现需求：

## P0-4 MCP 凭据过滤中间件

- **所属类别**：安全
- **依赖项**：无
- **难度**：低
- **可复用的现有能力**：`vage/mcp/client`、`vage/largemodel/` 中间件模式
- **排序理由**：MCP 已成第三方攻击面；实现 `credentialScrubber` 拦截器挂载到 MCP client；低代码量，强约束

## 用户补充指令

> 基于这个需求做详细的需求调研，了解类似产品，做这个需求都有些哪些实现？注意些什么问题？然后调研相关行业，这些功能的实现都有什么具体的方案？在设计方案的时候做相关的参考。实现完后，从 feature-todo.md 移除并加入 feature-implement.md。
