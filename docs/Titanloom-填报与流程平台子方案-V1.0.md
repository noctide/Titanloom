# Titanloom-填报与流程平台子方案-V1.0

> 文档状态：V1.0 架构基线｜修订日期：2026-10-03\
> 修订：2026-10-01 全文档边界统一，流程人工任务统一为 HumanTask，流程中的 Agent 节点改名 AgentCall（14.1），业务基表与人员主数据的归属改正（2），补隔离门禁 G-ISO 说明，新增 24 验收基线与 25 建设顺序；2026-10-02 新增 19.1 异常与恢复；2026-10-03 5.5 更正：Capture Host / Adapter 已在插件体系登记，宿主协议尚未发布。\
> 平台定位：Titanloom 的业务事实采集、业务交互与业务流转权威平台\
> 核心原则：业务事实只记录一次；采集入口允许差异；业务语义必须可解释、可映射、可关联；能力不设人为上限，资源必须有明确上限。

------------------------------------------------------------------------

## 1. 目标与边界

### 1.1 建设目标

填报与流程平台不是传统"在线表单 + 审批"系统，而是 Titanloom
中负责人工及外部业务事实进入平台、形成权威业务记录并完成业务流转的统一入口。

平台重点解决： - 高频工作登记造成的重复操作与用户负担； -
Excel/CSV/复制粘贴等既有业务输入格式差异； -
页面快速采集、浏览器扩展、API、Agent、自动化等多入口统一； -
表单计算、校验、基表引用、层级联动和复杂业务界面； -
多阶段流程中时间、字段权限、修订、退回、再次确认等生命周期问题； -
最终业务事实以统一契约进入数据处理中心，参与关联、清洗、计算、语义建模和看板制作。

### 1.2 核心边界

填报与流程平台负责： - Form / Quick Capture / Import / Paste / API /
Agent / Automation 输入； -
当前业务记录内的表达式、校验、Lookup、级联和即时计算； - Business
Command、BusinessRecord、Revision、DomainEvent、Provenance； -
Workflow、Task、业务事件时间和流程状态； - Schema Registry
与输入/业务契约。

数据处理平台负责： - 跨大量记录或 Dataset 的 JOIN； -
批量清洗、转换、窗口计算、聚合； - 历史重算、统计分析和指标生产； -
Semantic Layer 与 Published Dataset。

禁止 Form Runtime 演变为第二套 ETL；禁止数据处理平台静默改写权威
BusinessRecord。

------------------------------------------------------------------------

## 2. 总体架构

``` text
                         Titanloom 填报与流程平台
                                  │
 ┌────────────────────────────────┼────────────────────────────────┐
 │                                │                                │
 ▼                                ▼                                ▼
采集体系                         业务事实体系                      流程体系
 │                                │                                │
Form                             BusinessRecord                   Workflow
Quick Capture                    Revision                         HumanTask
Excel / CSV                      DomainEvent                      BusinessCommand
Paste                            Provenance                       Gateway
API                              Attachment                       Timer/Event
Agent                            Business Time                    Delegation
Automation                       Concurrency                      ServiceTask
 │                                │                                │
 └────────────────────────────────┼────────────────────────────────┘
                                  ▼
                          Schema Registry
                                  │
          ┌───────────────────────┼────────────────────────┐
          ▼                       ▼                        ▼
   Expression Engine         Mapping Engine           Policy Engine
   Validation Engine         Lookup Engine            Event Engine
   Reactive Graph            Import Profile           Command Engine
          │                       │                        │
          └───────────────────────┼────────────────────────┘
                                  ▼
                         Capability / Data SDK
                                  │
             ┌────────────────────┼────────────────────┐
             ▼                    ▼                    ▼
        数据处理平台     业务基表(数据处理)/主数据(平台主数据服务)   Agent
             │
             ▼
 Dataset → Pipeline → Semantic Layer → 可视化控制台
```

跨模块主动调用遵循平台统一链路：调用者 → Capability / BusinessCommand →
填报与流程领域 → 必要时公共 Job / Executor。WorkflowInstance / HumanTask
负责人工业务流转，JobRun / JobAttempt
只负责后台执行机制；执行层重试、失败或取消不得直接冒充审批、退回、撤回等业务结果。Shell
/ 门户只聚合待办、通知、深链接和运行引用，不取得
BusinessRecord、Revision、WorkflowInstance 或 HumanTask 所有权。

