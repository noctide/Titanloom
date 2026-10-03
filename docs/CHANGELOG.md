# Titanloom 设计基线变更记录

此文件记录对外可辨认的**设计基线**变化；方案正文只描述现行设计，架构取舍原因见《Titanloom-架构决策记录》。尚无对应代码发布和 Git tag 时，不将文档版本冒称为软件发行版。

## [Unreleased]

### 2026-10-03｜数据可视化与数据处理的接口补齐（拟定）

-   **数据处理 12.6**：新增字段标识 FieldRef（稳定字段身份，同义列由责任人显式绑定，不按列名自动归并）、Query 可筛选声明 filterable（按 FieldRef 声明参数槽位、操作与粒度，手写 SQL 须声明绑定位置）、数据代次 dataGeneration 与事件 `data.dataset.generation_changed`（单调递增，回退也递增，不含数据行）。此前可视化 8.2、8.3、16.2 引用了这些对象，但数据处理与能力契约均未定义。
-   **能力契约 9.1**：Query 请求新增按 FieldRef 的 `filters`（eq / in / range 左闭右开 / relative 由服务端解析 / search / hierarchy；未声明的筛选返回参数错误），响应新增 `dataGeneration` 与列 `fieldRef`。
-   **数据可视化**：8.2、8.3 改为引用数据处理 12.6；16.2 报表生成完成事件命名为 `visualization.report.generated`，经 Notification 与集成通道 DeliveryRoute 外发，受通道最高数据级别限制。
-   **集成与消息通道 3.1**：明确领域事件须经 Notification 形成通知类别后才可路由。
-   **已确认的两项**：同义列必须由责任人手动绑定，不按列名自动归并；版本回退时数据代次继续递增，不复用旧值。
-   **契约**：新增 field-ref、query-filterable、dataset-generation-changed 三个 Schema，Query 请求与响应 Schema 升至 0.2；新增第六部分测试 34 项，契约测试共 193 项。

### 2026-10-03｜设计基线 V1.0-rc1（首个公开版本）

-   **文档体系**：总体架构、公共平台规范、公共平台能力、业务能力域四层。包含总框架、平台能力契约与 Agent 接入规范、Shell 与门户及个人工作台、填报与流程、考勤签到、工具自动化、数据处理、知识与文档、数据可视化、后台治理、插件与扩展、集成与消息通道、智能管家、部署拓扑与容量规划、开源与第三方依赖治理、实施路线与首期工程基线、架构决策记录（ADR-001 至 ADR-019）。
-   **唯一 Owner**：Dataset / Query / Pipeline / PipelineRun 归数据处理；BusinessRecord / RecordRevision / WorkflowInstance / HumanTask 归填报与流程；Tool / AutomationPlanVersion / TriggerOccurrence / AutomationRun 等归工具自动化；KnowledgeSpace / Document / KnowledgeVersion 归知识；可视化 Page / Release / PlaybackSession 等归数据可视化；AgentTask 归智能管家；JobRun / JobAttempt 归公共 Job / Executor；Asset 字节归 Asset 服务。Shell、后台治理、搜索、关系图和扩展注册表只保存受控引用或可重建投影。
-   **运行与调用语义**：Domain Run 与 JobRun / JobAttempt 分离；Execution Retry 不等于 Domain Retry / Rerun；Job Cancel 不等于 Domain Run Cancel；Job succeeded 不自动等于领域发布、审批或业务成功。业务修改进入目标领域 Capability / Command；DomainEvent 表达已发生的事实；跨域调用重新鉴权，Context 不继承权限。
-   **告警与通知**：平台运行与治理告警归公共监控告警设施；业务告警（如 AttendanceAlert）归所属领域；Notification 只负责投递与已读。
-   **插件与扩展**：Contribution → Binding → Host 描述实现承载，Capability → Authorization → Domain 描述业务动作授权；安装、启用、Binding 激活均不自动授予 Capability。
-   **资源**：能力不设人为上限，资源必须有明确上限；共享服务器的物理内存不作为 Titanloom 可用配额。
-   **集成与消息通道**：出站 Webhook / 机器人通道、入站事件接入、统一可视化配置、出网守卫与密钥处理（草案）。
-   **契约骨架**（titanloom-contracts）：能力描述符、命令、准备结果、调用、确认、委托、JobRun、JobAttempt、PipelineRun、统一事件信封、Query 请求与响应、错误码注册表、配置激活、审计事件、平台告警规则、集成通道对象的 JSON Schema 草案及状态机，附测试。
-   **许可证**：Apache-2.0，署名 "Titanloom contributors"，贡献采用 DCO。
-   **持续集成**：GitHub Actions `contracts`（推送到 main 与 Pull Request 时运行全部契约测试）与 `dco`（检查每个提交带有与作者一致的 Signed-off-by 签署行），只读权限、不使用密钥。
-   **测试数据**：一律使用合成数据（实施路线 D-10）。
-   **文档头部**：各文档列出修订日期与修订摘要，便于判断正文是否为现行版本。
-   **状态**：本版本为**设计基线**，不表示系统已实现、性能已验收或安全已评审。
-   **待补齐**：OpenAPI / 数据库 Schema / 迁移脚本；可观测性与审计专项细则；备份恢复与灾备专项；部署容量实测；安全披露政策与发布签名；跨域 Contribution 视图（如确有消费者）。
