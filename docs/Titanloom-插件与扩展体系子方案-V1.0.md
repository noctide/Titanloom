# Titanloom · 插件与扩展体系子方案

> 版本：V1.0｜设计日期：2026-09-29｜修订日期：2026-10-03｜状态：详细设计稿，待工程实现与验收。
> 修订：2026-10-01 全文档边界统一，2 节补登记采集宿主、流程节点类型与表单脚本 / HTML Block 宿主（先发布协议再开放），8 节明确隔离门禁 G-ISO；2026-10-02 2 节新增通道预设（Channel Preset）；2026-10-03 8 节阶段 B 拆为 B1 / B2，与实施路线一致。
> 定位：Titanloom
> 公共平台规范之一，细化统一扩展模型、宿主协议、贡献注册、运行隔离与版本治理；能力语义、Agent
> 委托与公共命令协议以《Titanloom-平台能力契约与Agent接入规范-V1.0.md》现行同步基线为准。出现冲突时先记录架构决策并同步上位文档，不在本方案中暗改冻结边界。
> 项目面向企业场景设计；本文示例、字段和接口均为拟定契约，不表示生产环境已经具备这些能力。

## 1. 设计结论

平台采用**统一扩展模型、按贡献类型划分的宿主契约、受控运行宿主**。可独立交付的可变实现通过插件包提供；普通函数、每个页面和表单无需单独打包。"一切可扩展"描述的是实现替换与能力贡献上限，不意味着"一切业务对象都归插件系统"。Identity
/ Delegation、Authorization、Capability
Registry、受控数据出口、Asset、Job /
Executor、Audit、资源准入及不可变发布约束由可信平台底座保持；Dataset /
Pipeline、BusinessRecord、AutomationRun、KnowledgeVersion、Visualization
Release 等业务事实仍由各自领域 Owner
持有。公共服务的实现可经受控部署组合替换，但不得因此移除其治理语义。

一个 PackageVersion 可贡献多项类型不同的实现；被调用与授权的业务动作是
Capability，内部 Renderer、Operator 或组件不因注册而自动成为 Agent
Tool。能力目录、扩展注册表、宿主注册表分工：前者描述可调用业务动作，后两者记录交付包、贡献实现、绑定及运行状态。

  -----------------------------------------------------------------------------------------------------
  对象                    含义                                           身份与版本
  ----------------------- ---------------------------------------------- ------------------------------
  Extension               发布者命名空间内的逻辑产品                     稳定 extensionId

  PackageVersion          签名、摘要、依赖及文件构成的不可变交付物       packageVersion + digest

  Contribution            包提供的一项宿主实现，如解析器、算子、渲染器   contributionId +
                                                                         protocolVersion +
                                                                         implementationDigest

  Capability              可独立授权的业务动作，含输入输出、效果与验证   capabilityId + contractVersion

  Binding                 某贡献实现某契约或服务接口的精确绑定           bindingId + 精确实现版本

  Installation            部署级批准和运行授权                           packageVersion +
                                                                         deploymentCompositionVersion

  Enablement              在组织、空间或项目范围按贡献启用               scope + contributionId +
                                                                         policyVersion

  Instance                使用某版本贡献的连接、工具或 UI 配置           instanceId + configVersion
  -----------------------------------------------------------------------------------------------------

安装、范围启用、实例引用、能力授权及 Agent
三道门独立记录；声明权限只是需求，不是授予。扩展不能抢占 `titanloom`
能力命名空间，外部发布者命名空间由平台分配。

`Contribution -> Binding -> Host`
只回答"哪个固定实现可承载哪个契约"；`Capability -> Authorization -> Domain`
才回答"当前主体能否发起什么业务动作"。插件安装、Contribution
注册、Binding 激活、Executor 可用都不等于 Capability 已授权，更不转移
Dataset、Pipeline、BusinessRecord、AutomationRun、KnowledgeVersion、Release
等领域对象所有权。

## 2. 宿主协议与首期边界