------------------------------------------------------------------------

## 3. 核心设计原则

1.  **业务事实只记录一次。**
2.  **能自动获得的信息不要求用户重复填写。**
3.  **允许输入格式差异化，不强迫部门统一 Excel 模板。**
4.  **差异在 Mapping / Import Profile / Adapter
    层消化，业务语义保持稳定。**
5.  **Form ≠ Data。表单只是业务事实的一种交互入口。**
6.  **BusinessRecord 是权威业务事实；Dataset 是分析和加工语义。**
7.  **页面、Excel、Capture、API、Agent、Automation 最终经过统一
    Capability /
    BusinessCommand、权限和校验边界；不同调用端不建立业务后门。**
8.  **隐藏/显示属于 Presentation；字段是否赋值属于 Value
    Lifecycle，两者分离。**
9.  **动态规则可以变化，已经发生的业务事实不能随当前规则变化。**
10. **业务动作形成版本边界，UI 每次输入不形成 Revision。**
11. **复杂能力允许扩展，但执行资源必须有预算。**
12. **Draft 不影响已发布 Release；运行实例绑定不可变 Release。**

------------------------------------------------------------------------

## 4. 采集体系

### 4.1 统一输入模型

``` text
Input Source
   ├─ Form
   ├─ Quick Capture
   ├─ Excel / CSV
   ├─ Paste
   ├─ API
   ├─ Agent
   └─ Automation
        ↓
InputPayload
        ↓
Mapping / Validation
        ↓
BusinessCommand
        ↓
BusinessRecord / DomainEvent
```

Submission 仅作为 Form Runtime
的交互记录，不作为平台所有输入方式的核心长期对象。

### 4.2 最小必要人工输入

系统优先自动取得：当前身份、组织、时间、上下文、上次选择、当前项目/任务、可推导字段。用户只填写业务无法可靠自动获得的信息。

支持： - 默认值与最近值； - 上下文继承； - 连续登记； - 批量登记； -
草稿； - 快捷键； - 一键重复上一条； - 批量粘贴； - Import Profile； -
Agent 辅助补全。

### 4.3 EntryProfile

同一 RecordType
可以具有多个交互入口：快速登记、完整表单、主管修订、Excel
导入、API、Agent 等。入口不同，但最终产生同一业务语义。

------------------------------------------------------------------------

## 5. Quick Capture 快捷采集体系

### 5.1 定位

Quick Capture 用于解决"每干一次活就需要重新进入平台登记"的高频摩擦。

### 5.2 三层模型

-   **Capture Host**：Browser Extension、Web Floating Widget、Desktop
    Companion、Embedded SDK 等；
-   **Capture Adapter**：识别允许页面和允许读取的上下文；
-   **Capture
    Action**：登记完成、工时、异常、数量、结果、备注等业务动作。

### 5.3 Trigger

支持
ManualClick、Hotkey、Selection、ValueChange、DOMChange、Clipboard、ExternalEvent、APIEvent
等触发器。

### 5.4 提交等级

-   L0 Prepare：准备数据，由用户完整确认；
-   L1 Confirm：信息已准备，只需一次确认；
-   L2 Auto
    Commit：低风险、高频、明确授权场景自动提交，并提供可追溯撤销/作废能力。

自动读取与自动写入必须分开授权。

### 5.5 安全边界

Capture Host / Adapter 已在插件体系第 2 节登记为宿主类型，但宿主协议尚未发布；浏览器扩展与 Desktop Companion 尚未登记。在插件体系发布对应宿主协议并通过验收前，以上均不属首期交付范围，首期入口为 Form、Excel / CSV、Paste 与 API。


Capture Context 永远视为不可信输入。必须经过 Adapter → Mapping →
Validation → BusinessCommand。禁止扩展直接写数据库。

Adapter
必须声明：允许域/页面、可提取字段、版本、来源定位。默认不保存完整网页；需要证据时显式配置
Text Snapshot、Screenshot、DOM Fragment 或 External Record ID。

------------------------------------------------------------------------

## 6. Excel / CSV / 粘贴导入

### 6.1 不强制统一模板

