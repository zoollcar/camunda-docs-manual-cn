---

title: '历史和审批事件日志'
weight: 140

menu:
  main:
    identifier: "user-guide-process-engine-history"
    parent: "user-guide-process-engine"

---


历史事件流提供关于已执行的流程实例的审批信息。

{{< img src="../img/process-engine-history.png" title="流程引擎历史" >}}

流程引擎维护数据库内运行的流程实例的状态。这包括在流程实例达到等待状态时将其*写*(1)到数据库中，并在流程继续执行时*读*(2)该状态。我们称这个数据库为 *运行时数据库（runtime database）* 。除了维护运行时状态外，流程引擎还创建了一个审批日志，提供关于已执行流程实例的审批信息。我们称这个事件流为 *历史事件流（history event stream）* （3）。构成这个事件流的各个事件被称为 *历史事件（History Events）* ，包含关于已执行的流程实例、活动实例、改变的流程变量等的数据。
在默认配置中，流程引擎将简单地把这个事件流写入（4.）*历史数据库*。历史服务（HistoryService）的API允许查询（5）这个数据库。历史数据库和历史服务是可选的组件；如果历史事件流没有被记录到历史数据库中，或者如果用户选择将事件记录到不同的数据库，流程引擎仍然能工作，它仍然能够填充历史事件流。这是可能的，因为BPMN 2.0核心引擎组件不从历史数据库读取状态。也可以配置流程引擎的 "historyLevel" 配置项，来配置记录的数据量。

由于流程引擎不依赖历史数据库的存在来生成历史事件流，因此可以提供不同的后端实现来存储历史事件流。默认的后端实现是  "DbHistoryEventHandler" ，它将事件流记录到历史数据库。可以替换后端实现，为历史事件日志提供一个自定义的存储机制。


# 选择历史记录级别

历史级别控制流程引擎通过历史事件流提供的数据量。以下设置是开箱即用的：

* `NONE`: 不会记录历史事件。
* `ACTIVITY`: 以下事件将被记录：
    * 流程实例 START, UPDATE, END, MIGRATE: 当流程实例被 started, updated, ended 和 migrated 时触发。
    * 案例实例 CREATE, UPDATE, CLOSE: 当案例实例被 created, updated 和 closed 时触发。
    * 活动实例 START, UPDATE, END, MIGRATE: 当活动实例被 started, updated, ended 和 migrated 时触发。
    * 案例活动实例 CREATE, UPDATE, END: 当案例活动实例被 created, updated 和 ended 时触发。
    * 任务实例 CREATE, UPDATE, COMPLETE, DELETE, MIGRATE: 当任务实例被 created, updated (i.e., re-assigned, delegated etc.), completed, deleted 和 migrated 时触发。
* `AUDIT`: 除了`ACTIVITY`级别提供的事件，还记录了以下事件：
    * 流程变量 CREATE, UPDATE, DELETE, MIGRATE: 当流程变量被 created, updated, deleted 和 migrated时触发，默认历史记录后端（DbHistoryEventHandler）将变量实例事件写入历史变量实例数据表中。因为在更新流程变量时，此表中的行会更新，所以只有流程变量的最后一个值将可用。
* `FULL`: 除了`AUDIT`级别提供的事件，还记录了以下事件：
    * 表单属性 UPDATE：当表单属性被 created 或 updated 时被触发。
    * 默认的历史后端（DbHistoryEventHandler）将历史变量的更新情况写入数据库。这使得使用历史服务查看一个流程变量的中间值成为可能。
    * 用户操作日志 UPDATE: 当用户执行操作时触发，例如承接（claim）用户任务、委派（delegating）用户任务等。
    * 事件 CREATE, DELETE, RESOLVE, MIGRATE: 当事件被 created, deleted, resolved 和 migrated 时记录。
    * 历史Job日志 CREATE, FAILED, SUCCESSFUL, DELETED: 当Job被 created, execution, successful 或 deleted 时记录。
    * 决策实例评估：当DMN引擎进行决策评估时记录。
    * 批处理 START, END: 当批处理被 started 或 ended 时触发。
    * 身份 links ADD, DELETE: 当 身份link 被 added, deleted 以及设置或改变用户任务的受让人、拥有人时被记录。
    * 历史外部任务日志 CREATED, DELETED, FAILED, SUCCESSFUL: 作为外部任务被 created, deleted 或外部任务执行者报告了成功或失败时会记录。
* `AUTO`: 如果你打算在同一个数据库中运行多个引擎，那么 `AUTO` 级别是很有用的。在这种情况下，所有的引擎都必须使用相同的历史级别。与其手动保持配置同步，不如使用 `auto` 级别，引擎会自动确定数据库中已经配置的级别。如果没有找到，则使用默认值 `audit` 。请记住。如果你打算使用自定义历史级别，你必须为每个配置注册自定义级别，否则会出现异常。

如果你需要自定义历史事件的记录量，你可以提供一个自定义的 {{< javadocref page="?org/camunda/bpm/engine/impl/history/producer/HistoryEventProducer.html" text="HistoryEventProducer" >}} 实现并将其应用到流程引擎配置中。


# 设置历史记录级别

历史记录级别可以作为流程引擎配置中的一个属性提供。根据流程引擎的配置方式，该属性可以使用Java代码进行设置：

```java
ProcessEngine processEngine = ProcessEngineConfiguration
  .createProcessEngineConfigurationFromResourceDefault()
  .setHistory(ProcessEngineConfiguration.HISTORY_FULL)
  .buildProcessEngine();
```

也可以使用Spring XML或部署描述符（bpm-platform.xml, processes.xml）来设置。当使用Camunda JBoss子系统时，该属性可以通过JBoss配置（standalone.xml, domain.xml）来设置：

```xml
<property name="history">audit</property>
```

请注意，当使用默认的历史后台实现时，历史级别被存储在数据库中，以后不能更改。

{{< note title="History levels and Cockpit" class="info" >}}
[Camunda Platform Cockpit]({{< ref "/webapps/cockpit/_index.md" >}}) 应用程序在历史级别设置为 "FULL" 时工作得最好。较低的历史级别将禁用某些与历史有关的功能。
{{< /note >}}

# 默认历史后台实现执行方式

默认的历史后台实现将历史事件写到适当的数据库表中。然后可以使用 `HistoryService`或使用REST API来查询这些数据库表。


## 历史实体

有以下历史实体，与运行时数据相反，它们在流程和案例实例完成后，将留在数据库中。

* `HistoricProcessInstances` 包含有关当前和过去的流程实例的信息。
* `HistoricVariableInstances` 包含有关在流程实例中保持的最新状态的信息。
* `HistoricCaseInstances` 包含有关当前和过去案例实例的信息。
* `HistoricActivityInstances` 包含有关一项执行活动的信息。
* `HistoricCaseActivityInstances` 包含关于单个执行案例活动的信息。
* `HistoricTaskInstances` 包含有关当前和过去（已完成和删除）任务实例的信息。
* `HistoricDetails` 包含与历史流程实例，活动实例或任务实例相关的各种信息。
* `HistoricIncidents` 包含有关当前和过去的信息（即，删除或解决的）事件。
* `UserOperationLogEntry` 日志条目包含有关用户执行操作的信息。例如创建新任务，完成任务等操作。
* `HistoricJobLog` 包含有关Job执行的信息。日志记录有关Job生命周期的详细信息。
* `HistoricDecisionInstance` 包含关于决策的单个评估的信息，包括输入和输出值。
* `HistoricBatch` 包含有关当前和过去批处理的信息。
* `HistoricIdentityLinkLog` 包含有关当前和过去的identity links变更信息(added, deleted, 受让人、拥有人的设置或改变)。
* `HistoricExternalTaskLog` 包含有关外部任务执行信息。提供了有关外部任务的生命周期的详细信息。


## 历史流程实例