每一种贡献由所属宿主发布可版本化的协议，包括输入输出或事件
Schema、调用方向、执行位置、权限、模型交付、隔离、资源、失败、撤权、兼容与迁移。新增类型需先完成宿主协议、治理映射和验收，不能用
`custom.execute(any)` 规避审查。

  ------------------------------------------------------------------------------------------------------------------------------------------------------
  贡献类型                宿主与职责                                                                首期安排
  ----------------------- ------------------------------------------------------------------------- ----------------------------------------------------
  Connector / Importer /  数据接入与受控导入导出；外部凭据由服务端保管                              首批文件解析与必要数据源
  Exporter                                                                                          

  Operator                数据处理 DAG 节点的类型、参数、语义和结果                                 SQL、隔离 Python/Polars、配置算子

  Executor                执行任务或算子的受限实现；领取 JobRun                                     已认证的 SQL、Python/Polars 宿主

  Component / Renderer /  可视化构建及固定版本播放                                                  基础组件、图表、主页基础 3D
  Layout / Media                                                                                    

  Tool Implementation     为工具自动化领域的 Tool / ToolVersion 提供受控执行实现；不拥有            已验证的工具样板链路
                          AutomationPlan / AutomationRun                                            

  Automation Trigger /    为工具自动化领域提供触发源或动作适配；TriggerBinding / TriggerOccurrence  按真实需求认证
  Action Adapter          / AutomationRun 仍由工具自动化领域拥有                                    

  Model Adapter           模型服务与推理运行适配；AgentTask、目标分解与模型上下文仍由智能管家拥有   已验证的模型适配链路

  Knowledge Provider /    经所属领域服务提供知识及语义层能力                                        有具体需求后认证
  Semantic Provider                                                                                 

  Event Handler           受控消费已发生的 DomainEvent；需要产生业务副作用时必须调用目标 Capability 分类型评估，不默认开放第三方热安装
                          / Command                                                                 

  Notification / Storage  公共设施适配                                                              受保护部署变更，分类型评估，不默认开放第三方热安装
  / Auth Adapter                                                                                    
  Channel Preset /        出站通道预设（模板骨架、签名方案、限频建议）与入站适配预设；仅声明式配置，不含任意脚本；首期只内置通用 Webhook，其余按真实需求认证
  Inbound Adapter Preset  
  Capture Host / Adapter  表单平台的快捷采集宿主与页面适配器；仍经 Adapter → Mapping → Validation → BusinessCommand 需先发布宿主协议、认证后再开放，不属首期
  Workflow NodeType /     表单平台的流程节点类型、表单脚本与 HTML Block 宿主；WorkflowInstance / HumanTask 仍归表单平台 需先发布宿主协议、认证后再开放，不属首期
  Form Script / HTML Block 
  ------------------------------------------------------------------------------------------------------------------------------------------------------

上表是已知协议目录，不是能力上限。存储与身份适配属于受保护部署变更，缺失或不兼容必需治理服务时拒绝相关业务准入。知识内容、数据血缘、页面状态仍由领域服务拥有。

### 2.1 Operator 与 Executor 分离

DAG 节点引用精确 Operator 契约、参数和实现 Binding；Executor
负责实际计算。Operator 描述数据处理 DAG
节点的版本化计算语义与输入输出契约，Executor 描述领取 JobRun /
JobAttempt
后的受限执行实现；二者不能因为由同一插件包交付就合并所有权。算子声明输入输出类型及语义、确定性、副作用、所需能力、预算、超时、取消、重试、检查点兼容和验证器。发布前检查依赖环、类型和权限；运行记录输入版本、实现摘要、结果、诊断及血缘。执行器替换需证明输出语义及恢复兼容，不得默默改变计算口径。外部写入要有幂等或补偿策略。PipelineRun
的 DAG、质量和发布状态由数据处理拥有，JobRun / JobAttempt
的排队、领取、租约、执行尝试、执行重试和取消机制由公共 Job / Executor
设施拥有；Job 成功不自动等于 PipelineRun 发布成功，Job Cancel
也不自动等于 Domain Run Cancel。

## 3. 清单、构建与版本

清单采用受控 Schema，字段至少为
`extensionId`、发布者、许可证与源码来源、包版本与摘要、`provides`、`requires`、可选依赖、宿主种类、激活策略、配置
Schema、迁移引用、语言包、最小运行预算和完整性记录。每项 Contribution
单独登记协议版本、实现摘要、权限和资源需求。能力描述符另由能力目录登记
`operationKind`、`impactLevel`、`effectModel`、输入输出
Schema、`agentPolicies`、授权要求、结果验证和精确 `implementationRef`。

``` yaml
# 示意，不是已发布 Schema
extensionId: example.excel
packageVersion: 1.0.0
digest: sha256:<build-digest>
requires:
  sdk: '>=1.0 <2.0'
  host: data-import/1.x
provides:
  - contributionId: example.excel.importer
    kind: importer
    protocolVersion: 1.0.0
    implementationDigest: sha256:<module-digest>
    hostKind: isolated-file-worker
    requiredActions: [asset.read, dataset.stage_write]
    budgetProfile: import.small
    networkPolicy: deny
    filesystemPolicy: scoped_temp
```

包版本、贡献协议版本、能力契约版本、SDK
版本和部署组合版本分别管理。兼容区间只用于安装时解析；已发布页面、主页、流水线、已排队命令和运行尝试钉住精确版本、摘要及配置快照，不使用
`latest`。构建时解析和封装 Schema，不在执行时抓取任意远程
Schema。破坏性语义变更采用新 major
并提供迁移；扩大权限或副作用时重新审核及授权，不自动继承旧策略。

