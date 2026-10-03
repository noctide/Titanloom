# Titanloom · 智能管家子方案

> 版本：V1.0｜设计日期：2026-10-01｜修订日期：2026-10-02｜状态：领域设计基线，待接口、Schema、模型运行与工程验收。\
> 修订：2026-10-02 状态词统一为 canceled（4.1）；14.3 改为 Agent 只生成可视化草稿、不发布；新增 21.1 阶段与里程碑。\
> 定位：Titanloom 的平台级 Agent
> 工作域。智能管家负责理解用户目标、形成和修订执行计划、组织跨领域能力调用、维护
> AgentTask
> 上下文与恢复点、向用户汇报进度并在需要时请求确认或人工接管；它不是超级权限中心、第二套工作流引擎、第二套任务队列或业务事实数据库。\
> 上位约束：《Titanloom-总框架设计方案-V1.0.md》《Titanloom-平台能力契约与Agent接入规范-V1.0.md》《Titanloom-插件与扩展体系子方案-V1.0.md》及
> ADR-008、011、012、013、014。\
> 核心原则：**整个平台都可以是 Agent
> 的工作环境，但每一步能否进入、能看什么、能改什么，仍由目标能力、目标资源和当前委托决定。**

## 1. 建设目标

智能管家不是聊天框附带的"问答机器人"，而是 Titanloom
中面向人的统一智能协作入口。用户可以从独立工作台、Shell
侧栏或领域页面发起目标；智能管家理解目标，在当前权限与资源预算内发现可用能力，形成计划，调用一个或多个领域完成工作，并把等待、确认、失败、恢复和结果组织成一个持续可追踪的
AgentTask。

它重点解决：

-   用户不需要先知道"应该进入哪个模块、点击哪个按钮"，可以先表达目标；
-   跨模块工作由 Agent 组织，但业务动作仍由各领域执行；
-   长任务、人工确认、外部等待和模型中断后能够恢复，而不是依赖一次对话连接；
-   Agent
    可以主动检查中间结果、解释失败、调整计划，但不能绕过权限或静默扩大目标；
-   同一个 AgentTask
    在独立工作台和各领域侧栏之间保持一致，不因入口变化产生多个任务真相；
-   模型不可用时，确定性业务能力、已经提交的 Job / Automation / Workflow
    继续按各自 Owner 运行。

## 2. 边界

### 2.1 智能管家拥有

  ----------------------------------------------------------------------------------------------------
  对象                                含义
  ----------------------------------- ----------------------------------------------------------------
  AgentTask                           一个持续的用户目标及其 Agent 级生命周期，是本领域核心权威对象

  Goal                                当前明确目标、约束、完成条件和用户补充说明

  PlanVersion                         对当前目标的一份不可变计划版本；计划可重规划，但历史版本不覆盖

  AgentStep                           Agent 计划中的逻辑步骤，记录意图、依赖、所选
                                      Capability、结果引用与解释

  ContextBundle                       经授权和模型输入策略裁剪后，为一次推理/计划/步骤准备的上下文包

  Interaction                         用户与 Agent 的指令、澄清、确认、解释、汇报及关键交互记录

  Checkpoint                          可恢复的 Agent
                                      级检查点，记录任务推进所需的稳定引用而非复制全部业务数据

  Handoff                             转交人工处理、等待用户决定或由人工接管后再恢复 Agent 的记录
  ----------------------------------------------------------------------------------------------------

### 2.2 智能管家不拥有

-   Identity / Delegation / Authorization Decision；
-   Capability Descriptor、公共 Command / Invocation；
-   JobRun / JobAttempt / Executor；
-   AutomationPlan / AutomationRun / TriggerOccurrence；
-   WorkflowInstance / HumanTask / BusinessRecord；
-   Dataset / Query / Pipeline / PipelineRun；
-   Knowledge Document / KnowledgeVersion；
-   Visualization Page / Release / PlaybackSession；
-   Asset 字节、Secret、凭据；
-   Shell 导航、门户布局、Notification、平台告警和 Audit。

智能管家只保存这些对象的稳定 Ref、必要摘要和授权后可见的执行结果。Ref
不转移所有权。

## 3. 总体调用模型