对于每个流程实例，流程引擎将在历史数据库中创建单个记录，并在流程执行期间继续更新此记录。每个历史流程实例记录都可以获得分配的以下状态：

*  ACTIVE - 运行中的流程实例
*  SUSPENDED - 暂停的流程实例
*  COMPLETED - 通过正常结束事件完成的
*  EXTERNALLY_TERMINATED - 外部终止，例如通过REST API
*  INTERNALLY_TERMINATED - 内部终止，例如通过终止边界事件

以下状态可以被例如 REST API 或 Cockpit 的外部行为触发：ACTIVE, SUSPENDED, EXTERNALLY_TERMINATED。

## 查询历史记录

HistoryService 具有如下查询方法 `createHistoricProcessInstanceQuery()`,
`createHistoricVariableInstanceQuery()`, `createHistoricCaseInstanceQuery()`,
`createHistoricActivityInstanceQuery()`, `createHistoricCaseActivityInstanceQuery()`,
`createHistoricDetailQuery()`,
`createHistoricTaskInstanceQuery()`,
`createHistoricIncidentQuery()`,
`createUserOperationLogQuery()`,
`createHistoricJobLogQuery()`,
`createHistoricDecisionInstanceQuery()`,
`createHistoricBatchQuery()`,
`createHistoricExternalTaskLogQuery` 和 `createHistoricIdentityLinkLogQuery()`

它们可用于查询历史记录。

下面是几个例子，展示了历史查询API的一些可能性。对这些可能性的完整描述可以在 `org.camunda.bpm.engine.history` 包的Javadocs中找到。

**HistoricProcessInstanceQuery**

Get the ten `HistoricProcessInstances` that are finished and that took the most time to complete (the longest duration) of all finished processes with definition 'XXX'.

```java
historyService.createHistoricProcessInstanceQuery()
  .finished()
  .processDefinitionId("XXX")
  .orderByProcessInstanceDuration().desc()
  .listPage(0, 10);
```

**HistoricCaseInstanceQuery**

Get the ten `HistoricCaseInstances` that are closed and that took the most time to be closed (the longest duration) of all closed cases with definition 'XXX'.

```java
historyService.createHistoricCaseInstanceQuery()
  .closed()
  .caseDefinitionId("XXX")
  .orderByCaseInstanceDuration().desc()
  .listPage(0, 10);
```


**HistoricActivityInstanceQuery**

Get the last `HistoricActivityInstance` of type 'serviceTask' that has been finished in any process that uses the processDefinition with id 'XXX'.

``` java
historyService.createHistoricActivityInstanceQuery()
  .activityType("serviceTask")
  .processDefinitionId("XXX")
  .finished()
  .orderByHistoricActivityInstanceEndTime().desc()
  .listPage(0, 1);
```

**HistoricCaseActivityInstanceQuery**

Get the last `HistoricCaseActivityInstance` that has been finished in any case that uses the caseDefinition with id 'XXX'.

``` java
historyService.createHistoricCaseActivityInstanceQuery()
  .caseDefinitionId("XXX")
  .finished()
  .orderByHistoricCaseActivityInstanceEndTime().desc()
  .listPage(0, 1);
```

**HistoricVariableInstanceQuery**

Get all HistoricVariableInstances from a finished process instance with id 'XXX', ordered by variable name.

```java
historyService.createHistoricVariableInstanceQuery()
  .processInstanceId("XXX")
  .orderByVariableName().desc()
  .list();
```

**HistoricDetailQuery**

The next example gets all variable-updates that have been done in process with id '123'. Only HistoricVariableUpdates will be returned by this query. Note that it's possible for a certain variable name to have multiple HistoricVariableUpdate entries, one for each time the variable was updated in the process. You can use orderByTime (the time the variable update was done) or orderByVariableRevision (revision of runtime variable at the time of updating) to find out in what order they occurred.

```java
historyService.createHistoricDetailQuery()
  .variableUpdates()
  .processInstanceId("123")
  .orderByVariableName().asc()
  .list()
```

The next example gets all variable updates that were performed on the task with id '123'. This returns all HistoricVariableUpdates for variables that were set on the task (task local variables), and NOT on the process instance.

```java
historyService.createHistoricDetailQuery()
  .variableUpdates()
  .taskId("123")
  .orderByVariableName().asc()
  .list()
```

**HistoricTaskInstanceQuery**

Get the ten HistoricTaskInstances that are finished and that took the most time to complete (the longest duration) of all tasks.

```java
historyService.createHistoricTaskInstanceQuery()
  .finished()
  .orderByHistoricTaskInstanceDuration().desc()
  .listPage(0, 10);
```

Get HistoricTaskInstances that are deleted with a delete reason that contains 'invalid' and that were last assigned to user 'jonny'.

```java
historyService.createHistoricTaskInstanceQuery()
  .finished()
  .taskDeleteReasonLike("%invalid%")
  .taskAssignee("jonny")
  .listPage(0, 10);
```

**HistoricIncidentQuery**

Query for all resolved incidents:

```java
historyService.createHistoricIncidentQuery()
  .resolved()
  .list();
```

**UserOperationLogQuery**

Query for all operations performed by user 'jonny':

```java
historyService.createUserOperationLogQuery()
  .userId("jonny")
  .listPage(0, 10);
```

**HistoricJobLogQuery**

Query for successful historic job logs:

```java
historyService.createHistoricJobLogQuery()
  .successLog()
  .list();
```

**HistoricDecisionInstanceQuery**

Get all HistoricDecisionInstances from a decision with key 'checkOrder' ordered by the time when the decision was evaluated.

```java
historyService.createHistoricDecisionInstanceQuery()
  .decisionDefinitionKey("checkOrder")
  .orderByEvaluationTime()
  .asc()
  .list();
```

Get all HistoricDecisionInstances from decisions that were evaluated during the execution of the process instance with id 'XXX'. The HistoricDecisionInstances contains the input values on which the decision was evaluated and the output values of the matched rules.

```java
historyService.createHistoricDecisionInstanceQuery()
  .processInstanceId("XXX")
  .includeInputs()
  .includeOutputs()
  .list();
```

**HistoricBatchQuery**

Get all historic process instance migration batches ordered by id.

```java
historyService.createHistoricBatchQuery()
  .type(Batch.TYPE_PROCESS_INSTANCE_MIGRATION)
  .orderById().desc()
  .list();
```
**HistoricIdentityLinkLogQuery**

Query for all identity links that are related to the user 'demo'.

```java
historyService.createHistoricIdentityLinkLogQuery()
  .userId("demo")
  .list();
```

**HistoricExternalTaskLogQuery**

Query for failed historic external task logs:

```java
historyService.createHistoricExternalTaskLogQuery()
  .failureLog()
  .list();
```

## 历史报告

你可以使用报告部分来检索自定义统计信息和报告。目前，我们支持以下类型的报告：

* [实例持续时间报告]({{< relref "#实例持续时间报告" >}})
* [任务报告]({{< relref "#任务报告" >}})
* [完成实例报告]({{< relref "#完成实例报告" >}})



### 实例持续时间报告

查询关于已完成流程实例的持续时间的报告，按指定的时期分组。这些报告包括所有已完成的流程实例的最大、最小和平均持续时间，这些实例是在指定时期开始的。下面的代码片段查询了自引擎启动以来每个月的报告：

```java
historyService
  .createHistoricProcessInstanceReport()
  .duration(PeriodUnit.MONTH);
```

当前支持的时间刻度是 `MONTH` 和定于于 `org.camunda.bpm.engine.query.PeriodUnit` 的 `QUARTER` .

要裁剪报告，可以使用以下方法 ``HistoricProcessInstanceReport``:

* ``startedBefore``: 只考虑在给定日期之前启动的历史流程实例。
* ``startedAfter``: 只考虑在给定日期之后启动的历史流程实例。
* ``processDefinitionIdIn``: 只考虑给定流程定义Id对应的所有流程实例。
* ``processDefinitionKeyIn``: 只考虑一组给定的流程定义Id对应的流程实例。