依赖优先面向有语义的版本化服务或 Capability
契约；只有确实需要特定实现时才依赖具体包。解析器检查循环、冲突、宿主兼容、缺失的必需服务、预算及许可；结果写入不可变
DeploymentCompositionVersion。可信启动器验证批准的发布者、组合和摘要，普通扩展不能改写生产必需能力集合。启动失败释放预留资源并拒绝受影响服务入口。

## 4. 调用、权限与 SDK

人工 UI、Agent、CLI 和自动化经过相同领域服务及 Capability / Command
边界。服务端验证真实主体、运行端和委托，再按基础
ACL、范围、数据和模型交付规则裁决；Agent
另经平台资格、范围启用、资源/来源准入三道门。实际执行范围是主体、委托、插件运行授权、实例范围、数据访问及预算的交集。Agent
的发现、取数、修改和结果交付各自鉴权；安装插件不打开 Agent
权限。Contribution、Binding、Enablement
或运行上下文均不能替代目标领域授权。

Host SDK
同时规定宿主调用扩展及扩展调用平台的方向。Data/Dataset/Query、Asset、Job、Event、Knowledge、Visualization
等 SDK 是经过服务端控制的契约入口；扩展不能直接读取 Titanloom
PostgreSQL、其他模块私表或原始凭据，不自建用户、权限或查询出口。外部
Connector
如需访问获准的外部数据源，由受控连接服务使用密钥引用执行；不能把"禁止旁路平台库"误解为禁止合法的外部连接器。

运行上下文由可信服务构造并传递主体、委托、作用域、能力/贡献及实现版本、策略版本、traceId、预算与取消信号；调用方自行填写的同名字段不构成授权证据。SDK
不隐藏领域事务与审计规则。凭据保持在服务端密钥设施，扩展配置仅保存
secretRef；必要的运行时代理访问受作用域及有效期限制，原始密钥不得进入浏览器、模型上下文、普通日志或
Release。

Agent
可以在授权后诊断、生成、测试、安装、启停或发布扩展，但管理动作自身也是独立
Capability，需要委托、三道门、确认与业务审核。模型生成代码先形成草稿，经构建、验证、批准后才能进入固定版本部署；不得直接热推到
On-air。

## 5. 运行与隔离

受审查的内部后端实现可随模块部署；第三方代码和个人脚本使用操作系统强制边界的独立进程或经认证的隔离环境，限制身份、文件目录、网络出口、时间、CPU、内存、并发、临时存储与清理。仅有
Worker
名称或容器名称不足以证明隔离。无法落实隔离的目标部署不开放不可信代码执行。GPU、外部服务与新语言宿主在需要时单独认证，首期不预建通用插件虚拟机。

前端可信声明式组件由平台渲染；执行第三方 UI 需隔离
Origin、受控通信和数据投影，不把门户 Token、DOM 或作者权限交给组件。同源
CSP 不是唯一隔离措施。Renderer
故障限于组件并显示诊断占位，不使整页失效。Designer
的草稿代码经验证构建为固定产物；Player 只加载 Release
钉住的产物与语言资源，不能从扩展目录现场装源码。普通观看以当前访问者身份查询数据，无人值守展示用登记的展示身份。

网络、文件、进程和 Shell
默认拒绝，仅对具体贡献、目的地和用途批准；对外请求通过受控出口，保护内网地址和元数据服务。预算先做部署组合总池及实例准入，再按能力执行限制；交互与重计算队列分离。重试只用于声明可安全重试的操作，外部副作用必须有幂等或人工补偿。超时、撤权和紧急禁用使后续受限取数及交付停止，并记录具体受影响运行；断路、健康检查及故障隔离不得误报业务成功。

## 6. 生命周期与引用保护

登记草稿 → 清单/依赖/隔离校验 → 包安装及部署组合批准 → 范围启用 →
实例配置 → 能力绑定与授权 →
运行。停用区分停止新用、停止服务、紧急安全禁用及退役；卸载需先查询所有引用和数据迁移状态。包、贡献、能力、实例、宿主及运行状态各有真相源，不压进单一
`plugin_status`。

普通停止新用可在授权范围内保留旧 Release
对固定版本的依赖；紧急安全禁用与撤权优先于版本固定，受影响的展示按策略遮蔽或降级，执行停止后续受限操作。卸载前通过各权威来源或其受控投影检查
WorkspacePageVersion、Visualization Release / On-air、Pipeline /
PipelineRun、JobRun /
JobAttempt、AutomationPlanVersion、实例配置和保留期；扩展注册表只保存精确
Reference，不建立第二套依赖真相。有活跃引用不得物理删除，投影缺失或陈旧不能解释为"无依赖"。卸载不自动删除领域业务事实。升级先形成依赖与影响视图，复用现有血缘、关系及运行拓扑投影；列出受影响消费者、兼容和回退限制，测试通过后发布新组合或新
Release，不隐式重绑旧版本。

