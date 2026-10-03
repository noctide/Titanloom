# Titanloom 契约骨架（M0 草案）

来源：《平台能力契约与Agent接入规范》3.1、3.2、6.5、8.2、8.3、12.1 与总框架 8.3。
状态：草案，未冻结；与规范冲突时以规范为准。这里的 JSON Schema 只固化规范已明确写出的字段与禁止项，没有新增业务语义。

| 文件 | 内容 |
| --- | --- |
| capability-descriptor.schema.json | 能力描述符 v0.2（字段名对齐能力契约 3.3 示例；命名规则、业务写入必须幂等、draft_only 限制、Schema 引用须带摘要） |
| command.schema.json | 命令与状态（accepted 必须有 invocationRef，等待确认必须绑定影响摘要） |
| job-attempt.schema.json | JobAttempt：Worker、租约、单调 fencing token、副作用状态（来源：能力契约 8.1、8.4） |
| prepare-result / job-state-notification .schema.json | 命令准备结果（影响范围未知不得自动执行）、状态推送通知（只含 jobId 与 stateVersion） |
| job-run.schema.json | JobRun（unknown 副作用不得直接 canceled / succeeded，终态必须有 finishedAt） |
| error.schema.json | 错误体（不得携带栈等内部字段） |
| state-machines.json | 命令与 JobRun 的迁移表 |
| query-execute-request / query-execute-response .schema.json | Query 执行请求与响应（来源：能力契约 9.1–9.2）：不得提交 SQL 与连接，snapshot 必须带快照，decimal 用字符串编码 |
| config-activation / audit-event / platform-alert-rule .schema.json | 配置激活（pending 至 failed，逐目标记录）、审计事件（前后值只存摘要）、平台告警规则（不承载领域业务异常）（来源：后台治理 7.2–7.3、12.1–12.2） |
| error-codes.json | 错误码注册表：22 个取自能力契约 12.2，7 个拟定（proposed） |
| pipeline-run.schema.json | PipelineRun 领域状态（拟定）：运行、质量、发布三个维度与不变量（来源：数据处理 15.1） |
| idempotency-record.schema.json | 幂等记录：作用域、requestDigest、保留期与 tombstone（来源：能力契约 7.1、14.2） |
| invocation-context / delegation / confirmation / invocation / outbox-event .schema.json | 调用链对象：可信调用上下文、委托、确认、调用、事件信封（来源：能力契约 5.2、5.3、6.4、7.2、13，考勤 15.3） |
| channel-connection / delivery-route / inbound-endpoint / inbound-receipt .schema.json | 集成与消息通道对象（来源：集成与消息通道子方案 3、4）：目标只能是 SecretRef、入站必须鉴权与时间窗、命令类入站必须绑定身份 |
| test_contracts.py、test_contracts_part2.py、test_contracts_part3.py、test_contracts_part4.py、test_contracts_part5.py | 校验示例、负例与状态机一致性；`pip install jsonschema` 后分别运行 |

集成通道的 channel-connection、delivery-route、inbound-endpoint 现使用 revisionState 与 operationalState 两个字段，不再使用单一 state。尚未覆盖：MessageTemplate、DeliveryRecord / Attempt、ExternalIdentityRef，以及各领域能力的 inputSchema / outputSchema。字段只取自规范已写明部分，未写明的（如事件类型清单、数据级别枚举的最终取值）属提案，待冻结。