`startedBefore` 和 `startedAfter` 使用 `java.util.Date` (弃用) 或 `java.util.Calendar` 对象作为输入。

例如，人们可以查询从现在之前启动的所有历史流程实例并获得他们的持续时间：

 ```java
Calendar calendar = Calendar.getInstance();
historyService.createHistoricProcessInstanceReport()
  .startedBefore(calendar.getTime())
  .duration(PeriodUnit.MONTH);
 ```

### 任务报告

检索已完成任务的报告。对于任务报告，有两种可能的报告类型：计数和持续时间：

如果你在报告查询的最后使用`countByProcessDefinitionKey`或`countByTaskName`方法，报告包含一个已完成任务计数的列表，其中一个条目包含任务名称、任务的定义键、流程定义ID、流程定义键、流程定义名称和指定键在特定时期完成多少任务的计数。方法`countByProcessDefinitionKey`和`countByTaskName`然后根据'定义键'或'任务名称'的标准来分组计数报告。要检索按任务名称分组的任务计数报告，可以执行以下查询。

```java
historyService
  .createHistoricTaskInstanceReport()
  .countByTaskName();
```

如果报告类型被设定为持续时间，报告包含在特定时期内所有完成的任务实例的最小、最大和平均持续时间值。

```java
historyService
  .createHistoricTaskInstanceReport()
  .duration(PeriodUnit.MONTH);
```

支持的周期时间和查询的局限性与[实例期限报告]({{< relref "#实例持续时间报告" >}})类似。

### 完成实例报告

检索已完成的流程、决定或案例实例的报告。该报告帮助用户调整定义的历史生存时间。他们可以看到历史数据的摘要，可以在历史清理后进行清理。输出的字段是definition id, key, name, version, 完成的实例计数和"可清理"的实例的实例计数。

```java
historyService
  .createHistoricFinishedProcessInstanceReport()
  .list();

historyService
  .createHistoricFinishedDecisionInstanceReport()
  .list();

historyService
  .createHistoricFinishedCaseInstanceReport()
  .list();
```

## 根据历史事件的发生情况进行部分排序

有时你想按照历史事件发生的顺序来排序。请注意这里不能使用时间戳排序。

大多数历史事件都包含一个时间戳，它标志着事件所代表的行为发生的时间。然而，一般来说，这个时间戳不能用来对历史事件进行排序。因为流程引擎可以在多个集群节点上运行。

* 在一台机器上，时钟可能会由于运行时的网络同步而改变， 
* 在一个集群中，发生在一个流程实例中的事件可能会在不同的节点上产生，其中的时钟可能无法精确地同步到纳秒级。

为了解决这个问题，Camunda引擎产生了序列号，可以用来对历史事件的发生时间进行 *部分* 的排序。

在BPMN层面上，这意味着并发活动的实例（例如：在一个并行网关之后的不同并行分支上的活动）不能相互比较。属于happens-before关系的活动实例（在BPMN层面具有向后依赖顺序的活动实例）将根据该关系排序。

例如：

```java
List<HistoricActivityInstance> result = historyService
  .createHistoricActivityInstanceQuery()
  .processInstanceId("someProcessInstanceId")
  .orderPartiallyByOccurrence()
  .asc()
  .list();
```

请注意，在这个例子中，返回的历史活动实例的列表只是部分排序，如上所述。它保证了相关的活动实例是按它们的发生率排序的。不相关的活动实例的排序是随机的，不被保证。


# 用户操作日志

用户操作日志包含许多API操作的条目，可用于审计目的。它提供了关于执行的操作的数据以及关于操作中所涉及的更改的详细信息。在登录用户会话中执行操作时记录操作。要使用操作日志，必须将流程引擎历史级别设置为 “FULL”。

## 不考虑用户登录状态，记录用户操作日志

如果希望无论是否在用户登录的情况下都记录日志，那么可以将流程引擎的 "restrictUserOperationLogToAuthenticatedUsers" 配置项设为 "false"。

## 访问用户操作日志

用户操作日志可以通过Java API访问。调用`historyService.createUserOperationLogQuery().execute()`，历史服务可以用 `UserOperationLogQuery` 来执行。该查询可以通过各种过滤器进行限制。该查询也[在REST API中访问]({{< ref "/reference/rest/history/user-operation-log/get-user-operation-log-query.md" >}}).


## 用户操作日志条目

日志由 *操作* 和 *条目* 组成。一个操作对应于一个执行的动作，由一个或多个条目组成。条目作为操作详细执行情况。当进行用户操作日志查询时，返回的实体是 "UserOperationLogEntry" 类型，对应于 *条目* 。一个操作的所有条目都由一个相同的操作ID标出。

用户操作日志条目具有以下属性：

* **Operation ID**: 操作ID，唯一地识别执行操作的生成的ID。一个操作的多个条目都会引用相同的操作ID。
* **Operation Type**: 执行操作的类型。可用的操作类型在{{< javadocref page="?org/camunda/bpm/engine/history/UserOperationLogEntry.html" text="org.camunda.bpm.engine.history.UserOperationLogEntry" >}}中列出。请注意，一个操作可以由多种类型组成，例如，cascading API 操作是一个用户操作，但被分割成多种类型的操作。
* **Entity Type**: 操作所涉及的实体类型的标识符。可用的实体类型在{{< javadocref page="?org/camunda/bpm/engine/EntityTypes.html" text="org.camunda.bpm.engine.EntityTypes" >}}类中列出。与操作类型一样，一个操作可以处理一个以上的实体类型。
* **Category**: 该操作所关联的类别的名称。可用的类别在{{< javadocref page="?org/camunda/bpm/engine/history/UserOperationLogEntry.html" text="org.camunda.bpm.engine.history.UserOperationLogEntry" >}}中列出。例如，所有与任务相关的运行时操作，如申请和完成任务，都属于{{< javadocref page="?org/camunda/bpm/engine/history/UserOperationLogEntry.html#CATEGORY_TASK_WORKER" text="TaskWorker" >}}类别 。
* **Annotation**: 用户因为审核等原因而设置的文本注释。属于一个操作的多个日志条目具有相同的注释。
* **Entity IDs**: 一个作业日志条目包含实体ID，用于识别作业所涉及的实体。例如，一个任务的操作日志条目包含该任务的ID以及该任务所属的流程实例的ID。作为第二个例子，暂停一个流程定义的所有流程实例的日志条目不包含单个流程实例的ID，而只包含流程定义的ID。
* **User ID**: 执行该操作的用户的ID。
* **Timestamp**: 执行操作的时间。
* **Changed Property**: 用户操作改变的属性。用户操作可以改变多个属性，例如，流程实例的暂停更改了暂停状态属性。为操作中涉及的每个已更改的属性创建日志条目。
* **Old Property Value**: 更改属性的先前值。如果是`null`，则表示属性以前是`NULL`或未设置。
* **New Property Value**: 更改属性的新值。

## 用户操作日志的注释

用户操作日志对审计人工操作很有帮助。为了让人明白为什么要执行某个操作，有时只记录技术信息（如时间戳、操作类型等）是不够的，还要添加注释，把操作放在正确的业务背景中。

你可以直接为以下操作传递一个注释：

* [流程实例修改][op-log-set-annotation-instance-mod]

你还可以对一个已存在的操作日志设置注释：

可以通过Java API设置和清除注释：

```java
String operationId = historyService.createUserOperationLogQuery()
    .singleResult()
    .getOperationId();

String annotation = "Instances restarted due to wrong turn";
historyService.setAnnotationForOperationLogById(operationId, annotation);

historyService.clearAnnotationForOperationLogById(operationId);
```

**请注意:** 注释存在于属于操作日志的所有条目上。

也请参见REST API参考中的[设置][op-log-set-annotation-rest]和[清除][op-log-clear-annotation-rest]注释。