平台通过 Import Profile
将不同部门现有格式映射到业务字段，不要求用户重新整理成唯一标准 Excel。

### 6.2 Import Profile

保存： - 文件/Sheet 匹配规则； - 列到字段映射； - 类型转换； -
值映射； - Lookup； - 表达式； - 校验； - SchemaVersion； -
ProfileVersion。

### 6.3 部分成功

ImportBatch 支持
ACCEPTED、REJECTED、WARNING、PENDING_RESOLUTION。异常行可修复后单独
Retry，不要求重新导入整个文件。

### 6.4 来源追踪

保存
source_asset_id、import_profile_id/version、行号/Sheet、导入人、导入时间及生成
record_id，支持从看板指标追溯至 BusinessRecord、ImportBatch
和原文件位置。

### 6.5 表格粘贴

RepeatingGroup 和批量录入均支持 Excel 二维数据 Ctrl+C/Ctrl+V，并复用
Mapping Engine，不另造第二套映射体系。

------------------------------------------------------------------------

## 7. Form Runtime：可编程业务界面

### 7.1 字段类型

-   InputField
-   ReferenceField
-   LookupField
-   CascadeField
-   ComputedField
-   EventBoundField
-   HiddenField
-   CollectionField
-   RepeatingGroup
-   RecordCollection

### 7.2 Presentation

支持
Heading、Text/Markdown、Divider、Alert、Section、Group、Grid、Tabs、Collapse、Card、Image、Link、Spacer、HTMLBlock
等。

Presentation State：Visible / Hidden / ReadOnly / Disabled / Masked。

Presentation 状态不决定字段是否被初始化或写入。

### 7.3 Expression / Validation / Condition

三者严格分离： - Expression：计算一个值； - Validation：判断是否合法； -
Condition：决定显示、行为或流程分支。

示例：

``` text
duration = datetime_diff(end_time, start_time, "minutes")
valid = end_time > start_time
total = quantity * coefficient + project_coefficient
```

### 7.4 Lookup 与基表

表单不得直接 SQL 查询数据库。所有 Lookup 经 Data SDK / Query Capability
获取权威数据，并支持 effective_at 语义以取得业务发生时适用的历史值。

### 7.5 Reactive Dependency Graph

字段、Lookup、公式建立依赖图。上游字段变化时只重算受影响节点。发布前执行循环检测、复杂度检查、Lookup
数量检查；运行时限制执行时间、Lookup 次数、递归深度和结果大小。

### 7.6 计算结果策略

支持： - Derived：按需派生； - Materialized：计算后保存； - Snapshot
Derived：保存结果及当时依赖值、规则版本和数据快照引用。

------------------------------------------------------------------------

## 8. 高代码与扩展安全等级

### L0 Declarative

Expression、Validation、Condition、Lookup、Layout。默认能力。

### L1 Sandbox Script

（受隔离宿主门禁 G-ISO 约束，见《Titanloom-实施路线与首期工程基线》；门禁通过前不开放。）受限脚本，无直接网络、DOM、凭据和数据库权限；限制
CPU、内存、执行时间和结果大小；通过 Capability SDK 获取授权能力。

### L2 HTML Block

运行于 sandboxed iframe / 独立执行域，采用
CSP、消息来源与结构校验、CSS/DOM
隔离；不得直接获得主站凭据和数据库连接。

### L3 Plugin

需要复杂 UI、网络或平台能力时进入正式插件体系：注册 → 审核 → 授权 →
Release → Runtime。

------------------------------------------------------------------------

## 9. BusinessRecord 与 Schema Registry

### 9.1 BusinessRecord

核心字段：

``` text
record_id
record_type_id
schema_version
business_key
status
occurred_at
business_date
created_at
created_by
current_revision
data JSONB
```

### 9.2 PostgreSQL 存储策略

采用 **关系核心 + JSONB 动态业务字段 + 按需物化/投影**。

治理字段和高频公共关联键不应全部埋入 JSONB。Schema Registry 可以为高频
person_id、project_id、org_id、asset_id 等声明
searchable/indexed/materialized 策略，形成关系化投影。

禁止每创建一个 Form 就动态 CREATE TABLE。

### 9.3 Schema Registry

管理：RecordType、SchemaVersion、FieldDefinition、RelationDefinition、CollectionDefinition、ExpressionDefinition、ValidationDefinition、EventBinding、IndexPolicy、CompatibilityPolicy。

