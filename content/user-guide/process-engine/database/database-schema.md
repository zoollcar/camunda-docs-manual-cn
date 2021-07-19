---

title: '数据库表结构'
weight: 10
menu:
  main:
    identifier: "user-guide-process-engine-database-schema"
    parent: "user-guide-process-engine-database"

---

流程引擎的数据库由多个表组成。
表的名称都以ACT开头的。后面是说明表的用途的两个字符标识。这个用途标志也将与服务API大致匹配。

* `ACT_RE_*`: `RE`代表资源库。带有这个前缀的表包含 "静态 "信息，如流程定义和流程资源（图像、规则等）。
* `ACT_RU_*`: `RU`代表运行时。包含流程实例、用户任务、变量、Job等的运行时数据。引擎只在流程实例执行期间存储运行时数据，并在流程实例结束时删除记录。这使运行时表保持小而快。
* `ACT_ID_*`: `ID`代表身份信息。这些表包含身份信息有用户、组等。
* `ACT_HI_*`: `HI`代表历史。这些是包含历史数据的表，如过去的流程实例、变量、任务等。
* `ACT_GE_*`: 一般数据，在各种使用情况下使用。

流程引擎的主要表是流程定义、执行、任务、变量和事件订阅的实体。它们的关系显示在下面的UML模型中：

{{< img src="../../img/database-schema.png" title="Database Schema" >}}


## 流程定义 (`ACT_RE_PROCDEF`)

`ACT RE Procdef` 表包含所有已部署的流程定义。它包括有版本详细信息，资源名称或暂停状态等信息。


## 执行 (`ACT_RU_EXECUTION`)

`ACT_RU_EXECUTION` 表包含所有当前的执行。它包括有流程定义，父执行，业务密钥，当前活动和关于执行状态的不同元数据的信息。


## 任务 (`ACT_RU_TASK`)

`ACT_RU_TASK` 表包含所有正在运行的流程实例的所有执行中的任务。它包括有相应的流程实例，执行以及元数据等信息，例如创建时间，受让人或截止日期。


## 变量 (`ACT_RU_VARIABLE`)

`ACT_RU_VARIABLE` 表包含所有当前设置的流程或任务变量。它包括变量的名称、类型和值以及相应流程实例或任务的信息。


## 事件订阅 (`ACT_RU_EVENT_SUBSCR`)

`ACT_RU_EVENT_SUBSCR` 表包含所有当前现有的事件订阅。它包括预期事件的类型、名称和配置以及相应的流程实例和执行的信息。

## 表结构日志 (`ACT_GE_SCHEMA_LOG`)

`ACT_GE_SCHEMA_LOG` 表记录数据库表结构版本的历史记录。在进行表结构修改时都会写入记录。在数据库创建时会写入一条初试信息，以后每次修改都会记录`id` 、版本、数据库更新日期以及时间戳。

要从表结构日志中查询条目，可以使用查询API：
```java
List<SchemaLogEntry> entries = managementService.createSchemaLogQuery().list();
```

## Metrics日志 (ACT_RU_METER_LOG)

The `ACT_RU_METER_LOG` table contains a collection of runtime metrics that can help draw conclusions about usage, load and performance of the Camunda Platform. Metrics are reported as numbers in the Java `long` range and count the occurrence of specific events. Please find detailed information about how metrics are collected in the [Metrics User Guide]({{< ref "/user-guide/process-engine/metrics.md">}}).

The default configuration of the [MetricsReporter]({{< ref "/user-guide/process-engine/metrics.md#metrics-reporter">}}) will create one row per [metric]({{< ref "/user-guide/process-engine/metrics.md#built-in-metrics">}}) in `ACT_RU_METER_LOG` every 15 minutes.

{{< note title="Heads Up!" class="warning" >}}
If you are an enterprise customer, your license agreement might require you to report some metrics annually. Please store `root-process-instance-start`, `activity-instance-start`, `executed-decision-instances` and `executed-decision-elements` metrics for at least 18 months until they were reported.
{{< /note >}}

## Task Metrics 日志 (ACT_RU_TASK_METER_LOG)

The `ACT_RU_TASK_METER_LOG` table contains a collection of task related metrics that can help draw conclusions about usage, load and performance of the BPM platform. Task metrics contain a pseudonymized and fixed-length value of task assignees and their time of appearance. Please find detailed information about how task metrics are collected in the [Metrics User Guide]({{< ref "/user-guide/process-engine/metrics.md">}}).

Every assignment of a task to an assignee will create one row in `ACT_RU_TASK_METER_LOG`.

{{< note title="Heads Up!" class="warning" >}}
If you are an enterprise customer, your license agreement might require you to report some metrics annually. Please store task metrics for at least 18 months until they were reported.
{{< /note >}}

# 实体关系图

{{< note title="" class="info" >}}
  该数据库不是 **公共API** 的一部分。数据库的机构可能会在 次要 或 主要 版本更新时发生变化。

  **请注意：**
  以下图表基于MySQL数据库的。对于其他数据库，图可能略有不同。
{{< /note >}}


下面的实体关系图将数据库表和它们的显式外键约束可视化，按照BPMN引擎、DMN引擎、CMMN引擎、历史引擎和身份引擎进行分组。请注意，这些图并没有将表之间的隐性联系可视化。


## BPMN引擎

{{< img src="../../img/erd_715_bpmn.svg" title="BPMN Tables" >}}


## DMN引擎

{{< img src="../../img/erd_715_dmn.svg" title="DMN Tables" >}}


## CMMN引擎

{{< img src="../../img/erd_715_cmmn.svg" title="CMMN Tables" >}}


## 历史

为了允许不同的配置并保持表的灵活性，历史表不包含外键约束。

{{< img src="../../img/erd_715_history.svg" title="History Tables" >}}


## 身份

{{< img src="../../img/erd_715_identity.svg" title="Identity Tables" >}}