## 记录在用户操作日志中的操作词汇表

下面描述了在用户操作日志中记录的操作，以及作为其一部分创建的条目。

<table class="table table-striped">
  <tr>
    <th>Entity Type</th>
    <th>Operation Type</th>
	<th>Category</th>
    <th>Properties</th>
  </tr>
  <tr>
  <td>Task</td>
    <td>Assign</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>assignee</strong>: 被分配给任务的用户的ID</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Claim</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>assignee</strong>: 承接任务的用户的ID</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Complete</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>delete</strong>: 新的删除状态, <code>true</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Create</td>
	<td>TaskWorker</td>
    <td><i>没有记录额外的属性</i></td>
  </tr>
  <tr>
    <td></td>
    <td>Delegate</td>
	<td>TaskWorker</td>
    <td>
      委派任务时，创建了三个日志条目，其中包含以下属性之一：
      <ul>
        <li><strong>delegation</strong>: 由此产生的委托，<code>待办的</code></li>
        <li><strong>owner</strong>: 任务的原始所有者</li>
        <li><strong>assignee</strong>: 已分配给此任务的用户</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>TaskWorker</td>
    <td>
      <ul>
      <li><strong>delete</strong>: 新的删除状态, <code>true</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Resolve</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>delegation</strong>: 由此产生的委托 <code>已完成的</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetOwner</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>owner</strong>: 任务的新拥有者</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetPriority</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>priority</strong>: 任务的新优先事项</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>TaskWorker</td>
    <td>
      The manually changed property of a task, where manually means that a property got directly changed. Claiming a task via the TaskService wouldn't be logged with an update entry, but setting the assignee directly would be. One of the following is possible:
      <ul>
        <li><strong>description</strong>: The new description of the task</li>
        <li><strong>owner</strong>: The new owner of the task</li>
        <li><strong>assignee</strong>: The new assignee to the task</li>
        <li><strong>dueDate</strong>: The new due date of the task</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>DeleteHistory</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>nrOfInstances</strong>: the amount of decision instances that were deleted</li>
        <li><strong>async</strong>: by default <code>false</code> since the operation can only be performed synchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>ProcessInstance</td>
    <td>Create</td>
	<td>Operator</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>Activate</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: The new suspension state, <code>active</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Operator</td>
    <td>
      In case of regular operation:
      <ul><i>No additional property is logged</i></ul>
      In case of batch operation:
      <ul>
        <li><strong>nrOfInstances</strong>: the amount of process instances that were deleted</li>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch, <code>false</code> if operation was performed synchronously</li>
        <li><strong>deleteReason</strong>: the reason for deletion</li>
        <li><strong>type</strong>: <code>history</code> in case of deletion of historic process instances</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>ModifyProcessInstance</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>nrOfInstances</strong>: The amount of process instances modified</li>
        <li><strong>async</strong>: <code>true</code> if modification was performed asynchronously as a batch, <code>false</code> if modification was performed synchronously</li>
        <li><strong>processDefinitionVersion</strong>: The version of the process definition</li>
      </ul>
	</td>
  </tr>
  <tr>
    <td></td>
    <td>Suspend</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: The new suspension state, <code>suspended</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Migrate</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>processDefinitionId</strong>: The id of the process definition that instances are migrated to</li>
        <li><strong>nrOfInstances</strong>: The amount of process instances migrated</li>
        <li><strong>nrOfVariables</strong>: The amount of set variables. Only present when variables were set</li>
        <li><strong>async</strong>: <code>true</code> if migration was performed asynchronously as a batch, <code>false</code> if migration was performed synchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>RestartProcessInstance</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>nrOfInstances</strong>: The amount of process instances restarted</li>
        <li><strong>async</strong>: <code>true</code> if restart was performed asynchronously as a batch, <code>false</code> if restart was performed synchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>DeleteHistory</td>
  	<td>Operator</td>
    <td>
      <ul>
        <li><strong>nrOfInstances</strong>: the amount of process instances that were deleted</li>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch, <code>false</code> if operation was performed synchronously</li>
        <li><strong>deleteReason</strong>: the reason for deletion. This property exists only if the operation was performed asynchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>CreateIncident</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>incidentType</strong>: The type of incident that was created</li>
		<li><strong>configuration</strong>: The configuration of the incident that was created</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Resolve</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>incidentId</strong>: The id of the incident that was resolved</li>
      </ul>
    </td>
  </tr> 
  <tr>
    <td></td>
    <td>SetRemovalTime</td>
	  <td>Operator</td>
    <td>
      <ul>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch</li>
        <li><strong>nrOfInstances</strong>: The amount of updated instances</li>
        <li><strong>removalTime</strong>: The date of which an instance shall be removed</li>
        <li>
          <strong>mode</strong>: <code>CALCULATED_REMOVAL_TIME</code> if the removal time was calculated,
          <code>ABSOLUTE_REMOVAL_TIME</code> if the removal time was set explicitly,
          <code>CLEARED_REMOVAL_TIME</code> if the removal time was cleared
        </li>
        <li>
          <strong>hierarchical</strong>: <code>true</code> if the removal time was set across the hiearchy,
          <code>false</code> if the hierarchy was neglected
        </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetVariables</td>
	  <td>Operator</td>
    <td>
      <ul>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch</li>
        <li><strong>nrOfInstances</strong>: The amount of affected instances</li>
        <li><strong>nrOfVariables</strong>: The amount of set variables</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Incident</td>
    <td>SetAnnotation</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>incidentId</strong>: the id of the annotated incident</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>ClearAnnotation</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>incidentId</strong>: the id of the annotated incident</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>IdentityLink</td>
    <td>AddUserLink</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>candidate</strong>: The new candidate user associated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>DeleteUserLink</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>candidate</strong>: The previously associated user</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>AddGroupLink</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>candidate</strong>: The new group associated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>DeleteGroupLink</td>
	<td>TaskWorker</td>
    <td>
      <ul>
      <li><strong>candidate</strong>: The previously associated group</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Attachment</td>
    <td>AddAttachment</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>name</strong>: The name of the added attachment</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>DeleteAttachment</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>name</strong>: The name of the deleted attachment</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>JobDefinition</td>
    <td>ActivateJobDefinition</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>active</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetPriority</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>overridingPriority</strong>: the new overriding job priority. Is <code>null</code>, if the priority was cleared.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SuspendJobDefinition</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>suspended</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>ProcessDefinition</td>
    <td>ActivateProcessDefinition</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>active</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SuspendProcessDefinition</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>suspended</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>cascade</strong>: if the value is set to <code>true</code>, then all instances including history are also deleted.</li>
      </ul>
    </td>
  </tr>
   <tr>
    <td></td>
    <td>UpdateHistoryTimeToLive</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>historyTimeToLive</strong>: the new history time to live.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>DecisionDefinition</td>
    <td>UpdateHistoryTimeToLive</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>historyTimeToLive</strong>: the new history time to live.</li>
        <li><strong>decisionDefinitionId</strong>: the id of the decision definition whose history time to live is updated.</li>
        <li><strong>decisionDefinitionKey</strong>: the key of the decision definition whose history time to live is updated.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Evaluate</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>decisionDefinitionId</strong>: the id of the decision definition that was evaluated.</li>
        <li><strong>decisionDefinitionKey</strong>: the key of the decision definition that was evaluated.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>CaseDefinition</td>
    <td>UpdateHistoryTimeToLive</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>historyTimeToLive</strong>: the new history time to live.</li>
        <li><strong>caseDefinitionKey</strong>: the key of the case definition whose history time to live is updated.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Job</td>
    <td>ActivateJob</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>active</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetPriority</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>priority</strong>: the new priority of the job</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetJobRetries</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>retries</strong>: the new number of retries</li>
        <li><strong>nrOfInstances</strong>: the number of jobs that were updated.</li>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch, <code>false</code> if operation was performed synchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SuspendJob</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>suspended</code></li>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch, <code>false</code> if operation was performed synchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Execute</td>
	<td>Operator</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Operator</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>SetDueDate</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>duedate</strong>: the new due date of the job</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>RecalculateDueDate</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>creationDateBased</strong>: if the value is set to <code>true</code>, the new due date was calculated based on the creation date of the job. Otherwise, it was calculated using the date the recalcuation took place.</li>
		<li><strong>duedate</strong>: the new due date of the job</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>CreateHistoryCleanupJobs</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>immediatelyDue</strong>: <code>true</code> if the operation was performed immediately, <code>false</code> if the operation was scheduled regularly</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Variable</td>
    <td>ModifyVariable</td>
	<td>Operator/<br>TaskWorker</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>RemoveVariable</td>
	<td>Operator/<br>TaskWorker</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>SetVariable</td>
	<td>Operator/<br>TaskWorker</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>DeleteHistory</td>
	<td>Operator</td>
    <td>
      In case of single operation:
      <ul>
        <li><strong>name</strong>: the name of the variable whose history was deleted</li>
      </ul>
      In case of list operation by process instance:
      <ul><i>No additional property is logged</i></ul>
    </td>
  </tr>
  <tr>
    <td>Deployment</td>
    <td>Create</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>duplicateFilterEnabled</strong>: if the value is set to <code>true</code>, then during the creation of the deployment the given resources have been checked for duplicates in the set of previous deployments. Otherwise, the duplicate filtering has been not executed.</li>
        <li><strong>deployChangedOnly</strong>: this property is only logged when <code>duplicateFilterEnabled</code> is set to <code>true</code>. If the property value is set to <code>true</code> then only changed resources have been deployed. Otherwise, all resources are redeployed if any resource has changed.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>cascade</strong>: if the value is set to <code>true</code>, then all instances including history are also deleted.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Batch</td>
    <td>ActivateBatch</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>active</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SuspendBatch</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>suspensionState</strong>: the new suspension state <code>suspended</code></li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>cascadeToHistory</strong>: <code>true</code> if historic data related to the batch job is deleted as well, <code>false</code> if only the runtime data is deleted.</li>
      </ul>
    </td>    
  </tr>
  <tr>
    <td></td>
    <td>DeleteHistory</td>
	<td>Operator</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td></td>
    <td>SetRemovalTime</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch</li>
        <li><strong>nrOfInstances</strong>: The amount of updated instances</li>
        <li><strong>removalTime</strong>: The date of which an instance shall be removed</li>
        <li>
          <strong>mode</strong>: <code>CALCULATED_REMOVAL_TIME</code> if the removal time was calculated,
          <code>ABSOLUTE_REMOVAL_TIME</code> if the removal time was set explicitly,
          <code>CLEARED_REMOVAL_TIME</code> if the removal time was cleared
        </li>
      </ul>
    </td>
    </tr>
  <tr>
    <td>ExternalTask</td>
    <td>SetExternalTaskRetries</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>retries</strong>: the new number of retries</li>
        <li><strong>nrOfInstances</strong>: the amount of external tasks that were updated</li>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch, <code>false</code> if operation was performed synchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetPriority</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>priority</strong>: the new priority</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Unlock</td>
	<td>Operator</td>
    <td><i>No additional property is logged</i></td>
  </tr>
  <tr>
    <td>DecisionInstance</td>
    <td>DeleteHistory</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>nrOfInstances</strong>: the amount of decision instances that were deleted</li>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch, <code>false</code> if operation was performed synchronously</li>
        <li><strong>deleteReason</strong>: the reason for deletion. This property exists only if operation was performed asynchronously</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>SetRemovalTime</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>async</strong>: <code>true</code> if operation was performed asynchronously as a batch</li>
        <li><strong>nrOfInstances</strong>: The amount of updated instances</li>
        <li><strong>removalTime</strong>: The date of which an instance shall be removed</li>
        <li>
          <strong>mode</strong>: <code>CALCULATED_REMOVAL_TIME</code> if the removal time was calculated,
          <code>ABSOLUTE_REMOVAL_TIME</code> if the removal time was set explicitly,
          <code>CLEARED_REMOVAL_TIME</code> if the removal time was cleared
        </li>
        <li>
          <strong>hierarchical</strong>: <code>true</code> if the removal time was set across the hiearchy,
          <code>false</code> if the hierarchy was neglected
        </li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>CaseInstance</td>
    <td>DeleteHistory</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>nrOfInstances</strong>: The amount of case instances that were deleted. Only present if executed in bulk delete.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Metrics</td>
    <td>Delete</td>
	<td>Operator</td>
    <td>
      <ul>
        <li><strong>timestamp</strong>: The date for which all metrics older than that have been deleted. Only present if specified by the user.</li>
        <li><strong>reporter</strong>: The reporter for which all metrics reported by it have been deleted. Only present if specified by the user.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>TaskMetrics</td>
    <td>Delete</td>
    <td>Operator</td>
    <td>
      <ul>
        <li><strong>timestamp</strong>: The date for which all task metrics older than that have been deleted. Only present if specified by the user.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>OperationLog</td>
    <td>SetAnnotation</td>
	  <td>Operator</td>
    <td>
      <ul>
        <li><strong>operationId</strong>: the id of the annotated operation log</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>ClearAnnotation</td>
	  <td>Operator</td>
    <td>
      <ul>
        <li><strong>operationId</strong>: the id of the annotated operation log</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Filter</td>
    <td>Create</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>filterId</strong>: the id of the filter that been created</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>filterId</strong>: the id of the filter that been updated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>TaskWorker</td>
    <td>
      <ul>
        <li><strong>filterId</strong>: the id of the filter that been deleted</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>User</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>userId</strong>: the id of the user that has been created</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>userId</strong>: the id of the user that has been updated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>userId</strong>: the id of the user that has been deleted</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Unlock</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>userId</strong>: the id of the user that has been unlocked</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Group</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>groupId</strong>: the id of the group that has been created</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>groupId</strong>: the id of the group that has been updated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>groupId</strong>: the id of the group that has been deleted</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Tenant</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>tenantId</strong>: the id of the tenant that has been created</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>tenantId</strong>: the id of the tenant that has been updated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>tenantId</strong>: the id of the tenant that has been deleted</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Group membership</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>userId</strong>: the id of the user that has been added to the group</li>
		<li><strong>groupId</strong>: the id of the group that the user has been added to</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>userId</strong>: the id of the user that has been deleted from the group</li>
		<li><strong>groupId</strong>: the id of the group that the user has been deleted from</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>TenantMembership</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>tenantId</strong>: the id of the tenant that the group or user was associated with</li>
		<li><strong>userId</strong>: the id of the user that has been associated with the tenant. Is not present if the <code>groupId</code> is set</li>
		<li><strong>groupId</strong>: the id of the group that has been associated with the tenant. Is not present if the <code>userId</code> is set</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>tenantId</strong>: the id of the tenant that the group or user has been deleted from</li>
		<li><strong>userId</strong>: the id of the user that has been deleted from the tenant. Is not present if the <code>groupId</code> is set</li>
		<li><strong>groupId</strong>: the id of the group that has been deleted from the tenant. Is not present if the <code>userId</code> is set</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Authorization</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>permissions</strong>: the list of permissions that has been granted or revoked</li>
		<li><strong>permissionBits</strong>: the permissions bit mask that is persisted with the authorization</li>
		<li><strong>type</strong>: the type of authorization, can be either 0 (GLOBAL), 1 (GRANT) or 2 (REVOKE)</li>
		<li><strong>resource</strong>: the name of the resource type</li>
		<li><strong>resourceId</strong>: The id of the resource. Can be <code>'*'</code> if granted or revoked for all instances of the resource type.</li>
		<li><strong>userId</strong>: The id of the user the authorization is bound to. Can be <code>'*'</code> if granted or revoked for all users. Is not present when <code>groupId</code> is set.</li>
		<li><strong>groupId</strong>: The id of the group the authorization is bound to. Is not present when <code>userId</code> is set.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>permissions</strong>: the list of permissions that has been granted or revoked</li>
		<li><strong>permissionBits</strong>: the permissions bit mask that is persisted with the authorization</li>
		<li><strong>type</strong>: the type of authorization, can be either 0 (GLOBAL), 1 (GRANT) or 2 (REVOKE)</li>
		<li><strong>resource</strong>: the name of the resource type</li>
		<li><strong>resourceId</strong>: The id of the resource. Can be <code>'*'</code> if granted or revoked for all instances of the resource type.</li>
		<li><strong>userId</strong>: The id of the user the authorization is bound to. Can be <code>'*'</code> if granted or revoked for all users. Is not present when <code>groupId</code> is set.</li>
		<li><strong>groupId</strong>: The id of the group the authorization is bound to. Is not present when <code>userId</code> is set.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>permissions</strong>: the list of permissions that has been granted or revoked</li>
		<li><strong>permissionBits</strong>: the permissions bit mask that is persisted with the authorization</li>
		<li><strong>type</strong>: the type of authorization, can be either 0 (GLOBAL), 1 (GRANT) or 2 (REVOKE)</li>
		<li><strong>resource</strong>: the name of the resource type</li>
		<li><strong>resourceId</strong>: The id of the resource. Can be <code>'*'</code> if granted or revoked for all instances of the resource type.</li>
		<li><strong>userId</strong>: The id of the user the authorization is bound to. Can be <code>'*'</code> if granted or revoked for all users. Is not present when <code>groupId</code> is set.</li>
		<li><strong>groupId</strong>: The id of the group the authorization is bound to. Is not present when <code>userId</code> is set.</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td>Property</td>
    <td>Create</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>name</strong>: the name of the property that was created</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Update</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>name</strong>: the name of the property that was updated</li>
      </ul>
    </td>
  </tr>
  <tr>
    <td></td>
    <td>Delete</td>
	<td>Admin</td>
    <td>
      <ul>
        <li><strong>name</strong>: the name of the property that was deleted</li>
      </ul>
    </td>
  </tr>