Schema Registry 管业务结构契约，不承担整个 Titanloom 的万能元数据存储职责。

### 9.4 Schema 演进

历史记录绑定产生它的 SchemaVersion。新增 optional
字段可标记兼容；字段删除、类型变化等 Breaking Change
必须执行依赖分析，检查 Form、Workflow、Import Profile、Capture
Adapter、Dataset Mapping、Dashboard 等影响。

------------------------------------------------------------------------

## 10. 子表、明细与子业务对象

### 10.1 三类集合

-   CollectionField：简单值集合；
-   RepeatingGroup：依附于父记录的结构化重复明细；
-   RecordCollection：具有独立业务生命的子 BusinessRecord 集合。

### 10.2 独立对象判断

当一行具有独立
ID、权限、状态、流程、负责人、附件、修订历史、关系、生命周期或统计口径时，应考虑建模为独立
BusinessRecord，而不是普通子表行。

### 10.3 父子关系与流程关系分离

Record Relationship ≠ Workflow
Relationship。父记录可以完成，而子问题继续整改；流程也可以通过条件等待所有子记录达到指定状态。

### 10.4 行级公式

RepeatingGroup 支持 currentRow、parent、collection 作用域，以及
SUM/FILTER
等集合计算。常用操作提供图形化配置，高级需求再进入表达式/脚本。

### 10.5 大明细性能

支持虚拟滚动、分页、服务端过滤/排序、增量加载、批量编辑、批量校验和批量提交。UI
一次渲染量与业务容量上限分离。

------------------------------------------------------------------------

## 11. 时间模型与事件绑定字段

### 11.1 时间语义

必须区分： - Business Time：occurred_at、business_date； - System
Time：recorded_at、created_at、updated_at； - Workflow/Event
Time：submitted_at、approved_at、returned_at 等； - Effective
Time：规则/基表/关系从何时开始生效。

### 11.2 EventBoundField

字段可绑定业务事件，而不是仅配置 `default = now()`。

例如：

``` text
borrowed_at = timestamp(BORROW_SUBMITTED)
returned_at = timestamp(RETURN_CONFIRMED)
```

### 11.3 值来源

Field Value
Source：Manual、Default、Initial、Expression、Lookup、Reference、Agent、SystemContext、EventBinding。

Write
Policy：WriteOnce、Replace、Append、FirstEvent、LatestEvent、ManualOverride。

### 11.4 借记示例

第一次登记：borrowed_at 由借用提交事件写入，returned_at 保持
NULL。归还待办生成、打开均不等于归还；只有 ConfirmReturn Command
成功后由服务端产生 RETURN_CONFIRMED，returned_at 绑定其权威事件时间。

### 11.5 业务日/班次

TimeContext 支持
timestamp、timezone、calendar_date、business_date、shift_id，以处理跨午夜和非自然日业务周期。

------------------------------------------------------------------------

## 12. Command、TaskView 与字段策略

### 12.1 不用一张 Form 硬撑整个流程

同一 BusinessRecord 可以使用 BorrowCreate、Approval、ReturnConfirm
等不同 TaskView/EntryProfile。

### 12.2 Field Policy

按 TaskView 声明
Visible、Hidden、ReadOnly、Editable、Optional、Required、Masked。

"必填"优先定义为 Command 前置条件，而不是字段永久 required。

### 12.3 Business Command

禁止所有动作都叫 Submit。使用
Create、Submit、Approve、Reject、ConfirmReturn、Withdraw、Void、Correct
等具有业务语义的 Command。

执行链：

``` text
Command
 → Authorization
 → Preconditions
 → Validation
 → Write Policy
 → Revision/Event
 → Workflow Transition
 → Audit
```

------------------------------------------------------------------------

## 13. Revision、草稿与并发

### 13.1 Revision 边界

Draft 内可持续修改；执行正式业务 Command 时形成
Revision。退回后重新提交形成新 Revision，保留旧值和差异。

### 13.2 撤销/作废

已形成正式事实后不使用物理 DELETE 表达业务撤销，应通过 VOID/CORRECT 等
Revision/Event 留痕。

### 13.3 并发控制