``` text
Human
  │
  ▼
Shell / Domain Side Panel / AI Workbench
  │
  ▼
AgentTask ── Goal ── PlanVersion
  │
  ├─ discover ──> Capability Registry
  │
  ├─ prepare context ──> Authorization + Agent three gates + model-input policy
  │
  ├─ execute ──> Capability / Command ──> owning Domain
  │                                      ├─ direct result
  │                                      └─ JobRun -> Executor
  │
  ├─ observe <── Domain Result / RunRef / permitted Event / Notification
  │
  ├─ replan / wait / request confirmation / handoff
  │
  └─ report result to Human
```

Agent 不是 Domain Service 的上级。它只能通过目标领域公开的 Capability /
Command 进入业务状态机。

## 4. AgentTask

### 4.1 生命周期

AgentTask 使用自己的领域生命周期，不复用 Job 状态：

`draft -> active -> waiting -> completed / canceled / failed`

其中 `waiting` 必须带原因，例如：

-   waiting_user_input；
-   waiting_confirmation；
-   waiting_human_task；
-   waiting_domain_run；
-   waiting_external_result；
-   waiting_resource；
-   suspended_by_policy。

这些是 AgentTask 的等待原因，不改变被引用
WorkflowInstance、PipelineRun、AutomationRun 或 JobRun 的状态。

### 4.2 完成语义

AgentTask completed 表示：按当前 Goal 的完成条件，Agent
已完成应承担的组织、调用、核验和汇报工作。

因此：

-   某个 Job succeeded 不自动完成 AgentTask；
-   某个 Capability 返回成功也不自动完成 AgentTask；
-   AgentTask completed 不改写目标领域对象状态；
-   用户可要求"只帮我分析，不执行"，此时完成条件可以是形成经过来源标注的分析结果；
-   若目标包含业务修改，则必须以目标领域确认的结果作为完成证据。

### 4.3 取消

取消 AgentTask 只表示停止 Agent 继续规划和发起新的动作。已经提交给领域的
Command、JobRun、WorkflowInstance、AutomationRun 是否取消，由对应 Owner
和授权规则决定。

系统可以向用户展示"同时请求取消相关运行"的影响预览，但不能把 AgentTask
cancel 直接传播成领域 cancel。

## 5. Goal 与 PlanVersion

### 5.1 Goal

Goal 至少记录：

-   用户原始目标；
-   结构化目标摘要；
-   明确约束；
-   用户要求保留的偏好；
-   完成条件；
-   允许/禁止的动作范围；
-   当前 Workspace / Project / Resource 上下文引用；
-   发起者与有效 DelegationRef；
-   创建时间和后续用户修订。

模型推断出的内容不能冒充用户明确要求。无法安全确定的重要约束进入澄清或
Prepare。

### 5.2 PlanVersion

计划是 Agent 的执行解释，不是业务事实。每次实质重规划产生新的
PlanVersion，旧版本保留。

PlanVersion 包含：

-   planVersionId；
-   goalRevisionRef；
-   步骤 DAG / 顺序；
-   每步候选或已绑定 CapabilityRef；
-   依赖条件；
-   预计副作用；
-   是否需要确认；
-   资源预算摘要；
-   失败与恢复策略；
-   生成依据与模型运行引用。

计划不能提前授予权限。计划中出现某 Capability，只表示"拟调用"。

## 6. AgentStep 与能力调用

AgentStep 是 Agent 的逻辑步骤，不是 JobAttempt，也不是 Automation
StepExecution。

典型状态可为：

`planned -> preparing -> waiting_confirmation -> submitted -> observing -> succeeded / failed / skipped / blocked`

当 AgentStep 调用业务能力时：

1.  解析目标 Capability；
2.  获取当前 Capability Descriptor；
3.  检查 Agent 平台资格；
4.  检查当前范围是否启用；
5.  检查目标资源是否允许 Agent 进入；
6.  校验调用者身份、Delegation、业务 ACL、数据范围、模型输入与输出策略；
7.  高影响动作执行 Prepare / Impact Preview / Confirm；
8.  Submit Capability / Command；
9.  保存 Command / Invocation / DomainRun / JobRun 等稳定 Ref；
10. 根据目标领域结果推进 AgentStep，而不是自行改写业务状态。

跨领域的下一步重新执行上述过程，不因上一步成功而继承权限。

## 7. 三道门与委托

三道门继续由公共能力规范定义：

1.  平台资格；
2.  范围启用；
3.  资源/来源准入。

智能管家只消费判定结果，不复制授权引擎。