事件订阅遵守平台统一事件契约：Schema/版本、作用域、来源、授权、trace、去重和保留期。DomainEvent
表达已经发生的事实，不是目标领域写 API；Event Handler
需要产生业务副作用时必须调用目标领域 Capability / Command
并重新鉴权。事件不携带默认全量数据；消费者需要明细时再按当前权限请求。扩展之间不得直接导入私有代码或写对方表。后台治理中心只是管理
UI，扩展注册与生命周期服务属于平台基础设施，CLI 与 Agent 通过同一受控
API 访问。

## 7. 风险、审计与国际化

`impactLevel` 采用能力契约的 low/medium/high，不能映射为另一套"插件
R0---R4"。R0/R1/R2+ 仅适用于登记的 Agent
写入与控制动作的自治审核上限，复用能力规范第 3.1
节的证据、判定器版本和领域开启条件；安全审查按代码来源、执行面、网络、凭据、数据敏感度和副作用分别评估。来源改变审核流程和默认限制，不取消统一权限、审计及资源规则。

每次安装、批准、启用、调用、故障、迁移、停用与卸载记录主体、委托、范围、包/贡献/契约版本、摘要、结果、预算及追踪
ID；敏感参数脱敏。UI 提供 zh-CN/en-US 显示键及插件独立语言包；资源
ID、机器字段不翻译，扩展不能覆盖平台授权与关键确认文案。Release
锁定语言资源版本。

## 8. 首期实施与验收

  -------------------------------------------------------------------------------------------------------------------------------------
  阶段                    交付                                                   必须证明
  ----------------------- ------------------------------------------------------ ------------------------------------------------------
  A 契约底座              清单                                                   循环依赖拒绝、缺必需授权服务拒绝准入、精确版本可追踪
                          Schema、类型宿主、注册/绑定、组合锁定、SDK、部署校验   

  B1 真实链路（数据）     文件解析、配置及 SQL/Polars 内置算子                   不改 DAG 核心替换解析实现

  B2 真实链路（展示与     基础组件与 3D、工具/模型适配                           模型停用不影响人工业务
  适配）                                                                         

  C 隔离与治理            预算、撤权、诊断、范围启用、影响分析、卸载保护         跨空间不串凭据；Agent 关门拒绝；故障和资源越界被隔离

  D 按需开放              第三方 UI、其他语言/引擎、外部宿主                     每种宿主单独认证语义、权限、隔离、恢复及回退
  -------------------------------------------------------------------------------------------------------------------------------------

B1 对应《Titanloom-实施路线与首期工程基线》的 M2–M3，B2 对应 M6（以该文第 3 节阶段映射表为准）。阶段 C 的"隔离与治理"验收通过，即满足《Titanloom-实施路线与首期工程基线》中的隔离宿主门禁 G-ISO；在此之前，任何个人、第三方或用户提交的代码（含个人 SQL / Python 节点、工具 Python 样板、表单脚本）均不在共享服务器上执行，首期仅运行受审查的内置实现。

验收场景还包括：JobRun 重试保持实现版本；不兼容检查点拒绝恢复；停止
Polars 不影响无依赖的已发布查询；插件升级不热替换 On-air；旧 Release
引用阻止物理卸载；紧急禁用使旧固定产物安全降级；扩展请求未批准权限、直连平台库、越权模型交付均被拒绝。性能数字和资源限额须在目标服务器、业务负载和并发条件明确后登记并实测，不在设计稿中猜定。

## 9. 后续工程契约与文档同步

工程实施需进一步定稿：各宿主的 JSON Schema / ABI、签名及信任根、OS
隔离配置、密钥代理、预算层级和默认限额、迁移事务与回滚、事件投递语义、版本兼容矩阵、引用图及管理
API。数据库可按
Extension、PackageVersion、Contribution、Binding、Installation、Enablement、Instance、DeploymentCompositionVersion、RuntimeInstance、Reference、Execution
与 Audit 等实体拆解；权威 ACL、JobRun、Dataset、Release
和业务事实继续由原所属服务持有。

同步范围：总框架保持冻结的扩展原则并引用本方案；能力规范继续定义
Capability、Command、Delegation 与 Agent 语义；数据处理方案定义 Pipeline
/ PipelineRun / Operator 业务细则；工具自动化定义 Tool / Automation
业务对象；可视化定义构建/播放细则；后台治理中心复用本方案的管理 API
与现有影响投影，不复制扩展注册、数据血缘或领域运行真相。若改变公共权限、数据所有权、接口语义或依赖方向，先形成架构决策并同步受影响文档。