BusinessRecord 默认采用 Optimistic Concurrency Control。Command 携带
expected_revision；版本不一致返回
CONFLICT，并提供差异比较/重新应用。只有特殊场景使用独占锁。

------------------------------------------------------------------------

## 14. Workflow Engine

### 14.1 V1 基线节点

-   Start / End
-   HumanTask
-   AgentCall（向智能管家发起并等待一个 AgentTask；AgentTask 及其生命周期归智能管家，本节点只保存引用与业务结果）
-   ServiceTask
-   ExclusiveGateway
-   ParallelGateway
-   TimerWait
-   EventWait
-   SubProcess

支持提交、审批、退回、撤回、重新提交、确认、转办、加签/会签、超时、条件分支和并行。

V1 不实现完整 BPMN 2.0 引擎；通过 NodeType Plugin 保留未来扩展上限。

### 14.2 流程与业务对象分离

WorkflowInstance 不应成为 BusinessRecord
的内部状态机实现。业务对象与流程实例通过明确关系连接，允许同一记录触发不同流程或子记录拥有独立流程。

### 14.3 动态分支与办理人解析

流程可引用字段、基表和组织关系决定路径与办理人。动态 Assignment Rule
解析后必须保存 resolved_person 和 resolved_at
快照，避免未来组织变化污染历史。

### 14.4 Delegation / 代理

Task 同时记录
assigned_to、acting_by、delegation_id。实际执行人、原办理人和代理依据必须可审计。Agent
最小委托也复用该身份/委托模型。

### 14.5 批量办理

Task Center 支持批量批准、退回、分配、补字段。批量 Command
必须逐记录授权、逐记录校验并返回逐条结果；除业务明确要求外，不强制整批原子失败。

------------------------------------------------------------------------

## 15. 权限与治理

完整授权链：

``` text
Identity
 → Platform Capability
 → Resource Scope
 → Record Scope
 → Command Permission
 → Field Policy
```

Record Scope 支持本人、组织、项目、授权集合等范围规则。

平台权限/Capability
是权威授权层；数据库行级策略可作为纵深防御，但不得取代业务授权体系。

Agent、人、API、Automation 均不得绕过 Capability /
BusinessCommand、权限和 Data
SDK。跨领域调用必须按目标能力、目标资源和当前身份/委托重新鉴权；页面上下文、Workflow
Task 或 Agent 上下文均不传递为隐式权限。

------------------------------------------------------------------------

## 16. Provenance、附件与审计

### 16.1 字段级 Provenance

字段来源可标记
USER、LOOKUP、EXPRESSION、CAPTURE、IMPORT、AGENT、EVENT、SYSTEM
等，并记录必要的规则版本、数据快照、Adapter/Profile 等引用。

### 16.2 附件

AttachmentRef 记录
asset_id、record_id、revision_id、field_id、purpose、uploaded_by、uploaded_at，使附件与当时业务状态和字段语义绑定。

### 16.3 Audit

必须可回答：谁在什么时候通过什么入口、以什么权限、执行什么
Command、改变了什么字段、依据什么规则/版本、产生了什么事件。

------------------------------------------------------------------------

## 17. Agent 接入

Agent 是平台管家，但不是数据库超级管理员。

允许 Agent：读取授权上下文、补全、核验、分类、生成
InputPayload、提出修订、执行被授权 Command；AgentTask
本身由智能管家领域拥有，填报与流程平台只保存必要的关联引用、委托与业务结果。

禁止 Agent：旁路数据库、绕过 Capability、突破调用者/委托范围、修改不可变
Release、把凭据带入模型上下文。

Agent 生成的数据仍经过 Mapping / Validation / Command，并记录
Provenance。

------------------------------------------------------------------------

## 18. 与数据处理平台的数据契约

BusinessRecord 通过受控数据契约 / Data SDK
向数据处理平台提供授权数据，并通过 DomainEvent
表达已经发生的领域事实与变更语义。DomainEvent
不是跨域写接口，数据处理平台不得通过消费事件取得修改 BusinessRecord
的权力。数据处理平台形成 Dataset、Pipeline、Semantic Layer 和 Published
Dataset。

示例：

``` text
ProductionRecord ─┐
Attendance ────────┼→ Pipeline → Efficiency Dataset → Dashboard
Project Master ────┘
```