Agent 实际权限是调用者权限、Delegation、Agent 三道门、Capability
约束、资源 ACL、数据出口规则和运行预算的交集。**人没有的权限，Agent
不得通过"更聪明"获得；人拥有的权限，也不代表 Agent 默认拥有。**

Delegation
必须有明确主体、范围、能力、资源、期限及可撤销性。撤权后尚未开始的步骤立即重新判定；已经产生的业务事实按所属领域规则保留，不通过删除
AgentTask 擦除。

## 8. ContextBundle：上下文不是权限

### 8.1 上下文来源

ContextBundle 可以由以下受控来源组成：

-   用户在当前 Interaction 中主动提供的内容；
-   ShellContext 中允许传给 Agent 的 Workspace / Project / Page /
    selected resource refs；
-   Capability 返回的授权结果；
-   Dataset / Query / Knowledge / BusinessRecord 等领域的受控读取结果；
-   AgentTask 自己的 Goal、PlanVersion、Checkpoint 和历史 Interaction
    摘要；
-   用户上传并已进入受控 Asset / Attachment 流程的文件引用。

### 8.2 发送模型前的检查

页面"用户看得到"不等于"可以发送给模型"。模型输入前至少经过：

-   当前主体与 Delegation 校验；
-   Agent 三道门；
-   来源资源访问权；
-   model-input policy；
-   Secret / credential 排除；
-   必要的数据裁剪、脱敏和字段级限制；
-   模型/运行端允许的输入上限。

凭据、数据库连接密钥、服务 Token 和 SecretRef 实际值不得进入模型上下文。

### 8.3 ContextBundle 不复制事实

ContextBundle 保存必要快照、摘要、引用和来源证据，用于解释"Agent
当时基于什么做出计划"。业务事实仍以来源领域为准。需要最新值时重新通过来源
Capability 获取，不把旧 ContextBundle 当成当前事实。

## 9. 模型与推理运行

模型是 Agent 的推理实现，不是业务权限主体。

平台允许通过受控 Model Adapter
替换本地或远程模型实现。每次关键推理应能够关联：

-   modelAdapterRef；
-   model/version 或受控别名解析结果；
-   policy profile；
-   ContextBundleRef；
-   输出摘要或结构化结果；
-   token / latency / resource usage（可获得时）；
-   safety / validation result；
-   traceRef。

模型不可用时：

-   不新建依赖模型推理的步骤；
-   已提交的确定性 Job / Workflow / Automation 不因此停止；
-   AgentTask 可进入 waiting_resource / suspended_by_policy；
-   用户仍可直接使用各业务模块。

不承诺相同输入在不同模型或同一生成模型上得到字节级相同输出。

## 10. 高影响动作与人工控制

智能管家必须区分"建议""准备""确认""执行"。

对能力规范标记为需要确认或高影响的动作：

`Intent -> Prepare -> Impact Preview -> Authorization -> Human Confirm -> Execute`

确认必须绑定固定意图或足够精确的影响摘要，不能使用"以后类似操作都默认同意"隐式扩大范围。

以下情况优先请求人工：

-   目标或关键约束存在多种不可安全等价解释；
-   影响范围超出用户最初表达；
-   权限不足且需要用户选择替代路径；
-   外部副作用结果不可判定；
-   领域要求人工审批；
-   Agent 无法证明继续自动执行满足既定风险级别；
-   用户显式要求每一步确认。

人工可以暂停、取消 AgentTask、修改
Goal、拒绝计划、要求重规划或直接接管业务操作。

## 11. Handoff 与人工接管

Handoff 记录：

-   handoffId；
-   AgentTaskRef / AgentStepRef；
-   原因；
-   当前已完成动作；
-   尚未确认的动作；
-   相关 DomainRun / Job / HumanTask refs；
-   已知风险与不确定性；
-   建议人工下一步；
-   恢复条件。

人工接管后，Agent
不抢回控制权。只有用户明确恢复，或既定恢复条件满足且策略允许时，才生成新的
PlanVersion 继续。

## 12. Checkpoint 与恢复

AgentTask 必须可脱离浏览器连接持续存在。

Checkpoint 不保存"整个世界"的复制，而保存恢复所需的稳定引用：

-   Goal revision；
-   active PlanVersion；
-   已完成/阻塞 AgentStep；
-   Command / Invocation refs；
-   DomainRun / JobRun refs；
-   HumanTask refs；
-   ContextBundle refs；
-   待确认 Intent；
-   Handoff；
-   最后观测时间。