</table>


# 使用自定义历史记录后端

为了理解如何提供一个自定义的历史记录后台，首先看一下历史架构的更详细的介绍是很有用的：

{{< img src="../img/process-engine-history-architecture.png" title="History Architecture" >}}

每当运行时实体的状态被改变时，流程引擎的核心执行组件就会触发历史事件。为了使其灵活，历史事件的实际创建以及用运行时结构的数据填充历史事件的工作被委托给历史事件生产者。生产者被交给运行时数据结构（如ExecutionEntity或TaskEntity），创建一个新的历史事件，并用从运行时结构中提取的数据填充它。

该事件接下来被传递到构成*历史后端*的历史事件处理程序。上面的图包含了一个名为*事件传输*的逻辑组件。这应该是代表产生事件的流程引擎核心组件和历史事件处理程序之间的通道。在默认的实现中，事件被同步地传递给历史事件处理程序，并且在同一个JVM中。然而，从概念上讲，有可能将事件流发送到不同的JVM（可能运行在不同的机器上），并使交付成为异步的。一个很好的选择是事务性的消息队列（JMS）。

一旦事件到达历史事件处理程序，它可以被处理并存储在某种数据存储中。默认的实现是将事件写入历史数据库，这样就可以用历史服务来查询它们。

将历史事件处理程序换成一个自定义的实现，允许用户插入一个自定义的历史后端。要做到这一点，需要两个主要步骤。