数据处理中心发现业务事实异常时，应形成可追踪的 ValidationResult / Issue
或通过受控 Capability / BusinessCommand 请求所属领域修订；不得静默
UPDATE 权威 BusinessRecord。是否进入 Workflow
由填报与流程领域按业务规则决定，不由数据处理平台直接改写流程状态。

------------------------------------------------------------------------

## 19. Draft / Release 与运行稳定性

Form、Schema、Workflow、Import Profile、Capture Adapter、Expression
等可编辑定义必须经过：

``` text
Draft → Validate → Dependency Analysis → Test → Publish → Immutable Release
```

运行中的 FormInstance、WorkflowInstance、ImportBatch 等绑定对应
release_id。新 Release
默认不改变已经运行的实例，除非执行显式、可审计的迁移策略。

本节 Release
专指填报与流程领域中可编辑定义经过发布边界形成的不可变运行定义；它不等同于
BusinessRecord 的 Revision，也不等同于公共 JobRun / JobAttempt
的执行状态。BusinessRecord 的正式业务变化继续通过 Revision / DomainEvent
表达。

------------------------------------------------------------------------

### 19.1 异常与恢复

| 情形 | 预期行为 |
| --- | --- |
| Command 提交后响应丢失 | 按 commandId 与幂等键查回原结果，不重复生成 Revision 与 DomainEvent |
| 并发修改 | expected_revision 不符返回 CONFLICT，提供差异与重新应用，不静默覆盖 |
| 导入部分失败 | ImportBatch 逐行状态保留，异常行可单独 Retry，不重导整份文件 |
| DomainEvent 投递失败或乱序 | 业务事务内写 outbox，至少一次投递，消费者按稳定 ID 幂等；数据处理不可用不阻断业务提交 |
| 流程实例遇到新 Release | 运行实例继续绑定原 release_id，迁移需显式、可审计 |
| JobRun 重试或取消 | 只影响执行机制，不改变审批、退回等业务结果 |
| 动态办理人解析后组织变化 | 以保存的 resolved_person 与 resolved_at 为准 |
| 撤权发生在任务等待期间 | 办理时重新鉴权，旧授权快照不继续使用 |

## 20. 资源治理与性能基线

平台必须对以下能力设资源预算： - 表达式执行时间； - 脚本 CPU/内存； -
Lookup 次数； - 依赖图深度； - Import 行数/批次大小； - 单次批量 Command
数量； - RepeatingGroup 渲染窗口； - 附件大小与数量； - Capture
证据大小； - Workflow 并行任务数。

资源限制由后台治理中心统一配置、监控和审计；业务能力契约不因默认资源额度而人为封顶。

------------------------------------------------------------------------

## 21. V1 明确不做

-   完整 BPMN 2.0 实现；
-   任意代码直接访问数据库；
-   任意 HTML/脚本进入主站同权限 DOM；
-   所有动态字段自动建索引；
-   每个 Form 动态 CREATE TABLE；
-   Workflow 自建人员/组织主数据；
-   Form Runtime 承担大规模 ETL；
-   Capture 默认保存完整网页；
-   Agent 绕过 Business Command；
-   Hidden 自动等价于初始化；
-   默认值 `now()` 代替业务事件时间；
-   数据处理平台静默覆盖业务事实。

------------------------------------------------------------------------

## 22. 核心对象清单

``` text
RecordType
SchemaVersion
FieldDefinition
RelationDefinition
CollectionDefinition

FormSchema
FormRelease
EntryProfile
TaskView

InputPayload
ImportProfile
ImportBatch
MappingRule
CaptureAdapter
CaptureAction
CaptureContext

BusinessCommand
BusinessRecord
RecordRevision
DomainEvent
AttachmentRef
Provenance

ExpressionDefinition
ValidationDefinition
ConditionDefinition
LookupDefinition
EventBinding

WorkflowDefinition
WorkflowRelease
WorkflowInstance
HumanTask
Gateway
Delegation

ValidationResult
AuditRecord
```

公共能力直接引用 Titanloom 统一
Identity、Permission、Capability、Asset、Dataset、Query、JobRun /
JobAttempt、Audit、Notification、Plugin/Extension
契约，不在本子平台重复定义。WorkflowInstance /
HumanTask、BusinessRecord / RecordRevision 等领域对象仍由本平台拥有；公共
Job 只提供后台执行机制，不取得流程或业务事实状态所有权。