浏览器断线、模型进程重启或 Worker
失败后，恢复流程先查询已有引用的真实状态，再决定继续观察、重新规划或请求人工；不得因为"没收到返回"就换
idempotency key 重做业务动作。

## 13. 与 Automation、Workflow、Pipeline、Job 的关系

  --------------------------------------------------------------------------------
  对象                    Owner                   与 AgentTask 的关系
  ----------------------- ----------------------- --------------------------------
  AgentTask               智能管家                用户目标与 Agent 组织过程

  AutomationRun           工具自动化              确定性/预配置自动化运行；Agent
                                                  可调用或观察

  WorkflowInstance /      填报与流程              业务流转与人工任务；Agent
  HumanTask                                       可发起受控能力或等待

  PipelineRun             数据处理                数据 DAG 业务运行；Agent
                                                  可发起、观察、解释结果

  JobRun / JobAttempt     公共 Job / Executor     执行机制；Agent 只保存 Ref
                                                  和消费允许的执行结果
  --------------------------------------------------------------------------------

禁止把 AgentTask 设计成上述对象的"万能父状态机"。

对于周期性、确定性的重复工作，优先形成 AutomationPlan /
TriggerBinding；不要让一个 AgentTask 永久睡眠充当调度器。Agent
可以帮助用户创建自动化，但创建动作仍进入工具自动化 Capability。

## 14. Agent 与知识、数据和页面

### 14.1 知识

Agent
可以跨领域核验知识内容。例如发现知识文档可能错误时，可在授权范围内查询真正权威的数据、业务记录或配置，再决定提出
KnowledgeIssue、KCR 或执行允许的低风险修正。进入其他领域时重新鉴权。

### 14.2 数据

Agent 不直连 PostgreSQL
私有表。数据读取和分析通过数据处理、领域查询能力或受控数据出口。需要大规模
JOIN、重算、质量检查时，调用数据处理 Pipeline / Query
能力，而不是把数据塞进模型上下文完成第二套 ETL。

### 14.3 页面与可视化

Agent 可以帮助生成和修改可视化内容的草稿，但必须调用可视化
Capability；可视化的发布与 Take 首期不开放给 Agent（能力契约 6.4、总框架 2.3），开放须逐项显式授权并通过专项验收。Agent 不能修改不可变 Release 字节，也不能向 On-air
热推未登记代码。

Shell 侧栏可提供当前页面和选中对象的 Ref，但页面打开本身不构成 Agent
授权。

## 15. 主动工作与"管家"行为

智能管家允许主动协助，但主动性必须有来源。

允许的来源包括：

-   用户当前 AgentTask 的明确目标；
-   用户明确建立的 Automation / Trigger；
-   领域产生且当前策略允许 Agent 处理的事件或待办；
-   用户授权的周期性检查；
-   已进入 waiting 状态且恢复条件满足的 AgentTask。

不允许仅因为模型"觉得应该做"就持续扫描全平台、扩大资源范围或创建永久后台循环。

需要长期监控时，应转换为工具自动化领域的持久 AutomationPlan；Agent
负责解释、配置和在触发后参与需要推理的步骤。

## 16. 结果核验与汇报

Agent 的最终回答必须区分：

-   已执行并由目标领域确认的事实；
-   仍在运行/等待的事项；
-   Agent 基于证据形成的分析；
-   模型推断或建议；
-   未能验证的内容。

对于写操作，汇报至少关联可追踪的 Command / Domain result /
RunRef。对于分析结果，应保留主要来源
Ref；不能把模型生成文本冒充领域事实。

Agent 不以"没有报错"作为成功证据。

## 17. 审计与可观测性

关键 Agent 行为写入公共 Audit / Trace，但 Audit 不成为 AgentTask
状态机。

至少可追踪：

-   谁发起 AgentTask；
-   使用了什么 Delegation；
-   哪个 Goal / PlanVersion；
-   访问了哪些 Capability / Resource；
-   哪些内容进入了模型上下文；
-   哪些高影响动作经过确认；
-   产生了哪些 Command / DomainRun / Job refs；
-   何时重规划、暂停、接管和恢复；
-   最终向用户汇报了什么结果。