* 提供自定义的 {{< javadocref page="?org/camunda/bpm/engine/impl/history/handler/HistoryEventHandler.html" text="HistoryEventHandler" >}} 接口实现。
* 在流程引擎配置中配置自定义的实现。

{{< note title="Composite History Handling" class="info" >}}
  Note that if you provide a custom implementation of the HistoryEventHandler and wire it to the process engine, you override the default DbHistoryEventHandler. The consequence is that the process engine will stop writing to the history database and you will not be able to use the history service for querying the audit log. If you do not want to replace the default behavior but only provide an additional event handler, you can use the class `org.camunda.bpm.engine.impl.history.handler.CompositeHistoryEventHandler` that dispatches events to a collection of handlers.
{{< /note >}}
{{< note title="Spring Boot" class="info" >}}

Note that providing your custom `HistoryEventHandler` in a Spring Boot Starter environment works slightly differently. By default, the Camunda Spring Boot starter uses a `CompositeHistoryEventHandler` which wraps a list of HistoryEventHandler implementations that you can provide via the `customHistoryEventHandlers` engine configuration property. If you want to override the default `DbHistoryEventHandler`, you have to explicitly set the `enableDefaultDbHistoryEventHandler` engine configuration property to `false`.
{{< /note >}}


# 实现自定义历史级别

To provide a custom history level the interface `org.camunda.bpm.engine.impl.history.HistoryLevel` has to be implemented. The custom history level implementation
then has to be added to the process engine configuration, either by configuration or a process engine plugin.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

  <bean id="processEngineConfiguration" class="org.camunda.bpm.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration" >

    <property name="customHistoryLevels">
      <list>
        <bean class="org.camunda.bpm.example.CustomHistoryLevel" />
      </list>
    </property>

  </bean>

</beans>
```

The custom history level has to provide a unique id and name for the new history level.

```java
public int getId() {
  return 42;
}