------------------------------------------------------------------------

## 23. 最终架构结论

填报与流程平台不是为了让用户"填更多表"，而是为了以最低人工摩擦捕获真实业务事实，并保证这些事实在多入口、多阶段、多角色、多版本环境中仍然可解释、可追溯、可治理。

最终形成四条稳定主线：

1.  **入口自由**：Form、Capture、Excel、Paste、API、Agent、Automation
    均可进入；
2.  **事实稳定**：BusinessRecord、Revision、Event、Provenance
    保证业务事实不会因界面或规则变化失真；
3.  **流程解耦**：TaskView、Command、Workflow 负责业务流转，不把一张
    Form 当作整个流程；
4.  **分析统一**：业务事实经统一数据出口进入数据处理平台，再完成跨域关联、计算、语义建模和可视化。

该设计贯彻 Titanloom
的总体原则：**能力不设人为上限，资源必须有明确上限；架构面向未来，生产基线面向稳定。**

------------------------------------------------------------------------

## 24. 验收基线

下列场景用于工程验收；通过条件必须有自动化测试或可复现演练证据，不以截图代替。

| 编号 | 场景 | 通过条件 |
| --- | --- | --- |
| FW01 | 同一 Command 重复提交、响应丢失 | 同幂等键只产生一次 Revision 与一次 DomainEvent；超时后查回原 commandId |
| FW02 | 同一记录两人并发修改 | 后到者返回 CONFLICT 并给出差异，不静默覆盖 |
| FW03 | 退回后重新提交 | 形成新 Revision，旧值与差异保留；已确认事实不物理删除 |
| FW04 | 导入含错误行的 Excel | 合格行 ACCEPTED，异常行 REJECTED / PENDING_RESOLUTION 并可单独 Retry；可从记录追溯到文件、Sheet 与行号 |
| FW05 | 重复导入同一文件 | 按文件哈希与业务键去重，指标不被放大 |
| FW06 | 基表或 Schema 新版本发布 | 已运行实例与历史记录绑定原 Release / SchemaVersion，不被新规则改写 |
| FW07 | 审批人转办、代理办理 | HumanTask 记录 assigned_to、acting_by、delegation_id，撤权后后续办理被拒 |
| FW08 | 动态办理人解析 | 保存 resolved_person 与 resolved_at，组织调整不改变历史 |
| FW09 | 批量办理 | 逐条授权、逐条校验、逐条返回结果，不强制整批原子失败 |
| FW10 | 归还待办生成或打开 | returned_at 不变，仅 ConfirmReturn 成功后由 RETURN_CONFIRMED 写入 |
| FW11 | 表达式循环依赖、超预算 | 发布前被拒绝；运行时受时间、Lookup 次数、递归深度限制 |
| FW12 | 数据处理或模型不可用 | 填报、审批、导入照常提交；DomainEvent 经 outbox 后补投递 |
| FW13 | Agent 越权、不可信指令 | 服务端按调用者与委托拒绝；Provenance 记录 AGENT 来源 |

## 25. 建设顺序与完成标准

阶段编号以《Titanloom-实施路线与首期工程基线》的 M 序列为准，本节只声明本领域内部顺序。

- **阶段一（对应 M2）**：RecordType / Schema Registry、Form Runtime 的 L0 能力（Expression / Validation / Condition / Lookup）、BusinessCommand、BusinessRecord / Revision、乐观并发、草稿、Excel / CSV / 粘贴导入与 Import Profile、HumanTask 与最小 Workflow（Start / End / HumanTask / 排他网关 / 退回 / 重新提交）、附件经 Asset、Provenance 与 Audit、受控数据出口与 DomainEvent（outbox）。
- **阶段二**：并行网关、TimerWait / EventWait、SubProcess、转办 / 加签 / 会签、批量办理、RepeatingGroup 大明细性能、EventBoundField 完整语义、AgentCall 节点。
- **阶段三**：L1 脚本（待 G-ISO）、L2 HTML Block、Quick Capture 与 Capture Host（待插件宿主协议）、NodeType Plugin。

完成标准：第 24 节 FW01–FW13 通过；总框架 15.3 中与填报相关的场景通过；本方案不代表软件已实现。