模型内部不可获得的隐藏推理过程不作为审计要求；审计保存可解释的计划、输入来源、动作、结果和必要决策摘要。

## 18. 资源治理

Agent 的能力上限不通过人为删除能力实现，但实际运行必须有预算。

预算维度可包括：

-   同时 active AgentTask 数；
-   单任务最大推理/调用轮次；
-   模型 token / 请求预算；
-   Capability 调用次数；
-   Job 并发；
-   ContextBundle 大小；
-   文件/数据读取量；
-   总运行时长；
-   等待保留期；
-   日志和检查点保留；
-   外部 API 速率。

预算耗尽时进入明确 waiting / blocked 状态，不能静默无限重试。

## 19. UI

### 19.1 独立工作台

至少展示：

-   当前目标；
-   当前计划和步骤；
-   正在等待什么；
-   最近能力调用；
-   需要用户确认/输入的事项；
-   相关业务运行；
-   结果与证据；
-   暂停、取消、接管、恢复和重规划入口。

### 19.2 Shell 侧栏

侧栏与独立工作台引用同一
AgentTask，不建立第二套会话状态。切换页面只更新候选
ShellContext；是否进入 Agent ContextBundle 仍需经过权限与模型输入检查。

### 19.3 领域内嵌

领域页面可以提供"让管家解释/检查/处理此对象"等入口，传递的是稳定
ResourceRef 和用户意图，不直接把页面私有状态或凭据塞给模型。

## 20. 插件与扩展

Model Adapter、Agent Skill/Tool facade、Context Provider
等可以通过插件体系贡献实现，但：

-   插件不拥有 AgentTask；
-   插件不能绕过 Capability Registry 和三道门；
-   插件安装不等于 Agent 可用；
-   Context Provider 只能提供调用者当前允许进入模型的内容；
-   Tool facade 最终仍映射到受控 Capability / Command 或工具自动化
    Tool；
-   插件不能获得 Secret 明文并把它注入模型；
-   替换模型或 Planner 实现不改变业务 Owner。

## 21. 首期工程基线

首期优先实现：

1.  AgentTask / Goal / PlanVersion / AgentStep；
2.  独立工作台 + Shell 侧栏共享任务；
3.  Capability discovery；
4.  三道门 + Delegation + 目标资源重新鉴权；
5.  ContextBundle 构建与模型输入裁剪；
6.  Prepare / Confirm / Submit；
7.  Command / DomainRun / JobRun Ref 跟踪；
8.  waiting / Checkpoint / reconnect recovery；
9.  Handoff / 人工接管；
10. Audit / Trace；
11. 一个可替换 Model Adapter；
12. 至少用两个不同业务领域完成真实跨域验收。

首期不要求：

-   自研基础模型；
-   让 Agent 直接操作数据库；
-   把全部业务能力改造成 Agent 专用接口；
-   建立第二套 Workflow / Automation / Job 系统；
-   永久在线的 AgentTask 代替 Automation；
-   未经授权的全平台自主巡检；
-   让模型直接持有平台凭据。

### 21.1 阶段与里程碑

阶段编号以《Titanloom-实施路线与首期工程基线》的 M 序列为准。

- **M5 第一步（只读与草稿）**：AgentTask / Goal / PlanVersion / AgentStep、独立工作台与 Shell 侧栏共享任务、Capability discovery、三道门与 Delegation、ContextBundle 与模型输入裁剪、Checkpoint 与断线恢复、Audit / Trace、一个可替换 Model Adapter。能力范围限于查询、任务跟踪与草稿生成。
- **M5 第二步（受控执行）**：Prepare / Confirm / Submit、Command / DomainRun / JobRun Ref 跟踪、Handoff 与人工接管、至少两个业务领域的真实跨域验收。执行类能力逐项经三道门与专项验收开放。
- **M6 及以后**：主动工作、周期性检查转 AutomationPlan、多 Agent 协作等，均待真实需求与第 23 节待定项冻结后再定。

完成标准：第 22 节 A01–A15 中与已启用能力对应者全部通过。

## 22. 验收场景

### A01：同一任务跨入口恢复

用户从 Shell 侧栏发起 AgentTask，进入独立工作台后看到同一
Goal、PlanVersion 和进度，不生成重复任务。

### A02：跨领域重新鉴权

Agent 先读取知识，再尝试访问数据处理 Dataset。即使知识访问成功，Dataset
无权限时仍被拒绝。