public String getName() {
  return "custom-history";
}
```

If the history level is enabled, the method

```java
boolean isHistoryEventProduced(HistoryEventType eventType, Object entity)
```

is called for every history event to determine if the event should be saved to the history. The event types used in the
engine can be found in `org.camunda.bpm.engine.impl.history.event.HistoryEventTypes` (see [Javadocs][1]).

The second argument is the entity for which the event is triggered, e.g., a process instance, activity
instance or variable instance. If the `entity` is null the engine tests if the history level in general
handles such history events. If the method returns `false`, the engine will not generate
any history events of this type again. This means that if your history level only wants to generate the history
event for some instances of an event it must still return `true` if `entity` is `null`.

Please have a look at this [complete example][2] to get a better overview.

## Removal Time Inheritance
Historic instances inherit the [removal time]({{< relref "#removal-time" >}}) from the respective historic top-level
instance. If the custom history level is configured in a way, so that the historic top-level instance is not written,
the removal time is not available.

The following historic instances are considered as top-level instances:

* Batch instance
* Root process instance
* Root decision instance

## User Operation Logs and Custom History Level

The following implementation is required in order to enable User Operation Logs:

```java
public boolean isHistoryEventProduced(HistoryEventType eventType, Object entity) {
  if (eventType.equals(HistoryEventTypes.USER_OPERATION_LOG)){
    return true;
  }
  ...
}
```

# 历史清理

When used intensively, the process engine can produce a huge amount of historic data. *History Cleanup* is a feature that removes this data based on configurable time-to-live settings.

It deletes:

* Historic process instances plus all related historic data (e.g., historic variable instances, historic task instances, historic instance permissions, all comments and attachments related to them, etc.)
* Historic decision instances plus all related historic data (i.e., historic decision input and output instances)
* Historic case instances plus all related historic data (e.g., historic variable instances, historic task instances, etc.)
* Historic batches plus all related historic data (historic incidents and job logs)

History cleanup can be triggered manually or scheduled on a regular basis. Only [camunda-admins]({{< ref "/user-guide/process-engine/authorization-service.md#the-camunda-admin-group">}}) have permissions to execute history cleanup manually.

## History Cleanup by Example

Assume we have a billing process for which we must keep the history trail for ten years for legal compliance reasons. Then we have a holiday application process for which history data is only relevant for a short time. In order to reduce the amount of data we have to store, we want to quickly remove holiday-related data.

With history cleanup, we can assign the billing process a history time to live of ten years and the holiday process a history time to live of seven days. History cleanup then makes sure that history data is removed when the time to live has expired. This way, we can selectively keep history data based on its importance for our business. At the same time, we only keep what is necessary in the database.

Note: The exact time at which data is removed depends on a couple of configuration settings, for example the selected *history cleanup strategy*. The underlying concepts and settings are explained in the following sections.

## Basic Concepts

### Cleanable Instances

The following elements of Camunda history are cleanable:

* Process Instances
* Decision Instances
* Case Instances
* Batches

Note that cleaning one such instance always removes all dependent history data along with it. For example, cleaning a process instance removes the historic process instance as well as all historic activity instances, historic task instances, etc.

### History Time To Live (TTL)

*History Time To Live* (TTL) defines how long historic data shall remain in the database before it is cleaned up.

* Process, Case and Decision Instances: TTL can be defined in the XML file of the corresponding definition. This value can furthermore be changed after deployment via Java and REST API.
* Batches: TTL can be defined in the process engine configuration.

See the [TTL configuration section](#history-time-to-live) for how to set TTL.

### Instance End Time

*End Time* is the time when an instance is no longer active.

* Process Instances: The time when the instance finishes.
* Decision Instances: The time when the decision is evaluated.
* Case Instances: The time when the instance completes.
* Batches: The time when the batch completes.

The end time is persisted in the corresponding instance tables `ACT_HI_PROCINST`, `ACT_HI_CASEINST`, `ACT_HI_DECINST` and `ACT_HI_BATCH`.

### Instance Removal Time

*Removal Time* is the time after which an instance shall be removed. It is computed as `removal time = base time + TTL`. *Base time* is configurable and can be either the start or the end time of an instance. In particular, this means:

* Process Instances: Base time is either the time when the process instance starts or the time at which it finishes. This is configurable.
* Decision Instances: Base time is the time when the decision is evaluated.
* Case Instances: The removal time concept is not implemented for case instances.
* Batches: Base time is either the time when the batch is created or when the batch is completed. This is configurable.

For process and decision instances in a hierarchy (e.g. a process instance that is started by another process instance via a BPMN Call Activity), the removal time of all instances is always equal to the removal time of the root instance.

{{< img src="../img/history-cleanup-process-hierarchy.png" title="History Cleanup" >}}

The removal time is persisted in *all* history tables. So in case of a process instance, the removal time is present in `ACT_HI_PROCINST` as well as the corresponding secondary entries in `ACT_HI_ACTINST`, `ACT_HI_TASKINST` etc.

See the [Removal Time Strategy configuration section](#removal-time-strategy) for how to configure if the removal time is based on the start or end time of an instance.

## Cleanup Strategies

In order to use history cleanup, you must decide for one of the two avialable history cleanup strategies: *Removal-Time-based* or *End-Time-based* strategy. The *Removal-Time-based* strategy is the default strategy and recommended in most scenarios. The following sections describe the strategies and their differences in detail. See the [Cleanup Strategy configuration section](#cleanup-strategy) for how to configure each of the strategies.

### Removal-Time-based Strategy

The *removal-time-based cleanup strategy* deletes data for which the removal time has expired.

Strengths:

* Since every history table has a removal time attribute, history cleanup can be done with simple `DELETE FROM <TABLE> WHERE REMOVAL_TIME_ < <now>` SQL statements. This is much more efficient than end-time-based cleanup.
* Since removal time is consistent for all instances in a hierarchy, a hierarchy is always cleaned up entirely once the removal time has expired. It cannot happen that instances are removed at different times.

Limitations:

* Can only remove data for which a removal time is set. This is especially not the case for data which has been created with Camunda versions < 7.10.0.
* Changing the TTL of a definition only applies to history data that is created in the future. It does not dynamically update the removal time of already written history data. However, it is possible to [Set a Removal Time via Batch Operations]({{< ref "/user-guide/process-engine/batch-operations.md#set-a-removal-time">}}).
* History data of case instances is not cleaned up.

### End-Time-based Strategy

The *end-time-based cleanup strategy* deletes data whose end time plus TTL has expired. In contrast to the removal-time strategy, this is computed whenever history cleanup is performed.

Strengths:

* Changing the TTL of a definition also affects already written history data.
* Can remove data from any Camunda version.

Limitations:

* End time is only stored in the instances tables (`ACT_HI_PROCINST`, `ACT_HI_CASEINST`, `ACT_HI_DECINST` and `ACT_HI_BATCH`). To delete data from all history tables, the cleanable instances are first fetched via a `SELECT` statement. Based on that, `DELETE` statements are made for each history table. These statements can involve joins. This is less efficient than removal-time-based history cleanup.
* Instance hierarchies are not cleaned up atomically. Since the individual instances have different end times, they are going to be cleaned up at different times. In consequence, hierarchies can appear partially removed.
* [Historic Instance Permissions] are not cleaned up.
* [History Cleanup Jobs]({{< ref "/user-guide/process-engine/history.md#historycleanupjobs-in-the-historic-job-log">}}) are not removed from the historic job log.

## Cleanup Internals

History cleanup is implemented via jobs and performed by the [job executor]({{< ref "/user-guide/process-engine/the-job-executor.md">}}). It therefore competes for execution resources with other jobs, e.g. triggering of BPMN timer events.

Cleanup execution can be controlled in three ways:

* Cleanup Window: Determines a time frame in which history cleanup runs. This allows to use the job executor's resources only when there is little load on your system (e.g. at night time or weekends). Default value: No cleanup window is defined. That means that history cleanup is not performed automatically.
* Batch Size: Determines how many instances are cleaned up in one cleanup transaction. Default: 500.
* Degree of Parallelism: Determines how many cleanup jobs can run in parallel. Default: 1 (no parallel execution).

See the [Cleanup configuration section](#history-cleanup-configuration) for how to set each of these values.

If there is no cleanable data left, the cleanup job performs exponential backoff between runs to reduce system load. This backoff is limited to a maximum of one hour. Backoff does not apply to manual cleanup runs.

If cleanup fails, the job executor's [retry mechanism]({{< ref "/user-guide/process-engine/the-job-executor.md#failed-jobs">}}) applies. Once the cleanup job has run out of retries, it is not executed again until one of the following actions is performed:

* History cleanup is triggered manually
* The process engine is restarted (this resets the number of job retries to the default value)
* The number of job retries is increased manually (e.g. via Java or REST API)

The history cleanup jobs can be found via the API method `HistoryService#findHistoryCleanupJobs`.

## History Cleanup Configuration

### History Time To Live

#### Process/Decision/Case Definitions

Process instances are only cleaned up if their corresponding definition has a valid time to live (TTL).
Use the ["historyTimeToLive" extension attribute]({{< ref "/reference/bpmn20/custom-extensions/extension-attributes.md#historytimetolive">}}) of the process definition to define the TTL for all its instances:

```xml
<process id="oneTaskProcess" name="The One Task Process" isExecutable="true" camunda:historyTimeToLive="5">
...
</process>
```

TTL can also be defined in ISO-8601 date format. The function only accepts the notation to define the number of days.

```xml
<process id="oneTaskProcess" name="The One Task Process" isExecutable="true" camunda:historyTimeToLive="P5D">
...
</process>
```

Once deployed, TTL can be updated via Java API:

```java
  processEngine.getRepositoryService().updateProcessDefinitionHistoryTimeToLive(processDefinitionId, 5);
```

Setting the value to `null` clears the TTL. The same can be done via [REST API]({{< ref "/reference/rest/process-definition/put-history-time-to-live.md">}}).

For decision and case definitions, TTL can be defined in a similar way.

