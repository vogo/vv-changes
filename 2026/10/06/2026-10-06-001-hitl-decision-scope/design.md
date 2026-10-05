# 设计：HITL 决策范围（逐 call 批准与策略指纹）

> 日期：2026-10-06
> 对应需求：同目录 `requirement.md`

## D1. 批准按 call id

批级「已批准执行」开关删除。Resume 只把本批里 pending、Execute 且非 IsError 的 id 放进已批准集合。每个工具调用在进入 handler 前带上自己的 executing id，包括普通 Run 与并行批。permission 仅当该 id 在集合中时才跳过非交互 Dangerous 硬拒绝。空 id、nil context、id 不在集合中都拒绝。`TierBlocked` 在批准检查之前返回。

HTTP 决策仍走 interrupt `Decision`。schema 的 resume 请求不携带 Execute；测试在 store 上 `SubmitDecisions`。

## D2. Record 版本

写入版本为 3。可读集合是 2 与 3，两个存储后端共用同一判断。version 2 没有指纹，resume 不做指纹检查，仍按 call 批准，不把文件改写成 3。version 1 及其他版本拒绝。没有把旧文件迁移成新版本的工具。

## D3. 指纹

指纹是不透明 SHA-256 hex。材料是同一次组装出的 bash 规则（名称、档位、模式、原因，含默认规则）。guardian 关闭时材料记 `guardian=off`，忽略目录。guardian 开启时加入排序后的允许目录与工作目录。不含命令文本，不含密钥。

vage 只存指纹和每条调用的分类字符串，不解析 bash 档位。Witness 是可选能力，不挂在策略接口上。未实现 Witness 的策略写入空指纹，resume 跳过检查。vv 的 Dangerous bash 策略必须是具名结构体并实现 Witness，flagged 集合与 Intercept 一致。

## D4. 指纹不符

查找 `Supersedes` 先于任何执行路径，包括指纹又变回旧值。已有后继则返回该后继，不进入会发出 `interrupt_resumed` 的恢复路径。没有后继时：空指纹按既有路径执行；没有 Witness、评估不一致、或新规则不再 flag 任何调用，则失败关闭，不拿租约。指纹变化且仍有 flagged 调用时，先拿旧记录租约，再按字段白名单建新 Pending：决策与租约为零，不整份拷贝旧记录。创建失败则释放租约；完成旧记录失败则不释放租约，仍返回后继。

HTTP：无法建后继为 409 `policy_drift`。建出后继为 200，body 的 interrupt id 是新 id。`POST .../decisions` 不跟随 Supersedes。

## D5. CapInterrupt

`CapInterrupt` 只在 ProfileFull。它不是工具，注册表对该能力为 no-op，也不能用来给 worker 接 interrupt。Primary 用自己的 tool profile 装配，不看描述符上的 ReadOnly。Fallback Primary 与派生 worker 不装配。

## D6. Options 收口

interrupt 的 store、policy、租约从 Options 的一对方法读出，并按 profile 是否声明 `CapInterrupt` 写入工厂选项。nil Options 与 nil 工厂选项安全。不再按 profile 名字做门禁。

## D7. memory_set 的 op

`op` 默认 set。`op=delete` 删除单个 key，value 必须省略或为空；缺失 key 成功。没有第三个删除工具。Clear 仍只在 user-path。可选 `ttl` 为秒，0 或省略表示不过期，负数拒绝。`value` 不在 JSON schema 的 required 里，由 handler 按 op 校验。删除事件为 `memory.delete`，不含 value。

agent-path 删除：file 后端读到 owner 不匹配时返回会话禁止；只存在于对方会话目录的私有 key 是找不到，不删对方的记录。SQLite 对他会话删除保持静默成功，本轮不改。

## D8. store 为 nil

profile 声明了 Remember 但 persistent store 为 nil：记 Warn，不注册工具，不因此进程失败。未声明 Remember 则不记这条 Warn。Primary 装配与 worker 装配使用同一文案。

## D9. 启动审计

FileStore 上的版本审计不是 Store 接口方法。它与列表一样跳过 `.lock` 与临时文件。未知版本或目录读取失败只记错误日志，提示运维移走或删除后才能恢复那些记录，不因此启动失败。

## 非目标

- 不迁移 version 1 记录。
- 不给派生 worker 或 Fallback Primary 接 interrupt。
- 不新增 `memory_delete` 工具，不把 Clear 开放给 agent。
- 不改 largemodel，不把 bash 档位类型注入 vage interrupt。
- 未发版前不改已发布模块的 require / replace，不打 tag。