### A03：页面可见不等于模型可见

用户页面显示受限字段，但 model-input policy 禁止发送；ContextBundle
不包含该字段。

### A04：高影响确认

Agent Prepare 一个高影响修改，用户未 Confirm 前不得 Submit。

### A05：网络断线恢复

Submit 后浏览器断线；恢复时按原 commandId / RunRef
查询，不创建新的业务动作。

### A06：Job 成功但领域失败

节点 JobRun 成功，PipelineRun 因质量门禁失败；AgentTask
不报告"数据处理成功"。

### A07：AgentTask 取消不传播

取消 AgentTask 后，已经提交的 WorkflowInstance 不被自动取消；UI
显示其仍在运行并提供领域允许的后续操作。

### A08：人工接管

外部副作用结果不可判定时进入 Handoff；人工处理后可显式恢复 AgentTask。

### A09：撤权

任务等待期间撤销 Agent 资源准入；恢复时重新鉴权，后续步骤
blocked，不使用旧授权快照继续。

### A10：模型不可用

模型服务故障后 AgentTask 进入等待；已经运行的确定性 AutomationRun /
PipelineRun 不受影响。

### A11：长期监控转自动化

用户要求持续监控某条件；Agent 创建 AutomationPlan 的 Prepare/Confirm
流程，而不是让 AgentTask 永久轮询。

### A12：凭据隔离

调用需要 Secret 的 Capability 时，Secret
只在服务端/执行宿主解析；ContextBundle 和模型请求中不存在 Secret 明文。

### A13：计划变更留痕

用户修改目标后生成新的 Goal revision 和
PlanVersion；旧计划仍可追踪，不原地覆盖。

### A14：插件不增权

安装新的 Model Adapter 或 Tool facade 后，在未通过 Agent 三道门和目标
Capability 授权时仍不能调用目标资源。

### A15：结果证据

Agent 宣称业务修改完成时，必须能关联目标领域确认结果；只有模型文本或 Job
success 不满足完成证据。

## 23. 后续工程待定项

以下内容在现有方案中没有足够证据，不在 V1.0 中伪造最终答案：

-   具体 LLM / 本地模型产品及选型；
-   Planner 是单模型、双模型还是规则 + 模型组合；
-   ContextBundle 的精确 JSON Schema；
-   AgentTask / AgentStep 的最终数据库 Schema；
-   模型上下文、token、调用次数的生产配额；
-   模型输入分类与企业数据分级的最终映射；
-   是否需要专门的向量检索组件及其产品；
-   多 Agent 协作是否有真实业务需求；
-   主动 Agent 的默认通知策略；
-   模型评测集、准确率阈值和生产 SLO。

这些项目应通过真实业务样例、企业安全要求和目标服务器实测后再冻结，不因"Agent
平台应该有"而提前建设。

## 24. 与现行方案的同步关系

-   总框架：智能管家作为正式业务能力域，AgentTask Owner 在此落地。
-   平台能力契约：继续拥有
    Capability、Command、Delegation、三道门、幂等、Prepare/Confirm/Submit
    和公共 Job 语义。
-   Shell：拥有入口、页面上下文、导航和工作台承载；不拥有 AgentTask。
-   填报与流程：拥有 BusinessRecord、WorkflowInstance、HumanTask。
-   考勤：拥有排班、考勤事实、AttendanceAlert 等业务语义。
-   工具自动化：拥有
    Tool、AutomationPlan、TriggerOccurrence、AutomationRun；长期确定性监控由此承载。
-   数据处理：拥有 Dataset、Query、Pipeline、PipelineRun 及数据血缘。
-   知识与文档：拥有 Document、KnowledgeVersion、Issue/KCR；Agent
    可跨域核验但不夺权。
-   数据可视化：拥有 Page、Release、PlaybackSession 等；Agent 不改不可变
    Release、不热推 On-air。
-   后台治理：提供 Agent 策略、运行和资源治理视图，不复制 AgentTask
    状态机。
-   插件与扩展：提供 Model Adapter 等实现扩展，不改变 AgentTask 与
    Capability Owner。
-   部署容量：Agent 推理、调用和等待必须进入资源预算；模型 API
    成本/延迟单列。
-   ADR：若未来改变 AgentTask Owner、授权方向、跨域写路径或把
    Automation/Workflow/Job 并入 Agent，必须新增 ADR。