In case you want to provide an engine-wide default TTL for all process, decision and case definitions,
use the ["historyTimeToLive" attribute]({{< ref "/reference/deployment-descriptors/tags/process-engine.md#historytimetolive">}})
of the process engine configuration. This value is applied as the default whenever new definitions without TTL are deployed. Note that it therefore does not change the TTL of already deployed definitions. Use the API method given above to change TTL in this case.

#### Batches

TTL for batches can be defined via attribute of the process engine configuration.

```xml
<!-- default setting for all batch operations -->
<property name="batchOperationHistoryTimeToLive">P5D</property>
```

The `batchOperationsForHistoryCleanup` property can be configured in Spring based application or via custom [Process Engine Plugins]({{< ref "/user-guide/process-engine/process-engine-plugins.md">}}). It defines history time to live for each specific historic batch operation.

```xml
<!-- specific TTL for each operation type -->
<property name="batchOperationsForHistoryCleanup">
  <map>
    <entry key="instance-migration" value="P10D" />
    <entry key="instance-modification" value="P7D" />
    <entry key="instance-restart" value="P1D" />
    <entry key="instance-deletion" value="P1D" />
    <entry key="instance-update-suspension-state" value="P20D" />
    <entry key="historic-instance-deletion" value="P4D" />
    <entry key="set-job-retries" value="P5D" />
    <entry key="set-external-task-retries" value="P5D" />
    <entry key="process-set-removal-time" value="P0D" />
    <entry key="decision-set-removal-time" value="P0D" />
    <entry key="batch-set-removal-time" value="P0D" />
    <entry key="set-variables" value="P1D" />
    <!-- in case of custom batch jobs -->
    <entry key="custom-operation" value="P3D" />
  </map>
</property>
```

If the specific TTL is not set for a batch operation type, then the option `batchOperationHistoryTimeToLive` applies.

#### Job Logs

A history cleanup is always performed by executing a history cleanup job. As with all other jobs, the history cleanup job 
will produce events that are logged in the historic job log. By default, those entries will stay in the log indefinitely 
and cleanup must be configured explicitly. Please note that this only works for the [removal-time based history cleanup strategy]({{< ref "/user-guide/process-engine/history.md#removal-time-strategy">}}).

The `historyCleanupJobLogTimeToLive` property can be used to define a TTL for historic job log entries produced by 
history cleanup jobs. The property accepts values in the ISO-8601 date format. Note that only the notation to define a number of days is allowed.

```xml
<property name="historyCleanupJobLogTimeToLive">P5D</property>
```

#### Task Metrics

The process engine reports [runtime metrics]({{< ref "/user-guide/process-engine/metrics.md">}}) to the database that can help draw conclusions about usage, load, and performance of the BPM platform.
With every assignment of a user task, the related task worker is stored as a pseudonymized, fixed-length value in the `ACT_RU_TASK_METER_LOG` table together with a timestamp. Cleanup for this data needs to
be configured explicitly if needed.

The `taskMetricsTimeToLive` property can be used to define a TTL for task metrics entries produced by user task assignments. 
The property accepts values in the ISO-8601 date format. Note that only the notation to define a number of days is allowed.

```xml
<property name="taskMetricsTimeToLive">P540D</property>
```

{{< note title="Heads Up!" class="warning" >}}
If you are an enterprise customer, your license agreement might require you to report some metrics annually. Please store task metrics from `ACT_RU_TASK_METER_LOG` for at least 18 months until they were reported.
{{< /note >}}

### Cleanup Window

For automated history cleanup on a regular basis, a batch window must be configured - the period of time during the day when the cleanup is to run.

Use the following engine configuration properties to define a batch window for every day of the week:

```xml
<property name="historyCleanupBatchWindowStartTime">20:00</property>
<property name="historyCleanupBatchWindowEndTime">06:00</property>
```

Cleanup can also be scheduled individually for each day of the week (e.g. run cleanup only on weekends):

```xml
<!-- default for all weekdays -->
<property name="historyCleanupBatchWindowStartTime">20:00</property>
<property name="historyCleanupBatchWindowEndTime">06:00</property>

<!-- overriding batch window for saturday and sunday -->
<property name="saturdayHistoryCleanupBatchWindowStartTime">06:00</property>
<property name="saturdayHistoryCleanupBatchWindowEndTime">06:00</property>
<property name="sundayHistoryCleanupBatchWindowStartTime">06:00</property>
<property name="sundayHistoryCleanupBatchWindowEndTime">06:00</property>
```

By default, no cleanup window is configured. In that case, history cleanup is not performed automatically.

See the [engine configuration reference][configuration-options] for a complete list of all parameters.

### Cleanup Strategy

Removal-time-based or end-time-based cleanup can be selected as follows:

```xml
<property name="historyCleanupStrategy">removalTimeBased</property>
```

Valid values are `removalTimeBased` and `endTimeBased`. `removalTimeBased` is the default.

### Removal-Time Strategy

Removal time is defined per instance as `removal time = base time + TTL`. `base time` can be either the start or end time of the instance in case of process instances. This can be configured in the process engine configuration as follows:

```xml
<property name="historyRemovalTimeStrategy">end</property>
```

Valid values are `start`, `end` and `none`. `end` is the default value and the recommended option. `start` is a bit more efficient when the process engine populates the history tables, because it does not have to make extra `UPDATE` statements when an instance finishes.

{{< note title="Heads-up!" class="info" >}}
The calculation of the removal time can be enabled independently of the selected cleanup strategy of the process engine.
This allows to perform a custom cleanup procedure outside the process engine by leveraging database capabilities (e.g. via table partitioning by removal time).
{{< /note >}}

### Parallel Execution

The degree of parallel execution for history cleanup can be defined in the engine configuration as follows:

```xml
<property name="historyCleanupDegreeOfParallelism">4</property>
```

Valid values are integers from 1 to 8. 1 is the default value.

This property specifies the number of jobs used for history cleanup. In consequence, this value determines how many job executor threads and database connections may be busy with history cleanup at once. Choosing a high value can make cleanup faster, but may steal resources from other tasks the engine and database have to perform.

### Cleanup Batch Size

The number of instances that are removed in one cleanup transaction can be set as follows:

```xml
<property name="historyCleanupBatchSize">100</property>
```

The default (and maximum) value is 500. Reduce it if you notice transaction timeouts during history cleanup.

### Clustered Cleanup

In a multi-engine setup, you can configure whether a specific engine should participate in history cleanup or not.
Please make sure that the same cleanup execution configuration (window, batch size, degree of parallelism) is present 
on all participating nodes.

#### Cleanup Execution Participation per Node

Sometimes it is necessary to exclude some nodes in a multi-engine setup from performing history cleanup execution, 
e. g. to reduce the load on some nodes.

You can disable the history cleanup execution for each node with the following flag:
```xml
<property name="historyCleanupEnabled">false</property>
```

When you exclude a node from executing history cleanup, you don't need to specify the configuration properties 
related to the cleanup execution since the particular node ignores them. 

**Please Note:** The history cleanup configuration properties that are unrelated to the cleanup execution (e.g., 
time to live, removal time strategy) still need to be defined among all nodes. 

[configuration-options]: {{< ref "/reference/deployment-descriptors/tags/process-engine.md#history-cleanup-configuration-parameters">}}
[1]: http://docs.camunda.org/latest/api-references/javadoc/org/camunda/bpm/engine/impl/history/event/HistoryEventTypes.html
[2]: https://github.com/camunda/camunda-bpm-examples/tree/master/process-engine-plugin/custom-history-level
[op-log-set-annotation-rest]: {{< ref "/reference/rest/history/user-operation-log/set-annotation.md" >}}
[op-log-clear-annotation-rest]: {{< ref "/reference/rest/history/user-operation-log/clear-annotation.md" >}}
[op-log-set-annotation-instance-mod]: {{< ref "/user-guide/process-engine/process-instance-modification.md#annotation" >}}
[Historic Instance Permissions]: {{< ref "/user-guide/process-engine/authorization-service.md#historic-instance-permissions" >}}
